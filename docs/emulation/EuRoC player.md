---
tags: [emulation, monado, replay, datasets]
aliases: [EuRoC, euroc_player, dataset replay, EUROC_PATH, slambatch]
status: draft
author: "Claude Opus 5.5"
---
# EuRoC player

Part of [Tracer bullet roadmap](../Tracer%20bullet%20roadmap.md).

Source references are to `uwrealitylabs/standalone-monado-driver`, branch `dev` at `6bbefe8`, relative to `src/xrt/`.

Monado's dataset replay source. It reads a dataset in EuRoC format and plays camera frames, IMU samples and ground truth into sinks, in real time or faster. It implements `xrt_fs` (a frame server, see [Sinks](../runtime/Sinks.md)), so to every consumer it looks like a live camera-and-IMU source. Code: `drivers/euroc/euroc_player.cpp`; interface `drivers/euroc/euroc_interface.h`; built with `XRT_BUILD_DRIVER_EUROC`.

## Dataset format
EuRoC is the layout of the ETH EuRoC MAV datasets, also used by the [Monado SLAM datasets](https://huggingface.co/datasets/collabora/monado-slam-datasets) (egocentric, recorded on real XR headsets; preferred for us over the drone-based originals).

```
<dataset>/mav0/
  cam0/data.csv, cam0/data/<timestamp_ns>.png      one folder per camera
  cam1/...
  imu0/data.csv                                     timestamp, gyro xyz, accel xyz
  <gt device>/data.csv                              timestamp, px py pz, qw qx qy qz
```
The player looks for ground truth in `vicon0`, `mocap0`, `state_groundtruth_estimate0` or `leica0` (`euroc_player.cpp:205`), or the folder named by `EUROC_GT_DEVICE_NAME`.

## Where it sits
It replaces the capture layer. Everything above the sinks runs unchanged.

```
Live:    our capture (V4L2 cameras + IMU reader + clock mapping) ─┐
                                                                   ├─ xrt_slam_sinks ─► t_slam ─► VIT/Basalt
Replay:  EuRoC player ─────────────────────────────────────────────┘                    ▲ get_tracked_pose
                                                                          our HMD xrt_device ◄── core
```

| Replaced | Still real |
|---|---|
| Camera capture, IMU reading, hardware-to-monotonic clock mapping, our calibration | [t_slam](../tracking/t_slam.md), [VIT](../tracking/VIT.md)/Basalt, our device's pose composition, Monado core, compositor, apps |

## How a driver interfaces with it
Two calls. Both live in the builder's open function, in place of creating live capture.

1. **Create:** `euroc_player_create(xfctx, path, &config)` (`euroc_player.cpp:978`) returns an `xrt_fs` already registered with the frame context (`:1045`). Fill `config` first with `euroc_player_fill_default_config_for` (`euroc_interface.h:86`), which reads the dataset to find its camera count and whether it has colour and ground truth.
2. **Start:** `xrt_fs_slam_stream_start(player, sinks)` (`euroc_player.cpp:779`). The player stores the sinks and its playback thread pushes frames (`:694`), IMU samples (`:738`) and, if enabled, ground truth into them.

Sketch of a replay branch in our builder (proposed, not implemented; the task is described in [SLAM tracer bullet from EuRoC](../standalone-driver/task-guides/SLAM%20tracer%20bullet%20from%20EuRoC.md)):
```c
#include "euroc/euroc_interface.h"
#include "tracking/t_tracking.h"

struct euroc_player_config ep_cfg;
euroc_player_fill_default_config_for(&ep_cfg, replay_path);
ep_cfg.playback.play_from_start = true;      // default waits for the debug GUI

struct t_slam_tracker_config st_cfg;
t_slam_fill_default_config(&st_cfg);
st_cfg.cam_count = ep_cfg.playback.cam_count;
st_cfg.submit_from_start = true;             // default discards everything
st_cfg.slam_config = basalt_toml_for_dataset; // e.g. Basalt's msdmi.toml

struct xrt_tracked_slam *xts = NULL;
struct xrt_slam_sinks *sinks = NULL;
t_slam_create(xfctx, &st_cfg, &xts, &sinks);
t_slam_start(xts);

struct xrt_fs *src = euroc_player_create(xfctx, replay_path, &ep_cfg);
xrt_fs_slam_stream_start(src, sinks);
// Give xts to the HMD device; it queries xrt_tracked_slam_get_tracked_pose().
```
The same sequence, minus the device, is `drivers/euroc/euroc_runner.c:91–115`.

## What must match the dataset, not our hardware
- **SLAM calibration and config.** `SLAM_CONFIG` points at the Basalt TOML for that dataset (Basalt ships `data/vit/euroc.toml.in`, `msdmi`, `msdmg`, `msdmo`, etc.; see [VIT](../tracking/VIT.md)).
- **Camera count:** `t_slam`'s `cam_count` equals the player's.
- **`device_from_imu`** and other head transforms in the device: identity or the dataset's, not ours.
- **Image scale.** `EUROC_SCALE` resizes frames; the calibration must be scaled to match.

## Timestamps
By default (`use_source_ts = false`) timestamps are rebased to playback start: `mapped = start_ts + offset_ts + (ts − base_ts) / speed` (`euroc_player.cpp:416–419`), where `base_ts` is the dataset's first sample and `start_ts` the monotonic time playback began (`:617–618`). Rebased timestamps line up with the monotonic times the core queries poses at, so keep the default when a device and apps are involved. `monado-cli slambatch` uses source timestamps because it evaluates offline.

## Configuration
Environment variables, read by `euroc_player_fill_default_config_for` (`euroc_player.cpp:42–55`); every field can also be set in code.

| Variable | Default | Effect |
|---|---|---|
| `EUROC_PLAY_FROM_START` | false | If false, playback waits for the Start button in the debug GUI |
| `EUROC_USE_SOURCE_TS` | false | Use dataset timestamps instead of rebasing |
| `EUROC_SPEED` / `EUROC_MAX_SPEED` | 1.0 / false | Playback rate; max speed pushes as fast as possible |
| `EUROC_CAM_COUNT` | dataset's | Stream fewer cameras |
| `EUROC_GT` | dataset's | Push ground truth into the `gt` sink |
| `EUROC_SKIP_FIRST` | `0%` | Skip a percentage or seconds at the start |
| `EUROC_SCALE` | 1.0 | Resize frames |
| `EUROC_PAUSED`, `EUROC_SEND_ALL_IMUS_FIRST`, `EUROC_PRINT_PROGRESS`, `EUROC_LOG` | | Debugging |

## Other ways Monado uses it
| Use | How | Relevance to us |
|---|---|---|
| Headless evaluation | `monado-cli slambatch` → `euroc_run_dataset` (`drivers/euroc/euroc_runner.c`); writes trajectory CSVs | Check `t_slam` + Basalt on the RB5 with no device or core |
| Built-in fake headset | `EUROC_PATH` enables an auto-prober (`targets/common/target_lists.c:267`) that creates `drivers/euroc/euroc_device.c`; the prober's tracking factory creates `t_slam` and starts the player (`state_trackers/prober/p_tracking.c`, around `:255–279`) | Proves Basalt + core + apps before our driver exists. Replaces our whole driver |
| Prober substitution | With `EUROC_PATH` (or `VF_PATH`) set, `p_open_video_device` returns the player for any camera opened through the prober (`state_trackers/prober/p_prober.c:1277–1300`) | Only affects drivers that open cameras through the prober; our CSI cameras won't |

Because `EUROC_PATH` triggers the last two, a replay mode in our builder should read its own variable for the dataset path.

## Recording our own datasets
`euroc_recorder_create(xfctx, path, cam_count, imu_count, record_from_start)` (`auxiliary/tracking/t_euroc_recorder.h:35`) returns sinks that write a EuRoC-format dataset. `t_slam` already tees everything it receives into one (toggle in the debug GUI), and `wmr_source` does the same for WMR headsets. Once our live capture works, recordings from our own sensors replay through this same path, which is where accuracy tuning becomes meaningful ([Tracer bullet roadmap](../Tracer%20bullet%20roadmap.md)). The recorder writes ground truth to `mav0/gt/`, which the player only finds with `EUROC_GT_DEVICE_NAME=gt`.

## Open questions
- Which dataset the tracer uses first (a Monado SLAM dataset with two cameras is the natural start).
- Name of our builder's replay-path variable and how replay vs live is selected (config file vs environment).
