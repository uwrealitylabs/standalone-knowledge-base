---
tags: [tracking, hands, monado]
aliases: [Mercury, hand tracker]
status: draft
author: "Claude Opus 5.5"
---
# Hand tracking

Required for the headset. Open whether to build from scratch as a team learning exercise or integrate Monado's Mercury. The tracer ([Tracer bullet roadmap](../Tracer%20bullet%20roadmap.md)) uses Mercury.

## Monado structure
Two layers, created by the [Monado headset driver](../standalone-driver/Monado%20headset%20driver.md)'s builder:
- **Synchronous tracker** (`t_hand_tracking_sync`): processes frames synchronously and returns two hands. Mercury implements it via `t_hand_tracking_sync_mercury_create(calib, create_info, models_folder)`. Code in `src/xrt/tracking/hand/mercury/`. Creation info includes per-camera extras (image boundaries, camera orientations).
- **Asynchronous wrapper** (`t_hand_tracking_async`): `t_hand_tracking_async_default_create(xfctx, sync)`. Runs the sync tracker on its own thread, exposes frame sinks ([Sinks](../runtime/Sinks.md)), and offers `get_hand(...)` for devices.

Monado's standalone hand-tracking driver (`ht_device_create`) is for cameras with no headset driver. In our driver, create the async tracker directly.

## Open questions
- Mercury is built around one stereo calibration (one pair). Our plan uses front, side and bottom cameras ([Hardware plan](../hardware/Hardware%20plan.md)). Options:
  - Separate instances per pair, results merged in device code.
  - One pair per hand region.
  - Extend Mercury.
  Decide before scoping the hand-tracking workstream.
- Build from scratch vs integrate.
