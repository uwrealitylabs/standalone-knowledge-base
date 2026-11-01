---
tags: [standalone-driver, task-guide, slam, tracer-bullet, emulation]
aliases: [SLAM tracer bullet, SLAM replay task, EuRoC SLAM tracer]
status: draft
author: "Claude Opus 5.5"
---
# SLAM tracer bullet from EuRoC

Part of [Monado headset driver](../Monado%20headset%20driver.md). Implements the SLAM half of Stage 2 in the [Tracer bullet roadmap](../../Tracer%20bullet%20roadmap.md).

Code references: `uwrealitylabs/standalone-monado-driver`, branch `dev` at `6bbefe8`, paths relative to `src/xrt/` unless noted. The task itself is decided; the implementation details below are suggestions to adapt.

## Goal
Recorded camera and IMU data reaches an OpenXR app through our driver, running on the EB5:

```
EuRoC dataset ─► EuRoC player (xrt_fs) ─► t_slam sinks ─► t_slam ──VIT──► libbasalt.so
                                                            ▲ xrt_tracked_slam_get_tracked_pose(t)
                                  standalone HMD xrt_device ◄── Monado core ◄── hello_xr
          (all created and connected once, in standalone_open_system_impl)
```

This proves the whole SLAM path (build, loading, wiring, pose query, core) before any of our sensors or capture code exist. Later stages replace the EuRoC player with live capture behind the same sinks; nothing above them should need to change.

## Read first
| Note | Why |
|---|---|
| [EB5 Monado Development Setup](../guides/EB5%20Monado%20Development%20Setup.md) | The Ubuntu 24.04 container all building and running happens in |
| [Monado headset driver](../Monado%20headset%20driver.md) | What the builder, device and trackers each do; push below the device, pull above it; current state of our code |
| [Sinks](../../runtime/Sinks.md) | How data reaches trackers; who owns what; teardown |
| [EuRoC player](../../emulation/EuRoC%20player.md) | The replay source, the two calls a builder makes, configuration, timestamp rebasing, a code sketch |
| [t_slam](../../tracking/t_slam.md) | The SLAM adapter, its pose path and gotchas |
| [VIT](../../tracking/VIT.md) | How `t_slam` loads and calls Basalt; Basalt's behaviour and asserts |

Code worth reading alongside:
- `drivers/twrap/twrap_slam.c` (228 lines): the smallest `xrt_device` built around `t_slam`, including the Basalt axis correction (`:77`).
- `drivers/euroc/euroc_runner.c:91–115`: the replay wiring with no device.
- `drivers/rift_s/rift_s_tracker.c:634–680`: a complete pose composition chain on real hardware.

## Scope
**In:** building Basalt and Monado in the development container, Monado build options, builder wiring, the device's pose query, one dataset, running headless on the EB5.

**Out:** live capture and our sensors, hand tracking ([Hand tracking](../../tracking/Hand%20tracking.md)), calibrating our hardware, display and panel bring-up ([Display](../../hardware/Display.md)), GPU acceleration, and the [Mock capture](../emulation/Mock%20capture.md) branch, which can't feed SLAM. A 3DoF fallback is optional.

## Environment
The EB5's stock image is Ubuntu 18.04 with a 4.19 kernel, too old for Monado's dependencies. All builds and runs happen in the Ubuntu 24.04 container from [EB5 Monado Development Setup](../guides/EB5%20Monado%20Development%20Setup.md), over SSH; see also [OS and kernel](../../platform/OS%20and%20kernel.md).

Consequences for this task:
- **Containers are discarded on exit** (`--rm`). Anything installed to system paths inside a running container is lost. Either add build dependencies to the Dockerfile and rebuild the image, or keep artifacts under `/src` (host `~/src`), which is mounted.
- **The container user has no root.** The Dockerfile switches to an unprivileged user and installs no `sudo`, so scripts that call `sudo apt-get` won't work inside it. Package installs belong in the Dockerfile.
- **No display.** The setup guide builds the null compositor for headless testing; Basalt's own GUI must also stay off.
- **The EB5 is aarch64.** Prebuilt Basalt packages are x86-only, so Basalt is built from source. The same Dockerfile also builds on an x86 Linux laptop, which can be used for faster iteration; the exit criteria are checked on the EB5.

## Steps
### 1. Extend the image and build Basalt
- Clone Collabora's fork with its submodules into `~/src` on the host: `git clone --recursive https://gitlab.freedesktop.org/mateosss/basalt.git`.
- Its `scripts/install_deps.sh` lists the Ubuntu packages it needs (TBB, Boost, GLEW, fmt, lz4, gtest, OpenCV and others). Add that list to the Dockerfile's `apt-get install` line and rebuild the image; check that every package exists on 24.04 and arm64.
- Inside the container, build only the library and install it under the mounted volume:
  ```sh
  cd /src/basalt
  cmake --preset library -DCMAKE_INSTALL_PREFIX=/src/install
  cmake --build build --target install
  ```
  The `library` preset sets `BASALT_BUILD_SHARED_LIBRARY_ONLY`, producing `libbasalt.so` with the VIT interface already compiled in; no changes are needed to make it loadable. Config files (`*.toml`) land in `/src/install/share/basalt/` with their paths filled in for that prefix. If the build runs out of memory on the EB5, limit parallel jobs (`-j`).
- Set `show-gui=0` in the TOML you will use; the shipped files set it to 1, which overrides Monado's `SLAM_UI` and needs a display.
- Its VIT major version must match Monado's header (2).

### 2. Build our Monado fork with SLAM and EuRoC
- Clone our fork into `~/src/monado` as in the setup guide (`uwrealitylabs/standalone-monado-driver`, branch `dev`).
- Add to the setup guide's CMake command: `-DXRT_FEATURE_SLAM=ON -DXRT_BUILD_DRIVER_EUROC=ON -DXRT_BUILD_DRIVER_STANDALONE=ON`, keeping `-DXRT_MODULE_COMPOSITOR_NULL=ON`. `t_slam` is only compiled with `XRT_FEATURE_SLAM` (`auxiliary/tracking/CMakeLists.txt:71–72`); OpenCV, which it requires, is already in the image. Check CMake's summary shows `FEATURE_SLAM`, `DRIVER_EUROC` and `DRIVER_STANDALONE` as ON.
- In the container: `export VIT_SYSTEM_LIBRARY_PATH=/src/install/lib/libbasalt.so`.

### 3. Check SLAM without our driver
- Download a short sequence into `~/src`. Basalt's README suggests `MOO09_short_1_updown` from the [Monado SLAM datasets](https://huggingface.co/datasets/collabora/monado-slam-datasets) (Odyssey+, two cameras), which pairs with `msdmo.toml`.
- Run `monado-cli slambatch <dataset> /src/install/share/basalt/msdmo.toml <output dir>` (`targets/cli/cli_cmd_slambatch.c`). `t_slam` writes `tracking.csv` (estimated trajectory) plus `prediction.csv`, `filtering.csv` and `timing.csv` to the output directory (`auxiliary/tracking/t_tracker_slam.cpp:1561–1565`).
- Compare `tracking.csv` with the dataset's ground truth (`mav0/<gt device>/data.csv`), e.g. with `evo`. Keep this output as the reference for step 6.
- This isolates problems in the Basalt build or config from problems in our driver. If it fails, steps 4–6 will too.

### 4. Builder: create the tracker and connect the player
In `standalone_open_system_impl` (`targets/common/target_builder_standalone.c:61`), using the `xfctx` it already receives:
1. `t_slam_fill_default_config`, then set `submit_from_start = true`, `slam_config` (the TOML), and `cam_count` from the player.
2. `t_slam_create(xfctx, &cfg, &xts, &sinks)`, then `t_slam_start(xts)`.
3. `euroc_player_fill_default_config_for`, set `playback.play_from_start = true`, then `euroc_player_create(xfctx, path, &cfg)`.
4. `xrt_fs_slam_stream_start(player, sinks)`. No split nodes are needed while there is no hand tracking.
5. Pass `xts` to the HMD device.

Read the dataset path from our own environment variable or config entry, not `EUROC_PATH` (that also enables Monado's built-in fake EuRoC headset). The [EuRoC player](../../emulation/EuRoC%20player.md#how-a-driver-interfaces-with-it) note has a code sketch.

The frame context belongs to the system-devices object, so the player and `t_slam` are torn down with the device set; no extra cleanup code is needed.

### 5. Device: answer pose queries from the tracker
In `drivers/standalone/standalone_hmd.c`:
1. `standalone_create()` takes the `xrt_tracked_slam *`.
2. `standalone_get_tracked_pose` (`:99`) replaces its relation-history lookup with `xrt_tracked_slam_get_tracked_pose(xts, at_timestamp_ns, &rel)`.
3. Apply the Basalt-to-OpenXR axis correction (as in `twrap_slam.c:77`), then `device_from_imu`. Identity is acceptable for dataset replay.
4. While SLAM has not initialised, the returned relation is not tracked; return it with invalid flags rather than a fabricated pose (see `twrap_slam.c:105–113`).

`get_view_poses` already uses `u_device_get_view_poses`, which calls `get_tracked_pose`, so per-eye poses follow automatically.

### 6. Run end to end, headless
- In the container, with `XR_RUNTIME_JSON` set as in the setup guide: start `monado-service` with `XRT_COMPOSITOR_NULL=1` (selects the null compositor when the main one is also built, `targets/common/target_instance.c:43`).
- Run a client in a second shell in the same container (`docker exec`). `hello_xr` is not in the image; build it from Khronos' OpenXR-SDK-Source under `/src` and use its Vulkan graphics plugin. Without a GPU driver in the container, Vulkan falls back to Mesa's software renderer, which is enough with the null compositor.
- Set `SLAM_WRITE_CSVS=1` and `SLAM_CSV_PATH` (under `/src`) so `t_slam` writes `tracking.csv` from inside our driver, for comparison with step 3.

## Exit criteria
On the EB5, in the development container:
- `slambatch` trajectory error is comparable to published stock Basalt results on the same dataset.
- The trajectory `t_slam` writes inside our driver matches the `slambatch` run on the same dataset.
- The client receives a tracked head pose that follows the dataset's trajectory (e.g. logged view poses); no crashes on start or shutdown.
- Changes limited to our builder, driver and the Dockerfile. Anything requiring core Monado changes is flagged before it's made.

## Gotchas
| Symptom | Likely cause |
|---|---|
| Nothing tracks; Basalt never initialises | `SLAM_SUBMIT_FROM_START` left false (`t_slam` discards input) or `EUROC_PLAY_FROM_START` left false (player waits for the debug GUI) |
| `t_slam_create` fails | No `SLAM_CONFIG`/`slam_config`, wrong library path, or VIT major version mismatch |
| Basalt aborts on start | Missing calibration, or `cam_count` < 2 |
| Basalt aborts or fails to open a window | `show-gui=1` in the Basalt TOML; there is no display |
| Basalt aborts during playback | Cameras out of order or stereo timestamps differ (shouldn't happen with the player; will matter for live capture) |
| Pose is rotated 90° | Missing Basalt axis correction |
| Head pose frozen or jumping | Device returns stale or untracked relations as valid; or `EUROC_USE_SOURCE_TS` set, so timestamps don't line up with query times |
| Installed library or packages gone | Installed inside a `--rm` container instead of the image or `/src` |
| CMake complains the source directory changed | The build directory was configured in a container that mounted the source at a different path; keep mounts consistent or reconfigure |

## Open questions
- Whether Basalt should be baked into the image (reproducible, slower to change) or stay in the mounted `/src/install` prefix (as above).
- Which dataset sequence to standardise on for the team.
- Name of the replay-path variable and whether replay vs live is selected by environment or config file.
