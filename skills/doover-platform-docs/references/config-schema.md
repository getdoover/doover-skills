# Configuration Schema

Define user-configurable parameters in `app_config.py`.

## Basic Types

```python
from pydoover import config
from pathlib import Path

class MyConfig(config.Schema):
    def __init__(self):
        # Boolean with default
        self.enabled = config.Boolean(
            "Feature Enabled",
            description="Enable this feature",
            default=True
        )

        # Required string (no default)
        self.api_key = config.String(
            "API Key",
            description="Your API key"
        )

        # Integer with constraints
        self.retry_count = config.Integer(
            "Retry Count",
            description="Number of retries",
            default=3
        )

        # Number (float)
        self.threshold = config.Number(
            "Threshold",
            description="Detection threshold",
            default=0.5
        )

def export():
    """Export configuration schema to doover_config.json."""
    MyConfig().export(
        Path(__file__).parents[2] / "doover_config.json",
        "my_app"
    )
```

## Application References

Reference other Doover apps:

```python
self.simulator_key = config.Application(
    "Simulator App Key",
    description="Key of the simulator app to read from"
)

self.motor_control_app = config.Application(
    "Motor Control App",
    description="App key of the motor control instance"
)
```

This generates a field with `format: "doover-application"` that validates app keys. Use `get_tag()` with `app_key` parameter to read tags from referenced apps:

```python
motor_key = self.config.motor_control_app.value
state = self.get_tag("state", app_key=motor_key)
```

## Doover Schema Extensions

Config schemas use JSON Schema with Doover-specific extensions:

| Extension | Type | Description |
|-----------|------|-------------|
| `x-name` | string | Property identifier in code |
| `x-hidden` | boolean | Hide from UI |
| `x-required` | string | Conditional requirement expression |
| `format: "doover-application"` | string | Reference to another app |
| `format: "doover-device"` | string | Reference to a device |
| `format: "doover-schedule"` | string | Schedule configuration |

These are automatically generated when using `pydoover.config` classes.

## Arrays

Define lists of values:

```python
# Simple array
self.pins = config.Array(
    "Output Pins",
    element=config.Integer("Pin Number")
)

# Access elements
for pin in self.config.pins.elements:
    value = pin.value
```

## Nested Objects

Create complex nested structures:

```python
self.modbus_device = config.Object("Modbus Device")
self.modbus_device.add_elements(
    config.String("Host", default="localhost"),
    config.Integer("Port", default=502),
    config.Integer("Unit ID", default=1)
)
```

## Enums

Define enumerated choices:

```python
self.mode = config.Enum(
    "Operating Mode",
    choices=["auto", "manual", "standby"],
    default="auto"
)
```

## Computed Properties

Add convenience properties:

```python
class MyConfig(config.Schema):
    def __init__(self):
        self.timeout_minutes = config.Integer("Timeout (minutes)", default=5)

    @property
    def timeout_seconds(self):
        return self.timeout_minutes.value * 60
```

## Accessing Configuration

In the application class, access configuration values via the `config` attribute:

```python
async def main_loop(self):
    # Access config values
    pin = self.config.output_pin.value
    enabled = self.config.feature_enabled.value
    items = [item.value for item in self.config.items.elements]
```
