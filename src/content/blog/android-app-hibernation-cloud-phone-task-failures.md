---
title: "Android App Hibernation: Why Cloud Phone Tasks Fail Quietly After Idle"
description: Android app hibernation — Doze, App Standby, and auto-reset
  permissions — can break cloud phone scripts after idle. Learn the failure
  signature and how to confirm it before rewriting code that was never broken.
pubDate: 2026-08-02
updatedDate: 2026-08-02
---

## Answer First

**Definition:** Android app hibernation is the umbrella term for the platform's power- and storage-management features that progressively restrict apps the system considers idle or unused: Doze mode and App Standby (Android 6+), auto-reset of permissions for unused apps (Android 11+), and hibernation states including partial hibernation (Android 15+). An app in one of these states loses background CPU, network, alarms, and jobs — and can lose its granted permissions entirely. The app is still installed and opens normally; it is simply asleep, and scheduled work silently stops reaching it.

**Why:** When a cloud phone task fails, the default investigation blames the script or the app UI. But a managed Android fleet is a battery-conservation system first: the platform is explicitly designed to throttle apps that sit unused between task windows. Night windows, staggered schedules, and long idle gaps are exactly the conditions Doze, App Standby, and permission auto-reset are built to catch. Operators who don't recognize the failure signature rewrite working scripts, reinstall working apps, and burn days chasing a bug that lives in the OS, not in the code. This is a mobile-operations-layer problem, not a code problem — the same distinction that matters when agents fail mid-task in any unattended workflow ([AI Agents Need Brakes: What Controlled Takeover Means for Cloud Phone Automation](/blog/ai-agent-control-boundaries-cloud-phone-takeover/)).

**Example:** A fleet of cloud phones runs an account-sync app at 02:00 nightly. For months it works; then jobs start dying with generic timeouts. The script is byte-identical. The app UI opens fine when an operator looks at it. But `adb shell dumpsys deviceidle` shows the device has been in Doze since 01:40, `am get-standby-bucket` returns `rare`, and `dumpsys package` shows the sync permission auto-reset to denied. The script never broke. Android put the app to sleep — and every manual check woke it back up, which is exactly why the failure only appears in unattended windows.

## Key Facts

| Mechanism | Introduced | Trigger | What it does to your app |
|---|---|---|---|
| Doze mode | Android 6.0 | Screen off, device unplugged and stationary for a period | Defers background CPU, network, syncs, jobs, and alarms; work stalls until a maintenance window or user interaction |
| App Standby | Android 6.0 (buckets refined in Android 9) | App not used for a stretch of time | App is placed in a standby bucket — active, working set, frequent, or rare; background network and jobs are deferred or heavily restricted as the app sits unused |
| Auto-reset permissions | Android 11 | App unused for several months and targets Android 11+ | All runtime permissions reset to denied with no prompt; the task fails on the first call that needs a permission you assumed was granted |
| Hibernation / partial hibernation | Android 15 | App unused for months | App enters a restricted state with permissions revoked and background activity limited; on some devices, unused apps may also be auto-archived, which clears their stored data |

- Doze and App Standby target precisely the pattern your scheduled tasks depend on: an app doing nothing while the device sits idle between task windows.
- An app in the `rare` standby bucket can go long stretches without background network access; an app under deep Doze cannot rely on alarms, syncs, or open sockets to wake it.
- Android 11's auto-reset flips permission grants to denied with no user action and no error surfaced to your script — the failure looks like an app bug or a revoked credential.
- Android 15's partial hibernation is a battery-life push: apps that stay unused get permissions revoked and background restrictions applied without being uninstalled.
- All of this is default, documented platform behavior. None of it is a malfunction, and none of it shows up in your script's error handling: the app doesn't crash, it just stops being scheduled.

## Expert Explanation

There are three causes of a "script broke" report: the script, the app or its environment, and the platform's power management. The first two are visible to every debugging tool you already have — diffs, logs, UI inspection. The third is invisible because nothing throws. The process simply isn't given the resources to run. When your fleet runs the same script for months and then fails overnight, the platform layer deserves a check before a rewrite.

**How the mechanisms stack across versions.** Doze (Android 6+) engages when the device is unplugged, still, and the screen is off: the system enters light, then deep, Doze, deferring network access and batching alarms into maintenance windows. Foreground work continues; background work waits. Apps that must run while the device sleeps need an exemption — the battery-optimization "Unrestricted" setting or a device-owner configuration. App Standby applies the same philosophy per app: the less you interact with an app, the lower its bucket, and the less background network and job access it gets. A cloud phone that is only touched during scheduled windows is precisely the profile the system classifies as `rare`. Android 11 then adds auto-reset: an app targeting Android 11+ that goes unused for months has its runtime permissions silently reset to denied — no prompt, no notification that survives to your monitoring. Android 15's partial hibernation extends the idea: unused apps enter a hibernating state with permissions revoked and background activity restricted, and restoring them requires real interaction, not a scripted retry.

**Why the failure signature is so easy to miss.** The defining property of this failure class is that it only exists while the device stays idle. The moment anyone — human or automated — touches the app, it leaves the throttled state: Doze backs off, the standby bucket rises, permission checks re-run. So the standard verification workflow, "open the app and see what happens," is the one workflow guaranteed to make the bug disappear. The failure signature to look for instead: failures cluster at the first scheduled window after a long idle stretch; they repeat across unrelated scripts on the same device; they clear after any interaction; and the error itself is environmental — timeout, connection reset, "no network," or a permission denial at the first privileged call — rather than a logic error in the script.

**Reading the timing pattern.** Map failure time against last interaction, not against deployment time. If failures begin only after the device has been idle for hours, or after the app has been untouched for weeks, the script is not the variable. The device-side state at failure time — `dumpsys deviceidle` state, standby bucket, permission grants — is the evidence that distinguishes a sleeping app from a broken one, and it is the record your monitoring should be capturing, because logs without device power state cannot tell the two apart ([AI Agents Need Logs. Mobile Automation Needs Them Even More](/blog/ai-agent-logs-for-mobile-automation/)).

## Decision Framework

Run this checklist before you rewrite or reinstall anything:

- [ ] Diff or hash the script — confirm zero changes since it last worked.
- [ ] Check the app's version and targetSdk — a silent update can change behavior.
- [ ] Device state at failure time: `adb shell dumpsys deviceidle` — was the device in `IDLE` or `IDLE_MAINTENANCE` when the job ran?
- [ ] Standby bucket: `adb shell am get-standby-bucket <package>` — `active`, `working_set`, `frequent`, or `rare`?
- [ ] Battery exemption: `adb shell dumpsys deviceidle whitelist` — is the app "Unrestricted" or sitting in the optimized list?
- [ ] Permission state: `adb shell dumpsys package <package>` — compare granted permissions against what the task needs; look for auto-reset denials.
- [ ] Hibernation state: check `dumpsys package` for hibernating/archived flags on Android 15 devices.
- [ ] Timing pattern: do failures correlate with idle duration or last interaction rather than with code changes?

If any of the last five checks show throttling, the script is the symptom, not the cause. The fixes are operational, not code: run a scheduled "wake pass" that simulates real interaction ahead of each task window so grants stay fresh and buckets stay high; whitelist the app from battery optimization where policy allows; re-grant permissions on a cadence; and schedule work inside windows where the device is already active. None of this removes the need to track what your agents were authorized to do and what they actually touched — permission hygiene and audit trails still carry the operational weight ([AI Agents Need Permissions and Audit Trails. Mobile Automation Needs Them Too](/blog/ai-agent-permissions-audit-trails-cloud-phone/)), and a failed mid-task agent still needs a defined recovery path ([AI Agent Failed Mid-Task. What Happens Next?](/blog/ai-agent-fails-what-happens-next/)).

**Practical limits.** These checks require adb access to the device. OEM skins (MIUI, ColorOS, EMUI, One UI) layer their own aggressive battery managers on top of stock Android, so AOSP checks don't tell the whole story. No state here can be permanently disabled without root or device-owner privileges. Battery exemptions are per-device settings that scripts cannot silently change. Auto-reset and hibernation cannot be fully reversed from the command line — they need real interaction or reinstallation. And behavior varies by Android version, so verify on the exact image your fleet runs, not on a test device that is never idle.

## Key Takeaways

- Android app hibernation is a real, default, documented failure cause for idle fleets — check it before rewriting scripts.
- The signature: silent, environment-style failures at scheduled windows after long idle; manual checks pass because touching the device wakes it.
- Three mechanisms to verify: Doze/App Standby state, permission grants (Android 11 auto-reset), and hibernation (Android 15).
- Confirm with device-side state at failure time — `dumpsys deviceidle`, `am get-standby-bucket`, `dumpsys package` — not by re-running the task from the UI.
- Mitigations are operational: wake passes, battery exemptions where policy allows, permission re-grant cadence, and scheduling inside active windows.
- Capture device power state in your logs; without it, you can't tell a broken script from a sleeping app — and the next investigation starts from zero ([AI Agents Are Becoming Apps. But Who Handles the Mobile Operations Layer?](/blog/agentic-apps-need-mobile-operations-layer/)).

## FAQ

Q: What is the difference between Doze and Android app hibernation?

A: Doze (Android 6+) is a device-level battery mode: when the device is idle, unplugged, and stationary, background CPU, network, alarms, and jobs are deferred for apps. App hibernation (Android 15+) is app-level: an app that stays unused for months gets its permissions revoked and background restrictions applied until someone interacts with it. Doze throttles activity; hibernation revokes standing capabilities.

Q: If I set my app to "Unrestricted" battery mode, can it still fail?

A: Yes. The battery-optimization exemption addresses Doze and App Standby deferral on that one device — but it does not stop Android 11 auto-reset of permissions for unused apps, and it does not prevent Android 15 hibernation states. Exemptions are also a per-device setting that must be applied on every device in the fleet, and they can't be silently changed by scripts.

Q: My manual test passes — does that prove the app is fine?

A: No, and this is the trap. Interacting with the app (even opening it) wakes it out of Doze, raises its standby bucket, and re-triggers permission checks, so a manual check masks the exact state that failed at 02:00. Inspect device-side state while the device is still idle, before touching the app.

Q: How do I tell a script bug from hibernation from a network outage?

A: Look at the pattern. Hibernation failures cluster at scheduled windows after long idle, repeat across different scripts on the same device, and clear after any interaction. Script bugs are consistent regardless of idle time. Network outages are broader and correlate with external events. Then confirm with device state — standby bucket, deviceidle state, and permission grants — captured at failure time, not after you have woken the device.

## Sources

- Android Developers — Optimize for Doze and App Standby: https://developer.android.com/training/monitoring-device-state/doze-standby
- Android Developers — Auto-reset permissions for unused apps (Android 11 behavior change): https://developer.android.com/about/versions/11/privacy/auto-reset
- Android Developers — App hibernation: https://developer.android.com/topic/performance/app-hibernation
- Android Developers — Android 15 behavior changes (partial hibernation): https://developer.android.com/about/versions/15/behavior-changes-15

---

Note for editorial review: source URLs above were selected from long-standing Android documentation whose scope matches the claims, but live verification was not possible in this environment (network egress to developer.android.com timed out on repeated attempts, direct and proxied). Recommend a one-pass link check before publish. The configured optional URLs (developer.android.com/privacy-and-security/risks, mas.owasp.org/MASVS, and the Google Play support answer) were reviewed and deliberately omitted: their documented scope does not support the power-management claims adjacent to them.
