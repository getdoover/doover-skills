# Phase 3d: Docker Build

Generate the Docker device application code based on the app description.

## Phase Objective

Use the Doover device app development guide to generate working Docker application code that implements the functionality described in Phase 1.

## Execution Context

This phase runs as a subagent spawned by the orchestrator (SKILL.md). You have full access to:
- `Read` tool for reading documentation and existing files
- `Write` / `Edit` tools for creating/modifying code
- `Bash` tool for running commands
- `AskUserQuestion` tool if clarification is needed

Your job: Generate the app code, then return a summary.

## Prerequisites

- Phase 1 must be completed (app directory exists with `.appgen/PHASE.md`)

## Steps

### Step 1: Read State from Phase 1

Read `{app-directory}/.appgen/PHASE.md` to get:
- App name
- App description
- App directory path

### Step 2: Load Development Guide

Read the Doover device app development guide (bundled with this skill):
`references/mini-docs-docker.md`

This contains:
- Project structure
- Application class patterns
- Configuration schema
- UI components
- Best practices

### Step 3: Understand What to Build

Based on the app name and description, determine:
- What external service/API does this integrate with (if any)?
- What data should this app read/write?
- What UI elements does the user need?
- What configuration options should be exposed?

If the description is too vague, use `AskUserQuestion` to clarify.

### Step 4: Generate Application Code

**Import Guidelines:**
- Prefer using `pydoover` modules wherever possible
- Avoid adding external dependencies unless truly necessary for the integration
- Standard library imports (json, datetime, logging, asyncio, etc.) are fine
- If external packages ARE required (e.g., API client libraries), note them for Step 4b

Create/modify the following files in the app directory:

1. **`src/{app_name}/application.py`** - Core Application class
   - Implement `setup()` and `main_loop()`
   - Add UI callbacks as needed

2. **`src/{app_name}/app_config.py`** - Configuration schema
   - Define user-configurable parameters
   - Include `export()` function

3. **`src/{app_name}/app_ui.py`** - UI components
   - Variables for displaying state
   - Parameters for user input
   - Actions for user commands

4. **`src/{app_name}/__init__.py`** - Entry point
   - Import and run the application

### Step 4b: Add External Dependencies (if needed)

If any external packages were imported in Step 4, add them to the project:

```bash
cd {app-directory}
uv add {package-name}
```

For example, if the app needs an API client:
```bash
uv add httpx  # or requests, or specific SDK
```

Skip this step if only pydoover and standard library imports were used.

### Step 5: Update doover_config.json

Run the config export to update `doover_config.json`:

```bash
cd {app-directory}
uv run export-config
```

Or manually update if the export script isn't configured yet.

### Step 6: Verify Build

Run basic checks:

```bash
cd {app-directory}
uv sync
uv run python -c "from {app_name}.application import *; print('Import OK')"
```

## State Updates

After successful code generation, update `.appgen/PHASE.md`:

- Set current phase to "Phase 3 - Docker Build"
- Set status to "completed"
- Add Phase 3 to completed phases list
- Add notes about what was generated

## Completion Criteria

Phase 3 is complete when:
- [ ] Application class exists with `setup()` and `main_loop()`
- [ ] Config schema is defined
- [ ] UI components are defined
- [ ] Entry point is configured
- [ ] Code imports successfully
- [ ] `.appgen/PHASE.md` updated with status "completed"

## Summary for Orchestrator

When returning to the orchestrator, include:
- **Status**: success or failure
- **Files created/modified**: List of files touched
- **What the app does**: Brief description of generated functionality
- **Known limitations**: What doesn't work yet or needs manual work
- **Errors**: Any issues encountered
