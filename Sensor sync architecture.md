---
tags: [hardware, sync, timing, tracking]
aliases: [sync architecture, FSYNC, hardware sync]
status: draft
---
# Sensor sync architecture

Hardware that makes sensor capture times precise and puts them in a shared time base. Owned by the hardware lead (below the kernel). The software side is the [Sensor timestamping pipeline](Sensor%20timestamping%20pipeline.md).

## Why it matters
Nearly every algorithm on the headset fuses measurements from multiple sensors and assumes it knows when each was taken. Timing error becomes geometric error that grows with motion speed.

| Consumer | Assumption | Failure when violated |
|---|---|---|
| VIO / SLAM ([t_slam](t_slam.md)) | Frame times align with IMU samples | 200°/s head turn × 5 ms error = 1° ≈ 19 px feature misprediction (AR0234, ~100° HFOV over 1920 px). Tracking degrades during fast motion |
| Stereo depth | Both cameras of a pair expose at the same instant | 6 cm baseline, object at 1 m: 1° error moves depth to ~0.77–1.4 m. Hand depth jitters |
| Multi-view hand tracking | Front, side, bottom views are the same moment | A fast hand appears in two places |
| RGB–mono fusion | Known capture time per row | IMX678 rolling shutter spreads exposure over several ms; needs a per-row time model |
| Eye tracking | IR LEDs fire only during eye exposure | Wasted power and IR dose; no alternating glint patterns |
| Rendering | Pose and vblank timestamps share a clock | Wrong pose prediction. Monado uses `CLOCK_MONOTONIC` for both |

## Reference design
One master clock decides when sensors capture and records those times:
- **Trigger lines (FSYNC).** Drive external-trigger inputs so each stereo pair exposes simultaneously. Other groups run at integer multiples or divisors of the base rate.
- **One counter as time base.** Timestamp each trigger pulse and each IMU data-ready interrupt with the same free-running counter. Sub-microsecond, single domain.
- **Metadata in the stream.** Embed trigger timestamp, frame counter, and exposure time per frame (embedded-data line or MIPI virtual-channel packet).
- **LED strobe.** The same trigger logic gates the eye-tracking IR illuminators.

The original proposal had the FPGA as master for all sensors. The current split routes only the bottom and eye cameras through the FPGA (see [Hardware plan](Hardware%20plan.md)), so the design has to be adjusted.

## Why V4L2 buffer timestamps are not enough
- Qualcomm's camss stamps buffers at the SoC frame interrupt, after exposure and readout.
- An FPGA in the path adds variable delay between exposure and arrival.
- Stamping IMU samples in a kernel IRQ handler adds tens to hundreds of µs of jitter plus scheduling outliers.

## Validation
The trigger source toggles an LED visible to all cameras on every trigger. Any camera that sees the LED in the wrong state is exposing at the wrong time.

## Open questions
- Sync for the direct-to-RB5 cameras (front stereo, RGB, side) and the IMU. Who drives their triggers, and how do they share a time base with the FPGA-routed cameras?
- Trigger/slave-mode support on AR0234, OV9281, IMX678.
- How timestamps reach the software side for each path (embedded metadata vs other).
