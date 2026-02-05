# Phase 4d: Docker Build

Generate the Docker device application code based on the build plan.

## Phase Objective

Execute the build plan from PLAN.md to generate working Docker application code. The plan contains all decisions already made - follow it exactly without deviation.

## Execution Context

This phase runs as a subagent spawned by the orchestrator (SKILL.md). You have full access to:
- `Read` tool for reading documentation and existing files
- `Write` / `Edit` tools for creating/modifying code
- `Bash` tool for running commands

**Note:** Do NOT use `AskUserQuestion` - all questions should have been resolved in Phase 3 (Plan). If the plan is unclear, report this as an error.

Your job: Generate the app code following PLAN.md, then return a summary.

## Prerequisites

- Phase 3 must be completed (PLAN.md exists with implementation details)
- `.appgen/PHASE.md` shows Phase 3 completed

## Steps

### Step 1: Read State and Plan

1. Read `{app-directory}/.appgen/PHASE.md` to get:
   - App name
   - App directory path

2. Read `{app-directory}/.appgen/PLAN.md` to get:
   - Configuration schema design
   - UI elements to create
   - External integration details
   - Implementation notes

### Step 2: Load Documentation Chunks

1. Read the documentation index:
   `references/mini-docs/index.md`

2. Read the chunks listed in PLAN.md's "Documentation Chunks" section:
   - Required chunks (always read these)
   - Recommended chunks (read if listed)

3. **Keyword-based discovery**: Scan PLAN.md content for keywords from the index. If keywords match a chunk not already listed, read that chunk too. For example:
   - If PLAN.md mentions "battery", "voltage", "threshold" → read `docker-ui.md` for range coloring
   - If PLAN.md mentions "state machine", "transition" → read `docker-advanced.md`
   - If PLAN.md mentions "hardware", "gpio", "debounce" → read `docker-advanced.md`
   - If PLAN.md mentions "warning", "alert" → read `docker-ui.md`

4. Common chunks for Docker apps:
   - `references/mini-docs/config-schema.md` - Configuration patterns
   - `references/mini-docs/docker-application.md` - Application class structure
   - `references/mini-docs/docker-project.md` - Entry point and Dockerfile
   - `references/mini-docs/docker-ui.md` - UI components (if has_ui)
   - `references/mini-docs/docker-advanced.md` - State machines, workers, hardware I/O
   - `references/mini-docs/tags-channels.md` - Tags and channels

Use these for code patterns and syntax. The PLAN.md tells you *what* to build; the mini-docs tell you *how* to build it.

### Step 3: Generate Application Code

**Import Guidelines:**
- Prefer using `pydoover` modules wherever possible
- Avoid adding external dependencies unless specified in PLAN.md
- Standard library imports (json, datetime, logging, asyncio, etc.) are fine
- If external packages are listed in PLAN.md, note them for Step 3b

Create/modify the following files in the app directory:

1. **`src/{app_name}/application.py`** - Core Application class
   - Implement `setup()` and `main_loop()`
   - Add UI callbacks as specified in PLAN.md
   - Implement integration logic from PLAN.md

2. **`src/{app_name}/app_config.py`** - Configuration schema
   - Define fields exactly as specified in PLAN.md Configuration Schema
   - Include `export()` function

3. **`src/{app_name}/app_ui.py`** - UI components
   - Create Variables as specified in PLAN.md
   - Create Parameters as specified in PLAN.md
   - Create Actions as specified in PLAN.md

4. **`src/{app_name}/__init__.py`** - Entry point
   - Import and run the application

### Step 3b: Add External Dependencies (if needed)

If PLAN.md lists external packages in Implementation Notes, add them:

```bash
cd {app-directory}
uv add {package-name}
```

Skip this step if no external packages are listed.

### Step 4: Update doover_config.json

Run the config export to update `doover_config.json`:

```bash
cd {app-directory}
uv run export-config
```

Or manually update if the export script isn't configured yet.

## State Updates

After successful code generation, update `.appgen/PHASE.md`:

- Set current phase to "Phase 4 - Docker Build"
- Set status to "completed"
- Add Phase 4 to completed phases list
- Add notes about what was generated

## Completion Criteria

Phase 4 is complete when:
- [ ] Application class exists with `setup()` and `main_loop()`
- [ ] Config schema matches PLAN.md specification
- [ ] UI components match PLAN.md specification
- [ ] Entry point is configured
- [ ] External dependencies added (if any)
- [ ] `doover_config.json` updated
- [ ] `.appgen/PHASE.md` updated with status "completed"

## Summary for Orchestrator

When returning to the orchestrator, include:
- **Status**: success or failure
- **Files created/modified**: List of files touched
- **What the app does**: Brief description of generated functionality
- **Dependencies added**: List of external packages (if any)
- **Errors**: Any issues encountered
