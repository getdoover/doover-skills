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
- `icon_url` and `banner_url`

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

### Step 4: Update doover_config.json

Update the `doover_config.json` file with the validated image URLs from Phase 1:

1. Read `{app-directory}/doover_config.json`
2. Update the following fields under the app name key:
   - `icon_url` - Set to the validated icon URL
   - `banner_url` - Set to the validated banner URL
3. Write the updated config back to `doover_config.json`

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
- [ ] Image URLs validated (return 200 OK)
- [ ] `doover_config.json` updated with validated `icon_url` and `banner_url`
- [ ] `.appgen/PHASE.md` updated with status "completed"

## Summary for Orchestrator

When returning to the orchestrator, include:
- **Status**: success or failure
- **UI configured**: Whether UI was kept or removed
- **Images configured**: icon_url and banner_url applied
- **Files modified**: List of files touched
- **Errors**: Any issues encountered
