---
name: doover-cloud-apps
description: Guide for building Doover processors and integrations (cloud-based apps)
user_invocable: true
---

# Doover Cloud Apps

This skill covers **processors** and **integrations** - cloud-based Doover applications that run serverless and are triggered by events. For more information on **device applications** please see the `doover-device-apps` skill.

For more 

## Overview

| Type | Code | Purpose | Installed On | Triggers |
|------|------|---------|--------------|----------|
| **Processor** | `PRO` | Cloud logic for devices | Device | Channels, schedules |
| **Integration** | `INT` | External data ingestion | Organization | HTTP endpoints, schedules |

### When to Use Each

**Use a Device App when:**
- You need hardware access (GPIO, Modbus)
- Logic must run locally on the device
- Real-time response is critical

**Use a Processor when:**
- Logic can run in the cloud
- You need to react to channel messages
- You want to update device UI remotely
- Periodic cloud-based tasks are needed

**Use an Integration when:**
- External systems need to push data to Doover
- You need organization-wide data reception
- Bridging third-party APIs with devices

## Project Structure

### Processor

```
my-processor/
├── src/my_processor/
│   ├── __init__.py       # Handler entry point
│   ├── application.py    # Main application logic
│   ├── app_config.py     # Configuration schema
│   └── app_ui.py         # UI components (optional)
├── doover_config.json    # App metadata
├── build.sh              # Build script for deployment package (see below)
└── pyproject.toml
```

### Integration

```
my-integration/
├── src/my_integration/
│   ├── __init__.py       # Handler entry point
│   ├── application.py    # Main application logic
│   └── app_config.py     # Configuration schema
├── doover_config.json    # App metadata
├── build.sh              # Build script for deployment package (see below)
└── pyproject.toml
```

### Build script (deployment package)

Processors and integrations are deployed as a zip package. Add a build script to the project root (e.g. `build.sh`) and run it to produce `package.zip` for publishing:

```sh
#!/bin/sh

uv export --frozen --no-dev --no-editable --quiet -o requirements.txt

uv pip install \
   --no-deps \
   --no-installer-metadata \
   --no-compile-bytecode \
   --python-platform x86_64-manylinux2014 \
   --python 3.13 \
   --quiet \
   --target packages_export \
   --refresh \
   -r requirements.txt

rm -f package.zip

cd packages_export
zip -rq ../package.zip .
cd ..

zip -rq package.zip src

echo "OK"
```

This exports locked dependencies, installs them for the Lambda runtime platform, zips the installed packages, then adds your `src` tree. Ensure the script is executable (`chmod +x build.sh`). Add the build outputs to `.gitignore` so they are not committed:

```
packages_export/
package.zip
```

Run the script before publishing so the correct `package.zip` is used.

## Handler Entry Point

Both processors and integrations use the same handler pattern:

```python
# src/my_app/__init__.py
from typing import Any
from pydoover.cloud.processor import run_app
from .application import MyApp
from .app_config import MyAppConfig

def handler(event: dict[str, Any], context):
    """Lambda handler entry point."""
    MyAppConfig.clear_elements()
    run_app(
        MyApp(config=MyAppConfig()),
        event,
        context,
    )
```

## Application Class

Inherit from `pydoover.cloud.processor.Application`:

```python
from pydoover.cloud.processor import Application

class MyApp(Application):
    config: MyAppConfig  # Type hint for IDE

    async def setup(self):
        """Called once per invocation before event processing."""
        pass

    async def close(self):
        """Called once per invocation after event processing."""
        pass
```

## Event Handlers

Implement handlers for the events you want to respond to:

### on_message_create (Channel Messages)

Triggered when a message is published to a subscribed channel:

```python
from pydoover.cloud.processor import MessageCreateEvent

async def on_message_create(self, event: MessageCreateEvent):
    """React to channel messages."""
    channel = event.channel_name
    data = event.message.data
    author = event.author_id
    owner = event.owner_id

    # Process the message
    if channel == "sensor_data":
        await self.process_sensor_data(data)
```

### on_schedule (Scheduled Triggers)

Triggered on a schedule (cron or rate):

```python
from pydoover.cloud.processor import ScheduleEvent

async def on_schedule(self, event: ScheduleEvent):
    """Run periodic tasks."""
    schedule_id = event.schedule_id

    # Perform scheduled work
    await self.sync_external_api()
    await self.cleanup_old_data()
```

### on_ingestion_endpoint (HTTP Inbound)

Triggered when external systems POST to the ingestion endpoint:

```python
from pydoover.cloud.processor import IngestionEndpointEvent

async def on_ingestion_endpoint(self, event: IngestionEndpointEvent):
    """Receive external data."""
    payload = event.payload  # Parsed by parse_ingestion_event_payload
    ingestion_id = event.ingestion_id
    org_id = event.organisation_id

    # Process and forward to devices
    device_id = payload.get("device_id")
    await self.forward_to_device(device_id, payload)

def parse_ingestion_event_payload(self, payload: str):
    """Override to customize payload parsing."""
    import base64
    import json
    raw = base64.b64decode(payload)
    return json.loads(raw)
```

### on_deployment (First Install)

Triggered when the app is first installed:

```python
from pydoover.cloud.processor import DeploymentEvent

async def on_deployment(self, event: DeploymentEvent):
    """Initialize on first installation."""
    await self.set_tag("installed_at", datetime.now().isoformat())
```

### on_aggregate_update (Config Changes)

Triggered when aggregate data (like config) changes:

```python
from pydoover.cloud.processor import AggregateUpdateEvent

async def on_aggregate_update(self, event: AggregateUpdateEvent):
    """React to configuration changes."""
    pass
```

### on_manual_invoke (User Triggered)

Triggered manually from the dashboard:

```python
from pydoover.cloud.processor import ManualInvokeEvent

async def on_manual_invoke(self, event: ManualInvokeEvent):
    """Handle manual user action."""
    await self.run_diagnostic()
```

## Configuration

### Processor Config

```python
from pydoover import config
from pydoover.cloud.processor import ManySubscriptionConfig, ScheduleConfig

class MyProcessorConfig(config.Schema):
    def __init__(self):
        # Subscribe to channels
        self.subscription = ManySubscriptionConfig()

        # Optional: scheduled triggers
        self.schedule = ScheduleConfig()

        # App-specific config
        self.api_endpoint = config.String(
            "API Endpoint",
            default="https://api.example.com"
        )
```

### Integration Config

```python
from pydoover import config
from pydoover.cloud.processor import (
    IngestionEndpointConfig,
    ExtendedPermissionsConfig
)

class MyIntegrationConfig(config.Schema):
    def __init__(self):
        # HTTP ingestion endpoint
        self.integration = IngestionEndpointConfig()

        # Access to multiple devices
        self.permissions = ExtendedPermissionsConfig()

        # App-specific config
        self.api_key = config.String(
            "API Key",
            description="External service API key"
        )
```

### Subscription Config

Subscribe to device channels:

```python
self.subscription = ManySubscriptionConfig()
# User configures which channels to subscribe to
# e.g., ["sensor_data", "events", "alerts"]
```

### Schedule Config

Configure periodic triggers:

```python
self.schedule = ScheduleConfig()
# User sets: "rate(5 minutes)", "cron(0 9 * * ? *)", or "disabled"
```

### Ingestion Endpoint Config

Configure HTTP endpoint security:

```python
self.integration = IngestionEndpointConfig()
# Includes:
# - CIDR range filtering (IP whitelist)
# - HMAC-SHA256 signing verification
# - Throttling (max requests/second)
# - Mini-tokens for low-bandwidth devices
```

### Extended Permissions

Grant access to multiple devices (integrations):

```python
self.permissions = ExtendedPermissionsConfig()
# Configure access to:
# - Specific device IDs
# - Device groups
# - Apps installed on devices
# - All devices in organization
```

## doover_config.json

```json
{
  "my_processor": {
    "name": "my_processor",
    "display_name": "My Processor",
    "type": "PRO",
    "visibility": "PUB",
    "allow_many": true,
    "handler": "src.my_processor.handler",
    "lambda_config": {
      "Runtime": "python3.13",
      "Timeout": 300,
      "MemorySize": 128,
      "Handler": "src.my_processor.handler"
    },
    "config_schema": { }
  }
}
```

For integrations, use `"type": "INT"`.

## State Management

### Tags

Persist state across invocations:

```python
async def on_message_create(self, event):
    # Get tag with default
    devices = await self.get_tag("device_mapping", {})

    # Update state
    devices[event.owner_id] = {"last_seen": datetime.now().isoformat()}

    # Save tag
    await self.set_tag("device_mapping", devices)
```

### No In-Memory State

Cloud apps are serverless - each invocation starts fresh. Use tags or channels for any state that must persist.

## UI Management (Processors)

Processors can update device UI remotely:

```python
from pydoover.ui import ApplicationVariant

class MyProcessor(Application):
    async def setup(self):
        self.ui = MyUI()
        self.ui_manager.add_children(*self.ui.fetch())
        self.ui_manager.set_variant(ApplicationVariant.stacked)

    async def on_message_create(self, event):
        # Update UI based on message
        data = event.message.data
        self.ui.temperature.update(data.get("temperature"))
        self.ui.status.update("Online")

        # Push to connected clients
        await self.ui_manager.push_async()
```

## Publishing to Channels

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

## Connection Status (Processors)

Update device connection status:

```python
from pydoover.cloud.processor import ConnectionStatus, ConnectionType
from datetime import datetime, timezone, timedelta

async def on_message_create(self, event):
    # Mark device as online
    await self.ping_connection(
        online_at=datetime.now(timezone.utc),
        connection_status=ConnectionStatus.periodic_unknown,
        connection_type=ConnectionType.periodic,
        offline_at=datetime.now(timezone.utc) + timedelta(hours=1)
    )
```

## Custom Payload Parsing

Override for non-JSON payloads:

```python
def parse_ingestion_event_payload(self, payload: str):
    """Parse protobuf or other formats."""
    import base64
    raw = base64.b64decode(payload)

    # Try protobuf
    try:
        msg = MyProtobufMessage()
        msg.ParseFromString(raw)
        return msg
    except:
        pass

    # Fallback to JSON
    import json
    return json.loads(raw)
```

## Integration + Processor Pattern

A common pattern pairs an integration with a processor:

```
External System
      │
      │ HTTP POST
      ▼
┌─────────────┐
│ Integration │  ← Receives external data
└─────────────┘
      │
      │ publish to channel
      ▼
┌─────────────┐
│  Processor  │  ← Reacts to channel message
└─────────────┘
      │
      │ update UI
      ▼
┌─────────────┐
│  Device UI  │
└─────────────┘
```

### Integration Side

```python
class MyIntegration(Application):
    async def on_ingestion_endpoint(self, event):
        payload = event.payload
        device_id = payload["device_id"]

        # Store device mapping
        devices = await self.get_tag("devices", {})
        devices[device_id] = event.agent_id
        await self.set_tag("devices", devices)

        # Forward to device channel
        await self.api.publish_message(
            event.agent_id,
            "on_external_event",
            payload
        )
```

### Processor Side

```python
class MyProcessor(Application):
    async def setup(self):
        self.ui = MyUI()
        self.ui_manager.add_children(*self.ui.fetch())

    async def on_message_create(self, event):
        if event.channel_name == "on_external_event":
            data = event.message.data

            # Update UI
            self.ui.update_from_event(data)
            await self.ui_manager.push_async()

            # Update connection status
            await self.ping_connection(
                online_at=datetime.now(timezone.utc)
            )
```

## Exporting Config

Generate doover_config.json:

```python
# In app_config.py
from pathlib import Path

def export():
    MyAppConfig().export(
        Path(__file__).parents[2] / "doover_config.json",
        "my_app"
    )

if __name__ == "__main__":
    export()
```

In pyproject.toml:

```toml
[project.scripts]
export-config = "my_app.app_config:export"
```

## Key Differences from Device Apps

| Aspect | Device App | Processor/Integration |
|--------|------------|----------------------|
| **Runs on** | Docker on device | Serverless (Lambda) |
| **Lifecycle** | Long-running daemon | Per-event invocation |
| **State** | In-memory OK | Must use tags/channels |
| **Hardware** | Direct access | No hardware access |
| **UI updates** | Local | Remote via WebSocket |
| **Entry point** | `main()` loop | `handler()` per event |
| **Triggers** | Continuous loop | Events, schedules, HTTP |

## Best Practices

### Keep Invocations Short

Lambda has timeout limits. For long operations:
- Break into multiple scheduled invocations
- Use tags to track progress
- Consider device apps for continuous processing

### Handle Cold Starts

Each invocation may be a cold start:
- Initialize efficiently in `setup()`
- Don't rely on in-memory caching
- Use tags for persistent state

### Idempotent Handlers

Messages may be delivered multiple times:
- Track processed message IDs in tags
- Design handlers to be safely re-runnable

### Error Handling

```python
async def on_message_create(self, event):
    try:
        await self.process(event)
    except Exception as e:
        # Log error
        await self.set_tag("last_error", str(e))
        # Don't re-raise unless you want retry
        raise
```
