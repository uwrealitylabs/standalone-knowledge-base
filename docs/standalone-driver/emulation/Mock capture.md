---
tags: [standalone-driver, emulation, capture, testing]
aliases: [mock-camera-input, standalone-camera-mock, wrl_mock_capture, mock camera]
status: draft
author: "Claude Opus 5.5"
---
# Mock capture

Part of [Project Standalone](../../Project%20Standalone.md). Compare [EuRoC player](../../emulation/EuRoC%20player.md).

Source: branch `mock-camera-input` of `uwrealitylabs/standalone-monado-driver`, single commit `9363dbe` (2026-10-29) on top of `dev` at `3400e2f`. All code is under `tests/standalone-camera-mock/`; paths below are relative to that folder. Nothing under `src/xrt/` changes and the builder (`target_builder_standalone.c`) is untouched.

Per its README, the run was validated on Apple Silicon macOS (2026-10-28). Independent reproduction by a teammate, Linux/RB5 runs, and confirmation of the receiving interface are still pending.

## What it is
A synthetic camera source plus a test harness:
- **Source.** An `xrt_fs` frame server (see [Sinks](../../runtime/Sinks.md)) that generates images on a producer thread and pushes them into whatever sinks it is started with.
- **Harness.** Feeds the source through a few unmodified Monado sink nodes into an observer, then runs OpenCV corner detection in Python on the received pixels.

It does not create a builder, device, tracker (`t_slam`, Basalt, Mercury), service, compositor or OpenXR app.

## Components
| Package | Contents | Built from |
|---|---|---|
| `capture_mock/` | The source (`native/mock_capture.cpp`, `.h`), an original single-camera harness, fixtures, Python bindings and runner | Own CMake project; compiles Monado's `u_sink_queue.c` directly |
| `pipeline_matrix/` | Test graph: optional format converter, optional force-genlock pairing, queue, observer | Compiles `u_sink_converter.c`, `u_sink_force_genlock.c`, `u_frame.c`, `u_format.c` directly |
| `capture_e2e/` | Joins the source to the graph through a per-camera "guard" sink (`native/session.cpp`) and orchestrates end-to-end scenarios | Links the two above |

Entry point: `run.py` (full suite, or `--scenario four-l8` / `four-rgb`). Python 3.13, OpenCV via `capture_mock/requirements.lock`. `reviewed-sources.json` records hashes of the upstream Monado files it compiles and refuses to run if any changed.

## The source
API in `capture_mock/native/mock_capture.h`:

| Function | Behaviour |
|---|---|
| `wrl_mock_capture_create(_format)(xfctx, &config [, format])` | One camera. Implements `stream_start` into one sink; `slam_stream_start` returns `NOT_IMPLEMENTED` (`mock_capture.cpp:111`) |
| `wrl_mock_capture_group_create(_format)(xfctx, configs, n [, format])` | 1–4 cameras behind one `xrt_fs`. `slam_stream_start` routes `configs[i]` to `sinks.cams[i]` (`:257`, `:276`) |
| `wrl_mock_capture_group_schedule`, `_group_stats` | Set the shared start time; read per-camera counters |

Frames (`make_frame`, `:150`):
- A real `xrt_frame` with its own pixel buffer and refcount; `destroy` frees it (`:44`).
- One byte per pixel, `R8` by default or `L8` via the `_format` variants. Stride is width plus configurable padding.
- Content: a checkerboard plus a 12×12 square that moves sideways each frame, with a 16-byte identity header (camera, sequence, seed, session) in the first pixels and a CRC over the buffer.
- `timestamp` = shared scheduled start (monotonic) + simulated frame time (`:183`); `source_timestamp` = simulated time only. Cameras started together get equal timestamps; that is not a model of exposure sync.

Lifecycle: each camera and the group are frame-context nodes; `break_apart` stops and joins producer threads (`:118`, `:267`). Producer pushes then drops its reference (`:206`).

## The harness
```
mock group ─slam_stream_start─► guard (format L8? size? stride?) ─► [converter] ─► [force-genlock, stereo case]
            ─► u_sink_queue ─► observer ─► Python callback: copy, validate CRC, OpenCV corners, log
```
- Guard: `capture_e2e/native/session.cpp:15`, checks at `:46`.
- Graph: `pipeline_matrix/native/matrix.cpp:92` (genlock), `:97` (queue), `:113` (converter).

Scenarios cover pixel identity against a recorded baseline, layout and metadata, four concurrent streams, queue overload (drop-oldest accounting), malformed frames (wrong format, size, stride), interruption, stop/recreate, frames held after source destruction, and cleanup. README-reported results: 36 Python checks, 5 native checks and 12 scenarios passed; 1,350 frame hashes matched the baseline; overload processed 60 of 240 frames with 180 accounted drops.

## From the driver's point of view
It occupies the same slot as the [EuRoC player](../../emulation/EuRoC%20player.md): a substitute for our capture layer, created and started by the builder.

1. **Create:** `wrl_mock_capture_group_create_format(xfctx, configs, n, XRT_FORMAT_L8)`.
2. **Start:** `xrt_fs_slam_stream_start(mock, sinks)`.

It replaces only the cameras. Blockers to feeding [t_slam](../../tracking/t_slam.md) as written:
- **Refuses t_slam's sinks.** `t_slam_create` returns a bundle with `imus[0]`, `gt` and `masks[]` set (`src/xrt/auxiliary/tracking/t_tracker_slam.cpp:1520–1529`); the group's `start()` returns `NOT_IMPLEMENTED` if any are non-null (`mock_capture.cpp:280–282`). The builder would have to pass a cameras-only copy.
- **No IMU.** Basalt is visual-inertial and won't produce usable poses from images alone.
- **Untrackable content.** A flat pattern with a sliding square has no consistent 3D camera motion.
- **Default format.** `R8` is dropped by `t_slam`, which accepts only L8 and R8G8B8 (`t_tracker_slam.cpp:1314–1318`).
- **Not in the runtime build.** It is a separate shared library under `tests/`; the builder cannot link it until the source moves into `src/xrt/drivers/` and Monado's CMake.

| | EuRoC player | Mock capture |
|---|---|---|
| Frames | Recorded on real headsets or a drone | Synthetic, configurable count/size/rate/stride |
| IMU, ground truth | Yes | No |
| Drives SLAM | Yes | No |
| Needs datasets | Yes (GBs) | No |
| Fault injection | Pause, max speed | Malformed frames, interruption, overload, held frames |
| In Monado's build | Yes | No |

## Assessment (proposal, not a team decision)
- **Redundant as a tracker input.** For the tracer bullet, the EuRoC player does strictly more.
- **Possible remaining uses:**
  - load testing the capture-side graph at our sensor geometry (4 cameras, AR0234-like resolution and rate, camss-like stride) before sensors exist; the per-pixel generator may itself bottleneck at high resolution;
  - small, dataset-free CI for frame-contract plumbing;
  - deliberate fault injection.
- **Mostly redundant:** bit-exact routing checks (hashing EuRoC frames gives the same on real data) and re-verifying upstream nodes already used in production.
- **Maintenance:** hash-locking upstream files breaks on every upstream rebase; validation so far is macOS-only; most of the ~5,400 lines are orchestration, fixtures and reports versus ~540 lines of native source.
- **Gap neither covers:** the team's own capture code (multi-planar V4L2 from camss, 10-bit unpacking, buffer requeue, FPGA timestamp mapping). Exercising it needs emulation below capture (e.g. `vivid` in multi-planar mode); the guard/observer checks would be most valuable placed after that code.

Suggested direction: EuRoC replay for the tracer; keep the mock only if load testing or CI is planned, moved into `src/xrt/drivers/` as a builder-selectable source with the hash-locking dropped.

## Open questions
- Whether the team wants the mock for load testing/CI, or parks the branch.
- If kept: where it lives in the tree, and whether it gains a synthetic IMU (only useful for plumbing, not SLAM).
- Linux and RB5 reproduction of the reported results.
