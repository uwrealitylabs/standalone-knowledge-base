---
tags: [monado, driver, architecture]
aliases: [headset driver, UWRL Monado driver]
status: draft
---
# Monado headset driver

Part of [Project Standalone](../Project%20Standalone.md).

The whole driver-side package for our hardware inside the Monado fork. It runs in userspace inside `monado-service`; it is not a kernel driver. It has one face toward the core (`xrt_device`) and a lot of internal plumbing behind it.

## Data flow
```
Kernel: V4L2, IIO (camera, IMU buffers)
   │
   ▼
┌─ Headset driver ─────────────────────────────────────────┐
│  Capture ──sinks──► Trackers ──state──► xrt_device       │
│  (frames, IMU)      (SLAM, hands, eyes)  (HMD, hand, eye)│
│                                                          │
│  Builder: creates and wires all at startup (not runtime) │
└───────────────────────────────┬───────────────┬──────────┘
                     poses, input│        display│
┌─ Monado core ─────────────────▼───────────────▼──────────┐
│  OpenXR state tracker ──layers──► Compositor             │
│  (maps API to xrt calls)          (swapchains, warp,     │
│                                    distortion)           │
└──────────┬──────────────────────────────┬────────────────┘
  calls,   │                       present,│ vblank
  results  ▼                               ▼
OpenXR API (apps, via client library)   Kernel: DRM/KMS (Vulkan display target)
```
Arrows follow the data, not who initiates. "Display" carries the display description, distortion correction, and view poses for time warp.

### Push below the device, pull above it
- **Below (push).** Capture pushes frames as they arrive. Trackers process them on their own threads and store timestamped results, at sensor rate, whether or not anyone is asking.
- **Above (pull).** The core calls device functions when it needs an answer. The device asks the tracker for its state at that time and returns it in the same call.
- The tracker's timestamped history is what makes this work: data arrives asynchronously, queries arrive at arbitrary times.
- Capture never talks to the device directly, and the core never talks to capture or trackers.

| Trigger | Device call |
|---|---|
| `xrLocateViews` / `xrLocateSpace` at predicted display time | `get_view_poses(time)` / `get_tracked_pose(time)` |
| `xrLocateHandJointsEXT` | `get_hand_tracking(time)` |
| `xrSyncActions` | `update_inputs` |
| Compositor, just before display (time warp) | `get_view_poses` |

In out-of-process mode, the app's call goes through Monado's client library over IPC to the service, where the device call happens.

## Components
| Component | Monado interface | Responsibility |
|---|---|---|
| Builder | `xrt_builder` | Detects whether the headset is present. On startup, creates every device and tracker, connects the frame pipeline, assigns roles (head, left hand, right hand) and hands the finished set to the core |
| HMD device | `xrt_device` | Answers pose and view queries; provides display description and distortion |
| Hand and eye devices | `xrt_device` (one or more) | Expose hand joints and gaze to the core. Each wraps the matching tracker |
| Capture | Producer side of the sinks (own code, or Monado's V4L2 frame server) | Turns V4L2 buffers and IMU readings into frames and samples and pushes them → [Tracker sinks](../tracking/Tracker%20sinks.md) |
| Frame pipeline wiring | Frame context and pipeline nodes | Owns queues, converters, splitters, recorder and debug views; starts and stops them as a unit |
| Calibration loading | Monado's calibration and display structures | Parses camera, IMU, display and head-to-eye calibration into the forms trackers and devices expect |
| Pose composition | Tracker query interfaces | Converts the tracker's IMU-frame pose into head and eye poses; keeps the pose history used for prediction |
| Sensor control | V4L2 controls, or whatever path the hardware lead exposes | Auto-exposure and gain for tracking cameras; IR illuminators if software-controlled |
| Debug and tuning | Monado debug variables and logging | Live values and toggles in the debug UI, needed for tracking tuning |

## Not in this driver
- **Tracker algorithms.** The driver creates and connects trackers; Basalt, Mercury and our own trackers are separate libraries behind their interfaces. SLAM is loaded through [t_slam](../tracking/t_slam.md); hands via [Hand tracking](../tracking/Hand%20tracking.md).
- **Compositor and OpenXR state tracker.** Core, reused from upstream.
- **Display output.** The driver only describes the display. The compositor drives the panels through Vulkan and KMS (see [Display](../hardware/Display.md)).

## Builder
The builder is code, not configuration: written in C as part of the driver and registered in Monado's builder list (`src/xrt/targets/common/target_lists.c`). It is not in the data path; it acts once at startup.

**Startup sequence.** The core asks each registered builder whether its hardware is present, picks the best match, and calls its open function. The builder:
1. Loads calibration and config files.
2. Creates the trackers, each exposing its input sinks.
3. Creates the capture source and connects its output to those sinks, inserting queue, converter or recorder nodes as needed.
4. Creates the devices, giving each handles to the trackers it queries plus the calibration it needs (display description, head-to-eye transforms).
5. Registers capture and trackers with a shared frame context, which starts, stops and tears them down together.
6. Returns the devices and their roles (`xrt_system_devices`) to the core.

After that nothing is rewired, and the builder has no further role.

**Ownership.** Core owns the device set → device set owns the frame context → frame context owns capture and trackers. Each device holds references to the trackers it queries. Shutting down the device set tears down the whole driver side in order. The builder created the tree but is not part of it.

**Contracts.** The builder is the single place where every driver-side contract is applied: which camera feeds which tracker, which calibration goes where, which tracker each device queries. Replay versus live capture is also a builder decision (different source, same sinks). Natural owner: the team's integration lead.

**Configuration.** One headset config file read by the builder, with environment variables overriding it during development. Contents:
- Source mode: live capture, or replay with a dataset path.
- Mapping from each physical camera to a tracker input (explicit, not implied by push order).
- Calibration file paths.
- Which trackers are enabled.
- Whether recording is on.

On the finished headset ours is the only builder whose hardware is present, so selection is automatic.

## Device implementation
The code behind the `xrt_device` functions the core calls. The device calls the tracker, never the reverse. For `get_view_poses(T)`:
1. Ask SLAM for its pose at T (IMU-frame pose).
2. Apply the IMU-to-head transform → head pose.
3. Apply head-to-eye transforms and IPD → one pose per eye.
4. Attach each eye's FOV from the display description.
5. Set validity flags from tracker state.

Hands follow the same pattern: query the hand tracker, transform joints into tracking space, report validity.

**3DoF fallback.** Copy the Rift S pattern: a flag to track with 6DoF SLAM or fall back to IMU-only 3DoF fusion, keeping the headset usable while SLAM starts up or is lost.

## Display description
| Concern | Provided by | How |
|---|---|---|
| Per-eye resolution, viewports, FOV, refresh interval | HMD `xrt_device` | Static description filled by the builder from calibration |
| Lens distortion | HMD `xrt_device` | Compositor samples `compute_distortion` at startup into a mesh, applied every frame |
| Head and eye poses for time warp | HMD `xrt_device` | Compositor calls `get_view_poses` just before display |
| Physical output, presenting, vblank | Compositor output target | Direct mode via Vulkan display extensions on KMS, or a desktop window for development |

Nothing checks that the device's display description matches the kernel mode. If they disagree, the compositor still renders to the description, giving scaling artifacts or wrong frame pacing. The driver should check at startup and fail loudly. Display selection layers are in [Display](../hardware/Display.md).

## Core dependencies
The state tracker and compositor are device-agnostic: they use only what the device interface provides, which is why Stage 1 of the [Tracer bullet roadmap](../Tracer%20bullet%20roadmap.md) can use a simulated device. They are not capability- or platform-agnostic:
1. **What devices advertise.** Input mapping uses binding tables in core code. Standard inputs (head pose, hand tracking, eye gaze) need no core changes; a new physical controller would.
2. **Platform and GPU.** The compositor depends on Vulkan, Turnip and its output targets. Most RB5 risk is here.
3. **Extension coverage.** If an OpenXR extension isn't implemented in the state tracker, no device can expose it.

As long as the headset fits capabilities Monado already supports, all hardware-specific work stays in this driver.

## Team split
1. **Capture and sensor control.** Talks to kernel interfaces; works closest to the hardware lead.
2. **Builder, calibration loading and pipeline wiring.** The integration glue that connects everything at startup.
3. **Device implementations and pose composition.** The face toward the core.

## Reference drivers and reading order
Monado's WMR and Rift S drivers contain every component above; read one alongside the components table. Rift S is the closest match to our headset.
1. Monado docs, ["Writing a new driver"](https://monado.pages.freedesktop.org/monado/writing-driver.html): start from the Sample HMD driver.
2. [Doc index](https://monado.pages.freedesktop.org/monado/pages.html): Spaces, Frame Pacing/Timing, Mercury hand tracking, Understanding and Writing Targets.
3. Interface headers: `xrt/xrt_frame.h`, `xrt/xrt_tracking.h`, then `tracking/t_tracking.h`, `tracking/t_hand_tracking.h`.
4. Implementations: `t_tracker_slam.cpp`, `vit/vit_interface.h`, Mercury's `hg_interface.h` and `hg_sync.cpp`, `drivers/ht/ht_driver.c`.
5. Full driver: Rift S `rift_s_tracker.c` and its builder, alongside the sample HMD driver.
6. Background: Collabora, ["State of Monado's visual-inertial tracking"](https://www.collabora.com/news-and-blog/blog/2022/12/20/state-of-monado-visual-inertial-tracking).

Monado's XRT interfaces are subject to change. Keep the fork small; upstreaming the driver eventually is the lowest-maintenance path.

## Open questions
- One logical dual-DSI display vs two independent displays. Tracked in [Display](../hardware/Display.md).
- Sensor timestamp clock domain delivered by the hardware layer. See [Sensor sync architecture](../hardware/Sensor%20sync%20architecture.md) and [Sensor timestamping pipeline](Sensor%20timestamping%20pipeline.md).
- Multi-camera hand tracking design. See [Hand tracking](../tracking/Hand%20tracking.md).
