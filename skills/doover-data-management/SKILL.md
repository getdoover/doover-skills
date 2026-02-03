---
name: doover-data-management
description: Guide for Doover's channel-based data system, tags, and inter-agent communication
user_invocable: true
---

# Doover Data Management

This skill covers Doover's channel-based data architecture, including the core channel system and the conventions built on top of it.

## Channels: The Foundation

Doover's data system is built around **channels** - think of them as chatrooms where agents exchange data. Everything in Doover flows through channels.

### Core Concepts

| Concept | Description |
|---------|-------------|
| **Channel** | A named data stream that agents can publish to and subscribe to |
| **Agent** | Any entity that interacts with data: devices, users, organizations, applications |
| **Publish** | Send data to a channel |
| **Subscribe** | Receive data from a channel |

### How Channels Work

```
┌─────────┐     publish      ┌─────────────┐     subscribe     ┌─────────┐
│  Device │ ───────────────► │   Channel   │ ─────────────────► │  User   │
│  Agent  │                  │ sensor_data │                    │  Agent  │
└─────────┘                  └─────────────┘                    └─────────┘
                                    │
                                    │ subscribe
                                    ▼
                             ┌─────────────┐
                             │    Org      │
                             │   Agent     │
                             └─────────────┘
```

Each agent has its own set of channels. When you publish to a channel, any agent subscribed to that channel receives the data.

### Publishing to Channels

```python
import json
from datetime import datetime

class MyApplication(Application):
    async def main_loop(self):
        # Prepare data
        data = {
            "timestamp": datetime.now().isoformat(),
            "temperature": 25.5,
            "humidity": 60.0
        }

        # Publish to a channel
        await self.device_agent.publish_to_channel_async(
            "sensor_readings",
            json.dumps(data)
        )
```

### Channel Naming

Channels are identified by name. Use descriptive, hierarchical names:

```python
# Sensor data channels
await self.publish("sensors/temperature/indoor", data)
await self.publish("sensors/temperature/outdoor", data)

# Event channels
await self.publish("events/alerts", alert_data)
await self.publish("events/status_changes", status_data)

# Metrics channels
await self.publish("metrics/performance", metrics_data)
```

## Systems Built on Channels

Channels are the foundation, but Doover defines conventions (systems) that use channels in specific ways to provide higher-level functionality.

### Overview of Channel Systems

| System | Channels Used | Purpose |
|--------|---------------|---------|
| **Tags** | `tag_values` | Key-value state persistence |
| **UI** | `ui_state`, `ui_cmds` | User interface generation |
| **Future UI** | `ui_schema`, `tag_values` | Next-gen UI system |

```
┌─────────────────────────────────────────────────────────────┐
│                     Channel Systems                          │
├─────────────────┬─────────────────┬─────────────────────────┤
│   Tags System   │    UI System    │    Custom Systems       │
│  (tag_values)   │ (ui_state,      │   (your channels)       │
│                 │  ui_cmds)       │                         │
├─────────────────┴─────────────────┴─────────────────────────┤
│                    Core Channels                             │
│              (publish / subscribe)                           │
└─────────────────────────────────────────────────────────────┘
```

## Tags System

The **tags system** is a convention built on the `tag_values` channel that provides key-value state persistence.

### How Tags Work

When you call `set_tag()`, it publishes to the `tag_values` channel. When you call `get_tag()`, it reads from this channel's current state.

```python
# This publishes to the tag_values channel
await self.set_tag("temperature", 25.5)

# This reads from the tag_values channel
temp = self.get_tag("temperature")
```

### Setting Tags

```python
class MyApplication(Application):
    async def main_loop(self):
        # Simple values
        await self.set_tag("temperature", 25.5)
        await self.set_tag("is_running", True)
        await self.set_tag("status", "operational")

        # Structured data
        await self.set_tag("sensor_data", {
            "temperature": 25.5,
            "humidity": 60.0,
            "pressure": 1013.25,
            "timestamp": datetime.now().isoformat()
        })

        # Lists
        await self.set_tag("recent_values", [25.5, 26.0, 25.8])
```

### Getting Tags

```python
class MyApplication(Application):
    async def main_loop(self):
        # With default value
        temp = self.get_tag("temperature", default=0.0)
        enabled = self.get_tag("feature_enabled", default=True)

        # Without default (returns None if not set)
        status = self.get_tag("status")
        if status is None:
            status = "unknown"

        # Structured data
        sensor_data = self.get_tag("sensor_data", default={})
        if sensor_data:
            temp = sensor_data.get("temperature", 0)
```

### Tags from Other Agents

Access tags from other agents using their app key:

```python
class MyApplication(Application):
    async def main_loop(self):
        # Get another agent's app key from config
        sim_key = self.config.simulator_app_key.value

        # Read tags from that agent's tag_values channel
        sensor_value = self.get_tag("sensor_reading", app_key=sim_key)
        sensor_status = self.get_tag("status", app_key=sim_key)

        if sensor_value is not None:
            await self.process_sensor_data(sensor_value)
```

### Common Tag Patterns

#### State Tracking

```python
async def main_loop(self):
    # Track application state
    await self.set_tag("state", self.state.state)
    await self.set_tag("state_changed_at", datetime.now().isoformat())

    # Track errors
    error_count = self.get_tag("error_count", default=0)
    if self.has_error:
        await self.set_tag("error_count", error_count + 1)
        await self.set_tag("last_error", str(self.error))
```

#### Configuration Override

```python
async def main_loop(self):
    # Allow runtime override via tags
    threshold = self.get_tag("threshold_override")
    if threshold is None:
        threshold = self.config.threshold.value

    if self.sensor_value > threshold:
        await self.trigger_alert()
```

#### Request/Response

```python
async def main_loop(self):
    # Check for pending requests
    request = self.get_tag("pending_request")

    if request:
        request_type = request.get("type")
        request_id = request.get("id")

        # Process
        if request_type == "reset":
            await self.reset()
            result = "success"
        else:
            result = "unknown_request"

        # Clear request, set response
        await self.set_tag("pending_request", None)
        await self.set_tag("last_response", {
            "request_id": request_id,
            "result": result
        })
```

## UI System

The **UI system** generates user interfaces from channel data. Currently it uses the `ui_state` and `ui_cmds` channels.

> **Note:** The UI system is transitioning to use `ui_schema` and `tag_values` channels in the future.

### How UI Works

1. Your app defines UI components (variables, parameters, actions)
2. The `ui_state` channel carries the current state of all components
3. The `ui_cmds` channel carries user commands (button clicks, parameter changes)
4. The UI renders based on `ui_state` and sends commands via `ui_cmds`

```
┌──────────────┐                      ┌──────────────┐
│     App      │ ──── ui_state ────►  │      UI      │
│              │ ◄─── ui_cmds ──────  │   (browser)  │
└──────────────┘                      └──────────────┘
```

### UI Components

Define components in your UI class:

```python
from pydoover import ui

class MyUI:
    def __init__(self):
        # Display values (published to ui_state)
        self.temperature = ui.NumericVariable(
            "temp", "Temperature", precision=1, unit="°C"
        )
        self.is_running = ui.BooleanVariable("running", "Running")

        # User inputs (generate ui_cmds when changed)
        self.setpoint = ui.NumericParameter("setpoint", "Setpoint")
        self.start_btn = ui.Action("start", "Start")

    def fetch(self):
        return (self.temperature, self.is_running,
                self.setpoint, self.start_btn)

    def update(self, temp, running):
        self.temperature.update(temp)
        self.is_running.update(running)
```

### Handling UI Commands

UI commands come through callbacks:

```python
class MyApplication(Application):
    async def setup(self):
        self.ui = MyUI()
        self.ui_manager.add_children(*self.ui.fetch())

    @ui.callback("start")
    async def on_start(self, new_value):
        """Called when user clicks start button."""
        await self.start_process()
        self.ui.start_btn.coerce(None)  # Clear button state

    @ui.callback("setpoint")
    async def on_setpoint_change(self, new_value):
        """Called when user changes setpoint."""
        self.target_temp = new_value
```

## Inter-Agent Communication

Since every agent has channels, agents can communicate by publishing and subscribing to each other's channels.

### Coordinator Pattern

One agent coordinates multiple workers:

```python
class CoordinatorApplication(Application):
    async def main_loop(self):
        # Collect status from worker agents via their tag_values channels
        worker_keys = [w.value for w in self.config.worker_apps.elements]

        statuses = {}
        for key in worker_keys:
            status = self.get_tag("status", app_key=key)
            statuses[key] = status or "unknown"

        # Aggregate and publish our own status
        all_running = all(s == "running" for s in statuses.values())
        await self.set_tag("worker_statuses", statuses)
        await self.set_tag("all_workers_running", all_running)
```

### Worker Pattern

Workers respond to coordinator commands:

```python
class WorkerApplication(Application):
    async def main_loop(self):
        # Check for commands from coordinator
        my_key = self.config.app_key.value
        coordinator_key = self.config.coordinator_key.value

        command = self.get_tag(f"command_{my_key}", app_key=coordinator_key)

        if command == "shutdown":
            await self.graceful_shutdown()
            await self.set_tag("status", "stopped")
        elif command == "restart":
            await self.restart()
            await self.set_tag("status", "running")

        # Report status back via our tag_values channel
        await self.set_tag("status", self.current_status)
        await self.set_tag("last_heartbeat", datetime.now().isoformat())
```

### Direct Channel Publishing

For high-volume or specialized data, publish directly to custom channels:

```python
class MyApplication(Application):
    async def main_loop(self):
        # High-frequency sensor data - use dedicated channel
        await self.device_agent.publish_to_channel_async(
            "high_freq_sensors",
            json.dumps({
                "timestamp": time.time(),
                "readings": self.sensor_buffer
            })
        )

        # Aggregated data for other agents - use tags
        await self.set_tag("sensor_summary", {
            "avg": statistics.mean(self.sensor_buffer),
            "count": len(self.sensor_buffer)
        })
```

## Alerts and Notifications

Alerts use the UI system's AlertStream component:

```python
from pydoover import ui

class MyUI:
    def __init__(self):
        self.notifications = ui.AlertStream()

    def fetch(self):
        return (self.notifications,)

class MyApplication(Application):
    async def setup(self):
        self.ui = MyUI()
        self.ui_manager.add_children(*self.ui.fetch())

    async def send_alert(self, message: str):
        """Send alert through UI system."""
        await self.ui.notifications.send_alert(message)

        # Also log to a channel for history
        await self.device_agent.publish_to_channel_async(
            "alerts",
            json.dumps({
                "message": message,
                "timestamp": datetime.now().isoformat()
            })
        )
```

### Alert Deduplication

```python
class MyApplication(Application):
    def __init__(self, *args, **kwargs):
        super().__init__(*args, **kwargs)
        self.active_alerts = set()

    async def send_alert_once(self, alert_id: str, message: str):
        """Send alert only once until cleared."""
        if alert_id in self.active_alerts:
            return False

        await self.ui.notifications.send_alert(message)
        self.active_alerts.add(alert_id)
        return True

    def clear_alert(self, alert_id: str):
        self.active_alerts.discard(alert_id)

    async def main_loop(self):
        if self.temperature > 35:
            await self.send_alert_once("high_temp", f"High temp: {self.temperature}°C")
        else:
            self.clear_alert("high_temp")
```

## Publishing Patterns

### Rate-Limited Publishing

```python
import time

class MyApplication(Application):
    def __init__(self, *args, **kwargs):
        super().__init__(*args, **kwargs)
        self.last_publish_times = {}

    async def publish_throttled(self, channel: str, data: dict,
                                min_interval: float = 5.0):
        """Publish with rate limiting per channel."""
        now = time.time()
        last_time = self.last_publish_times.get(channel, 0)

        if now - last_time >= min_interval:
            await self.device_agent.publish_to_channel_async(
                channel, json.dumps(data)
            )
            self.last_publish_times[channel] = now
            return True
        return False
```

### Change-Based Publishing

```python
class MyApplication(Application):
    def __init__(self, *args, **kwargs):
        super().__init__(*args, **kwargs)
        self.last_values = {}

    async def publish_on_change(self, channel: str, key: str,
                                value: float, threshold: float = 0.1):
        """Publish only when value changes significantly."""
        last = self.last_values.get(key)

        if last is None or abs(value - last) >= threshold:
            await self.device_agent.publish_to_channel_async(
                channel,
                json.dumps({"key": key, "value": value})
            )
            self.last_values[key] = value
            return True
        return False
```

### Structured Channel Data

```python
async def publish_telemetry(self):
    """Publish well-structured telemetry data."""
    data = {
        "metadata": {
            "device_id": self.config.device_id.value,
            "app_version": "1.2.0",
            "timestamp": datetime.now().isoformat()
        },
        "readings": {
            "temperature": {
                "value": self.temperature,
                "unit": "celsius",
                "quality": "good"
            }
        },
        "status": {
            "uptime_seconds": self.uptime,
            "error_count": self.error_count
        }
    }

    await self.device_agent.publish_to_channel_async(
        "device_telemetry", json.dumps(data)
    )
```

## Data Aggregation

### Rolling Statistics

```python
from collections import deque
import statistics

class RollingStats:
    def __init__(self, window_size: int = 100):
        self.values = deque(maxlen=window_size)

    def add(self, value: float):
        self.values.append(value)

    @property
    def mean(self) -> float:
        return statistics.mean(self.values) if self.values else 0.0

    @property
    def min(self) -> float:
        return min(self.values) if self.values else 0.0

    @property
    def max(self) -> float:
        return max(self.values) if self.values else 0.0

class MyApplication(Application):
    def __init__(self, *args, **kwargs):
        super().__init__(*args, **kwargs)
        self.temp_stats = RollingStats(window_size=60)

    async def main_loop(self):
        self.temp_stats.add(self.temperature)

        # Publish aggregates via tags
        await self.set_tag("temperature_stats", {
            "current": self.temperature,
            "mean": round(self.temp_stats.mean, 2),
            "min": round(self.temp_stats.min, 2),
            "max": round(self.temp_stats.max, 2)
        })
```

## Debugging

### Channel Viewer

Open the channel viewer to inspect published data:

```bash
doover app channels
```

### Debug Channel

```python
class MyApplication(Application):
    async def debug_log(self, category: str, message: str, data: dict = None):
        """Publish debug info to debug channel."""
        if not self.config.debug_enabled.value:
            return

        await self.device_agent.publish_to_channel_async(
            "debug",
            json.dumps({
                "timestamp": datetime.now().isoformat(),
                "category": category,
                "message": message,
                "data": data
            })
        )
```

## Best Practices

### Choose the Right Mechanism

| Use Case | Mechanism | Why |
|----------|-----------|-----|
| State that other agents read | Tags (`tag_values` channel) | Persisted, easy to query |
| High-frequency data | Direct channel publish | Lower overhead |
| User-visible values | UI components (`ui_state`) | Rendered in UI |
| Historical/logging data | Custom channels | Time-series storage |

### Tag Naming

```python
# Use descriptive, hierarchical names
await self.set_tag("sensor.temperature.indoor", 25.5)
await self.set_tag("status.running", True)
await self.set_tag("status.error_count", 0)
```

### Error Handling

```python
async def safe_publish(self, channel: str, data: dict):
    """Publish with error handling."""
    try:
        await self.device_agent.publish_to_channel_async(
            channel, json.dumps(data)
        )
    except Exception as e:
        log.error(f"Failed to publish to {channel}: {e}")
        # Don't re-raise - allow app to continue
```

### Memory Management

```python
class MyApplication(Application):
    def __init__(self, *args, **kwargs):
        super().__init__(*args, **kwargs)
        # Use bounded collections
        self.value_history = deque(maxlen=1000)
        self.last_values = {}

    async def main_loop(self):
        # Periodic cleanup
        if len(self.last_values) > 100:
            self.last_values.clear()
```
