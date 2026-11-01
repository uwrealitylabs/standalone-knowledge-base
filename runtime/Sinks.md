---
tags: [monado, runtime, interfaces, frame-graph]
aliases: [sink, frame sink, xrt_frame_sink, frame graph, frame context, Tracker sinks, sink data types]
status: draft
author: "Claude Opus 5.5"
---
# Sinks

Part of [Monado headset driver](../standalone-driver/Monado%20headset%20driver.md).

Source references are to `uwrealitylabs/standalone-monado-driver`, branch `dev` at `6bbefe8`, as `path:line` relative to `src/xrt/`. Line numbers drift; search by symbol if one is off.

A sink is an object that receives data through a single push function. Capture-to-tracker plumbing in Monado is a graph of sinks: producers push into sinks; some sinks are endpoints (trackers), others process the data and push it onward (queues, splitters, converters). A push is a plain function call, so the receiver's code runs on the pusher's thread. Every thread hop in the graph is an explicit queue node.

## Interfaces
| Interface | Push function | Carries | Defined at |
|---|---|---|---|
| `xrt_frame_sink` | `push_frame(sink, struct xrt_frame *)` | One camera image (refcounted) | `include/xrt/xrt_frame.h:57`; helper `xrt_sink_push_frame` `:73` |
| `xrt_imu_sink` | `push_imu(sink, struct xrt_imu_sample *)` | One IMU reading | `include/xrt/xrt_tracking.h:183` |
| `xrt_pose_sink` | `push_pose(sink, struct xrt_pose_sample *)` | Timestamped pose (ground truth) | `xrt_tracking.h:196` |
| `xrt_device_masks_sink` | `push_device_masks(sink, struct xrt_device_masks_sample *)` | Per-camera-view bounding boxes, one per device (e.g. each hand) | `xrt_tracking.h:206` |
| `xrt_slam_sinks` | none (container) | Pointers: `cams[]`, `imus[]`, `gt`, `masks[]`, plus counts. Null means disabled | `xrt_tracking.h:216` |

Implementations embed the interface struct in their own struct and recover themselves with `container_of`. Example: `t_slam_receive_cam##N` does `container_of(sink, TrackerSlam, cam_sinks[N])` (`auxiliary/tracking/t_tracker_slam.cpp:1357`).

## Data types
**`xrt_frame`** (`include/xrt/xrt_frame.h:24`), one camera image:

| Field | Meaning |
|---|---|
| `reference`, `destroy`, `owner` | Refcount and release callback; lets a producer wrap a driver buffer and get it back when the last consumer drops it |
| `width`, `height`, `stride`, `size`, `data` | Dimensions, row pitch in bytes, total size, CPU pointer to pixels |
| `format` | `xrt_format`: L8, RGB variants, YUYV/UYVY, Bayer, etc. |
| `stereo_format` | Single view or a stereo pair in one image (e.g. side-by-side) |
| `timestamp`, `source_timestamp` | ns. `timestamp` in Monado's monotonic clock; `source_timestamp` keeps the device's original clock |
| `source_sequence`, `source_id` | Frame counter; originating `xrt_fs` |

**`xrt_imu_sample`**: `timestamp_ns`, `accel_m_s2` and `gyro_rad_secs` as `xrt_vec3_f64`, in the IMU's own axes, SI units.

**`xrt_pose_sample`**: timestamped pose from an external reference. Evaluation and recording only.

**`xrt_device_masks_sample`** (`include/xrt/xrt_tracking.h:161`): per camera view, one optional rectangle per device. Mercury pushes hand bounding boxes so SLAM ignores features on moving hands.

**`xrt_slam_sinks`**: the index in `cams[]` is the camera index the calibration refers to. The hand-tracking async wrapper reuses the type for its two inputs (`cams[0]` left, `cams[1]` right).

## Contract points for our headset
- **Pixel format.** Mono tracking cameras arrive as L8 (`t_slam` accepts only L8 and R8G8B8). If V4L2 delivers packed 10-bit raw, capture or a conversion node unpacks it. That costs CPU per frame; decide where it happens.
- **Timestamps.** `timestamp` in Monado's clock, at the time the contract specifies (ideally mid-exposure), identical across both frames of a stereo pair. Basalt aborts on mismatched stereo timestamps ([VIT](../tracking/VIT.md)). See [Sensor timestamping pipeline](Sensor%20timestamping%20pipeline.md).
- **Camera order.** Within a set, push camera 0 first, then 1, … Basalt aborts otherwise; the hand-tracking wrapper drops out-of-order frames.
- **Camera index mapping.** `cams[n]` matches calibration entry n. Make the mapping explicit in builder config, not implied by push order.
- **IMU convention.** SI units, IMU axes, steady rate, same clock as frames. Whether the IMU-to-camera transform is applied by SLAM or device code is decided in calibration, not in the samples.
- **Hold time.** If frames wrap V4L2 buffers, put a bounded queue in front of any consumer that may hold frames long. Otherwise a slow tracker starves capture of buffers.

## Ownership and lifetime
- **The receiver owns the sink.** Sinks are fields of the receiving object; whoever pushes holds a borrowed pointer. `t_slam_create` hands out pointers into its own `TrackerSlam` (`t_tracker_slam.cpp:1514–1529`).
- **Frame graph nodes.** Every `xrt_frame_sink` implementation must also implement `xrt_frame_node` and register with an `xrt_frame_context` in its constructor (doc comment, `xrt_frame.h:44–54`; `xrt_frame_context_add` `:158`). IMU, pose and mask sinks have no such rule (`xrt_tracking.h:174–182`) and usually live inside an object that is a frame node anyway.
- **Two-phase teardown.** `xrt_frame_context_destroy_nodes` (`xrt_frame.h:170`) calls `break_apart` on every node (stop threads, drop frame references), then `destroy` on every node. Nothing can push into freed memory during shutdown.
- **Frame references.** `xrt_frame_reference(&dst, src)` (`xrt_frame.h:131`) increments/decrements the refcount; the last release calls the frame's `destroy`, which is how a producer gets a wrapped driver buffer (e.g. a V4L2 buffer) back. A sink that keeps a frame past the push call takes its own reference (`util/u_sink_queue.c:109`, `tracking/hand/t_hand_tracking_async.c:165`). A sink that only reads during the call takes none (`t_slam`; its SLAM library copies the pixels, see [VIT](../tracking/VIT.md)).

## Implementations
### Endpoints (trackers and recorders)
| Implementer | Sinks it owns | References |
|---|---|---|
| [t_slam](../tracking/t_slam.md) | `cams[0..n-1]`, `imus[0]`, `gt`, `masks[]` | Wiring `t_tracker_slam.cpp:1514–1529`; handlers `receive_frame` `:1275`, `t_slam_receive_imu` `:1228`, ground truth `:1196`, masks `:1212` |
| Hand-tracking async wrapper ([Hand tracking](../tracking/Hand%20tracking.md)) | `left`, `right` frame sinks, also exposed as `sinks.cams[0..1]` | Create `tracking/hand/t_hand_tracking_async.c:316`; receivers `:165`, `:183`. Drops frames while busy; requires left before right |
| 3DoF IMU tracker | one `xrt_imu_sink` | `auxiliary/util/u_tracked_imu_3dof.c:65` |
| EuRoC recorder | a full `xrt_slam_sinks`; writes a EuRoC-format dataset | `auxiliary/tracking/t_euroc_recorder.h:34` |

### Intermediate nodes
All in `auxiliary/util/u_sink_*.c`, declared in `auxiliary/util/u_sink.h`.

| Node | Behaviour | Create (`u_sink.h`) |
|---|---|---|
| Queue | Own thread; bounded; when full, drops the **oldest** frame (`u_sink_queue.c:99–106`); `max_size` 0 means unbounded | `u_sink_queue_create` `:127` |
| Simple queue | Own thread; holds a single frame; a new frame replaces the pending one (latest-only) | `u_sink_simple_queue_create` `:138` |
| Split / split multi | Pushes the same frame to 2 (or up to N) downstream sinks | `u_sink_split_create` `:169`, `u_sink_split_multi_create` `:158` |
| Force genlock | Takes two frames in any order; pushes left then right once their timestamps are close enough. Should not drop | `u_sink_force_genlock_create` `:197` |
| Stereo SBS split / combiner / deinterleaver | Split a side-by-side frame into two; combine two into one stereo frame; deinterleave stereo frames | `:178`, `:188`, `:118` |
| Converters, half scale, quirk | Pixel format conversion; downscale; fix-ups for device quirks | `:50–110`, `:101`, `:147` |
| Debug sink | Tees frames to the debug GUI only while a viewer is open | `u_sink_debug_*` `:223–257` |

## Producers
| Producer | What it pushes | References (`drivers/`) |
|---|---|---|
| V4L2 frame server | Frames into one sink | `v4l2/v4l2_driver.c:860` |
| UVC, video file | Frames | `uvc/uvc_driver.c:284`, `vf/vf_driver.c:218` |
| EuRoC player (replay) | Frames and IMU from a dataset into its own sinks, then forwarded to the sinks it was started with | In `euroc/euroc_player.cpp:492`, `:515`; out `:694`, `:738` |
| WMR | Camera code → WMR source → out sinks | `wmr/wmr_camera.c:429`; `wmr/wmr_source.c:106` (frames), `:160` (IMU) |
| Rift S | Frames and IMU straight into `t_slam` sinks | `rift_s/rift_s_tracker.c:602`, `:542` |
| Vive, RealSense, DepthAI | Frames and IMU | `vive/vive_source.c:140`, `:155`; `realsense/rs_hdev.c:639`, `:677`; `depthai/depthai_driver.cpp:434`, `:588` |
| Mercury hand tracker | Hand bounding boxes into SLAM's `masks[]` sink | `tracking/hand/mercury/hg_sync.cpp:582` |
| `t_slam` itself | Tees every frame, IMU sample and popped pose to its EuRoC recorder | `t_tracker_slam.cpp:1362`, `:1263`, `:809` |

### Pattern: source node
`wmr/wmr_source.c` is the most useful template for our capture layer. The source:
1. Owns its own input sinks (`cam_sinks`, `imu_sinks`, `:72`, `:340–352`) that the hardware code pushes into.
2. Translates hardware time to Monado's monotonic clock inside the receive handler (`xf->timestamp += ws->cam_hw2mono`, around `:101`). This is stage 3 of the [Sensor timestamping pipeline](Sensor%20timestamping%20pipeline.md).
3. Holds back frames until the first IMU sample has arrived.
4. Forwards to the downstream sinks given to `xrt_fs::slam_stream_start` (`:259–265`; interface `include/xrt/xrt_frameserver.h:125`).
5. Tees everything to a EuRoC recorder, so any live session can be recorded as a dataset.

Because the source is an `xrt_fs` with `slam_stream_start`, live capture and the EuRoC player are interchangeable from the builder's point of view.

## Wiring examples
North Star with a DepthAI camera (`targets/common/target_builder_north_star.c:343–366`):
```
DepthAI xrt_fs ──frames──► force_genlock ──► split ──► t_slam cams[0], cams[1]
               │                                └──► hand async left, right
               └──IMU──────────────────────────────► t_slam imus[0]
Mercury ──device masks──► t_slam masks[0]
```
This block references `slam_sinks->imu`, a field that no longer exists (now `imus[]`). It is behind `#ifdef XRT_BUILD_DRIVER_DEPTHAI` and likely does not compile; read it for structure only.

Other examples:
- Rift S and WMR split each tracking camera to SLAM and hands: `rift_s/rift_s_tracker.c:424–425`, `wmr/wmr_hmd.c:1748–1749`.
- Minimal replay harness, no device at all: `drivers/euroc/euroc_runner.c:91–115` (frame context → `t_slam_create` → EuRoC player → `xrt_fs_slam_stream_start`).

## Open questions
- Where our graph needs queue nodes: whether a slow consumer (hand tracking, SLAM frontend) can stall the capture thread, and whether drop-oldest or latest-only is right per consumer.
- Whether stereo pairing is guaranteed by hardware sync or needs a force-genlock node.
