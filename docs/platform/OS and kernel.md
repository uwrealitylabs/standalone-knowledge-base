---
tags: [os, kernel, linux]
aliases: [OS, kernel, Arch]
status: draft
author: "Claude Opus 5.5"
---
# OS and kernel

Part of [Project Standalone](../Project%20Standalone.md).

## Distribution
Plan is not detailed yet. Leaning toward **Arch Linux ARM plus a custom package repository** (PKGBUILDs for kernel, Monado, drivers) instead of forking a distro. Nothing in the requirements needs a fork.

## Current dev environment
- The team's RB5 runs Ubuntu 18.04, accessed remotely over Tailscale/SSH.
- Newer software (e.g. Monado build deps) runs in an Ubuntu 24.04 Docker container; no in-place OS upgrade. Setup: [EB5 Monado Development Setup](../standalone-driver/guides/EB5%20Monado%20Development%20Setup.md).
- The image ships no vendor Vulkan driver (Adreno GLES/EGL only). A self-built Mesa Turnip works in containers on the Adreno 650 through the KGSL kernel driver.

## Kernel choice
|          | Mainline (Linaro upstream SM8250)                              | Qualcomm vendor BSP (4.19-era)            |
| -------- | -------------------------------------------------------------- | ----------------------------------------- |
| GPU      | Freedreno/Turnip; good fit for Monado                          | Proprietary Adreno                        |
| Cameras  | Raw RDI capture via camss; no ISP, so RGB needs GPU debayering | CamX/ISP                                  |
| DSP/NPU  | Hexagon harder to use                                          | Available                                 |
| Tradeoff | Open, modern, fits Arch                                        | Old kernel, vendor lock-in, poor Arch fit |

Mainline fits an Arch + Monado stack as long as losing the ISP and DSP is acceptable.

## Open questions
- Mainline vs vendor kernel (not decided).
- Whether the GPU SLAM plan and RGB debayering fit the GPU budget without the ISP/DSP.
