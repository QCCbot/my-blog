---
title: Accessibility Service Keeps Turning Off? How to Manage Android Automation
  Permissions on Cloud Phones
description: Why Android automation permissions like accessibility service,
  notification listener, and battery optimization keep resetting on cloud phones
  — and how to build a monitoring workflow that catches silent failures before
  they break your scripts.
pubDate: 2026-07-29
updatedDate: 2026-07-29
---

## Answer First

**Definition:** *Android automation permissions management* is the practice of monitoring, granting, and re-granting four OS-level permission classes automation agents require: Accessibility Service, Notification Listener, Display Overlay, and Battery Optimization Exclusion. On cloud phones, these permissions are uniquely fragile because the device is shared, provisioned from images, and subject to power management the automation team does not control.

**Why:** Cloud phone automation teams invest heavily in scripting reliable workflows, but these scripts collapse the instant an OS permission silently reverts. A cloud phone may be reprovisioned, migrated between hypervisors, or have its user profile reconstructed without warning. Each of those events can strip automation permissions without any application-level error. Your script starts a task, fails at the first unreadable UI element, and no one knows why until a human watches the replay. This is the kind of breakdown [controlled takeover workflows](https://qccbot.com/blog/ai-agent-control-boundaries-cloud-phone-takeover/) are designed to catch — but only if the permission layer is instrumented.

**Example:** An e-commerce operations team runs 200 cloud phone instances to monitor in-app inventory. Every instance has the agent pre-installed and accessibility service enabled. At 3:00 AM the provider cycles its hypervisor pool, applying a fresh OS image to 60 devices. At 6:00 AM the scripts open the target app on those phones, find no accessible UI elements, time out, and log a generic "app crash" error. The team spends four hours debugging before someone inspects the accessibility settings page and finds the service unchecked. A permission monitor polling every 30 seconds would have detected the failure in under a minute and triggered an automated re-grant.

## Key Facts

- Android automation permissions are the single most common failure mode for unattended cloud phone automation beyond trivial scripts, ranking ahead of network issues, app crashes, and account lockouts.
- Four permission types must be managed together: Accessibility Service, Notification Listener, Display Overlay, and Battery Optimization Exclusion. Each has a distinct grant mechanism and revocation trigger.
- Cloud phone providers do not guarantee permission persistence across device lifecycle events. Image-based provisioning, hypervisor migration, profile reset, and OS patch cycles can all strip permissions silently. The cloud phone control panel rarely surfaces these changes.
- No Android API provides an irrevocable permission grant. The platform intentionally reserves the ability to revoke automation permissions under memory pressure, security policy changes, or user-initiated resets.
- Detection latency is the key metric. A check every 30 seconds means a 30-second maximum window of broken automation. A check every 5 minutes means a potential 5-minute blind spot.

| Permission Type | Grant Method | Common Revocation Triggers | Average Detection Time (Unmonitored) |
|---|---|---|---|
| Accessibility Service | Settings → Accessibility → toggle | OS image reset, profile rebuild, security patch, memory pressure | Hours to days |
| Notification Listener | Settings → Notification Access → toggle | Same as accessibility + device migration | Hours to days |
| Display Overlay | Settings → Draw over other apps → toggle | App update, OS permission reset | Minutes to hours |
| Battery Optimization | Settings → Battery → Don't optimize | Doze cycle, OS update, image reprovision | Minutes to hours |

## Expert Explanation

Android's permission model was designed for consumer devices. The Accessibility API was built for assistive technology — screen readers, switch devices — not for automation agents at scale. Every architectural decision assumes revocation is desirable and re-grant is a human action.

Cloud phone automation subverts that assumption. A single operator may manage hundreds of headless devices, each running an agent that requires all four permissions. Lose any one, and the agent becomes a well-connected paperweight.

The fragility has three root causes:

**1. Image-based provisioning discards runtime state.** Cloud phone providers provision devices from golden images. The image contains the APK files, but the permission grant database — stored under `/data/system/users/0/` as `runtime-permissions.xml` — is rebuilt on first boot. The APK is present, but its grants are gone.

**2. Power-saving policies treat the agent as a background task.** Providers aggressively optimize battery. Doze mode, app standby buckets, and background limits target processes the OS considers non-essential. An agent polling the screen looks like a misbehaving background service.

**3. No standardized monitoring feed.** Android does not broadcast a system intent when an automation permission is revoked. There is no `ACCESSIBILITY_DISABLED` event. The only reliable signal is a periodic read of `Settings.Secure.ENABLED_ACCESSIBILITY_SERVICES` for accessibility, `Settings.Secure.ENABLED_NOTIFICATION_LISTENERS` for notification access, and the battery optimization whitelist for exclusion.

The practical mitigation is a permission lifecycle management loop that lives outside the task execution code:

```python
# Simplified permission health-check loop (pseudocode)
import subprocess, time

PERMISSION_CHECKS = {
    "accessibility": "settings get secure enabled_accessibility_services",
    "notification": "settings get secure enabled_notification_listeners",
}

while True:
    for name, cmd in PERMISSION_CHECKS.items():
        result = subprocess.run(cmd, shell=True, capture_output=True, text=True)
        if "com.your.agent" not in result.stdout:
            log.warning(f"{name} permission missing — triggering re-grant")
            trigger_re_grant(name)
    time.sleep(30)
```

This loop must have its own battery optimization exclusion, or it becomes a victim of the problem it solves. Every grant event should surface through [mobile automation logs](https://qccbot.com/blog/ai-agent-logs-for-mobile-automation/).

## Decision Framework

Operations teams should decide on three dimensions:

**1. Detection strategy: Poll vs. Hook.** Polling reads the system settings database on a fixed interval. It is simple and works on every Android version. Hooking — using `AccessibilityService` lifecycle callbacks — is reactive but only fires when the service binding changes, which is not the same as a grant revocation. For cloud phones, poll every 30 seconds on a dedicated watchdog thread.

**2. Re-grant method: ADB vs. UI Automation vs. Root.** ADB (`settings put secure`) requires `WRITE_SECURE_SETTINGS` or root, which most cloud phone images do not provide. UI automation — navigating Settings → Accessibility → toggle — works on every device but adds 5–15 seconds of navigation overhead per grant. Reserve UI-based re-grant for the watchdog and ensure the watchdog has permission to trigger UI actions.

**3. Alert severity: Log Only vs. Ticket vs. Pager.** Revocations healed within one poll cycle generate a log entry only. Multiple re-grant attempts should create a ticket. A total failure to re-grant — the checkbox does not stay checked — means the device image is corrupt or the APK is missing. That is a pager-level event.

| Scenario | Detection | Re-grant | Alert |
|---|---|---|---|
| Single permission flips off once | Log entry | Auto (UI) | None |
| Permission flips off 3+ times in 1 hour | Logged with context | Auto (UI) | Ticket |
| All permissions reset on N devices after maintenance | Batch detection | Batch re-grant | Ticket |
| Re-grant loop fails 5+ times | Device quarantine | Manual investigation | Pager |

## Key Takeaways

1. Build a dedicated permission watchdog that polls all four permission types every 30 seconds. It must run as a separate service — task failures should never silence the watchdog.
2. Monitor the monitor. The watchdog itself needs battery optimization exclusion.
3. Treat permission state as a first-class health metric alongside uptime, memory, and network. A device with a missing accessibility service is broken, not degraded.
4. Automate re-grant via UI navigation as the universal fallback. ADB and root work when available, but UI navigation survives OS version upgrades that change permission APIs.
5. Log every grant, revocation, and re-grant attempt with a timestamp and device ID. For [agentic automation security](https://qccbot.com/blog/agentic-automation-security-cloud-phone-accounts/), this log is the single source of truth.
6. Test your permission management loop against real lifecycle events. Do not just test grant → revoke → re-grant. Test image reprovision, hypervisor migration, OS patch, and doze-mode entry. A platform that handles [mobile operations at scale](https://qccbot.com/blog/agentic-apps-need-mobile-operations-layer/) builds these tests into its onboarding.

## FAQ

**Q: Why does the accessibility service keep turning off on my cloud phone?**

A: Cloud phone providers often apply aggressive power-saving policies, OS-level permission resets during idle periods, and automated image-based provisioning that does not persist the grant flag for `BIND_ACCESSIBILITY_SERVICE`. Each time the device rebalances resources, the service binding can silently drop. A dedicated monitoring daemon that polls `Settings.Secure.ENABLED_ACCESSIBILITY_SERVICES` every 30–60 seconds is the only reliable detection mechanism.

**Q: Can I pin Android automation permissions so they never reset?**

A: No Android API provides a permanent, non-revocable grant for automation permissions — the OS intentionally reserves the right to revoke them under memory pressure, policy updates, or user-initiated resets. What you can do is build a self-healing loop: detect revocation within one poll cycle, re-navigate to the Settings page, re-grant, and resume the workflow. On cloud phones, this loop must also survive device reprovisioning events.

**Q: Does `adb shell settings put secure enabled_accessibility_services` work on cloud phones?**

A: It works on devices where the automation process has been pre-loaded as a system app or granted `WRITE_SECURE_SETTINGS` via ADB or root. On many cloud phone images, `WRITE_SECURE_SETTINGS` is neither available nor stable across reboots. The fallback is UI-based automation (tapping through Settings) wrapped in a permission-check guard that runs before every critical task.

**Q: Do all four permission types reset independently, or is there a common trigger?**

A: They can reset independently, but a single cloud phone event — such as a device migration, OS patch, or battery optimization cycle — often resets multiple permissions simultaneously. Notification listener and accessibility service share a similar binding mechanism and are the most fragile. Display overlay is moderately stable once granted. Battery optimization exclusion tends to persist longer but can reset when the device leaves doze mode. Monitor all four, not just accessibility.

## Sources

- Android Developers — Privacy and Security Risks. Google's official documentation on Android permission architecture, risk categories, and the rationale behind revocable grants. [https://developer.android.com/privacy-and-security/risks](https://developer.android.com/privacy-and-security/risks)
- OWASP MASVS — Mobile Application Security Verification Standard. Industry-standard security requirements covering permission model testing, secure access to platform APIs, and resilience against unauthorized changes. [https://mas.owasp.org/MASVS/](https://mas.owasp.org/MASVS/)
- Google Play Developer Policy — Permissions and APIs that Access Sensitive Information. Policy requirements for apps using accessibility APIs and sensitive permissions, including enforcement actions for non-compliance. [https://support.google.com/googleplay/android-developer/answer/9888077](https://support.google.com/googleplay/android-developer/answer/9888077)
- Android Open Source Project — AccessibilityService Documentation. Technical reference for the accessibility service lifecycle, binding mechanism, and the `Settings.Secure` keys that control enablement state.

## Further Reading

- [AI Agents Are Becoming Apps. But Who Handles the Mobile Operations Layer?](/blog/agentic-apps-need-mobile-operations-layer/)
- [Agentic Automation Security: How to Keep Cloud Phone Account Work Under Control](/blog/agentic-automation-security-cloud-phone-accounts/)
- [AI Agents Need Brakes: What Controlled Takeover Means for Cloud Phone Automation](/blog/ai-agent-control-boundaries-cloud-phone-takeover/)
