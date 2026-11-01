---
tags: [hardware, sensors]
aliases: [hardware, sensor list, BOM]
status: draft
author: "Claude Opus 5.5"
---
# Hardware plan

Part of [Project Standalone](../Project%20Standalone.md).

## Compute
- Qualcomm RB5 dev kit (QRB5165, Snapdragon 865 class, Adreno 650 GPU). A 2020-era SoC.
- The QRB5165 supports up to 7 concurrent cameras, and the dev kit's mezzanine exposes fewer connectors than that.
- An FPGA aggregates the cameras that exceed the limit. Current split (per hardware lead):
  - **Via FPGA:** bottom and eye-tracking cameras.
  - **Direct to RB5:** front stereo, front RGB, side cameras, IMU.

## Cameras
| Group | Sensor | Shutter | Purpose | Path |
|---|---|---|---|---|
| Front stereo pair | AR0234 mono, no IR-cut | Global | SLAM, depth, hand tracking | RB5 |
| Front RGB | IMX678 | Rolling | Color depth, object recognition, demo recording with AR overlay, color temperature | RB5 |
| Side stereo pairs | AR0234 mono | Global | Extended hand-tracking coverage | RB5 |
| Bottom | OV9281 mono | Global | Hand tracking with hands at rest (Vision Pro-style ergonomics) | FPGA |
| Eye tracking ×2 | OV9281 mono, IR-pass filter | Global | Gaze | FPGA |

Bottom and eye cameras were originally specified as SC035HGS.

Sensor notes:
- AR0234: 1920×1200 at up to 120 fps. More resolution than SLAM needs; bin or crop to save bandwidth.
- IMX678 is rolling shutter, so fusing it with the mono cameras is always approximate (see [Sensor sync architecture](Sensor%20sync%20architecture.md)).

## IMU
- LSM6DSV family (per hardware lead), whichever variant is cheaper; on-chip ML/fusion not needed.
- North Star's IMU does not carry over. The Deck X gets its IMU from the embedded RealSense T261, which Intel has discontinued. The T261 tracks on its own clock, so it cannot be aligned with our cameras.

## Display and optics
From Project North Star → [Display](Display.md).

## Also required
- **IR illuminators** (850 or 940 nm) for eye tracking. The IR-pass filter is useless without them. Strobe tied to the eye-camera exposure (see [Sensor sync architecture](Sensor%20sync%20architecture.md)).
- **Power and thermals.** Many cameras plus the SoC under sustained load. The RB5 dev kit is physically large; this revision likely needs a tethered compute pack.

## Open questions
- Exact side-camera count (one pair or one pair per side) and total camera count on the direct RB5 path versus its connector limit.
- External-trigger / slave-mode support for each sensor (datasheets).
- Passthrough plan: one RGB camera gives mono passthrough with parallax error. North Star is optical see-through, so passthrough may not be a goal.
