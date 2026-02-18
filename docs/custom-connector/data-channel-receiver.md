---
title: Add Data Channel Receiver
parent: Add your Custom Connector
nav_order: 3
---

# Add Data Channel Receiver

A **Data Channel Receiver** accepts commands or data sent from the operator's dashboard to the robot over a WebRTC DataChannel. Typical uses include velocity commands, mode switches, or any control input.

---

## Class Hierarchy

```
IDataChannelHandler                      (interface)
└── ADataChannelHandler                  (abstract base)
    └── IDataChannelReceivable           (receives raw DataBuffer)
        └── DataChannelReceivable<T>     (template — deserialize then handle)
            ├── DataChannelStringReceivable   (T = std::string)
            ├── DataChannelJsonReceivable     (T = Json::Value)
            └── DataChannelByteReceivable     (T = std::vector<uint8_t>)
```

Pick the base class that matches the format your dashboard sends:

| Base class | Input format | When to use |
|:---|:---|:---|
| `DataChannelStringReceivable` | Plain UTF-8 string | Simple text commands |
| `DataChannelJsonReceivable` | JSON object | Structured control data |
| `DataChannelByteReceivable` | Raw binary | Protobuf or custom binary protocol |
| `DataChannelReceivable<T>` | Any type | Custom deserialization |

---

## Step 1: Implement the Receiver

Override `HandleData()` with your logic. Deserialization from the raw `DataBuffer` is handled by the base class automatically.

### JSON Example (most common)

```cpp
#include <mosaic/handlers/data_channel/data_channel_receivable.hpp>
#include <json/json.h>

class MyCommandReceiver : public mosaic::handlers::DataChannelJsonReceivable {
public:
    explicit MyCommandReceiver(const std::string& label)
        : DataChannelJsonReceivable(label) {}

    void HandleData(const Json::Value& data) override {
        if (!data.isMember("linear_x") || !data.isMember("angular_z")) {
            MOSAIC_LOG_ERROR("Invalid command: missing fields.");
            return;
        }

        const float linear_x  = data["linear_x"].asFloat();
        const float angular_z = data["angular_z"].asFloat();

        // Apply the command to your robot
        robot_.SetVelocity(linear_x, angular_z);
    }

private:
    MyRobot& robot_;
};
```

### String Example

```cpp
class MyModeReceiver : public mosaic::handlers::DataChannelStringReceivable {
public:
    explicit MyModeReceiver(const std::string& label)
        : DataChannelStringReceivable(label) {}

    void HandleData(const std::string& data) override {
        if (data == "autonomous") {
            robot_.SetMode(Mode::Autonomous);
        } else if (data == "manual") {
            robot_.SetMode(Mode::Manual);
        }
    }
};
```

### Custom Type Example

Use `DataChannelReceivable<T>` when you need to deserialize from a binary format (e.g. Protobuf):

```cpp
struct MyCommand { float linear_x; float angular_z; };

class MyBinaryReceiver : public mosaic::handlers::DataChannelReceivable<MyCommand> {
public:
    explicit MyBinaryReceiver(const std::string& label)
        : DataChannelReceivable(label) {}

protected:
    MyCommand ConvertBufferToData(const webrtc::DataBuffer& buffer) override {
        // Deserialize from the raw buffer (e.g. Protobuf, custom struct)
        MyCommand cmd;
        std::memcpy(&cmd, buffer.data.data<uint8_t>(), sizeof(MyCommand));
        return cmd;
    }

    void HandleData(const MyCommand& cmd) override {
        robot_.SetVelocity(cmd.linear_x, cmd.angular_z);
    }
};
```

---

## Step 2: Implement the Configurer

Inherit from `ADCHandlerConfigurer`:

```cpp
#include <mosaic/auto_configurer/connector/configurable_connectors.hpp>

class MyCommandConfigurer : public mosaic::auto_configurer::ADCHandlerConfigurer {
public:
    std::string GetConnectorType() const override {
        return "my-command-receiver";   // must match the YAML 'type' field
    }

    void Configure() override {
        handler_ = std::make_shared<MyCommandReceiver>(connector_config_->label);
    }
};
```

---

## Step 3: Register and Configure

```cpp
#include <mosaic/auto_configurer/connector/connector_resolver.hpp>
#include <mosaic/auto_configurer/auto_configurer.hpp>

int main() {
    mosaic::auto_configurer::ConnectorResolver::GetInstance()
        .RegisterConfigurableConnector<MyCommandConfigurer>();

    mosaic::auto_configurer::AutoConfigurer auto_configurer;
    auto_configurer.AutoConfigure("mosaic_config.yaml");

    auto connector = auto_configurer.GetMosaicConnector();

    // HandleData() is called automatically on every incoming message.
    // No polling required in your main loop.

    std::this_thread::sleep_until(std::chrono::steady_clock::time_point::max());
    connector->ShuttingDown();
}
```

{: .note }
> `HandleData()` is invoked on the WebRTC network thread. If your handler accesses shared state, protect it with a mutex or use an atomic flag (e.g. `std::atomic<bool>`) to hand off the data to your main loop safely.

---

## Step 4: YAML Configuration

```yaml
connectors:
  - type: 'my-command-receiver'
    label: 'cmd_vel'
```

---

## Built-in Examples

**TurtleBot3 teleop receiver** (`mosaic-turtlebot3-wo-ros-example`)

Receives `{ "linear_x": 0.5, "angular_z": -0.3 }` from the dashboard and writes the values to a shared context struct that the main loop reads:

```cpp
void TeleopReceiver::HandleData(const Json::Value& data) {
    ctx_->teleop->linear_x = data["linear_x"].asFloat();
    ctx_->teleop->angular_z = data["angular_z"].asFloat();
    ctx_->teleop_changed->store(true);
}
```

**ROS2 Twist receiver** (`mosaic-ros2-geometry`)

Receives a direction string (`"up"`, `"down"`, `"left"`, `"right"`, `"stop"`) and publishes the corresponding `geometry_msgs/Twist` to a ROS2 topic:

```cpp
void TwistDataChannel::HandleData(const TwistMessage& data) {
    if      (data.direction == "up")    SendTwist( 2.0,  0.0);
    else if (data.direction == "down")  SendTwist(-2.0,  0.0);
    else if (data.direction == "left")  SendTwist( 2.0,  2.0);
    else if (data.direction == "right") SendTwist( 2.0, -2.0);
    else if (data.direction == "stop")  SendTwist( 0.0,  0.0);
}
```

---

## ROS2: Using `ROS2ADCHandlerConfigurer`

For ROS2 projects, inherit from `ROS2ADCHandlerConfigurer`. The configurer gets access to `mosaic_node_` so you can create publishers inside `Configure()`:

```cpp
#include <mosaic-ros2-base/configurer/ros2_a_dc_handler_configurer.h>

class MyROS2CommandConfigurer : public mosaic::ros2::ROS2ADCHandlerConfigurer {
public:
    std::string GetConnectorType() const override {
        return "my-ros2-command-receiver";
    }

    void Configure() override {
        auto publisher = mosaic_node_->create_publisher<geometry_msgs::msg::Twist>(
            connector_config_.params.at("topic_name"), 10);

        mosaic_node_->AddPublisher(publisher);
        handler_ = std::make_shared<MyCommandReceiver>(connector_config_.label, publisher);
    }
};
```