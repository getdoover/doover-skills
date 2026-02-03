---
name: doover-app-explorer
description: Search and explore existing Doover apps for integration and functionality evaluation
user_invocable: true
---

# Doover App Explorer

This skill helps you search and explore existing Doover applications to evaluate functionality, find apps for integration, and understand available components.

## Important Note

The App Explorer and public API only show **public** and **Doover core** apps. Your organization may have access to additional **private apps** that are not listed here. Private apps are shared directly between organizations and won't appear in public searches.

To see all apps available to you, check your organization's app library in the Doover admin panel.

## App Explorer UI

Browse available apps visually at:

**https://admin.doover.com/app-explorer**

The web interface provides:
- Searchable list of all public applications
- App details including description and configuration
- Dependency visualization
- Installation statistics

## API Access

Query apps programmatically using the public API:

```
GET https://api.doover.com/public/applications/
```

### Pagination

| Parameter | Default | Description |
|-----------|---------|-------------|
| `page` | 1 | Page number |
| `per_page` | 100 | Results per page (max 100) |

### Example Request

```bash
curl "https://api.doover.com/public/applications/?page=1&per_page=100"
```

### Response Structure

```json
{
  "count": 33,
  "next": null,
  "previous": null,
  "results": [
    {
      "id": "93956442461108741",
      "name": "platform_interface",
      "display_name": "Platform Interface",
      "description": "Core hardware I/O interface",
      "long_description": "Full markdown documentation...",
      "type": "DEV",
      "visibility": "COR",
      "allow_many": false,
      "config_schema": { ... },
      "depends_on": [],
      "image_name": "ghcr.io/getdoover/platform_interface",
      "approx_installs": 150,
      "stars": 5
    }
  ]
}
```

## Application Fields

| Field | Type | Description |
|-------|------|-------------|
| `id` | string | Unique identifier |
| `name` | string | Machine-readable name (snake_case) |
| `display_name` | string | Human-readable title |
| `description` | string | Brief summary |
| `long_description` | string | Full markdown documentation |
| `type` | enum | App type (see below) |
| `visibility` | enum | Visibility level (see below) |
| `allow_many` | boolean | Can run multiple instances |
| `config_schema` | object | JSON Schema for configuration |
| `depends_on` | array | IDs of required apps |
| `image_name` | string | Docker image path |
| `approx_installs` | integer | Approximate install count |
| `stars` | integer | User rating |

## App Types

Doover has three types of applications:

| Type | Name | Runs On | Purpose |
|------|------|---------|---------|
| `DEV` | Device App | Docker on devices | Hardware control, local logic, device UI |
| `PRO` | Processor | Cloud (serverless) | Cloud logic triggered by channels or schedules |
| `INT` | Integration | Cloud (serverless) | Receive external data for an organization |

### Device Apps (`DEV`)

Run as Docker containers on field devices. They:
- Access hardware via platform interface (GPIO, Modbus)
- Run continuously with a main loop
- Provide real-time device UI

### Processors (`PRO`)

Cloud-based apps triggered by:
- Channel messages (react to device data)
- Schedules (periodic tasks)

Installed per-device, can update device UI remotely.

### Integrations (`INT`)

Cloud-based apps that:
- Receive HTTP POST from external systems
- Are installed at the organization level
- Forward data to device channels

See the `doover-cloud-apps` skill for detailed documentation on processors and integrations.

## Visibility Levels

| Visibility | Description | In Public API |
|------------|-------------|---------------|
| `PUB` | Public - available to all users | Yes |
| `COR` | Core - system infrastructure apps | Yes |
| `PRI` | Private - shared between specific organizations | No |

**Note:** Private apps (`PRI`) are not visible in the public API or App Explorer. They are shared directly between organizations and only accessible to authorized users.

## Core System Apps

These are foundational apps that other apps depend on:

### Platform Interface
- **Purpose**: Hardware I/O access (digital/analog inputs and outputs)
- **Provides**: gRPC interface for GPIO, pulse counting, scheduled outputs
- **Used by**: Apps needing hardware control

### Device Agent
- **Purpose**: Cloud connectivity and data management
- **Provides**: Channel publishing, tag management, remote configuration
- **Used by**: All apps needing cloud features

### Modbus Interface
- **Purpose**: Modbus RTU/TCP communication
- **Provides**: Register read/write, polling subscriptions, virtual servers
- **Used by**: Apps communicating with Modbus devices

## Finding Apps for Integration

### By Functionality

Search the API or UI for apps by their purpose:

```bash
# Fetch all apps and filter locally
curl "https://api.doover.com/public/applications/?per_page=100" | \
  jq '.results[] | select(.description | test("modbus"; "i"))'
```

### By Dependencies

Find apps that depend on a specific app:

```python
import requests

response = requests.get("https://api.doover.com/public/applications/?per_page=100")
apps = response.json()["results"]

# Find apps depending on platform_interface
platform_id = "93956442461108741"
dependent_apps = [
    app for app in apps
    if platform_id in app.get("depends_on", [])
]
```

### By Config Schema

Examine config schemas to understand integration points:

```python
# Find apps that accept application references
apps_with_app_refs = [
    app for app in apps
    if "doover-application" in str(app.get("config_schema", {}))
]
```

## Understanding Config Schemas

Config schemas use JSON Schema with Doover extensions:

### Standard Properties

```json
{
  "config_schema": {
    "$schema": "https://json-schema.org/draft/2020-12/schema",
    "type": "object",
    "properties": {
      "pump_pin": {
        "title": "Pump Output Pin",
        "type": "integer",
        "default": 0,
        "minimum": 0,
        "maximum": 31
      }
    },
    "required": ["pump_pin"]
  }
}
```

### Doover Extensions

| Extension | Description |
|-----------|-------------|
| `x-name` | Property identifier in code |
| `x-hidden` | Hide from UI (true/false) |
| `x-required` | Conditional requirement |
| `format: "doover-application"` | Reference to another app |
| `format: "doover-device"` | Reference to a device |
| `format: "doover-schedule"` | Schedule configuration |

### Application References

Apps can reference other apps in their config:

```json
{
  "data_logger_app": {
    "title": "Data Logger Application",
    "type": "string",
    "format": "doover-application",
    "description": "App key of the data logger to send data to"
  }
}
```

## Example Apps

### Small Motor Control

Controls engine ignition, starter, and emergency stop:

- **Dependencies**: Platform Interface
- **Tags exposed**: `state`, `run_request_reason`
- **Config**: Pin assignments for starter, ignition, run sense

```json
{
  "name": "small_motor_control",
  "display_name": "Small Motor Control",
  "depends_on": ["93956442461108741"],
  "config_schema": {
    "properties": {
      "starter_pin": { "type": "integer" },
      "ignition_pin": { "type": "integer" },
      "run_sense_pins": { "type": "array" }
    }
  }
}
```

### Modbus Channel Relay

Bridges Modbus devices to Doover channels:

- **Dependencies**: Modbus Interface, Device Agent
- **Purpose**: Read Modbus registers and publish as JSON to channels
- **Config**: Register mappings, data types, channel names

### Generator Control

Complex state machine for generator management:

- **Dependencies**: Platform Interface, Small Motor Control
- **Features**: Auto start/stop, warmup/cooldown cycles, scheduling
- **States**: off, starting, warmup, running, cooldown, stopping, error

## Integrating with Existing Apps

### Reading Tags from Other Apps

```python
class MyApp(Application):
    async def main_loop(self):
        # Get app key from config
        motor_app_key = self.config.motor_control_app.value

        # Read state from motor control app
        motor_state = self.get_tag("state", app_key=motor_app_key)
        run_reason = self.get_tag("run_request_reason", app_key=motor_app_key)

        if motor_state == "running":
            await self.process_motor_running()
```

### Coordinating with Other Apps

```python
class CoordinatorApp(Application):
    async def main_loop(self):
        # Read status from multiple apps
        apps = [a.value for a in self.config.managed_apps.elements]

        statuses = {}
        for app_key in apps:
            statuses[app_key] = self.get_tag("status", app_key=app_key)

        # Coordinate based on aggregate status
        all_ready = all(s == "ready" for s in statuses.values())
        if all_ready:
            await self.start_sequence()
```

### Config for App References

```python
from pydoover.config import Schema, Application

class MyConfig(Schema):
    def __init__(self):
        self.motor_control_app = Application(
            "Motor Control App",
            description="App key of the motor control instance"
        )

        self.data_logger_app = Application(
            "Data Logger App",
            description="App key for data logging"
        )
```

## Evaluating Apps

When evaluating an app for use:

1. **Check dependencies** - Ensure required apps are available
2. **Review config schema** - Understand required configuration
3. **Read long_description** - Full documentation and usage notes
4. **Check allow_many** - Whether multiple instances are supported
5. **Review tags exposed** - What data the app publishes
6. **Check install count** - Indicator of maturity/reliability

## Searching Tips

### Find Hardware Control Apps

Look for apps depending on Platform Interface:

```bash
curl -s "https://api.doover.com/public/applications/?per_page=100" | \
  jq '[.results[] | select(.depends_on | index("93956442461108741"))] |
      .[] | {name, display_name, description}'
```

### Find Protocol Bridges

Search for "modbus", "mqtt", "relay", "bridge" in descriptions:

```bash
curl -s "https://api.doover.com/public/applications/?per_page=100" | \
  jq '.results[] | select(.description | test("modbus|mqtt|bridge"; "i")) |
      {name, display_name, description}'
```

### Find Apps with Specific Config Types

```bash
# Apps accepting schedule configuration
curl -s "https://api.doover.com/public/applications/?per_page=100" | \
  jq '.results[] | select(.config_schema | tostring | test("doover-schedule")) |
      {name, display_name}'
```

## Using App Explorer in Development

### Before Building a New App

1. Search the explorer for existing functionality
2. Check if an existing app can be extended or configured
3. Look for apps with similar dependencies for patterns
4. Review config schemas for integration approaches

### When Integrating Apps

1. Find the target app's ID and exposed tags
2. Add application reference to your config schema
3. Use `get_tag()` with the app key to read data
4. Coordinate via tags or channels

### Understanding the Ecosystem

The app explorer helps you understand:
- What infrastructure apps are available
- Common patterns in config schemas
- How apps depend on each other
- What functionality already exists
