---
title: How to Design a Cloud Phone Task Evidence Log
description: A practical framework for building an immutable, reviewable
  evidence trail for cloud phone automation — screenshots, timestamps, and
  structured log entries that let operations teams prove what ran, when, and
  what the phone showed.
pubDate: 2026-07-28
updatedDate: 2026-07-28
---

## Answer First

**Definition:** A cloud phone task evidence log is an append-only, cryptographically verifiable record of what happened on a managed Android device during an automated task run — every screenshot captured, every UI element inspected, every assertion passed or failed, every timeout, and every human review decision — each entry stamped with a monotonic timestamp and a content hash that links it to the prior entry.

**Why:** [Traditional automation logs](/blog/ai-agent-logs-for-mobile-automation/) record what the *orchestrator* did — the command issued, the exit code, and perhaps stdout. They do not capture what the *phone showed*. When a task runs on a cloud phone — tapping buttons, reading SMS messages, filling forms — the phone's screen state is the ground truth. An evidence log makes that ground truth durable, immutable, and reviewable so an operations team can reconstruct the entire session from stored proof alone, without trusting the device's volatile state.

**Example:** Consider a daily reconciliation task that logs into a fintech app on a cloud phone, checks transaction history, exports a CSV, and emails it. A plain task log shows: `adb shell input tap 540 1200` → `exit 0`. When the [AI agent fails mid-task](/blog/ai-agent-fails-what-happens-next/), that exit code tells you nothing about what the phone displayed. An evidence log shows: a screenshot of the login screen with an "authenticated" badge and a trusted-timestamp assertion; a screenshot of the transaction list with OCR-extracted row counts; a checksum of the exported CSV; and a human reviewer's sign-off on the final email. Months later, when an auditor asks "did the reconciliation run, and what data did it see?", the evidence log answers without re-running the task.

## Key Facts

- **Evidence is not logs.** Task logs capture *execution intent*; evidence logs capture *observable outcome*. The two complement each other but serve different review and compliance purposes.
- **Immutable storage is the baseline.** The evidence store must prevent retroactive deletion or modification by any party — including the automation system that wrote it.
- **Screenshots alone are not enough.** A screenshot without a trusted timestamp, a device identifier, and a link back to the originating task run is an orphaned image. Evidence entries must be self-describing.
- **Human review boundaries must be recorded.** When a task pauses for human judgment, the evidence log captures what the human saw, what decision they made, and when they made it. This turns the review step from a process gap into an auditable [control point for agentic automation security](/blog/agentic-automation-security-cloud-phone-accounts/).
- **Retention policies are a design constraint.** A fleet of 100 devices running 50 tasks per day generates tens of gigabytes of evidence per month. Storage tiering, compression, and lifecycle rules are not optional bolt-ons.
- **Content hashing creates verifiable chains.** Each evidence entry carries a SHA-256 hash of its own content and a pointer to the prior entry's hash, making retroactive edits detectable without a central authority.

## Expert Explanation

Designing a cloud phone task evidence log means solving three problems: capture, storage, and verification.

### Capture Layer

The capture layer runs on the orchestrator side, not on the phone — a recycled phone cannot be trusted to store its own evidence. The orchestrator issues a command, waits for the result, and *then* captures proof of what the phone displayed.

Capture happens at every decision boundary:

1. **Before an action** — a screenshot of the starting state, with an OCR text dump of visible UI elements.
2. **After an action** — a screenshot of the resulting state, plus the assertion that was checked (e.g., "element with text 'Success' is visible").
3. **On timeout or error** — a screenshot of the error state, the raw `adb` or instrumentation output, and a reason code.
4. **At human review stops** — the screen state shown to the reviewer, the reviewer's decision, and their session identifier.

Each capture produces an evidence entry: a JSON object containing a screenshot file reference (or base64), structured metadata, a monotonic timestamp, the task run ID, the step name, the content hash, and the hash of the preceding entry.

### Storage Layer

The storage layer is a write-once log. Append-only databases or object storage with S3 Object Lock or GCS Bucket Lock are viable when combined with an index that maps task runs to their evidence keys.

| Requirement | Implementation | Why |
|---|---|---|
| Append-only writes | Object lock (WORM) or append-only DB table | Prevents deletion or overwrite |
| Content addressing | SHA-256 hash as storage key | Enables deduplication and verification |
| Cross-region durability | Replicate the evidence store to a second region | Survives regional outages and supports audit independence |
| Lifecycle tiering | Hot → Cold → Glacier at 30/90/365 days | Controls storage cost without losing compliance access |

The evidence index — mapping `(task_run_id, step_index) → evidence_entry` — is the primary query interface. Operations teams search by task run, device, date range, or assertion failure reason code. The index is rebuilt from the raw evidence store on disaster recovery; the raw entries are the source of truth.

### Verification Layer

Verification is a separate, read-only process — a CI/CD pipeline or scheduled job — that re-computes the hash chain of every evidence entry in a time window and reports any break. The chain is anchored to the first entry's hash, so any discrepancy flags tampering immediately.

For stronger guarantees, the hash of the latest entry can be periodically published to a SIEM or transparency service, making it possible to prove the log has not been altered since the last publication — even if the evidence store itself is compromised.

### Practical Limits

- **Storage cost:** At 200 KB per screenshot and 50 task steps per run, a single run consumes roughly 10 MB. A 100-device fleet at 50 runs per day generates 50 GB daily. Plan for compression and configurable capture frequency — not every step needs a screenshot.
- **Throughput:** Batch evidence writes every 500 ms or at step boundaries to avoid slowing task execution at fleet scale.
- **Clock skew:** Use a monotonic sequence counter scoped to the task run, not wall-clock time, as the canonical ordering key.
- **Hash churn:** Verify incrementally — check only entries added since the last verification run, and anchor the chain head periodically.

## Decision Framework

When evaluating whether your cloud phone automation stack has adequate evidence logging, ask these five questions in order:

1. **Can you prove the phone's state at every decision point?** If you only have task runner logs, you cannot. You need screenshots, OCR extracts, or instrumentation dumps captured before and after each action.
2. **Is the evidence store append-only and immutable?** If the automation system or any operator can delete or edit past entries, the log is not an audit trail. Look for write-once storage, object lock policies, or cryptographic hash chaining.
3. **Can you reconstruct a complete session from stored evidence alone?** Cloud phones are ephemeral. If your evidence depends on data that lives only on the device, that evidence vanishes when the phone is recycled.
4. **Are human review decisions captured as evidence entries?** Manual approvals are control points — they give operations teams a [controlled takeover mechanism](/blog/ai-agent-control-boundaries-cloud-phone-takeover/) when the automation reaches an unexpected state. If the reviewer's decision lives only in a ticketing system or Slack thread, it is disconnected from the evidence.
5. **Is the evidence verifiable by an independent party?** A hash chain or content-addressed store lets auditors validate integrity without access to the automation system itself.

If you answered "no" to any of these, start by fixing the capture layer — that is the highest-leverage change — then work up through immutability, session reconstruction, human-review capture, and independent verification.

## Key Takeaways

- A cloud phone task evidence log is distinct from a task execution log. It captures observable phone state at every decision boundary, not just orchestrator commands and exit codes.
- Immutability is non-negotiable — enforce it at the storage layer with write-once semantics, object lock policies, or cryptographic hash chains.
- Screenshots must carry provenance metadata: device ID, task run ID, step index, sequence counter, and content hashes. An orphaned image is not evidence.
- Human review decisions are part of the evidence trail. Record what the reviewer saw, what they decided, and when — in the same append-only store as the automated steps.
- Design for verification from day one. A hash chain built into the entry format lets any party validate integrity without trusting the storage backend.
- Storage lifecycle management is a first-class concern. Tier evidence from hot to cold storage as it ages, and never delete entries that overlap with open investigations or audit windows.

## FAQ

**Q: How long should a cloud phone task evidence log be retained?**
A: Retention depends on compliance scope. For SOC 2/ISO 27001, keep raw evidence for at least 90 days and indexes for one year. Regulated verticals may mandate three to seven years. Design the store to support tiered lifecycle policies and never delete evidence overlapping an active investigation.

**Q: Can evidence logs debug a failed task after the phone is recycled?**
A: Yes, if capture ran before the failure. Every step should emit proof incrementally. When a phone is recycled, the evidence log survives because it lives in durable backend storage — not on the device.

**Q: What is the difference between a task log and an evidence log?**
A: A task log records what the automation engine did (commands, exit codes, stdout). It is optimized for debugging. An evidence log records what the phone showed and what was concluded at each decision point. It is optimized for audit. Evidence logs are immutable and retained longer.

**Q: Do screen recordings count as sufficient evidence?**
A: Rarely. Video is hard to index and hard to verify as untampered. A stronger approach captures discrete evidence frames at each decision boundary — screenshots with timestamped metadata, OCR extracts, checksums — written to an append-only store.

## Sources

- Android Security & Privacy Risks overview — Android Developers. [https://developer.android.com/privacy-and-security/risks](https://developer.android.com/privacy-and-security/risks)
- OWASP Mobile Application Security Verification Standard (MASVS). [https://mas.owasp.org/MASVS/](https://mas.owasp.org/MASVS/)
- Google Play Developer Policy — User Data and Privacy. [https://support.google.com/googleplay/android-developer/answer/9888077](https://support.google.com/googleplay/android-developer/answer/9888077)
