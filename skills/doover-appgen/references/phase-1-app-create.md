# Phase 1: App Creation

Create the base Doover application and GitHub repository.

## Phase Objective

Interactively gather app configuration from the user, create a new Doover application, and set up the GitHub repository.

## Execution Context

This phase runs as a subagent spawned by the orchestrator (SKILL.md). You have full access to:
- `AskUserQuestion` tool for gathering user input
- `Bash` tool for running commands
- `Write` tool for creating state files

Your job: Execute this phase completely, update state files, and return a summary.

## Prerequisites

None - this is the first phase.

## Interactive Steps

### Step 1: Gather App Details

Use `AskUserQuestion` to collect required information:

**Question 1: App Name**
- Ask for the application name
- This will be used as the app identifier
- Should be lowercase, no spaces (kebab-case recommended)

**Question 2: App Description**
- Ask for a brief description of what the app does
- Used in app metadata

**Question 3: GitHub Repository Location**
- Ask for the full repo path (e.g., `my-org/my-app-name`)
- This is where the app code will be hosted

**Hardcoded defaults (not asked):**
- Container registry: `ghcr.io/getdoover`
- Repository visibility: public

### Step 2: Determine Target Directory

- If user provided a path, use that as the target
- Otherwise, the app will be created in a subdirectory of the current working directory
- The app directory name will match the app name

### Step 3: Run App Creation Command

Execute the `doover app create` command with collected parameters:

```bash
doover app create --name {app-name} --description "{description}" --git --container-registry "ghcr.io/getdoover"
```

The `--git` flag ensures a local git repository is initialized with an initial commit.

If owner org key is known, add: `--owner-org-key {key}`

### Step 4: Create GitHub Repository

After local app creation succeeds, create the GitHub repository and push:

```bash
cd {app-directory}
gh repo create {org/repo-name} --source . --push --public
```

This command:
- Creates the GitHub repository (public)
- Sets it as the remote origin
- Pushes the initial commit

### Step 5: Verify Creation

After both commands complete:
1. Verify the app directory exists locally
2. Check for expected files (doover_config.json, src/, etc.)
3. Verify the GitHub repo was created (check `git remote -v`)
4. Note the full local path and GitHub URL

## State Updates

After successful app and repo creation, create the state file:

**Create directory:** `{app-directory}/.appgen/`

**Create file:** `{app-directory}/.appgen/PHASE.md`

```markdown
# AppGen State

## Current Phase
Phase 1 - App Creation

## Status
completed

## App Details
- **Name:** {app-name}
- **Description:** {description}
- **Container Registry:** ghcr.io/getdoover
- **Target Directory:** {full-path}
- **GitHub Repo:** {org/repo-name}
- **Repo Visibility:** public
- **GitHub URL:** https://github.com/{org/repo-name}

## Completed Phases
- [x] Phase 1: App Creation - {timestamp}

## User Decisions
- App name: {app-name}
- Description: {description}
- GitHub repo: {org/repo-name}

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

## Phase Completion Output

When phase completes successfully, confirm to the user:

```
Phase 1 Complete: App Creation

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
