---
title: "Your Script Works on One Cloud Phone but Fails on Another: Handling
  Device Variability in Your Fleet"
description: Why the same script passes on one cloud phone and fails on another
  — and how ops teams diagnose device variability (OS, resolution, OEM quirks,
  WebView, image drift) and prevent batch failures.
pubDate: 2026-08-15
updatedDate: 2026-08-15
---

## Answer First

**Definition:** A "cloud phone script fails on some devices" incident is a device-variability failure: the same script, the same app version, and the same account behave differently across devices in one fleet because of differences in Android OS version, screen resolution and density, OEM or ROM behavior, accessibility-API and WebView versions, or drift between the device images the fleet was provisioned from.

**Why:** Cloud phone fleets look uniform, but they rarely are: devices come from different images, built at different times, with different OS builds and independently updated components. Scripts assume a stable UI surface: pixel coordinates, view hierarchies, timing, app internals. Every device is an environment, and the device is part of the system under test. When fleet members diverge, the script's assumptions stop holding on some of them, not because the script changed but because the device it runs on did. The fleet itself is a failure variable: in a batch operation, the failure rate is a property of the device mix.

**Example:** A login flow taps "Sign in" at pixel coordinates (540, 1180). On the 1080×2400 staging image, the tap lands on the button and the run passes. On a 720×1600 image in the same batch, the same coordinates land above the button, hit empty space, and the login times out. Script, app version, and account are identical: the device is the only thing that changed.

## Key Facts

- Device-to-device variance is a standing cause of "works here, fails there" in scripted mobile operations, touching every layer a script relies on: OS version, screen geometry, OEM/ROM behavior, the accessibility tree, and WebView rendering.
- Android OS version changes the accessibility node structure uiautomator-style automation depends on: the same app can expose different nodes, IDs, and semantics on different Android versions.
- Pixel coordinates are not portable: Android layouts use density-independent pixels (dp), so the same layout maps to different pixels across resolutions and densities.
- OEM and ROM builds vary in system dialogs, default settings, and permission flows even at the same Android version, variation the Compatibility Definition Document explicitly permits.
- WebView is a separate component updated independently of the OS, and device-image drift means "the same device model" stops being the same environment over time.

## Expert Explanation

The failure looks like a flaky script, but the evidence usually points at the device. Most teams re-run it, watch it pass on the staging device, and call it flaky, the exact pattern that hides device variability. The script was environment-sensitive. Here is what differs inside a fleet, and how each difference breaks a script.

### Android OS version differences

Scripts that interact with apps through the accessibility API, the standard approach for uiautomator-driven automation, depend on the view hierarchy an app exposes. That hierarchy is not stable across OS versions: node types, content descriptions, and even whether an element is exposed to automation at all can differ between Android versions with the same app. A selector that resolves on Android 13 can silently return nothing on Android 14, and the script fails exactly where the fleet groups diverge. The Compatibility Definition Document is the floor every device must meet, not a guarantee of identical behavior but the formal allowance for device variation.

### Screen resolution and density

The most common cause of "it worked on that phone" is coordinates. Cloud phone images come in many resolutions (720×1600, 1080×2400, and others), and Android layouts are defined in dp precisely because raw pixels are not comparable across devices. A 480 dp button centered might span pixels 540–630 on a 1080-wide screen and 360–440 on a 720-wide screen. A script that taps (540, y) is right on the first device and wrong on the second. The fix is not better coordinates: stop depending on raw coordinates as a primary strategy, or verify every tap point against each image's resolution before a run.

### OEM and ROM quirks

Two devices can report the same Android version and still behave differently. OEMs and ROM builders customize system components such as permission dialogs, battery and background settings, the launcher, and system UI behavior. The Compatibility Definition Document permits wide implementation latitude while keeping apps compatible, which is why the same app presents slightly different flows on different images: an unexpected dialog, an extra confirmation step, a permission prompt that appears on one image and not another.

### Accessibility-API and WebView differences

Even a script that never touches web content can be affected by WebView, because many apps embed web UI for login, payments, or content. WebView is a separate component updated independently of the OS, so two devices on the same Android version can run different builds, render the same page differently, and shift the elements a script reaches. "Same app version" is not "same rendering environment."

### Device-image drift

The fleet-level version of all of the above is drift in the images themselves. Staging images get rebuilt or patched while older production images stay frozen; new devices get provisioned from a newer image than the batch that ran yesterday. The fleet silently becomes a set of distinct environments that share a name. This is why an operation can pass at 9:00 and fail at 11:00 with no change to the script: the image changed, or the mix of images changed. Treat device-image drift like code drift: version it, review changes, and always know what is running where.

## Decision Framework

Diagnose before you "fix the script." Use this sequence when a script passes on some devices and fails on others:

| Step | What to check | What the result tells you |
| --- | --- | --- |
| 1. Compare failing vs passing devices | OS version, image ID, build date, WebView version, resolution and density | Whether the failing group shares a difference the passing group lacks |
| 2. Pull uiautomator/accessibility dumps | Diff the dumps at the failing step | Whether the view hierarchy diverges: missing node, changed ID, semantics |
| 3. Verify coordinates vs resolution | Map each tap point to the failing device's actual pixels | Whether coordinate assumptions silently miss their targets |
| 4. Check for image drift | Confirm each device's image matches the run plan, and none changed mid-batch | Whether the failure is a fleet-mix problem rather than a script problem |

Once the cause is identified, the prevention workflow:

1. **Pin device images.** Record the image ID and build for every device; provision new devices only from approved images.
2. **Test on a small device matrix before batch runs.** Cover the OS versions and resolution buckets your fleet contains, not just your staging image.
3. **Group devices by image.** Run batches within an image group, or at minimum record which image each device ran on, so failures stay attributable.
4. **Treat device-image drift like code drift:** version images, review changes before rollout, and give an image update the same scrutiny as a script change.
5. **Log the environment with the run.** Record OS version, image ID, resolution, and WebView version per device per run, so the next "works here, fails there" is answered from logs instead of archaeology; [mobile automation needs that logging even more than desktop agents do](/blog/ai-agent-logs-for-mobile-automation/).

Practical limits: a full matrix of every image × OS × resolution combination is usually not worth the cost; group devices by the characteristics that change behavior. Expect a small residual failure rate even in a well-standardized fleet, because the goal is explainable failures, not zero failures. WebView and app updates can change behavior after your matrix test, so re-run the matrix on a schedule or after any image change.

## Key Takeaways

- When a cloud phone script fails on some devices, treat the fleet as the variable: the script, app, and account may be identical while the devices are not.
- Check these four first: OS version, screen resolution and density, OEM/ROM behavior, and accessibility/WebView differences, all visible in device metadata.
- Diagnosis is a fixed sequence: compare OS versions and images, diff uiautomator/accessibility dumps, verify coordinates against resolution, and confirm images didn't drift mid-batch.
- Prevention is standardization: pin and version device images, test on a small representative matrix before batch runs, and group devices by image: the operations-layer equivalent of a [control tower](/blog/ai-agent-control-tower-for-mobile-app-workflows/) for your fleet of agents.
- Log the environment with every run. When a task does fail mid-run, [what happens next](/blog/ai-agent-fails-what-happens-next/) depends on how much environment context you captured.

## FAQ

**Q: Why does the same script pass on one cloud phone and fail on another?**
A: Because the devices are different environments even when they share a model name. Android OS version, screen resolution and density, OEM/ROM behavior, accessibility-API output, WebView version, and the device image all shift how a script's assumptions hold. A "cloud phone script fails on some devices" report is usually a fleet-composition problem, not a code problem.

**Q: How do I find out which device difference broke my script?**
A: Compare the failing and passing groups by OS version, image ID, resolution, and WebView version; pull uiautomator/accessibility dumps at the failing step and diff them; verify tap coordinates against each device's resolution; and confirm no image changed mid-batch.

**Q: How do I fix a script that taps the wrong place on some devices?**
A: Replace coordinate-based interaction with hierarchy-based selectors where the app exposes them, using content descriptions or stable IDs from the accessibility tree. Where coordinates are unavoidable, verify each image's resolution and either branch per resolution bucket or exclude mismatched devices from the batch. Then prevent recurrence by pinning images and standardizing the resolution mix.

**Q: Do I need to test on every device image before a batch run?**
A: Not every image, every image group. Build a small matrix covering the OS versions, resolution buckets, and image families your fleet contains, and run it before each batch or after any image change. What matters is the real spread of your fleet, not coverage of every image.

## Sources

- Android UI Automator — inspecting and dumping the UI hierarchy: https://developer.android.com/training/testing/ui-automator
- Android Compatibility Definition Document (CDD) — the contract that defines where devices may vary: https://source.android.com/docs/compatibility/cdd
- Support different pixel densities — why dp exists and layouts map to pixels: https://developer.android.com/training/multiscreen/screendensities
- Build web apps in WebView — how embedded web UI renders in Android apps: https://developer.android.com/develop/ui/views/layout/webapps
