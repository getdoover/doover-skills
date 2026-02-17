# Phase 2w: Widget Config (Processor Side)

Configure the processor side of the widget application. The widget JS scaffolding is handled separately by the add-widget skill (invoked by the orchestrator immediately after this phase).

## Phase Objective

Set up the Python processor that registers and serves the widget: replace the app-template's default Python code with the widget processor pattern (RemoteComponent + ui_state push), create pyproject.toml and build.sh, rewrite doover_config.json with the processor base structure, and resolve Python dependencies.

The JavaScript widget side (clone template, rename, move folder, update config with file_deployments/deployment_channel_messages, npm install) is **not** done here — the add-widget skill handles all of that.

## Execution Context

This phase runs as a subagent spawned by the orchestrator (SKILL.md). You have full access to:
- `Read` tool for reading state and existing files
- `Write` / `Edit` tools for modifying files
- `Bash` tool for running commands
- `AskUserQuestion` tool for prompting user if image URLs are invalid

Your job: Set up the processor side, then return a summary.

## Prerequisites

- Phase 1 must be completed (app directory exists with `.appgen/PHASE.md`)
- App type must be "widget"

## Steps

### Step 1: Read State from Phase 1

Read `{app-directory}/.appgen/PHASE.md` to get:
- App name (snake_case, e.g., `my_widget_app`)
- App description
- `icon_url`
- GitHub repo path (e.g., `org/repo-name`)

Derive name variants:
- **snake_case**: app name as-is (e.g., `my_widget_app`)
- **PascalCase**: capitalize each word, no separators (e.g., `MyWidgetApp`)
- **kebab-case**: lowercase with hyphens (e.g., `my-widget-app`)
- **Title Case**: capitalize each word with spaces (e.g., `My Widget App`)

### Step 2: Write Python Processor Files

Replace the app-template's default Python source with the widget processor pattern.

**Remove existing source:**
```bash
rm -rf {app-dir}/src/{app_name}/
mkdir -p {app-dir}/src/{app_name}/
```

**Create `src/{app_name}/__init__.py`:**
```python
from pydoover.cloud.processor import run_app

from .application import {PascalCase}App
from .app_config import {PascalCase}Config


def handler(event, context):
    """Lambda handler entry point."""
    {PascalCase}Config.clear_elements()
    return run_app(
        {PascalCase}App(config={PascalCase}Config()),
        event,
        context,
    )
```

**Create `src/{app_name}/application.py`:**
```python
import logging

from pydoover.cloud.processor import Application
from pydoover.cloud.processor.types import AggregateUpdateEvent
from pydoover.ui import RemoteComponent

from .app_config import {PascalCase}Config

log = logging.getLogger(__name__)

WIDGET_NAME = "{PascalCase}"
FILE_CHANNEL = "{snake_case}"


class {PascalCase}App(Application):
    """
    {Title Case} Application.

    On deployment, the deployment_config aggregate is updated, which
    triggers on_aggregate_update via our subscription. We then push
    ui_state so the widget appears in the UI interpreter.
    """

    config: {PascalCase}Config

    async def setup(self):
        """Called once before processing any event."""
        self.ui_manager.set_children([
            RemoteComponent(
                name=WIDGET_NAME,
                display_name=WIDGET_NAME,
                component_url=FILE_CHANNEL,
            ),
        ])

    async def on_aggregate_update(self, event: AggregateUpdateEvent):
        """Triggered when deployment_config aggregate is updated (i.e. on deployment)."""
        log.info(f"Aggregate update received for agent {self.agent_id}")
        await self.ui_manager.push_async(even_if_empty=True)

        # Patch defaultOpen onto our application so the widget is
        # expanded on page load instead of collapsed.
        await self.api.update_aggregate(
            self.agent_id,
            "ui_state",
            {"state": {"children": {self.app_key: {"defaultOpen": True}}}},
        )
        log.info(f"Pushed ui_state with {WIDGET_NAME} widget entry")
```

**Create `src/{app_name}/app_config.py`:**
```python
from pathlib import Path

from pydoover import config
from pydoover.cloud.processor import SubscriptionConfig


class {PascalCase}Config(config.Schema):
    def __init__(self):
        self.subscription = SubscriptionConfig(default="deployment_config")


def export():
    {PascalCase}Config().export(
        Path(__file__).parents[2] / "doover_config.json",
        "{app_name}",
    )


if __name__ == "__main__":
    export()
```

Replace all `{PascalCase}`, `{snake_case}`, `{Title Case}`, and `{app_name}` placeholders with the actual derived values.

### Step 3: Create pyproject.toml

Write `{app-dir}/pyproject.toml`:

```toml
[project]
name = "{kebab-case}"
version = "0.1.0"
description = "{app description from PHASE.md}"
readme = "README.md"
requires-python = ">=3.11"
dependencies = [
    "pydoover>=0.4.18",
]

[project.scripts]
export-config = "{app_name}.app_config:export"

[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"

[tool.uv.sources]
pydoover = { git = "https://github.com/getdoover/pydoover", branch = "doover-2" }
```

### Step 4: Create build.sh

Write `{app-dir}/build.sh` and make it executable:

```bash
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
mkdir -p packages_export
cd packages_export
zip -rq ../package.zip .
cd ..

zip -rq package.zip src

echo "OK"
```

```bash
chmod +x {app-dir}/build.sh
```

### Step 5: Create README.md placeholder

Write a minimal `{app-dir}/README.md` (required for `uv sync` to succeed):

```markdown
# {Title Case}

{app description from PHASE.md}
```

### Step 6: Rewrite doover_config.json (Processor Base)

Rewrite `{app-dir}/doover_config.json` with the processor base structure. Do **NOT** include `file_deployments` or `deployment_channel_messages` — the add-widget skill will add those.

```json
{
    "{app_name}": {
        "name": "{app_name}",
        "display_name": "{Title Case}",
        "type": "PRO",
        "visibility": "PUB",
        "allow_many": true,
        "description": "{app description from PHASE.md}",
        "long_description": "README.md",
        "depends_on": [],
        "handler": "src.{app_name}.handler",
        "lambda_config": {
            "Runtime": "python3.13",
            "Timeout": 300,
            "MemorySize": 128,
            "Handler": "src.{app_name}.handler"
        },
        "organisation_id": null,
        "key": null,
        "owner_org_id": null,
        "code_repo_id": null,
        "container_registry_profile_id": null,
        "repo_branch": "main",
        "image_name": null,
        "icon_url": "{validated icon_url}",
        "staging_config": {},
        "export_config_command": null,
        "run_command": null,
        "lambda_arn": null,
        "config_schema": {},
        "banner_url": null
    }
}
```

### Step 7: Validate Icon URL

Same validation process as Phase 2p Step 4:

1. Make an HTTP HEAD or GET request to the icon URL
2. Check that the response status is 200 OK
3. If the image is too large (>100KB), download, resize to 256x256, save to `{app-dir}/assets/icon.png`, and update `icon_url` to the GitHub raw URL
4. If validation fails, use `AskUserQuestion` to prompt for a replacement URL

Apply the validated URL to `doover_config.json`.

### Step 8: Remove Build-Image Workflow

Widget apps are deployed as zip packages (like processors), not container images:

1. Delete `.github/workflows/build-image.yml` if it exists
2. Remove `Dockerfile` if present

### Step 9: Install Python Dependencies

```bash
cd {app-dir} && uv sync
```

### Step 10: Export Config

Run the config export to populate `config_schema` in doover_config.json:

```bash
cd {app-dir} && uv run export-config
```

### Step 11: Update State

Update `.appgen/PHASE.md`:
- Set current phase to "Phase 2 - Widget Config"
- Set status to "completed"
- Add Phase 2 to completed phases list
- Record derived name variants (PascalCase, kebab-case, snake_case) for use by add-widget
- Note that processor side is configured; widget scaffolding pending (add-widget)

## Completion Criteria

Phase 2 is complete when:
- [ ] Python source `src/{app_name}/` has __init__.py, application.py, app_config.py
- [ ] pyproject.toml exists with correct project name and export-config script
- [ ] build.sh exists and is executable
- [ ] doover_config.json has processor base structure (app key, lambda_config, icon_url)
- [ ] doover_config.json does NOT yet have file_deployments or deployment_channel_messages (add-widget adds those)
- [ ] Icon URL validated
- [ ] Python dependencies resolved (uv sync)
- [ ] Config schema exported (uv run export-config)
- [ ] `.github/workflows/build-image.yml` removed (if present)
- [ ] `.appgen/PHASE.md` updated with status "completed"

## Summary for Orchestrator

When returning to the orchestrator, include:
- **Status**: success or failure
- **Processor configured**: Python module name and class name
- **Config exported**: Whether config schema was populated
- **Name variants**: PascalCase, kebab-case, snake_case (needed by add-widget)
- **Files created/modified**: List of files touched
- **Errors**: Any issues encountered
- **Next**: Orchestrator should invoke add-widget skill to scaffold the JS widget
