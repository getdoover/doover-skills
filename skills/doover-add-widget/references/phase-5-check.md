# Phase 5: Widget Check

Validate the widget builds successfully and produces the expected output.

## Phase Objective

Run the widget build and verify the output file exists, the configuration is correct, and all expected files are in place.

## Execution Context

This phase runs as a subagent spawned by the orchestrator (SKILL.md). You have full access to:
- `Read` tool for reading files
- `Bash` tool for running build and validation commands

Your job: Validate the widget, report any errors, then return a summary.

## Prerequisites

- Phase 2 must be completed at minimum (widget is scaffolded)
- If Phases 3-4 ran, Phase 4 must be completed (widget code written)
- `.appgen/WIDGET_PHASE.md` shows the previous phase completed

## Steps

### Step 1: Read State

Read `{app-dir}/.appgen/WIDGET_PHASE.md` to get:
- Widget name (PascalCase, kebab-case, snake_case)
- App directory path

### Step 2: Build Widget

Run the widget build:

```bash
cd {app-dir}/{kebab-case} && npm run build
```

**Expected**: Build completes without errors.

**If build fails**:
- Note the full error output
- Common issues:
  - Import errors (wrong hook name or import path)
  - JSX syntax errors
  - Missing dependencies (forgot to `npm install` a package)
  - Module Federation resolution errors (wrong shared module path)
- Report in summary with the full error output

### Step 3: Verify Build Output

Check that the built JavaScript file exists:

```bash
ls -la {app-dir}/assets/{PascalCase}.js
```

**Expected**: File exists and has non-zero size.

**If missing**: The ConcatenatePlugin may have failed or the build output path is misconfigured. Check:
- `{app-dir}/{kebab-case}/rsbuild.config.ts` — verify the output filename matches `{PascalCase}.js`
- Build log output for any ConcatenatePlugin errors

### Step 4: Verify doover_config.json

Read `{app-dir}/doover_config.json` and verify:

1. **file_deployments** — contains an entry with:
   - `"name": "{snake_case}"`
   - `"file_dir": "assets/{PascalCase}.js"`
   - `"mime_type": "text/javascript"`

2. **deployment_channel_messages** — contains a `ui_state` channel entry with the widget in `state.children`:
   - `"type": "uiRemoteComponent"`
   - `"name": "{PascalCase}"`
   - `"componentUrl": "{snake_case}"`

### Step 5: Verify File Structure

Check that expected files exist:

**Widget directory** (`{app-dir}/{kebab-case}/`):
- `rsbuild.config.ts` — Module Federation config
- `ConcatenatePlugin.ts` — Build output concatenation plugin
- `package.json` — Dependencies
- `src/{PascalCase}.js` — Widget component source

**Build output**:
- `assets/{PascalCase}.js` — Built widget file

Report any missing files.

### Step 6: Update State

Update `.appgen/WIDGET_PHASE.md`:
- Set current phase to "Phase 5 - Check"
- Set status to "completed" (even if errors found — document them)
- Add Phase 5 to completed phases list
- Note validation results and any errors

## Validation Results

Document the results of each check:

| Check | Status | Notes |
|-------|--------|-------|
| npm run build | PASS/FAIL | {error details if any} |
| Build output exists | PASS/FAIL | {file size} |
| doover_config.json | PASS/FAIL | {issues if any} |
| File structure | PASS/FAIL | {missing files if any} |

## Completion Criteria

Phase 5 is complete when:
- [ ] `npm run build` executed (pass or fail documented)
- [ ] Build output checked
- [ ] `doover_config.json` verified
- [ ] File structure verified
- [ ] `.appgen/WIDGET_PHASE.md` updated with validation results

## Summary for Orchestrator

When returning to the orchestrator, include:
- **Status**: success (all checks passed) or partial (some checks failed)
- **Checks passed**: List of successful validations
- **Checks failed**: List of failed validations with error details
- **Recommended fixes**: If any checks failed, suggest what to fix
- **Errors**: Any issues running the validation itself
