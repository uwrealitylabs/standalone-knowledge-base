---
tags: [monado, tracking, slam]
aliases: [SLAM tracker, t_tracker_slam]
status: draft
---
# t_slam

Part of [Project Standalone](Project%20Standalone.md).

Monado's SLAM tracker adapter (`src/xrt/auxiliary/tracking/t_tracker_slam.cpp`). It sits between the [Monado headset driver](Monado%20headset%20driver.md) and an external SLAM library. The team configures and wires it; it should not need modification.

## Interfaces
| Face | Interface | Used by |
|---|---|---|
| Input | `xrt_slam_sinks` (camera, IMU, ground-truth, hand-mask sinks); see [Tracker sinks](Tracker%20sinks.md) | Capture or replay source |
| Output | `xrt_tracked_slam_get_tracked_pose(xts, time, &pose)` | HMD device |
| Algorithm | VIT, a C interface to a shared library loaded at runtime | Basalt fork now, GPU SLAM later |

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
- `vit/vit_interface.h` (`src/external/vit_includes/`) appears in both Monado and the external system; the external system implements it.
- Loaded at runtime from `VIT_SYSTEM_LIBRARY_PATH` (default `libbasalt.so`). Main implementation is Collabora's Basalt fork (gitlab.freedesktop.org/mateosss/basalt).
- **For our GPU SLAM:** it is a shared library implementing VIT, swapped in by changing the path. t_slam, the sinks and device code stay unchanged. The VIT header is the contract for that workstream.

## Notes
- The SLAM repo and the Monado fork must agree on the VIT interface version; pin matching commits.
- Zero-copy GPU input (dma-buf) is not carried by VIT today; adding it is a core change.
