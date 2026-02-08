# Cloud Handler & Events

Both processors and integrations use the same handler pattern for Lambda-based execution.

## Handler Entry Point

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
