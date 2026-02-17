# Phase 3w: Widget Build

Minimal phase — the widget is already scaffolded from the template.

## Phase Objective

Acknowledge that the widget app structure is set up and ready for customization. The widget JS component and processor Python code are both in place from the template. Customizing the widget UI (building the actual interactive components) is handled by the add-widget skill's build pipeline (future work).

## Execution Context

This phase runs as a subagent spawned by the orchestrator (SKILL.md). You have full access to:
- `Read` tool for reading state files
- `Write` / `Edit` tools for updating state

Your job: Confirm the widget is scaffolded, update state, and return immediately.

## Prerequisites

- Phase 2 must be completed (widget app structure set up)

## Steps

### Step 1: Read State

Read `{app-directory}/.appgen/PHASE.md` to get:
- App name
- App directory path

Derive name variants (PascalCase, kebab-case, snake_case).

### Step 2: Confirm Widget Structure

Verify the following files exist (read to confirm, do not modify):

- `{app-dir}/{kebab-case}/src/{PascalCase}.js` — the widget JS component (starting point for UI customization)
- `{app-dir}/{kebab-case}/rsbuild.config.ts` — Module Federation config
- `{app-dir}/src/{app_name}/application.py` — processor with RemoteComponent setup
- `{app-dir}/src/{app_name}/app_config.py` — config schema

### Step 3: Update State

Update `.appgen/PHASE.md`:
- Set current phase to "Phase 3 - Widget Build"
- Set status to "completed"
- Add Phase 3 to completed phases list
- Note: "Widget scaffolded from template. JS component ready for customization."

No user interaction needed — mark as completed immediately.

## Completion Criteria

Phase 3 is complete when:
- [ ] Widget JS component file confirmed to exist
- [ ] Processor application.py confirmed to exist
- [ ] `.appgen/PHASE.md` updated with status "completed"

## Summary for Orchestrator

When returning to the orchestrator, include:
- **Status**: success
- **Widget component**: Path to the JS component file
- **Processor**: Path to the application.py file
- **Note**: Widget is scaffolded and ready for customization via add-widget skill
