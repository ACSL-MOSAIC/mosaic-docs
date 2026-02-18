---
title: Key Concept of MOSAIC
nav_order: 4
---

# Key Concept of MOSAIC

This page explains how the robot side and the dashboard side are connected, and how data moves from a robot connector all the way to a dashboard widget.

---

## The Big Picture

```
        ROBOT                                  DASHBOARD (Browser)
┌─────────────────────┐                 ┌──────────────────────────────┐
│                     │                 │                              │
│  mosaic-core        │                 │  MosaicProvider              │
│  ┌───────────────┐  │   WebRTC P2P    │  ┌────────────────────────┐ │
│  │  Connector    │◄─┼─────────────────┼─►│  Store                 │ │
│  │  (Handler)    │  │  DataChannel /  │  │  (ReceivableStore /    │ │
│  └───────────────┘  │  MediaTrack     │  │   SendableStore / ...)  │ │
│                     │                 │  └────────────┬───────────┘ │
│  AutoConfigurer     │                 │               │ subscribe() │
│  ConnectorResolver  │                 │  ┌────────────▼───────────┐ │
│                     │                 │  │  Widget                │ │
│                     │                 │  │  (React Component)     │ │
└─────────────────────┘                 │  └────────────────────────┘ │
                                        └──────────────────────────────┘
        │                                              │
        └──────────────────────────────────────────────┘
                  Signaling via MOSAIC-Server
                  (/ws/robot  ←→  /ws/user)
```

A robot and an operator browser establish a **direct WebRTC peer-to-peer connection**. MOSAIC-Server acts only as the signaling intermediary during connection setup; actual data never passes through the server.

---

## Robot Side: Connectors

On the robot, a **Connector** is a C++ object that either sends data to the dashboard, receives commands from it, or streams video.

| Connector type | Base class | Direction |
|:---|:---|:---|
| Data Channel Sender | `DataChannelSendable` | Robot → Dashboard |
| Data Channel Receiver | `DataChannelReceivable<T>` | Dashboard → Robot |
| Media Streamer | `AMediaTrackHandler` | Robot → Dashboard (video) |

Each connector is identified by a **label** (e.g. `"sensor_data"`, `"cmd_vel"`, `"camera"`). This label becomes the WebRTC DataChannel name or MediaTrack name, and it must match the `connectorId` configured on the dashboard side.

### How Connectors Are Set Up

The `AutoConfigurer` reads `mosaic_config.yaml` and uses the `ConnectorResolver` to map each `type` field to a registered configurer class. The configurer's `Configure()` method creates the actual handler and sets `handler_`. See the [Custom Connector Introduction](./custom-connector/introduction) page for a full walkthrough.

---

## Dashboard Side: Connector → Store → Widget

### RobotConnector

On the dashboard, a **RobotConnector** object describes one end of a WebRTC channel from the browser's perspective:

```typescript
class RobotConnector {
  robotId: string      // which robot
  connectorId: string  // must match the robot connector's label
  dataType: MosaicDataType
}
```

`MosaicDataType` encodes both the **data format** and the **direction**:

| dataType | Format | Direction |
|:---|:---|:---|
| `"json-r2u"` | JSON | Robot → Dashboard |
| `"json-u2r"` | JSON | Dashboard → Robot |
| `"json-bi"` | JSON | Bidirectional |
| `"string-r2u"` | UTF-8 string | Robot → Dashboard |
| `"string-u2r"` | UTF-8 string | Dashboard → Robot |
| `"byte-r2u"` | Binary | Robot → Dashboard |
| `"byte-u2r"` | Binary | Dashboard → Robot |
| `"media"` | Video/audio | Robot → Dashboard |

The `connectorId` of a `RobotConnector` must exactly match the `label` of the robot-side connector.

### Store

A **Store** is the state container that sits between the WebRTC layer and React components. When a DataChannel receives bytes, the WebRTC connection layer calls `store.notifySubscribers(arrayBuffer)`, delivering the data to every component that has subscribed.

There are four store base classes:

| Store type | Used when | Key API |
|:---|:---|:---|
| `ReceivableStore` | Robot → Dashboard (`*-r2u`) | `subscribe(callback)` |
| `SendableStore` | Dashboard → Robot (`*-u2r`) | `sendData(payload)` |
| `BidirectionalStore` | Both directions (`*-bi`) | `subscribe()` + `sendData()` |
| `MediaStreamStore` | Video track (`media`) | Provides `MediaStream` |

Stores are managed by `StoreManager`, which creates them on demand and tracks how many widgets are using each one (reference counting). When all widgets unmount, the store is released.

### Widget

A **Widget** is a React component that calls `useMosaicStore()` to obtain a store, then subscribes to it for live data:

```typescript
function MySensorWidget({ robotConnector }: Props) {
  const { getOrCreateStore, releaseStore } = useMosaicStore()
  const [value, setValue] = useState<number | null>(null)

  useEffect(() => {
    const store = getOrCreateStore(robotConnector) as ReceivableStore

    const unsubscribe = store.subscribe(async (arrayBuffer) => {
      const json = JSON.parse(new TextDecoder().decode(arrayBuffer))
      setValue(json.value)
    })

    return () => {
      unsubscribe()
      releaseStore(robotConnector)
    }
  }, [robotConnector])

  return <div>Sensor: {value}</div>
}
```

Multiple widgets can subscribe to the same store simultaneously. Each widget gets its own callback and they all receive the same data.

---

## End-to-End Data Flow

### Robot → Dashboard (receiving sensor data)

```
1. Robot connector calls SendJson(payload)
        │
        ▼
2. WebRTC DataChannel delivers bytes to the browser
        │
        ▼
3. WebRTCConnection.onmessage → store.notifySubscribers(arrayBuffer)
        │
        ▼
4. Every subscribed widget callback fires with the ArrayBuffer
        │
        ▼
5. Widget decodes bytes, updates React state, re-renders
```

### Dashboard → Robot (sending commands)

```
1. Widget calls store.sendData(payload)
        │
        ▼
2. SendableStore writes to RTCDataChannel.send()
        │
        ▼
3. WebRTC DataChannel delivers bytes to the robot
        │
        ▼
4. Robot connector's HandleData() is called with the deserialized value
        │
        ▼
5. Robot executes the command
```

### Video Stream

```
1. Robot's AMediaTrackHandler calls SendFrame(cv::Mat, start_time)
        │
        ▼
2. WebRTC encodes frame and delivers it as a MediaTrack
        │
        ▼
3. MediaStreamStore provides a MediaStream object
        │
        ▼
4. Widget attaches the MediaStream to an <video> element
```

---

## Label ↔ connectorId Matching

The most common integration mistake is a mismatch between the robot-side `label` and the dashboard-side `connectorId`. They must be identical:

```yaml
# mosaic_config.yaml (robot side)
connectors:
  - type: 'my-sensor-sender'
    label: 'sensor_data'      # ← this value
```

```typescript
// Dashboard widget (browser side)
const robotConnector: RobotConnector = {
  robotId: 'my-robot',
  connectorId: 'sensor_data',  // ← must match exactly
  dataType: 'json-r2u',
}
```

---

## Connection Lifecycle

Stores expose lifecycle hooks so widgets and stores can react to connection events:

| Event | When it fires |
|:---|:---|
| `onBeforeConnected` | Before the WebRTC handshake begins |
| `onAfterConnected` | DataChannel is open and ready |
| `onAfterDisconnected` | Peer disconnected |
| `onAfterConnectionFailed` | Handshake failed |

The `afterConnected()` method on a store is called automatically once the DataChannel is open. This is the right place to start sending periodic data or subscribe to self-generated events (like the built-in `ConnectionCheckingStore` does for its ping/pong heartbeat).