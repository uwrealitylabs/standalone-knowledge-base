---
tags: [software, timing, tracking, monado]
aliases: [timestamping, timestamp pipeline]
status: draft
---
# Sensor timestamping pipeline

Software between V4L2/IIO capture and Monado that turns hardware capture times into `CLOCK_MONOTONIC` timestamps. Depends on [[Sensor sync architecture]]. Its output feeds the capture source in the [[Monado headset driver]].

The boundary with the hardware lead's work depends on how timestamps are delivered; see the open questions.

## Stages
| # | Stage | Detail |
|---|---|---|
| 1 | Metadata extraction | Parse embedded timestamp, frame counter, exposure time from each buffer |
| 2 | Exposure-center correction | Global shutter: trigger time + exposure/2. Auto-exposure makes trigger-only stamps drift by ms. IMX678: store start time + line readout time |
| 3 | Clock-domain translation | Map hardware counter ticks to `CLOCK_MONOTONIC`. Method: hardware pulses a GPIO while logging its counter; kernel stamps the IRQ; running linear fit (offset + drift) averages out IRQ jitter |
| 4 | Frame-set assembly | Group stereo pairs / hand-tracking sets by frame counter. Detect drops; discard incomplete sets instead of mispairing |
| 5 | Delivery | Push frames and IMU samples into the SLAM sinks ([[t_slam]]) and hand-tracking sinks in the monotonic domain |
| 6 | Residual offset calibration | Camera–IMU calibration (Kalibr, Basalt tools) estimates a remaining offset. With hardware sync it should be small and constant; large or unstable means an upstream bug |
| 7 | Monitoring | Log trigger-to-arrival latency, jitter, drop rate. Otherwise sync bugs look like algorithm bugs |

## Open questions
- Which stages live in kernel drivers vs userspace vs FPGA firmware.
- Timestamp source for direct-to-RB5 cameras if they carry no embedded metadata.
