---
title: Add Media Streamer
parent: Add your Custom Connector
nav_order: 4
---

# Add Media Streamer

A **Media Streamer** sends a live video stream from the robot to the operator's browser via a WebRTC MediaTrack. Internally, frames are passed as `cv::Mat` objects and encoded by the WebRTC layer.

---

## Class Hierarchy

```
IMediaTrackHandler        (interface)
└── AMediaTrackHandler    (abstract base — use this)
    └── YourMediaTrack    (your implementation)
```

`AMediaTrackHandler` also inherits from `Recordable`, which provides optional video-to-file recording.

---

## Step 1: Implement the Media Track

Inherit from `AMediaTrackHandler` and implement `Start()` and `Stop()`. Call `SendFrame()` whenever a new frame is ready.

```cpp
#include <mosaic/handlers/media_track/media_track_handler.hpp>
#include <opencv2/opencv.hpp>
#include <thread>
#include <chrono>

class MyVideoTrack : public mosaic::handlers::AMediaTrackHandler {
public:
    explicit MyVideoTrack(const std::string& track_name)
        : AMediaTrackHandler(track_name, /* recordable= */ false) {}

    ~MyVideoTrack() override {
        MyVideoTrack::Stop();
    }

    void Start() override {
        if (IsRunning()) return;

        SetRunning(true);
        SetStopFlag(false);
        start_time_ = std::chrono::steady_clock::now();

        thread_ = std::make_unique<std::thread>([this]() {
            while (!GetStopFlag()) {
                cv::Mat frame = CaptureFrame();   // your capture logic
                if (!frame.empty()) {
                    SendFrame(frame, start_time_);
                }
                std::this_thread::sleep_for(std::chrono::milliseconds(33)); // ~30 fps
            }
        });
    }

    void Stop() override {
        if (!IsRunning()) return;

        SetStopFlag(true);
        if (thread_ && thread_->joinable()) {
            thread_->join();
        }
        thread_.reset();
        SetRunning(false);
    }

private:
    cv::Mat CaptureFrame() {
        // Return the latest frame as a cv::Mat (BGR format)
        // e.g. from a camera, sensor, or simulation renderer
        return cv::Mat();
    }

    std::chrono::steady_clock::time_point start_time_;
    std::unique_ptr<std::thread> thread_;
};
```

### Key Points

| Method / Field | Description |
|:---|:---|
| `Start()` | Called when a WebRTC peer connects. Start your capture thread here. |
| `Stop()` | Called on disconnect. Join your thread and release resources. |
| `SendFrame(frame, start_time)` | Encodes and sends the frame to the peer. Call this from inside the loop. |
| `IsRunning()` / `SetRunning(bool)` | Guards against double-start. |
| `GetStopFlag()` / `SetStopFlag(bool)` | Use as the loop exit condition. |
| `recordable` constructor arg | Set `true` to enable video-to-file recording via `Recordable`. |

{: .note }
> `SendFrame()` expects frames in **BGR** format (OpenCV default). If your source produces RGB or RGBA, convert with `cv::cvtColor` before calling `SendFrame()`.

---

## Step 2: Implement the Configurer

Inherit from `AMTHandlerConfigurer` to wire your track into the AutoConfigurer system.

```cpp
#include <mosaic/auto_configurer/connector/configurable_connectors.hpp>

class MyVideoConfigurer : public mosaic::auto_configurer::AMTHandlerConfigurer {
public:
    std::string GetConnectorType() const override {
        return "my-video-sender";   // must match the YAML 'type' field
    }

    void Configure() override {
        // Read params from the YAML config
        const auto fps = std::stoi(connector_config_->params.at("fps"));

        handler_ = std::make_shared<MyVideoTrack>(connector_config_->label);
    }
};
```

---

## Step 3: Register and Configure

Register the configurer before calling `AutoConfigure()`:

```cpp
#include <mosaic/auto_configurer/connector/connector_resolver.hpp>
#include <mosaic/auto_configurer/auto_configurer.hpp>

int main() {
    mosaic::auto_configurer::ConnectorResolver::GetInstance()
        .RegisterConfigurableConnector<MyVideoConfigurer>();

    mosaic::auto_configurer::AutoConfigurer auto_configurer;
    auto_configurer.AutoConfigure("mosaic_config.yaml");

    auto connector = auto_configurer.GetMosaicConnector();

    // Block until interrupted
    std::this_thread::sleep_until(std::chrono::steady_clock::time_point::max());

    connector->ShuttingDown();
}
```

---

## Step 4: YAML Configuration

```yaml
connectors:
  - type: 'my-video-sender'
    label: 'camera'
    params:
      fps: 30
```

The `label` becomes the WebRTC track name visible on the dashboard. `params` are available inside `Configure()` via `connector_config_->params`.

---

## Built-in Example: OpenCV Camera

mosaic-core ships `opencv-sender-camera`, which is a complete reference implementation of this pattern.

```yaml
connectors:
  - type: 'opencv-sender-camera'
    label: 'camera_stream'
    params:
      fps: 30
      camera_id: 0
      width: 640
      height: 480
```

It opens a V4L2 device with `cv::VideoCapture`, reads frames in a loop, and calls `SendFrame()` — exactly the same structure as the example above.

---

## ROS2: Using `ROS2AMTHandlerConfigurer`

For ROS2 projects, inherit from `ROS2AMTHandlerConfigurer` instead. This gives access to the shared `MosaicNode` for creating subscriptions.

```cpp
#include <mosaic-ros2-base/configurer/ros2_a_mt_handler_configurer.h>

class MyROS2VideoConfigurer : public mosaic::ros2::ROS2AMTHandlerConfigurer {
public:
    std::string GetConnectorType() const override {
        return "my-ros2-video-sender";
    }

    void Configure() override {
        handler_ = std::make_shared<MyVideoTrack>(connector_config_.label, mosaic_node_);

        const auto sub = mosaic_node_->create_subscription<sensor_msgs::msg::Image>(
            connector_config_.params.at("topic_name"),
            10,
            [this](sensor_msgs::msg::Image::SharedPtr msg) {
                // convert and forward to MyVideoTrack
            });

        mosaic_node_->AddSubscription(sub);
    }
};
```

The built-in `ros2-sender-sensor-Image` connector follows this exact pattern — see `mosaic-ros2-sensor` for the full implementation.