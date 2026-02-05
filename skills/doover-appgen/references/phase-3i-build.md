# Phase 3i: Integration Build

Generate the integration application code based on the app description.

## Phase Objective

Use the Doover cloud apps development guide to generate working integration code that implements the functionality described in Phase 1.

## Execution Context

This phase runs as a subagent spawned by the orchestrator (SKILL.md). You have full access to:
- `Read` tool for reading documentation and existing files
- `Write` / `Edit` tools for creating/modifying code
- `Bash` tool for running commands
- `AskUserQuestion` tool if clarification is needed

Your job: Generate the integration code, then return a summary.

## Prerequisites

- Phase 1 must be completed (app directory exists with `.appgen/PHASE.md`)
- App type must be "integration"

## Steps

### Step 1: Read State from Phase 1

Read `{app-directory}/.appgen/PHASE.md` to get:
- App name
- App description
- App directory path

### Step 2: Load Development Guide

Read the Doover cloud apps development guide (bundled with this skill):
`references/mini-docs-integration.md`

This contains:
- Project structure for integrations
- Handler entry point pattern
- Ingestion endpoint handling (`on_ingestion_endpoint`)
- Configuration schema with ingestion and permissions
- State management via tags
- Integration + Processor pattern

### Step 3: Understand What to Build

Based on the app name and description, determine:
- What external system will send data to this integration?
- What format will the incoming data be in (JSON, protobuf, etc.)?
- How should the data be routed to devices?
- What security/validation is needed (IP whitelist, HMAC signing)?
- What configuration options should be exposed?

If the description is too vague, use `AskUserQuestion` to clarify.

### Step 4: Generate Application Code

**Import Guidelines:**
- Prefer using `pydoover` modules wherever possible
- Import from `pydoover.cloud.processor` for integration-specific classes
- Avoid adding external dependencies unless truly necessary
- Standard library imports (json, datetime, logging, asyncio, base64, etc.) are fine
- If external packages ARE required (e.g., payload parsing libraries), note them for Step 4b

Create/modify the following files in the app directory:

1. **`src/{app_name}/__init__.py`** - Lambda handler entry point
   - Define `handler(event, context)` function
   - Import and run the application with `run_app()`

2. **`src/{app_name}/application.py`** - Core Application class
   - Inherit from `pydoover.cloud.processor.Application`
   - Implement `setup()` and `close()`
   - Implement `on_ingestion_endpoint()` for HTTP data reception
   - Optionally implement `parse_ingestion_event_payload()` for custom parsing
   - Optionally implement `on_schedule()` for periodic tasks

3. **`src/{app_name}/app_config.py`** - Configuration schema
   - Define user-configurable parameters
   - Include `IngestionEndpointConfig` for HTTP endpoint settings
   - Include `ExtendedPermissionsConfig` for multi-device access
   - Include `export()` function

### Step 4b: Add External Dependencies (if needed)

If any external packages were imported in Step 4, add them to the project:

```bash
cd {app-directory}
uv add {package-name}
```

For example, if the integration needs protobuf parsing:
```bash
uv add protobuf
```

Skip this step if only pydoover and standard library imports were used.

### Step 5: Update doover_config.json

Ensure `doover_config.json` has the correct integration configuration:
- `type` should be `"INT"`
- `handler` should point to the Lambda handler (e.g., `"src.my_integration.handler"`)
- Include `lambda_config` with Runtime, Timeout, MemorySize, Handler

Run the config export to update the config schema:

```bash
cd {app-directory}
uv run export-config
```

Or manually update if the export script isn't configured yet.

### Step 6: Create Build Script

Create `build.sh` for packaging the integration for deployment:

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

Make it executable:
```bash
chmod +x {app-directory}/build.sh
```

Add build outputs to `.gitignore`:
```
packages_export/
package.zip
```

### Step 7: Remove Unnecessary Files

If the project was created from the device app template, remove Docker-specific files:
- Remove `Dockerfile` (integrations deploy as zip packages, not containers)
- Remove `.github/workflows/build-image.yml` if present
- Remove `simulators/` directory if present

### Step 8: Verify Build

Run basic checks:

```bash
cd {app-directory}
uv sync
uv run python -c "from {app_name} import handler; print('Import OK')"
```

## State Updates

After successful code generation, update `.appgen/PHASE.md`:

- Set current phase to "Phase 3 - Integration Build"
- Set status to "completed"
- Add Phase 3 to completed phases list
- Add notes about what was generated

## Completion Criteria

Phase 3 is complete when:
- [ ] Lambda handler exists in `__init__.py`
- [ ] Application class exists with `on_ingestion_endpoint` handler
- [ ] Config schema is defined with ingestion/permissions config
- [ ] `doover_config.json` has `type: "INT"` and `lambda_config`
- [ ] `build.sh` script is created
- [ ] Code imports successfully
- [ ] `.appgen/PHASE.md` updated with status "completed"

## Summary for Orchestrator

When returning to the orchestrator, include:
- **Status**: success or failure
- **Files created/modified**: List of files touched
- **What the integration does**: Brief description of generated functionality
- **Expected payload format**: What data format the integration expects
- **Known limitations**: What doesn't work yet or needs manual work
- **Errors**: Any issues encountered
