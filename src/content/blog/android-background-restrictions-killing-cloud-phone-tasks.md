---
title: "Why Android Kills Background Tasks on Cloud Phones: Doze, Battery
  Optimization, and App Standby"
description: Why Android kills background tasks on cloud phones — Doze mode, App
  Standby buckets, battery restriction, and Android 15/16 cached-app freezing —
  plus a concrete check order when scripts stop after backgrounding.
pubDate: 2026-08-16
updatedDate: 2026-08-16
---

## Answer First

**Definition:** When operators report an "android background task killed" issue on a cloud phone, the task usually wasn't killed at all. Android's power-management stack — Doze mode, App Standby buckets, per-app battery restriction, and, on Android 15 and 16, cached-app freezing — pauses, defers, or terminates an app's background work under documented rules that apply to every app on the device. Telling which layer stopped your task is the difference between a five-minute fix and a redesign.

**Why:** Android treats a backgrounded app as low-value work: battery and memory come first, and your script's desire to keep running isn't part of the calculation. Cloud phones make this sharper in three ways. They run 24/7 with many apps competing for resources. They're often left idle with the screen off — precisely the condition Doze was built for. And newer Android versions keep tightening the rules: Android 15 freezes cached apps after roughly six hours in the background, and Android 16 extends the freeze to more devices and shortens the window on lower-RAM devices. So when you see "my task runs a while, then stops after the app goes to background," you're usually looking at one of four OS layers — or, just as often, at the platform's own "screen off" visibility pause, which isn't an OS event at all.

**Example:** You start a 45-minute AutoJS routine on a managed cloud phone and switch away. Twelve minutes in, the screen times out. The device is unplugged and idle, so Doze engages: timers are deferred, network is restricted, and execution is squeezed into brief maintenance windows. A few minutes later, the script stops. `adb shell dumpsys deviceidle` shows deep doze; logcat shows no crash. The task wasn't killed — it was throttled out of existence. On an Android 15 image, the same script that survives Doze still ends with its process frozen after about six hours in the background.

## Key Facts

- Doze mode engages when a device is stationary, unplugged, and screen-off for a period; it defers alarms, jobs, and syncs, restricts network access, and grants only brief, increasingly rare maintenance windows.
- App Standby Buckets rank apps by recency of use — active, working set, frequent, rare, restricted — and lower buckets receive progressively more throttling.
- Per-app battery restriction (usually Settings → Apps → Battery, though the path is device-specific) sets an app to Unrestricted, Optimized, or Restricted.
- Android 15 freezes cached apps after roughly six hours in the background; Android 16 extends the freeze to more devices and lowers the threshold to about four hours on devices with less than 8 GB of RAM.
- Even foreground services are time-boxed: Android 15 caps dataSync and mediaProcessing foreground services at six hours.
- "Killed," "frozen," and "visibility paused" are three different events — mixing them up is the most common misdiagnosis on cloud phone fleets.

## Expert Explanation

### The OS layers that end background work, in one table

| Layer | What triggers it | What actually happens | Where to look |
|---|---|---|---|
| Doze mode | Screen off, idle, stationary, unplugged | Timers, jobs, and syncs deferred; network restricted; short maintenance windows | `dumpsys deviceidle` |
| App Standby bucket | App unused for days or weeks | Lower buckets defer jobs and restrict network access | `am get-standby-bucket <pkg>` |
| Per-app battery restriction | Operator or user setting | "Restricted" stops background work; "Unrestricted" exempts from most throttling | Settings → Battery; `cmd appops get` |
| Cached-app freezing (Android 15/16) | App backgrounded for hours | Process frozen — no CPU, no network, no timers | logcat "Freezing process" |
| Screen-off visibility pause | Platform control, not OS | App visibility paused by the management layer | Platform console/logs |

### Doze mode

Doze is the big one because cloud phones meet its conditions constantly: screen off, unplugged, not moving. The system conserves battery by restricting apps' access to network and CPU-intensive services and deferring jobs, syncs, and standard alarms. Your script's timers are exactly the kind of work Doze defers. The device wakes into maintenance windows periodically, which is why backgrounded tasks often run "for a while" in fits and starts, then stop for good. Apps exempted from battery optimization (below) are largely released from Doze's restrictions.

### App Standby buckets

Standby buckets rank every app by how recently and often it was used. A freshly foregrounded app sits in the active bucket; a script engine last touched days ago drifts toward rare or restricted. Lower buckets mean deferred jobs and restricted network — and on cloud phones, automation apps are, by definition, rarely foregrounded. This is a slow-motion throttle: the same script that ran fine on day one can be demoted by week two. Buckets can be inspected and, on managed devices, adjusted with `adb shell am get-standby-bucket <pkg>` and `am set-standby-bucket`.

### Per-app battery restriction

This is the manual lever. Device settings expose per-app battery control; setting an app to Unrestricted (the "ignore battery optimizations" exemption) lifts most Doze and App Standby throttling for that app. Two practical limits: the setting path varies by OEM build, and Google Play policy only lets apps *request* this exemption when their core function justifies it — a generic script engine usually won't qualify. On a managed fleet, the operator sets it from device settings instead, which is legitimate.

### Android 15 and 16: cached-app freezing

Android 15 introduced background memory limits: cached (backgrounded) apps are frozen after roughly six hours — the process stays alive, but there's no CPU, no timers, no network. Android 16 extends this to more devices and shortens the window to about four hours on devices with under 8 GB of RAM. This is the newest way a task can "stop after a while," and it's the one most likely to surprise teams moving to newer OS images. It's also the one with the fewest workarounds: the exemption logic belongs to the OS, not to a battery toggle.

### Why AutoJS and long-running scripts are hit hardest

AutoJS and similar script engines run JavaScript inside a normal app process. To Android, that isn't "work" — no system service, no persistent-work contract — so it gets no special treatment. The scripts depend on timers and synchronous execution, which Doze defers and freezing halts outright. The original AutoJS project is no longer actively maintained, and community forks carry it on; none of them changes what the OS does to a backgrounded process. Long-running scripts (30 minutes or more) cross every threshold: Doze's idle timer, bucket demotion over days, and the hour-scale freeze on Android 15 and 16.

### The platform layer: screen-off visibility pauses

Before blaming Android, check the management layer. Cloud phone platforms — the mobile operations layer that QCCBot lives in — often pause app visibility when the screen is off, to save resources or keep the fleet deterministic. That pause is a platform event: it shows up in platform logs, not in `am_kill` or freeze messages, and it stops visibility without touching the process. Separating "the OS froze my process" from "the platform paused my app's visibility" is step zero, and it's where misdiagnosis usually starts.

## Decision Framework

For the classic failure — "my task runs a while, then stops after the app goes to background" — check in this order, and don't change any setting until step 4:

1. **Reproduce with the screen on.** Re-run the task with the screen kept awake. If it completes, the pause was the platform's screen-off visibility control, not an OS kill.
2. **Ask whether the process is even dead.** `adb shell dumpsys activity processes | grep <pkg>`: process alive but not executing means throttling or freezing, not a kill.
3. **Check Doze.** `adb shell dumpsys deviceidle` shows device state and whitelist. Deep doze plus an un-whitelisted app means Doze is your culprit.
4. **Check the standby bucket.** `adb shell am get-standby-bucket <pkg>`: rare or restricted means the OS has been throttling you for days, not minutes.
5. **Check the battery restriction.** Inspect the app's per-app battery setting and `adb shell cmd appops get <pkg> RUN_ANY_IN_BACKGROUND` for background-operation restrictions.
6. **Check logcat for freeze/kill events.** "Freezing process" and `am_kill` lines are the actual "android background task killed" evidence. Log everything — this is exactly the case where logs decide the fix ([AI agents need logs, and mobile automation needs them even more](/blog/ai-agent-logs-for-mobile-automation/)).

**Safely changeable in device settings:** set the automation app to Unrestricted battery; disable ROM-specific "suspend execution" or "pause app activity" toggles if the build has them; keep the app in recents; whitelist it from Doze where the build allows; keep the screen on for short tasks. None of these is a guarantee — they are the manual levers Android documents. Treat device-level changes as fleet policy, not one-off tweaks ([Agentic Automation Security: How to Keep Cloud Phone Account Work Under Control](/blog/agentic-automation-security-cloud-phone-accounts/)).

**App or platform behavior — don't fight it in settings:** cached-app freezing rules (Android 15/16), foreground-service timeouts, Play policy on battery-exemption requests, and platform-side visibility pauses. If a task legitimately runs for hours, the durable answer is to design for the limits: split work into resumable steps, use persistent-work APIs like WorkManager where the app supports them, and monitor from the platform side ([an AI agent control tower exists so this is visible, not guessed](/blog/ai-agent-control-tower-for-mobile-app-workflows/)). Changing settings to "fix" an OS rule you can't change just moves the failure.

## Key Takeaways

- An "android background task killed" report is usually throttling, deferral, or freezing — not a crash, and rarely a platform bug.
- Four OS layers end background work — Doze, App Standby buckets, battery restriction, cached-app freezing — plus a fifth, non-OS layer: the platform's screen-off visibility pause.
- Check in order — screen-on reproduction, process liveness, Doze state, standby bucket, battery restriction, logcat — before changing anything.
- Unrestricted battery is the highest-leverage safe change on managed cloud phones, but it does not override cached-app freezing or foreground-service timeouts.
- Long-running script engines like AutoJS are the most affected because they depend on timers in an ordinary process; plan around the limits rather than around a permanent exemption.
- When a mid-task failure does happen, the real question is what happens next — and that answer belongs in your operations loop, not just in OS logs ([AI Agent Failed Mid-Task. What Happens Next?](/blog/ai-agent-fails-what-happens-next/)). For operations teams, background throttling is a fleet-wide condition, not a per-device mystery — which is why the mobile operations layer exists ([AI Agents Are Becoming Apps. But Who Handles the Mobile Operations Layer?](/blog/agentic-apps-need-mobile-operations-layer/)).

## FAQ

Q: Is my task actually being killed by Android, or is something else going on?
A: Usually the process was never killed. Android defers or freezes background work through Doze, App Standby buckets, battery restriction, and cached-app freezing. A genuine kill leaves an `am_kill` or "Process killed" line in logcat; throttling and freezing don't. Check logcat and `dumpsys deviceidle` before assuming the worst.

Q: If I set my automation app to Unrestricted battery, will that fix everything?
A: It lifts most Doze and App Standby throttling for that app, which fixes the most common failure mode. It does not override Android 15/16 cached-app freezing, foreground-service timeouts, or the platform's screen-off visibility pause. It's the right first change, not the only change.

Q: Why does my script run for a few minutes and then stop instead of failing immediately?
A: Because it's being throttled, not killed. Doze defers execution but opens short maintenance windows, so the script runs in bursts and then stops when the windows stop. On newer Android images, the same gradual stop happens when a cached app is frozen after hours in the background.

Q: Can I disable Android 15/16 cached-app freezing on cloud phones?
A: Not from device settings — it's an OS memory-management rule. The realistic options are keeping the app out of the cached state (a foreground service, where the app and OS allow it), keeping the screen on for short tasks, or designing tasks to finish within the window and splitting longer work into resumable steps with monitoring.

## Sources

- Android Developers — Optimize for Doze and App Standby: https://developer.android.com/training/monitoring-device-state/doze-standby
- Android Developers — App Standby Buckets: https://developer.android.com/topic/performance/appstandby
- Android 15 behavior changes (background memory limits): https://developer.android.com/about/versions/15/behavior-changes-all
- Android 16 behavior changes (background execution limits): https://developer.android.com/about/versions/16/behavior-changes-all
- Android Developers — WorkManager for persistent background work: https://developer.android.com/topic/libraries/architecture/workmanager
