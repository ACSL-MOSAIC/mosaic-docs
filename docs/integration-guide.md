---
title: Integration Guide
nav_order: 3
---

# Integration Guide

This page walks you through everything needed to get MOSAIC running end-to-end — from standing up the server to streaming data from your robot.

There are two sides to integrate:

| Side | What you need |
|:---|:---|
| **Server** | A running MOSAIC-Server (signaling + dashboard) |
| **Robot** | mosaic-core installed, and a `mosaic_config.yaml` configured |

---

## Step 1 — Set Up the Server

You have two options:

### Option A: Use the hosted server (recommended to get started)

A MOSAIC-Server is already running at **[https://mosaic.acslgcs.com](https://dev-mosaic.acslgcs.com)**.
You can sign up and use it directly — no server setup required.

### Option B: Self-host your own server

If you need a private deployment, follow the [Custom Deployment]({% link docs/custom-deployment.md %}) guide to run MOSAIC-Server on your own infrastructure with Docker Compose.

---

## Step 2 — Register Your Robot on the Dashboard

1. Log in to the dashboard (hosted or your own instance)
2. Navigate to **Robots → Add Robot** and give it a name
3. The server will generate:
   - **Robot UUID** — used as `robot_id` in `mosaic_config.yaml`
   - **Auth Token** — used as `params.token` in `mosaic_config.yaml`
4. Navigate to **Settings → ICE Servers** and add your TURN server credentials (required for NAT traversal)

Keep these values ready — you'll need them in Step 4.

---

## Step 3 — Install mosaic-core on the Robot

Install the mosaic-core C++ library on your robot's machine.
Choose the guide that matches your setup:

- [Ubuntu (native build)]({% link docs/installation/ubuntu.md %}) — for direct installation on Ubuntu 22.04/24.04
- [ROS2]({% link docs/installation/ros2.md %}) — for ROS2 Humble/Jazzy environments
- [Docker]({% link docs/installation/docker.md %}) — for containerized deployments

---

## Step 4 — Configure the Robot

Create a `mosaic_config.yaml` file on the robot using the Robot UUID and Auth Token from Step 2:

```yaml
server:
  ws_url: 'wss://api-mosaic.acslgcs.com'  # or your self-hosted URL
  auth:
    type: 'simple-token'
    robot_id: '<robot-uuid>'
    params:
      token: '<auth-token>'
  webrtc:
    ice_servers:
      - urls:
          - 'turn:your-turn-server:3478'
        username: 'username'
        credential: 'credential'

connectors:
  - type: 'opencv-sender-camera'
    label: 'camera'
    params:
      camera_id: 0
      fps: 30
      width: 640
      height: 480
```

See [Config on Robot]({% link docs/config-on-robot.md %}) for the full reference.

---

## Step 5 — Run

**Without ROS2:**

Build your application with mosaic-core and run it with the config file path.

**With ROS2:**

```bash
ros2 launch mosaic-ros2-bringup mosaic_bringup_launch.py \
  mosaic_config:=/path/to/mosaic_config.yaml
```

Once running, the robot connects to MOSAIC-Server and appears as **online** in the dashboard.
Open the dashboard, select the robot, and the WebRTC peer connection is established automatically.

---

## Summary

```
1. Get a MOSAIC-Server          → use mosaic.acslgcs.com, or self-host
2. Register the robot           → get robot_id + token from the dashboard
3. Install mosaic-core          → Ubuntu / ROS2 / Docker
4. Write mosaic_config.yaml     → paste robot_id, token, ICE server info
5. Run                          → robot appears online in the dashboard
```