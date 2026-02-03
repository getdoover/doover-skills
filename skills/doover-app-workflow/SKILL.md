---
name: doover-app-workflow
description: End-to-end workflow for creating, developing, and publishing Doover apps
user_invocable: true
---

# Doover App Development Workflow

This skill guides you through the complete process of creating a new Doover application, from initial concept to published deployment.

## Project Creation: Use CLI or Template

When creating a new Doover app project, use one of these methods. Do NOT use mkdir or manually create files.

**For Device Apps - use the Doover CLI:**
```bash
doover app create --name my-app-name --description "My app description"
```

**For Processors/Integrations or any app - use the GitHub template:**
```bash
gh repo create my-org/my-app-name --template getdoover/app-template --private
gh repo clone my-org/my-app-name
cd my-app-name
# Then rename src/app_template to src/my_app_name
mv src/app_template src/my_app_name
```

## Workflow Overview

```
1. Research & Design
   └── Search existing apps for integration points
   └── Determine app type (device, processor, integration)
   └── Identify dependencies and patterns

2. Create Project
   └── Run: doover app create --name NAME --description "..."
   └── OR: gh repo create ORG/NAME --template getdoover/app-template
   └── Rename src/app_template to src/your_app_name

3. Configure Project
   └── Update doover_config.json with org key and registry
   └── Set up development environment

4. Implementation
   └── Define configuration schema
   └── Build UI components
   └── Implement application logic
   └── Add state management if needed

5. Testing
   └── Run local simulators
   └── Test with doover app run
   └── Validate configuration schema

6. Publishing
   └── Publish to Doover platform
   └── GitHub Actions builds images
   └── Ready for deployment
```

## Step 1: Research & Design

Search for existing apps and patterns to inform your design.

### Search the App Explorer

Check what already exists:

```bash
# Search via API
curl -s "https://api.doover.com/public/applications/?per_page=100" | \
  jq '.results[] | select(.description | test("YOUR_KEYWORD"; "i")) | {name, display_name, description}'
```

Or browse: https://admin.doover.com/app-explorer

**Note:** The public API only shows public and core apps. Your organization may have access to additional private apps.

### Questions to Answer

1. **Does a similar app exist?**
   - Can you use or extend an existing app?
   - What patterns do similar apps use?

2. **What type of app do you need?**

   | If you need... | Use a... |
   |----------------|----------|
   | Hardware access on device | Device App |
   | React to device data in cloud | Processor |
   | Receive external webhooks | Integration |
   | Push data to external service | Device App or Processor |

3. **What apps will you integrate with?**
   - Platform Interface (for GPIO)
   - Modbus Interface (for Modbus devices)
   - Device Agent (for channels/tags)
   - Other custom apps

4. **What channels/tags will you use?**
   - What data will you publish?
   - What data will you subscribe to?
   - What tags for state persistence?

### Example: Power BI Integration

For "push tag data to Power BI data streams":

1. **Search existing apps:**
   ```bash
   curl -s "https://api.doover.com/public/applications/?per_page=100" | \
     jq '.results[] | select(.description | test("power|bi|stream|push"; "i"))'
   ```

2. **App type decision:**
   - Need to read device tags → access device data
   - Push to external API → HTTP client needed
   - Could be a **Processor** (cloud-based, triggered by schedule or tag changes)
   - Or a **Device App** (if you need real-time local processing)

3. **Dependencies:**
   - Device Agent (for tag access)
   - No hardware access needed

4. **Data flow:**
   ```
   Device Tags → Processor (on schedule) → Power BI Streaming API
   ```

## Step 2: Create Project

Create the project using the Doover CLI or GitHub template. See "Project Creation" section above for commands.

**For Device Apps:**
```bash
doover app create --name my-app-name --description "My app description"
cd my-app-name
```

**For Processors/Integrations:**
```bash
gh repo create my-org/my-app-name --template getdoover/app-template --private
gh repo clone my-org/my-app-name
cd my-app-name
mv src/app_template src/my_app_name
```

**Add the cloud app build script (processors/integrations only):** When creating a processor or integration from the template, add a build script that produces the deployment package. Create a file (e.g. `build.sh`) in the project root with execute permission and the following contents:

```sh
#!/bin/sh

uv export --frozen --no-dev --no-editable --quiet -o requirements.txt

uv pip install \
   --no-deps \
   --no-installer-metadata \
   --no-compile-bytecode \
   --python-platform x86_64-manylinux2014 \
   --python 3.13 \
   --quiet \
   --target packages_export \
   --refresh \
   -r requirements.txt

rm -f package.zip

cd packages_export
zip -rq ../package.zip .
cd ..

zip -rq package.zip src

echo "OK"
```

Run this script to generate `package.zip` for publishing. See the `doover-cloud-apps` skill for more on cloud app structure and deployment.

## Step 3: Configure Project

After creating the project from the template, configure it for your organization.

### Configure Organization and Registry

You'll need:
- **Owner Organization Key** - Your organization's UUID
- **Container Registry Profile Key** - Credentials for pushing images

<!-- TODO: API/CLI commands for retrieving these will be added -->

**Placeholder - Retrieving Organization Key:**
```bash
# Coming soon: doover org list
# For now, contact your Doover administrator or check the admin panel
```

**Placeholder - Retrieving Registry Profile Key:**
```bash
# Coming soon: doover registry list
# For now, contact your Doover administrator or check the admin panel
```

### doover_config.json field reference

`doover_config.json` holds one entry per app (keyed by app name). Set these fields when configuring a new project:

| Field | Who sets it | Notes |
|-------|-------------|--------|
| **key** (app key) | **Do not set or edit manually.** Only `doover app publish` updates this. | For a new app, leave it absent or empty; the first `doover app publish` will create and set the app key correctly on the platform and in config. For existing apps, the key identifies the app on Doover—never replace it by hand. |
| **name** | You | Internal app name (e.g. `my_app_name`), must match the config key and package. |
| **display_name** | You | Human-readable name shown in the admin UI. |
| **type** | You | e.g. `DEV`. |
| **visibility** | You | e.g. `PRI` or `PUB`. |
| **owner_org_key** | You | Your organization's UUID. |
| **image_name** | You | Full image ref (e.g. `ghcr.io/your-org/my-app-name`). |
| **container_registry_profile_key** | You | Registry profile UUID for pushing images. |
| **build_args** | You (optional) | e.g. `--platform linux/amd64,linux/arm64`. |

**App key and id fields:** The **key** field is the app’s unique identifier on Doover. Do not generate or paste a UUID into it. Only run `doover app publish`; it will create the app (if new) and set or keep the app key correct in `doover_config.json`. Manually changing the app key can break linkage between your repo and the published app.

Update `doover_config.json` with the fields you control (omit or leave **key** unset for a new app):

```json
{
  "my_app_name": {
    "name": "my_app_name",
    "display_name": "My App Name",
    "type": "DEV",
    "visibility": "PRI",
    "owner_org_key": "YOUR_ORG_KEY",
    "image_name": "ghcr.io/your-org/my-app-name",
    "container_registry_profile_key": "YOUR_REGISTRY_PROFILE_KEY",
    "build_args": "--platform linux/amd64,linux/arm64"
  }
}
```

After the first successful `doover app publish`, the **key** field will be added or updated by the CLI; do not edit it afterward.

### Set Up Development Environment

```bash
# Install uv (if not already installed)
curl -LsSf https://astral.sh/uv/install.sh | sh

# Install dependencies
uv sync

# Verify setup
uv run pytest
```

### Configure CLI Profile

Set the default CLI profile for Doover commands:

```bash
# Use the dv2 profile (default for most users)
export DOOVER_PROFILE=dv2
```

Or specify per-command:

```bash
doover app publish --profile dv2
```

Add to your shell profile (`~/.bashrc`, `~/.zshrc`) to persist:

```bash
echo 'export DOOVER_PROFILE=dv2' >> ~/.zshrc
source ~/.zshrc
```

## Step 4: Implementation

### Define Configuration Schema

Edit `src/my_app/app_config.py`:

```python
from pathlib import Path
from pydoover import config

class MyAppConfig(config.Schema):
    def __init__(self):
        # Required settings
        self.api_endpoint = config.String(
            "API Endpoint",
            description="Power BI streaming dataset URL",
        )

        self.api_key = config.String(
            "API Key",
            description="Authentication key (if required)",
            default="",
        )

        # Optional settings with defaults
        self.push_interval_seconds = config.Integer(
            "Push Interval (seconds)",
            description="How often to push data",
            default=60,
            minimum=10,
            maximum=3600,
        )

        self.tags_to_push = config.Array(
            "Tags to Push",
            element=config.String("Tag Name"),
            description="List of tag names to send to Power BI",
        )

def export():
    MyAppConfig().export(
        Path(__file__).parents[2] / "doover_config.json",
        "my_app_name"
    )
```

Regenerate the config schema:
```bash
uv run export-config
```

### Build UI Components (Device Apps)

Edit `src/my_app/app_ui.py`:

```python
from pydoover import ui

class MyAppUI:
    def __init__(self):
        self.last_push = ui.DateTimeVariable(
            "last_push",
            "Last Push"
        )

        self.push_count = ui.NumericVariable(
            "push_count",
            "Total Pushes",
            precision=0
        )

        self.status = ui.TextVariable(
            "status",
            "Status"
        )

        self.push_now = ui.Action(
            "push_now",
            "Push Now",
            colour=ui.Colour.blue
        )

    def fetch(self):
        return (
            self.last_push,
            self.push_count,
            self.status,
            self.push_now,
        )

    def update(self, last_push, count, status):
        self.last_push.update(last_push)
        self.push_count.update(count)
        self.status.update(status)
```

### Implement Application Logic

Edit `src/my_app/application.py`:

```python
import json
import logging
import time
from datetime import datetime

import requests
from pydoover.docker import Application
from pydoover import ui

from .app_config import MyAppConfig
from .app_ui import MyAppUI

log = logging.getLogger(__name__)

class MyApp(Application):
    config: MyAppConfig
    loop_target_period = 10  # Check every 10 seconds

    def __init__(self, *args, **kwargs):
        super().__init__(*args, **kwargs)
        self.ui = None
        self.push_count = 0
        self.last_push_time = 0

    async def setup(self):
        self.ui = MyAppUI()
        self.ui_manager.add_children(*self.ui.fetch())

        # Load persisted state
        self.push_count = self.get_tag("push_count", default=0)

        log.info("App initialized")

    async def main_loop(self):
        now = time.time()
        interval = self.config.push_interval_seconds.value

        # Check if it's time to push
        if now - self.last_push_time >= interval:
            await self.push_data()

        # Update UI
        self.ui.update(
            datetime.fromtimestamp(self.last_push_time) if self.last_push_time else None,
            self.push_count,
            "Running"
        )

    @ui.callback("push_now")
    async def on_push_now(self, value):
        await self.push_data()
        self.ui.push_now.coerce(None)

    async def push_data(self):
        """Collect tags and push to Power BI."""
        try:
            # Collect tag values
            data = {}
            for tag_config in self.config.tags_to_push.elements:
                tag_name = tag_config.value
                value = self.get_tag(tag_name)
                if value is not None:
                    data[tag_name] = value

            # Add timestamp
            data["timestamp"] = datetime.now().isoformat()

            # Push to Power BI
            response = requests.post(
                self.config.api_endpoint.value,
                json=[data],  # Power BI expects array
                headers={"Content-Type": "application/json"},
                timeout=30
            )
            response.raise_for_status()

            # Update state
            self.push_count += 1
            self.last_push_time = time.time()
            await self.set_tag("push_count", self.push_count)

            log.info(f"Pushed data: {data}")

        except Exception as e:
            log.error(f"Push failed: {e}")
            self.ui.status.update(f"Error: {e}")
```

### Update Entry Point

Edit `src/my_app/__init__.py`:

```python
from pydoover.docker import run_app
from .application import MyApp
from .app_config import MyAppConfig

def main():
    run_app(MyApp(config=MyAppConfig()))
```

### Update pyproject.toml

```toml
[project]
name = "my-app-name"
version = "0.1.0"
requires-python = ">=3.11"
dependencies = [
    "pydoover>=0.4.13",
    "requests>=2.32.0",
]

[project.scripts]
doover-app-run = "my_app_name:main"
export-config = "my_app_name.app_config:export"
```

## Step 5: Testing

### Create Simulator

Edit `simulators/sample/main.py` to generate test data:

```python
import random
from pydoover.docker import Application, run_app
from pydoover import config

class TestSimulator(Application):
    async def setup(self):
        pass

    async def main_loop(self):
        # Simulate tag data
        await self.set_tag("temperature", random.uniform(20, 30))
        await self.set_tag("humidity", random.uniform(40, 60))
        await self.set_tag("pressure", random.uniform(1000, 1020))

def main():
    run_app(TestSimulator(config=config.Schema()))

if __name__ == "__main__":
    main()
```

### Configure Test Environment

Edit `simulators/app_config.json`:

```json
{
  "api_endpoint": "https://api.powerbi.com/beta/YOUR_WORKSPACE/datasets/YOUR_DATASET/rows?key=YOUR_KEY",
  "api_key": "",
  "push_interval_seconds": 30,
  "tags_to_push": ["temperature", "humidity", "pressure"]
}
```

### Run Locally

```bash
# Start the simulator and app
doover app run

# In another terminal, view channel data
doover app channels

# Run tests
doover app test

# Check code quality
doover app lint --fix
doover app format --fix
```

### Verify Behavior

1. Check simulator is generating tag data
2. Verify app is reading tags correctly
3. Confirm data is being pushed (check logs or Power BI)
4. Test the "Push Now" button
5. Verify UI updates correctly

## Step 6: Publishing

### Pre-Publish Checklist

- [ ] All tests pass: `doover app test`
- [ ] Linting clean: `doover app lint`
- [ ] Config schema exported: `uv run export-config`
- [ ] `doover_config.json` has correct org key and registry profile
- [ ] README.md updated with usage instructions
- [ ] Version number set in pyproject.toml

### Commit and Push

```bash
git add .
git commit -m "Initial implementation of Power BI integration"
git push origin main
```

### Publish to Doover

```bash
# Publish using default profile (dv2)
doover app publish

# Explicitly specify profile
doover app publish --profile dv2

# Or publish to staging first
doover app publish --staging

# Skip container build (config only)
doover app publish --skip-container
```

### GitHub Actions

The template includes GitHub Actions workflows that:
1. Build Docker images on push/release
2. Push to configured container registry
3. Support multi-platform builds (amd64, arm64)

Check `.github/workflows/` for the workflow configuration.

### Verify Deployment

1. Check the Doover admin panel for your app
2. Verify the app appears in your organization's app library
3. Install on a test device
4. Monitor logs and behavior

## Complete Example: Power BI Streaming App

Here's a complete example based on the workflow above:

### Research Results

- No existing Power BI app found
- Will create a Device App (for real-time local access to tags)
- Depends on: Device Agent (for tag access)
- Data flow: Local tags → App → Power BI Streaming API

### Project Creation

```bash
# Create from GitHub template
gh repo create my-org/powerbi-streamer --template getdoover/app-template --private
gh repo clone my-org/powerbi-streamer
cd powerbi-streamer

# Rename from template
mv src/app_template src/powerbi_streamer

# Update pyproject.toml, doover_config.json, and imports
```

### Final File Structure

```
powerbi-streamer/
├── src/powerbi_streamer/
│   ├── __init__.py
│   ├── application.py
│   ├── app_config.py
│   └── app_ui.py
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

### Configuration Schema

Users configure:
- Power BI streaming endpoint URL
- Push interval
- Which tags to include

### Behavior

1. App starts and loads config
2. Every N seconds (configurable):
   - Reads specified tags
   - Formats as JSON
   - POSTs to Power BI streaming endpoint
3. UI shows last push time, count, status
4. Manual "Push Now" button for testing

## Troubleshooting

### Build Failures

```bash
# Check Docker build locally
docker build -t my-app .

# Rebuild without cache
doover app build -- --no-cache
```

### Publish Failures

```bash
# Verify authentication
doover auth status

# Check registry access
docker login ghcr.io

# Verify org key and registry profile
cat doover_config.json | jq '.my_app.owner_org_key'
```

### Runtime Errors

```bash
# View logs locally
doover app run

# View container logs
docker logs my-app

# Check channel data
doover app channels
```

## Next Steps

After publishing:

1. **Install on devices** via Doover admin panel
2. **Monitor** via device dashboards
3. **Iterate** - make changes, test, republish
4. **Share** - set visibility to `PUB` if making public

For cloud-based alternatives (processors/integrations), see the `doover-cloud-apps` skill.
