---
title: "Cloud Phone Batch Preflight: What to Check Before Running Tasks on Many
  Phones"
description: "A cloud phone batch preflight checklist: seven ordered,
  automatable checks with pass/fail criteria per device, a fleet-level abort
  rule, and when to re-run the gate. Run tasks on many phones without burning
  the batch."
pubDate: 2026-08-17
updatedDate: 2026-08-17
---

## Answer First

**Definition:** A cloud phone batch preflight is a short, ordered set of automatable pass/fail checks run against every device in a fleet before a batch task starts. Each device receives a verdict — pass, fail, or quarantine — and the batch executes only on devices that can actually complete it.

**Why:** Batch failures are rarely random. Storage that fills mid-run, a screen that fell asleep, an expired session, a staged app rollout that split the fleet across two versions — all of these are detectable in seconds per device before any task runs. Most of the failure modes this blog documents reactively, like the mid-task triage in [AI Agent Failed Mid-Task. What Happens Next?](/blog/ai-agent-fails-what-happens-next/), share a small set of root causes a preflight catches in advance. Checking cheap signals first catches the devices that would have failed halfway through a run before they burn task slots and produce partial evidence.

**Example:** A team schedules a 500-device batch that opens a storefront app, screenshots listings, and flags anomalies for human review. The gate runs seven checks per device and finishes in about a minute across the fleet: 473 passes, 24 storage failures, 3 expired sessions. The batch runs on the 473; the 27 failures are quarantined with their check results attached. Without the gate, those 27 devices would have started tasks that failed mid-run, each leaving partial evidence and manual follow-up.

## Key Facts

A preflight is a checklist with an explicit pass/fail criterion per device per check, ordered cheap-first: the cheap checks catch most failures, and a device stops being checked after its first hard failure.

| # | Check | What it catches | Pass | Fail |
|---|-------|-----------------|------|------|
| 1 | Storage headroom | Evidence capture and app writes fail or corrupt mid-run | ≥ 500 MB free, or ≥ 10% of total | Below threshold |
| 2 | Network / proxy reachability | Outbound calls time out; tasks "succeed" with empty data | Control endpoint reachable, proxy authenticated, latency under threshold | Timeout or auth error on a required endpoint |
| 3 | Screen / awake state | UI automation mis-taps or waits forever | Screen on, not in Doze, expected app in foreground | Screen off, locked, or app backgrounded |
| 4 | App version pinning | Scripts assume behavior that changed between versions | Installed version matches the version the task was validated against | Version mismatch |
| 5 | Session / login state | Task runs, but every action fails authentication | Token valid, account reachable, no re-auth or MFA prompt | Expired token or logged-out state |
| 6 | Clock / timezone | Scheduled actions fire wrong; evidence timestamps lie | Clock within tolerance of a reference; timezone matches task assumptions | Skew or timezone mismatch beyond tolerance |
| 7 | Evidence capacity | Review is blind because screenshots or logs never persisted | Log stream healthy, screenshot target writable, quota covers the run | Logs dropped, writes fail, quota exceeded |

Practical limits: preflight is a readiness gate, not proof of correctness. It cannot validate task logic, predict a device going offline mid-run, or judge visual output — that is the [AI Agent Control Tower for Mobile App Workflows: What Operations Teams Actually Need](/blog/ai-agent-control-tower-for-mobile-app-workflows/) layer. If a check needs a human to interpret the result, it is a review task, not a preflight check.

## Expert Explanation

Every check maps a documented failure mode to a state that is measurable before the run.

**Storage headroom — the silent write failure.** The most common mid-batch killer is not a crash but evidence that never persisted: screenshots and logs missing, so the review step is blind. Evidence is the deliverable in mobile automation, as [AI Agents Need Logs. Mobile Automation Needs Them Even More](/blog/ai-agent-logs-for-mobile-automation/) argues; a gate that measures free space and confirms the evidence target is writable converts a fleet-wide surprise into a per-device verdict.

**Network and proxy reachability — the environment check.** Cloud phone fleets typically route task traffic through proxies or egress pools. When several devices fail at once, the cause is almost always environmental — a rotated credential, a new IP block, a rate limit — and re-running tasks re-fails them. This is the check most likely to trigger the fleet-level abort rule below.

**Screen and awake state — Doze and lock.** Android suspends background work aggressively; [Doze and App Standby](https://developer.android.com/training/monitoring-device-state/doze-standby) defer jobs when the screen is off or the device is idle. UI automation that assumes a foreground app, an unlocked screen, and active rendering will stall or mis-tap otherwise.

**App version pinning — drift.** Scripts are validated against one app version, and fleets drift: [staged rollouts on Google Play](https://support.google.com/googleplay/android-developer/answer/9888077) deliver updates to some devices but not others, and an update can land overnight. If the installed version differs from the validated one, the gate fails the device — the same discipline OWASP's [MASVS](https://mas.owasp.org/MASVS/) applies to verifying the app under test. Version discipline matters more on phones than browsers — there is no shared DOM contract, as [AI Agents Can Use Browsers. What About Phone Apps?](/blog/ai-agents-can-use-browsers-what-about-phone-apps/) explains.

**Session and login state — the wasted run.** An expired token does not fail fast: the task starts, every action bounces with an auth error, and the run produces noise. In account-driven automation, session hygiene is a security control as much as an operational one, as covered in [Agentic Automation Security: How to Keep Cloud Phone Account Work Under Control](/blog/agentic-automation-security-cloud-phone-accounts/). The gate treats a re-auth or MFA screen as a hard fail.

**Clock and timezone — timing skew.** Scheduled actions, rate-limit windows, and evidence timestamps all depend on the device clock; a drifted clock makes tasks fire early or late and poisons evidence review. Cheap to read, cheap to fix, and almost always a fleet-wide pattern when it appears.

**Evidence capacity — the review step.** The final check confirms the pipeline that carries screenshots and logs to the review queue; if it fails, the run "succeeds" while the output never arrives — an automation problem becomes a trust problem.

## Decision Framework

Per-device verdicts are only half the gate; the fleet-level decision needs an explicit rule so operators do not improvise mid-incident.

- **Green — fewer than 1% of devices fail any check.** Quarantine the failures, run the batch on the rest, review the quarantined devices before the next run.
- **Yellow — 1–5% fail, spread across different checks.** Same as green, but the quarantined list is a work item — those devices need attention before rejoining the fleet.
- **Red — more than 5% fail any check, more than 2% fail the same check, or several devices fail a shared-infrastructure check (network/proxy).** Abort the entire run. Fix the environment or the fleet, then re-run the gate. Do not "retry and hope": a fleet-level pattern re-fails the whole batch.

Attach each device's check results to its run record so the review queue starts with context — the audit discipline of [AI Agents Need Permissions and Audit Trails. Mobile Automation Needs Them Too](/blog/ai-agent-permissions-audit-trails-cloud-phone/) — and quarantined devices are never silently re-enqueued.

**Re-run the gate after every environment change.** The checklist is a snapshot, and readiness decays. Treat it as change management: re-run after an app update, a device swap, an OS update, a network migration, a proxy rotation, or a new account batch. A gate run only at onboarding runs on stale state; its value is being the last thing that happens before the batch starts.

## Key Takeaways

- A preflight is a gate, not a report: failing devices do not enter the run, and their check results travel with them into the review queue.
- Order checks cheap-first — storage, clock, network, screen, version, session, evidence — and stop checking a device after its first hard failure.
- Use an explicit fleet-level rule: quarantine below the threshold, abort above it, and treat shared-infrastructure failures as environment problems, not device problems.
- Re-run the gate after every environment change; readiness decays, and a stale gate is worse than none.
- Every check maps to a failure mode the blog already documents; the preflight is a runbook you can build from existing scripts, monitoring, and task execution — no new platform features required.

## FAQ

**Q: How long does a preflight take on a fleet of hundreds of devices?**
A: Seconds per device with cheap-first ordering and fail-fast per device. Storage, clock, and screen state are single property reads; version and session checks are heavier. A 500-device gate typically completes in about a minute.

**Q: What counts as "too many failures to proceed"?**
A: Abort the whole run when more than 5% of devices fail any check, more than 2% fail the same check, or several devices fail a shared-infrastructure check. Below those thresholds, quarantine the failures and run on the rest. The discipline is deciding the threshold before the run, not during it.

**Q: Can a preflight guarantee a batch succeeds?**
A: No. It is a readiness gate, not proof of correctness: it cannot validate task logic, predict a device going offline mid-run, or judge visual output — that is the human review step. Its job is narrower: catch the predictable readiness states that cause most mid-run failures before they cost a run.

**Q: Should we run preflight on every batch, even small ones?**
A: Run it whenever the batch is unattended, touches shared infrastructure, or produces evidence the review step depends on — which is most batches. The marginal cost is seconds per device, so skipping it saves little and risks a full re-run plus triage. For tiny, attended, low-stakes runs, a storage-and-session-only version is a reasonable minimum.

## Sources

- [Android security risks](https://developer.android.com/privacy-and-security/risks) — app updates, permissions, and device state as risk factors for automation targets
- [OWASP Mobile Application Security Verification Standard (MASVS)](https://mas.owasp.org/MASVS/) — verifying the app under test, including version and integrity
- [Google Play: staged rollouts](https://support.google.com/googleplay/android-developer/answer/9888077) — how staged app rollouts create version drift across a fleet
- [Android: Optimize for Doze and App Standby](https://developer.android.com/training/monitoring-device-state/doze-standby) — background-work suspension behind the screen/awake check
