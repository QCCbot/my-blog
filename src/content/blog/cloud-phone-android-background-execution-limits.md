---
title: Why Do Android Tasks Stop in the Background? Doze, App Standby, and
  Force-Stop Explained for Cloud Phone Teams
description: Doze, App Standby, force-stop, and other Android background
  execution limits silently kill cloud phone tasks. Learn to detect each state
  and what monitoring should flag.
pubDate: 2026-08-07
updatedDate: 2026-08-07
---

## Answer First

**Definition:** Android background execution limits are the operating system's rules for what an app may do when it is not on screen — background service limits (Android 8.0+), Doze, App Standby, the force-stop state, and app hibernation. They apply to every Android device, including cloud phones, no matter what your automation script asked for.

**Why:** When a task silently fails or a script can't relaunch an app, operators usually blame the script — the selector, the timeout, the network. But a distinct, recurring cause sits one layer down: the OS itself refuses to run the app's work because the device is in a restricted state. Managed fleets are exactly where these states accumulate: screens left off, apps unused for weeks, packages force-stopped by an operator or a system "cleaner." A monitoring layer that only watches script logs will keep reporting "script failed" while the real cause never shows up in the app's logs at all. This is part of the mobile operations layer that automation inherits when [AI agents become apps](/blog/agentic-apps-need-mobile-operations-layer/).

**Example:** A worker schedules an in-app action for 02:00 and walks away. The screen is off, so Doze defers the job. The app hasn't been used in days, so its standby bucket has decayed to "rare" and its alarms are throttled further. The job fires late or not at all. The relaunch script then sends an intent to start the app — but the app was force-stopped earlier, so the process stays dead and the intent goes nowhere. The operator sees one entry: task failed.

## Key Facts

- Android 8.0 (API 26) introduced background execution limits: apps can no longer start arbitrary background services, and most implicit broadcasts are no longer delivered.
- Doze (Android 6.0+) defers network access, jobs, and alarms when the device is stationary, unplugged, and the screen is off; from Android 9, a lighter Doze state applies as soon as the screen is off.
- App Standby (Android 6.0+) and App Standby Buckets (Android 9) grade apps into active, working set, frequent, and rare buckets; the rarer the bucket, the harder the throttling of jobs, alarms, and network.
- Force-stop puts an app in a stopped state: the process is killed, alarms and jobs are cancelled, and broadcasts — including BOOT_COMPLETED after a reboot — are blocked until the app is launched again.
- App hibernation (Android 12+) revokes permissions and stops background work for apps that go unused for months.
- OEM ROMs (Xiaomi, OPPO/OnePlus, vivo, and others) layer their own aggressive background killers on top of stock Android behavior.
- Practical limit: all of these states can be detected, but most cannot be "fixed" from a script — force-stop persists until the app is launched, and Doze will not honor tasks that need constant network while the screen is off.

## Expert Explanation

Mechanism by mechanism: what each state is, how to detect it on a device, and what it does to scripts and task relaunches.

### Background service limits (Android 8.0+)

Since Android 8.0, an app targeting the current API level cannot simply start a background service: the system throws an exception unless the app runs a foreground service with a visible notification, and most implicit broadcasts no longer reach the app. Scripts that assume a service keeps running in the background fail on modern builds even when everything else is healthy.

Detect: check the app's target SDK (`adb shell dumpsys package <pkg>` shows targetSdk) and watch logcat for `IllegalStateException` or `ForegroundServiceStartNotAllowedException` around the failure time. Flag it as a service-start denial — not something more script retries will fix.

### Doze and screen-off behavior

Doze is the reason tasks that run perfectly with the screen on "randomly" fail at night. Deep Doze requires the device to be stationary, unplugged, and screen-off; then network access is deferred, wakelocks are ignored, and jobs and alarms wait for maintenance windows. On Android 9+, light Doze already engages when the screen is off. Fleets where screens are routinely left off sit in this state most of the day.

Detect: `adb shell dumpsys deviceidle` shows the current state (for example ACTIVE vs. IDLE) and whether the screen is on; `dumpsys power` shows wakefulness. Correlate the failure timestamp with screen-off. Flag it as "task scheduled during a screen-off/Doze window," not "app crashed."

### App Standby buckets

The system tracks how recently an app was used and grades it: active, working set, frequent, rare (Android 9+). Jobs and alarms are throttled progressively and network access is restricted, so an app that has not been opened in days may not run its scheduled work on time — or at all. This is fully silent: no error, no crash, just a job that never executes.

Detect: `adb shell am get-standby-bucket <pkg>` returns the current bucket. Flag bucket decay to frequent or rare for task-critical apps — it is a leading indicator of future silent failures.

### Force-stop

Force-stop is the strongest state. When a user or an OEM "battery cleaner" force-stops an app, the system kills the process, cancels its alarms and jobs, and refuses to deliver broadcasts to it — including BOOT_COMPLETED after a reboot. The app does not run again until it is launched. From a script's point of view this is the worst failure mode: relaunch attempts can appear to succeed while nothing actually starts. This is the state that breaks relaunch loops, and it is exactly what an [agent-failure runbook](/blog/ai-agent-fails-what-happens-next/) has to distinguish from a script bug.

Detect: `adb shell dumpsys package <pkg>` — look for the stopped flag on the app's user entry. On the device, Settings → Apps shows a disabled Force stop button for stopped apps. Treat force-stop as a human-review event; it is not recoverable by retrying the script.

### App hibernation (Android 12+)

If an app goes unused for months, the system may hibernate it: permissions are revoked and background work stops. Long-running accounts onboarded and then never touched are the natural candidates.

Detect: check the app's entry under Settings → Apps (some builds surface an "Unused apps" page); `dumpsys package` exposes the state on newer builds. Flag hibernated apps as offline until they are re-initialized.

### OEM background killers

Stock behavior is the baseline, not the ceiling. Xiaomi/HyperOS, ColorOS, OriginOS, and others add aggressive app-killing that can stop an app even when its standby bucket says active. Test matrices must include the actual device image you run, not just the Android version.

## Decision Framework

| State | How to detect on the device | What it does to scripts | What monitoring should flag |
|---|---|---|---|
| Background service limits | targetSdk + logcat `ForegroundServiceStartNotAllowedException` | Service start fails immediately | Service denied — check the app build, not the script |
| Doze / screen-off | `dumpsys deviceidle` state, `dumpsys power` | Jobs and alarms deferred to maintenance windows; network paused | Task scheduled during a screen-off window |
| Standby bucket | `am get-standby-bucket <pkg>` | Throttled or skipped jobs and alarms; restricted network | Bucket decay to frequent or rare |
| Force-stop | `dumpsys package <pkg>` stopped flag | Process stays dead; broadcasts and relaunches blocked until launch | Force-stop detected — human review required |
| Hibernation (12+) | Settings → Apps / `dumpsys package` | Permissions revoked; background work stopped | Hibernated app marked offline |
| OEM killer | Device-specific battery/app settings | Process killed outside stock rules | Unexplained process death on specific images |

Practical limits, stated honestly: a monitoring layer can read all of these states and correlate them with task outcomes — that is the fixable part. It cannot make force-stop disappear, cannot make Doze tolerate always-on network, and cannot undo an OEM kill without a manual launch. The job of the monitoring layer is to say which state caused the failure, so the operator stops debugging the script.

## Key Takeaways

- Android's background execution model — not the script — is a first-class cause of silent task failures on cloud phones.
- Diagnose by state first: screen-off/Doze, standby bucket, force-stop, service limits, hibernation — each has a cheap adb check.
- Force-stop and hibernation are the two states no relaunch script can defeat; route them to human review.
- Test on the real device image: OEM killers override stock rules.
- Correlate every task outcome with device state at the failure timestamp — a [task log alone is not enough](/blog/ai-agent-logs-for-mobile-automation/). Teams need a [control plane that surfaces device state next to task status](/blog/ai-agent-control-tower-for-mobile-app-workflows/), not another retry knob.

## FAQ

**Does swiping an app out of Recents equal force-stopping it?**

No. Swiping an app away removes it from the task list and typically kills its process, but it does not set the stopped state. A force-stopped app additionally has its alarms and jobs cancelled, receives no broadcasts, and stays dead until it is launched. Monitoring should treat the two as different events.

**Can a script relaunch a force-stopped app?**

Not reliably. The stopped state is cleared only when the app is launched again — an explicit activity start. Sending broadcasts, starting services, or scheduling jobs will not bring it back, and OEM builds vary. Treat force-stop as a human-review item: launch the app once, verify it, then resume automation.

**Why does a task run with the screen on but fail with the screen off?**

Because screen-off is the primary Doze trigger. With the screen off, Android defers network access, jobs, and alarms to maintenance windows, so scheduled work runs late or not at all. Check `dumpsys deviceidle` at the failure timestamp instead of assuming the app crashed.

**Are these limits the same on every Android device?**

No. Stock Android behavior is the baseline; Xiaomi, OPPO/OnePlus, vivo, and other OEMs add their own aggressive background management on top, and even stock limits change by API level. Always validate relaunch and scheduling behavior on the exact device images your fleet runs.

## Sources

- Android Developers — Optimize for Doze and App Standby: https://developer.android.com/training/monitoring-device-state/doze-standby
- Android Developers — Background work: Services (background execution limits): https://developer.android.com/develop/background-work/services
- Android Developers — Background execution limits (Android 8.0): https://developer.android.com/about/versions/oreo/background
- Android Developers — App hibernation: https://developer.android.com/topic/performance/app-hibernation
