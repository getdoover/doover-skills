# Phase 5p: Processor Check

Validate the generated processor application code.

## Phase Objective

Perform validation checks on the generated processor to ensure it's correctly structured, dependencies resolve, imports work, and configuration exports successfully.

## Execution Context

This phase runs as a subagent spawned by the orchestrator (SKILL.md). You have full access to:
- `Read` tool for reading files
- `Bash` tool for running validation commands

Your job: Validate the code, report any errors, then return a summary.

## Prerequisites

- Phase 4 must be completed (application code generated)
- `.appgen/PHASE.md` shows Phase 4 completed

## Steps

### Step 1: Read State

Read `{app-directory}/.appgen/PHASE.md` to get:
- App name
- App directory path

### Step 2: Validate Dependencies

Run `uv sync` to ensure all dependencies resolve correctly:

```bash
cd {app-directory}
uv sync
```

**Expected**: Command completes without errors.

**If errors occur**:
- Note the missing or conflicting dependencies
- Report in summary

### Step 3: Validate Imports

Test that the Lambda handler can be imported:

```bash
cd {app-directory}
uv run python -c "from {app_name} import handler; print('Import OK')"
```

**Expected**: Prints "Import OK" without errors.

**If errors occur**:
- Note the import error message
- Identify the problematic module/file
- Report in summary

### Step 4: Validate Config Schema

Run the config schema export to validate configuration:

```bash
cd {app-directory}
doover config-schema export
```

**Expected**: Command completes without errors, schema is valid.

**If errors occur**:
- Note the configuration error
- Identify the problematic field in app_config.py
- Report in summary

### Step 5: Check File Structure

Verify the expected files exist:

```bash
cd {app-directory}
ls -la src/{app_name}/
ls -la {app-directory}/
```

**Expected files in src/{app_name}/**:
- `__init__.py` - Lambda handler entry point
- `application.py` - Application class
- `app_config.py` - Configuration schema
- `app_ui.py` - UI components (optional, if has_ui)

**Expected files in app root**:
- `build.sh` - Build script for packaging
- `doover_config.json` - Doover configuration

Report any missing files.

### Step 6: Validate doover_config.json

Read and verify `doover_config.json` has correct processor configuration:

```bash
cd {app-directory}
cat doover_config.json
```

**Expected**:
- `type` is `"PRO"`
- `handler` points to correct Lambda handler
- `lambda_config` section exists with Runtime, Timeout, MemorySize, Handler

### Step 7: Update State

Update `.appgen/PHASE.md`:
- Set current phase to "Phase 5 - Processor Check"
- Set status to "completed" (even if errors found - document them)
- Add Phase 5 to completed phases list
- Note validation results and any errors

## Validation Results

Document the results of each check:

| Check | Status | Notes |
|-------|--------|-------|
| Dependencies (uv sync) | PASS/FAIL | {error details if any} |
| Imports (handler) | PASS/FAIL | {error details if any} |
| Config Schema | PASS/FAIL | {error details if any} |
| File Structure | PASS/FAIL | {missing files if any} |
| doover_config.json | PASS/FAIL | {issues if any} |

## Completion Criteria

Phase 5 is complete when:
- [ ] `uv sync` executed (pass or fail documented)
- [ ] Import check executed (pass or fail documented)
- [ ] `doover config-schema export` executed (pass or fail documented)
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
