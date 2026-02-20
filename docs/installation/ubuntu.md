---
title: Installation on Ubuntu
parent: Installation
nav_order: 1
---

# Installation on Ubuntu

This guide covers building and installing **mosaic-core** — the robot-side C++ library — on Ubuntu.

{: .note }
> mosaic-core has been tested on **Ubuntu 24.04 (x86_64 / arm64)**.

---

## Prerequisites

### System Dependencies

Install the required packages:

```bash
sudo apt-get update
sudo apt-get install -y \
  git \
  cmake \
  build-essential \
  libssl-dev \
  libopencv-dev \
  libjsoncpp-dev \
  libfmt-dev \
  libyaml-cpp-dev \
  libcpprest-dev
```

### CMake 3.14+

Verify your CMake version:

```bash
cmake --version
```

If the version is below 3.14, install a newer version from [cmake.org](https://cmake.org/download/).

---

## 1. Clone the Repository

```bash
git clone https://github.com/ACSL-MOSAIC/mosaic-core.git
cd mosaic-core
```

---

## 2. Set Up WebRTC

mosaic-core depends on Google's WebRTC native library (`libwebrtc.a`). You can either use the **pre-built tarball** (recommended) or **build from source**.

### Option A: Use Pre-built Tarball (Recommended)

Pre-built tarballs for WebRTC branch `6723` are included in the repository.

**x86_64:**
```bash
cd third_party
tar -xzf webrtc-tarballs/webrtc-6723-x86.tar.gz
cd ..
```

**arm64:**
```bash
cd third_party
tar -xzf webrtc-tarballs/webrtc-6723-arm64.tar.gz
cd ..
```

After extraction, verify the library exists:

```bash
ls third_party/webrtc/src/out/Default/obj/libwebrtc.a
```

---

### Option B: Build WebRTC from Source

{: .warning }
> Building WebRTC from source takes **1–2 hours** and requires ~20 GB of disk space.

**Step 1. Fetch the WebRTC source**

```bash
cd third_party
bash webrtc-scripts/fetch_webrtc_source.sh
cd ..
```

**Step 2. Build WebRTC**

Run the build script for your architecture:

```bash
# x86_64
bash third_party/webrtc-scripts/build_webrtc_x86_64.sh

# arm64
bash third_party/webrtc-scripts/build_webrtc_arm64.sh
```

---

## 3. Build mosaic-core

```bash
mkdir build && cd build
cmake .. -DCMAKE_BUILD_TYPE=Release
make -j$(nproc)
```

### CMake Options

| Option | Default | Description |
|:---|:---|:---|
| `BUILD_PYTHON_BINDINGS` | `ON` | Build Python bindings |
| `BUILD_TESTS` | `ON` | Build unit tests |
| `BUILD_SHARED_LIBS` | `ON` | Build as shared library |

Example — disable Python bindings and tests:

```bash
cmake .. -DCMAKE_BUILD_TYPE=Release \
         -DBUILD_PYTHON_BINDINGS=OFF \
         -DBUILD_TESTS=OFF
```

---

## 4. Install

```bash
sudo make install
```

This installs:
- Headers → `/usr/local/include/mosaic/`
- Library → `/usr/local/lib/`
- CMake config → `/usr/local/lib/cmake/mosaic-core/`

You can then find the library from another CMake project with:

```cmake
find_package(mosaic-core REQUIRED)
target_link_libraries(your_target mosaic::mosaic-core)
```

---

## 5. Python Bindings (Optional)

If `BUILD_PYTHON_BINDINGS=ON` (default), the Python package is built alongside the C++ library.

Install it in editable mode from the build output:

```bash
pip install -e build/python
```

Verify the installation:

```python
import mosaic_core
print(mosaic_core.__version__)
```

---

## Troubleshooting

**`libwebrtc.a` not found**

Make sure the WebRTC tarball was extracted correctly:
```bash
ls third_party/webrtc/src/out/Default/obj/libwebrtc.a
```

**`find_package` fails for a dependency**

Install the missing package via `apt-get`. For example, if `cpprestsdk` is not found:
```bash
sudo apt-get install -y libcpprest-dev
```

**Linker errors with `dl` or `pthread`**

These are system libraries and should always be available on Ubuntu. If missing:
```bash
sudo apt-get install -y libc6-dev
```