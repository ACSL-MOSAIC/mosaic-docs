---
title: Installation with ROS2
parent: Installation
nav_order: 2
---

# Installation with ROS2

This guide covers building and installing the **mosaic-ros2** packages, which bridge ROS2 topics to MOSAIC's WebRTC data channels and media tracks.

{: .note }
> mosaic-ros2 supports **ROS2 Humble** and **ROS2 Jazzy** on Ubuntu 22.04 / 24.04.

---

## Prerequisites

### 1. Install mosaic-core

mosaic-ros2 depends on mosaic-core. Follow the [Installation on Ubuntu](ubuntu.html) guide first to install mosaic-core.

### 2. Install ROS2

Install ROS2 Humble or Jazzy by following the [official ROS2 installation guide](https://docs.ros.org/en/humble/Installation.html). Make sure the ROS2 environment is sourced:

```bash
source /opt/ros/humble/setup.bash   # or jazzy
```

### 3. Install Additional Dependencies

```bash
sudo apt-get install -y \
  ros-${ROS_DISTRO}-cv-bridge \
  ros-${ROS_DISTRO}-sensor-msgs \
  ros-${ROS_DISTRO}-geometry-msgs \
  libprotobuf-dev \
  protobuf-compiler \
  uuid-dev
```

---

## Package Overview

mosaic-ros2 is organized as a set of ROS2 packages. Clone the repository into your colcon workspace:

```bash
cd ~/ros2_ws/src
git clone https://github.com/ACSL-MOSAIC/mosaic-ros2.git
```

The repository contains the following packages:

| Package | Role |
|:---|:---|
| `mosaic-ros2-base` | Core ROS2 integration (`ROS2AutoConfigurer`, `MosaicNode`, `RosLogger`) |
| `mosaic-ros2-geometry` | Connectors for `geometry_msgs` (Twist, TwistStamped, PoseWithCovarianceStamped) |
| `mosaic-ros2-sensor` | Connectors for `sensor_msgs` (Image, PointCloud2, NavSatFix) |
| `mosaic-ros2-bringup` | Main executable and launch files |

---

## Build

Build all packages with colcon:

```bash
cd ~/ros2_ws
colcon build --symlink-install --packages-select \
  mosaic-ros2-base \
  mosaic-ros2-geometry \
  mosaic-ros2-sensor \
  mosaic-ros2-bringup
```

Source the workspace after building:

```bash
source ~/ros2_ws/install/setup.bash
```

---

## Running

### 1. Prepare a Config File

Create a `mosaic_config.yaml` file. See [Config on Robot](../config-on-robot.html) for the full reference.

A minimal example:

```yaml
server:
  ws_url: 'wss://dev-mosaic.acslgcs.com'
  auth:
    type: 'simple-token'
    robot_id: 'your-robot-id'
    params:
      token: 'your-token'
  webrtc:
    ice_servers:
      - urls:
          - 'turn:your-turn-server:3478'
        username: 'username'
        credential: 'credential'

connectors:
  - type: 'ros2-sender-sensor-Image'
    label: 'camera'
    params:
      topic_name: '/camera/color/image_raw'

  - type: 'ros2-receiver-geometry-Twist'
    label: 'cmd_vel'
    params:
      topic_name: '/cmd_vel'
```

### 2. Launch the Node

```bash
ros2 launch mosaic-ros2-bringup mosaic_bringup_launch.py \
  mosaic_config:=/path/to/mosaic_config.yaml \
  mosaic_log_level:=info \
  webrtc_log_level:=none
```

### Launch Arguments

| Argument | Default | Options |
|:---|:---|:---|
| `mosaic_config` | `./mosaic_config.yaml` | Path to config YAML |
| `mosaic_log_level` | `info` | `debug`, `info`, `warning`, `error` |
| `webrtc_log_level` | `none` | `none`, `verbose`, `info`, `warning`, `error` |

---

## Available Connector Types

### Sender (Robot → Operator)

| Connector Type | ROS2 Message | Description |
|:---|:---|:---|
| `ros2-sender-sensor-Image` | `sensor_msgs/Image` | Camera image stream (via MediaTrack) |
| `ros2-sender-sensor-PointCloud2` | `sensor_msgs/PointCloud2` | LiDAR point cloud (Protobuf over DataChannel) |
| `ros2-sender-sensor-NavSatFix` | `sensor_msgs/NavSatFix` | GPS coordinates |
| `ros2-sender-geometry-PoseWithCovarianceStamped` | `geometry_msgs/PoseWithCovarianceStamped` | Robot pose (e.g. AMCL) |

### Receiver (Operator → Robot)

| Connector Type | ROS2 Message | Description |
|:---|:---|:---|
| `ros2-receiver-geometry-Twist` | `geometry_msgs/Twist` | Velocity command |
| `ros2-receiver-geometry-TwistStamped` | `geometry_msgs/TwistStamped` | Stamped velocity command |

---

## Troubleshooting

**`find_package(mosaic-core REQUIRED)` fails**

mosaic-core must be installed system-wide before building mosaic-ros2:
```bash
# In the mosaic-core build directory
sudo make install
```

**`protoc` not found**

```bash
sudo apt-get install -y protobuf-compiler
```

**`cv_bridge` not found**

```bash
sudo apt-get install -y ros-${ROS_DISTRO}-cv-bridge
```

**`ROS_DISTRO` not set warning during build**

Source your ROS2 environment before running colcon:
```bash
source /opt/ros/humble/setup.bash
colcon build ...
```