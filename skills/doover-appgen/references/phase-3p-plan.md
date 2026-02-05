# Phase 3p: Processor Plan

Analyze requirements, resolve ambiguity, and create a build plan for the processor application.

## Phase Objective

Analyze the app description, resolve all ambiguity, gather missing information via external documentation or user questions, and produce a clear PLAN.md that the Build phase can execute without guesswork.

## Execution Context

This phase runs as a subagent spawned by the orchestrator (SKILL.md). You have full access to:
- `Read` tool for reading documentation and existing files
- `Write` / `Edit` tools for creating/modifying files
- `Bash` tool for running commands
- `AskUserQuestion` tool for clarifying requirements
- `WebSearch` / `WebFetch` tools for finding external API documentation

Your job: Analyze, clarify, plan, then return a summary.

## Prerequisites

- Phase 2 must be completed (app directory exists with configured template)
- `.appgen/PHASE.md` contains app name, description, and type

## Steps

### Step 1: Read State and Documentation Index

1. Read `{app-directory}/.appgen/PHASE.md` to get:
   - App name
   - App description
   - App directory path
   - Has UI flag

2. Read the documentation index:
   `references/mini-docs/index.md`

3. Based on the index, read the required chunks for Processor apps:
   - `references/mini-docs/config-schema.md` - Configuration patterns
   - `references/mini-docs/cloud-handler.md` - Handler and event patterns
   - `references/mini-docs/cloud-project.md` - Project setup and build script
   - `references/mini-docs/processor-features.md` - Processor-specific features (ManySubscriptionConfig, ScheduleConfig, UI management, connection status)

4. If has_ui is true, also read:
   - `references/mini-docs/docker-ui.md` - UI component patterns (same for processors)

### Step 2: Analyze Requirements

Based on the app description, identify:

1. **External Integration**: Does this integrate with an external service/API?
   - If yes, note the service name
   - Search for API documentation using WebSearch
   - If docs not found, ask user for documentation URL or key details

2. **Event Triggers**:
   - What events should trigger this processor?
   - Channel messages (on_message_create)?
   - Schedule (on_schedule)?
   - Both?

3. **Data Flow**:
   - What channels should it subscribe to?
   - What data comes in from those channels?
   - What processing is needed?
   - What data goes out? (tags, channel messages, API calls)

4. **Configuration Needs**:
   - What settings should be user-configurable?
   - What channel subscriptions are needed?
   - What schedule triggers are needed?
   - What are sensible defaults?

5. **UI Elements** (if has_ui is true):
   - What Variables should display processor state on devices?

### Step 3: Resolve Ambiguity

If ANY of the following are unclear from the description, use `AskUserQuestion`:

- What external service/API to integrate with
- What channels to subscribe to
- What events should trigger processing
- What schedule intervals are needed
- What data format to expect or produce
- What specific configuration options are needed
- Any domain-specific requirements

**Do not guess.** Get clear answers before proceeding.

### Step 4: Research External Documentation

If integrating with an external service:

1. Use `WebSearch` to find official API documentation
2. Use `WebFetch` to read the documentation
3. Extract key information:
   - Authentication method (API key, OAuth, etc.)
   - Relevant API endpoints
   - Request/response formats
   - Rate limits or quotas

If documentation cannot be found:
- Ask the user for the documentation URL
- Or ask the user to describe the key API details

### Step 5: Design Configuration Schema

Based on your analysis, design the configuration schema:

1. List all configuration fields needed
2. Include `ManySubscriptionConfig` for channel subscriptions
3. Include `ScheduleConfig` if scheduled triggers needed
4. Determine types (string, integer, boolean, array, object)
5. Identify required vs optional fields
6. Set sensible defaults where appropriate
7. Write descriptions for each field

### Step 6: Design Event Handlers

Determine which event handlers are needed:

1. **on_message_create** - If responding to channel messages
   - What channels?
   - What message types?
   - What processing logic?

2. **on_schedule** - If running on a schedule
   - What interval?
   - What periodic tasks?

3. **setup/close** - Initialization and cleanup
   - What resources need initialization?
   - What cleanup is needed?

### Step 7: Write PLAN.md

Create `{app-directory}/.appgen/PLAN.md` with the following structure:

```markdown
# Build Plan

## App Summary
- Name: {app_name}
- Type: processor
- Description: {one-line summary}

## External Integration
- Service: {name of external service/API, or "None"}
- Documentation: {URL or "N/A"}
- Authentication: {method - API key, OAuth, etc., or "N/A"}

## Data Flow
- Inputs: {channels, schedules, or other triggers}
- Processing: {what the processor does with the data}
- Outputs: {tags, channel messages, API calls}

## Configuration Schema
| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| {field_name} | {type} | {yes/no} | {value} | {description} |

### Subscriptions
- Channel pattern: {pattern, e.g., "cmds"}
- Message types: {types to listen for}

### Schedule (if applicable)
- Interval: {cron expression or interval}
- Purpose: {what runs on schedule}

## Event Handlers
| Handler | Trigger | Description |
|---------|---------|-------------|
| on_message_create | Channel message | {what it does} |
| on_schedule | {interval} | {what it does} |

## Tags (Output)
| Tag Name | Type | Description |
|----------|------|-------------|
| {name} | {type} | {what it contains} |

## UI Elements (if has_ui)

### Variables (Display on Device)
| Name | Type | Description |
|------|------|-------------|
| {name} | {type} | {what it displays} |

## Documentation Chunks

### Required Chunks
- `config-schema.md` - Configuration types and patterns
- `cloud-handler.md` - Handler and event patterns
- `cloud-project.md` - Project setup and build script
- `processor-features.md` - ManySubscriptionConfig, ScheduleConfig, UI management

### Recommended Chunks
{List additional chunks based on app requirements}
- `docker-ui.md` - If has UI components
- `tags-channels.md` - If uses tags or channels

### Discovery Keywords
{List keywords for Build phase to auto-discover additional chunks}
Example: subscription, schedule, cron, ui_manager, push_async, connection

## Implementation Notes
- {Key patterns to follow}
- {External packages needed, if any}
- {Special considerations}
```

### Step 8: Update State

Update `.appgen/PHASE.md`:
- Set current phase to "Phase 3 - Processor Plan"
- Set status to "completed"
- Add Phase 3 to completed phases list
- Note that PLAN.md was created

## Completion Criteria

Phase 3 is complete when:
- [ ] Mini-docs read and patterns understood
- [ ] App description analyzed for requirements
- [ ] All ambiguity resolved (via user questions or documentation)
- [ ] External documentation found or gathered (if needed)
- [ ] PLAN.md created with complete implementation details
- [ ] No open questions remain
- [ ] `.appgen/PHASE.md` updated with status "completed"

## Summary for Orchestrator

When returning to the orchestrator, include:
- **Status**: success or failure
- **External integration**: What service/API will be integrated (if any)
- **Event handlers**: What triggers the processor
- **Questions asked**: What clarifications were needed from user
- **Documentation found**: What external docs were referenced
- **Plan summary**: Brief overview of what will be built
- **Errors**: Any issues encountered
