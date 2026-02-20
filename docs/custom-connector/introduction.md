---
title: Introduction
parent: Add your Custom Connector
nav_order: 1
---

# Introduction

A **Connector** is the unit that bridges your robot's data with the MOSAIC dashboard over WebRTC. Each connector wraps a single handler — a video track, a data channel sender, or a data channel receiver — and is described by a `type` field in the YAML configuration file.

mosaic-core ships with a small set of built-in connectors (e.g. `opencv-sender-camera`, `connection-check`). When you need to send custom sensor data, receive custom commands, or stream video from a non-standard source, you implement your own connector.

---

## How It All Fits Together

```
mosaic_config.yaml
       │
       ▼
  AutoConfigurer                 ← reads YAML, orchestrates everything
       │
       ├─ ConnectorResolver      ← maps YAML 'type' → Configurer factory
       │       │
       │       └─ YourConfigurer ← creates and wires your Handler
       │
       └─ MosaicConnector        ← holds all active connectors, manages WebRTC
```

---

## ConnectorResolver

`ConnectorResolver` is a **singleton** registry that maps each connector `type` string to a factory that produces the corresponding configurer.

### Built-in Registrations

The resolver pre-registers the following types at startup:

| Type string | Description |
|:---|:---|
| `opencv-sender-camera` | Captures frames from a V4L2 camera via OpenCV and streams them |
| `connection-check` | Sends periodic heartbeat data to verify the connection is alive |

### Registering a Custom Connector

Call `RegisterConfigurableConnector<T>()` on the singleton before `AutoConfigure()` runs:

```cpp
#include <mosaic/auto_configurer/connector/connector_resolver.hpp>

mosaic::auto_configurer::ConnectorResolver::GetInstance()
    .RegisterConfigurableConnector<MyCommandConfigurer>();
```

Internally this stores a `ConfigurableConnectorFactory<T>` for `MyCommandConfigurer`. When the YAML `type` field matches `GetConnectorType()` on your configurer, the factory instantiates and runs it.

---

## AutoConfigurer

`AutoConfigurer` reads the YAML file and drives the full setup sequence.

### AutoConfigure() Flow

```
AutoConfigure("mosaic_config.yaml")
    │
    ├─ 1. ReadConfigs()            — parse YAML into ConnectorConfig objects
    ├─ 2. CreateMosaicConnector()  — instantiate the MosaicConnector (ICE, TURN, etc.)
    ├─ 3. ResolveConnectors()      — look up each 'type' in ConnectorResolver
    ├─ 4. BeforeConfigure()        — pre-configuration hook (e.g. ROS2 node init)
    ├─ 5. ConfigureConnectors()    — call Configure() on each resolved configurer
    └─ 6. AfterConfigure()         — post-configuration hook (e.g. register publishers)
```

`Configure()` on your configurer runs at step 5. By that point `connector_config_` (the parsed YAML block for your connector) is already set, so you can safely read `connector_config_->label` and `connector_config_->params`.

### YAML Structure

```yaml
server:
  ws_url: 'wss://api-mosaic.your-domain.com'
  auth:
    type: 'simple-token'
    robot_id: '<robot-uuid>'
    params:
      token: '<encrypted-token>'
  webrtc:
    ice_servers:
      - urls:
          - 'turn:your-turn-server:3478'
        username: 'username'
        credential: 'credential'

connectors:
  - type: 'my-sensor-sender'      # matched against GetConnectorType()
    label: 'sensor_data'          # becomes the DataChannel or MediaTrack name
    params:
      topic_name: '/my/sensor'    # arbitrary key-value pairs for your configurer
```

---

## Configurer Types

Choose the base class that matches what your connector does:

| Base class | Connector kind | Detailed guide |
|:---|:---|:---|
| `ADCHandlerConfigurer` | Single DataChannel (send **or** receive) | [Data Channel Sender](./data-channel-sender) · [Data Channel Receiver](./data-channel-receiver) |
| `AMultipleDCHandlerConfigurer` | Multiple DataChannels in one configurer | — |
| `AMTHandlerConfigurer` | Video / audio MediaTrack | [Media Streamer](./media-streamer) |

For ROS2 projects, use the `ROS2A*Configurer` variants (`ROS2ADCHandlerConfigurer`, `ROS2AMTHandlerConfigurer`) which expose the shared `MosaicNode` for creating publishers and subscriptions.

---

## Minimal End-to-End Example

Below is the shortest possible custom connector — a JSON data channel sender:

**1. The Handler**

```cpp
#include <mosaic/handlers/data_channel/data_channel_sendable.hpp>

class MySensorChannel : public mosaic::handlers::DataChannelSendable {
public:
    explicit MySensorChannel(const std::string& label)
        : DataChannelSendable(label) {}

    void Send(float value) {
        if (!Sendable()) return;
        Json::Value payload;
        payload["value"] = value;
        SendJson(payload);
    }
};
```

**2. The Configurer**

```cpp
#include <mosaic/auto_configurer/connector/configurable_connectors.hpp>

class MySensorConfigurer : public mosaic::auto_configurer::ADCHandlerConfigurer {
public:
    std::string GetConnectorType() const override { return "my-sensor"; }

    void Configure() override {
        channel_ = std::make_shared<MySensorChannel>(connector_config_->label);
        handler_ = channel_;
    }

    std::shared_ptr<MySensorChannel> channel_;
};
```

**3. main()**

```cpp
#include <mosaic/auto_configurer/connector/connector_resolver.hpp>
#include <mosaic/auto_configurer/auto_configurer.hpp>

int main() {
    auto& resolver = mosaic::auto_configurer::ConnectorResolver::GetInstance();
    resolver.RegisterConfigurableConnector<MySensorConfigurer>();

    mosaic::auto_configurer::AutoConfigurer auto_configurer;
    auto_configurer.AutoConfigure("mosaic_config.yaml");

    auto connector = auto_configurer.GetMosaicConnector();

    while (true) {
        float reading = sensor.Read();
        my_channel->Send(reading);
        std::this_thread::sleep_for(std::chrono::milliseconds(100));
    }

    connector->ShuttingDown();
}
```

**4. YAML**

```yaml
connectors:
  - type: 'my-sensor'
    label: 'sensor_data'
```

The following pages walk through each connector type in detail.