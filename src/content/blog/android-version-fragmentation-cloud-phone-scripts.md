---
title: "Why Your Cloud Phone Script Works on Some Devices but Not Others:
  Android Version Fragmentation"
description: "Android version fragmentation automation scripts fail across cloud
  phone fleets for OS-level reasons: WebView drift, accessibility API changes,
  Doze/App Standby, gesture vs 3-button navigation, scoped storage, and per-app
  language. Here's the version-drift checklist to run before blaming the
  script."
pubDate: 2026-08-09
updatedDate: 2026-08-09
---

## Answer First

**Definition:** Android version fragmentation is the OS-level divergence between devices in the same fleet — different API levels, security patch levels, updatable components like WebView, battery policies, navigation modes, storage models, and locale resolution — that makes the identical automation script behave differently on each device. It is the reason a script that runs cleanly on one cloud phone fails on another with the same app installed.

**Why:** A script never automates "the app" in the abstract. It automates the app as rendered by one specific device: one WebView engine, one accessibility tree, one battery policy, one navigation bar. Every selector, coordinate, wait, and file path encodes an implicit assumption about that device's OS layer. When the device changes — a different Android version, a WebView that never updated, gesture navigation instead of three buttons — the assumption breaks and the script fails even though its logic never changed. Any operator running android version fragmentation automation scripts hits this wall constantly: the failure looks like a code bug, but it is a platform-contract change. The older the fleet and the newer the Android rollout, the wider the gap gets — and Android 15's rollout alongside decoupled WebView updates is widening it right now.

**Example:** A script opens a WebView-based sign-in page and taps "Continue" 400 ms after the page signals ready. On the test device, WebView is Chromium 126, the button renders where it always has, and the tap lands. On a fleet device whose ROM shipped Chromium 90 and has no Google Play channel to update it, the same page renders differently, the button sits 60 px lower, and the tap hits the footer link. The script did exactly what it was told on both devices — one of them was simply answering to a different platform.

## Key Facts

- **WebView is decoupled from the OS.** Since Android 5.0, WebView updates through Google Play on Play-enabled devices and is frozen at the ROM version on devices without it — so two devices on the same Android version can run wildly different web-rendering engines.
- **Accessibility semantics keep changing.** Each major release adjusts the accessibility tree, node actions, and service requirements, and Google Play now polices what accessibility services may do — the surface that selector- and semantics-based automation depends on.
- **Doze and App Standby make background work conditional.** Battery policy tightened from Android 6.0 through App Standby buckets in Android 9 and the user-revocable `SCHEDULE_EXACT_ALARM` permission in Android 12; OEM ROMs add their own aggressive process killers on top.
- **Gesture navigation replaced the 3-button bar.** Gesture nav became the default in Android 10, so scripts that tap fixed coordinates for Home/Back/Recents miss on devices with no nav bar.
- **Scoped storage removed global file access.** Android 10 made it opt-out, Android 11 enforced it, Android 13 split media into granular permissions, and Android 14 added the photo picker — legacy paths and broad storage reads silently stop working.
- **Per-app language split locale from the device.** Android 13 added per-app language preferences, so an app can resolve a different language than the device setting, breaking text-based selectors.
- **This is an OS-layer problem, not an app-UI or environment problem.** App-UI changes and environment-class causes (network, instance state) are separate failure families; this article covers only the six OS-level differences above.

## Expert Explanation

### 1. WebView: the OS component that isn't tied to the OS

WebView is the single most common hidden variable in cloud phone fleets. Because it ships as a Play-updatable component rather than part of the OS image, its version has no fixed relationship to the Android version beside it. A Play-enabled device updates WebView in the background; a device without Play services keeps the engine that shipped in its ROM — sometimes years old, sometimes a different Chromium baseline entirely. Your "wait for element" turns into a race against a rendering engine you never pinned down, and [scripts written for browser-like flows](https://www.qccbot.com/blog/ai-agents-can-use-browsers-what-about-phone-apps/) are exactly the ones that break first. Check with `adb shell dumpsys webviewupdate` to see the installed engine and whether updates are enabled, and treat WebView version as part of your device fingerprint, not a detail.

### 2. Accessibility APIs: the automation surface that keeps changing

Selector-based automation leans on the accessibility tree, and that tree is not stable across versions. Node semantics, focus behavior, and service declaration rules have shifted repeatedly — and as of recent releases, Google Play requires accessibility services to declare their purpose and enforce compliance, which affects what automation providers are even allowed to ship. On top of that, OEM ROMs modify accessibility behavior in ways AOSP does not. The result: a `byText` or `byResourceId` lookup that resolves on one device and returns nothing on another, and a permission that is sensitive enough that [permissions and audit trails](https://www.qccbot.com/blog/ai-agent-permissions-audit-trails-cloud-phone/) are not a nice-to-have but the actual operating requirement. The practical limit: you cannot make an app expose semantics it does not expose; you can only detect the missing node and fail loudly.

### 3. Doze and App Standby: background work is conditional

Scheduled steps are where fragmentation bites hardest. Doze (Android 6.0), App Standby, deep doze refinements, App Standby buckets (Android 9), and the user-revocable `SCHEDULE_EXACT_ALARM` permission (Android 12) each made background execution more conditional. OEM ROMs — many of which are the base of cheap cloud phone images — add aggressive freeze and kill behavior on top. A script that "always" ran at 3 a.m. on the dev device may never fire on a fleet device whose app is in a standby bucket and not whitelisted from battery optimization. There is no script-side fix that forces a frozen app to run; the fix is device configuration (whitelist, exact-alarm grant) plus monitoring that flags the silent miss. When a task dies mid-run instead, the question is not "what broke the step" but [what happens next](https://www.qccbot.com/blog/ai-agent-fails-what-happens-next/) — which is why failure detection belongs in the platform, not in the script's happy path.

### 4. Gesture vs 3-button navigation

Coordinate-based scripts assume a physical reality: a back button at x, a home button at y. Gesture navigation (default since Android 10) removed those coordinates, and edge-to-edge enforcement in newer releases pushes app content under the system bars, shifting every tap target. A script that swipes from the left edge to open a drawer instead triggers the system back gesture; a script that taps where Home used to be taps nothing. The defensive pattern is navigation-agnostic actions — key events like `KEYCODE_BACK` instead of coordinates — and reading the actual navigation mode (`Settings.Global.NAVIGATION_MODE`) at start rather than assuming it.

### 5. Scoped storage

Scripts that touch shared storage — screenshots, exported files, media uploads — assume a file model that has been dismantled piece by piece. Android 10 allowed opting out, Android 11 enforced scoped storage, Android 13 replaced `READ_EXTERNAL_STORAGE` with granular `READ_MEDIA_*` permissions, and Android 14 added the user-mediated photo picker. Paths like `/sdcard/...` that worked on one image silently fail or return nothing on another. The fix is to use app-private directories or the `MediaStore` APIs and to stop hardcoding legacy paths — and to accept that on some API levels, the "right" API barely exists, which is a genuine practical limit, not a code smell.

### 6. Per-app language (Android 13+)

Text-based selectors quietly assume the app speaks the same language as the script writer's test device. Android 13's per-app language preferences broke that assumption: an app can now resolve a locale independent of the device setting, and the `LOCALE_CHANGED` broadcast is no longer sent for locale changes. The same script expecting "Continue" can match "Continue" on one device and miss on a device where the app resolved to another locale. The robust habit is preferring resource-agnostic selectors — IDs, semantics, positions relative to stable anchors — and treating locale as part of the device fingerprint, not a given.

## Decision Framework

Run this version-drift checklist **before** debugging the script. If any box fails on the failing device and passes on the working device, you have drift, not a bug.

- [ ] **Record a device fingerprint at provisioning:** Android version (API level), security patch month, WebView version, accessibility-service state, navigation mode, battery-optimization whitelist status, per-app locale.
- [ ] **Diff the failing device against the device where the script passed** — fingerprint first, script second.
- [ ] **Check WebView:** `adb shell dumpsys webviewupdate` — version, current provider, updates enabled?
- [ ] **Check battery policy:** is the automation app whitelisted from battery optimization? Is `SCHEDULE_EXACT_ALARM` granted (Android 12+)?
- [ ] **Check navigation:** gesture or 3-button? Does the script contain coordinate taps for system buttons?
- [ ] **Check storage:** does the script read or write shared paths? What API level is the device?
- [ ] **Check locale:** what language does the app actually resolve, vs the device setting and vs your selectors?
- [ ] **Check accessibility:** is the service enabled, and does the target app expose the nodes your selectors need?
- [ ] **Reproduce on a matched fingerprint**, then decide: config fix (whitelist, alarm grant, WebView image, locale) or script fix.
- [ ] **Log the fingerprint with every run** so the next failure is diagnosed from history, not from scratch — this is exactly the kind of signal [mobile automation logging](https://www.qccbot.com/blog/ai-agent-logs-for-mobile-automation/) exists to preserve.

For operations teams running fleets of mixed images, the drift checklist is a control-plane concern: knowing which devices carry which platform contract is what separates [a control tower for mobile workflows](https://www.qccbot.com/blog/ai-agent-control-tower-for-mobile-app-workflows/) from a pile of flaky runs.

## Key Takeaways

- A failing script is usually a correct script running against a different platform contract — check the OS layer before touching the code.
- Six OS-level differences explain most cross-device failures: WebView version, accessibility semantics, Doze/App Standby, gesture vs 3-button navigation, scoped storage, and per-app language.
- Fingerprint every device at provisioning and diff fingerprints on failure; never debug the script without the fingerprint in hand.
- Prefer configuration fixes (battery whitelist, exact-alarm grant, WebView-capable image, locale pinning) over script hacks — script workarounds break again at the next update.
- Treat accessibility as a sensitive, audited permission: it is the most powerful surface your automation touches, and [keeping cloud phone account work under control](https://www.qccbot.com/blog/agentic-automation-security-cloud-phone-accounts/) starts with knowing what that surface can reach.
- **Practical limits:** no script can force a frozen app to run, cannot make an app expose semantics it hides, and cannot outrun an OEM ROM that diverges from AOSP. The honest engineering answer is defensive selectors, config discipline, and monitoring that turns a "flaky script" into a known drift pattern.

## FAQ

**Q: Why does the same script pass on one cloud phone and fail on another running the same app version?**
**A:** Because the script automates the app as rendered by one device's OS layer, not the app in the abstract. The six differences that bite most often are WebView engine version, accessibility API semantics, Doze/App Standby battery policy, gesture vs 3-button navigation, scoped storage enforcement, and per-app language resolution — all of which change the environment your selectors, coordinates, waits, and file paths are written against.

**Q: Do I need to update WebView on every device in the fleet?**
**A:** On devices with Google Play and updates enabled, WebView updates arrive automatically. On devices without Play — common on many cloud phone images — WebView is whatever the ROM shipped and may be years old. Check with `adb shell dumpsys webviewupdate` and either update the device image or treat the older engine as a known constraint your selectors must tolerate. You cannot force an update on a device with no channel for one.

**Q: Is accessibility-based automation a security or policy risk?**
**A:** Accessibility services are one of the most powerful Android APIs, which is why Google Play now requires apps that use them to declare their purpose and comply with its accessibility-service policy. Operationally, treat the accessibility permission as sensitive: enable it only where needed, keep its use scoped and audited, and review what your automation can touch — without assuming any platform guarantee about acceptance.

**Q: When a device fails, should I fix the script or the device?**
**A:** Run the version-drift checklist first. If the failure tracks config drift — battery optimization not whitelisted, missing `SCHEDULE_EXACT_ALARM`, gesture mode, per-app locale, stale WebView — fix the device config, because patching the script around the wrong config will break again at the next update. If the OS changed an API's semantics in a way no config controls, adapt the script defensively. In practice, most fleet failures are config; most single-device failures are script.

## Sources

- Android Developers — *Optimize for Doze and App Standby*: https://developer.android.com/training/monitoring-device-state/doze-standby
- Android Developers — *Schedule repeating alarms* (exact alarms and `SCHEDULE_EXACT_ALARM`): https://developer.android.com/develop/background-work/services/alarms/schedule
- Android Developers — *WebView* (independent, updatable component and versioning): https://developer.android.com/develop/ui/views/webview
- Android Developers — *Accessibility overview* (accessibility services and API surface): https://developer.android.com/guide/topics/ui/accessibility
- Android Developers — *Per-app language preferences*: https://developer.android.com/guide/topics/resources/app-languages
- Android Developers — *Data and file storage overview* (scoped storage): https://developer.android.com/training/data-storage
- Android Developers — *Android 11 behavior changes: storage*: https://developer.android.com/about/versions/11/privacy/storage

## Further Reading

- [AI Agents Are Becoming Apps. But Who Handles the Mobile Operations Layer?](/blog/agentic-apps-need-mobile-operations-layer/)
- [Agentic Automation Security: How to Keep Cloud Phone Account Work Under Control](/blog/agentic-automation-security-cloud-phone-accounts/)
- [AI Agents Need Brakes: What Controlled Takeover Means for Cloud Phone Automation](/blog/ai-agent-control-boundaries-cloud-phone-takeover/)
