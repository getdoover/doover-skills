# Processor Features

Processor-specific configuration and capabilities.

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

## Connection Status

Update device connection status to show when device was last seen:

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

### Processor Side Example

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
