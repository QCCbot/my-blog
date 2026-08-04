---
title: "Make Cloud Phone Tasks Safe to Re-Run: Idempotency for Mobile Automation
  Teams"
description: "How to make cloud phone task steps safe to re-run: run IDs in the
  evidence log, verify-before-act checks, and exception-queue signals for
  idempotent automation."
pubDate: 2026-08-05
updatedDate: 2026-08-05
---

## Answer First

**Definition:** An operation is idempotent when running it multiple times produces the same end state as running it once. In mobile automation, an idempotent task step is one that can be re-run — after a failure, an automatic retry, or a manual re-trigger — without creating a second effect: one post published, one message delivered, one order claimed, never two. Idempotent automation is the design property that a re-run changes nothing the first run already changed.

**Why:** Cloud phone tasks fail in the worst place: after the side effect happened, before the task recorded that it did. A device loses network mid-upload, a script times out waiting for a confirmation dialog, an app crashes right after "Submit" is tapped. Every failure leaves the same ambiguity — did it go through or not? — and the default answer is to re-run. With non-idempotent steps, "not sure" becomes "twice": a storefront gets two listings, a customer gets two messages, a limited order gets claimed twice. It sharpens now because AI agents retry autonomously: every automatic retry multiplies the damage of steps never made safe to re-run.

**Example:** A scheduling task publishes an announcement in a storefront app. The tap lands, the app accepts the post, then the device drops off the network before the task records "posted." Marked failed, the task is re-run and posts again: two listings, one manual cleanup. The idempotent version reads the screen first, sees the announcement already live, and stops.

## Key Facts

- Mobile automation cannot borrow API-style idempotency: consumer apps accept no idempotency keys, so on-screen state plus the task's evidence log are the only ground truth.
- A unique run ID in the evidence log before a side-effect step is the mobile equivalent of an idempotency key: any re-run can detect that this task already attempted it.
- Verify-before-act — checking the screen for "already done" before executing — turns a blind replay into a stateful re-entry that resumes from the first unproven step.
- The exception queue needs an evidence-based verdict for every item: safe to re-run, or needs human.
- Idempotency has hard limits: stale or cached UI, missing screenshots, expired run IDs, and human-verification steps all break the assumption that a check proves anything.

## Expert Explanation

### Why mobile double effects are structural, not accidental

In API automation, idempotency is a protocol feature. Stripe accepts an idempotency key per request and replays the original result on retry; AWS services accept a client request identifier and honor "at most once" semantics. A phone app offers none of that — no key header, no transaction ID, no endpoint to hand a token to — and the app itself is often not idempotent: a "Post" button posts again on every tap. The automation layer must absorb the idempotency duty itself, using the weakest tools in the industry: pixels on a screen.

Classify failures first, because re-run behavior depends on which one you hit. A step that fails before its side effect is safe to re-run blindly. A step that fails after the app confirmed the effect but before the task recorded it needs a different answer. The dangerous middle — a network timeout right after the tap — must never be re-run on a guess. Design around the last two, and the first takes care of itself.

### Pattern 1: Record a unique run ID in the task evidence log

Before any side-effect step, write a run ID — generated once per run, reused by every retry — into the evidence log. Re-runs check the log first: if the run ID is already present, this is a re-entry, not a fresh run, and the task resumes at the point of failure instead of starting over. It is the same principle as the client request identifier in AWS's idempotent API design, with the same bonus: the identifier makes the run auditable. Without an identity, the log cannot tell you whether two entries are the same event or two events; the run ID turns it into a dedupe key rather than a diary ([AI Agents Need Logs. Mobile Automation Needs Them Even More](/blog/ai-agent-logs-for-mobile-automation/)).

### Pattern 2: Verify before you act

Before every side-effect step, answer "did the effect already happen?" from the screen: confirm the post is not already live, the message is not already in the conversation, the order is not already claimed. Execute only when the check returns a clear "not yet." When the check is ambiguous — a loading spinner, a "processing" screen, a dialog that could mean either outcome — do not act; the step is not safe to re-run and must surface to the exception queue. This mirrors the [OWASP Mobile Application Security Verification Standard (MASVS)](https://mas.owasp.org/MASVS/), which treats verifiable evidence for critical actions — non-repudiation — as a baseline expectation, not a nicety.

### Pattern 3: Treat retries as stateful re-entry, not blind replays

A retry that carries its run ID and prior evidence is a resumption: load what the run already did, verify current state, and continue from the first step not proven complete — or stop if all are proven done. A blind replay assumes nothing happened, and that assumption is precisely what double-posts. Deciding when to retry and designing how a re-run behaves are separate problems: the first is retry policy, the territory of what happens when an [AI agent fails mid-task](/blog/ai-agent-fails-what-happens-next/); the second is the design work this article covers.

### Pattern 4: Give the exception queue a re-run signal

Most re-run decisions happen in a human review loop, and that loop scales only if evidence speaks a language it can act on. Every exception item should carry an evidence-based verdict: safe to re-run (the failure predates any side effect, or a verify check confirmed nothing happened) versus needs human (the effect is unconfirmed, the state ambiguous, or a check could not complete). Teams running a [control tower for mobile app workflows](/blog/ai-agent-control-tower-for-mobile-app-workflows/) find this signal is what makes [controlled takeover and human review](/blog/ai-agent-control-boundaries-cloud-phone-takeover/) practical at volume — a reviewer triages a thousand ambiguous items by verdict, not by opening each one. It matters more as agents gain autonomy: an agent that cannot tell "re-run is safe" from "re-run doubles the damage" should be stopped before it decides.

### Practical limits

Idempotency for UI-driven mobile work is bounded. Screens lie: an effect may not render immediately (async queues, caches), so a verify check can pass while the effect is still in flight — it proves screen state, not server state. Evidence can be incomplete: the screenshot captured before the confirmation rendered, the log lost when the device died. Run IDs expire: like Stripe pruning idempotency keys, any retention window lets an old, late retry slip past dedupe. Check-then-act races: two tasks on the same account can both verify "not done" and both act, so verification must be paired with evidence and, where possible, serialized per account. Some steps — human approval, CAPTCHAs, step-up authentication — must never be re-run by design; they are needs-human by definition ([permissions and audit trails for cloud phone automation](/blog/ai-agent-permissions-audit-trails-cloud-phone/)).

## Decision Framework

Set a default re-run posture per step type, then run the checklist before any re-run.

| Step type | Typical failure mode | Default re-run posture | Guard |
|---|---|---|---|
| Publish / post / submit (create) | Ambiguous failure after tap | Verify-before-act; stop if confirmed | Screen check + run ID evidence |
| Send message / DM | Effect confirmed, unrecorded | Verify conversation history before resend | Evidence log + screen check |
| Claim / redeem / order | Race risk; double-claim | Verify state; escalate on ambiguity | State check + needs-human signal |
| Edit / update / status change | Usually naturally idempotent | Safe to re-run | Log entry with run ID |
| Read / navigate / check | No side effect | Always safe | None required |

Checklist before any re-run:

- The step wrote its run ID to the evidence log before acting.
- A verify check confirmed the effect has not already occurred.
- The check result was unambiguous, or the step is flagged needs-human.
- No other task is running against the same account in parallel.

## Key Takeaways

- Idempotent automation is a design property, not a retry policy: build steps that re-run without double effects, then decide freely when to retry them.
- A unique run ID in the evidence log is the foundation — without it, a re-run cannot tell "already did this" from "never did this."
- Verify-before-act is the mobile substitute for server-side idempotency keys: the screen is the only transaction ID consumer apps give you.
- Retries are stateful re-entry: resume from the first unproven step, never replay from zero, and let every exception carry a safe-to-re-run / needs-human verdict.
- Treat idempotency as bounded: stale UI, incomplete evidence, expired run IDs, and check-then-act races define where a human must decide.

## FAQ

**Q: What does idempotent mean for a phone automation task?**

**A:** A step is idempotent when re-running it produces the same end state as running it once — one post, one message, one claim, not two. Mobile automation achieves this at the design level: run IDs in the evidence log and verify-before-act checks replace the idempotency keys server APIs would provide.

**Q: How can I know a step already happened if the app exposes no API?**

**A:** Verify from the screen before acting: is the post live, the message in the conversation, the order claimed? Pair that with the evidence log — a screenshot plus a run ID recorded before the step — so the check is auditable and a re-run resumes instead of replaying.

**Q: Is every retry of a task a re-run?**

**A:** No. A retry that carries the same run ID and prior evidence is a stateful re-entry: it resumes from the first step not proven complete. A blind replay starts from zero and assumes nothing happened — that assumption is what double-posts. Re-enter; never replay.

**Q: When is a step not safe to re-run, even with checks?**

**A:** When on-screen state is ambiguous (processing screens, dialogs), evidence is incomplete, the run ID has expired, two tasks could race on the same account, or the step requires human verification such as approval or CAPTCHA. In those cases the exception queue marks the item needs-human instead of allowing a re-run.

## Sources

- [AWS Builders' Library — Making retries safe with idempotent APIs](https://aws.amazon.com/builders-library/making-retries-safe-with-idempotent-APIs/): client request identifiers and at-most-once semantics.
- [Stripe API Reference — Idempotent requests](https://docs.stripe.com/api/idempotent_requests): idempotency keys for safe retries and key pruning.
- [OWASP Mobile Application Security Verification Standard (MASVS)](https://mas.owasp.org/MASVS/): baseline security expectations for mobile apps, including verifiable evidence for critical actions.
