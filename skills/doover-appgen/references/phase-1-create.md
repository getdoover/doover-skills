# Phase 1: Creation

Create the base Doover application and GitHub repository.

## Phase Objective

Interactively gather app configuration from the user, create a new Doover application, and set up the GitHub repository.

## Execution Context

This phase runs as a subagent spawned by the orchestrator (SKILL.md). You have full access to:
- `AskUserQuestion` tool for gathering user input
- `Bash` tool for running commands
- `Write` / `Edit` tools for creating and modifying files
- `WebSearch` tool for finding brand images online

Your job: Execute this phase completely, update state files, and return a summary.

## Prerequisites

None - this is the first phase.

## Interactive Steps

### Step 1: Gather App Details

Use `AskUserQuestion` to collect required information:

**Question 1: App Name**
- Ask for the application name
- This will be used as the app identifier
- No spaces (kebab-case recommended)

**Question 2: App Description**
- Ask for a brief description of what the app does
- Used in app metadata

**Question 3: GitHub Repository Location**
- Ask for the full repo path (e.g., `my-org/my-app-name`)
- This is where the app code will be hosted

**Question 4: App Type**
- Ask what type of Doover app to create
- Options:
  - **Docker App** - Device-based, runs in a Docker container on Doover devices
  - **Processor** - Cloud-based, event-triggered logic that runs serverless
  - **Integration** - Cloud-based, receives data from external systems via HTTP endpoints
- Store the selection as: `docker`, `processor`, or `integration`

**Question 5: Should this app have a UI?** (conditional)
- **Only ask if app_type is `docker` or `processor`**
- **Skip this question for `integration`** (integrations never have UI)
- Options: Yes, No
- Store the selection as: `has_ui: true` or `has_ui: false`
- For integrations, automatically set `has_ui: false` without asking

**Hardcoded defaults (not asked):**
- Container registry: `ghcr.io/getdoover`
- Repository visibility: public

### Step 2: Determine Target Directory

- If user provided a path, use that as the target
- Otherwise, the app will be created in a subdirectory of the current working directory
- The app directory name will match the app name

### Step 3: Run App Creation Command

Execute the `doover app create` command with collected parameters.

**Important:** The CLI has interactive prompts that fail in non-TTY environments. Always provide `--owner-org-key` and `--container-profile-key` flags (with empty strings if unknown) to skip the interactive prompts:

```bash
doover app create --name {app-name} --description "{description}" --git --container-registry "ghcr.io/getdoover" --owner-org-key "" --container-profile-key ""
```

If the owner org key or container profile key are known, provide them instead of empty strings.

The `--git` flag ensures a local git repository is initialized with an initial commit.

### Step 4: Create GitHub Repository

After local app creation succeeds, rename the default branch to `main` and create the GitHub repository:

```bash
cd {app-directory}
git branch -m master main
gh repo create {org/repo-name} --source . --push --public
```

This:
- Renames the default branch from `master` to `main`
- Creates the GitHub repository (public)
- Sets it as the remote origin
- Pushes the initial commit

### Step 5: Verify Creation

After both commands complete:
1. Verify the app directory exists locally
2. Check for expected files (doover_config.json, src/, etc.)
3. Verify the GitHub repo was created (check `git remote -v`)
4. Note the full local path and GitHub URL

### Step 6: Find Brand Images

The app needs an icon and banner image for display in the Doover admin UI. Based on the app name and description, attempt to identify a brand or service associated with the app.

**Image requirements:**
- **Icon:** Will be displayed at 64x64px. Should have a transparent or white background.
- **Banner:** Will be displayed at max 800px wide x 100px tall. Should have a transparent or white background.

**Process:**
1. Analyze the app name and description to identify if it integrates with a known brand/service (e.g., "WhatsApp", "Slack", "Power BI")
2. If a brand is identified, use `WebSearch` to find official or high-quality logo/icon URLs:
   - Search for "{brand} logo png transparent" or "{brand} icon svg"
   - Look for URLs from official sources, Wikipedia, or reputable logo repositories
   - Prefer SVG or high-resolution PNG with transparent backgrounds
3. If suitable images are found, store the URLs for `icon_url` and `banner_url`
4. If no brand is identified, or images cannot be found, or it's ambiguous which images to use:
   - Use `AskUserQuestion` to prompt the user for icon and banner URLs
   - Explain what's needed (icon 64x64, banner 800x100, transparent/white background preferred)

Store the URLs in the PHASE.md state file. The actual update to `doover_config.json` will happen in Phase 2.

## State Updates

After successful app and repo creation, create the state file:

**Create directory:** `{app-directory}/.appgen/`

**Create file:** `{app-directory}/.appgen/PHASE.md`

```markdown
# AppGen State

## Current Phase
Phase 1 - Creation

## Status
completed

## App Details
- **Name:** {app-name}
- **Description:** {description}
- **App Type:** {docker|processor|integration}
- **Has UI:** {true|false}
- **Container Registry:** ghcr.io/getdoover
- **Target Directory:** {full-path}
- **GitHub Repo:** {org/repo-name}
- **Repo Visibility:** public
- **GitHub URL:** https://github.com/{org/repo-name}
- **Icon URL:** {icon-url}
- **Banner URL:** {banner-url}

## Completed Phases
- [x] Phase 1: Creation - {timestamp}

## User Decisions
- App name: {app-name}
- Description: {description}
- GitHub repo: {org/repo-name}
- App type: {docker|processor|integration}
- Has UI: {true|false}
- Icon URL: {icon-url}
- Banner URL: {banner-url}

## Next Action
Phase 1 complete. Ready for next phase when available.
```

Replace placeholders with actual values gathered during the process.

## Completion Criteria

Phase 1 is complete when:
- [ ] App directory exists at the target location
- [ ] `doover_config.json` file is present in the app directory
- [ ] GitHub repository exists and is set as remote origin
- [ ] Initial commit has been pushed to GitHub
- [ ] `.appgen/PHASE.md` has been created with status "completed"
- [ ] `.appgen/PHASE.md` contains `icon_url` and `banner_url`

## Phase Completion Output

When phase completes successfully, confirm to the user:

```
Phase 1 Complete: Creation

Created Doover app "{app-name}" at:
  {full-path}

GitHub repository:
  https://github.com/{org/repo-name}

App details saved to .appgen/PHASE.md

Ready for next phase when available.
```

## Summary for Orchestrator

When returning to the orchestrator, include:
- **Status**: success or failure
- **App name**: The name of the created app
- **App directory**: Full path to the app directory
- **GitHub URL**: Full URL to the GitHub repository
- **Errors**: Any issues encountered (if applicable)
