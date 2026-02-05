# Docker Advanced Patterns

Patterns for state machines, workers, and hardware I/O in device apps.

## State Machines

State machines manage complex device lifecycles with multiple modes, timeouts, and transitions. Use `pydoover.state.StateMachine` with `queued=True` so transitions are serialized during processing.

### Basic State Machine

Define `states` (with optional `timeout` and `on_timeout`) and `transitions` (trigger, source, dest; use `"*"` for any source). Implement `on_enter_<state>` and `on_exit_<state>` async callbacks for side effects.

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

### State Machine with Conditions

Add an `evaluate_state()` that reads inputs/tags and triggers transitions:

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

### Using in Application

In `setup()`, create `self.state = MyAppState()` (or `MyAppState(self)` if it needs app reference). In `main_loop()`, call `current_state = await self.state.spin_state()`, then drive outputs and UI from `current_state` and `await self.set_tag("state", current_state)`.

## Workers and Scheduled Tasks

### Async Worker

Use an `asyncio.Event` to gate work, an `asyncio.Queue` for results, and `asyncio.create_task(worker.run())` in `setup()`. For CPU-bound work use `loop.run_in_executor(None, fn)`.

### Thread Worker

For I/O (e.g. video capture) use a `threading.Thread` with a `queue.Queue`; from the main loop call `queue.get(timeout=1.0)` to receive the latest frame.

### Scheduled Task

Store `shutdown_at` and run `asyncio.create_task(_worker(shutdown_at))` that sleeps in a loop (e.g. `min(remaining, 60)` seconds) until the time, then perform the action. Cancel the task to cancel the schedule.

## Hardware I/O Patterns

### Platform Interface

Access hardware I/O via the platform interface. Requires `"platform_interface"` in `depends_on`.

```python
class MyApplication(Application):
    async def main_loop(self):
        # Read digital inputs
        values = await self.platform_iface.get_di_async([1, 2, 3])
        # values = {1: True, 2: False, 3: True}

        # Write digital output
        await self.platform_iface.set_do_async(pin=4, value=True)

        # Read analog inputs (if supported)
        analog = await self.platform_iface.get_ai_async([1, 2])
        # analog = {1: 2.5, 2: 3.3}
```

### Reading Multiple Pins

```python
# Get pin numbers from config
pins = [p.value for p in self.config.input_pins.elements]

# Read all pins at once
values = await self.platform_iface.get_di_async(pins)

# Check specific pin
if values.get(pins[0]):
    # Pin is HIGH
    pass

# Combined conditions
if any(values.values()):  # Any pin high
    pass
if all(values.values()):  # All pins high
    pass
```

### Modbus Interface

For Modbus RTU/TCP communication. Requires `"modbus_interface"` in `depends_on`.

```python
class MyApplication(Application):
    async def main_loop(self):
        # Read holding registers (function code 3)
        registers = await self.modbus_iface.read_registers_async(
            start_address=100,
            count=10,
            register_type=3  # 3 = holding registers
        )

        # Read input registers (function code 4)
        inputs = await self.modbus_iface.read_registers_async(
            start_address=0,
            count=5,
            register_type=4  # 4 = input registers
        )

        # Write single register
        await self.modbus_iface.write_register_async(
            address=200,
            value=1
        )
```

### Debounced Input

Keep `last_value`, `last_change_time`, and `confirmed_value`; only set `confirmed_value` when `time.time() - last_change_time >= stable_time`. Use for noisy digital inputs.

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

### Output with Safety

Rate-limit changes (min interval between toggles), skip if state unchanged, and check a config flag (e.g. `outputs_enabled`) before calling `set_do_async`.

## Data Aggregation

### Rolling Statistics

```python
from collections import deque
import statistics

class RollingStats:
    def __init__(self, window_size: int = 100):
        self.values = deque(maxlen=window_size)

    def add(self, value: float):
        self.values.append(value)

    @property
    def mean(self) -> float:
        return statistics.mean(self.values) if self.values else 0.0

    @property
    def min(self) -> float:
        return min(self.values) if self.values else 0.0

    @property
    def max(self) -> float:
        return max(self.values) if self.values else 0.0

class MyApplication(Application):
    def __init__(self, *args, **kwargs):
        super().__init__(*args, **kwargs)
        self.temp_stats = RollingStats(window_size=60)

    async def main_loop(self):
        self.temp_stats.add(self.temperature)

        # Publish aggregates via tags
        await self.set_tag("temperature_stats", {
            "current": self.temperature,
            "mean": round(self.temp_stats.mean, 2),
            "min": round(self.temp_stats.min, 2),
            "max": round(self.temp_stats.max, 2)
        })
```

## Error Recovery and Performance

### State-based Recovery

Add an `error` state with `timeout` and `on_timeout: "attempt_recovery"`, and a `recovering` state that calls app recovery logic and triggers `recovery_success` or `recovery_failed`; track retry count and cap it.

### Graceful Degradation

Try primary source, then secondary, then cached data; update cache when any source succeeds and set a status tag when none are available.

### Throttling

Only publish or call external APIs when `time.time() - last_time >= interval`.

### Batching

Accumulate updates in a dict and flush when size reaches a limit or on a timer.
