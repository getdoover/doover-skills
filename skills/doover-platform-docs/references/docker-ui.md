# Docker UI Components

Define user interface elements in `app_ui.py`.

## Variables (Display)

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

## Parameters (User Input)

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

## Actions (Buttons)

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

## State Commands (Multi-Option)

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

## Warnings and Alerts

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

### Alert Deduplication

Prevent sending duplicate alerts:

```python
class MyApplication(Application):
    def __init__(self, *args, **kwargs):
        super().__init__(*args, **kwargs)
        self.active_alerts = set()

    async def send_alert_once(self, alert_id: str, message: str):
        """Send alert only once until cleared."""
        if alert_id in self.active_alerts:
            return False

        await self.ui.notifications.send_alert(message)
        self.active_alerts.add(alert_id)
        return True

    def clear_alert(self, alert_id: str):
        self.active_alerts.discard(alert_id)

    async def main_loop(self):
        if self.temperature > 35:
            await self.send_alert_once("high_temp", f"High temp: {self.temperature}°C")
        else:
            self.clear_alert("high_temp")
```

### Showing/Hiding Warnings

```python
async def main_loop(self):
    # Show warning when condition is true
    if self.battery_voltage < 11.5:
        self.ui.low_battery.hidden = False
    else:
        self.ui.low_battery.hidden = True
```

## Range Coloring

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

## Submodules (Grouping)

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

## Registering UI in Application

```python
class MyApplication(Application):
    async def setup(self):
        self.ui = MyUI()
        self.ui_manager.add_children(*self.ui.fetch())
        self.ui_manager.set_display_name(self.config.display_name.value)
```
