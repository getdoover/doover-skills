# Phase 4p: Processor Build

Generate the processor application code based on the build plan.

## Phase Objective

Execute the build plan from PLAN.md to generate working processor code. The plan contains all decisions already made - follow it exactly without deviation.

## Execution Context

This phase runs as a subagent spawned by the orchestrator (SKILL.md). You have full access to:
- `Read` tool for reading documentation and existing files
- `Write` / `Edit` tools for creating/modifying code
- `Bash` tool for running commands

**Note:** Do NOT use `AskUserQuestion` - all questions should have been resolved in Phase 3 (Plan). If the plan is unclear, report this as an error.

Your job: Generate the processor code following PLAN.md, then return a summary.

## Prerequisites

- Phase 3 must be completed (PLAN.md exists with implementation details)
- `.appgen/PHASE.md` shows Phase 3 completed
- App type must be "processor"

## Steps

### Step 1: Read State and Plan

1. Read `{app-directory}/.appgen/PHASE.md` to get:
   - App name
   - App directory path

2. Read `{app-directory}/.appgen/PLAN.md` to get:
   - Configuration schema design
   - Event handlers to implement
   - Subscription/schedule configuration
   - Tags to create
   - External integration details
   - Implementation notes

### Step 2: Load Documentation Chunks

1. Read the documentation index:
   `{platform-docs-path}/references/index.md`

2. Read the chunks listed in PLAN.md's "Documentation Chunks" section:
   - Required chunks (always read these)
   - Recommended chunks (read if listed)

3. **Keyword-based discovery**: Scan PLAN.md content for keywords from the index. If keywords match a chunk not already listed, read that chunk too. For example:
   - If PLAN.md mentions "subscription", "schedule" → read `processor-features.md`
   - If PLAN.md mentions "ui_manager", "push_async" → read `processor-features.md` and `docker-ui.md`
   - If PLAN.md mentions "connection", "ping_connection" → read `processor-features.md`

4. Common chunks for Processor apps:
   - `{platform-docs-path}/references/config-schema.md` - Configuration patterns
   - `{platform-docs-path}/references/cloud-handler.md` - Handler and event patterns
   - `{platform-docs-path}/references/cloud-project.md` - Project setup and build script
   - `{platform-docs-path}/references/processor-features.md` - ManySubscriptionConfig, ScheduleConfig, UI management
   - `{platform-docs-path}/references/docker-ui.md` - UI components (if has_ui)
   - `{platform-docs-path}/references/tags-channels.md` - Tags and channels

Use these for code patterns and syntax. The PLAN.md tells you *what* to build; the platform docs tell you *how* to build it.

### Step 3: Generate Application Code

**Import Guidelines:**
- Prefer using `pydoover` modules wherever possible
- Import from `pydoover.cloud.processor` for processor-specific classes
- Avoid adding external dependencies unless specified in PLAN.md
- Standard library imports (json, datetime, logging, asyncio, etc.) are fine
- If external packages are listed in PLAN.md, note them for Step 3b

Create/modify the following files in the app directory:

1. **`src/{app_name}/__init__.py`** - Lambda handler entry point
   - Define `handler(event, context)` function
   - Import and run the application with `run_app()`

2. **`src/{app_name}/application.py`** - Core Application class
   - Inherit from `pydoover.cloud.processor.Application`
   - Implement `setup()` and `close()`
   - Implement event handlers as specified in PLAN.md
   - Create tags as specified in PLAN.md

3. **`src/{app_name}/app_config.py`** - Configuration schema
   - Define fields exactly as specified in PLAN.md Configuration Schema
   - Include `ManySubscriptionConfig` as specified
   - Include `ScheduleConfig` if specified in PLAN.md
   - Include `export()` function

4. **`src/{app_name}/app_ui.py`** - UI components (if specified in PLAN.md)
   - Create Variables as specified
   - Only needed if the processor updates device UI

### Step 3b: Add External Dependencies (if needed)

If PLAN.md lists external packages in Implementation Notes, add them:

```bash
cd {app-directory}
uv add {package-name}
```

Skip this step if no external packages are listed.

### Step 4: Update doover_config.json

Ensure `doover_config.json` has the correct processor configuration:
- `type` should be `"PRO"`
- `handler` should point to the Lambda handler (e.g., `"src.my_processor.handler"`)
- Include `lambda_config` with Runtime, Timeout, MemorySize, Handler

Run the config export to update the config schema:

```bash
cd {app-directory}
uv run export-config
```

### Step 5: Create Build Script

Create `build.sh` for packaging the processor for deployment:

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

### Step 6: Remove Unnecessary Files

If the project was created from the device app template, remove Docker-specific files:
- Remove `Dockerfile` (processors deploy as zip packages, not containers)
- Remove `.github/workflows/build-image.yml` if present
- Remove `simulators/` directory if present

## State Updates

After successful code generation, update `.appgen/PHASE.md`:

- Set current phase to "Phase 4 - Processor Build"
- Set status to "completed"
- Add Phase 4 to completed phases list
- Add notes about what was generated

## Completion Criteria

Phase 4 is complete when:
- [ ] Lambda handler exists in `__init__.py`
- [ ] Application class exists with event handlers per PLAN.md
- [ ] Config schema matches PLAN.md specification
- [ ] `doover_config.json` has `type: "PRO"` and `lambda_config`
- [ ] `build.sh` script is created
- [ ] External dependencies added (if any)
- [ ] `.appgen/PHASE.md` updated with status "completed"

## Summary for Orchestrator

When returning to the orchestrator, include:
- **Status**: success or failure
- **Files created/modified**: List of files touched
- **What the processor does**: Brief description of generated functionality
- **Event handlers implemented**: Which events the processor responds to
- **Dependencies added**: List of external packages (if any)
- **Errors**: Any issues encountered
