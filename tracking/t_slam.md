---
tags: [monado, tracking, slam]
aliases: [SLAM tracker, t_tracker_slam]
status: draft
author: "Claude Opus 5.5"
---
# t_slam

Part of [Project Standalone](../Project%20Standalone.md).

Monado's SLAM tracker adapter (`src/xrt/auxiliary/tracking/t_tracker_slam.cpp`). It sits between the [Monado headset driver](../standalone-driver/Monado%20headset%20driver.md) and an external SLAM library. The team configures and wires it; it should not need modification.

## Interfaces
| Face | Interface | Used by |
|---|---|---|
| Input | `xrt_slam_sinks` (camera, IMU, ground-truth, device-mask sinks); see [Sinks](../runtime/Sinks.md) | Capture or replay source |
| Output | `xrt_tracked_slam_get_tracked_pose(xts, time, &pose)` | HMD device |
| Algorithm | [VIT](VIT.md), a C interface to a shared library loaded at runtime | Basalt fork now, GPU SLAM later |

## Responsibilities
1. Load the VIT library from the configured path.
2. Pass calibration and the library's own config file.
3. Forward frames and IMU samples to the library.
4. Store timestamped pose estimates in a history.
5. Answer pose queries for arbitrary times by interpolation or prediction.
6. Convert coordinate conventions; optional smoothing.
7. Tooling: debug UI, CSV trajectories, EuRoC recording, ground-truth comparison.

## Usage
`t_slam_fill_default_config` → set fields of `t_slam_tracker_config` → `t_slam_create(xfctx, config, ...)` → `t_slam_start`. Create returns both the sinks and the query object.

Config fields: VIT system library path, algorithm-specific config file, camera count, prediction level, CSV output.

## Internals
- Camera frame sinks and an IMU sink forwarding to the SLAM system.
- Loaded VIT function pointers.
- History of poses produced purely by the external tracker.
- Selectable prediction type; smoothing filters; EuRoC recorder; debug UI variables.

## VIT
The algorithm sits behind [VIT](VIT.md), a C ABI loaded at runtime from `VIT_SYSTEM_LIBRARY_PATH` (default `libbasalt.so`). The call sequence, Basalt's implementation and the requirements for our GPU SLAM library are in that note.

## Pose path
On each `get_tracked_pose(t)` (`t_tracker_slam.cpp:1165`): drain new poses from the library (`flush_poses`), add them to a relation history, predict to `t` per `SLAM_PREDICTION_TYPE` (default dead reckoning on buffered IMU samples), then optionally filter. The result is the IMU's pose in the SLAM system's world frame; the device applies axis correction and `device_from_imu`.

## Gotchas
- `SLAM_SUBMIT_FROM_START` defaults to false (`t_tracker_slam.cpp:91`). Until it is set or toggled in the debug GUI, everything pushed into the sinks is discarded. `monado-cli slambatch` sets it; a custom builder must too.
- The first frame must come from camera 0 (debug assert), and timestamps must increase. IMU samples older than the last are dropped; out-of-order frames only produce a warning here, but Basalt aborts on wrong camera order or mismatched stereo timestamps (see [VIT](VIT.md)).
- `SLAM_CAM_COUNT` defaults to 2.

## Notes
- First integration into our driver: [SLAM tracer bullet from EuRoC](../standalone-driver/task-guides/SLAM%20tracer%20bullet%20from%20EuRoC.md).
- The SLAM repo and the Monado fork must agree on the VIT interface version; pin matching commits.
- Zero-copy GPU input (dma-buf) is not carried by VIT today; adding it is a core change.
