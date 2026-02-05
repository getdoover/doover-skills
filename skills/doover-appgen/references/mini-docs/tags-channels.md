# Tags & Channels

Doover's data system is built around **channels** - named data streams that agents can publish to and subscribe to. Tags and UI are conventions built on top of channels.

## Channel Architecture

```
┌─────────┐     publish      ┌─────────────┐     subscribe     ┌─────────┐
│  Device │ ───────────────► │   Channel   │ ─────────────────► │  Cloud  │
│   App   │                  │ sensor_data │                    │   App   │
└─────────┘                  └─────────────┘                    └─────────┘
```

Each agent (device, app, user) has its own set of channels. Key system channels:
- `tag_values` - Powers the Tags system
- `ui_state` / `ui_cmds` - Powers the UI system

## Tags (State Persistence)

Tags are a convention built on the `tag_values` channel that provides key-value state persistence. When you call `set_tag()`, it publishes to `tag_values`. When you call `get_tag()`, it reads from this channel's current state.

### Setting Tags

```python
# Simple values
await self.set_tag("temperature", 25.5)
await self.set_tag("is_running", True)
await self.set_tag("status", "operational")

# Structured data
await self.set_tag("sensor_data", {
    "temperature": 25.5,
    "humidity": 60.0,
    "timestamp": datetime.now().isoformat()
})

# Lists
await self.set_tag("recent_values", [25.5, 26.0, 25.8])
```

### Getting Tags

```python
# Get with default
temp = self.get_tag("temperature", default=0.0)

# Without default (returns None if not set)
status = self.get_tag("status")
if status is None:
    status = "unknown"

# Get from another app
sim_value = self.get_tag("sensor_reading", app_key="sim_app_key")
```

### Tags from Other Apps

Access tags from other agents using their app key:

```python
# Get another app's key from config
motor_app_key = self.config.motor_control_app.value

# Read tags from that app
motor_state = self.get_tag("state", app_key=motor_app_key)
run_reason = self.get_tag("run_request_reason", app_key=motor_app_key)

if motor_state == "running":
    await self.process_motor_running()
```

### Cloud Apps (Processors/Integrations)

In cloud apps, tag operations are async:

```python
async def on_message_create(self, event):
    # Get tag with default
    devices = await self.get_tag("device_mapping", {})

    # Update state
    devices[event.owner_id] = {"last_seen": datetime.now().isoformat()}

    # Save tag
    await self.set_tag("device_mapping", devices)
```

**Note:** Cloud apps are serverless - each invocation starts fresh. Use tags for any state that must persist.

### Common Tag Patterns

**State tracking:**
```python
await self.set_tag("state", self.state.state)
await self.set_tag("state_changed_at", datetime.now().isoformat())
```

**Request/Response:**
```python
# Check for pending request
request = self.get_tag("pending_request")
if request:
    # Process request...
    await self.set_tag("pending_request", None)
    await self.set_tag("last_response", {"request_id": request["id"], "result": "success"})
```

## Channels (Data Streaming)

Publish data to channels for logging, external consumption, and inter-app communication.

### Publishing Data (Device Apps)

```python
import json

async def main_loop(self):
    data = {
        "timestamp": datetime.now().isoformat(),
        "temperature": 25.5,
        "humidity": 60.0
    }

    await self.device_agent.publish_to_channel_async(
        "sensor_data",
        json.dumps(data)
    )
```

### Channel Naming

Use descriptive, hierarchical names:

```python
await self.device_agent.publish_to_channel_async("sensors/temperature", data)
await self.device_agent.publish_to_channel_async("events/alerts", alert_data)
await self.device_agent.publish_to_channel_async("metrics/performance", metrics)
```

### Publishing Data (Cloud Apps)

Forward data to device channels:

```python
async def on_ingestion_endpoint(self, event):
    payload = event.payload
    device_agent_id = await self.get_device_agent(payload["device_id"])

    # Publish to device's channel
    await self.api.publish_message(
        device_agent_id,
        "external_events",
        payload
    )
```

## Publishing Patterns

### Rate-Limited Publishing

```python
class MyApplication(Application):
    def __init__(self, *args, **kwargs):
        super().__init__(*args, **kwargs)
        self.last_publish_times = {}

    async def publish_throttled(self, channel: str, data: dict, min_interval: float = 5.0):
        """Publish with rate limiting per channel."""
        now = time.time()
        last_time = self.last_publish_times.get(channel, 0)

        if now - last_time >= min_interval:
            await self.device_agent.publish_to_channel_async(channel, json.dumps(data))
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

    async def publish_on_change(self, channel: str, key: str, value: float, threshold: float = 0.1):
        """Publish only when value changes significantly."""
        last = self.last_values.get(key)

        if last is None or abs(value - last) >= threshold:
            await self.device_agent.publish_to_channel_async(
                channel, json.dumps({"key": key, "value": value})
            )
            self.last_values[key] = value
            return True
        return False
```

## Inter-Agent Communication

### Coordinator Pattern

One agent coordinates multiple workers:

```python
class CoordinatorApplication(Application):
    async def main_loop(self):
        # Collect status from worker agents
        worker_keys = [w.value for w in self.config.worker_apps.elements]

        statuses = {}
        for key in worker_keys:
            statuses[key] = self.get_tag("status", app_key=key) or "unknown"

        # Aggregate and publish coordinator status
        all_running = all(s == "running" for s in statuses.values())
        await self.set_tag("worker_statuses", statuses)
        await self.set_tag("all_workers_running", all_running)
```

### Worker Pattern

Workers respond to coordinator commands:

```python
class WorkerApplication(Application):
    async def main_loop(self):
        coordinator_key = self.config.coordinator_key.value
        my_key = os.environ.get("APP_KEY")

        # Check for commands from coordinator
        command = self.get_tag(f"command_{my_key}", app_key=coordinator_key)

        if command == "shutdown":
            await self.graceful_shutdown()

        # Report status back
        await self.set_tag("status", self.current_status)
        await self.set_tag("last_heartbeat", datetime.now().isoformat())
```

## Debugging Channels

Open the channel viewer:

```bash
doover app channels
```

### Debug Channel Pattern

```python
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
