# Docker Application Class

The core of every Doover device app is a class inheriting from `pydoover.docker.Application`.

## Basic Structure

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

## Lifecycle Methods

| Method | Purpose | When Called |
|--------|---------|-------------|
| `__init__` | Initialize instance variables | Once at startup |
| `setup()` | Initialize UI, state machine, start workers | Once after init |
| `main_loop()` | Main application logic | Repeatedly |

## UI Callbacks

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

## Loop Control

Control the main loop timing:

```python
class MyApplication(Application):
    loop_target_period = 2  # Target 2 seconds between loop iterations
```

Guidelines:
- Use 0.1-0.5 for responsive hardware control
- Use 2 for most apps
- Use 5-60 for slow-changing data

## Best Practices

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
