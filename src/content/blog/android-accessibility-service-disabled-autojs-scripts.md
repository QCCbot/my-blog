---
title: AutoJS Script Stopped Working With No Error? Check the Accessibility
  Service First
description: AutoJS script stopped working with no error? "Autojs accessibility
  service not working" is usually the cause — check the service before the code.
  A triage guide for cloud phone ops teams.
pubDate: 2026-08-14
updatedDate: 2026-08-14
---

## Answer First

When an AutoJS script — or any on-device Android automation, including a cloud-phone platform's own agent — stops working with no error in the task logs, the first thing to check is not the code. It is the accessibility service. In the large majority of silent-failure cases the script is fine; the service it depends on is not.

**Definition:** The accessibility service is the Android system component that lets an automation app observe the screen and act on it: reading the UI hierarchy, clicking elements by text or id, scrolling, and typing. AutoJS and similar automation clients expose this as a service the user must explicitly enable in Settings → Accessibility. If the toggle is off, every screen-touching API call has nothing to operate on — selectors return nothing and clicks land nowhere.

**Why:** Android does not treat an accessibility service's enabled state as stable. A device restart, an app "Force stop," an aggressive OEM background process manager, an app update, or a permission reset on some ROMs can silently stop or disable it. When the service drops, the automation runtime usually cannot raise a proper script-level error: it throws a generic failure, returns "no matching element," or simply hangs. The task log shows a run that started and then produced nothing useful. Ops teams then spend hours re-debugging code that was never the problem.

**Example:** A nightly task opens a target app and taps "Confirm." It runs clean for weeks. After a scheduled cloud-phone instance reboot, the run log shows "started" and then "no matching element" — no stack trace. Re-running the script by hand appears to do nothing. Checking Settings → Accessibility shows the automation's service toggle is off. Re-enable it, re-run, and the task completes. The script never changed.

## Key Facts

- Accessibility services are user-enabled, system-bound components: they run only while enabled and while their host app process is alive.
- On Android, "Force stop" disables the hosting app's accessibility service until the user turns it back on.
- Device and cloud-phone instance restarts can silently drop the toggle on some devices and ROMs, especially under aggressive background process management.
- When the service is down, failures look like script failures: generic errors, empty results, "no matching element," or a hang — rarely a clear "service disabled" message.
- Correct triage order: verify service status, re-enable, restart the device if needed, then confirm in task logs. Re-debugging code comes last.
- The same failure mode hits AutoJS-style scripts and a platform's own on-device automation that relies on accessibility.

## Expert Explanation

**How the service actually runs.** An accessibility service is declared by the automation app and bound by the Android system when the user enables it. Once enabled, the system keeps the connection alive and delivers screen events to it. That design is the source of the troubleshooting trap: the automation runtime assumes the service exists, and most scripts never check the binding before calling UI APIs. When the binding is gone, the calls fail at the system boundary, not inside the script — so there is rarely a meaningful traceback. AutoJS-style clients expose a status check (in Auto.js, the `auto.service` object and `auto.waitFor()`), which is exactly what a startup guard should assert before touching the screen.

**Why Android silently kills it.** Several distinct mechanisms produce the same symptom:

- **Device restart.** Most devices restore enabled services after reboot, but some ROMs and cloud-phone images boot with accessibility toggles reset — particularly when the host app is started late or blocked from auto-start.
- **Force stop.** Android disables a stopped app's accessibility service until it is re-enabled. Anyone force-stopping the automation app — an operator, an OEM cleaner, an update flow — turns the service off.
- **Aggressive background management.** OEM battery savers and some cloud-phone ROMs kill host processes; a killed host means no service, and the ROM may prevent the system from rebinding it.
- **Updates and permission resets.** App updates or auto-reset behavior on newer Android versions can reset runtime permissions and, on some ROMs, the accessibility binding with them ([AI Agents Need Permissions and Audit Trails](/blog/ai-agent-permissions-audit-trails-cloud-phone/)).
- **Cloud-phone specifics.** Managed instances reboot, get re-provisioned, and do not always persist settings across a reset, so an entire fleet can lose its automation service with no operator touching a single device.

Accessibility is a high-privilege API, and the ecosystem treats it accordingly: Android's security guidance classifies such capabilities as risk areas, store policy requires apps to declare and justify accessibility-service use, and security standards explicitly flag accessibility services as a vector for leaking sensitive data. That scrutiny is why the platform is conservative about keeping the service alive — and why your automation should treat the service as infrastructure, not as a given. The flip side is that a fleet running automation with an accessibility service should keep that access scoped and auditable, since the capability itself is sensitive ([Agentic Automation Security](/blog/agentic-automation-security-cloud-phone-accounts/)).

**Why there is no error.** If the service drops before the run starts, the script's first selector call returns nothing and the run dies quietly. If it drops mid-run, calls start returning empty results until the script hits its own fallback — and what happens next depends on how your fleet handles a [failed task](/blog/ai-agent-fails-what-happens-next/). Either way, nothing on the device raises a "service disabled" exception by default, and the platform's [task log](/blog/ai-agent-logs-for-mobile-automation/) only records what the script reported. The error is genuinely absent — the failure lives one layer below the code.

## Decision Framework

Use this table to separate "service dropped" from "script bug" before touching a single line of code:

| Signal | Service dropped | Genuine script bug |
| --- | --- | --- |
| Failure begins after reboot, force stop, or re-provision | Likely | Possible |
| Error message | Generic, "no matching element," or hang | Specific traceback with line numbers |
| Same script works after re-enabling the service | Confirmed service issue | Not a service issue |
| Failure pattern | Every task on the device fails at once | Only one task or step fails |
| Code changed recently | Check service first anyway | Look at selectors, timing, and UI drift |

Verification checklist, in order:

1. **Check service status.** Open Settings → Accessibility and confirm the automation service shows "On." On a cloud phone, do this through the platform's remote view or via `adb shell settings get secure enabled_accessibility_services`.
2. **Re-enable.** Toggle the service off and on and accept the system confirmation prompt.
3. **Restart the device or instance** if the toggle resets again — and note whether the image persists settings across reboots.
4. **Confirm in task logs.** Re-run one minimal task and verify it completes; look for a clean run rather than a UI failure. The logs your platform keeps for task execution are the source of truth here — if you are not correlating device state with run results, that is the gap to close.
5. **Only then look at the script.** If the failure survives a confirmed-alive service, you now have a real code problem worth debugging.

**Practical limits.** This guide covers service-state failures. It does not cover genuine code bugs, selector drift when a target app changes its UI, root- or ADB-based automation that never needed an accessibility service, or ROM-specific quirks. Re-enabling the service is not a guarantee: if the service drops again immediately, the underlying cause is the ROM, the image, or the process manager, not the toggle. Some cloud-phone images require re-enabling per instance after every reboot, and fleet-wide "all tasks fail at once" should be treated as a [device-state incident](/blog/ai-agent-control-tower-for-mobile-app-workflows/), not a script incident.

## Key Takeaways

- When a script fails with no error, check the accessibility service before the code.
- Learn the service-drop signature: generic errors, empty selectors, hangs, and whole-fleet failure.
- Triage order: status, re-enable, restart, logs, then code.
- Treat the service as infrastructure: verify it after every reboot, force stop, or re-provision.
- If every task on a device fails at once, the device is the problem — debug the fleet state, not the script.

## FAQ

Q: How do I tell if the accessibility service is off rather than my script failing?
A: Check the enabled accessibility services in device settings (or the platform's device view). If the toggle is off, the cause is the service. Re-enable it and re-run one minimal task: if it completes, the script was never the problem.

Q: Why does the service keep turning off after reboot?
A: Some ROMs and cloud-phone images do not restore enabled services at boot, and aggressive background managers can prevent the system from rebinding the host app. If the toggle resets on every reboot, the fix is at the image or ROM level, such as persisting settings and whitelisting the host app from battery optimization.

Q: Will re-enabling the service always fix my script?
A: No. Re-enabling only fixes service-state failures. If the failure recurs with a confirmed-alive service, investigate the script, selectors, or the target app's UI. If it recurs right after re-enabling, look at the ROM or process manager rather than the toggle.

Q: Do cloud phones behave differently from physical Android devices?
A: They run the same Android accessibility stack, but instances reboot and get re-provisioned more often, and some cloud-phone ROMs are aggressive about background processes. Service drops are therefore more frequent and typically take down every task on the instance at once.

## Sources

- Android Developers — AccessibilityService (how services are declared, enabled, and bound by the system): https://developer.android.com/guide/topics/ui/accessibility/service
- Google Play Console Help — AccessibilityService APIs and usage disclosure (store requirement to declare and justify accessibility-service use): https://support.google.com/googleplay/android-developer/answer/9888077
- OWASP MASVS / MASWE-0040 — Sensitive data leaked via accessibility services: https://mas.owasp.org/MASVS/
