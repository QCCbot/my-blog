---
title: Your AutoJS Script Works on One Phone. Why Does It Fail on Another?
description: An AutoJS script works on one phone but not another because of
  screen size, Android version, and OEM skins. Here's how to diagnose,
  pre-check, and harden scripts for your fleet.
pubDate: 2026-08-04
updatedDate: 2026-08-04
---

## Answer First

**Definition:** "My AutoJS script works on one phone but not another" is the situation where identical script code, running against the same app version, produces different results — or outright failure — on a second device. Nothing changed in the script and nothing changed in the app. The device changed.

**Why:** AutoJS interacts with a phone's UI through three device-dependent mechanisms: pixel coordinates, screen-relative resources, and accessibility nodes. Screen size, resolution, and DPI move and resize every element on screen. The Android version changes accessibility behavior and permission flows. OEM UI skins — MIUI/HyperOS, ColorOS, OriginOS, One UI, EMUI/HarmonyOS, and others — rename or restructure UI elements, draw their own status bars and floating windows, and aggressively manage background processes. A script written and tested on one phone quietly encodes that phone's rendering: its resolution, its node tree, its timing. Run it anywhere else and those assumptions surface.

**Example:** A script taps "Confirm" at a fixed coordinate — say (540, 1800) — tuned on a 1080×2400 test phone. On a 720×1600 or 1080×2160 device, that button lives at different pixel coordinates, so the tap lands on empty space and the script times out. Or consider a selector like `text("Continue")`: it matches on a stock Android build but returns nothing on an OEM skin that renders the same button with different wording or a missing content description. Same code, same app version — different device, different result.

This is not an app update breaking your selectors on every device at once, and it's not a hardware failure that a replacement checklist fixes. It's steady-state variance across a heterogeneous fleet — and it's exactly the problem that QCCBot's device groups, evidence log, and monitoring exist to make visible instead of mysterious.

## Key Facts

- **Three independent variables decide whether a script runs:** screen size/resolution/DPI, Android version, and OEM UI skin. Each one changes what AutoJS sees, so they compound when a fleet mixes all three.
- **Coordinates are a resolution bet.** Fixed pixel taps and hard-coded `bounds()` comparisons silently miss on devices with different resolutions or densities. Screenshots captured for image matching also come back at different sizes and DPIs.
- **Selectors are an accessibility-tree bet.** `text()`, `desc()`, and `id()` selectors resolve against a node tree that Android versions and OEM skins build differently — labels get reworded, content descriptions get dropped, resource IDs get remapped.
- **Timing and lifecycle are a behavior bet.** Slower devices miss `findOne()` timeouts, and OEM battery managers can freeze or kill the automation service mid-run on some phones and not others.
- **AI-generated scripts amplify the problem.** Models generate AutoJS against a single captured screen, so they tend to bake in one device's resolution and node structure — then fail quietly across a fleet.
- **Not to be confused with:** app-side UI changes (a new app build breaks selectors fleet-wide at once, the subject of our app-UI-change articles) or device-replacement events (a swapped phone needs re-setup). This is the everyday middle ground: a fleet that runs fine, one device at a time, but not consistently.

## Expert Explanation

### The three layers where fleet scripts break

**Coordinates.** AutoJS `click(x, y)` and `swipe()` work in raw screen pixels. On a 1080×2400 phone a button at y=1800 is in the lower third; on a 720×1600 phone the same physical button might sit near y=1200 — or a completely different spot if the OEM's status bar and navigation gestures shift the layout. Image-matching helpers (`findImage`, `findColor`) have the same problem: a template captured on one resolution either won't match at another or will match the wrong region. Android's own guidance on supporting different screen sizes exists precisely because no two devices render the same layout identically.

**Selectors.** When you use `text("Continue")` or `desc("Search")`, AutoJS walks the accessibility node tree. That tree is built by the app under the constraints of the Android version and the OEM's accessibility layer. A node that exists on one device can be missing, renamed, or flattened into a parent node on another. This is why a selector-based script can fail on a single OEM skin while working everywhere else — the tree itself differs.

**Timing and lifecycle.** The same script that completes in four seconds on a flagship can time out on a budget device, and OEM battery optimizers can suspend the automation service entirely when the screen is off or the app is backgrounded. The failure signature — "worked for five minutes, then died" — is a device-behavior problem, not a script-logic problem.

### Why "tested on one phone" is a trap

Scripts are born on a single device: you install the app, record the flow, tune the coordinates, and it works. What you've actually verified is "this script works on *this* phone." Every hard-coded coordinate, every selector that matched by luck, and every timeout tuned to one device's speed is a latent assumption. The trap is that the first deployment masks it: a small batch on identical phones passes, and the fleet's heterogeneity only shows up at scale, where failures scatter across devices with no obvious pattern. AI-generated scripts make this worse — they're written against one screenshot, with no awareness of any other screen.

### Diagnose device-vs-script causes (evidence first)

Before changing anything, capture evidence. For every failed run, record the device model, Android version, resolution, DPI/density, AutoJS app version, and accessibility status — then a screenshot or screen recording at the moment of failure. QCCBot's evidence log is built for this: each task run can carry the device info and visual evidence alongside the outcome, so "why did this fail?" is answerable from the record instead of from memory. This is the same discipline as [AI agents needing logs](https://www.qccbot.com/blog/ai-agent-logs-for-mobile-automation/), applied to the device layer: a failure without device context is a failure you can't classify.

Then diff. Compare a failing device against the device where the script works, one variable at a time. If failures cluster by resolution class, it's coordinate drift. If they cluster by Android version, it's accessibility behavior. If they cluster by OEM, it's the skin. If they're spread across everything, suspect the script or the AutoJS app version itself — mixing AutoJS releases with different script engines adds a fourth variable you can't see from the script alone.

### Pre-check the fleet before a batch run

The cheapest fix is to not discover variance mid-batch. Before scheduling a run, inventory the target devices: model, screen size, resolution, DPI, Android version, and any per-device settings the script depends on (autostart exemptions, battery optimization exclusions, notification access). QCCBot's device groups let you scope runs to a defined set of managed devices, so the pre-check becomes "confirm the group's specs against the script's assumptions" instead of hoping. Google Play's device catalog exists for the same reason on the app side: you can't reason about a fleet you haven't enumerated. Devices that don't meet the script's baseline belong in a different group or an explicit exclusion — not in the batch.

### Write scripts that survive the fleet

- **Prefer selectors over coordinates.** `text()`, `desc()`, and `id()` match against the accessibility tree and are resolution-independent. When a selector is ambiguous, narrow it with `boundsWithin()` rather than falling back to raw pixel taps.
- **Keep coordinates relative.** If you must tap or swipe, derive positions from `device.width` and `device.height`, or use `setScreenMetrics()` to normalize a target resolution — never hard-code pixels tuned on one phone.
- **Poll instead of assuming.** Replace single `findOne(timeout)` calls with `exists()` polling loops and generous timeouts, so slow devices fail slowly and visibly instead of randomly.
- **Add fallback chains.** If the primary selector finds nothing, try an alternative (`desc()` fallback for `text()`, a sibling node, a relative position) before giving up.
- **Standardize the engine.** Pin the AutoJS app version across the fleet so engine differences don't masquerade as device differences.
- **Per-class thresholds.** Image-matching confidence and verification tolerances should be configurable per spec class, not global.

The operational side — scoping runs to [device groups, monitoring, and human review](https://www.qccbot.com/blog/ai-agent-control-tower-for-mobile-app-workflows/) — is what turns "some runs failed" into "this spec class fails at this step, here's the evidence."

### Practical limits — what hardening can't guarantee

Robust selectors reduce variance but never eliminate it. OEMs can reword text, drop content descriptions, or render custom fonts that defeat OCR-based matching. Two devices in the same spec class can still differ in subtle ways — a different system font, an extra status-bar icon shifting a layout, a floating window intercepting taps. And no configuration produces pixel-identical rendering across skins, so screenshot-verification thresholds will need per-class tuning and occasional adjustment. Pre-checks and relative selectors shrink the problem; monitoring and human review are the backstop that catches what they miss.

## Decision Framework

When a run fails on some devices, work through this order — it prevents both "rewrite the script" and "blame the phone" reactions:

1. **Confirm it's not app-side.** Check whether the app version is uniform and whether the failure is fleet-wide. If it broke everywhere at once, that's an app update, not device variance.
2. **Pull the evidence.** Compare device info and failure screenshots from the evidence log across failing and passing devices.
3. **Classify by spec.** Group failures by resolution, Android version, and OEM. Clustering reveals the cause; random scatter suggests a script or engine issue.
4. **Choose the fix.** Script fix (relative selectors, polling), device fix (autostart, battery exemptions, permissions), or fleet fix (regroup, exclude, reschedule with a different class). When a run fails mid-task, treat the aftermath as a [failure-handling question](https://www.qccbot.com/blog/ai-agent-fails-what-happens-next/) with evidence attached.
5. **Verify and monitor.** Re-run on the failing class, then watch the next batch for new clusters — and keep the audit trail so [permissions and review stay accountable](https://www.qccbot.com/blog/ai-agent-permissions-audit-trails-cloud-phone/).

| Failure signature | Most likely cause | First check |
|---|---|---|
| Tap misses or hits the wrong element on some devices | Resolution/DPI coordinate drift | Compare screen size and density; switch to selectors or relative coordinates |
| `text()`/`desc()` finds nothing on one OEM only | OEM skin renamed or restructured the node tree | Pull the accessibility tree + screenshot at failure; add a fallback selector |
| Script stops mid-run on some phones, fine on others | OEM battery manager / autostart killed the service | Check device battery and autostart settings; standardize exemptions |
| "Timeout" / element not found, mostly on budget devices | Slow device exceeded tuned timeouts | Raise timeouts; replace `findOne` with `exists()` polling |
| Same code behaves differently across two phones with same specs | Mixed AutoJS app versions/engines | Standardize the AutoJS version across the fleet |

## Key Takeaways

- The same script and app version failing on another phone is normally device-side variance — screen size/DPI, Android version, or OEM skin — not a broken script.
- Hard-coded coordinates and tuned timeouts are assumptions about the device you tested on. Selectors, relative positioning, and polling are the portable equivalents.
- Diagnose with evidence: device model, Android version, resolution, DPI, AutoJS version, and a failure screenshot. Compare failing vs. passing devices before changing anything.
- Pre-check fleet specs before batch runs and scope scripts to device groups with matching spec classes; exclude or regroup devices that don't meet the baseline.
- AI-generated AutoJS is especially prone to single-device assumptions — treat "it works on my phone" as the start of validation, not the end.
- Monitoring and human review remain the backstop: hardened scripts shrink the variance problem, but heterogeneous fleets never become homogeneous.

## FAQ

**Q: Why does my AutoJS script work on one phone but not another when the app version is identical?**
**A:** The script and app didn't change — the device did. AutoJS drives the UI through pixel coordinates, screen-relative resources, and accessibility nodes, and all three vary by device. Resolution and DPI move elements and change screenshot sizes; the Android version changes accessibility behavior and permission flows; OEM skins rename or restructure UI elements and manage background processes differently. Your script was effectively written against one phone's rendering, so it carries that phone's assumptions with it.

**Q: Should I use coordinates or selectors in AutoJS?**
**A:** Prefer selectors. `text()`, `desc()`, `id()`, and `bounds()`-based matching against the accessibility node tree survives resolution differences far better than fixed pixel coordinates, because nodes are described in a resolution-independent way. When you must use coordinates, keep them relative (for example, `device.width / 2` or `setScreenMetrics()` normalization) instead of hard-coded pixel values. Coordinates alone are the most common reason a script that works on one phone fails on another.

**Q: Why does my script stop mid-run on some phones but not others?**
**A:** The usual suspects are OEM battery managers and autostart restrictions. Skins like MIUI/HyperOS, ColorOS, and EMUI aggressively freeze or kill background services, including the automation service the script depends on, and many devices require manual autostart and battery-exemption settings. Slow devices can also exceed your `findOne` timeouts. Standardize those device settings across the fleet and raise timeouts with `exists()` polling before you suspect the script itself.

**Q: How do I tell whether the device or the script is at fault?**
**A:** Compare evidence, not assumptions. Capture the device model, Android version, resolution, DPI, AutoJS app version, and a screenshot at the moment of failure, then diff a failing device against the device where the script works. If the failure clusters by spec class (one resolution, one Android version, one OEM), it's device-side variance. If it fails everywhere at once after an app update, it's app-side. That distinction decides whether you fix the script, fix the device settings, or adjust the fleet configuration.

## Sources

- **Support different screen sizes and densities** — Android developer documentation on why layouts and coordinates vary across screen sizes, resolutions, and DPIs: https://developer.android.com/guide/practices/screens_support
- **Build accessible apps** — Android developer documentation on accessibility services and how apps expose UI to automation through the accessibility tree: https://developer.android.com/guide/topics/ui/accessibility
- **Set up your device catalog** — Google Play Console documentation on testing apps across a defined set of device configurations before release: https://support.google.com/googleplay/android-developer/answer/9888077
- **Android distribution dashboard** — Google's public breakdown of device characteristics across the Android ecosystem, showing why fleet heterogeneity is the norm, not the exception: https://developer.android.com/about/dashboards

For the broader operations context: [the mobile operations layer for AI agents](https://www.qccbot.com/blog/agentic-apps-need-mobile-operations-layer/) is where device-level variance like this gets managed, and it starts with [logs](https://www.qccbot.com/blog/ai-agent-logs-for-mobile-automation/) and [audit trails](https://www.qccbot.com/blog/ai-agent-permissions-audit-trails-cloud-phone/).

## Further Reading

- [AI Agents Are Becoming Apps. But Who Handles the Mobile Operations Layer?](/blog/agentic-apps-need-mobile-operations-layer/)
- [Agentic Automation Security: How to Keep Cloud Phone Account Work Under Control](/blog/agentic-automation-security-cloud-phone-accounts/)
- [AI Agents Need Brakes: What Controlled Takeover Means for Cloud Phone Automation](/blog/ai-agent-control-boundaries-cloud-phone-takeover/)
