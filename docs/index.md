---
title: Home
layout: home
nav_order: 1
permalink: /
---

# MOSAIC

**MOSAIC** is a WebRTC-based real-time robot control and monitoring system. It provides peer-to-peer communication between robots and an operator dashboard over the public internet — without a dedicated relay server.

---

## What it does

- Streams live **video** and **sensor data** from robots to a browser dashboard
- Sends **control commands** back to robots with low latency
- Traverses NATs and firewalls automatically via ICE/STUN/TURN
- Manages multiple heterogeneous robots simultaneously from a single interface

## How it fits together

| Component | Role |
|:---|:---|
| **mosaic-core** | C++ WebRTC client library running on the robot |
| **MOSAIC-Server** | Signaling server + React operator dashboard |
| **mosaic-ros2** | ROS2 connector packages for mosaic-core |

Once the initial WebRTC handshake completes through MOSAIC-Server, video and data flow **directly** peer-to-peer between each robot and the operator.

---

## Get started

- [What is MOSAIC?]({% link docs/what-is-mosaic.md %}) — architecture and design rationale
- [Integration Guide]({% link docs/integration-guide.md %}) — how to set up the server and connect your robot
- [Key Concepts]({% link docs/key-concept.md %}) — connectors, data channels, and media streams