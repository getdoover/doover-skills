# Phase 4w: Widget Check

Validate the generated widget application code.

## Phase Objective

Perform validation checks on the widget app to ensure the widget builds, Python imports work, config exports successfully, and file structure is correct.

## Execution Context

This phase runs as a subagent spawned by the orchestrator (SKILL.md). You have full access to:
- `Read` tool for reading files
- `Bash` tool for running validation commands
- `Write` / `Edit` tools for fixing issues

Your job: Validate the code, report any errors, then return a summary.

## Prerequisites

- Phase 3 must be completed (widget confirmed scaffolded)
- `.appgen/PHASE.md` shows Phase 3 completed

## Steps

### Step 1: Read State

Read `{app-directory}/.appgen/PHASE.md` to get:
- App name (snake_case)
- App directory path

Derive name variants (PascalCase, kebab-case).

### Step 2: Validate Widget Build

Build the widget to verify the JS toolchain works:

```bash
cd {app-dir}/{kebab-case} && npm run build
```

**Expected**: Command completes without errors and `{app-dir}/assets/{PascalCase}.js` exists.

**If errors occur**:
- Note the build error message
- Check that Node 18+ is available
- Check that npm dependencies are installed
- Report in summary

### Step 3: Validate Python Imports

Test that the application class can be imported:

```bash
cd {app-dir}
uv run python -c "from {app_name}.application import {AppClass}; print('Import OK')"
```

Where `{AppClass}` is the PascalCase app name + "App" (e.g., `MyWidgetApp` for `my_widget`).

**Expected**: Prints "Import OK" without errors.

**If errors occur**:
- Note the import error message
- Identify the problematic module/file
- Report in summary

### Step 4: Validate Config Export

Run the config export to verify the config schema is valid:

```bash
cd {app-dir}
uv run export-config
```

**Expected**: Command completes without errors, and `doover_config.json` has `config_schema` populated (not empty `{}`).

**If errors occur**:
- Note the configuration error
- Identify the problematic field in app_config.py
- Report in summary

### Step 5: Check File Structure

Verify the expected files exist:

**Widget files in `{kebab-case}/`:**
- `rsbuild.config.ts` — Module Federation config
- `ConcatenatePlugin.ts` — build plugin
- `package.json` — npm dependencies
- `src/{PascalCase}.js` — widget component

**Python files in `src/{app_name}/`:**
- `__init__.py` — Lambda handler entry point
- `application.py` — Application class with RemoteComponent
- `app_config.py` — Configuration schema

**Root files:**
- `doover_config.json` — Doover configuration
- `pyproject.toml` — Python project config
- `build.sh` — Lambda packaging script

Report any missing files.

### Step 6: Validate doover_config.json

Read and verify `doover_config.json` has correct widget+processor configuration:

**Expected**:
- Top-level app key matches app name (snake_case)
- `type` is `"PRO"`
- `handler` points to correct Lambda handler (`src.{app_name}.handler`)
- `lambda_config` section exists with Runtime, Timeout, MemorySize, Handler
- `file_deployments.files` has an entry for the widget JS file
- `deployment_channel_messages` has a ui_state entry with the widget's RemoteComponent

### Step 7: Update State

Update `.appgen/PHASE.md`:
- Set current phase to "Phase 4 - Widget Check"
- Set status to "completed" (even if errors found — document them)
- Add Phase 4 to completed phases list
- Note validation results and any errors

## Validation Results

Document the results of each check:

| Check | Status | Notes |
|-------|--------|-------|
| Widget build (npm run build) | PASS/FAIL | {error details if any} |
| Built JS file exists | PASS/FAIL | {path checked} |
| Python imports | PASS/FAIL | {error details if any} |
| Config export | PASS/FAIL | {error details if any} |
| File structure | PASS/FAIL | {missing files if any} |
| doover_config.json | PASS/FAIL | {issues if any} |

## Completion Criteria

Phase 4 is complete when:
- [ ] Widget build executed (pass or fail documented)
- [ ] Built JS file existence checked
- [ ] Import check executed (pass or fail documented)
- [ ] Config export executed (pass or fail documented)
- [ ] File structure verified
- [ ] doover_config.json verified
- [ ] `.appgen/PHASE.md` updated with validation results

## Summary for Orchestrator

When returning to the orchestrator, include:
- **Status**: success (all checks passed) or partial (some checks failed)
- **Checks passed**: List of successful validations
- **Checks failed**: List of failed validations with error details
- **Recommended fixes**: If any checks failed, suggest what to fix
- **Errors**: Any issues running the validation itself
