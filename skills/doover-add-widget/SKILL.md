---
name: doover-add-widget
description: Add a Doover widget to an existing app. Use when users want to add a remote UI component/widget to their Doover application, scaffold a new widget from the widget template, or need help setting up Module Federation widgets.
user_invocable: true
---

# Add Widget to Doover App

Add a widget (remote UI component) to an existing Doover application. This skill scaffolds a widget from the template and optionally customizes it through a phase-based workflow.

## Overview

This skill guides the process of adding a widget to a Doover app step-by-step. Each phase handles a specific part of the process, with clear completion criteria before proceeding to the next.

**Workflow:**
1. Determine app directory and widget name (orchestrator)
2. Scaffold widget boilerplate from template (Phase 2)
3. Extract JS/UI patterns from reference repos (Phase R, optional)
4. Plan widget JS implementation (Phase 3, optional)
5. Build widget JS component code (Phase 4, optional)
6. Validate widget build (Phase 5)

Phases R, 3, and 4 run when the widget needs customization beyond the template boilerplate — either because a widget description was provided (describing what the widget should do) or because reference repos were supplied. When just adding a blank widget scaffold, only Phases 2 and 5 run.

## Skill Directory Structure

```
skills/doover-add-widget/
├── SKILL.md                      # This file — orchestrator and routing
└── references/
    ├── phase-2-config.md         # Phase 2: Scaffold widget boilerplate
    ├── phase-r-refs.md           # Phase R: Extract JS/UI reference patterns (optional)
    ├── phase-3-plan.md           # Phase 3: Plan widget JS implementation
    ├── phase-4-build.md          # Phase 4: Build widget JS code
    └── phase-5-check.md          # Phase 5: Validate widget build
```

**External dependency:** Platform documentation lives in `skills/doover-platform-docs/` (the `doover-platform-docs` skill). Subagents are given `{platform-docs-path}` to locate widget-specific docs (`widget-architecture.md`, `widget-hooks.md`).

Phase files are loaded on-demand. Do not read all phases upfront.

## State Storage

State is stored in the **app directory**, not the skill directory:

```
{app-dir}/
└── .appgen/
    ├── WIDGET_PHASE.md       # Widget phase state, name variants, references
    ├── WIDGET_REFERENCES.md  # JS/UI reference patterns (created in Phase R, optional)
    └── WIDGET_PLAN.md        # Widget build plan (created in Phase 3)
```

The `.appgen/WIDGET_PHASE.md` file tracks:
- Widget name variants (PascalCase, kebab-case, snake_case, Title Case)
- Widget/app description
- Current phase and status
- Completed phases
- Reference list (if any)

These state files are separate from the appgen state files (`PHASE.md`, `PLAN.md`, `REFERENCES.md`) to avoid conflicts when add-widget is invoked as part of the appgen workflow.

## Name Conventions

The widget name is derived in four formats:

| Format | Example | Used For |
|--------|---------|----------|
| PascalCase | `MyCustomWidget` | MF scope, JS filename, component function |
| kebab-case | `my-custom-widget` | Folder name, npm package name |
| snake_case | `my_custom_widget` | doover componentUrl, file deployment name |
| Title Case | `My Custom Widget` | Display name |

The rename script in the widget template accepts any format and derives all four.

## Available Phases

| Phase | Name | Description |
|-------|------|-------------|
| 2 | Config | Scaffold widget boilerplate (clone template, rename, integrate) |
| R | References | Extract JS/UI patterns from reference repos (optional) |
| 3 | Plan | Analyze requirements, load widget platform docs, create WIDGET_PLAN.md |
| 4 | Build | Write widget JS component code from WIDGET_PLAN.md |
| 5 | Check | Validate widget build and configuration |

## Phase Gating Protocol

**Rules:**
1. Each phase must complete before the next begins
2. Check `.appgen/WIDGET_PHASE.md` status before loading next phase
3. Phase R is optional — only runs if references are provided
4. Phases 3-4 are optional — only run if widget customization is requested
5. Phase 5 always runs as the final phase
6. Never skip phases or run phases out of order

## Phase Execution Protocol

Each phase runs in an isolated subagent to manage context efficiently.

### Orchestrator Responsibilities (this file)

1. Determine app directory and widget name
2. Determine customization scope (blank scaffold vs custom widget)
3. Initialize `.appgen/WIDGET_PHASE.md` with widget state
4. Spawn a Task subagent for each phase
5. Report the subagent's summary to the user
6. Check completion and spawn next phase

### Subagent Responsibilities

1. Read the phase instructions from `references/phase-{N}-*.md`
2. Read existing state from `.appgen/WIDGET_PHASE.md`
3. Execute the phase steps
4. Update `.appgen/WIDGET_PHASE.md` with results
5. Return a summary to the orchestrator

## Prerequisites

- An existing Doover app directory with a `doover_config.json`
- GitHub CLI (`gh`) authenticated
- Node.js 18+ available

## Execution Instructions

**When this skill is invoked:**

### Step 1: Determine App Directory

**From appgen:** App directory is provided in the prompt → use it directly, skip the question.

**Standalone:** Check if the user specified a path in their prompt. If not, use `AskUserQuestion`:
- "Where is the Doover app you want to add a widget to?"
- Options: "Current directory" / "I'll provide a path"
- If "I'll provide a path": ask for the absolute path to the app directory

**Verify** that `doover_config.json` exists in the resolved directory. If it does not exist, STOP and tell the user this must be run from within a Doover app directory that has a `doover_config.json`.

Store this as `{app-dir}`.

### Step 2: Determine Widget Name

**From appgen:** Widget name variants are provided in the prompt → use them directly, skip the question.

**Standalone:** Check if the user provided a widget name in their prompt (e.g., "add a widget called MyDashboard"). If not, use `AskUserQuestion`:
- "What would you like to name your widget?"

Accept any naming format (PascalCase, kebab-case, snake_case, or space-separated words).

Derive all name variants using these rules:
- **Split words**: Insert boundary before uppercase letters, replace non-alphanumeric with spaces, split on whitespace
- **PascalCase**: Capitalize first letter of each word, join with no separator
- **kebab-case**: Lowercase all words, join with hyphens
- **snake_case**: Lowercase all words, join with underscores
- **Title Case**: Capitalize first letter of each word, join with spaces

Report the derived names to the user before proceeding:
```
Widget name:
  PascalCase:  {PascalCase}
  kebab-case:  {kebab-case}
  snake_case:  {snake_case}
  Title Case:  {Title Case}
```

### Step 3: Gather Detailed Widget Description

**This step always runs**, regardless of whether the skill was invoked from appgen or standalone. Widgets are visually bespoke — the appgen app description alone is rarely detailed enough to drive the widget's UI design. This step gathers the specifics.

**From appgen (app description available):**

Present the existing app description to the user and ask for widget-specific detail using `AskUserQuestion`:

> "The app description from the project setup is:
>
> *{app description from PHASE.md}*
>
> Widgets need more visual and behavioral detail than a general app description provides. Please describe how this widget should look and behave — for example:
> - What data should be displayed and how (tables, charts, status cards, etc.)?
> - What user controls are needed (buttons, forms, toggles, etc.)?
> - What layout or visual arrangement do you have in mind?
> - Any specific channels, data fields, or interactions?"
>
> Options: "I'll describe it" / "Just use a blank scaffold for now"

If user selects "I'll describe it", they provide free-text input. If they select "blank scaffold", set `needs_customization: false`.

**Standalone (no app description):**

Use `AskUserQuestion`:

> "Widgets are bespoke UI components — describe in detail how this widget should look and behave. For example:
> - What data should it display and how (tables, charts, status indicators, etc.)?
> - What user controls or interactions are needed?
> - What layout or visual arrangement do you have in mind?
> - What channels or data sources should it use?"
>
> Options: "I'll describe it" / "Just add a blank scaffold"

If user selects "I'll describe it", they provide free-text input. If they select "blank scaffold", set `needs_customization: false`.

**After this step:**
- If the user provided a detailed description → store as `widget_description`, set `needs_customization: true`
- If the user chose blank scaffold → set `widget_description` to "blank scaffold", set `needs_customization: false`

### Step 4: Gather References (Standalone Only)

**From appgen:** References are passed in the prompt (or "none") → skip this step.

**Standalone:** Use `AskUserQuestion`:
- "Do you have any references you'd like to draw from? These can be code repos (for UI patterns) or images (mockups, screenshots, design references, or image assets to include in the widget)."
- Options: "No references needed" / "Yes, I have references"

If yes → enter the reference-gathering loop:

**Reference-gathering loop (repeat until user says no more):**

1. Use `AskUserQuestion` with **two questions** in a single call:
   - **Question 1:** "What is the reference?" (header: "Source") — options: "Local file or directory path" / "GitHub org/repo" / "Image URL"
   - **Question 2:** "What should be extracted or how should this be used?" (header: "Extract") — options: "UI layout/component pattern" / "Data access/hook pattern" / "Visual design reference (image)" / "Image asset to include in widget" (user can type their own via Other)

2. Record the answers as a numbered reference (Reference 1, Reference 2, etc.) with the reference type:
   - `repo` — code repository (local dir or GitHub)
   - `image` — design mockup, screenshot, or visual reference
   - `asset` — image file the user wants included in the widget itself

3. Use `AskUserQuestion` with **one question**:
   - "Would you like to add another reference?" (header: "More refs?")
   - Options: "No, that's all" / "Yes, add another"

4. If "Yes, add another" → go back to step 1
5. If "No, that's all" → exit loop, set `has_references: true`, proceed

### Step 5: Resolve Platform Docs Path

The platform documentation lives in the `doover-platform-docs` skill, located relative to this skill:
- `{platform-docs-path}` = `{this-skill-path}/../doover-platform-docs`
- Verify the path exists (check for `{platform-docs-path}/references/index.md`)
- This path is passed to Phase 3 and Phase 4 subagents

### Step 6: Initialize State

Create `{app-dir}/.appgen/` directory if it doesn't exist.

Create `{app-dir}/.appgen/WIDGET_PHASE.md`:

```markdown
# Widget Phase State

## Widget Details
- PascalCase: {PascalCase}
- kebab-case: {kebab-case}
- snake_case: {snake_case}
- Title Case: {Title Case}
- App directory: {app-dir}

## App Description
{app description from appgen PHASE.md, or "N/A" if standalone}

## Widget Description
{detailed widget description from Step 3, or "blank scaffold"}

## Current Phase
- Phase: Phase 2 - Config
- Status: pending

## Completed Phases
(none yet)

## Customization
- has_references: {true/false}
- needs_customization: {true/false}

## References
{If has_references is true:}
### Reference List
1. Location: {path, github org/repo, or image URL}
   Type: {repo|image|asset}
   Extract: {what to extract, or how to use the image}

(Repeat for each reference collected.)
```

### Step 7: Spawn Phase Subagents

Execute phases sequentially using the Task tool. After each phase, check `WIDGET_PHASE.md` to confirm completion before proceeding.

**Phase flow:**
- Always: Phase 2 (Config) — scaffold boilerplate
- If `has_references`: Phase R (References) — extract JS/UI patterns
- If `needs_customization`: Phase 3 (Plan) → Phase 4 (Build) — design and implement
- Always: Phase 5 (Check) — validate build

---

## Subagent Invocation Templates

**Phase 2 Subagent (Config):**

```
Task tool with:
  subagent_type: "general-purpose"
  prompt: |
    Execute Phase 2 (Config) of the add-widget skill.

    App directory: {app-dir}
    Skill location: {path-to-this-skill}
    Widget PascalCase: {PascalCase}
    Widget kebab-case: {kebab-case}
    Widget snake_case: {snake_case}

    Instructions:
    1. Read {skill-path}/references/phase-2-config.md
    2. Read {app-dir}/.appgen/WIDGET_PHASE.md for widget details
    3. Follow the scaffolding steps:
       - Clone getdoover/widget-template into {app-dir}/_widget-tmp
       - Run rename.js with {PascalCase}
       - Move widget/ folder to {app-dir}/{kebab-case}/
       - Delete _widget-tmp
       - Update doover_config.json with file_deployments + deployment_channel_messages
       - Update .gitignore with widget patterns
       - Run npm install in {app-dir}/{kebab-case}/
    4. Update {app-dir}/.appgen/WIDGET_PHASE.md with completion status
    5. Return a summary including:
       - Widget directory created
       - doover_config.json entries added
       - npm install result
       - Any errors
```

**Phase R Subagent (References — Conditional):**

After Phase 2, read `has_references` from `WIDGET_PHASE.md`:
- If `true` → spawn Phase R subagent below
- If `false` → skip to Phase 3 check

```
Task tool with:
  subagent_type: "general-purpose"
  prompt: |
    Execute Phase R (References) of the add-widget skill.

    App directory: {app-dir}
    Skill location: {path-to-this-skill}

    Instructions:
    1. Read {app-dir}/.appgen/WIDGET_PHASE.md for the reference list
    2. Read {skill-path}/references/phase-r-refs.md for full instructions
    3. For each reference: resolve ambiguity, acquire source, extract JS/UI patterns only
    4. Write {app-dir}/.appgen/WIDGET_REFERENCES.md with curated patterns
    5. Cleanup any temp clone directories
    6. Update {app-dir}/.appgen/WIDGET_PHASE.md with completion status
    7. Return a summary including:
       - References processed and skipped
       - JS/UI aspects extracted
       - Any errors
```

**Phase 3 Subagent (Plan — Conditional):**

After Phase R (or Phase 2 if no references), check `needs_customization` from `WIDGET_PHASE.md`:
- If `true` → spawn Phase 3 subagent below
- If `false` → skip to Phase 5

```
Task tool with:
  subagent_type: "general-purpose"
  prompt: |
    Execute Phase 3 (Plan) of the add-widget skill.

    App directory: {app-dir}
    Skill location: {path-to-this-skill}
    Platform docs: {platform-docs-path}

    Instructions:
    1. Read {app-dir}/.appgen/WIDGET_PHASE.md for widget details and description
    2. Read {skill-path}/references/phase-3-plan.md for full instructions
    3. Load widget platform documentation:
       - {platform-docs-path}/references/widget-architecture.md
       - {platform-docs-path}/references/widget-hooks.md
    4. Read the current widget component: {app-dir}/{kebab-case}/src/{PascalCase}.js
    5. If {app-dir}/.appgen/WIDGET_REFERENCES.md exists, read it for reference patterns
    6. Analyze requirements and resolve ambiguity via AskUserQuestion
    7. Create {app-dir}/.appgen/WIDGET_PLAN.md with complete widget design
    8. Update {app-dir}/.appgen/WIDGET_PHASE.md with completion status
    9. Return a summary including:
       - Data channels planned
       - User interactions planned
       - External deps needed
       - Questions asked
       - Brief plan summary
```

**Phase 4 Subagent (Build — Conditional):**

Only spawn if Phase 3 completed (WIDGET_PLAN.md exists).

```
Task tool with:
  subagent_type: "general-purpose"
  prompt: |
    Execute Phase 4 (Build) of the add-widget skill.

    App directory: {app-dir}
    Skill location: {path-to-this-skill}
    Platform docs: {platform-docs-path}

    IMPORTANT: Do NOT use AskUserQuestion. All requirements are in WIDGET_PLAN.md.
    If the plan is unclear, report this as an error.

    Instructions:
    1. Read {app-dir}/.appgen/WIDGET_PHASE.md for widget name and paths
    2. Read {app-dir}/.appgen/WIDGET_PLAN.md for implementation details
    3. Read {skill-path}/references/phase-4-build.md for full instructions
    4. Load widget platform documentation for correct patterns:
       - {platform-docs-path}/references/widget-architecture.md
       - {platform-docs-path}/references/widget-hooks.md
    5. Install external dependencies if listed in WIDGET_PLAN.md
    6. Write widget JS component code following the plan exactly
    7. Update {app-dir}/.appgen/WIDGET_PHASE.md with completion status
    8. Return a summary including:
       - Files created/modified
       - Hooks used
       - External deps installed
       - Component structure
       - Any errors
```

**Phase 5 Subagent (Check):**

Always runs as the final phase.

```
Task tool with:
  subagent_type: "general-purpose"
  prompt: |
    Execute Phase 5 (Check) of the add-widget skill.

    App directory: {app-dir}
    Skill location: {path-to-this-skill}

    Instructions:
    1. Read {app-dir}/.appgen/WIDGET_PHASE.md for widget name and paths
    2. Read {skill-path}/references/phase-5-check.md for full instructions
    3. Run validation checks:
       - npm run build in {app-dir}/{kebab-case}/
       - Verify assets/{PascalCase}.js exists
       - Verify doover_config.json entries
       - Verify file structure
    4. Update {app-dir}/.appgen/WIDGET_PHASE.md with validation results
    5. Return a summary including:
       - Checks passed
       - Checks failed (with details)
       - Recommended fixes (if any)
```

### Step 8: Report Summary

After all phases complete, report to the user:

```
Widget "{PascalCase}" added successfully.

Files created:
  {kebab-case}/                    — widget source directory
  {kebab-case}/rsbuild.config.ts   — RSBuild + Module Federation config
  {kebab-case}/ConcatenatePlugin.ts — bundles output into single JS file
  {kebab-case}/package.json        — dependencies
  {kebab-case}/src/{PascalCase}.js — widget component (edit this)
  assets/{PascalCase}.js           — built widget output

Config updated:
  doover_config.json               — file deployment + ui_state entry added

Validation:
  {Phase 5 results summary}

Next steps:
  Edit {kebab-case}/src/{PascalCase}.js to customize your widget
  cd {kebab-case} && npm run build
```

---

## Error Handling

If a subagent reports failure:
1. Report the error to the user
2. The phase status in `WIDGET_PHASE.md` should remain "in_progress"
3. Suggest the user can re-invoke the skill to retry
4. Re-invoking will read `WIDGET_PHASE.md` and resume from the current phase

---

## Doover 2.0: Processor Must Push ui_state

**Important:** The `deployment_channel_messages` field in `doover_config.json` is a **Doover 1.x feature only**. In Doover 2.0, this field is not processed. For widgets to appear in the UI interpreter, the app's processor must programmatically push `ui_state` on deployment.

The processor should use pydoover's `Application` class (`pydoover>=0.4.18`) with `RemoteComponent`:

```python
from pydoover.cloud.processor import Application
from pydoover.ui import RemoteComponent

WIDGET_NAME = "{PascalCase}"
FILE_CHANNEL = "{snake_case}"

class MyApp(Application):
    async def setup(self):
        self.ui_manager.set_children([
            RemoteComponent(
                name=WIDGET_NAME,
                display_name=WIDGET_NAME,
                component_url=FILE_CHANNEL,
            ),
        ])

    async def on_aggregate_update(self, event):
        await self.ui_manager.push_async(even_if_empty=True)
```

Key points:
- `component_url` must match the `file_deployments` channel name (snake_case)
- `deployment_channel_messages` can be kept for Doover 1.x backward compat, but the processor is authoritative for Doover 2.0
- If the app already has a processor with an `Application` subclass, add the `RemoteComponent` to the existing `setup()` method
- When invoked from appgen, Phase 2w already handles the processor side — this note is for standalone use

---

## Reference: App Directory Structure After Adding Widget

```
{app-dir}/
├── doover_config.json          (updated with widget entries)
├── .appgen/
│   ├── WIDGET_PHASE.md         (widget phase tracking)
│   ├── WIDGET_REFERENCES.md    (if Phase R ran)
│   └── WIDGET_PLAN.md          (if Phase 3 ran)
├── {kebab-case}/               (widget folder)
│   ├── ConcatenatePlugin.ts
│   ├── package.json
│   ├── rsbuild.config.ts
│   └── src/
│       └── {PascalCase}.js     (widget component)
├── assets/                     (created on first build)
│   └── {PascalCase}.js         (built output)
└── ... (other app files)
```
