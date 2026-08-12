---
title: Cloud Phone Screen Is Off? Why Tasks Pause When No One Is Watching
description: "Why cloud phone tasks stop running when the screen is off: how
  Doze, app standby, and background state pause unattended runs — plus how to
  diagnose and fix them."
pubDate: 2026-08-12
updatedDate: 2026-08-12
---

## Answer First

**Definition:** When unattended cloud phone tasks stop running — pausing mid-run, timing out on a wait, or silently no-oping a step — the culprit is often not the script. Android's power-management stack (Doze mode, app standby, lock screens, background-foreground state) treats an idle, screen-off device as permission to throttle exactly the work your task depends on. "Cloud phone tasks stop running" is frequently a device-state failure wearing a script-bug costume.

**Why:** When a human is watching, the screen is on, the app is in the foreground, and the device never enters the idle states Android uses to decide what may run. Unattended, the same device does: the screen sleeps, the OS declares it idle, Doze defers network and CPU work, app standby throttles untouched apps, and a backgrounded app can lose services or get killed. The script doesn't change — the device state does. Hence the signature: worked while watched, failed unattended.

**Example:** A nightly inventory-sync task logs in, then waits up to 90 seconds for an in-app confirmation element. At 02:00 the screen sleeps, the device idles, the app's network calls are deferred; the wait times out and the run is marked failed. The identical task passes at 14:00 with the screen on. Nothing in the script changed — the device state did.

## Key Facts

- Doze defers network, jobs, and syncs when a device is idle with the screen off; app standby restricts apps nobody has interacted with — automation apps fit that description because they run unattended.
- Screen-off and lock are state changes the OS acts on, not cosmetic display settings.
- Device-state failures have a fingerprint: they correlate with idle windows, pause on waits rather than crash on actions, and hit individual devices, not fleets.
- They form a distinct failure family from script bugs, popups, and network issues — and are routinely misdiagnosed as all three.

## Expert Explanation

Android's idle machinery assumes that if the screen is off and nobody is interacting, the device is idle and power should be conserved. Four mechanisms enforce that assumption, and each can quietly break an unattended run.

**Doze mode.** With the screen off and the device idle (and, on physical hardware, unplugged), the system enters Doze: network access and jobs are deferred, and the app gets only periodic maintenance windows. A task that spends its runtime waiting for a server response — a sync, a status poll, a two-factor confirmation — can stall until the next maintenance window or time out.

**App standby.** Even outside Doze, Android restricts apps the user hasn't recently interacted with. The more reliably your task runs unattended, the more the OS believes the app is unused — and the more aggressively it throttles it.

**Lock screen and screen-off.** On a locked device, input is blocked and fresh UI stops rendering — and many tasks interact with elements that only exist while the app is foreground with the screen on. If device policy locks the phone during an idle window, the next task may find a device that looks healthy but refuses every interaction.

**Background-foreground state.** Modern Android restricts what background services can do even with the screen on, and cached apps can be killed under memory pressure. A wake lock only keeps the CPU running — not the app foreground, and not an app-standby exemption. Without the right foreground or exemption state, the OS revokes the very resources the task is using.

| State | What the OS does | Failure signature |
|---|---|---|
| Doze mode | Defers network, jobs, syncs; throttles CPU | Run stalls on a wait or network call during idle; times out |
| App standby | Defers background work for unused apps | Works at 14:00, no-ops at 03:00 after hours without interaction |
| Lock screen / screen-off | Blocks input, stops UI rendering, marks device idle | Taps hit nothing; elements never appear; device "looks fine" |
| Backgrounded app | Restricts services, may kill the cached app | Run dies mid-step with no script error; app state lost |

**Why teams misdiagnose it.** The device looks healthy in monitoring, logs show a timeout rather than an error, and the same script passes during the day — so flaky selectors, a popup, or a platform bug all look plausible. They are per-device and time-correlated, so they rarely surface as fleet-wide incidents; each run fails quietly on its own schedule.

**How cloud phones differ.** Hosted Android images don't always behave like a phone in a pocket: some cloud phone vendors disable Doze, simulate a plugged-in battery, or tune OEM-style power managers differently, so on some fleets this failure family barely exists and on others it's the top cause of failed runs — verify per fleet rather than assume. Physical-device fleets add OEM battery optimizers, which can be more aggressive than stock Android.

Whether a person or an agent drives the session, the OS still sees an idle, screen-off device and throttles it — a device that looks fine can be silently refusing to cooperate. That's the gap the [operations layer for agentic mobile work](/blog/agentic-apps-need-mobile-operations-layer/) has to cover, and since these failures are quiet, [logging for mobile automation](/blog/ai-agent-logs-for-mobile-automation/) must capture device state, not just script steps.

## Decision Framework

Before you blame the script, confirm or rule out device state with this checklist:

- [ ] **Capture the failure signature.** Did the run pause on a wait or network-dependent step rather than crash on an explicit action? Pause-on-wait points to device state; crash-on-action points to script or UI.
- [ ] **Correlate with idle windows.** Do failures cluster at night, after long no-interaction gaps, or after a screen-off event? Device-state failures are time-correlated; script bugs are not.
- [ ] **Check device state at failure time.** Was the screen off, the device idle, the app foreground? Use platform monitoring and device logs, not memory.
- [ ] **Reproduce with the screen forced on.** Rerun with a wake lock, screen-on setting, or active session; if it passes, device state is confirmed.
- [ ] **Rule out sibling families.** Popups and UI changes break runs at any hour; network issues tend to hit the whole fleet; script bugs fail identically with the screen on.
- [ ] **Look for OS-level evidence.** Battery stats, Doze and app-standby exemptions, and power-related logcat lines confirm which mechanism throttled the app.

If the checklist points at device state, the task design — not the script logic — needs to change. And failed runs need a recovery path: a silent no-op that "succeeded" is worse than a loud failure — see [how AI agent failures should be handled mid-task](/blog/ai-agent-fails-what-happens-next/).

## Key Takeaways

- Treat idle, screen-off, and background state as first-class failure causes on cloud phones — a distinct family from script bugs, popups, and network issues.
- Confirm before blaming: use the decision framework, and keep logs that record device state so you can prove which mechanism throttled the run.
- Design for unattended operation: keep the app foreground where the platform allows, use wake-lock and screen-on settings deliberately, and re-check screen and foreground state after long idle windows before critical steps — design for the OS, not against it.
- Respect practical limits: OEM battery optimizers and cloud phone images vary, so verify Doze behavior per fleet; wake locks cost battery and thermal headroom and require permission — [permissions and audit trails matter for mobile automation](/blog/ai-agent-permissions-audit-trails-cloud-phone/) for good reason.
- Monitor device state, not just task outcome: a [control tower view for mobile workflows](/blog/ai-agent-control-tower-for-mobile-app-workflows/) is what surfaces "ran but did nothing" runs before they compound.

## FAQ

**Q: Why do my cloud phone tasks stop running at night but pass during the day?**

A: The most likely cause is device state, not the script. At night the screen sleeps, the device idles, and Android's power stack — Doze and app standby — defers the network and background work your task depends on. The daytime pass happens because a watched session keeps the screen on and the app foreground. Confirm with the decision framework — correlate failures with idle windows, check device state at failure time, rerun with the screen forced on — and if it passes, device state was the culprit.

**Q: If I use a wake lock, is my task guaranteed to keep running?**

A: No. A wake lock keeps the CPU awake but does not keep the app foreground, does not exempt it from app standby, and can be overridden by aggressive OEM battery optimizers. They also cost battery and thermal headroom and need the right permission. Treat a wake lock as one tool in a set — foreground state, exemptions, re-checking after idle windows — not a guarantee.

**Q: How do I tell a device-state pause from a popup or UI change?**

A: By determinism and timing. Popups and UI changes break runs at any hour, regardless of screen state, and typically fail the same way on every device. Device-state failures are time-correlated with idle windows, pause on waits rather than crashing, hit individual devices, and pass when the screen is forced on.

**Q: Should I just disable Doze on my cloud phones?**

A: Not as a first move. Some cloud phone vendors already disable or tune Doze, so blanket changes may do nothing; on other fleets, disabling it trades away battery and thermal stability for the whole device. The more robust fix is task design: keep the app foreground, hold the right exemptions, and re-check state after idle windows so tasks tolerate Doze instead of depending on its absence.

## Sources

- Android Developers — [Optimize for Doze and App Standby](https://developer.android.com/training/monitoring-device-state/doze-standby): documented Doze and app standby behavior, including what is deferred when a device is idle with the screen off.
- Android Developers — [Keep the device awake](https://developer.android.com/training/scheduling/wakelocks): what wake locks do and don't do, and recommended patterns for keeping the CPU on with the screen off.
- Android Developers — [Background execution limits](https://developer.android.com/about/versions/oreo/background): Android 8.0+ restrictions on background services, independent of screen state.
- Android Developers — [App power best practices](https://developer.android.com/topic/performance/vitals/power): guidance on minimizing wake locks and optimizing for Doze and app standby.
