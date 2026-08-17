---
title: "Scripts Tap the Text Box but Nothing Gets Typed: Clipboard and Text
  Input Failures on Cloud Phones"
description: "A diagnostic ladder for silent Android automation text input
  failures: injection method, Android clipboard privacy changes (10/13), and
  IME/focus quirks — with quick checks and fallback entry methods for every
  layer."
pubDate: 2026-08-17
updatedDate: 2026-08-17
---

## Answer First

**Definition:** A silent text-entry failure is any case where an automation script taps (or focuses) a text field on an Android device and the field ends up empty — or filled with characters the script never sent — while every step still reports success. There is no error dialog, no toast, no failed assertion: the tap lands, the field highlights, and the text simply never arrives.

**Why:** Every way of getting text into a field depends on assumptions that quietly stop being true: the injection method must be supported by the app, Android's clipboard rules must allow the operation, and the input method (IME) must be present, focused, and set to the right language. When one breaks, the platform has nothing to report — silent by design. On cloud phones the effect compounds: fleets of near-identical devices drift apart in Android version, OEM image, and IME state, and nobody is watching a given phone — the mobile form of [an agent failing mid-task](/blog/ai-agent-fails-what-happens-next/) with no exception to surface.

**Example:** A script writes `admin@example.com` to the clipboard, taps the email field, and sends a paste. On Android 13 a "Clipboard shown by …" overlay appears when a background component reads the clipboard back to verify — and on Android 10 that background read returns nothing, because only the focused app or the default IME may read the clipboard. The field stays empty; the task log shows a completed paste.

## Key Facts

- **Three failure layers.** Text is injected via accessibility `setText`, clipboard paste, or ADB `input text`; each fails differently and needs a different fallback.
- **Android 10 (API 29) restricted clipboard reads.** Apps not in the foreground and not the default IME can no longer read clipboard data.
- **Android 13 (API 33) added a clipboard preview.** The system shows an overlay naming the app whenever clipboard content is read.
- **Accessibility `setText` bypasses the clipboard entirely.** It writes directly to the field node, but needs an enabled accessibility service and an editable node.
- **ADB `input text` goes through the keyboard path.** It injects key events into whatever has focus, failing without focus, on non-ASCII text, or when no default IME is provisioned.
- **WebView fields behave differently from native fields.** Some ignore key-event injection, some ignore accessibility text, and a few apps block all scripted input — the same divide as [agents that can use browsers but not phone apps](/blog/ai-agents-can-use-browsers-what-about-phone-apps/).

## Expert Explanation

Treat Android automation text input as three independent layers. When a script taps a field and nothing appears, the bug is almost always in one of them — and the fix usually lives in a different layer.

### Layer 1: How the script injects text

Three entry methods cover almost every automation stack:

- **Accessibility `setText`.** An accessibility service targets the field's node and performs `ACTION_SET_TEXT`, writing the value directly — no clipboard, no IME. *Symptom of failure:* the field never reacts and the service logs nothing, typically because the service is disabled, the field is a custom view with no editable node, or the app blocks secure fields. *Quick check:* confirm the service is enabled and a screen reader can read the field. *Fallback:* clipboard paste or ADB input.
- **Clipboard paste.** The script writes text to the clipboard, focuses the field, and triggers paste. It works in almost every app, which is why so many stacks default to it. *Symptom of failure:* paste fires but nothing appears, or the field fills with stale content from a previous run. *Quick check:* write a marker string and read it back immediately while the target app still has focus. *Fallback:* accessibility `setText`, which never touches the clipboard.
- **ADB `input text`.** `adb shell input text "…"` simulates key events, making it the most literal and least reliable method. Spaces must be escaped as `%s`, non-ASCII and many symbols are dropped, and the text goes wherever focus happens to be. *Symptom of failure:* characters appear in the wrong field, partial text appears, or nothing appears. *Quick check:* run the same command manually and watch the focused field. *Fallback:* accessibility `setText` or clipboard paste.

### Layer 2: Android's clipboard privacy changes

Paste-based automation is the layer most often broken by the platform itself:

- **Android 10 restricted background clipboard reads.** Only the focused app or the default IME may read the clipboard. Frameworks that use a background service to write, read back, or paste fail silently: the write succeeds, the read returns nothing.
- **Android 13 added the clipboard preview.** Any read of clipboard content shows an overlay naming the reading app.
- **Clipboard auto-clear.** Many Android 13+ devices clear clipboard content after a short time unless an app with focus is using it — a script that writes minutes before pasting gets nothing.

The pattern in all three: *the clipboard is a handoff, not storage* — and one more reason [clipboard data deserves the same controls as any other credential](/blog/agentic-automation-security-cloud-phone-accounts/). *Quick check:* write and paste in the same instant; if that works but a delayed paste fails, the clipboard — not the app — is the problem. *Fallback:* re-write the clipboard immediately before paste, or drop the clipboard entirely and use accessibility `setText`.

### Layer 3: Input method and focus quirks

The last layer is the environment around the keyboard:

- **Default IME not provisioned.** Fresh cloud-phone images often have no default input method set, or the keyboard app is installed but not activated. `input text` and key events silently go nowhere.
- **Language and keyboard switching.** If the default IME is a non-Latin layout, ASCII characters can be composed, transliterated, or dropped — the script sends "john" and the field receives nothing, or an unexpected string.
- **WebView text fields.** Fields rendered inside a WebView can ignore both key-event injection and accessibility text, especially when the page uses custom JavaScript input handling. The same script works in the app's native UI and fails in its in-app browser or checkout.

*Quick check:* screenshot immediately after the tap. If the keyboard is not visible, focus or the IME is the problem; if it is visible but typing does nothing, test the same text in a plain field, like a browser address bar. *Fallback:* provision a fixed default IME and language once in the image, and have the script re-tap, wait for focus, then retry.

## Decision Framework

Walk from cheapest check to cheapest fix, routing anything ambiguous to [monitoring and human review](/blog/ai-agent-control-tower-for-mobile-app-workflows/):

| Symptom | Likely layer | Quick check | Fallback entry method |
| --- | --- | --- | --- |
| Paste fires, field stays empty | Clipboard | Read clipboard back immediately; look for the Android 13 preview overlay | Accessibility `setText` |
| Field fills with stale text | Clipboard | Check clipboard contents before the paste step | Clear, then re-write right before paste |
| Text lands in the wrong field | IME/focus | Confirm focus with a screenshot | Re-tap, wait for focus, retry |
| ASCII works, symbols don't | Injection | Test `input text` with a symbol in a terminal | Accessibility `setText` |
| Works in native UI, fails in webviews | IME/focus | Try the same field in the app's browser | Accessibility `setText`, then paste |
| Nothing types, keyboard invisible | IME/focus | Screenshot; check IME settings | Provision a default IME in the image |

**Provisioning checklist (once per image, not per task):** set and activate a fixed default IME; set the IME language explicitly; disable clipboard auto-clear where the device allows; enable the automation accessibility service; pick one primary injection method and log it per task.

## Key Takeaways

- **Assume no entry method works everywhere.** Build a fallback chain — accessibility `setText` → clipboard paste → ADB `input text` — and step down it on failure.
- **Verify by reading the field, not the clipboard.** A pasted value and a received value are different things; reading the clipboard verifies the wrong one.
- **Treat the clipboard as a handoff.** Write immediately before paste, never minutes earlier; expect Android 10+ background restrictions and Android 13+ previews in screenshots.
- **Provision the keyboard once.** IME and language failures look exactly like app failures, and are the most common silent cause on fresh cloud-phone images.
- **Log the injection attempt.** Record method, target field, and field value after entry — when a fleet behaves differently, logs are the only way to see which layer failed, and [mobile automation needs them even more](/blog/ai-agent-logs-for-mobile-automation/).
- **Some fields are off-limits by design.** Apps can mark fields (passwords, financial forms) so no legitimate injected input is accepted — app policy, not a script bug, and no entry method is a legitimate workaround.

## FAQ

**Q: My script taps a field and nothing appears, but the app looks completely normal. What is most likely wrong?**

A: Start with the IME. Freshly provisioned cloud phones often have no default keyboard set, or a keyboard language that doesn't match the scripted text, so ADB `input text` and paste-based flows silently fail. Screenshot the field right after the tap: no keyboard visible points to IME or focus; a visible keyboard with no text points to injection or clipboard restrictions.

**Q: Does the Android 13 clipboard preview break automation?**

A: It breaks *invisible* automation: any read of clipboard content now shows a "Clipboard shown by …" overlay that can confuse monitoring and human review. It does not block paste itself, but combined with Android 10's background read restriction and auto-clear on many Android 13+ devices, paste becomes the least reliable entry method. Prefer accessibility `setText`.

**Q: Which text entry method is most reliable for cloud phone automation?**

A: Accessibility `setText` — it writes directly to the field, bypassing clipboard and keyboard. Its price is an enabled accessibility service and an app that exposes the field as editable. Use it first, with clipboard paste second and ADB `input text` last.

**Q: Why does the same script work in an app's native fields but fail in its WebView fields?**

A: WebView-rendered fields often ignore key-event injection and sometimes ignore accessibility text, particularly when the page uses custom JavaScript input handling. It is a per-app quirk, not a platform setting, so device-level checks won't catch it — test each app's webview separately and assign a different fallback.

## Sources

- OWASP MASVS: Mobile Application Security Verification Standard — clipboard misuse (MASWE-0030) and accessibility data exposure (MASWE-0040) controls: https://mas.owasp.org/MASVS/
- Android 10 privacy changes — background clipboard access restriction: https://developer.android.com/about/versions/10/privacy/changes
- Android 13 behavior changes — clipboard preview shown when clipboard data is read: https://developer.android.com/about/versions/13/behavior-changes-13
- Android Debug Bridge (adb) reference — `input` command and key-event injection: https://developer.android.com/tools/adb
- Accessibility service reference — `ACTION_SET_TEXT` for setting text on a node: https://developer.android.com/reference/android/accessibilityservice/AccessibilityService
