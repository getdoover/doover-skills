# Phase 2: Widget Config (Scaffolding)

Scaffold the widget boilerplate by cloning the template, renaming, and integrating into the app.

## Phase Objective

Clone the widget template, rename it to match the widget name, move it into the app directory, register it in `doover_config.json`, and install dependencies. This creates the base widget boilerplate — a working 3-layer React component with Module Federation configured.

## Execution Context

This phase runs as a subagent spawned by the orchestrator (SKILL.md). You have full access to:
- `Read` tool for reading state and existing files
- `Write` / `Edit` tools for modifying files
- `Bash` tool for running commands

Your job: Scaffold the widget boilerplate, then return a summary.

## Prerequisites

- App directory exists with a `doover_config.json`
- Widget name variants are known (PascalCase, kebab-case, snake_case)
- GitHub CLI (`gh`) authenticated
- Node.js 18+ available

## Steps

### Step 1: Read State

Read `{app-dir}/.appgen/WIDGET_PHASE.md` to get:
- Widget name variants (PascalCase, kebab-case, snake_case)
- App directory path

### Step 2: Clone Widget Template

Clone the widget template to a temporary directory:

```bash
gh repo clone getdoover/widget-template {app-dir}/_widget-tmp
```

**Verify** the clone succeeded and `{app-dir}/_widget-tmp/rename.js` exists before proceeding.

### Step 3: Rename Widget

Run the rename script from inside the cloned template:

```bash
cd {app-dir}/_widget-tmp && node rename.js {PascalCase}
```

The rename script updates:
- `widget/rsbuild.config.ts` — MF scope, exposes, ConcatPlugin output name
- `widget/package.json` — npm package name
- `widget/src/{PascalCase}.js` — component function name, renamed from `WidgetTemplate.js`
- `doover_config.json` — template's config (not the app's)
- `ui_state_paste.json` — testing helper
- `pyproject.toml` — project name
- `src/widget_template/*.py` — Python source files (renamed directory + class names)

**Verify** the rename completed successfully (check the script's output) before proceeding.

### Step 4: Move Widget Folder and Delete Temp Clone

Move the renamed `widget/` folder into the app directory:

```bash
mv {app-dir}/_widget-tmp/widget {app-dir}/{kebab-case}
```

Then delete the entire temporary clone (removes rename.js, Python processor files, and all other template-level files not needed when adding a widget to an existing app):

```bash
rm -rf {app-dir}/_widget-tmp
```

**Verify** that `{app-dir}/{kebab-case}/rsbuild.config.ts` exists and `{app-dir}/_widget-tmp` no longer exists.

### Step 5: Update doover_config.json

Read the app's `{app-dir}/doover_config.json` and make the following two additions:

#### 5a: File Deployment Entry

Add an entry to `file_deployments.files` for the widget's built JavaScript file:

```json
{
    "name": "{snake_case}",
    "file_dir": "assets/{PascalCase}.js",
    "mime_type": "text/javascript"
}
```

If `file_deployments` doesn't exist, create it:
```json
"file_deployments": {
    "files": [
        {
            "name": "{snake_case}",
            "file_dir": "assets/{PascalCase}.js",
            "mime_type": "text/javascript"
        }
    ]
}
```

If `file_deployments.files` exists, append the new entry to the end of the array.

#### 5b: UI State Entry

Find the `ui_state` channel entry inside `deployment_channel_messages` and add the widget as a new child at the **bottom** of `channel_message.state.children`:

```json
"{PascalCase}": {
    "type": "uiRemoteComponent",
    "name": "{PascalCase}",
    "componentUrl": "{snake_case}",
    "children": {}
}
```

**Important:** The widget MUST be added as the LAST entry in `children` so it appears below any existing UI components.

If `deployment_channel_messages` doesn't exist, create it:
```json
"deployment_channel_messages": [
    {
        "channel_name": "ui_state",
        "channel_message": {
            "state": {
                "children": {
                    "{PascalCase}": {
                        "type": "uiRemoteComponent",
                        "name": "{PascalCase}",
                        "componentUrl": "{snake_case}",
                        "children": {}
                    }
                }
            }
        }
    }
]
```

If `deployment_channel_messages` exists but has no `ui_state` entry, add one. If the `ui_state` entry exists but has no `state.children`, create the structure.

**Use 4-space indentation when writing JSON to match Doover conventions.**

### Step 6: Update .gitignore

Ensure these patterns are present in `.gitignore` (add any that are missing):

```
assets/
**/node_modules/
**/dist/
```

If no `.gitignore` exists, create one with these entries.

### Step 7: Install Dependencies

```bash
cd {app-dir}/{kebab-case} && npm install
```

**Verify** that `{app-dir}/{kebab-case}/node_modules` exists. If `npm install` fails, report the error and note that Node 18+ is required.

### Step 8: Update State

Update `{app-dir}/.appgen/WIDGET_PHASE.md`:
- Set current phase to "Phase 2 - Config"
- Set status to "completed"
- Add Phase 2 to completed phases list

## Completion Criteria

Phase 2 is complete when:
- [ ] Widget template cloned and renamed
- [ ] Widget folder moved to `{app-dir}/{kebab-case}/`
- [ ] Temp clone deleted
- [ ] `doover_config.json` updated with `file_deployments` and `deployment_channel_messages`
- [ ] `.gitignore` updated with widget patterns
- [ ] `npm install` completed successfully
- [ ] `.appgen/WIDGET_PHASE.md` updated with status "completed"

## Summary for Orchestrator

When returning to the orchestrator, include:
- **Status**: success or failure
- **Widget directory**: Path to the created widget folder
- **doover_config.json**: Entries added (file_deployments, deployment_channel_messages)
- **Files created/modified**: List of files touched
- **Errors**: Any issues encountered
