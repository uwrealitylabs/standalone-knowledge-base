---
tags: [monado, tracking, interfaces]
aliases: [sinks, xrt_slam_sinks, frame sinks]
status: draft
---
# Tracker sinks

The push interface between capture and trackers in the [[Monado headset driver]]. Four data types, each with a sink that has a single push function.

## Data types
**`xrt_frame`** (one camera image)
| Field | Meaning |
|---|---|
| `reference`, `destroy`, `owner` | Refcount and release callback; lets a producer wrap a driver buffer and get it back when the last consumer drops it |
| `width`, `height`, `stride`, `size`, `data` | Dimensions, row pitch, total size, CPU pointer to pixels |
| `format` | `xrt_format`: L8, RGB variants, YUYV/UYVY, Bayer |
| `stereo_format` | Single view or stereo pair in one image (e.g. side-by-side) |
| `timestamp`, `source_timestamp` | ns. `timestamp` in Monado's monotonic clock; `source_timestamp` keeps the device's original clock |
| `source_sequence`, `source_id` | Frame counter; originating `xrt_fs` |

**`xrt_imu_sample`**: `timestamp_ns`, `accel_m_s2` and `gyro_rad_secs` as `xrt_vec3_f64`, in the IMU's own axes, SI units.

**Ground-truth pose**: timestamped pose from an external reference. Evaluation and recording only.

**Hand masks**: image regions covered by hands, so SLAM can ignore features on moving hands. Check the current header for layout.

## Sink interfaces
- `xrt_frame_sink::push_frame(sink, frame)`. Implementations also implement `xrt_frame_node` and register with an `xrt_frame_context` in their constructor, so the pipeline is torn down together. IMU, pose and hand-mask sinks follow the same pattern.
- SLAM receives them bundled as `xrt_slam_sinks`: `cam_count`, `imu_count`, `cams[8]`, `imus[3]`, `gt`, plus hand masks. Null sinks are disabled. Index in `cams[]` is the camera index the calibration refers to.

## Sink implementations
| Category | Examples |
|---|---|
| Trackers | SLAM camera/IMU/ground-truth sinks ([[t_slam]]); hand-tracking async wrapper camera sinks ([[Hand tracking]]) |
| Threading | Bounded queue (`u_sink_queue_create`), simple queue |
| Routing | `u_sink_split_create` (one frame to two sinks), multi-way split, stereo side-by-side splitter, combiner, genlock helper |
| Conversion | Format converters (grayscale, RGB, YUV), deinterleaver, quirk-fixing node |
| Tooling | EuRoC recorder, debug GUI views |

Producers in upstream Monado: V4L2 frame server, EuRoC player, WMR/Vive/RealSense sources, Rift S camera code. WMR and Rift S use the split node to fan the same frames to SLAM and hand tracking, which is the pattern our driver needs. The IMU goes to SLAM only.

## Contract points
- **Pixel format.** Mono tracking cameras arrive as L8. If V4L2 delivers packed 10-bit raw, capture or a conversion node unpacks it. That costs CPU per frame; decide where it happens.
- **Timestamps.** `timestamp` in Monado's clock, at the time the contract specifies (ideally mid-exposure), identical across both frames of a stereo pair. SLAM pairs frames by timestamp. See [[Sensor timestamping pipeline]].
- **Camera index mapping.** `cams[n]` matches calibration entry n. Make the mapping explicit in builder config.
- **IMU convention.** SI units, IMU axes, steady rate, same clock as frames. Whether the IMU-to-camera transform is applied by SLAM or device code is decided in calibration, not in the samples.
- **Hold time.** If frames wrap V4L2 buffers, put a bounded queue in front of any consumer that may hold frames long. Otherwise a slow tracker starves capture of buffers.
