---
title: What is MOSAIC?
nav_order: 2
---

# What is MOSAIC?

MOSAIC is a **WebRTC-based real-time robot control and monitoring system** designed for low-latency, bidirectional communication between robots and operators.

It provides a complete infrastructure — from the robot-side C++ library to the web-based operator dashboard — so you can focus on your robot's application logic rather than building communication pipelines from scratch.

---

## Why WebRTC?

Traditional robot communication stacks (ROS topics over LAN, custom TCP/UDP sockets, etc.) struggle when robots operate over the public internet. Firewalls, NATs, and network instability make reliable low-latency communication difficult.

WebRTC was originally designed for real-time browser video calls. MOSAIC repurposes it for robotics:

- **Peer-to-peer connections** bypass servers after initial handshake, minimizing latency
- **NAT/firewall traversal** via ICE/STUN/TURN works across the public internet
- **DataChannels** carry arbitrary binary or text data (sensor readings, commands, etc.)
- **MediaStreams** carry live video with hardware-accelerated encoding/decoding

---

## System Overview

```
┌──────────────────────────────────────────────────────┐
│                  Operator Browser                    │
│         (MOSAIC-Server Dashboard / React)            │
└──────┬──────────────────────────────┬────────────────┘
       │  P2P (WebRTC)                │  P2P (WebRTC)
       │                              │
┌──────▼──────┐                ┌──────▼──────┐
│   Robot A   │                │   Robot B   │
│ mosaic-core │                │ mosaic-core │
└──────┬──────┘                └──────┬──────┘
       │  WebSocket (signaling)       │  WebSocket (signaling)
       └──────────────┬───────────────┘
┌─────────────────────▼────────────────────────────────┐
│                  MOSAIC-Server                       │
│         (Spring WebFlux + PostgreSQL)                │
└──────────────────────────────────────────────────────┘
```

The signaling server (MOSAIC-Server) only brokers the initial WebRTC handshake. Once connected, video and data flow **directly** between each robot and the operator over individual P2P connections.

---

## Components

| Component | Language | Role |
|:---|:---|:---|
| **mosaic-core** | C++ / Python | WebRTC client library for robots |
| **MOSAIC-Server** | Spring WebFlux + React | Signaling server + operator dashboard |
| **mosaic-ros2** | C++ | ROS2 topic ↔ MOSAIC connector packages |
| **mosaic-isaac-sim-example** | Python | Isaac Lab simulation example |
| **mosaic-turtlebot3-wo-ros-example** | C++ | TurtleBot3 example without ROS |

---

## Key Features

**Real-time Communication**
- Peer-to-peer WebRTC connections with sub-100ms latency
- Automatic reconnection and connection state management

**Multi-Robot Support**
- Control and monitor multiple heterogeneous robots simultaneously from a single dashboard

**Live Video Streaming**
- Camera feeds via WebRTC MediaStream with adaptive quality
- Multiple streams per robot

**Sensor Data & Commands**
- Bidirectional data via WebRTC DataChannels
- Send velocity commands, receive LiDAR scans, GPS, IMU, and more

**Flexible Integration**
- Works with ROS2, Isaac Lab, or bare-metal C++/Python
- YAML-based configuration — no code changes needed to add new data streams

---

## Who Is It For?

MOSAIC is aimed at robotics researchers and engineers who want to:

- Monitor and control robots operating **outside the lab** over the public internet
- Stream **live video** from robots without a dedicated streaming server
- Build a **custom operator dashboard** without implementing WebRTC from scratch
- Integrate with **existing ROS2** systems with minimal boilerplate