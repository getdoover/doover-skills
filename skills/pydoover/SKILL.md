---
name: pydoover
description: API reference for the pydoover Python library for Doover application development
user_invocable: true
---

# pydoover Python Library

pydoover is the Python library for building applications on the Doover platform. It provides the Application framework, UI components, configuration schemas, hardware interfaces, and cloud connectivity.

## Installation

```bash
pip install pydoover
```

Or with uv:

```bash
uv add pydoover
```

## Package Structure

```
pydoover/
├── docker/           # Application framework
│   ├── application   # Base Application class
│   ├── device_agent/ # Cloud connectivity (Device Agent)
│   ├── platform/     # Hardware I/O interface
│   └── modbus/       # Modbus protocol interface
├── ui/               # UI components
├── config/           # Configuration schemas
├── state/            # State machine support
└── utils/            # Utility functions
```

## Application Class

The `Application` class is the foundation for all Doover apps.

### Basic Usage

```python
from pydoover.docker import Application, run_app
from pydoover.config import Schema

class MyApp(Application):
    async def setup(self):
        """Called once at startup."""
        self.set_tag("ready", True)

    async def main_loop(self):
        """Called repeatedly."""
        value = self.get_di(0)
        self.set_tag("input", value)

if __name__ == "__main__":
    run_app(MyApp(config=Schema()))
```

### Constructor

```python
Application(
    config: Schema,                          # Configuration schema
    app_key: str = None,                     # Unique app identifier
    is_async: bool = None,                   # Force async mode
    device_agent: DeviceAgentInterface = None,
    platform_iface: PlatformInterface = None,
    modbus_iface: ModbusInterface = None,
    name: str = None,
    test_mode: bool = False,
    config_fp: str = None,                   # Config file path
    healthcheck_port: int = None,
)
```

### Lifecycle Methods

| Method | Purpose |
|--------|---------|
| `setup()` | Initialize UI, state, resources. Called once. |
| `main_loop()` | Main logic. Called repeatedly. |
| `on_shutdown_at(dt)` | Called when shutdown is scheduled. |
| `check_can_shutdown()` | Return True if safe to shutdown. |

### Loop Control

```python
class MyApp(Application):
    loop_target_period = 2  # Target 2 seconds between iterations
```

### Key Attributes

| Attribute | Type | Description |
|-----------|------|-------------|
| `config` | `Schema` | Configuration values |
| `device_agent` | `DeviceAgentInterface` | Cloud connectivity |
| `platform_iface` | `PlatformInterface` | Hardware I/O |
| `modbus_iface` | `ModbusInterface` | Modbus protocol |
| `ui_manager` | `UIManager` | UI element manager |
| `app_key` | `str` | Unique app identifier |
| `test_mode` | `bool` | Running in test mode |

## Tag Methods

Tags provide key-value state persistence via the `tag_values` channel.

### Setting Tags

```python
# Set tag on this app
await self.set_tag("temperature", 25.5)
await self.set_tag("status", {"state": "running", "uptime": 3600})

# Set only if value changed (default behavior)
await self.set_tag("value", 100, only_if_changed=True)

# Set tag on another app
await self.set_tag("command", "start", app_key="other_app_key")
```

### Getting Tags

```python
# Get from this app
temp = self.get_tag("temperature", default=0.0)

# Get from another app
value = self.get_tag("sensor", app_key="sim_app_key", default=None)
```

### Global Tags

```python
# System-wide tags
await self.set_global_tag("system_status", "online")
status = self.get_global_tag("system_status", default="unknown")
```

### Tag Subscriptions

```python
def on_temperature_change(tag_key, value):
    print(f"Temperature changed to {value}")

self.subscribe_to_tag("temperature", on_temperature_change)

# Subscribe to another app's tag
self.subscribe_to_tag("sensor", callback, app_key="other_app")

# Subscribe to global tag
self.subscribe_to_tag("system_status", callback, global_tag=True)
```

## Hardware Interface (Platform)

Access digital and analog I/O through `platform_iface`.

### Digital I/O

```python
# Read digital input
value = self.get_di(pin=0)
value = await self.platform_iface.get_di_async(pin=0)

# Read multiple pins
values = await self.platform_iface.get_di_async([0, 1, 2])
# Returns: {0: True, 1: False, 2: True}

# Write digital output
self.set_do(pin=4, value=True)
await self.platform_iface.set_do_async(pin=4, value=1)

# Schedule output change
await self.platform_iface.schedule_do(pin=4, value=False, delay_secs=5.0)
```

### Analog I/O

```python
# Read analog input
voltage = self.get_ai(pin=0)
voltage = await self.platform_iface.get_ai_async(pin=0)

# Write analog output
self.set_ao(pin=0, value=2.5)
await self.platform_iface.set_ao_async(pin=0, value=2.5)

# Schedule analog output
await self.platform_iface.schedule_ao(pin=0, value=5.0, delay_secs=10.0)
```

### Pulse Counter

Count pulses on digital inputs (e.g., flow meters):

```python
# Create pulse counter
counter = self.platform_iface.get_new_pulse_counter(
    pin=5,
    edge="rising",           # "rising", "falling", or "both"
    rate_window_secs=60,
    auto_start=True
)

# Set callback for pulses
def on_pulse(pin, di_value, dt_secs, count, edge):
    rate = count / dt_secs if dt_secs > 0 else 0
    print(f"Count: {count}, Rate: {rate}/sec")

counter.callback = on_pulse

# Access values
total_count = counter.count
timestamps = counter.pulse_timestamps

# Control
counter.start_listener_pulses()
counter.stop_listener_pulses()
```

## Modbus Interface

Communicate via Modbus RTU/TCP through `modbus_iface`.

### Reading Registers

```python
# Read holding registers
values = self.read_modbus_registers(
    address=100,
    count=10,
    register_type="holding",
    modbus_id=1,
    bus_id="bus1"
)

# Async version
values = await self.modbus_iface.read_registers_async(
    start_address=100,
    num_registers=10,
    register_type="holding",
    modbus_id=1,
    bus_id="bus1"
)
```

### Writing Registers

```python
# Write holding registers
self.write_modbus_registers(
    address=200,
    values=[100, 200, 300],
    register_type="holding",
    modbus_id=1
)

# Async version
await self.modbus_iface.write_registers_async(
    start_address=200,
    values=[100, 200, 300],
    register_type="holding"
)
```

### Register Types

| Type | Description |
|------|-------------|
| `"coil"` | Read/write boolean (function 1/5/15) |
| `"discrete_input"` | Read-only boolean (function 2) |
| `"holding"` | Read/write register (function 3/6/16) |
| `"input"` | Read-only register (function 4) |

### Polling Subscriptions

```python
def on_registers_read(values):
    print(f"Read values: {values}")

self.modbus_iface.add_read_register_subscription(
    start_address=100,
    num_registers=10,
    register_type="holding",
    poll_secs=5.0,
    callback=on_registers_read
)
```

## Channel Methods

Publish and subscribe to channels directly.

### Publishing

```python
import json

# Publish to channel
await self.device_agent.publish_to_channel_async(
    "sensor_data",
    json.dumps({"temperature": 25.5}),
    max_age=300,        # Optional: max age in seconds
    record_log=True     # Optional: force logging
)

# Sync version
self.publish_to_channel("sensor_data", data)
```

### Subscribing

```python
def on_message(channel_name, data):
    print(f"Received on {channel_name}: {data}")

self.subscribe_to_channel("commands", on_message)

# Via device agent
self.device_agent.add_subscription("commands", on_message)
```

### Getting Channel Data

```python
# Get latest aggregate from channel
data = self.device_agent.get_channel_aggregate("sensor_data")
```

## Configuration Schema

Define user-configurable parameters in `pydoover.config`.

### Schema Class

```python
from pydoover.config import Schema, Integer, String, Boolean, Number, Enum, Array, Object

class MyConfig(Schema):
    def __init__(self):
        self.pump_pin = Integer(
            "Pump Pin",
            default=0,
            minimum=0,
            maximum=31,
            description="Digital output pin for pump"
        )

        self.threshold = Number(
            "Threshold",
            default=25.0,
            minimum=0.0,
            maximum=100.0
        )

        self.enabled = Boolean(
            "Enabled",
            default=True
        )

        self.device_name = String(
            "Device Name",
            default="sensor-1",
            length=50,
            pattern=r"^[a-z0-9-]+$"
        )

        self.mode = Enum(
            "Mode",
            choices=["auto", "manual", "standby"],
            default="auto"
        )
```

### Accessing Config Values

```python
class MyApp(Application):
    async def main_loop(self):
        pin = self.config.pump_pin.value
        threshold = self.config.threshold.value
        enabled = self.config.enabled.value
```

### Config Types

#### Integer

```python
count = Integer(
    "Count",
    default=10,
    minimum=0,
    maximum=100,
    description="Number of items"
)
```

#### Number (float)

```python
rate = Number(
    "Flow Rate",
    default=5.5,
    minimum=0.0,
    maximum=100.0
)
```

#### Boolean

```python
enabled = Boolean(
    "Enable Feature",
    default=True
)
```

#### String

```python
name = String(
    "Name",
    default="device",
    length=50,
    pattern=r"^[a-zA-Z0-9_-]+$"
)
```

#### DateTime

```python
scheduled_time = DateTime(
    "Scheduled Time",
    description="When to run"
)
```

#### Enum

```python
# List of choices
mode = Enum(
    "Mode",
    choices=["fast", "slow", "off"],
    default="slow"
)

# Python Enum
from enum import Enum as PyEnum

class Speed(PyEnum):
    FAST = "fast"
    SLOW = "slow"

speed = Enum("Speed", choices=Speed, default=Speed.SLOW)
```

#### Array

```python
pins = Array(
    "Output Pins",
    element=Integer("Pin"),
    min_items=1,
    max_items=8,
    unique_items=True
)

# Accessing elements
for pin in self.config.pins.elements:
    value = pin.value
```

#### Object

```python
device = Object("Device Settings")
device.add_elements(
    String("Host", default="localhost"),
    Integer("Port", default=502),
    Boolean("Enabled", default=True)
)
```

### Special Config Types

#### Application (reference to another app)

```python
logger_app = Application(
    "Logger App",
    description="App key of the data logger"
)
```

#### Device (reference to a device)

```python
target = Device(
    "Target Device",
    description="Device to send commands to"
)
```

### Exporting Schema

```python
from pathlib import Path

def export():
    MyConfig().export(
        Path(__file__).parents[2] / "doover_config.json",
        "my_app"
    )
```

## UI Components

UI elements in `pydoover.ui` for display and user interaction.

### Variables (Display Values)

```python
from pydoover import ui

# Numeric display
temperature = ui.NumericVariable(
    name="temperature",
    display_name="Temperature",
    precision=1,
    ranges=[
        ui.Range("Low", 0, 15, ui.Colour.blue),
        ui.Range("Normal", 15, 30, ui.Colour.green),
        ui.Range("High", 30, 50, ui.Colour.red),
    ]
)

# Update value
temperature.update(25.5)

# Text display
status = ui.TextVariable(
    name="status",
    display_name="Status"
)
status.update("Running")

# Boolean display
running = ui.BooleanVariable(
    name="running",
    display_name="Is Running"
)
running.update(True)

# DateTime display
last_update = ui.DateTimeVariable(
    name="updated",
    display_name="Last Update"
)
last_update.update(datetime.now())
```

### Parameters (User Input)

```python
# Numeric input
setpoint = ui.NumericParameter(
    name="setpoint",
    display_name="Setpoint",
    min_val=0,
    max_val=100
)

# Text input
message = ui.TextParameter(
    name="message",
    display_name="Message",
    is_text_area=False
)

# Multi-line text
notes = ui.TextParameter(
    name="notes",
    display_name="Notes",
    is_text_area=True
)

# DateTime picker
schedule = ui.DateTimeParameter(
    name="schedule",
    display_name="Schedule Time",
    include_time=True
)
```

### Actions (Buttons)

```python
start_btn = ui.Action(
    name="start",
    display_name="Start",
    colour=ui.Colour.green,
    requires_confirm=False
)

stop_btn = ui.Action(
    name="stop",
    display_name="Emergency Stop",
    colour=ui.Colour.red,
    requires_confirm=True
)
```

### Slider

```python
speed = ui.Slider(
    name="speed",
    display_name="Speed Control",
    min_val=0,
    max_val=100,
    step_size=5
)

# Dual slider for range selection
range_slider = ui.Slider(
    name="range",
    display_name="Value Range",
    min_val=0,
    max_val=100,
    dual_slider=True
)
```

### StateCommand (Dropdown)

```python
mode = ui.StateCommand(
    name="mode",
    display_name="Operating Mode",
    user_options=[
        ui.Option("auto", "Automatic"),
        ui.Option("manual", "Manual"),
        ui.Option("standby", "Standby")
    ]
)
```

### WarningIndicator

```python
warning = ui.WarningIndicator(
    name="low_battery",
    display_name="Low Battery",
    can_cancel=True
)
```

### AlertStream

```python
alerts = ui.AlertStream()

# Send alert
await alerts.send_alert("Battery critically low!")
```

### Submodule (Grouping)

```python
battery = ui.Submodule("battery", "Battery Status")
battery.add_children(
    ui.NumericVariable("voltage", "Voltage"),
    ui.NumericVariable("current", "Current"),
    ui.Action("charge", "Start Charging")
)
```

### Colour Constants

```python
ui.Colour.blue
ui.Colour.red
ui.Colour.green
ui.Colour.yellow
ui.Colour.orange
ui.Colour.purple
ui.Colour.grey
ui.Colour.limegreen
ui.Colour.tomato

# Custom colors
ui.Colour.from_hex("#FF5733")
ui.Colour.from_string("coral")
```

### Range (for NumericVariable)

```python
ui.Range(
    label="Normal",
    min_val=10,
    max_val=30,
    colour=ui.Colour.green,
    show_on_graph=True
)
```

### Option (for StateCommand)

```python
ui.Option(name="value", display_name="Display Text")
```

## UI Decorators

Use decorators for cleaner UI callback handling.

### @ui.callback

```python
class MyApp(Application):
    async def setup(self):
        self.ui = MyUI()
        self.ui_manager.add_children(*self.ui.fetch())

    @ui.callback("start_button")
    async def on_start(self, new_value):
        await self.start_process()
        self.ui.start_button.coerce(None)

    @ui.callback(r"do_\d+_toggle")  # Regex pattern
    async def on_do_toggle(self, element, new_value):
        pin = int(element.name.split("_")[1])
        await self.set_do(pin, new_value)
```

### @ui.action

```python
class MyApp(Application):
    @ui.action("start", display_name="Start", colour=ui.Colour.green)
    async def start(self):
        await self.begin_operation()
```

### @ui.slider

```python
class MyApp(Application):
    @ui.slider("volume", display_name="Volume", min_val=0, max_val=100)
    async def on_volume(self, new_value):
        self.set_volume(new_value)
```

### @ui.state_command

```python
class MyApp(Application):
    @ui.state_command(
        "mode",
        display_name="Mode",
        user_options=[ui.Option("fast", "Fast"), ui.Option("slow", "Slow")]
    )
    async def on_mode(self, new_value):
        self.current_mode = new_value
```

## State Machine

Async state machine built on the `transitions` library.

```python
from pydoover.state import StateMachine

class MyState:
    states = [
        {"name": "off"},
        {"name": "starting", "timeout": 30, "on_timeout": "start_failed"},
        {"name": "running"},
        {"name": "stopping", "timeout": 10, "on_timeout": "force_stop"},
        {"name": "error", "timeout": 300, "on_timeout": "reset"},
    ]

    transitions = [
        {"trigger": "start", "source": "off", "dest": "starting"},
        {"trigger": "started", "source": "starting", "dest": "running"},
        {"trigger": "stop", "source": "running", "dest": "stopping"},
        {"trigger": "stopped", "source": "stopping", "dest": "off"},
        {"trigger": "start_failed", "source": "starting", "dest": "error"},
        {"trigger": "force_stop", "source": "stopping", "dest": "off"},
        {"trigger": "reset", "source": "error", "dest": "off"},
        {"trigger": "error", "source": "*", "dest": "error"},
    ]

    def __init__(self):
        self.state_machine = StateMachine(
            states=self.states,
            transitions=self.transitions,
            model=self,
            initial="off",
            queued=True,
        )

    # State callbacks
    async def on_enter_running(self):
        print("Entered running state")

    async def on_exit_running(self):
        print("Exiting running state")

# Usage
state = MyState()
await state.start()      # Trigger transition
print(state.state)       # Current state: "starting"
await state.started()    # Trigger next transition
```

### State Definition

```python
{
    "name": "running",           # State name
    "timeout": 60,               # Seconds before timeout
    "on_timeout": "timeout_fn",  # Trigger on timeout
    "on_enter": "enter_fn",      # Callback on enter
    "on_exit": "exit_fn",        # Callback on exit
}
```

### Transition Definition

```python
{
    "trigger": "start",     # Method name to trigger
    "source": "off",        # Source state(s), "*" for any
    "dest": "running",      # Destination state
}
```

## Utility Functions

### Async/Sync Compatibility

#### maybe_async decorator

```python
from pydoover.utils import maybe_async

class MyClass:
    @maybe_async()
    def my_method(self, value):
        return f"sync: {value}"

    async def my_method_async(self, value):
        return f"async: {value}"

# Automatically uses sync or async based on context
obj = MyClass()
result = obj.my_method("test")        # Sync context
result = await obj.my_method("test")  # Async context
```

#### call_maybe_async

```python
from pydoover.utils import call_maybe_async

async def process():
    # Works with both sync and async functions
    result = await call_maybe_async(some_function, arg1, arg2)
    return result
```

#### get_is_async

```python
from pydoover.utils import get_is_async

is_async = get_is_async()  # Detect current context
```

### Change Detection

```python
from pydoover.utils import on_change

class Sensor:
    def my_callback(self, new_val, old_val, is_first, name):
        print(f"{name}: {old_val} -> {new_val}")

    @on_change("my_callback", name="temperature")
    def read_temperature(self):
        return get_sensor_value()
```

### Diff Operations

```python
from pydoover.utils import generate_diff, apply_diff

old = {"a": 1, "b": 2, "c": 3}
new = {"a": 1, "b": 5, "d": 4}

diff = generate_diff(old, new)
# {"b": 5, "c": None, "d": 4}

result = apply_diff(old, diff)
# {"a": 1, "b": 5, "d": 4}
```

### 4-20mA Sensor Scaling

```python
from pydoover.utils import map_reading

# Map 4-20mA signal to 0-100 range
value = map_reading(
    in_val=12.0,           # Current reading (mA)
    output_values=[0, 100], # Output range
    raw_readings=[4, 20],   # Input range (4-20mA)
    ignore_below=3          # Ignore readings below 3mA
)
# Returns: 50.0
```

### CaseInsensitiveDict

```python
from pydoover.utils import CaseInsensitiveDict

d = CaseInsensitiveDict({"Content-Type": "application/json"})
print(d["content-type"])  # "application/json"
print(d["CONTENT-TYPE"])  # "application/json"
```

## Device Agent Interface

Cloud connectivity through `device_agent`.

### Availability

```python
# Check if Device Agent is available
if self.device_agent.get_is_dda_available():
    # Connected to local agent

if self.device_agent.get_is_dda_online():
    # Connected to cloud

# Wait for availability
await self.device_agent.await_dda_available_async(timeout=300)
```

### Channel Sync

```python
# Wait for channels to sync
await self.device_agent.wait_for_channels_sync_async(
    channels=["config", "commands"],
    timeout=10
)
```

## UIManager

Manages UI elements and cloud synchronization.

```python
from pydoover.ui import UIManager

manager = UIManager(
    app_key="my_app",
    client=device_agent,
    auto_start=True,
    is_async=True
)

# Set UI elements
manager.set_children([temp_var, status_var, start_btn])

# Add children
manager.add_children(new_element1, new_element2)

# Get/set commands
cmd = manager.get_command("start_button")
manager.coerce_command("start_button", None)

# Force sync
await manager.handle_comms_async(force_log=True)

# Set status icon
manager.set_status_icon("running")
```

## Complete Example

```python
from pydoover.docker import Application, run_app
from pydoover.config import Schema, Integer, Number, Boolean
from pydoover import ui
from datetime import datetime

class PumpConfig(Schema):
    def __init__(self):
        self.pump_pin = Integer("Pump Pin", default=0)
        self.flow_pin = Integer("Flow Sensor Pin", default=5)
        self.target_flow = Number("Target Flow (L/min)", default=10.0)

class PumpUI:
    def __init__(self):
        self.flow_rate = ui.NumericVariable(
            "flow", "Flow Rate", precision=2,
            ranges=[
                ui.Range("Low", 0, 5, ui.Colour.orange),
                ui.Range("OK", 5, 15, ui.Colour.green),
                ui.Range("High", 15, 50, ui.Colour.red),
            ]
        )
        self.status = ui.TextVariable("status", "Status")
        self.start = ui.Action("start", "Start", colour=ui.Colour.green)
        self.stop = ui.Action("stop", "Stop", colour=ui.Colour.red)

    def fetch(self):
        return (self.flow_rate, self.status, self.start, self.stop)

class PumpApp(Application):
    config: PumpConfig

    async def setup(self):
        self.ui = PumpUI()
        self.ui_manager.add_children(*self.ui.fetch())
        self.running = False

        # Setup pulse counter
        self.counter = self.platform_iface.get_new_pulse_counter(
            pin=self.config.flow_pin.value,
            edge="rising"
        )

    @ui.callback("start")
    async def on_start(self, value):
        self.running = True
        await self.platform_iface.set_do_async(self.config.pump_pin.value, 1)
        self.ui.start.coerce(None)

    @ui.callback("stop")
    async def on_stop(self, value):
        self.running = False
        await self.platform_iface.set_do_async(self.config.pump_pin.value, 0)
        self.ui.stop.coerce(None)

    async def main_loop(self):
        # Update flow rate
        flow = self.counter.count / 60.0
        self.ui.flow_rate.update(flow)

        # Update status
        status = "Running" if self.running else "Stopped"
        self.ui.status.update(status)

        # Update tags
        await self.set_tag("flow_rate", flow)
        await self.set_tag("running", self.running)

if __name__ == "__main__":
    run_app(PumpApp(config=PumpConfig()))
```
