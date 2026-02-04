---
name: doover-device-apps
description: Comprehensive guide for creating and developing Doover device applications, including state machines, workers, and hardware I/O patterns
user_invocable: true
---

# Doover App Development

This skill provides comprehensive guidance for creating, developing, and deploying Doover **device applications**.

**This skill focuses on device apps.** For cloud apps (processors and integrations), see the `doover-cloud-apps` skill.

**This skill focuses on development of existing device applications.** for guidance specifically on creating / setting up devops of a new application - please see the 'doover-app-workflow' skill.

## What is a Device App?

A device app is a containerized (docker) Python application (primarily, but not exclusively python) that runs on Doover devices. Device apps:

- Run in Docker containers on linux devices
- Communicate via channels (data streaming) and tags (state persistence)
- Provide UI components for user interaction
- Can control hardware via the platform interface
- Support state machines for complex workflows


## Typical Project Structure

```
my-app/
├── src/my_app/
│   ├── __init__.py         # Entry point with main() function
│   ├── application.py      # Core Application class
│   ├── app_config.py       # Configuration schema definitions
│   ├── app_ui.py           # UI component definitions
│   └── app_state.py        # State machine (optional)
├── simulators/
│   ├── sample/
│   │   ├── main.py         # Simulator application
│   │   ├── Dockerfile
│   │   └── pyproject.toml
│   ├── docker-compose.yml  # Local testing orchestration
│   └── app_config.json     # Sample configuration
├── tests/
│   ├── __init__.py
│   └── test_imports.py     # Basic validation tests
├── doover_config.json      # Application metadata and schema
├── pyproject.toml          # Python project configuration
├── Dockerfile              # Application container
└── README.md
```

## Application Class

The core of every Doover app is a class inheriting from `pydoover.docker.Application`.

### Basic Structure

```python
from pydoover.docker import Application, run_app
from pydoover import ui

class MyApplication(Application):
    config: MyConfig  # Type hint for IDE autocomplete

    def __init__(self, *args, **kwargs):
        super().__init__(*args, **kwargs)
        self.ui = None
        self.state = None

    async def setup(self):
        """Initialize UI, state machine, and resources."""
        self.ui = MyUI()
        self.ui_manager.add_children(*self.ui.fetch())
        self.ui_manager.set_display_name(self.config.display_name.value)

    async def main_loop(self):
        """Called repeatedly - implement your main logic here."""
        # Read inputs
        value = self.get_tag("sensor_value")

        # Process data
        result = self.process(value)

        # Update outputs
        self.ui.update(result)
        await self.set_tag("processed_value", result)
```

### Lifecycle Methods

| Method | Purpose | When Called |
|--------|---------|-------------|
| `__init__` | Initialize instance variables | Once at startup |
| `setup()` | Initialize UI, state machine, start workers | Once after init |
| `main_loop()` | Main application logic | Repeatedly |

### UI Callbacks

Handle user interactions with the `@ui.callback()` decorator:

```python
class MyApplication(Application):
    async def setup(self):
        self.ui = MyUI()
        self.ui_manager.add_children(*self.ui.fetch())

    @ui.callback("start_button")
    async def on_start(self, new_value):
        """Called when user clicks start button."""
        await self.set_tag("running", True)
        self.ui.start_button.coerce(None)  # Clear button state

    @ui.callback("threshold_param")
    async def on_threshold_change(self, new_value):
        """Called when user changes threshold parameter."""
        self.threshold = new_value
```

### Accessing Configuration

Access configuration values via the `config` attribute:

```python
async def main_loop(self):
    # Access config values
    pin = self.config.output_pin.value
    enabled = self.config.feature_enabled.value
    items = [item.value for item in self.config.items.elements]
```

### Loop Control

Control the main loop timing:

```python
class MyApplication(Application):
    loop_target_period = 2  # Target 2 seconds between loop iterations
```

## Configuration Schema

Define user-configurable parameters in `app_config.py`.

### Basic Types

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

### Application References

Reference other Doover apps:

```python
self.simulator_key = config.Application(
    "Simulator App Key",
    description="Key of the simulator app to read from"
)
```

This generates a special format field that validates app keys.

### Arrays

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

### Nested Objects

Create complex nested structures:

```python
self.modbus_device = config.Object("Modbus Device")
self.modbus_device.add_elements(
    config.String("Host", default="localhost"),
    config.Integer("Port", default=502),
    config.Integer("Unit ID", default=1)
)
```

### Enums

Define enumerated choices:

```python
self.mode = config.Enum(
    "Operating Mode",
    choices=["auto", "manual", "standby"],
    default="auto"
)
```

### Computed Properties

Add convenience properties:

```python
class MyConfig(config.Schema):
    def __init__(self):
        self.timeout_minutes = config.Integer("Timeout (minutes)", default=5)

    @property
    def timeout_seconds(self):
        return self.timeout_minutes.value * 60
```

## UI Components

Define user interface elements in `app_ui.py`.

### Variables (Display)

Show values to users:

```python
from pydoover import ui

class MyUI:
    def __init__(self):
        # Boolean status
        self.is_running = ui.BooleanVariable("running", "Running")

        # Text display
        self.status_text = ui.TextVariable("status", "Status")

        # Numeric with precision
        self.temperature = ui.NumericVariable(
            "temp",
            "Temperature",
            precision=1,
            unit="°C"
        )

        # DateTime
        self.last_update = ui.DateTimeVariable("updated", "Last Update")

    def fetch(self):
        return (self.is_running, self.status_text,
                self.temperature, self.last_update)

    def update(self, running, status, temp):
        self.is_running.update(running)
        self.status_text.update(status)
        self.temperature.update(temp)
        self.last_update.update(datetime.now())
```

### Parameters (User Input)

Accept input from users:

```python
# Text input
self.message = ui.TextParameter("message", "Message to Send")

# Numeric input
self.setpoint = ui.NumericParameter(
    "setpoint",
    "Temperature Setpoint",
    precision=1
)
```

### Actions (Buttons)

Create clickable actions:

```python
# Simple button
self.start = ui.Action("start", "Start")

# Styled button with confirmation
self.emergency_stop = ui.Action(
    "estop",
    "Emergency Stop",
    colour=ui.Colour.red,
    requires_confirm=True
)

# Hidden button (show/hide dynamically)
self.reset = ui.Action("reset", "Reset", hidden=True)

# Position for ordering
self.action1 = ui.Action("a1", "First", position=1)
self.action2 = ui.Action("a2", "Second", position=2)
```

### State Commands (Multi-Option)

Create option selectors:

```python
self.mode = ui.StateCommand(
    "mode",
    "Operating Mode",
    user_options=[
        ui.Option("auto", "Automatic"),
        ui.Option("manual", "Manual"),
        ui.Option("standby", "Standby")
    ]
)
```

### Warnings and Alerts

Display warnings and send notifications:

```python
# Warning indicator (show/hide based on condition)
self.low_battery = ui.WarningIndicator(
    "low_battery",
    "Low Battery Warning",
    hidden=True
)

# Alert stream for notifications
self.notifications = ui.AlertStream()

# In application code:
await self.ui.notifications.send_alert("Battery critically low!")
```

### Range Coloring

Color numeric values based on ranges:

```python
self.voltage = ui.NumericVariable(
    "voltage",
    "Battery Voltage",
    precision=2,
    ranges=[
        ui.Range("Low", 0, 11.5, ui.Colour.red),
        ui.Range("Normal", 11.5, 13.5, ui.Colour.green),
        ui.Range("High", 13.5, 15, ui.Colour.yellow)
    ]
)
```

### Submodules (Grouping)

Group related UI elements:

```python
class MyUI:
    def __init__(self):
        # Create submodule
        self.battery = ui.Submodule("battery", "Battery Status")

        # Create child elements
        self.voltage = ui.NumericVariable("voltage", "Voltage")
        self.current = ui.NumericVariable("current", "Current")
        self.charge_btn = ui.Action("charge", "Start Charging")

        # Add children to submodule
        self.battery.add_children(
            self.voltage,
            self.current,
            self.charge_btn
        )

    def fetch(self):
        return (self.battery,)  # Return parent, children included
```

## doover_config.json

The `doover_config.json` file contains application metadata and is auto-generated from your config schema.

### Structure

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

### Key Fields

| Field | Purpose |
|-------|---------|
| `key` | Unique identifier (UUID) |
| `name` | Internal name (snake_case) |
| `display_name` | Human-readable name |
| `type` | `DEV` or `PROD` |
| `visibility` | `PUB` (public) or `PRI` (private) |
| `allow_many` | Can multiple instances run? |
| `depends_on` | Required system apps |
| `image_name` | Docker image registry path |
| `build_args` | Docker build arguments |
| `config_schema` | JSON Schema for configuration |

### Regenerating

Regenerate after config changes:

```bash
uv run export-config
```

Or via Python:

```python
from my_app.app_config import export
export()
```

## Simulators

Simulators enable local testing without real hardware.

### Simulator Application

Create `simulators/sample/main.py`:

```python
import random
from pydoover.docker import Application, run_app
from pydoover import config

class SampleSimulator(Application):
    async def setup(self):
        pass

    async def main_loop(self):
        # Simulate sensor data
        await self.set_tag("temperature", random.uniform(20, 30))
        await self.set_tag("humidity", random.uniform(40, 60))

def main():
    run_app(SampleSimulator(config=config.Schema()))

if __name__ == "__main__":
    main()
```

### Docker Compose

Create `simulators/docker-compose.yml`:

```yaml
services:
  device_agent:
    image: spaneng/doover_device_agent:apps
    network_mode: host

  sample_simulator:
    build: ./sample
    network_mode: host
    environment:
      - APP_KEY=sim_app_key

  my_app:
    build: ../
    network_mode: host
    environment:
      - APP_KEY=test_app_key
      - CONFIG_FP=/app/simulators/app_config.json
```

### Sample Configuration

Create `simulators/app_config.json`:

```json
{
  "enabled": true,
  "simulator_app_key": "sim_app_key",
  "threshold": 25.0
}
```

### Running Locally

```bash
doover app run
```

This runs `docker compose up` in the simulators directory.

## Platform Interface

Access hardware I/O via the platform interface.

### Digital I/O

```python
class MyApplication(Application):
    async def main_loop(self):
        # Read digital input
        value = await self.platform_iface.get_di_async([1, 2, 3])
        # value = {1: True, 2: False, 3: True}

        # Write digital output
        await self.platform_iface.set_do_async(pin=4, value=True)
```

### Reading Multiple Pins

```python
# Get pin numbers from config
pins = [p.value for p in self.config.input_pins.elements]

# Read all pins
values = await self.platform_iface.get_di_async(pins)

# Check specific pin
if values.get(pins[0]):
    # Pin is HIGH
    pass
```

## Tags (State Persistence)

Tags store state that persists between loop iterations and can be shared between apps.

### Setting Tags

```python
# Set a simple value
await self.set_tag("temperature", 25.5)

# Set structured data
await self.set_tag("status", {
    "state": "running",
    "uptime": 3600,
    "errors": []
})
```

### Getting Tags

```python
# Get with default
temp = self.get_tag("temperature", default=0.0)

# Get from another app
sim_value = self.get_tag("sensor_reading", app_key="sim_app_key")
```

## Channels (Data Streaming)

Publish data to channels for logging and external consumption.

### Publishing Data

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

### Debugging Channels

Open the channel viewer:

```bash
doover app channels
```

## Entry Point

The `__init__.py` file bootstraps your application.

### Standard Pattern

```python
from pydoover.docker import run_app
from .application import MyApplication
from .app_config import MyConfig

def main():
    run_app(MyApplication(config=MyConfig()))
```

### pyproject.toml Script

```toml
[project.scripts]
doover-app-run = "my_app:main"
export-config = "my_app.app_config:export"
```

## Dockerfile

Use the multi-stage build pattern for efficient images.

### Standard Dockerfile

```dockerfile
FROM spaneng/doover_device_base AS base_image
LABEL com.doover.app="true"
LABEL com.doover.managed="true"
HEALTHCHECK --interval=30s --timeout=2s --start-period=5s \
    CMD curl -f "127.0.0.1:$HEALTHCHECK_PORT" || exit 1

## BUILDER STAGE ##
FROM base_image AS builder
COPY --from=ghcr.io/astral-sh/uv:0.7.3 /uv /uvx /bin/
ENV UV_COMPILE_BYTECODE=1 UV_LINK_MODE=copy
ENV UV_PYTHON_DOWNLOADS=0
WORKDIR /app
RUN uv venv --system-site-packages
RUN --mount=type=cache,target=/root/.cache/uv \
    --mount=type=bind,source=uv.lock,target=uv.lock \
    --mount=type=bind,source=pyproject.toml,target=pyproject.toml \
    uv sync --locked --no-install-project --no-dev
COPY . /app
RUN --mount=type=cache,target=/root/.cache/uv \
    uv sync --locked --no-dev

## FINAL STAGE ##
FROM base_image AS final_image
COPY --from=builder --chown=app:app /app /app
ENV PATH="/app/.venv/bin:$PATH"
CMD ["doover-app-run"]
```

### Key Labels

| Label | Purpose |
|-------|---------|
| `com.doover.app="true"` | Identifies as Doover app |
| `com.doover.managed="true"` | Platform manages lifecycle |

## Advanced Patterns

Patterns extracted from production device apps: state machines, workers, and hardware I/O.

### State Machines

State machines manage complex device lifecycles with multiple modes, timeouts, and transitions. Use `pydoover.state.StateMachine` with `queued=True` so transitions are serialized during processing.

**Basic state machine:** define `states` (with optional `timeout` and `on_timeout`) and `transitions` (trigger, source, dest; use `"*"` for any source). Implement `on_enter_<state>` and `on_exit_<state>` async callbacks for side effects.

```python
from pydoover.state import StateMachine
import logging
log = logging.getLogger(__name__)

class MyAppState:
    states = [
        {"name": "off"},
        {"name": "starting", "timeout": 10, "on_timeout": "timeout_error"},
        {"name": "running"},
        {"name": "stopping", "timeout": 5, "on_timeout": "force_stop"},
        {"name": "error", "timeout": 60, "on_timeout": "reset"},
    ]
    transitions = [
        {"trigger": "start", "source": "off", "dest": "starting"},
        {"trigger": "started", "source": "starting", "dest": "running"},
        {"trigger": "stop", "source": "running", "dest": "stopping"},
        {"trigger": "stopped", "source": "stopping", "dest": "off"},
        {"trigger": "timeout_error", "source": "starting", "dest": "error"},
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

    async def on_enter_starting(self):
        log.info("Starting device...")
    async def on_enter_running(self):
        log.info("Device is running")
    async def on_enter_error(self):
        log.error("Device entered error state")
```

**State machine with conditions:** add an `evaluate_state()` that reads inputs/tags and triggers transitions (e.g. when `run_requested` and state is `off`, call `await self.start_auto()`). Use `spin_state(max_iterations=15)` to re-evaluate until the state stabilizes.

```python
async def evaluate_state(self):
    is_running = self.app.get_is_running()
    run_requested = self.app.get_tag("run_requested")
    if self.state == "off" and run_requested:
        await self.start_auto()
    elif self.state == "starting_auto" and is_running:
        await self.started()
    elif self.state == "running_auto" and not run_requested:
        await self.stop_auto()
    # ... etc

async def spin_state(self, max_iterations=15):
    for _ in range(max_iterations):
        old = self.state
        await self.evaluate_state()
        if self.state == old:
            break
    return self.state
```

**Using in the application:** in `setup()`, create `self.state = MyAppState()` (or `MyAppState(self)` if it needs app reference). In `main_loop()`, call `current_state = await self.state.spin_state()`, then drive outputs and UI from `current_state` and `await self.set_tag("state", current_state)`.

### Workers and Scheduled Tasks

- **Async worker:** use an `asyncio.Event` to gate work, an `asyncio.Queue` for results, and `asyncio.create_task(worker.run())` in `setup()`. For CPU-bound work use `loop.run_in_executor(None, fn)`.
- **Thread worker:** for I/O (e.g. video capture) use a `threading.Thread` with a `queue.Queue`; from the main loop call `queue.get(timeout=1.0)` to receive the latest frame.
- **Scheduled task:** store `shutdown_at` and run `asyncio.create_task(_worker(shutdown_at))` that sleeps in a loop (e.g. `min(remaining, 60)` seconds) until the time, then perform the action. Cancel the task to cancel the schedule.

### Hardware I/O Patterns

- **Debounced input:** keep `last_value`, `last_change_time`, and `confirmed_value`; only set `confirmed_value` when `time.time() - last_change_time >= stable_time`. Use for noisy digital inputs.
- **Output with safety:** rate-limit changes (min interval between toggles), skip if state unchanged, and check a config flag (e.g. `outputs_enabled`) before calling `set_do_async`.
- **Multiple pins:** read all at once with `get_di_async(pin_list)` and map results by pin number; use helpers like `any(inputs.values())` or `all(inputs.values())` for combined conditions.

### Error Recovery and Performance

- **State-based recovery:** add an `error` state with `timeout` and `on_timeout: "attempt_recovery"`, and a `recovering` state that calls app recovery logic and triggers `recovery_success` or `recovery_failed`; track retry count and cap it.
- **Graceful degradation:** try primary source, then secondary, then cached data; update cache when any source succeeds and set a status tag when none are available.
- **Throttling:** only publish or call external APIs when `time.time() - last_time >= interval`.
- **Batching:** accumulate updates in a dict and flush when size reaches a limit or on a timer.

## Best Practices

### Loop Period

Set an appropriate loop period to balance responsiveness and resource usage:

```python
class MyApplication(Application):
    loop_target_period = 2  # 2 seconds for most apps
    # Use 0.1-0.5 for responsive hardware control
    # Use 5-60 for slow-changing data
```

### Error Handling

Handle errors gracefully to prevent app crashes:

```python
async def main_loop(self):
    try:
        data = await self.fetch_data()
        await self.process(data)
    except ConnectionError as e:
        log.error(f"Connection failed: {e}")
        await self.set_tag("status", "error")
        # Don't re-raise - let loop continue
    except Exception as e:
        log.exception(f"Unexpected error: {e}")
        raise  # Re-raise critical errors
```

### Debouncing Hardware Inputs

Prevent spurious state changes from noisy inputs:

```python
class MyApplication(Application):
    def __init__(self, *args, **kwargs):
        super().__init__(*args, **kwargs)
        self.last_value = None
        self.last_change_time = 0

    def get_debounced_value(self, current_value, stable_time=0.5):
        """Return stable value after debounce period."""
        if current_value != self.last_value:
            self.last_value = current_value
            self.last_change_time = time.time()

        if time.time() - self.last_change_time < stable_time:
            return None  # Still settling

        return current_value
```

### Clearing Action States

Always clear action buttons after handling:

```python
@ui.callback("start_button")
async def on_start(self, new_value):
    await self.start_process()
    self.ui.start_button.coerce(None)  # Clear button
```

### Logging

Use Python's logging module:

```python
import logging

log = logging.getLogger(__name__)

class MyApplication(Application):
    async def main_loop(self):
        log.info(f"Processing started")
        log.debug(f"Current state: {self.state}")
        log.warning(f"Low battery: {voltage}V")
        log.error(f"Failed to connect: {error}")
```

### Testing

Write basic import and config tests:

```python
# tests/test_imports.py
def test_import_app():
    from my_app.application import MyApplication
    assert MyApplication

def test_config():
    from my_app.app_config import MyConfig
    config = MyConfig()
    assert isinstance(config.to_dict(), dict)

def test_ui():
    from my_app.app_ui import MyUI
    ui = MyUI()
    assert ui.fetch()
```

Run tests:

```bash
doover app test
```

### Code Quality

Lint and format your code:

```bash
doover app lint --fix
doover app format --fix
```
