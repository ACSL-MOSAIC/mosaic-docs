---
title: Add Data Channel Sender
parent: Add your Custom Connector
nav_order: 2
---

# Add Data Channel Sender

A **Data Channel Sender** pushes data from the robot to the operator over a WebRTC DataChannel. It is suited for telemetry, sensor readings, GPS coordinates, or any structured data you want to display on the dashboard.

---

## Class Hierarchy

```
IDataChannelHandler         (interface)
└── ADataChannelHandler     (abstract base)
    └── DataChannelSendable (use this)
        └── YourSender      (your implementation)
```

---

## Step 1: Implement the Sender

Inherit from `DataChannelSendable` and add a method that calls one of the built-in send helpers.

```cpp
#include <mosaic/handlers/data_channel/data_channel_sendable.hpp>
#include <json/json.h>

class MySensor;  // forward declaration of your data source

class MySensorDataChannel : public mosaic::handlers::DataChannelSendable {
public:
    explicit MySensorDataChannel(const std::string& label)
        : DataChannelSendable(label) {}

    // Call this whenever you have new data to send
    void OnDataReceived(const MySensor& sensor) {
        if (!Sendable()) return;  // drop if peer is not connected yet

        Json::Value payload;
        payload["value"] = sensor.value;
        payload["timestamp"] = sensor.timestamp_us;

        SendJson(payload);
    }
};
```

### Send Methods

`DataChannelSendable` provides four send helpers:

| Method | Data format | Binary flag |
|:---|:---|:---|
| `SendString(str)` | UTF-8 string | `false` |
| `SendStringAsByte(str)` | String as binary blob | `true` |
| `SendJson(json)` | JSON serialized to string | `false` |
| `Send(DataBuffer, isAsync)` | Raw `webrtc::DataBuffer` | — |

All methods accept an optional `isAsync` flag (default `false`). When `true`, the buffer is queued and sent via `SendAsync`, which is safer when sending large or frequent data from a background thread.

{: .note }
> Always check `Sendable()` before sending. It returns `true` only while the DataChannel is in the `kOpen` state. Calls made before the peer connects are silently dropped.

---

## Step 2: Implement the Configurer

Inherit from `ADCHandlerConfigurer` to integrate with the AutoConfigurer system.

```cpp
#include <mosaic/auto_configurer/connector/configurable_connectors.hpp>

class MySensorConfigurer : public mosaic::auto_configurer::ADCHandlerConfigurer {
public:
    std::string GetConnectorType() const override {
        return "my-sensor-sender";   // must match the YAML 'type' field
    }

    void Configure() override {
        handler_ = std::make_shared<MySensorDataChannel>(connector_config_->label);
    }
};
```

---

## Step 3: Register and Send

Register the configurer, run `AutoConfigure()`, then send data from your main loop:

```cpp
#include <mosaic/auto_configurer/connector/connector_resolver.hpp>
#include <mosaic/auto_configurer/auto_configurer.hpp>

int main() {
    mosaic::auto_configurer::ConnectorResolver::GetInstance()
        .RegisterConfigurableConnector<MySensorConfigurer>();

    mosaic::auto_configurer::AutoConfigurer auto_configurer;
    auto_configurer.AutoConfigure("mosaic_config.yaml");

    // Retrieve your handler after configuration
    auto connector = auto_configurer.GetMosaicConnector();

    // Get a typed pointer to your handler
    // (store it during Configure() for easier access)
    while (true) {
        MySensor reading = sensor.Read();
        my_data_channel->OnDataReceived(reading);
        std::this_thread::sleep_for(std::chrono::milliseconds(100));
    }

    connector->ShuttingDown();
}
```

{: .note }
> A convenient pattern is to store the `shared_ptr` to your handler inside the Configurer and expose a getter, so the main loop can retrieve it after `AutoConfigure()`.

---

## Step 4: YAML Configuration

```yaml
connectors:
  - type: 'my-sensor-sender'
    label: 'sensor_data'
    params:
      topic_name: '/my/sensor/topic'   # custom params as needed
```

The `label` becomes the DataChannel name used by the dashboard to identify the stream.

---

## Built-in Example: NavSatFix Sender

`ros2-sender-sensor-NavSatFix` in `mosaic-ros2-sensor` is a complete reference implementation of this pattern. It subscribes to a `sensor_msgs/NavSatFix` ROS2 topic and sends the GPS coordinates as JSON:

```cpp
// On each message callback:
Json::Value json;
json["latitude"]  = msg->latitude;
json["longitude"] = msg->longitude;
json["altitude"]  = msg->altitude;
json["timestamp"] = timestamp_us;
SendJson(json);
```

---

## ROS2: Using `ROS2ADCHandlerConfigurer`

For ROS2 projects, inherit from `ROS2ADCHandlerConfigurer` instead of `ADCHandlerConfigurer`. This gives access to the shared `MosaicNode` for creating topic subscriptions.

```cpp
#include <mosaic-ros2-base/configurer/ros2_a_dc_handler_configurer.h>

class MyROS2SensorConfigurer : public mosaic::ros2::ROS2ADCHandlerConfigurer {
public:
    std::string GetConnectorType() const override {
        return "my-ros2-sensor-sender";
    }

    void Configure() override {
        auto channel = std::make_shared<MySensorDataChannel>(connector_config_.label);
        handler_ = channel;

        subscription_ = mosaic_node_->create_subscription<MySensorMsg>(
            connector_config_.params.at("topic_name"),
            10,
            [channel](MySensorMsg::SharedPtr msg) {
                channel->OnDataReceived(*msg);
            });

        mosaic_node_->AddSubscription(subscription_);
    }

private:
    std::shared_ptr<rclcpp::Subscription<MySensorMsg>> subscription_;
};
```