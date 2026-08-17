---
title: "Your App Keeps Stopping on the Cloud Phone: A Crash Playbook for
  Automation Teams"
description: '"App keeps stopping" means process death, not a popup or timeout.
  Learn how to recognize, capture, and triage Android app crashes on cloud
  phones before restarting.'
pubDate: 2026-08-17
updatedDate: 2026-08-17
---

## Answer First

**Definition:** A crash is the death of an app's process on the device. When a cloud phone shows "X keeps stopping" — or its slower cousin "X isn't responding" — the Android system is telling you the app's process was killed or stopped responding, not that a dialog interrupted it. For automation teams this is a distinct failure class: popups, bans, and network drops interrupt a running process; a crash ends it. Nothing in the app runs again until something restarts it.

**Why:** The distinction decides everything downstream. A popup needs a click, a network drop needs a retry, a ban needs a cooldown — and a crash needs a diagnosis. Misclassify a crash as a timeout and you'll restart and re-run, masking the defect and burning runs. Misclassify a ban as a crash and you'll roll back versions while the account problem keeps happening. Crashes also masquerade as "task did nothing" outcomes: the agent's log ends cleanly because there is no error to log — the process simply ceased.

**Example:** A scheduling task opens a delivery app, signs in, and places an order. The run log ends at "waiting for the checkout screen" with no error, and the task is marked as having done nothing. Ten seconds later, the next run's screenshot shows the same "keeps stopping" dialog. Logcat pulled before the restart shows a FATAL EXCEPTION in AndroidRuntime at the exact minute the log went quiet. That is a crash: the process died mid-task, and no amount of re-login or network retry will fix it.

## Key Facts

- "X keeps stopping" is the system's crash dialog; "X isn't responding" is its ANR (application not responding) dialog. Both are rendered by the system UI, and both mean the process failed, not just the UI.
- The failure class is defined by process death: an uncaught exception, a native fault, or the system killing an unresponsive process.
- A crash destroys the task's last state: screenshots after the dialog may show a different screen, and the app is gone from recents.
- Logs alone are ambiguous: empty task logs, timed-out interactions, and "app did nothing" outcomes are all compatible with a crash — which is why they're so often misdiagnosed.
- Restarting destroys the evidence. Android's logcat buffer is volatile — once it rolls over or the device is recycled, the crash trace is unrecoverable.
- Auto-restart masks recurring crashes: each restart looks like a successful recovery, so recurrence stays invisible.

## Expert Explanation

### How to recognize a crash

Look for four signatures in sequence:

1. **Dialog text.** "X keeps stopping" is a crash; "X isn't responding" is an ANR. An ANR can sometimes be resolved by tapping "wait" — but on a headless cloud phone nobody taps it, so the process is usually killed.
2. **"App recently crashed" banners.** Reopening the app after a crash shows the system's "app recently crashed" notice, and some launchers re-surface the dialog on the next launch.
3. **Empty task logs.** The strongest automation-specific signature: the agent's last action succeeded ("tapped Login"), then the log simply stops — no error, no retry, no timeout — because the process died before it could log anything.
4. **Missing from recents.** An app that crashed mid-task is absent from the recent-apps list, while an app blocked by a popup or a network error is still there.

Timing is a tell too: crashes that follow the same action — login, file upload, a screen transition — point at a code path, not a device glitch.

### Capture evidence before restarting

When you see the dialog, the crash is already a forensics scene. Capture in this order:

- **Screenshot the dialog** with the status-bar clock visible to correlate with log timestamps.
- **Pull logcat around the crash window.** Filter for FATAL EXCEPTION, AndroidRuntime, and your app's package name, plus the minute before and after the log went quiet.
- **Run dumpsys** for activity and memory state (dumpsys activity, dumpsys meminfo) — this distinguishes an uncaught exception from a low-memory kill.
- **Record app and device state:** app version, last update time, storage usage, and whether the session was still logged in.
- **Save the agent's action trail** with timestamps — the last successful step and when — plus the audit record of which agent and task ran on that device.

Then, and only then, restart. If your platform captures this automatically before recycling a device, treat that as a feature: [logs carry most of the signal in mobile automation](/blog/ai-agent-logs-for-mobile-automation/), and so do the [permissions and audit trails](/blog/ai-agent-permissions-audit-trails-cloud-phone/) that tell you who ran what.

### Crash vs. soft ban vs. session logout

All three can end a task with the same visible outcome — "the app did nothing" — but they fail at different layers:

| Signature | Crash | Soft ban | Session logout |
|---|---|---|---|
| Process dies | Yes | No | No |
| Network / API error | Usually none | Yes (4xx, rate limits) | Sometimes (401, invalid token) |
| App still in recents | No | Yes | Yes |
| System dialog shown | "keeps stopping" / "isn't responding" | No | No |
| Log signature | FATAL EXCEPTION, process death | API error codes, throttling | Redirect to login, token expiry |
| Fix class | Version rollback, device state | Cooldown, account hygiene | Re-authentication |

Two complications make this harder. First, a ban or logout can cause a crash if the app mishandles an API error with an unhandled exception. Second, a crash can be misread as a ban because the auto-restart loop fails repeatedly. The decision rule is where the failure happened: process/device level (crash) versus account/network level (ban, logout). If the app is still alive in recents and the API is answering with errors, you are in ban territory, not crash territory — follow the [security guidance for cloud phone account work](/blog/agentic-automation-security-cloud-phone-accounts/) instead.

### Practical limits

Evidence capture has hard limits: logcat buffers are small and volatile, so a delayed capture may miss the crash; a restart or device recycle destroys the trace permanently; a crash on one device image may not reproduce on another, so reproduction on demand isn't guaranteed; and a dialog screenshot shows the symptom, not the stack trace. Since cloud-phone devices may be recycled between tasks, export evidence before the device is released.

## Decision Framework

Restart now when the crash is a first occurrence, no evidence exists yet, and the task is time-sensitive. Arm evidence capture first, and treat the restart as the start of an investigation, not the end of one.

Investigate first when:

- The same app or the same action has crashed before. Auto-restart will simply burn runs — the same logic as [any mid-task agent failure](/blog/ai-agent-fails-what-happens-next/) applies, except this "failure" left no error message.
- The crashes started right after an app update. This is the single most common recurring pattern; the fix is version pinning or rollback, not more restarts.
- The crash is tied to one device or profile — that points to storage, account data, or device state, and dumpsys plus storage checks are the next step.
- The logs look like a ban but the app is dying. Run the process-death check (recents, logcat) before reaching for the ban playbook.

Why auto-restart alone is a trap: every "successful" restart produces a green run, and recurrence hides in aggregate stats. Track crashes per app, version, and device — not just per task.

When a human review pass is the right call: recurrence on the same code path, evidence pointing at a version or device defect, any account-level symptom also present, or the app being business-critical. A crash that kills a login or order step is exactly the case for [controlled takeover with a human in the loop](/blog/ai-agent-control-boundaries-cloud-phone-takeover/), and for the [review tooling operations teams actually use](/blog/ai-agent-control-tower-for-mobile-app-workflows/).

## Key Takeaways

- "Keeps stopping" and "isn't responding" mean process death — a different failure class from popups, bans, and network drops.
- Capture before you restart. Screenshot, logcat, dumpsys, and version/storage state are worthless after the process is gone.
- Verify the layer before choosing the playbook. Recents plus logcat decide crash versus ban versus logout.
- Auto-restart is a first response, not a fix. Track recurrence per app, version, and device, and escalate.
- Recurring crashes earn a human review pass — especially after updates or on account-critical flows.

## FAQ

**Q: What does "X keeps stopping" actually mean on a cloud phone?**
A: It means the app's process died. Android detected an uncaught exception, a native fault, or an unresponsive process and killed it. The system shows the dialog, so it appears even if the app cannot render anything. It is not a popup, a ban, or a network problem.

**Q: Is "keeps stopping" the same as a ban or a timeout?**
A: No. A ban or logout fails at the account/network layer and the app stays alive; a crash fails at the process layer and the app disappears from recents. The confusing part is that all three can produce an empty task log — which is why the process-death check (recents, logcat FATAL EXCEPTION) matters before you choose a fix.

**Q: Should I auto-restart the app whenever it keeps stopping?**
A: As a first response, yes — but only if evidence capture runs first and you track recurrence. Auto-restart alone masks a recurring crash as a series of successful recoveries, and a crash that repeats on the same action or version needs a diagnosis (often a rollback) rather than more restarts.

**Q: What evidence should I collect before restarting?**
A: A screenshot of the dialog (with clock), logcat around the crash window filtered for FATAL EXCEPTION and the app package, dumpsys activity and memory state, the app version and last update time, storage/account state, and the agent's last actions with timestamps. After a restart or device recycle, most of this is unrecoverable.

## Sources

- Understand crashes — Android developers: https://developer.android.com/topic/performance/crashes
- Understand ANRs — Android developers: https://developer.android.com/topic/performance/anr
- Processes and app lifecycle — Android developers: https://developer.android.com/guide/components/activities/process-lifecycle
- logcat — Android developers: https://developer.android.com/tools/logcat
- dumpsys — Android developers: https://developer.android.com/tools/dumpsys
- About app crashes and ANRs — Google Play Console help: https://support.google.com/googleplay/android-developer/answer/9888077
