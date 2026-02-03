---
name: doover-cli
description: Complete reference for the Doover CLI app management commands
user_invocable: true
---

# Doover CLI Reference

This skill provides a complete reference for the `doover app` CLI commands used to create, develop, test, and deploy Doover applications.

## Overview

The Doover CLI provides commands for the full application lifecycle:

| Command | Purpose |
|---------|---------|
| `doover app create` | Create a new app from template |
| `doover app run` | Run app locally with simulators |
| `doover app build` | Build Docker image |
| `doover app publish` | Deploy to Doover platform |
| `doover app test` | Run pytest tests |
| `doover app lint` | Check code with ruff |
| `doover app format` | Format code with ruff |
| `doover app channels` | Debug channel data |

## App Creation

### `doover app create`

Create a new application using the interactive wizard.

```bash
doover app create --name my-app --description "My application description"
```

#### Options

| Option | Required | Description |
|--------|----------|-------------|
| `--name TEXT` | Yes | Application name (snake_case recommended) |
| `--description TEXT` | Yes | Short description of the app |
| `--git / --no-git` | No | Initialize git repository |
| `--container-registry` | No | Registry: `ghcr.io/getdoover`, `ghcr.io/other`, `DockerHub (spaneng)`, `DockerHub (other)` |
| `--owner-org-key TEXT` | No | Organization key for ownership |
| `--container-profile TEXT` | No | Container registry profile |

#### Examples

```bash
# Basic creation
doover app create --name temperature-monitor --description "Monitor temperature sensors"

# With git initialization
doover app create --name my-app --description "My app" --git

# Specify container registry
doover app create \
  --name my-app \
  --description "My app" \
  --container-registry "ghcr.io/getdoover"

# With organization ownership
doover app create \
  --name my-app \
  --description "My app" \
  --owner-org-key "org-uuid-here"
```

#### Generated Structure

The command creates a complete project:

```
my-app/
├── src/my_app/
│   ├── __init__.py
│   ├── application.py
│   ├── app_config.py
│   ├── app_ui.py
│   └── app_state.py
├── simulators/
│   ├── sample/
│   │   ├── main.py
│   │   ├── Dockerfile
│   │   └── pyproject.toml
│   ├── docker-compose.yml
│   └── app_config.json
├── tests/
│   ├── __init__.py
│   └── test_imports.py
├── doover_config.json
├── pyproject.toml
├── Dockerfile
└── README.md
```

## Local Development

### `doover app run`

Run the application locally using docker-compose from the simulators directory.

```bash
doover app run
```

#### Options

| Option | Default | Description |
|--------|---------|-------------|
| `[REMOTE]` | None | Remote host to run on |
| `--port INTEGER` | 2375 | Port for remote connection |

#### Examples

```bash
# Run locally (most common)
doover app run

# Run on remote device
doover app run 192.168.1.100

# Run on remote with custom port
doover app run 192.168.1.100 --port 2376

# Pass additional docker-compose arguments
doover app run -- --build
doover app run -- -d  # Run detached
```

#### How It Works

1. Looks for `simulators/docker-compose.yml`
2. Runs `docker compose up` with the specified services
3. Services typically include:
   - `device_agent` - Doover device agent
   - Simulator services - Generate test data
   - Application - Your app under development

#### Stopping

Press `Ctrl+C` to stop all services, or if running detached:

```bash
cd simulators && docker compose down
```

## Build and Publish

### `doover app build`

Build the Docker image for your application.

```bash
doover app build
```

#### Options

| Option | Default | Description |
|--------|---------|-------------|
| `[APP_FP]` | `.` | Path to application directory |
| `--buildx / --no-buildx` | `--buildx` | Use docker buildx for multi-platform |

#### Examples

```bash
# Build current directory
doover app build

# Build specific app
doover app build /path/to/my-app

# Build without buildx (single platform)
doover app build --no-buildx

# Pass additional docker build arguments
doover app build -- --no-cache
```

#### Build Arguments

Build arguments are read from `doover_config.json`:

```json
{
  "my_app": {
    "build_args": "--platform linux/amd64,linux/arm64"
  }
}
```

### `doover app publish`

Publish your application to the Doover platform and container registry.

```bash
doover app publish
```

#### Options

| Option | Default | Description |
|--------|---------|-------------|
| `[APP_FP]` | `.` | Path to application directory |
| `--skip-container / --no-skip-container` | `--no-skip-container` | Skip container build/push |
| `--staging / --no-staging` | `--no-staging` | Force staging mode |
| `--export-config / --no-export-config` | `--export-config` | Export config before publishing |
| `--buildx / --no-buildx` | `--buildx` | Use docker buildx |
| `--profile TEXT` | None | Config profile for authentication |

#### Examples

```bash
# Full publish (build, push, update)
doover app publish

# Update config only (skip container)
doover app publish --skip-container

# Publish to staging
doover app publish --staging

# Don't regenerate config
doover app publish --no-export-config

# Use specific profile
doover app publish --profile production
```

#### Publish Process

1. Exports configuration (regenerates `doover_config.json`)
2. Builds Docker image
3. Pushes image to configured registry
4. Updates application on Doover platform

## Testing and Quality

### `doover app test`

Run pytest tests on your application.

```bash
doover app test
```

#### Options

| Option | Default | Description |
|--------|---------|-------------|
| `[APP_FP]` | `.` | Path to application directory |

Additional arguments are passed to pytest.

#### Examples

```bash
# Run all tests
doover app test

# Run with verbose output
doover app test -- -v

# Run specific test file
doover app test -- tests/test_application.py

# Run tests matching pattern
doover app test -- -k "test_config"

# Show print statements
doover app test -- -s

# Run tests in specific app
doover app test /path/to/my-app
```

#### Test File Structure

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
    components = ui.fetch()
    assert len(components) > 0

def test_state():
    from my_app.app_state import MyState
    state = MyState()
    assert state.state_machine.state == "initial"
```

### `doover app lint`

Run the ruff linter to check code quality.

```bash
doover app lint
```

#### Options

| Option | Default | Description |
|--------|---------|-------------|
| `[APP_FP]` | `.` | Path to application directory |
| `--fix / --no-fix` | `--no-fix` | Auto-fix linting issues |

#### Examples

```bash
# Check for issues
doover app lint

# Auto-fix issues
doover app lint --fix

# Lint specific app
doover app lint /path/to/my-app

# Lint and fix specific app
doover app lint /path/to/my-app --fix
```

#### Common Issues

| Code | Issue | Fix |
|------|-------|-----|
| F401 | Unused import | Remove or use the import |
| F841 | Unused variable | Remove or prefix with `_` |
| E501 | Line too long | Break into multiple lines |
| E302 | Expected 2 blank lines | Add blank lines between functions |

### `doover app format`

Format code using ruff formatter.

```bash
doover app format
```

#### Options

| Option | Default | Description |
|--------|---------|-------------|
| `[APP_FP]` | `.` | Path to application directory |
| `--fix / --no-fix` | `--no-fix` | Apply formatting fixes |

#### Examples

```bash
# Check formatting (dry run)
doover app format

# Apply formatting
doover app format --fix

# Format specific app
doover app format /path/to/my-app --fix
```

## Debugging

### `doover app channels`

Open the channel viewer in your browser for debugging published data.

```bash
doover app channels
```

#### Options

| Option | Default | Description |
|--------|---------|-------------|
| `--host TEXT` | `localhost` | Host to connect to |
| `--port INTEGER` | `49100` | Port for channel viewer |

#### Examples

```bash
# Open local channel viewer
doover app channels

# Connect to specific host
doover app channels --host 192.168.1.100

# Use custom port
doover app channels --port 8080
```

#### Usage

1. Run your app with `doover app run`
2. Open channel viewer with `doover app channels`
3. View real-time channel data in your browser
4. Debug data format and timing issues

## Workflow Examples

### New App Development

```bash
# 1. Create new app
doover app create --name sensor-monitor --description "Monitor sensors"
cd sensor-monitor

# 2. Edit code
# ... modify src/sensor_monitor/*.py

# 3. Run locally
doover app run

# 4. Debug channels (in another terminal)
doover app channels

# 5. Run tests
doover app test

# 6. Check code quality
doover app lint --fix
doover app format --fix

# 7. Publish when ready
doover app publish
```

### Iterative Development

```bash
# Run in background
doover app run -- -d

# Make changes to code
# ...

# Rebuild and restart
cd simulators && docker compose up -d --build my_app

# View logs
docker compose logs -f my_app

# Stop when done
docker compose down
```

### CI/CD Pipeline

```bash
# Install dependencies
uv sync

# Run linting
doover app lint
if [ $? -ne 0 ]; then exit 1; fi

# Run tests
doover app test -- -v
if [ $? -ne 0 ]; then exit 1; fi

# Build image
doover app build

# Publish to staging
doover app publish --staging

# Publish to production
doover app publish --profile production
```

### Remote Development

```bash
# Deploy to remote device for testing
doover app run 192.168.1.100

# View remote logs
ssh device "docker logs -f my_app"

# Publish directly to device
doover app build
docker save my_app | ssh device "docker load"
```

## Configuration Export

### Regenerating Config

After modifying `app_config.py`, regenerate `doover_config.json`:

```bash
# Using the project script
uv run export-config

# Or directly
uv run python -c "from my_app.app_config import export; export()"
```

### Verifying Config

```bash
# Check JSON is valid
python -m json.tool doover_config.json > /dev/null

# View generated schema
cat doover_config.json | jq '.my_app.config_schema'
```

## CLI Profiles

Profiles configure which Doover environment to use. The default profile is `dv2`.

### Setting the Default Profile

Set via environment variable (recommended):

```bash
# Add to ~/.bashrc or ~/.zshrc
export DOOVER_PROFILE=dv2
```

### Per-Command Profile

Override for a single command:

```bash
doover app publish --profile dv2
doover app publish --profile staging
doover app publish --profile production
```

### Available Profiles

| Profile | Description |
|---------|-------------|
| `dv2` | Default Doover environment |

<!-- TODO: Add commands to list/manage profiles when available -->

## Troubleshooting

### Build Failures

```bash
# Clean Docker cache
docker builder prune

# Rebuild without cache
doover app build -- --no-cache

# Check Dockerfile syntax
docker build --check .
```

### Run Failures

```bash
# Check container logs
cd simulators && docker compose logs

# Verify compose file
docker compose config

# Check port conflicts
lsof -i :49100
```

### Test Failures

```bash
# Run with verbose output
doover app test -- -v --tb=long

# Run single test for debugging
doover app test -- tests/test_imports.py::test_config -v
```

### Publish Failures

```bash
# Verify authentication
doover auth status

# Check registry access
docker login ghcr.io

# Test build locally first
doover app build
```
