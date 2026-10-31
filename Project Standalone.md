---
tags: [hub, overview]
aliases: [UWRL standalone headset, standalone headset]
status: draft
---
# Project Standalone

UW Reality Labs' in-house standalone XR headset. Compute is a Qualcomm RB5. Display and optics come from Project North Star, in a modified form factor. The software stack is a Godot launcher running on a Monado fork with custom drivers, on an Arch-based OS.

## Map
| Folder | Notes |
|---|---|
| (top level) | [Tracer bullet roadmap](Tracer%20bullet%20roadmap.md) |
| `hardware/` | [Hardware plan](hardware/Hardware%20plan.md), [Display](hardware/Display.md), [Sensor sync architecture](hardware/Sensor%20sync%20architecture.md) |
| `platform/` | [OS and kernel](platform/OS%20and%20kernel.md) |
| `runtime/` | [Monado headset driver](runtime/Monado%20headset%20driver.md), [Sensor timestamping pipeline](runtime/Sensor%20timestamping%20pipeline.md) |
| `tracking/` | [Tracker sinks](tracking/Tracker%20sinks.md), [t_slam](tracking/t_slam.md), [Hand tracking](tracking/Hand%20tracking.md) |

## Software components
| Layer   | Scope                                                                                                                                                                                                    |
| ------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| App     | Launcher in Godot via OpenXR. Starts other OpenXR apps and composites non-XR apps as in-app windows. Repo: https://github.com/uwrealitylabs/standalone-launcher                                          |
| CV      | SLAM: hardware-accelerated (e.g. GPU) implementation, not Basalt out of the box. Hand tracking: required; open whether to build from scratch as a learning exercise or integrate (e.g. Monado's Mercury) |
| Runtime | Monado fork with drivers for our hardware → [Monado headset driver](runtime/Monado%20headset%20driver.md)                                                                                                                                    |
| OS      | Arch-based; see [OS and kernel](platform/OS%20and%20kernel.md)                                                                                                                                                                        |
| Drivers | Displays, cameras, IMU. Panel driver ported from the RPi 5 build → [Display](hardware/Display.md)                                                                                                                           |

## Ownership
- Hardware lead: everything below the kernel (FPGA firmware, camera sync, hardware timestamping).
- Software team: kernel drivers and up.

## Launcher dependency
Launching other OpenXR apps and compositing alongside them depends on Monado's multi-client service mode and overlay support. Confirm these cover the launcher's needs before building on them.
