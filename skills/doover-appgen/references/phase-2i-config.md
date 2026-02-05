# Phase 2i: Integration Config

Configure the integration application template.

## Phase Objective

Apply configuration changes to the base integration template. Integrations never have UI, so this phase always removes UI-related files.

## Execution Context

This phase runs as a subagent spawned by the orchestrator (SKILL.md). You have full access to:
- `Read` tool for reading state and existing files
- `Write` / `Edit` tools for modifying files
- `Bash` tool for running commands
- `AskUserQuestion` tool for prompting user if image URLs are invalid

Your job: Configure the template, then return a summary.

## Prerequisites

- Phase 1 must be completed (app directory exists with `.appgen/PHASE.md`)
- App type must be "integration"

## Steps

### Step 1: Read State from Phase 1

Read `{app-directory}/.appgen/PHASE.md` to get:
- App name (e.g., `my_integration`)
- App description
- `icon_url` and `banner_url`
- Note: `has_ui` is always false for integrations

### Step 2: Remove UI Components

Integrations never have UI, so always perform these steps:

1. Remove `src/{app_name}/app_ui.py` if it exists
2. Edit `src/{app_name}/application.py` to:
   - Remove the import line: `from .app_ui import export as export_ui` (if present)
   - Remove any UI-related code (if present)

### Step 3: Remove Build-Image Workflow

Integrations are deployed as zip packages, not container images. Remove the Docker build workflow:

1. Delete `.github/workflows/build-image.yml` if it exists
2. Optionally remove `Dockerfile` (not needed for integrations)

This prevents CI from building unnecessary Docker images for cloud apps.

### Step 4: Validate Image URLs

Before applying the image URLs to doover_config.json, validate that they are accessible and return valid images.

**Validation process:**

For each URL (`icon_url` and `banner_url`):
1. Make an HTTP HEAD or GET request to the URL using `curl -sI {url}` or `WebFetch`
2. Check that the response status is 200 OK
3. Optionally verify the Content-Type header contains `image/` (e.g., `image/svg+xml`, `image/png`)

**If validation fails:**
- Use `AskUserQuestion` to prompt the user for a replacement URL
- Explain which URL failed and why
- Re-validate the new URL before proceeding

**Example validation command:**
```bash
curl -sI "https://example.com/image.svg" | head -20
```

Look for `HTTP/... 200` in the response. If you see 404, 403, or other errors, the URL is invalid.

### Step 5: Update doover_config.json

The app-template's `doover_config.json` needs to be transformed for integration apps. Rewrite the config to match this structure:

```json
{
    "{app_name}": {
        "name": "{app_name}",
        "display_name": "{Display Name}",
        "type": "INT",
        "visibility": "PUB",
        "allow_many": true,
        "description": "{description from PHASE.md}",
        "long_description": "README.md",
        "depends_on": [],
        "handler": "src.{app_name}.handler",
        "lambda_config": {
            "Runtime": "python3.13",
            "Timeout": 300,
            "MemorySize": 128,
            "Handler": "src.{app_name}.handler"
        },
        "organisation_id": null,
        "key": null,
        "owner_org_id": null,
        "code_repo_id": null,
        "container_registry_profile_id": null,
        "repo_branch": "main",
        "image_name": null,
        "icon_url": "{icon_url from PHASE.md}",
        "banner_url": "{banner_url from PHASE.md}",
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
- `display_name` should be a human-readable version (e.g., "My Integration")
- `type` must be `"INT"` for integrations
- `handler` and `lambda_config.Handler` must point to `src.{app_name}.handler`
- Do NOT include an `id` field - this gets added during publishing
- `config_schema` will be populated later by running `uv run export-config`
- Remove any fields from the template that are not in this structure

### Step 6: Update State

Update `.appgen/PHASE.md`:

- Set current phase to "Phase 2 - Integration Config"
- Set status to "completed"
- Add Phase 2 to completed phases list
- Note that UI was removed (integrations never have UI)

## Completion Criteria

Phase 2 is complete when:
- [ ] `app_ui.py` removed
- [ ] `application.py` updated to remove UI imports/code (if any)
- [ ] `.github/workflows/build-image.yml` removed (if present)
- [ ] Image URLs validated (return 200 OK)
- [ ] `doover_config.json` restructured for integration (type: "INT", lambda_config, etc.)
- [ ] `doover_config.json` has validated `icon_url` and `banner_url` set
- [ ] `doover_config.json` does NOT have an `id` field
- [ ] `.appgen/PHASE.md` updated with status "completed"

## Summary for Orchestrator

When returning to the orchestrator, include:
- **Status**: success or failure
- **Build workflow removed**: Whether build-image.yml was removed
- **Config restructured**: doover_config.json updated for integration type
- **Files modified**: List of files touched
- **Errors**: Any issues encountered
