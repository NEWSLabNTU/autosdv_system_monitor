# AutoSDV System Monitor

A ROS2 package that provides real-time monitoring of system topics through a web interface.

## Features

- Real-time monitoring of sensor and vehicle topics
- Web-based dashboard accessible at `http://localhost:8080`
- Configurable topic monitoring through YAML configuration
- Support for various message types:
  - Sensor data (LiDAR, Camera, IMU, GPS)
  - Vehicle status (velocity, control)
  - Diagnostics
  - System logs

## Usage

### Launch the monitor
```bash
ros2 launch autosdv_system_monitor autosdv_system_monitor.launch.yaml
```

### Configuration

Topics to monitor are configured in `config/monitor_topics.yaml`. Each topic entry contains:
- Message type
- Topic name
- Display name
- Parameter name (for enabling/disabling)
- QoS reliability setting

### Parameters

- `monitor_lidar`: Enable/disable LiDAR topic monitoring (default: true)
- `monitor_camera`: Enable/disable camera topic monitoring (default: true)
- `monitor_imu`: Enable/disable IMU topic monitoring (default: true)
- `monitor_gps`: Enable/disable GPS topic monitoring (default: true)
- `monitor_vehicle`: Enable/disable vehicle topic monitoring (default: true)
- `monitor_diagnostics`: Enable/disable diagnostics monitoring (default: true)
- `report_interval_sec`: Status update interval in seconds (default: 5.0)
- `web_server_host`: Web server host address (default: localhost)
- `web_server_port`: Web server port (default: 8080)
- `topics_config_file`: Path to topics configuration file

## Web Interface

Access the monitoring dashboard at `http://localhost:8080` (or the configured host/port).

The interface displays:
- Topic message rates and frequencies
- Time since last message
- Topic status (OK, STALE, ERROR, NO DATA)
- Detailed message information for specific topic types
- System diagnostics

## Dependencies

- ROS2 Humble
- Python packages: flask, watchdog, pyyaml
- Autoware messages: autoware_vehicle_msgs, autoware_control_msgs
## Mode availability strip

Seven chips, one per mode root of Autoware's diagnostic graph, answering "can
the vehicle still go autonomous" without reading 41 diagnostic rows to work it
out. Until this existed the graph that decides mode availability had no runtime
view at all: none of the 19 `*_rviz_plugin` packages in Autoware 1.5.0
references diagnostics.

**The page talks to rosbridge directly; this node does not proxy it.** That is
deliberate. The two graph topics **disagree on QoS** — struct is RELIABLE +
TRANSIENT_LOCAL, status is BEST_EFFORT + VOLATILE — and a subscriber applying
one profile to both silently receives nothing on one of them. rosbridge derives
the profile per topic from the publishers, so the trap is unavailable rather
than merely avoided.

**struct and status are joined by array index.** `DiagNodeStatus` carries no
path field, so an off-by-one mislabels every chip while still looking entirely
plausible, and an aggregator restart rebuilds the graph with a new id. The strip
compares the id on every status and re-subscribes to struct when it changes,
which is what makes rosbridge re-deliver the latched message.

Three states that are easy to collapse into one, and must not be:

| state | shown as |
|---|---|
| unknown — struct known, no status yet | an outlined chip. A monitor started before Autoware must not render this as red |
| bridge down | chips blanked, and the note says so in words. A page that cannot reach the bridge must never look like a page reporting no faults |
| stalled — bridge up, no status for 3 s | said so. The chips would otherwise sit there looking healthy while the aggregator is dead |

`latch_level` is surfaced as an underline, so a mode that faulted earlier in the
session is still attributable after it clears. Below the strip, the failing-path
view names which leaf made which mode unavailable, and the fail-safe timeline
gives the order things happened in.

The refresh controls belong to the **sensor tables**, not to the strip: the
strip is pushed over rosbridge on every change and is unaffected by them, which
is why the bar is labelled and the strip carries a LIVE marker.

### Testing it

```bash
node test/mode_strip_render_test.js
```

Drives the shipped page JavaScript against a graph fixture captured from a live
aggregator, with a stub DOM and WebSocket. No vehicle and no ROS needed. The
fixture keeps the real node ordering on purpose: renumbering it would retire the
only test that can catch an index-join regression.

`launch_rosbridge:=false` when something else already runs a bridge on the host.
