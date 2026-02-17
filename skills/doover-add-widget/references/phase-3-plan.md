# Phase 3: Widget Plan

Analyze requirements, load platform documentation, and create a build plan for the widget's JavaScript component.

## Phase Objective

Analyze what the widget should do, resolve all ambiguity, load the Doover widget platform documentation, and produce a clear `WIDGET_PLAN.md` that the Build phase can execute without guesswork.

## Execution Context

This phase runs as a subagent spawned by the orchestrator (SKILL.md). You have full access to:
- `Read` tool for reading documentation and existing files
- `Write` / `Edit` tools for creating WIDGET_PLAN.md
- `AskUserQuestion` tool for clarifying requirements
- `WebSearch` / `WebFetch` tools for finding external library documentation

Your job: Analyze, clarify, plan, then return a summary.

## Prerequisites

- Phase 2 must be completed (widget is scaffolded)
- `.appgen/WIDGET_PHASE.md` contains widget name and description

## Steps

### Step 1: Read State and Platform Documentation

1. Read `{app-dir}/.appgen/WIDGET_PHASE.md` to get:
   - Widget name (PascalCase, kebab-case, snake_case)
   - App description (high-level, from appgen — may be "N/A" if standalone)
   - Widget description (detailed, from user — describes visual layout, behavior, interactions)
   - App directory path

2. Read the widget platform documentation:
   - `{platform-docs-path}/references/widget-architecture.md` — 3-layer component pattern, Module Federation config, shared modules from `customer_site`, ConcatenatePlugin, Tailwind CSS styling conventions
   - `{platform-docs-path}/references/widget-hooks.md` — Platform hooks (`useAgent`, `useAgentChannel`, `useChannelUpdateAggregate`, `useChannelSendMessage`), `dataProvider` methods, `useParams`/`useRemoteParams`, permissions, WebSocket behavior, common read/write patterns

3. Read the current widget component to understand the starting point:
   - `{app-dir}/{kebab-case}/src/{PascalCase}.js` — The scaffolded 3-layer component from the template

### Step 1b: Check for Reference Patterns

Check if `{app-dir}/.appgen/WIDGET_REFERENCES.md` exists:
- If yes, read it completely
- Note the extracted JS/UI patterns and how they apply to this widget
- These patterns should inform your design decisions in subsequent steps
- Reference relevant patterns in the WIDGET_PLAN.md

### Step 2: Analyze Requirements

Based on the widget/app description, identify:

1. **Data Sources**: What channels does this widget need to read?
   - Channel names and expected data structure
   - Real-time updates needed? (`useAgentChannel` with WebSocket auto-subscription)
   - Historical data needed? (`useChannelMessages`, `getDataSeries`)
   - Agent info needed? (`useAgent`)

2. **User Interactions**: What can the user do in this widget?
   - Write to channels? (`useChannelUpdateAggregate`)
   - Send commands? (`useChannelSendMessage`, `useAgentSendUiCmd`)
   - Permission requirements? (`useHasAgentPermission` with `AGENT_PERMISSION`)
   - Form inputs, buttons, toggles, selections?

3. **Visual Layout**: What does the widget look like?
   - What information sections are displayed?
   - Tables, charts, forms, status indicators, cards?
   - Responsive design considerations?

4. **External Libraries**: Does the widget need additional npm packages?
   - Charting (recharts, chart.js)?
   - Date handling (date-fns, dayjs)?
   - Other UI utilities?
   - Note: these must work as non-shared deps within Module Federation

### Step 3: Resolve Ambiguity

If ANY of the following are unclear from the description, use `AskUserQuestion`:

- What channels to read from or write to
- What data fields to display and in what format
- What user interactions are needed
- What visual layout or component arrangement is expected
- Any domain-specific terminology or concepts

**Do not guess.** Get clear answers before proceeding.

### Step 4: Research External Libraries (if needed)

If the widget needs charting, complex data processing, or other capabilities beyond what the platform provides:

1. Use `WebSearch` to find appropriate React-compatible libraries
2. Verify they work as standard npm dependencies (not shared via Module Federation)
3. Note installation commands for Phase 4
4. Note import patterns and key API usage

### Step 5: Design Component Structure

Based on your analysis, design the widget's component structure:

1. **Hooks needed**: List all platform hooks and their parameters
   - Which hooks go in Layer 2 (hooks wrapper) vs Layer 3 (inner component)
   - `useRemoteParams()` is always in Layer 2
   - `useAgent()` with loading guard is always in Layer 2
   - Data hooks (`useAgentChannel`) can be in Layer 2 or Layer 3
   - `useParams()` from `react-router` is used in Layer 3

2. **State management**: Local state (`useState`) for UI interactions
   - Form input values
   - Toggle/selection states
   - Derived/computed display values

3. **Component hierarchy**: If the widget is complex (>150 lines in inner component), plan sub-components
   - Each sub-component in its own file under `{kebab-case}/src/components/`
   - Import from the main component file

4. **Styling approach**: Tailwind utility classes
   - Layout structure (flex, grid)
   - Spacing and sizing
   - Colors using platform CSS variables where appropriate
   - Note: Shadcn components are NOT available via Module Federation — replicate with Tailwind

### Step 6: Write WIDGET_PLAN.md

Create `{app-dir}/.appgen/WIDGET_PLAN.md` with the following structure:

```markdown
# Widget Build Plan

## Widget Summary
- Name: {PascalCase}
- Folder: {kebab-case}
- Description: {what the widget does}

## Data Flow

### Channels Read
| Channel | Hook | Data Fields | Purpose |
|---------|------|-------------|---------|
| {name} | useAgentChannel | {fields} | {why} |

### Channels Written
| Channel | Hook | Data Fields | Purpose |
|---------|------|-------------|---------|
| {name} | useChannelUpdateAggregate | {fields} | {why} |

### Permissions Required
| Permission | Constant | Purpose |
|------------|----------|---------|
| {name} | AGENT_PERMISSION.{X} | {why} |

## Visual Layout

{Description of the widget's visual structure}

### Component Tree
```
{PascalCase}Inner
├── {SubComponent1}
├── {SubComponent2}
└── ...
```

### Layout Description
- {Section 1}: {what it contains, layout approach}
- {Section 2}: {what it contains, layout approach}

## Hooks Summary

### Layer 2 (Hooks Wrapper)
- `useRemoteParams()` → `agentId`
- `useAgent(agentId)` → loading guard
- {other hooks if needed at this level}

### Layer 3 (Inner Component)
- `useParams()` → `agentId`
- {useAgentChannel calls}
- {mutation hook calls}
- {permission hook calls}

## External Dependencies
| Package | Version | Purpose |
|---------|---------|---------|
| {name} | {ver} | {why} |

(Omit this section if no external dependencies are needed.)

## Implementation Notes
- {Key patterns from platform docs to follow}
- {Styling conventions}
- {Loading/error state handling approach}
- {Any special considerations}

## Reference Patterns Applied
(Include this section only if WIDGET_REFERENCES.md was present)
| Pattern | Source | How Applied |
|---------|--------|-------------|
| {name} | {ref} | {how it influences this widget} |
```

### Step 7: Update State

Update `.appgen/WIDGET_PHASE.md`:
- Set current phase to "Phase 3 - Plan"
- Set status to "completed"
- Add Phase 3 to completed phases list
- Note that WIDGET_PLAN.md was created

## Completion Criteria

Phase 3 is complete when:
- [ ] Widget platform documentation loaded and understood
- [ ] Widget requirements analyzed
- [ ] All ambiguity resolved (via user questions or documentation)
- [ ] External library research done (if needed)
- [ ] WIDGET_PLAN.md created with complete component design
- [ ] No open questions remain
- [ ] `.appgen/WIDGET_PHASE.md` updated with status "completed"

## Summary for Orchestrator

When returning to the orchestrator, include:
- **Status**: success or failure
- **Data channels**: What channels the widget reads/writes
- **User interactions**: What controls are planned
- **External deps**: Any npm packages needed
- **Questions asked**: What clarifications were needed from user
- **Plan summary**: Brief overview of what will be built
- **Errors**: Any issues encountered
