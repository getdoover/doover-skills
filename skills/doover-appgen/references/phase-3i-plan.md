# Phase 3i: Integration Plan

Analyze requirements, resolve ambiguity, and create a build plan for the integration application.

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

2. Read the documentation index:
   `{platform-docs-path}/references/index.md`

3. Based on the index, read the required chunks for Integration apps:
   - `{platform-docs-path}/references/config-schema.md` - Configuration patterns
   - `{platform-docs-path}/references/cloud-handler.md` - Handler and event patterns
   - `{platform-docs-path}/references/cloud-project.md` - Project setup and build script
   - `{platform-docs-path}/references/integration-features.md` - Integration-specific features (IngestionEndpointConfig, ExtendedPermissionsConfig, device routing)

### Step 1b: Check for Reference Patterns

Check if `{app-directory}/.appgen/REFERENCES.md` exists:
- If yes, read it completely
- Note the extracted patterns and how they apply to this app
- These patterns should inform your design decisions in subsequent steps
- Reference relevant patterns in the Implementation Notes section of PLAN.md

### Step 2: Analyze Requirements

Based on the app description, identify:

1. **External System**: What external system will send data?
   - Note the system/service name
   - Search for API/webhook documentation using WebSearch
   - If docs not found, ask user for documentation URL or payload details

2. **Payload Format**:
   - What format is the incoming data? (JSON, protobuf, XML, etc.)
   - What fields are in the payload?
   - How is the device identified in the payload?

3. **Device Routing**:
   - How to map incoming data to Doover devices?
   - What field contains the device identifier?
   - How to match against serial_number_match patterns?

4. **Security Requirements**:
   - IP whitelist needed?
   - HMAC signature validation?
   - API key validation?

5. **Data Flow**:
   - What data comes in via webhook?
   - What processing is needed?
   - What tags should be updated on devices?

6. **Configuration Needs**:
   - What settings should be user-configurable?
   - Extended permissions pattern (which device fields to access)?

### Step 3: Resolve Ambiguity

If ANY of the following are unclear from the description, use `AskUserQuestion`:

- What external system sends the webhooks
- What the payload format looks like
- How to identify which device the data belongs to
- What security validation is required
- What data should be extracted and stored
- Any domain-specific requirements

**Do not guess.** Get clear answers before proceeding.

### Step 4: Research External Documentation

If integrating with an external service:

1. Use `WebSearch` to find official webhook/API documentation
2. Use `WebFetch` to read the documentation
3. Extract key information:
   - Webhook payload structure
   - Authentication/validation methods
   - Device identification fields
   - Sample payloads

If documentation cannot be found:
- Ask the user for the documentation URL
- Or ask the user to provide a sample payload

### Step 5: Design Configuration Schema

Based on your analysis, design the configuration schema:

1. Include `IngestionEndpointConfig` for webhook settings
2. Include `ExtendedPermissionsConfig` for multi-device access
3. Define device matching pattern (usually based on external device ID)
4. List any additional configuration fields needed
5. Determine types and defaults
6. Write descriptions for each field

### Step 6: Design Ingestion Handler

Plan the `on_ingestion_endpoint` implementation:

1. **Payload Parsing**:
   - What format? (JSON, protobuf, etc.)
   - What fields to extract?
   - Custom `parse_ingestion_event_payload` needed?

2. **Device Routing**:
   - How to get device identifier from payload?
   - How to use `get_extended_permission_agents()`?
   - How to match devices?

3. **Data Storage**:
   - What tags to update?
   - What format for stored data?

4. **Response**:
   - What to return (200 OK, error codes)?

### Step 7: Write PLAN.md

Create `{app-directory}/.appgen/PLAN.md` with the following structure:

```markdown
# Build Plan

## App Summary
- Name: {app_name}
- Type: integration
- Description: {one-line summary}

## External Integration
- Service: {name of external system sending webhooks}
- Documentation: {URL or "N/A"}
- Webhook Format: {JSON, protobuf, etc.}

## Data Flow
- Inputs: {webhook from external system}
- Processing: {parse payload, route to device, extract data}
- Outputs: {tags updated on matched devices}

## Payload Structure
```json
{
  "device_id": "...",
  "field1": "...",
  "field2": "..."
}
```

## Device Routing
- Identifier field: {field in payload that identifies device}
- Match pattern: {how to match against serial_number_match}
- Extended permissions: {device field pattern, e.g., "device_id_*"}

## Security
- IP Whitelist: {yes/no, IPs if yes}
- HMAC Validation: {yes/no, details if yes}
- Other: {any other validation}

## Configuration Schema
| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| {field_name} | {type} | {yes/no} | {value} | {description} |

## Tags (Output)
| Tag Name | Type | Description |
|----------|------|-------------|
| {name} | {type} | {what data it stores} |

## Documentation Chunks

### Required Chunks
- `config-schema.md` - Configuration types and patterns
- `cloud-handler.md` - Handler and event patterns
- `cloud-project.md` - Project setup and build script
- `integration-features.md` - IngestionEndpointConfig, ExtendedPermissionsConfig, device routing

### Recommended Chunks
{List additional chunks based on app requirements}
- `tags-channels.md` - If uses tags or channels

### Discovery Keywords
{List keywords for Build phase to auto-discover additional chunks}
Example: ingestion, payload, parse, device routing, hmac, cidr

## Reference Patterns Applied
(Include this section only if REFERENCES.md was present)
| Pattern | Source Aspect | How Applied |
|---------|--------------|-------------|
| {name} | {what was extracted} | {how it influences this plan} |

## Implementation Notes
- {Key patterns to follow}
- {Payload parsing approach}
- {External packages needed, if any}
- {Special considerations}
```

### Step 8: Update State

Update `.appgen/PHASE.md`:
- Set current phase to "Phase 3 - Integration Plan"
- Set status to "completed"
- Add Phase 3 to completed phases list
- Note that PLAN.md was created

## Completion Criteria

Phase 3 is complete when:
- [ ] Mini-docs read and patterns understood
- [ ] App description analyzed for requirements
- [ ] All ambiguity resolved (via user questions or documentation)
- [ ] External documentation found or gathered (if needed)
- [ ] Payload structure understood or documented
- [ ] PLAN.md created with complete implementation details
- [ ] No open questions remain
- [ ] `.appgen/PHASE.md` updated with status "completed"

## Summary for Orchestrator

When returning to the orchestrator, include:
- **Status**: success or failure
- **External system**: What sends webhooks to this integration
- **Payload format**: Data format and key fields
- **Questions asked**: What clarifications were needed from user
- **Documentation found**: What external docs were referenced
- **Plan summary**: Brief overview of what will be built
- **Errors**: Any issues encountered
