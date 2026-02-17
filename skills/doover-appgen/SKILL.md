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
    ├── phase-1-create.md         # Phase 1: Initial creation
    ├── phase-2d-config.md        # Phase 2: Docker config
    ├── phase-2p-config.md        # Phase 2: Processor config
    ├── phase-2w-config.md        # Phase 2: Widget config
    ├── phase-2i-config.md        # Phase 2: Integration config
    ├── phase-r-refs.md           # Phase R: Reference extraction (optional)
    ├── phase-3d-plan.md          # Phase 3: Docker planning
    ├── phase-3p-plan.md          # Phase 3: Processor planning
    ├── phase-3w-build.md         # Phase 3: Widget build (minimal)
    ├── phase-3i-plan.md          # Phase 3: Integration planning
    ├── phase-4d-build.md         # Phase 4: Docker build
    ├── phase-4p-build.md         # Phase 4: Processor build
    ├── phase-4w-check.md         # Phase 4: Widget validation
    ├── phase-4i-build.md         # Phase 4: Integration build
    ├── phase-5d-check.md         # Phase 5: Docker validation
    ├── phase-5p-check.md         # Phase 5: Processor validation
    ├── phase-5i-check.md         # Phase 5: Integration validation
    └── phase-6-document.md       # Phase 6: Documentation (all app types)
```

**External dependency:** Platform documentation lives in `skills/doover-platform-docs/` (the `doover-platform-docs` skill). Subagents are given `{platform-docs-path}` to locate these docs.

Phase files are loaded on-demand. Do not read all phases upfront.

## State Storage

State is stored in the **target app directory**, not the skill directory:

```
{target-app}/
└── .appgen/
    ├── PHASE.md       # Current phase, status, user decisions
    ├── REFERENCES.md  # Reference patterns (created in Phase R, optional)
    └── PLAN.md        # Build plan (created in Phase 3)
```

The `.appgen/PHASE.md` file tracks:
- Current phase and status
- App configuration details
- Completed phases with timestamps
- User decisions made during the process

The `.appgen/PLAN.md` file (created in Phase 3) contains:
- Implementation plan for the Build phase
- Configuration schema design
- UI elements or event handlers to create
- External integration details

## Available Phases

| Phase | Name | Description |
|-------|------|-------------|
| 1 | Creation | Create the base Doover app using `doover app create` |
| 2d/2p/2w/2i | Config | Configure template based on app type |
| R | References | Extract patterns from reference repositories (optional) |
| 3d/3p/3i | Plan | Analyze requirements, resolve ambiguity, create PLAN.md |
| 3w | Build | Widget build (minimal — template already scaffolded) |
| 4d/4p/4i | Build | Generate application code from PLAN.md |
| 4w | Check | Widget validation (build, imports, config) |
| 5d/5p/5i | Check | Validate generated code |
| 6 | Document | Generate comprehensive README.md documentation |

Phase 2 variant is selected based on `app_type` from Phase 1:
- `docker` → phase-2d-config.md
- `processor` → phase-2p-config.md
- `widget` → phase-2w-config.md
- `integration` → phase-2i-config.md

Phase 3 variant is selected based on `app_type` from Phase 1:
- `docker` → phase-3d-plan.md
- `processor` → phase-3p-plan.md
- `widget` → phase-3w-build.md (minimal — template already scaffolded)
- `integration` → phase-3i-plan.md

Phase 4 variant is selected based on `app_type` from Phase 1:
- `docker` → phase-4d-build.md
- `processor` → phase-4p-build.md
- `widget` → phase-4w-check.md (validation — no build phase needed)
- `integration` → phase-4i-build.md

Phase 5 variant is selected based on `app_type` from Phase 1:
- `docker` → phase-5d-check.md
- `processor` → phase-5p-check.md
- `widget` → (skip — validation already done in Phase 4w)
- `integration` → phase-5i-check.md

Phase 6 is shared by all app types.

**Widget app phase flow:** 1 → 2w → add-widget → 3w → 4w → 6

After Phase 2w completes for widget apps, the orchestrator invokes the **add-widget skill** to scaffold the JavaScript widget side (clone template, rename, move folder, update doover_config.json with file_deployments/deployment_channel_messages, .gitignore, npm install). Any reference repos collected in Phase 1 are passed directly to the add-widget invocation (not through Phase R).
Widget apps skip Phase R (references are handled by add-widget) and Phase 5 (validation is handled in Phase 4w).

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
3. Ask user questions as needed via `AskUserQuestion` (Phase 3 only for Build-related questions)
4. Execute commands (doover CLI, gh CLI, etc.)
5. Update `.appgen/PHASE.md` with results and user decisions
6. Return a summary to the orchestrator

### Why Subagents?

- **Context isolation**: Each phase starts fresh without baggage from previous phases
- **State via files**: PHASE.md and PLAN.md are the sources of truth for inter-phase communication
- **Efficient**: Smaller context per phase = faster, cheaper execution
- **Resumable**: Re-invoking the skill reads PHASE.md and resumes from current phase

## Execution Instructions

**When this skill is invoked:**

### Step 1: Determine Target Directory

- If user specified a path, use that
- Otherwise, use the current working directory
- Store this as `{target-dir}` for subagent prompts

### Step 1b: Resolve Platform Docs Path

The platform documentation lives in the `doover-platform-docs` skill, located relative to this skill:
- `{platform-docs-path}` = `{this-skill-path}/../doover-platform-docs`
- Verify the path exists (check for `{platform-docs-path}/references/index.md`)
- This path is passed to Phase 3 and Phase 4 subagents

### Step 2: Check for Existing State

Look for `{target-dir}/.appgen/PHASE.md`:
- If not found → Start at Phase 1
- If found → Read it to determine current phase and status
  - If status is "completed" → Proceed to next phase
  - If status is "in_progress" → Resume current phase

### Step 2b: Detect Inline References

Before spawning Phase 1, check the user's initial prompt for references mentioned inline (e.g., "run appgen and use org/repo as a reference for their MQTT retry logic").

If references are detected in the prompt:
- Parse out: repository location + what to extract for each reference
- Pass this as `references_from_prompt` in the Phase 1 subagent prompt so Phase 1 can write them into PHASE.md (setting `has_references: true`) without re-asking the user
- Phase 1 will skip Question 6 (reference repos) since references are already provided

If no references are detected, proceed normally — Phase 1 will ask about references as Question 6.

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

    References from prompt (if any): {references_from_prompt or "none"}

    Instructions:
    1. Read {skill-path}/references/phase-1-create.md
    2. Follow the interactive steps - use AskUserQuestion to gather:
       - App name (kebab-case)
       - App description
       - GitHub repo path (org/repo-name)
       - App type (docker, processor, widget, or integration)
       - Has UI (only for docker/processor - widgets always have UI, integrations never have UI)
       - Reference repos (skip if references_from_prompt was provided)
    3. Run `doover app create` with the gathered parameters
    4. Run `gh repo create` to create and push to GitHub
    5. Create {target-dir}/{app-name}/.appgen/PHASE.md with all state (including references)
    6. Return a summary including:
       - App name and directory path
       - GitHub URL
       - Whether references were recorded
       - Any issues encountered
```

**Phase 2 Subagent Invocation:**

First, read the `app_type` from `{app-dir}/.appgen/PHASE.md` to determine which Phase 2 variant to use:
- `docker` → use `phase-2d-config.md`
- `processor` → use `phase-2p-config.md`
- `widget` → use `phase-2w-config.md`
- `integration` → use `phase-2i-config.md`

```
Task tool with:
  subagent_type: "general-purpose"
  prompt: |
    Execute Phase 2 of the doover-appgen skill.

    App directory: {app-dir}
    Skill location: {path-to-this-skill}
    App type: {app_type}  # docker, processor, widget, or integration

    Instructions:
    1. Read {app-dir}/.appgen/PHASE.md to get app name, app type, and has_ui value
    2. Based on app type, read the appropriate phase instructions:
       - docker: {skill-path}/references/phase-2d-config.md
       - processor: {skill-path}/references/phase-2p-config.md
       - widget: {skill-path}/references/phase-2w-config.md
       - integration: {skill-path}/references/phase-2i-config.md
    3. Follow the phase instructions to configure the template
    4. For docker/processor: if has_ui is false, remove app_ui.py and UI-related code
    5. For widget: set up processor side only (per phase-2w instructions) — widget JS scaffolding is handled separately by add-widget
    6. Update {app-dir}/.appgen/PHASE.md with completion status
    7. Return a summary including:
       - Files modified
       - Configuration applied
       - Name variants (PascalCase, kebab-case, snake_case) — needed for widget apps
       - Any errors
```

**Add-Widget Invocation — Widget Apps Only:**

After Phase 2w completes for widget apps, invoke the add-widget skill to scaffold and optionally customize the JavaScript widget side. This must run before Phase 3w.

Read `app_type` from `{app-dir}/.appgen/PHASE.md`:
- If `widget` → spawn add-widget subagent below
- Otherwise → skip to Phase R check

The widget name should match the app name. Read the name variants from PHASE.md (recorded by Phase 2w) and pass them to add-widget. Also read:
- The app description from PHASE.md — to determine if widget customization is needed
- The References section from PHASE.md — if references were provided, pass them to add-widget

The add-widget skill is itself a phase-based orchestrator. It will run its own internal phases:
- Phase 2 (Config): Scaffold widget boilerplate (clone template, rename, integrate)
- Phase R (References): Extract JS/UI patterns from reference repos (if references provided)
- Phase 3 (Plan): Plan widget JS implementation (if customization is needed)
- Phase 4 (Build): Write widget JS component code (if plan was created)
- Phase 5 (Check): Validate widget build

**Note:** The add-widget skill will use `AskUserQuestion` to ask the user for a detailed widget description (Step 3 of add-widget). This is expected — widgets are visually bespoke and the appgen app description alone is not detailed enough to drive the widget's UI design. The app description is passed as context, but the user will be asked for additional specifics about visual layout, data display, controls, etc.

```
Task tool with:
  subagent_type: "general-purpose"
  prompt: |
    Execute the add-widget skill to scaffold and build the JavaScript widget.

    App directory: {app-dir}
    Add-widget skill location: {path-to-this-skill}/../doover-add-widget
    Widget PascalCase: {PascalCase}
    Widget kebab-case: {kebab-case}
    Widget snake_case: {snake_case}
    Widget Title Case: {Title Case}
    App description (from appgen): {app description from PHASE.md}

    References (from Phase 1, if any):
    {reference list from PHASE.md, or "none"}

    Context: This is part of the appgen workflow. Phase 2w has already set up the
    processor side (Python files, pyproject.toml, build.sh, doover_config.json base
    structure). The add-widget skill now handles the entire JS widget side through
    its own phase-based pipeline.

    Instructions:
    1. Read the add-widget skill: {add-widget-skill-path}/SKILL.md
    2. The app directory and widget name are already known — skip Steps 1-2 of the
       add-widget orchestrator.
    3. Execute Step 3 (Gather Detailed Widget Description) — this ALWAYS runs.
       Present the app description above as context, then ask the user for
       detailed widget-specific information (visual layout, data display,
       controls, interactions). Widgets are bespoke, so the general app
       description is not enough.
    4. Skip Step 4 (Gather References) — references are provided above.
    5. Initialize WIDGET_PHASE.md (Step 6) with the provided details:
       - Set widget name variants from above
       - Set app description from above
       - Set widget description from user's response to Step 3
       - Set has_references based on whether references were provided
       - Set needs_customization based on whether user provided a detailed description
    6. Execute the add-widget phase pipeline (Step 7):
       - Phase 2 (Config): Scaffold widget boilerplate
       - Phase R (References): If references were provided
       - Phase 3 (Plan) + Phase 4 (Build): If customization is needed
       - Phase 5 (Check): Validate the build
    7. Return a summary including:
       - Widget directory created
       - doover_config.json entries added (file_deployments, deployment_channel_messages)
       - Whether widget was customized (Phases 3-4 ran)
       - Validation results (Phase 5)
       - Any errors
```

**Phase R Subagent Invocation (References) — Conditional:**

After Phase 2 completes (and add-widget completes, for widget apps), read `has_references` and `app_type` from `{app-dir}/.appgen/PHASE.md`:
- If `app_type` is `widget` → **skip Phase R entirely** (references were passed to add-widget instead)
- If `has_references` is `true` and Phase R is not yet completed → spawn Phase R subagent below
- If `has_references` is `false` or Phase R already completed → skip directly to Phase 3

```
Task tool with:
  subagent_type: "general-purpose"
  prompt: |
    Execute Phase R (References) of the doover-appgen skill.

    App directory: {app-dir}
    Skill location: {path-to-this-skill}

    Instructions:
    1. Read {app-dir}/.appgen/PHASE.md for the reference list
    2. Read {skill-path}/references/phase-r-refs.md for full instructions
    3. For each reference: resolve ambiguity, acquire source, extract patterns
    4. Write {app-dir}/.appgen/REFERENCES.md with curated patterns
    5. Cleanup any temp clone directories
    6. Update {app-dir}/.appgen/PHASE.md with completion status
    7. Return a summary including:
       - References processed and skipped
       - Aspects extracted
       - Any errors
```

**Phase 3 Subagent Invocation (Plan):**

First, read the `app_type` from `{app-dir}/.appgen/PHASE.md` to determine which Phase 3 variant to use:
- `docker` → use `phase-3d-plan.md`
- `processor` → use `phase-3p-plan.md`
- `widget` → use `phase-3w-build.md` (minimal — just confirms widget is scaffolded)
- `integration` → use `phase-3i-plan.md`

**For widget apps**, use this simplified invocation:

```
Task tool with:
  subagent_type: "general-purpose"
  prompt: |
    Execute Phase 3 (Widget Build) of the doover-appgen skill.

    App directory: {app-dir}
    Skill location: {path-to-this-skill}
    App type: widget

    Instructions:
    1. Read {app-dir}/.appgen/PHASE.md to get app name and path
    2. Read {skill-path}/references/phase-3w-build.md
    3. Confirm widget files exist (JS component, rsbuild config, processor application.py)
    4. Update {app-dir}/.appgen/PHASE.md with completion status
    5. Return a summary including:
       - Widget component path
       - Processor path
       - Confirmation that scaffold is complete
```

**For docker/processor/integration apps**, use this invocation:

```
Task tool with:
  subagent_type: "general-purpose"
  prompt: |
    Execute Phase 3 (Plan) of the doover-appgen skill.

    App directory: {app-dir}
    Skill location: {path-to-this-skill}
    Platform docs: {platform-docs-path}  # path to doover-platform-docs skill
    App type: {app_type}  # docker, processor, or integration

    Instructions:
    1. Read {app-dir}/.appgen/PHASE.md to get app name, description, and app type
    2. Based on app type, read the appropriate phase instructions:
       - docker: {skill-path}/references/phase-3d-plan.md
       - processor: {skill-path}/references/phase-3p-plan.md
       - integration: {skill-path}/references/phase-3i-plan.md
    3. If {app-dir}/.appgen/REFERENCES.md exists, read it for reference patterns
    4. Read the documentation index: {platform-docs-path}/references/index.md
    5. Based on app type, read the required documentation chunks from {platform-docs-path}/references/
    6. Analyze the app description and identify requirements
    7. If the description is ambiguous, use AskUserQuestion to clarify
    8. If external integration is needed, use WebSearch to find API documentation
    9. Create {app-dir}/.appgen/PLAN.md with the complete build plan (include Documentation Chunks section and Reference Patterns Applied section if references exist)
    10. Update {app-dir}/.appgen/PHASE.md with completion status
    11. Return a summary including:
       - What external integrations were identified
       - What questions were asked (if any)
       - What documentation was found (if any)
       - Brief summary of the plan
       - Any errors
```

**Phase 4 Subagent Invocation (Build):**

First, read the `app_type` from `{app-dir}/.appgen/PHASE.md` to determine which Phase 4 variant to use:
- `docker` → use `phase-4d-build.md`
- `processor` → use `phase-4p-build.md`
- `widget` → use `phase-4w-check.md` (validation, not build)
- `integration` → use `phase-4i-build.md`

**For widget apps**, use this validation invocation:

```
Task tool with:
  subagent_type: "general-purpose"
  prompt: |
    Execute Phase 4 (Widget Check) of the doover-appgen skill.

    App directory: {app-dir}
    Skill location: {path-to-this-skill}
    App type: widget

    Instructions:
    1. Read {app-dir}/.appgen/PHASE.md to get app name and path
    2. Read {skill-path}/references/phase-4w-check.md
    3. Run validation checks:
       - Widget builds (npm run build) and built JS file exists
       - Python imports work (application class can be imported)
       - Config export works (uv run export-config)
       - File structure is correct
       - doover_config.json has correct structure
    4. Update {app-dir}/.appgen/PHASE.md with validation results
    5. Return a summary including:
       - Checks passed
       - Checks failed (with details)
       - Recommended fixes (if any)
```

**For docker/processor/integration apps**, use this build invocation:

```
Task tool with:
  subagent_type: "general-purpose"
  prompt: |
    Execute Phase 4 (Build) of the doover-appgen skill.

    App directory: {app-dir}
    Skill location: {path-to-this-skill}
    Platform docs: {platform-docs-path}  # path to doover-platform-docs skill
    App type: {app_type}  # docker, processor, or integration

    IMPORTANT: Do NOT use AskUserQuestion. All requirements should be in PLAN.md.
    If PLAN.md is unclear, report this as an error.

    Instructions:
    1. Read {app-dir}/.appgen/PHASE.md to get app name and path
    2. Read {app-dir}/.appgen/PLAN.md for implementation details
    3. Based on app type, read the appropriate phase instructions:
       - docker: {skill-path}/references/phase-4d-build.md
       - processor: {skill-path}/references/phase-4p-build.md
       - integration: {skill-path}/references/phase-4i-build.md
    4. Read documentation chunks for code patterns:
       - Read chunks listed in PLAN.md's "Documentation Chunks" section
       - Use keyword-based discovery to find additional relevant chunks from {platform-docs-path}/references/index.md
    5. Generate application code following PLAN.md exactly
    6. If external packages needed, run: uv add {package}
    7. Run: uv run export-config
    8. Update {app-dir}/.appgen/PHASE.md with completion status
    9. Return a summary including:
       - Files created/modified
       - What the app does
       - Dependencies added
       - Any errors
```

**Phase 5 Subagent Invocation (Check):**

First, read the `app_type` from `{app-dir}/.appgen/PHASE.md` to determine which Phase 5 variant to use:
- `docker` → use `phase-5d-check.md`
- `processor` → use `phase-5p-check.md`
- `widget` → **skip Phase 5** (validation already done in Phase 4w) — proceed directly to Phase 6
- `integration` → use `phase-5i-check.md`

```
Task tool with:
  subagent_type: "general-purpose"
  prompt: |
    Execute Phase 5 (Check) of the doover-appgen skill.

    App directory: {app-dir}
    Skill location: {path-to-this-skill}
    App type: {app_type}  # docker, processor, or integration

    Instructions:
    1. Read {app-dir}/.appgen/PHASE.md to get app name and path
    2. Based on app type, read the appropriate phase instructions:
       - docker: {skill-path}/references/phase-5d-check.md
       - processor: {skill-path}/references/phase-5p-check.md
       - integration: {skill-path}/references/phase-5i-check.md
    3. Run validation checks:
       - uv sync (dependencies resolve)
       - Import check (code imports successfully)
       - doover config-schema export (config is valid)
    4. Verify file structure is correct
    5. Update {app-dir}/.appgen/PHASE.md with validation results
    6. Return a summary including:
       - Checks passed
       - Checks failed (with details)
       - Recommended fixes (if any)
       - Any errors
```

**Phase 6 Subagent Invocation (Document):**

Phase 6 is shared by all app types - there is no variant selection needed.

```
Task tool with:
  subagent_type: "general-purpose"
  prompt: |
    Execute Phase 6 (Document) of the doover-appgen skill.

    App directory: {app-dir}
    Skill location: {path-to-this-skill}

    Instructions:
    1. Read {app-dir}/.appgen/PHASE.md to get app details (name, description, type, GitHub URL, icon URL)
    2. Read {skill-path}/references/phase-6-document.md for instructions
    3. Read the application code to understand what it does:
       - {app-dir}/doover_config.json for configuration schema
       - {app-dir}/src/{app_name}/application.py for core functionality
       - {app-dir}/src/{app_name}/app_config.py for config options
       - {app-dir}/src/{app_name}/app_ui.py for UI elements (if present)
    4. Generate a comprehensive README.md following the template in phase-6-document.md
    5. Include all required sections: Overview, Features, Getting Started, Configuration, Tags/UI, How It Works, etc.
    6. Update {app-dir}/.appgen/PHASE.md with completion status
    7. Return a summary including:
       - README sections generated
       - Configuration items documented
       - Tags/UI elements documented
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
