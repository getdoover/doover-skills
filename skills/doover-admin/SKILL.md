---
name: doover-admin
description: Administration guide for the Doover platform
user_invocable: true
---

# Doover Administration

This skill covers administration topics for the Doover platform, including organization setup, device types, and deployment configuration.

## Organizations

Organizations are the top-level entity in Doover, containing users, devices, and applications.

### Organization Structure

```
Organization
├── Users (members with roles)
├── Device Types (templates for devices)
├── Devices (physical/virtual devices)
├── Applications (apps deployed to devices)
└── API Keys (for programmatic access)
```

### User Roles

| Role | Permissions |
|------|------------|
| Owner | Full access, billing, can delete org |
| Admin | Manage users, devices, apps |
| Developer | Deploy apps, view devices |
| Viewer | Read-only access to dashboards |

## Device Types

Device types define templates for devices with pre-configured applications.

### Creating a Device Type

Device types specify:
- Base hardware configuration
- Default applications to install
- Configuration templates
- Supported capabilities

### doover_config.json Fields

Key fields for device type integration:

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
    "owner_org_key": "org-uuid",
    "image_name": "ghcr.io/getdoover/my_app",
    "container_registry_profile_key": "",
    "build_args": "--platform linux/amd64,linux/arm64"
  }
}
```

### Field Descriptions

| Field | Type | Description |
|-------|------|-------------|
| `key` | UUID | Unique identifier for the app |
| `name` | string | Internal name (snake_case) |
| `display_name` | string | Human-readable name |
| `type` | enum | `DEV` (development) or `PROD` (production) |
| `visibility` | enum | `PUB` (public) or `PRI` (private) |
| `allow_many` | boolean | Allow multiple instances on one device |
| `description` | string | Short description |
| `long_description` | string | Path to README or long description |
| `depends_on` | array | Required system apps |
| `owner_org_key` | UUID | Owning organization |
| `image_name` | string | Docker image registry path |
| `container_registry_profile_key` | UUID | Registry credentials profile |
| `build_args` | string | Docker build arguments |

## Application Dependencies

Applications can depend on system services:

### Common Dependencies

```json
{
  "depends_on": [
    "platform_interface",
    "device_agent",
    "modbus_interface"
  ]
}
```

| Dependency | Purpose |
|------------|---------|
| `platform_interface` | GPIO, hardware I/O access |
| `device_agent` | Channel publishing, tag management |
| `modbus_interface` | Modbus RTU/TCP communication |

### Platform Interface

Provides access to device hardware:

```python
class MyApplication(Application):
    async def main_loop(self):
        # Digital input
        values = await self.platform_iface.get_di_async([1, 2, 3])

        # Digital output
        await self.platform_iface.set_do_async(pin=4, value=True)

        # Analog input (if supported)
        analog = await self.platform_iface.get_ai_async([1])
```

### Device Agent

Provides data management services:

```python
class MyApplication(Application):
    async def main_loop(self):
        # Publish to channel
        await self.device_agent.publish_to_channel_async(
            "sensor_data",
            json.dumps({"temperature": 25.5})
        )
```

### Modbus Interface

For Modbus communication:

```python
class MyApplication(Application):
    async def main_loop(self):
        # Read holding registers
        registers = await self.modbus_iface.read_registers_async(
            start_address=100,
            count=10,
            register_type=3  # Holding registers
        )

        # Write register
        await self.modbus_iface.write_register_async(
            address=200,
            value=1
        )
```

## Deployment Configuration

### Container Registry Setup

Configure container registry access in `doover_config.json`:

```json
{
  "my_app": {
    "image_name": "ghcr.io/getdoover/my_app",
    "container_registry_profile_key": "registry-profile-uuid"
  }
}
```

Supported registries:
- GitHub Container Registry (ghcr.io)
- Docker Hub
- Private registries

### Build Configuration

Multi-platform builds for ARM and x86 devices:

```json
{
  "my_app": {
    "build_args": "--platform linux/amd64,linux/arm64"
  }
}
```

Common build arguments:
- `--platform linux/amd64` - Intel/AMD 64-bit
- `--platform linux/arm64` - ARM 64-bit (Raspberry Pi 4, etc.)
- `--platform linux/arm/v7` - ARM 32-bit (older Pi models)

### Publishing Workflow

```bash
# 1. Ensure you're authenticated
doover auth login

# 2. Build for target platforms
doover app build

# 3. Publish to platform
doover app publish

# For staging/testing
doover app publish --staging

# Production with specific profile
doover app publish --profile production
```

## Environment Variables

Applications receive configuration via environment variables:

### Standard Variables

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
        log.info(f"Config from {config_path}")
```

### Docker Compose Environment

For local development:

```yaml
services:
  my_app:
    build: ../
    network_mode: host
    environment:
      - APP_KEY=test_app_key
      - CONFIG_FP=/app/simulators/app_config.json
```

## Visibility and Access Control

### Application Visibility

| Visibility | Description |
|------------|-------------|
| `PUB` | Public - visible to all organizations |
| `PRI` | Private - only visible to owner organization |

### Setting Visibility

In `doover_config.json`:

```json
{
  "my_app": {
    "visibility": "PRI",
    "owner_org_key": "your-org-uuid"
  }
}
```

### Sharing Private Apps

Private apps can be shared with specific organizations through the Doover platform.

## Monitoring and Health

### Health Checks

Applications include health check endpoints:

```dockerfile
HEALTHCHECK --interval=30s --timeout=2s --start-period=5s \
    CMD curl -f "127.0.0.1:$HEALTHCHECK_PORT" || exit 1
```

### Application Status

Monitor application status via tags:

```python
class MyApplication(Application):
    async def main_loop(self):
        # Report health status
        await self.set_tag("health", {
            "status": "healthy",
            "uptime_seconds": self.uptime,
            "last_error": None,
            "version": "1.0.0"
        })
```

### Error Reporting

```python
class MyApplication(Application):
    async def main_loop(self):
        try:
            await self.do_work()
            await self.set_tag("status", "running")
            await self.set_tag("last_error", None)
        except Exception as e:
            log.error(f"Work failed: {e}")
            await self.set_tag("status", "error")
            await self.set_tag("last_error", str(e))
            await self.set_tag("last_error_time", datetime.now().isoformat())
```

## Configuration Profiles

### CLI Profiles

Use profiles for different environments:

```bash
# Default profile
doover app publish

# Staging profile
doover app publish --profile staging

# Production profile
doover app publish --profile production
```

### Profile Setup

Configure profiles in Doover CLI:

```bash
# Add a profile
doover config add-profile staging --api-url https://staging.doover.com

# List profiles
doover config list-profiles

# Switch default profile
doover config set-profile production
```

## Versioning

### Semantic Versioning

Use semantic versioning for apps:

```json
{
  "metadata": {
    "version": "1.2.3"
  }
}
```

- MAJOR: Breaking changes
- MINOR: New features, backwards compatible
- PATCH: Bug fixes

### Version in Code

```python
__version__ = "1.2.3"

class MyApplication(Application):
    async def setup(self):
        await self.set_tag("app_version", __version__)
```

## Debugging Production

### Remote Logs

Access logs from deployed devices:

```bash
# Via Docker on device
ssh device "docker logs my_app --tail 100"

# Follow logs
ssh device "docker logs my_app -f"
```

### Debug Mode

Enable debug mode via configuration:

```python
class MyApplication(Application):
    async def main_loop(self):
        if self.config.debug_mode.value:
            await self.debug_log("main_loop", {
                "temperature": self.temp,
                "state": self.state.state
            })

    async def debug_log(self, context: str, data: dict):
        await self.device_agent.publish_to_channel_async(
            "debug",
            json.dumps({
                "timestamp": datetime.now().isoformat(),
                "context": context,
                "data": data
            })
        )
```

### Channel Debugging

Use the channel viewer for real-time debugging:

```bash
doover app channels --host device-ip
```

## Common Administrative Tasks

### Updating an Application

1. Make code changes
2. Update version in `pyproject.toml`
3. Test locally: `doover app run`
4. Run tests: `doover app test`
5. Publish: `doover app publish`

### Rolling Back

Keep previous versions available:
- Don't overwrite tags in registry
- Use version-specific image tags
- Document breaking changes

### Scaling

For high-traffic applications:
- Use time-based throttling for channel publishing
- Batch updates when possible
- Monitor memory usage

### Security

- Never commit secrets to code
- Use configuration for API keys
- Validate all external input
- Log security events
