---
tags: [plan, roadmap, tracer-bullet]
aliases: [Roadmap, tracer bullet, tracer plan, sequencing, milestones]
status: draft
---
# Tracer bullet roadmap

Part of [Project Standalone](Project%20Standalone.md).

First priority is an end-to-end tracer bullet from sensors up to the OpenXR API, with hardware blockers avoided by emulating inputs at defined boundaries. Each stage replaces emulation with real code or hardware, leaving everything above it unchanged.

## Emulation points
Three input-side boundaries, from the [Monado headset driver](runtime/Monado%20headset%20driver.md) data flow. Each one further down removes more of our own code from the test.

| Emulation point | What replaces what | Real code exercised | Use it for |
|---|---|---|---|
| Kernel interfaces | vivid / v4l2loopback / vkms stand in for kernel drivers | Capture, trackers, device, core | Capture code: setup, buffers, formats, drop handling |
| Sinks | A replay source pushes recorded frames and IMU into the tracker sinks | Trackers, device, core | Perception and pose output on real recorded data |
| `xrt_device` | Monado's simulated device replaces the whole headset driver | Core and apps only | App → runtime → compositor path before any driver exists |

- Kernel-level emulation exercises everything, but test patterns give SLAM nothing to track. Sink-level replay is the only point where perception runs on meaningful data.
- Display side has its own emulation points: vkms at the kernel, or a desktop window at the compositor's output target. Either combines with any input-side option.

## Stage 1: Prove the platform
Goal: show the RB5, kernel, Turnip and stock Monado run OpenXR apps. No custom code.
- Build Monado for Arch Linux ARM on the RB5 dev kit.
- Run it with Monado's simulated device (it ships with Monado and provides a display description and pose), presenting through the Vulkan display target (`VK_KHR_display`) to the dev kit's HDMI output. HDMI is also a KMS connector, so the panels aren't needed yet.
- Run `hello_xr`, then the Godot launcher.

**Exit criteria:** apps render through the compositor in direct mode on the RB5, with stable frame pacing.

## Stage 2: Headset driver against emulated inputs
Goal: all driver-side code runs end to end without custom hardware.

**Driver skeleton**
- Builder, HMD device, hand-tracking device.
- Config loading: source mode, camera-to-tracker mapping, calibration paths.
- North Star display description and distortion.

**Sink-level replay first**
- A replay source pushes recorded frames and IMU into the SLAM and hand-tracking sinks. Use the [Monado SLAM datasets](https://huggingface.co/datasets/collabora/monado-slam-datasets) (egocentric, recorded on real XR headsets) over EuRoC (drone).
- SLAM runs through [t_slam](tracking/t_slam.md) and VIT with stock Basalt as the baseline. Hands run through Mercury ([Hand tracking](tracking/Hand%20tracking.md)).

**Kernel-level emulation next**
- v4l2loopback, fed with the same recorded imagery, exercises the real V4L2 capture code.
- vivid tests buffer and format plumbing.
- IMU stays on sink-level replay.

**Display output:** desktop window or vkms, then HDMI on the RB5.

**Convergence:** loopback replay → real capture code → perception → OpenXR. Only the IMU input is still emulated.

**Exit criteria** (correctness, not peak accuracy):
- Trajectory error comparable to stock Basalt on the same datasets (proves nothing in the path corrupts data).
- Latency and throughput within budget on the RB5.
- No drops or timing errors.
- Display description matches the output mode.

Do not tune SLAM for peak accuracy on recorded datasets. Their resolution, FOV, frame rate and IMU differ from ours, so tuning would fit parameters to the wrong cameras.

## Stage 3: Swap in hardware incrementally
Goal: replace one emulated contract at a time as hardware arrives.
- **Display:** HDMI first, then the DSI panels. Requires the one-display-or-two decision ([Display](hardware/Display.md)).
- **Partial hardware:** individual sensors on breakout boards (a single AR0234 or OV9281, the IMU on a dev board) to record real data. Accuracy tuning starts to mean something here.
- **IMU:** the real sensor replaces replay.
- **Direct-connected cameras:** front stereo, side, RGB.
- **FPGA cameras:** bottom and eye tracking, once the FPGA link and its driver exist.
- **Optional, in parallel:** synthetic data rendered to match our exact camera models, to extend tuning before full hardware.

**Exit criteria:** a fully functional headset. Each swap is its own milestone, so hardware blocks progress one piece at a time.

## Beyond the tracer
Capability order once the tracer works: 3DoF → 6DoF → hands → eyes and RGB features. Work items not yet scheduled:
- Calibration tooling: camera–IMU and display distortion.
- Eye-tracking software.
- Replacing Basalt with the GPU SLAM library (swap via VIT path; see [t_slam](tracking/t_slam.md)).

## Open questions
- Stage 1 assumes Arch Linux ARM and a mainline kernel with Turnip. The team board currently runs Ubuntu 18.04 with Docker ([OS and kernel](platform/OS%20and%20kernel.md)). Decide whether Stage 1 waits for the OS move or proves what it can on the current image.
