# Integration Features

Integration-specific configuration and capabilities.

## Configuration

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

## Device Routing

Map incoming data to Doover devices using extended permissions:

```python
class MyIntegration(Application):
    async def on_ingestion_endpoint(self, event):
        payload = event.payload
        device_id = payload["device_id"]

        # Get all devices this integration has permission to access
        agents = await self.get_extended_permission_agents()

        # Find the device matching the external ID
        for agent in agents:
            serial = await self.get_serial_number_match(agent)
            if serial == device_id:
                # Found the device, update its tags
                await self.set_tag("latest_data", payload, agent_id=agent)
                break
```

## Integration Side Example

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

The integration handles:
- Receiving HTTP webhooks from external systems
- Parsing custom payload formats
- Device routing via serial number matching
- Publishing to channels for processors to consume

The processor handles:
- Subscribing to channels
- Processing data and updating UI
- Managing connection status
