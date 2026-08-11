---
title: Your Cloud Phone Task Didn't Fail — Android Doze Mode Put It to Sleep
description: Cloud phone tasks stalling? Often it's Android Doze mode, not your
  script. Learn to tell OS throttling from script failure and which device-side
  controls you own.
pubDate: 2026-08-11
updatedDate: 2026-08-11
---

## Answer First

If an unattended cloud phone task comes back "failed" with a clean log — no exception, no non-zero exit — the script probably never ran. Android's power management put the task to sleep.

**Definition:** Doze mode is Android's power-saving state for idle devices: when a phone has the screen off, is stationary, and is unplugged, Android suspends network access and wake locks, defers alarms and background jobs, and lets work run only in short maintenance windows. App Standby buckets and background execution limits (Android 8.0+) extend the same throttling to background apps on every modern Android version.

**Why:** The conditions that trigger Doze are exactly the conditions of unattended cloud phone automation: screen off, no interaction, long quiet stretches between scheduled tasks. The OS silently delays or drops the work — the app never receives the alarm, the job never starts, the upload never gets network. The script doesn't fail; it just doesn't run. Teams add retries, raise timeouts, buy more phones — the throttling keeps eating all of it because the device state never changed.

**Example:** An operator schedules a fleet-wide sync for 2:00 a.m. At 2:00 a.m. every phone sits idle, screen off. Doze defers the sync alarm to the next maintenance window; several phones run the job at 5:40 a.m., a few report it as failed because it never started in time, and the ones that did start stall halfway through an upload before network is suspended again. Execution logs are empty of errors. The scripts are fine. The phones were asleep.

## Key Facts

- Doze engages automatically when a device is idle: screen off, stationary, not charging (Android 6.0+). Network and wake locks are suspended; alarms and jobs wait for maintenance windows.
- App Standby buckets (active, working set, frequent, rare, never) rank app usage; lower buckets mean deferred jobs and, when unplugged, restricted network.
- Background execution limits (Android 8.0+) stop background apps from starting services freely or relying on implicit broadcasts; Android 14+ requires foreground services to declare a type, and Android 15 caps some at six hours.
- Per-app battery optimization ("Unrestricted," "Optimized," "Restricted") is the user-facing override; a device owner or MDM can apply exemptions fleet-wide.
- The throttling is invisible to the script: logs stay clean because the work was never delivered, not because it failed.
- Android Doze mode cloud phone tasks are a recurring "task failed for no reason" cause on every managed fleet — worth a standing triage step.

## Expert Explanation

**What Doze actually does.** Full Doze engages after the device has been idle a while: alarms are deferred (except exact alarms), jobs wait, and network and wake locks are suspended. Periodically the device wakes into a maintenance window long enough to drain pending work, then sleeps. To a task, the world stops between windows.

**App Standby and buckets.** Android 9 replaced the original App Standby model with buckets. A rarely used app in the "rare" bucket gets deferred jobs and, when unplugged, restricted network — far more than one in "active" or "working set." An agent app that runs once a day at 2:00 a.m. is, by usage pattern, exactly what the system decides is rare.

**Background execution limits and foreground services.** Since Android 8, a background app can't start services freely or rely on implicit broadcasts; it must use JobScheduler, WorkManager, or a foreground service with a visible notification. Android 14 requires declaring a foreground service type and matching permission; Android 15 caps dataSync and mediaProcessing services at six hours. Work that "should just run" has been deliberately made harder to run.

**The cloud phone wrinkle.** Full Doze needs the framework to believe the device is unplugged, stationary, and screen-off. Virtual cloud phones vary: some emulated environments model the battery as charging or the device as in motion, which suppresses full Doze. But App Standby buckets and background execution limits apply on any Android instance, virtual or physical — verify on the fleet image you run. Fleet operators meet this same class of ambient failure in the agentic layer: [AI agents are becoming apps, and someone has to handle the mobile operations layer they run on](/blog/agentic-apps-need-mobile-operations-layer/).

**Why it masquerades as failure.** The orchestrator sees "job didn't complete in time," the operator sees a red row, and the script shows no error — so retries kick in, reset usage signals, and can push the app into an even lower bucket. Notifications arrive late, timers misfire, and uploads stall because delivery, alarms, and network were deferred or suspended. Every symptom looks like a script problem, and none of them are.

## Decision Framework

Check the device before you touch the code:

1. **Reproduce with the screen on.** Run the task foregrounded, screen on, charging. If it works there and fails unattended, suspect power management first.
2. **Read device state.** `dumpsys deviceidle` shows the idle state; `am get-standby-bucket <package>` the app's bucket; `dumpsys jobscheduler` and `dumpsys alarm` the deferred work.
3. **Correlate timestamps.** Do failures cluster at night, during screen-off stretches, or at maintenance-window boundaries? That pattern is the OS fingerprint; real script failures don't schedule themselves around sleep cycles.
4. **Check the execution log for actual errors.** An exception, non-zero exit, or mid-transfer timeout is evidence of a script or environment bug; a clean log with a "didn't run" verdict is evidence of throttling. Logs are the deciding document — [mobile automation needs them even more](/blog/ai-agent-logs-for-mobile-automation/).
5. **Apply device-side controls, then re-test in the idle state** before touching retry logic.

**Triage table:**

| Symptom | Likely layer | Evidence to collect |
|---|---|---|
| Runs while you watch, fails unattended | Doze / App Standby | `dumpsys deviceidle`, bucket, screen-off timeline |
| Alarm or notification arrives late | Doze alarm deferral | `dumpsys alarm`, maintenance-window timestamps |
| Upload stalls mid-transfer, resumes on unlock | Network suspended / service killed | logcat, battery stats, `dumpsys jobscheduler` |
| Job pending forever, no error | JobScheduler throttling | job state, standby bucket |
| Script throws a real exception or non-zero exit | Script or environment bug | execution log, error code, foreground repro |

**The controls you actually own.** Per-device and per-app — fleet policy is how you scale them:

- **Keep the device awake:** the "Stay awake" developer option or `svc power stayon true` while charging, plus no screen timeout on the fleet profile.
- **Exempt the agent app from battery optimization:** "Unrestricted" in battery settings, `dumpsys deviceidle whitelist +<package>`, or a device-owner/MDM policy on managed devices.
- **Raise the standby bucket for testing:** `am set-standby-bucket <package> active` — the system re-buckets by usage, so use it for testing, not as a fix.
- **Move critical work to a foreground service** with the declared type and permission (Android 14+), and budget for the Android 15 six-hour cap.
- **Use a high-priority push channel** so delivery wakes the device instead of relying on polling that Doze defers.
- **Test in the idle state before rollout:** `dumpsys deviceidle force-idle` and `am set-inactive <package> true` reproduce production conditions on demand.

**Practical limits.** These controls are real but not guarantees: OEM ROMs add their own aggressive battery managers that can override stock behavior; exemptions are per-app and can be reset by users or factory resets; keep-awake and whitelisting cost battery and heat; there is no supported, unrooted way to disable Doze system-wide; and foreground services carry time and type limits on recent versions. Make throttling visible and testable — don't try to defeat it.

## Key Takeaways

- A clean execution log with an unattended failure is a power-management red flag, not a script bug.
- Android Doze mode cloud phone tasks are throttled in three layers — Doze, App Standby buckets, and background execution limits — and you need evidence from all three.
- `dumpsys deviceidle`, `am get-standby-bucket`, `dumpsys jobscheduler`, and `dumpsys alarm` are your triage instruments; timestamp patterns are the OS fingerprint.
- Keep-awake settings, battery-optimization exclusion, foreground services, and high-priority push are the controls operators actually own — apply as fleet policy, not phone by phone.
- When the evidence does point to a real failure, treat it differently: the playbook for [an agent that genuinely failed mid-task](/blog/ai-agent-fails-what-happens-next/) is not the same as un-throttling a sleeping device.
- Monitoring is the other half — surface device state next to execution history, the way an [AI agent control tower for mobile workflows](/blog/ai-agent-control-tower-for-mobile-app-workflows/) would.

## FAQ

**Q: How can I tell whether Doze throttled the task instead of the script failing?**
A: Check device state first using the Decision Framework commands: the idle state, the app's standby bucket, and any deferred jobs or alarms. Then correlate: does the task fail at night with the screen off and succeed foregrounded with the screen on? If the log has no exception or non-zero exit and behavior tracks the screen state, that's throttling, not a bug.

**Q: Does Doze even apply to cloud phones — they're not physical devices?**
A: It depends on how the vendor emulates the device. Full Doze requires the framework to believe the device is idle — screen off, stationary, unplugged — and some cloud phone environments model the battery as charging or the device as in motion, which suppresses it. But App Standby buckets and background execution limits apply on any Android instance, virtual or physical. Test on your actual fleet image with `dumpsys deviceidle force-idle`; never assume virtual equals immune.

**Q: Can I just whitelist my agent app or disable battery optimization on all phones?**
A: Partially, with trade-offs. "Unrestricted" or the power whitelist stops Doze from throttling that app, but it's per-app and per-device, can be reset by users or factory resets, shortens battery life, and OEM battery managers may override it. Pair it with keep-awake settings and foreground services, and re-test in the idle state.

**Q: What's the difference between Doze, App Standby, and background execution limits?**
A: Three layers of the same system. Doze suspends the whole device's network, wake locks, alarms, and jobs while it's idle. App Standby buckets rank how often an app is used and throttle jobs and network for rarely used apps when the device isn't charging. Background execution limits (Android 8+) restrict what a background app may do at all — starting services, receiving broadcasts — even on a device that's awake and charging. A task can be hit by any of the three, so triage checks all of them.

## Sources

- **Optimize for Doze and App Standby** — Android Developers. Doze behavior, maintenance windows, and the official testing commands: https://developer.android.com/training/monitoring-device-state/doze-standby
- **App Standby Buckets** — Android Developers. Bucket definitions and their job and network restrictions: https://developer.android.com/topic/performance/app-standby
- **Background execution limits** — Android Developers. Service-start and implicit-broadcast restrictions in Android 8.0: https://developer.android.com/about/versions/oreo/background
- **Foreground services** — Android Developers. The notification requirement and type/permission rules on modern Android: https://developer.android.com/guide/components/foreground-services
