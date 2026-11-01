---
author: Nathaniel Reilly <nathanmreilly@gmail.com>, Claude Opus 5.5
---
This guide is for those who are looking to work on our monado driver and test on a Qualcomm RB5/EB5. The guide assumes you have access to an interactive shell on this device, and that the device is currently running the stock OS image.

# Docker Container & Initial Build

Create a `Dockerfile` with the following contents in a build directory, e.g. `~/docker/monado-dev` . Since the stock image (Ubuntu 18, Linux 4.19) is outdated for our purposes, this defines a docker image using Ubuntu 24.04 which supports the packages we need.

```dockerfile
FROM ubuntu:24.04
ENV DEBIAN_FRONTEND=noninteractive
ARG UID=1000
ARG GID=1000

RUN apt-get update && apt-get install -y --no-install-recommends \
      build-essential git wget unzip cmake ninja-build libeigen3-dev curl patch \
      python3 pkg-config libx11-dev libx11-xcb-dev libxxf86vm-dev libxrandr-dev \
      libxcb-randr0-dev libvulkan-dev glslang-tools libglvnd-dev libgl1-mesa-dev \
      ca-certificates libusb-1.0-0-dev libudev-dev libhidapi-dev libwayland-dev \
      libuvc-dev libavcodec-dev libopencv-dev libv4l-dev libcjson-dev libsdl2-dev \
      libegl1-mesa-dev libbsd-dev \
      vulkan-tools mesa-vulkan-drivers \
    && rm -rf /var/lib/apt/lists/*

RUN (getent group $GID || groupadd -g $GID dev) \
 && (id -u $UID >/dev/null 2>&1 || useradd -m -u $UID -g $GID dev)
USER $UID:$GID
WORKDIR /src
```

Build the docker image:

```sh
cd ~/docker/monado-dev
docker build -t monado-dev --build-arg UID=$(id -u) --build-arg GID=$(id -g) .
```

Clone our Monado fork (`dev` branch) into `~/src/monado`. For upstream Monado, use `https://gitlab.freedesktop.org/monado/monado.git` instead; keep the target directory name `monado` so the paths below still match:

```sh
mkdir -p ~/src
cd ~/src && git clone -b dev https://github.com/uwrealitylabs/standalone-monado-driver.git monado
```

Build monado from source by running a temporary container:

```sh
docker run --rm -v ~/src:/src -w /src/monado monado-dev bash -c "cmake -G Ninja -B build -DCMAKE_BUILD_TYPE=RelWithDebInfo -DXRT_MODULE_COMPOSITOR_NULL=ON && ninja -C build"
```

_Notice we set `XRT_MODULE_COMPOSITOR_NULL=ON` to build the null compositor module, which is a headless compositor that we can use for testing without a connected display (e.g. through ssh). Select it at runtime with `XRT_COMPOSITOR_NULL=1`._

_This command mounts `~/src` at `/src` and builds in `/src/monado`, the same paths as the development container below. CMake records the source path when it configures, so a build directory configured under a different mount fails to rebuild; if that happens, delete `~/src/monado/build` and configure again._

Now, we can get started with development. Run the docker container with privileged settings in an interactive shell:

```sh
docker run -it --rm --privileged --network host -v /dev:/dev -v /run/udev:/run/udev:ro -v $XDG_RUNTIME_DIR:/run/user/host -e XDG_RUNTIME_DIR=/run/user/host -e WAYLAND_DISPLAY=$WAYLAND_DISPLAY -e XR_RUNTIME_JSON=/src/monado/build/openxr_monado-dev.json -v ~/src:/src -w /src/monado monado-dev bash
```

# Vulkan driver

The image installs Ubuntu's `mesa-vulkan-drivers`. That package may not include Turnip's backend for the KGSL kernel driver the stock image uses, in which case Vulkan falls back to Mesa's software renderer (lavapipe). Check inside the container:

```sh
vulkaninfo --summary
```

If the only device is `PHYSICAL_DEVICE_TYPE_CPU` (llvmpipe), Vulkan is running in software. That is fine for headless work with the null compositor. For GPU work, use the self-built Mesa Turnip that works in containers on the Adreno 650 (see [OS and kernel](../../platform/OS%20and%20kernel.md)): mount its install directory into the container and point `VK_ICD_FILENAMES` at its ICD JSON (e.g. `freedreno_icd.aarch64.json`).
