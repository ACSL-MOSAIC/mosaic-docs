---
title: Installation with Docker
parent: Installation
nav_order: 3
---

# Installation with Docker

Docker is the recommended way to run mosaic-core on a robot without manually building WebRTC or installing system dependencies.

---

## How It Works

The Dockerfile clones the mosaic-core repository, extracts the pre-built WebRTC tarball, and builds the library — all inside the container. Your application binary is then built on top of it. The final image ships everything needed to run on the robot.

---

## Writing a Dockerfile

Below is the structure used by the TurtleBot3 example. Use it as a template for your own robot application:

```dockerfile
FROM ubuntu:24.04

ENV DEBIAN_FRONTEND=noninteractive

# Install build dependencies
RUN apt-get update && apt-get install -y \
    build-essential cmake git \
    libboost-all-dev \
    libjsoncpp-dev libssl-dev libopencv-dev \
    libcpprest-dev libfmt-dev uuid-dev

# Build yaml-cpp
ARG YAML_CPP_VERSION=0.9.0
RUN git clone --depth 1 --branch yaml-cpp-${YAML_CPP_VERSION} \
    https://github.com/jbeder/yaml-cpp.git /tmp/yaml-cpp && \
    cd /tmp/yaml-cpp && mkdir build && cd build && \
    cmake -DYAML_BUILD_SHARED_LIBS=ON .. && \
    make -j$(nproc) && make install && \
    rm -rf /tmp/yaml-cpp

# Clone and build mosaic-core
ARG MOSAIC_CORE_VERSION=v1.1.0
RUN git clone --depth 1 --branch ${MOSAIC_CORE_VERSION} \
    https://github.com/ACSL-MOSAIC/mosaic-core.git /workspace/mosaic-core

WORKDIR /workspace/mosaic-core/third_party
RUN tar -xzf webrtc-tarballs/webrtc-6723-arm64.tar.gz   # or webrtc-6723-x86.tar.gz

WORKDIR /workspace/mosaic-core
RUN mkdir -p build && cd build && \
    cmake -DBUILD_PYTHON_BINDINGS=OFF -DBUILD_TESTS=OFF \
          -DCMAKE_BUILD_TYPE=Release .. && \
    make install

# Copy and build your application
WORKDIR /workspace/project
COPY CMakeLists.txt .
COPY src ./src
RUN mkdir -p build && cd build && cmake .. && make

CMD ["build/your_robot_app"]
```

{: .note }
> Change `webrtc-6723-arm64.tar.gz` to `webrtc-6723-x86.tar.gz` for x86_64 targets.

---

## Build the Image

**x86_64:**

```bash
cd docker
docker compose build
```

**arm64 (Raspberry Pi / Jetson) — cross-compilation from x86_64 host:**

```bash
cd docker
bash build-arm64.sh
```

The arm64 script uses `docker buildx` with QEMU emulation. The resulting image can be saved and transferred to the target device:

```bash
docker save your-image:arm64 | gzip > robot-image.tar.gz

# On the target device (Raspberry Pi, Jetson, etc.):
docker load < robot-image.tar.gz
```

---

## docker-compose.yml

Mount your config file and expose the hardware device:

```yaml
services:
  robot-app:
    image: your-robot-image:latest
    container_name: robot-controller
    volumes:
      - ./mosaic_config.yaml:/app/mosaic_config.yaml
    devices:
      - /dev/ttyACM0:/dev/ttyACM0   # adjust for your hardware
    privileged: true
    restart: unless-stopped
```

---

## Run

```bash
cd docker
docker compose up
```