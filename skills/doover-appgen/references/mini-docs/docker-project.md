# Docker Project Setup

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

## Entry Point

The `__init__.py` file bootstraps your application:

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

Use the multi-stage build pattern for efficient images:

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

## Environment Variables

Applications receive configuration via environment variables:

| Variable | Description |
|----------|-------------|
| `APP_KEY` | Unique key for this app instance |
| `CONFIG_FP` | Path to configuration JSON file |
| `HEALTHCHECK_PORT` | Port for health check endpoint |

### Using in Application

```python
import os

class MyApplication(Application):
    async def setup(self):
        app_key = os.environ.get("APP_KEY")
        config_path = os.environ.get("CONFIG_FP")
        log.info(f"Starting app {app_key}")
```

## Publishing Workflow

```bash
# 1. Ensure you're authenticated
doover auth login

# 2. Run tests
doover app test

# 3. Build for target platforms
doover app build

# 4. Publish to platform
doover app publish

# For staging/testing
doover app publish --staging

# Production with specific profile
doover app publish --profile production
```

## Versioning

Use semantic versioning in your app:

```python
# In application.py or __init__.py
__version__ = "1.2.3"

class MyApplication(Application):
    async def setup(self):
        await self.set_tag("app_version", __version__)
```

- MAJOR: Breaking changes
- MINOR: New features, backwards compatible
- PATCH: Bug fixes

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

## Testing

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
