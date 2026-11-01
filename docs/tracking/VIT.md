---
tags: [tracking, slam, interfaces, vit]
aliases: [vit_interface, Visual-Inertial Tracking interface, VIT interface]
status: draft
author: "Claude Opus 5.5"
---
# VIT

Part of [t_slam](t_slam.md).

Sources:
- **Monado side:** `uwrealitylabs/standalone-monado-driver`, branch `dev` at `6bbefe8`. Paths relative to `src/xrt/` unless they start with `src/external/`.
- **Basalt side:** GitHub mirror `ChristophHaag/basalt-monado` at `12249ee` (2025-03-18). The mirror describes itself as not kept up to date. The upstream fork (gitlab.freedesktop.org/mateosss/basalt) was not reachable when this note was written; re-check Basalt references against it.

VIT (Visual-Inertial Tracking) is a C ABI between a consumer (Monado's [t_slam](t_slam.md)) and a SLAM system shipped as a shared library. Both sides vendor the same header: Monado at `src/external/vit_includes/vit/vit_interface.h`, Basalt at `thirdparty/vit/vit_interface.h`. Both copies are version 2.0.1 (`VIT_HEADER_VERSION_*`, `vit_interface.h:27–29`).

Effect: Monado has no build or link dependency on the SLAM system. The algorithm is chosen at runtime by library path. Our GPU SLAM plugs in by implementing this header.

## API
Two opaque handles: `vit_tracker_t` and `vit_pose_t`. Functions return `vit_result_t` except the destroys. Declarations at `vit_interface.h:347–491`.

| Group | Functions | Notes |
|---|---|---|
| Version | `vit_api_get_version` | The loader calls this first; the major version must match the consumer's header |
| Lifecycle | `vit_tracker_create(config)`, `_destroy`, `_start`, `_stop`, `_reset`, `_is_running` | `vit_config` (`:147`): `file` (implementation-specific config path), `cam_count`, `imu_count`, `show_ui` |
| Capabilities | `_has_image_format`, `_get_supported_extensions`, `_get_enabled_extensions`, `_enable_extension` | Extensions (`:101`): `ADD_CAMERA_CALIBRATION`, `ADD_IMU_CALIBRATION`, `POSE_TIMING`, `POSE_FEATURES` |
| Calibration | `_add_camera_calibration`, `_add_imu_calibration` | Only if the matching extension is supported. Camera models: none (pinhole), KB4, RT8 |
| Input | `_push_imu_sample`, `_push_img_sample` | `vit_imu_sample`: timestamp, accel, gyro. `vit_img_sample` (`:188`): `cam_index`, timestamp, `data`/`width`/`height`/`stride`/`size`, `format` (L8 or R8G8B8), `masks` (regions to ignore) |
| Output | `_pop_pose`, `vit_pose_get_data`, `_get_timing`, `_get_features`, `vit_pose_destroy` | `vit_pose_data` (`:208`): timestamp, position, orientation quaternion, linear velocity, all `float`. The caller owns a popped pose until it destroys it |

## How t_slam calls it
### Loading
`auxiliary/tracking/t_vit_loader.c`:
1. `dlopen` the library (`:44`). Path from `VIT_SYSTEM_LIBRARY_PATH`, default `libbasalt.so` (`t_tracker_slam.cpp:55`, `:88`).
2. Resolve `vit_api_get_version`, call it, reject a major version mismatch (`:61–70`).
3. Resolve every other symbol with `dlsym` into the `t_vit_bundle` function-pointer struct (`:73–92`).
4. `t_vit_bundle_unload` closes the library (`:99–102`).

### Call sequence
All in `auxiliary/tracking/t_tracker_slam.cpp`.

| Phase | t_slam function | VIT call | Line |
|---|---|---|---|
| Create | `t_slam_create` | load bundle | `:1474` |
| | Requires `SLAM_CONFIG` or a `slam_calib` struct, else fails | | `:1478–1483` |
| | | `tracker_create({file, cam_count, show_ui})` | `:1492` |
| | | `tracker_get_supported_extensions` | `:1498` |
| | `send_calibration`, only if `slam_calib` was given | `add_camera_calibration` per camera, `add_imu_calibration` | `:1508` → `:1098`, `:1125` |
| | Timing and feature statistics | `enable_extension(POSE_TIMING / POSE_FEATURES)` | `:437`, `:557` |
| Start | `t_slam_start` | `tracker_start` | `:1425` |
| Frame | `receive_frame`, via a camera sink | `push_img_sample` | `:1352` |
| IMU | `t_slam_receive_imu` | `push_imu_sample` | `:1260` |
| Query | `t_slam_get_tracked_pose` → `flush_poses` | `pop_pose` until empty; `pose_get_data`; `pose_destroy` | `:1178` → `:753–820` |
| Reset | Debug GUI button | `tracker_reset` | `:986` |
| Teardown | `t_slam_node_break_apart`, `t_slam_node_destroy` | `tracker_stop`; `tracker_destroy`; unload | `:1381`; `:1415–1416` |

### Behaviour on the Monado side
- **Threads.** Both push calls run on the pusher's thread (capture or replay). The image `data` pointer is only valid during the call, so the implementation must copy or finish with it before returning.
- **Formats.** Only L8 and R8G8B8 are mapped (`:1314–1318`); other formats are logged and dropped.
- **Masks.** Each image sample carries the latest device masks for that camera (from the [sink](../runtime/Sinks.md) `masks[]`), e.g. hand bounding boxes from Mercury.
- **Flushing.** Poses are also flushed when the last camera's frame arrives (`:1286–1288`), so the history stays current while no app is querying.
- **After popping.** t_slam computes angular velocity by finite difference against the previous pose and stores the relation in its history (`:798–802`). `predict_pose` (`:827`) then extrapolates to the requested time per `SLAM_PREDICTION_TYPE` (`auxiliary/tracking/t_tracking.h:663`): none, pose only, gyro, accel+gyro, or dead reckoning (default; integrates IMU samples received after the last SLAM pose).
- **`imu_count` is never set.** `tracker_create` receives `imu_count = 0`. Basalt ignores the field; an implementation that validates it would fail.
- **No coordinate convention.** VIT does not define the world frame. Drivers hard-code a Basalt-specific axis correction after querying (`drivers/rift_s/rift_s_tracker.c:609`, called at `:655`; `drivers/twrap/twrap_slam.c:77`, called at `:115`); the Rift S code carries a todo saying this belongs in the SLAM system.

## How Basalt implements it
### Layers
1. **C ABI shim:** `thirdparty/vit/vit_implementation_helper.cpp`. Each `vit_*` function `static_cast`s the opaque handle to `vit::Tracker *` (an abstract C++ class) and calls the matching virtual method. `vit_api_get_version` is at `:16`.
2. **Tracker class:** `basalt::vit_implementation::Tracker` (`include/basalt/vit/vit_tracker.hpp`) implements `vit::Tracker` and forwards to a pimpl, `Tracker::Implementation` (`src/vit/vit_tracker.cpp:87`).
3. **Entry point:** `vit_tracker_create` constructs it (`src/vit/vit_tracker.cpp:706`).
4. **Build:** compiled into `libbasalt.so` (`CMakeLists.txt:449–453`).

### Behaviour per call (`src/vit/vit_tracker.cpp`)
| VIT call | Basalt behaviour | Line |
|---|---|---|
| create | Asserts `cam_count > 1` (no monocular support). If `file` is set, parses a unified TOML config (`show-gui`, `cam-calib`, `config-path`, `marg-data`, `print-queue`, `use-double`, `deterministic`), then loads the VIO config JSON and calibration JSON | `:144–166`, `:198` |
| add calibration | Stores the calibration; applied at start, overriding file calibration. Converts camera models none/KB4/RT8 to Basalt's | `:635–643`; applied `:244`, `:289` |
| start | `initialize()`: applies calibrations, asserts IMU and every camera are calibrated, creates the optical-flow frontend and VIO estimator and connects their queues. `start()`: launches the state-consumer thread | `:351–401` |
| push IMU | Copies into `ImuData`, pushes to both the VIO IMU queue and the optical-flow IMU queue | `:424–434` |
| push image | **Asserts** cameras arrive in order 0, 1, … and that every camera's timestamp equals camera 0's. Copies pixels into a 16-bit image (`value << 8`), attaches masks, and pushes the assembled multi-camera frame to the frontend when the last camera arrives | `:436–489` (asserts `:438`, `:462`; copy `:480`) |
| pop pose | Non-blocking `try_pop` from an output queue of capacity 32; wraps the state as a VIT pose | `:491–515`; capacity `:165` |
| pose data | `T_w_i` (world-from-IMU) translation and quaternion, `vel_w_i`, `t_ns` | `:652–666` |
| stop | Pushes null sentinels into the queues and joins threads | `:403–416` |

### Pipeline
```
push_img_sample ──► optical-flow frontend thread ──► VIO estimator thread ──► out_state_queue
push_imu_sample ──►        (IMU queue)        └──────────► (IMU queue)              │
                                                                   state_consumer thread
                                                                                    ▼
pop_pose ◄──────────── monado_out_state_queue (cap 32, drops oldest when full, :542)
```
The returned pose is the IMU's pose in Basalt's world frame, which is why drivers apply an axis correction and chain `device_from_imu` afterwards (see [Monado headset driver](../standalone-driver/Monado%20headset%20driver.md#device-implementation)).

### Configs
`data/vit/*.toml.in` (EuRoC, TUM-VI, Monado SLAM datasets `msdmi`/`msdmg`/`msdmo`, D455) are the unified configs `SLAM_CONFIG` points at. Each references a calibration JSON and VIO config JSON in `data/euroc/`, `data/msd/`, etc. CMake fills in install paths (`CMakeLists.txt:556–564`).

## Implications for our work
- **Stereo ordering is fatal in Basalt.** t_slam only warns on out-of-order timestamps; Basalt aborts on wrong camera order or mismatched stereo timestamps. Capture must guarantee pairs (hardware sync or a force-genlock node, see [Sinks](../runtime/Sinks.md)).
- **Per-frame CPU copy.** Every image is copied and widened to 16 bits on the pushing thread. At AR0234 resolution that cost matters; bin or crop before pushing.
- **No monocular.** Basalt needs at least two cameras.
- **Calibration source.** Either files referenced by the TOML or programmatic via `slam_calib` and the calibration extensions. Programmatic keeps the builder's calibration as the single source of truth.
- **For our GPU SLAM library:**
  - implement the full symbol set the loader resolves (`t_vit_loader.c:73–92`), or loading fails;
  - copy or consume image data within the push call (the ABI has no zero-copy or GPU-buffer path);
  - own its threading and queues; `pop_pose` must not block;
  - return IMU poses with increasing timestamps; agree with the driver on the world-frame convention;
  - match VIT major version 2.

## Open questions
- Verify the Basalt references against the upstream fork; the mirror used here is from March 2025.
- Where the axis correction should live for our headset: driver code, or our SLAM library returning OpenXR conventions.
- Whether to pass calibration programmatically from the builder or keep it in Basalt's files.
