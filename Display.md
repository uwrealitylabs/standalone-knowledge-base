---
tags: [hardware, display, drivers]
aliases: [displays, panels, panel driver]
status: draft
---
# Display

Part of [Hardware plan](Hardware%20plan.md).

## Panels
- 2× 1440×1600 VS035ZSM-NW0-69P0 MIPI DSI panels. These are the same panels used across the North Star revisions.
- Optics from Project North Star. Revision not yet confirmed: [Combine Reality Deck X](https://docs.projectnorthstar.org/project-north-star/mechanical/combine-reality-deck-x) or [Northstar Next](https://docs.projectnorthstar.org/project-north-star/northstar-next/introduction).

## Drive path
- North Star drives the panels through a DisplayPort-to-MIPI bridge. We drive them over direct DSI.
- On the RB5, direct DSI means using both DSI ports, one per panel.
- The existing panel driver from the RPi 5 build (RP1 support) will be ported, or replaced with an existing alternative: https://github.com/uwrealitylabs/rpi-panel-driver
- Required: device-tree or panel-driver bring-up, plus DRM/KMS direct mode for Monado's compositor.

## Software boundary
The compositor drives the panels. The [Monado headset driver](Monado%20headset%20driver.md) only describes them (resolution, refresh, distortion).

## What selects the display
| Layer | Code | Decides |
|---|---|---|
| 1. Kernel | Device tree + panel driver + msm DRM | Which connectors exist (e.g. DSI-1, DSI-2) and their modes |
| 2. Vulkan driver | Turnip (Mesa shared display code) | Exposes KMS connectors as Vulkan displays drivable without a desktop |
| 3. Monado compositor target | Output targets in `src/xrt/compositor/main/` | Which display or window the compositor opens |
| 4. HMD device | Display description in `xrt_device` | Resolution, viewports, refresh the compositor renders for |

Monado targets: direct mode for PCs (X11 RandR, NVIDIA, Wayland DRM leasing), windowed (X11, Wayland), and `VK_KHR_display` picking a display by index. Selection follows platform defaults plus env vars/settings; recent Monado reorganized targets into factories, so read the current tree.

With no desktop, the `VK_KHR_display` target is the fit. Confirm early, since both can fail independently of our code:
- Turnip on mainline exposes the RB5's DSI outputs as Vulkan displays.
- Monado's Vulkan-display target works on that stack. Its display selection logic was designed for PC headsets attached to desktops.

The dev kit's HDMI output is also a KMS connector, so the target can be proven on a monitor first ([Tracer bullet roadmap](Tracer%20bullet%20roadmap.md) Stage 1).

**Consistency risk.** Nothing checks that layer 4 matches layers 1–3. The display bring-up contract should state the exact kernel mode, the builder should load a matching description, and the driver should verify at startup and fail loudly.

## Open questions
- One logical dual-DSI display vs two independent displays. Monado's compositor presents to one surface with both eyes on it (verify in current code). One wide logical display fits as-is, each eye a half viewport; two displays likely need compositor changes, which are expensive to carry in a fork. Settle with the hardware lead before panel driver work goes far.
- Refresh rate and persistence settings.
- Distortion calibration method for the North Star combiner optics.
