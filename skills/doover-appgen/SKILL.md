---
name: doover-appgen
description: Generate Doover applications step-by-step with guided phases. Use when users want to create a new Doover app, scaffold application components, or need help with the app generation workflow.
---

# Doover AppGen Skill

Generate Doover applications through an interactive, phase-based workflow.

## Overview

This skill guides users through creating Doover applications step-by-step. Each phase handles a specific part of the app generation process, with clear completion criteria before proceeding to the next phase.

**Workflow:**
1. Gather requirements interactively (app name, description, registry)
2. Execute commands to create/configure the app
3. Track state in the target app's `.appgen/` directory
4. Proceed through phases sequentially

## Skill Directory Structure

```
skills/doover-appgen/
├── SKILL.md                      # This file - orchestrator and routing
└── references/
    ├── phase-1-app-create.md     # Phase 1: Initial app creation
    ├── phase-2-app-build.md      # Phase 2: Generate app code
    └── mini-docs.md              # Doover app development guide
```

Phase files are loaded on-demand. Do not read all phases upfront.

## State Storage

State is stored in the **target app directory**, not the skill directory:

```
{target-app}/
└── .appgen/
    └── PHASE.md    # Current phase, status, user decisions
```

The `.appgen/PHASE.md` file tracks:
- Current phase and status
- App configuration details
- Completed phases with timestamps
- User decisions made during the process

## Available Phases

| Phase | Name | Description |
|-------|------|-------------|
| 1 | App Creation | Create the base Doover app using `doover app create` |
| 2 | App Build | Generate application code based on description |

Additional phases will be added incrementally.

## Phase Gating Protocol

**Rules:**
1. Each phase must complete before the next begins
2. Check `.appgen/PHASE.md` status before loading next phase
3. A phase is complete when all completion criteria are met
4. Never skip phases or run phases out of order

**To check current phase:**
1. Look for `.appgen/PHASE.md` in the target directory
2. If it doesn't exist, start at Phase 1
3. If it exists, read the current phase and status

## Phase Execution Protocol

Each phase runs in an isolated subagent to manage context efficiently.

### Orchestrator Responsibilities (this file)

1. Determine the target directory (user-specified or cwd)
2. Read `.appgen/PHASE.md` to determine current phase (or start at Phase 1)
3. Spawn a Task subagent to execute that phase
4. Report the subagent's summary to the user
5. Check if more phases remain; if so, spawn the next phase subagent

### Subagent Responsibilities

1. Read the phase instructions from `references/phase-{N}-*.md`
2. Read any existing state from `.appgen/` files
3. Ask user questions as needed via `AskUserQuestion`
4. Execute commands (doover CLI, gh CLI, etc.)
5. Update `.appgen/PHASE.md` with results and user decisions
6. Return a summary to the orchestrator

### Why Subagents?

- **Context isolation**: Each phase starts fresh without baggage from previous phases
- **State via files**: PHASE.md is the single source of truth for inter-phase communication
- **Efficient**: Smaller context per phase = faster, cheaper execution
- **Resumable**: Re-invoking the skill reads PHASE.md and resumes from current phase

## Execution Instructions

**When this skill is invoked:**

### Step 1: Determine Target Directory

- If user specified a path, use that
- Otherwise, use the current working directory
- Store this as `{target-dir}` for subagent prompts

### Step 2: Check for Existing State

Look for `{target-dir}/.appgen/PHASE.md`:
- If not found → Start at Phase 1
- If found → Read it to determine current phase and status
  - If status is "completed" → Proceed to next phase
  - If status is "in_progress" → Resume current phase

### Step 3: Spawn Phase Subagent

Use the Task tool to spawn a subagent for the current phase.

**Phase 1 Subagent Invocation:**

```
Task tool with:
  subagent_type: "general-purpose"
  prompt: |
    Execute Phase 1 of the doover-appgen skill.

    Target directory: {target-dir}
    Skill location: {path-to-this-skill}

    Instructions:
    1. Read {skill-path}/references/phase-1-app-create.md
    2. Follow the interactive steps - use AskUserQuestion to gather:
       - App name (kebab-case)
       - App description
       - GitHub repo path (org/repo-name)
    3. Run `doover app create` with the gathered parameters
    4. Run `gh repo create` to create and push to GitHub
    5. Create {target-dir}/{app-name}/.appgen/PHASE.md with all state
    6. Return a summary including:
       - App name and directory path
       - GitHub URL
       - Any issues encountered
```

**Phase 2 Subagent Invocation:**

```
Task tool with:
  subagent_type: "general-purpose"
  prompt: |
    Execute Phase 2 of the doover-appgen skill.

    App directory: {app-dir}
    Skill location: {path-to-this-skill}

    Instructions:
    1. Read {app-dir}/.appgen/PHASE.md to get app name and description
    2. Read {skill-path}/references/phase-2-app-build.md for phase instructions
    3. Read {skill-path}/references/mini-docs.md for Doover app patterns
    4. Generate application code (prefer pydoover imports, avoid external deps):
       - src/{app_name}/application.py
       - src/{app_name}/app_config.py
       - src/{app_name}/app_ui.py
       - src/{app_name}/__init__.py
    5. If external packages needed, run: uv add {package}
    6. Update {app-dir}/.appgen/PHASE.md with completion status
    7. Return a summary including:
       - Files created/modified
       - What the app does
       - Known limitations
       - Any errors
```

### Step 4: Report and Continue

After the subagent returns:
1. Report its summary to the user
2. Check if more phases exist
3. If yes, spawn the next phase subagent
4. If no, report workflow completion

### Error Handling

If a subagent reports failure:
1. Report the error to the user
2. The phase status in PHASE.md should remain "in_progress"
3. Suggest the user can re-invoke the skill to retry
