---
title: "Why Cloud Phone Tasks Stop Running on Their Own: Android Background
  Execution Limits"
description: Scheduled cloud phone tasks can stop running silently when Android
  background execution limits block them before they start. How Doze, App
  Standby, OEM battery whitelists, autostart, and screen-lock settings kill
  tasks — plus a fleet-wide settings checklist to fix the configuration.
pubDate: 2026-08-04
updatedDate: 2026-08-04
---

## Answer First

**Definition:** A cloud phone task that "stops running on its own" is usually not a task failure at all. Android's background execution limits — Doze mode, App Standby, OEM battery-optimization whitelists, the autostart permission, and screen-lock or keep-awake settings — can prevent a scheduled task from ever starting. The task doesn't crash, error, or exit. It simply never begins, which is why it looks like it stopped by itself.

**Why:** These limits exist to save battery on consumer phones. Android and manufacturers defer or kill background work when a device is idle, unplugged, screen-off, or "unused" — sensible for a phone in a pocket, destructive for an unattended fleet of cloud phones running scripts around the clock. And the mechanism is silent by design: no process starts, no log is written, no exit code is produced. The failure is invisible until a human notices the schedule went quiet.

**Example:** A team schedules an account-health check for 02:00. The device has been idle with the screen off for an hour, entered Doze, and the system defers the job's alarm for hours. The dashboard shows "no run" with zero logs: no error, no exit code. The script is fine — background execution limits stopped it before the first line of code could run.

That distinction matters. A task that starts and fails mid-run is a script or environment problem — we covered what happens next in [AI Agent Failed Mid-Task. What Happens Next?](/blog/ai-agent-fails-what-happens-next/). A task that produces no logs at all is almost always a configuration problem, fixable once at provisioning instead of being papered over with monitoring.

## Key Facts

- Android restricts background execution at several layers; the failure signature is missing logs, not error logs.
- Doze defers jobs, alarms, and network while the device is idle, unplugged, and stationary with the screen off.
- App Standby buckets apps by usage — active, working set, frequent, rare — and throttles background work in lower buckets.
- Stock Android is only the baseline: major OEMs (Xiaomi, Huawei, OPPO, vivo, Samsung) layer on battery managers with "autostart" and "protected apps" whitelists that are often more aggressive than Doze.
- The autostart permission is often off by default; without it, an app never receives boot-time or scheduled start signals.
- Screen-lock and keep-awake settings decide whether the device sleeps at all; the same misconfiguration repeats across a fleet unless fixed once at provisioning.

## Expert Explanation

### The silent-kill failure class

Every scheduled task passes through a gate: the OS decides whether the app may start background work. When the answer is "no," nothing happens and nothing is recorded — the defining symptom of the silent-kill class, and the opposite of a mid-run failure, which leaves a log trail you can follow ([AI Agents Need Logs. Mobile Automation Needs Them Even More](/blog/ai-agent-logs-for-mobile-automation/)). Whether an agent drives a browser or a phone app, the device it runs on has to stay awake and be permitted to run it ([AI Agents Can Use Browsers. What About Phone Apps?](/blog/ai-agents-can-use-browsers-what-about-phone-apps/)).

### Doze mode and App Standby

Doze is Android's device-level sleep: when the device is unplugged, stationary, and screen-off for a while, Android defers background CPU, network, and alarm delivery, waking only for brief maintenance windows. Inexact alarms and scheduled jobs are pushed back; even exact alarms are throttled. A task scheduled for 02:00 on a device idle since 01:00 simply doesn't fire at 02:00.

App Standby is the per-app companion: the system buckets every app by how recently and often it was used — active, working set, frequent, rare — and limits background work for lower-bucket apps. An automation app that runs in the background and is rarely "used" by a human can drift into the rare bucket and get throttled to a trickle. Check the bucket with `adb shell am get-standby-bucket <package>`; exemption is a per-device decision.

### OEM battery-optimization whitelists

Stock Android is the floor, not the ceiling. Android background execution limits affect cloud phone tasks differently from device to device, and the OEM layer is where the most aggressive and least documented limits live. Chinese and Korean OEMs in particular ship their own power managers: MIUI's autostart and battery saver, EMUI's "protected apps," ColorOS and realme UI's background-activity controls, OriginOS, and One UI's "sleeping apps" list. These layers can block work where stock Android would allow it — including, on some firmware, right after the screen locks. Two identical scripts can behave completely differently across devices, and no single command fixes every one.

### The autostart permission

Separate from battery optimization, most OEMs gate automatic app starts behind an off-by-default "autostart" permission. Without it, an app may never respond to boot events or background start signals. Granting battery-unrestricted status but not autostart — or vice versa — leaves a hole in the configuration. On some firmware the grant resets after a system update or a "cleanup," so treat it as a re-check item. Grant hygiene matters as much here as for account access ([AI Agents Need Permissions and Audit Trails. Mobile Automation Needs Them Too](/blog/ai-agent-permissions-audit-trails-cloud-phone/)).

### Screen-lock and keep-awake settings

Background limits engage only when the device is actually idle, and "idle" starts with the screen off. Screen timeout, lock-screen security, "stay awake while charging," and OEM "screen-lock cleanup" settings decide whether a device sleeps — and whether Doze, App Standby, and OEM killers get a chance to act. A headless cloud phone still turns its screen off and idles; the same limits apply.

### Why this is a configuration problem, not a monitoring problem

It's tempting to treat silent non-runs as a visibility gap and build watchdogs. Monitoring still has a role — verifying configuration and catching post-start failures — but it's the wrong first fix: a task that never starts has no log to watch. The fix is the checklist below, applied once per device model at provisioning and re-checked after updates. On QCCBot, where managed Android devices, scripted task execution, monitoring, and human review are the operating model, device configuration is the foundation: a task that can't start is a fleet problem, not a workflow problem. Control over device state is part of the operations layer teams need ([AI Agent Control Tower for Mobile App Workflows: What Operations Teams Actually Need](/blog/ai-agent-control-tower-for-mobile-app-workflows/)).

## Decision Framework

The first question is always: did the task produce any logs at all?

- **No logs at all** → silent-kill class → run the settings checklist; don't touch the script.
- **Logs start, then stop mid-run** → a genuine failure → treat it as a script or environment problem (see [AI Agent Failed Mid-Task. What Happens Next?](/blog/ai-agent-fails-what-happens-next/)).
- **Full logs but wrong results** → a logic or data problem → review the run; don't reconfigure the device.

Then, for any device that must run scheduled work, apply this checklist once at provisioning:

| # | Setting | Where to check | Why it matters |
|---|---------|----------------|----------------|
| 1 | Battery optimization / "unrestricted" | Settings → Apps → Battery (per app) | Stops Doze/App Standby deferring the app's jobs |
| 2 | App Standby bucket | `adb shell am get-standby-bucket <package>` | Confirms the app isn't in the "rare" bucket |
| 3 | Autostart / auto-launch permission | OEM settings (MIUI, EMUI, ColorOS, OriginOS, One UI) | Allows boot-time and scheduled starts; off by default |
| 4 | Protected / background whitelist | OEM battery manager or security app | Stops the OEM layer killing work on screen lock |
| 5 | Battery saver | Settings → Battery | Disables system-wide background throttling |
| 6 | Screen timeout / stay awake while charging | Display settings + developer options | Controls whether the device enters the idle state |
| 7 | Lock-screen cleanup | OEM settings | Stops firmware cleaning the app on screen lock |

**Practical limits:** this is a list of things to verify, not a promise that every device will comply. OEM setting names and behavior vary by brand, model, and firmware; grants can reset after updates; and each major Android version adds restrictions (Android 14 is a recent example). Emulators, cloud devices, and physical hardware behave differently — validate on your fleet. No universal switch exists, and no tool can override a platform-enforced restriction.

## Key Takeaways

- "Stopped running on its own" usually means the task never started — a missing-log signature is the tell.
- Five layers can block a task before it runs: Doze, App Standby, OEM whitelists, autostart, and screen-lock/keep-awake settings.
- Stock Android behavior is only the baseline; OEM battery managers are more aggressive and less documented.
- Fix silent non-runs at the configuration layer, per device model, at provisioning — don't build monitoring to detect a config problem you could eliminate.
- Re-check configuration after system updates; OEMs can silently reset whitelist and autostart grants.
- Keep monitoring for verifying configuration and catching post-start failures.

## FAQ

**Q: Why does my scheduled cloud phone task never run but show no error?**

A: Because it never started. Android's background execution limits — or the OEM layer — decided not to launch the app at that moment. No process starts, so there is nothing to report: no error, no exit code, no log. The absence of logs is the diagnostic signature of the silent-kill failure class.

**Q: Do Doze and App Standby affect devices that are plugged in or running in a data center?**

A: Doze's deep stage generally requires the device to be unplugged, stationary, and screen-off. But App Standby buckets are based on app usage rather than charging, OEM battery managers often act regardless of charging state, and battery saver can engage on its own at a configured threshold. Cloud phones are not automatically immune — verify on your fleet instead of assuming AC power is a shield.

**Q: Will granting "unrestricted battery" and enabling autostart fix everything?**

A: Not by themselves. They clear the Doze/App Standby and autostart layers on that device, but lock-screen cleanup, OEM "protected apps" lists, battery saver, and newer Android-version restrictions can still block work. The grants are per app, per device, and can reset after updates — treat them as part of a checklist, not a one-time switch.

**Q: How do I check which background limits are actually affecting a device?**

A: Combine adb checks (`am get-standby-bucket`, `dumpsys deviceidle`, `dumpsys battery`) with the OEM settings screens, then run a minimal scheduled probe task that writes a single log line. If the probe produces no log, the configuration is the problem; if it logs, the configuration is fine and the issue is elsewhere.

## Sources

- Android Developers — Optimize for Doze and App Standby: https://developer.android.com/training/monitoring-device-state/doze-standby
- Android Developers — Schedule repeating alarms: https://developer.android.com/training/schedule-alarms
- Android Developers — Behavior changes: Apps targeting Android 14 (background optimization): https://developer.android.com/about/versions/14/changes/background-optimization
- AOSP — Battery Saver: https://source.android.com/docs/core/power/battery_saver
