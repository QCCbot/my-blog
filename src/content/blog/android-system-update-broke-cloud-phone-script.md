---
title: My Cloud Phone Script Broke After an Android System Update. What Should I
  Check First?
description: "Your AutoJS script broke after an Android update? Check the system
  layer first: OS and WebView changes, accessibility-service resets, and
  permissions — in a fixed order."
pubDate: 2026-08-01
updatedDate: 2026-08-01
---

## Answer First

**Definition:** A script break caused by the Android system layer rather than by the app you automate or the script's own logic. The system layer includes OS version bumps, security patches, WebView updates, accessibility-service resets, and targetSdk behavior changes. The app can look identical; the platform underneath it changed.

**Why:** Because scripts like AutoJS run on top of system services that Android can change or reset with no visible signal. If you start diagnosing at the app layer — hunting for new popups or stale selectors — you burn hours rewriting scripts that never broke. A system-layer check takes minutes and usually points at the real cause: an update you did not initiate. On managed cloud-phone fleets this is a recurring, involuntary failure source, which is exactly why it deserves its own check before any script surgery.

**Example:** A cloud-phone fleet runs the same AutoJS script for weeks. One night devices roll from Android 12 to Android 13 with a fresh WebView build. The next morning, the script that reads in-app web content fails on part of the fleet. First assumption: the site changed its selectors. Reality: the accessibility service AutoJS depends on was reset during the update, and the WebView engine bumped. The check order below surfaces both in minutes instead of hours.

## Key Facts

- Android OS updates and security patches land at the system layer and can change how apps behave even when the app you automate never updated.
- Android System WebView updates independently of the OS, so selector-based automation over web content can break between WebView versions with zero app changes.
- AutoJS depends on the Android accessibility service, which is opt-in, must be explicitly enabled, and can be turned off or reset — including after updates on some device images.
- Google Play requires apps to keep raising targetSdkVersion, and each bump can change runtime behavior: permission models, background execution, and notification rules.
- The same script can fail on some phones and not others because fleet devices update at different times — which reads as flaky logic, not a platform change.

## Expert Explanation

Most "why did my script fail" articles diagnose the app layer: the app redesigned a screen, a popup intercepted a tap, a selector went stale. Those are real causes, but they are not the only ones. On managed cloud phones there is a whole class of breakage that originates below the app, in the Android system layer. If you operate automation at fleet scale, this is the layer you own — it is the [mobile operations layer](/blog/agentic-apps-need-mobile-operations-layer/) that most agent frameworks never touch.

Four distinct system-layer causes explain most "broke after an update" cases:

**OS version bumps and security patches.** Managed cloud phones receive Android updates automatically. A new Android version redefines the constraints every app runs under: stricter background execution, tighter foreground-service rules, a changed permission model. The automated app does not need to change — its contract with the OS did. Android publishes behavior-change lists for each release precisely because apps must adapt when the platform shifts.

**WebView auto-updates.** If your script drives in-app web content — login pages, H5 interfaces, web views inside apps — it is interacting with the WebView rendering engine, which updates through the Play Store independently of the OS version. A WebView update can change rendering, timing, and how the view tree is exposed, even when the site's HTML is byte-identical. Automation tools read that view tree; an engine change alters what the tree contains, and selectors that resolved yesterday resolve to nothing today.

**Accessibility-service resets.** AutoJS relies on an accessibility service to inspect the UI hierarchy and inject actions. Accessibility services are not on by default: they are enabled in Settings and can be disabled at any time. On some device images, an update resets that toggle. The script then runs but cannot see the screen — producing errors that look like selector problems when the actual cause is a dead service. This is the single most common silent breakage after a system update, and it needs zero script edits to fix.

**targetSdk behavior changes.** Even when the OS version string stays the same, the apps you automate quietly raise their targetSdkVersion over time — Google Play requires it — and each jump activates a new batch of platform behavior: Android 13's runtime notification permission, Android 14's stricter foreground-service and background-start rules. Automation that depended on silent notifications or background work breaks even though neither the OS nor your script changed.

The practical limit of this layer is honest: you cannot prevent Android updates on managed devices unless your platform supports pinning OS images, and you should not skip security patches indefinitely. WebView versions are not freely freezable on most managed images. Some changes — engine-level WebView behavior, targetSdk shifts — are not reversible at all. The goal of the check order is to find the correct, smallest fix, not to freeze the platform in place.

## Decision Framework

Use this check order in order. Step 1 sets the context for everything after it; steps 2 and 3 catch the most common causes without a single line of script changes.

| # | Check | How to verify | If it failed |
|---|-------|---------------|--------------|
| 1 | OS / WebView changed | Settings → About phone: Android version, security patch date; Apps → Android System WebView version | Note the delta — it is your failure timeline |
| 2 | Accessibility service survived | Settings → Accessibility → AutoJS service toggle | Re-enable it, re-run before touching selectors |
| 3 | Permissions intact | App info → permissions: overlay, notification, storage, special access | Regrant; Android 13+ needs explicit notification permission |
| 4 | Test on one phone | Run the exact failing script on a single device | Narrows it to device state vs. script logic |
| 5 | Re-selector or pin version | Update selectors for the new engine/UI, or pin the app version where your platform allows | Do one, not both, and re-test |

**Step 1 — confirm the system actually changed.** Every other step's meaning depends on this. If the Android version, security patch date, and WebView version are identical to your last known-good run, you are in app-layer territory — and you ruled the system layer out in under two minutes. If they changed, you have your timeline: compare the failure start against the update.

**Step 2 — verify the accessibility service survived.** The most skipped check and the most common cause. Re-enable the service in Settings → Accessibility and re-run the script before you edit anything. A large share of "broke after update" reports end here.

**Step 3 — re-check permissions.** Updates and reinstalls can reset overlay, notification, and storage grants — [permissions and audit trails](/blog/ai-agent-permissions-audit-trails-cloud-phone/) are exactly where this class of breakage hides. Android 13 and later require a separate runtime notification permission, and automation that relied on notifications or overlay rendering breaks silently when it is missing.

**Step 4 — test on one phone.** Reproduce the failure on a single device before touching the fleet. This is the same discipline as any [failed-task post-mortem](/blog/ai-agent-fails-what-happens-next/): narrow the blast radius, confirm the repro, then act.

**Step 5 — re-selector or pin the version, one at a time.** Only now rewrite selectors for the new WebView or UI, or [pin the automated app's version](/blog/ai-agent-control-tower-for-mobile-app-workflows/) where your platform supports it. Change one variable at a time and re-test on the same device. Remember that pinning the app does not stop OS or WebView updates — it only isolates one variable.

## Key Takeaways

- Check the system layer before the app layer: OS version, security patch date, and WebView version first.
- Verify the accessibility service is still enabled before you touch a single selector — it is the most common silent breakage.
- Re-check permissions after every update; Android 13+'s notification permission is a classic silent regression.
- Test on one device before rewriting anything fleet-wide.
- Change one variable at a time: re-selector or pin the app version, never both blindly.
- Keep [logs](/blog/ai-agent-logs-for-mobile-automation/) of device state — OS, WebView, service status — so the next update is a diff, not a mystery.

## FAQ

**Q: How do I know whether the Android update broke my script, or the app itself changed?**
A: Compare versions first: Android version and security patch date, WebView version, and the app's version plus last-updated date. If the system changed inside the failure window and the app did not, check the accessibility service and permissions before rewriting anything.

**Q: Why does AutoJS need the accessibility service, and why would an update turn it off?**
A: AutoJS uses an accessibility service to read the UI hierarchy and simulate touches. Accessibility services are opt-in and can be disabled, and some device images reset accessibility toggles during or after an update. Re-enable it under Settings → Accessibility and re-run the script before touching script logic.

**Q: How do I check the WebView version on a cloud phone?**
A: Usually via Settings → Apps → Android System WebView, where app info shows the version. Compare it against the version at your last known-good run, which you should record in notes or logs. Because WebView updates independently of the OS, an unchanged Android version does not mean an unchanged rendering engine.

**Q: Should I pin the app version or rewrite my selectors?**
A: Test on one phone first. If selectors are stale against the new engine, update them. If you need fleet-wide stability, pin the automated app's version where your platform supports it — but pinning the app does not stop OS or WebView updates. Do one change at a time and re-test on the same device.

## Sources

- Android developer documentation — Accessibility services (AccessibilityService): https://developer.android.com/guide/topics/ui/accessibility/service
- Google Play Console Help — Target API level requirements: https://support.google.com/googleplay/android-developer/answer/9888077
- Android 14 behavior changes for apps targeting Android 14: https://developer.android.com/about/versions/14/behavior-changes-14
- Android 13 runtime notification permission: https://developer.android.com/about/versions/13/changes/notification-permission
- OWASP Mobile Application Security Verification Standard (MASVS): https://mas.owasp.org/MASVS/
