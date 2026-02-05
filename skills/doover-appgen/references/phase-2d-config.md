# Phase 2d: Docker Config

Configure the Docker device application template based on user choices from Phase 1.

## Phase Objective

Apply configuration changes to the base app template based on user choices from Phase 1, particularly the `has_ui` setting.

## Execution Context

This phase runs as a subagent spawned by the orchestrator (SKILL.md). You have full access to:
- `Read` tool for reading state and existing files
- `Write` / `Edit` tools for modifying files
- `Bash` tool for running commands
- `AskUserQuestion` tool for prompting user if image URLs are invalid

Your job: Configure the template, then return a summary.

## Prerequisites

- Phase 1 must be completed (app directory exists with `.appgen/PHASE.md`)
- App type must be "docker"

## Steps

### Step 1: Read State from Phase 1

Read `{app-directory}/.appgen/PHASE.md` to get:
- App name
- `has_ui` value (true or false)
- `icon_url`

### Step 2: Apply UI Configuration

**If `has_ui` is false:**

1. Remove `src/{app_name}/app_ui.py`
2. Edit `src/{app_name}/application.py` to:
   - Remove the import line: `from .app_ui import export as export_ui`
   - Remove the UI export from the Application class (the `export_ui()` call in `setup()` or similar)
   - Remove any UI callback registrations

**If `has_ui` is true:**

No changes needed - the template already includes UI support.

### Step 3: Validate Image URLs

Before applying the icon URL to doover_config.json, validate that it is accessible and returns a valid image.

**Validation process:**

For the `icon_url`:
1. Make an HTTP HEAD or GET request to the URL using `curl -sI {url}` or `WebFetch`
2. Check that the response status is 200 OK
3. Optionally verify the Content-Type header contains `image/` (e.g., `image/svg+xml`, `image/png`)

**If validation fails:**
- Use `AskUserQuestion` to prompt the user for a replacement URL
- Explain that the URL failed and why
- Re-validate the new URL before proceeding

**Example validation command:**
```bash
curl -sI "https://example.com/image.svg" | head -20
```

Look for `HTTP/... 200` in the response. If you see 404, 403, or other errors, the URL is invalid.

### Step 4: Update doover_config.json

The app-template's `doover_config.json` needs to be cleaned up for Docker device apps. Rewrite the config to match this structure:

```json
{
    "{app_name}": {
        "name": "{app_name}",
        "display_name": "{Display Name}",
        "type": "DEV",
        "visibility": "PUB",
        "allow_many": true,
        "description": "{description from PHASE.md}",
        "long_description": "README.md",
        "depends_on": [],
        "owner_org_key": "",
        "image_name": "{container_registry}/{app_name}:main",
        "container_registry_profile_key": "",
        "build_args": "--platform linux/amd64,linux/arm64",
        "lambda_config": {},
        "organisation_id": null,
        "key": null,
        "owner_org_id": null,
        "code_repo_id": null,
        "container_registry_profile_id": null,
        "repo_branch": "main",
        "icon_url": "{icon_url from PHASE.md}",
        "staging_config": {},
        "export_config_command": null,
        "run_command": null,
        "lambda_arn": null,
        "config_schema": {}
    }
}
```

**Important notes:**
- The top-level key and `name` field must match the app name (snake_case)
- `display_name` should be a human-readable version (e.g., "My App")
- `type` must be `"DEV"` for Docker device apps
- `image_name` should be set to `{container_registry}/{app_name}:main` (e.g., `ghcr.io/getdoover/my_app:main`)
- `build_args` enables multi-architecture builds (amd64 for cloud, arm64 for devices)
- `lambda_config` should be an empty object `{}` (not used for Docker apps but field is expected)
- Do NOT include an `id` field - this gets added during publishing
- `config_schema` will be populated later by running `uv run export-config`

### Step 5: Update State

Update `.appgen/PHASE.md`:

- Set current phase to "Phase 2 - Docker Config"
- Set status to "completed"
- Add Phase 2 to completed phases list
- Note what was configured (UI removed or kept)

## Completion Criteria

Phase 2 is complete when:
- [ ] `app_ui.py` removed if `has_ui` is false
- [ ] `application.py` updated to remove UI imports/code if `has_ui` is false
- [ ] Icon URL validated (returns 200 OK)
- [ ] `doover_config.json` restructured for Docker device app (type: "DEV", image_name, build_args, etc.)
- [ ] `doover_config.json` has validated `icon_url` set
- [ ] `doover_config.json` does NOT have an `id` field
- [ ] `.appgen/PHASE.md` updated with status "completed"

## Summary for Orchestrator

When returning to the orchestrator, include:
- **Status**: success or failure
- **UI configured**: Whether UI was kept or removed
- **Config restructured**: doover_config.json updated for Docker device type
- **Files modified**: List of files touched
- **Errors**: Any issues encountered
