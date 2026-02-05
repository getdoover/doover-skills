# doover_config.json

The `doover_config.json` file contains application metadata and is auto-generated from your config schema.

## Structure (Device App)

```json
{
  "my_app": {
    "key": "uuid-goes-here",
    "name": "my_app",
    "display_name": "My Application",
    "type": "DEV",
    "visibility": "PUB",
    "allow_many": true,
    "description": "Short description",
    "long_description": "README.md",
    "depends_on": ["platform_interface"],
    "owner_org_key": "",
    "image_name": "ghcr.io/getdoover/my_app",
    "container_registry_profile_key": "",
    "build_args": "--platform linux/amd64,linux/arm64",
    "config_schema": {
      "$schema": "https://json-schema.org/draft/2020-12/schema",
      "type": "object",
      "properties": {
        "enabled": {
          "title": "Feature Enabled",
          "type": "boolean",
          "default": true
        }
      },
      "required": ["api_key"]
    }
  }
}
```

## Structure (Processor/Integration)

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

## Key Fields

| Field | Purpose |
|-------|---------|
| `key` | Unique identifier (UUID) |
| `name` | Internal name (snake_case) |
| `display_name` | Human-readable name |
| `type` | `DEV` (device), `PRO` (processor), or `INT` (integration) |
| `visibility` | `PUB` (public) or `PRI` (private) |
| `allow_many` | Can multiple instances run? |
| `depends_on` | Required system apps (device apps only) |
| `image_name` | Docker image registry path (device apps only) |
| `build_args` | Docker build arguments (device apps only) |
| `handler` | Lambda handler path (cloud apps only) |
| `lambda_config` | Lambda runtime settings (cloud apps only) |
| `config_schema` | JSON Schema for configuration |

## Application Dependencies

Device apps can depend on system services via the `depends_on` field:

```json
{
  "my_app": {
    "depends_on": [
      "platform_interface",
      "device_agent",
      "modbus_interface"
    ]
  }
}
```

### Common Dependencies

| Dependency | Purpose | Access Via |
|------------|---------|------------|
| `platform_interface` | GPIO, digital/analog I/O | `self.platform_iface` |
| `device_agent` | Channel publishing, tag management | `self.device_agent` |
| `modbus_interface` | Modbus RTU/TCP communication | `self.modbus_iface` |

Only include dependencies your app actually uses.

## Regenerating

Regenerate after config changes:

```bash
uv run export-config
```

Or via Python:

```python
from my_app.app_config import export
export()
```
