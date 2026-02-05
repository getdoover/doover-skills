# Phase 6: Document

Generate comprehensive README.md documentation for the application.

## Phase Objective

Create a professional, comprehensive README.md file that documents the application's purpose, configuration, usage, and development information.

## Execution Context

This phase runs as a subagent spawned by the orchestrator (SKILL.md). You have full access to:
- `Read` tool for reading existing files and code
- `Write` / `Edit` tools for creating/modifying the README
- `Bash` tool for running commands if needed

Your job: Generate the README.md, then return a summary.

## Prerequisites

- Phase 5 must be completed (application code validated)
- `.appgen/PHASE.md` contains app details

## Steps

### Step 1: Gather Information

Read the following files to understand the application:

1. `{app-directory}/.appgen/PHASE.md` - App name, description, type, URLs
2. `{app-directory}/doover_config.json` - Configuration schema, app metadata
3. `{app-directory}/src/{app_name}/application.py` - Core functionality
4. `{app-directory}/src/{app_name}/app_config.py` - Configuration options
5. `{app-directory}/src/{app_name}/app_ui.py` - UI components (if present)

### Step 2: Generate README.md

Create a comprehensive README.md with the following sections. Use the structure below as a template:

---

## README Template

```markdown
# {App Display Name}

<img src="{icon_url}" alt="App Icon" style="max-width: 100px;">

**{Short one-line description of what the app does}**

[![Version](https://img.shields.io/badge/version-0.1.0-blue.svg)]({github_url})
[![License](https://img.shields.io/badge/license-Apache%202.0-blue.svg)]({github_url}/blob/main/LICENSE)

[Getting Started](#getting-started) | [Configuration](#configuration) | [Developer]({github_url}/blob/main/DEVELOPMENT.md) | [Need Help?](#need-help)

<br/>

## Overview

{2-3 paragraphs explaining what this application does, why it's useful, and what problems it solves.}

### Features

- {Feature 1}
- {Feature 2}
- {Feature 3}
- {etc.}

<br/>

## Getting Started

### Prerequisites

{List any prerequisites - external accounts, API keys, hardware requirements, etc.}

1. {Prerequisite 1}
2. {Prerequisite 2}

### Installation

{Installation steps if any - for device apps, explain how to add the app to a device}

### Quick Start

{Brief steps to get the app running with minimal configuration}

<br/>

## Configuration

| Setting | Description | Default |
|---------|-------------|---------|
| **{Setting Name}** | {What this setting does} | {Default value or *Required*} |
| ... | ... | ... |

{For complex settings like arrays or objects, provide additional explanation and examples}

### Example Configuration

```json
{
  "setting_1": "value",
  "setting_2": 123
}
```

<br/>

## {Tags / UI Elements}

{For processors/integrations, document the tags exposed. For device apps, document UI elements.}

### Tags (for processors/integrations)

This processor exposes the following status tags:

| Tag | Description |
|-----|-------------|
| **{tag_name}** | {What this tag contains} |
| ... | ... |

### UI Elements (for device apps)

This application provides the following UI elements:

**Variables (Display)**
| Element | Description |
|---------|-------------|
| **{variable_name}** | {What this displays} |

**Parameters (User Input)**
| Element | Description |
|---------|-------------|
| **{parameter_name}** | {What this controls} |

**Actions (Buttons)**
| Element | Description |
|---------|-------------|
| **{action_name}** | {What this does when clicked} |

<br/>

## How It Works

{Numbered list explaining the application's workflow}

1. {Step 1 - e.g., "The processor is triggered by..."}
2. {Step 2 - e.g., "It reads configuration and..."}
3. {Step 3 - e.g., "Data is processed by..."}
4. {Step 4 - e.g., "Results are sent to..."}
5. {Step 5 - e.g., "Status is updated in..."}

<br/>

## Integrations

{Explain what other systems, apps, or services this application works with}

This {app type} works with:

- {Integration 1}
- {Integration 2}
- {etc.}

<br/>

## Need Help?

- Email: support@doover.com
- [Doover Documentation](https://docs.doover.com)
- [App Developer Documentation]({github_url}/blob/main/DEVELOPMENT.md)

<br/>

## Version History

### v0.1.0 (Current)
- Initial release
- {Feature 1}
- {Feature 2}
- {Feature 3}

<br/>

## License

This app is licensed under the [Apache License 2.0]({github_url}/blob/main/LICENSE).
```

---

### Step 3: Customize for App Type

Adjust the README based on the app type:

**For Docker Device Apps:**
- Emphasize UI elements section
- Include information about the main loop and how often it runs
- Document hardware I/O if applicable
- Include simulator information if relevant

**For Processors:**
- Emphasize the Tags section
- Document event handlers (on_message_create, on_schedule, etc.)
- Explain channel subscriptions
- Document any scheduled triggers

**For Integrations:**
- Emphasize the ingestion endpoint
- Document expected payload formats
- Include example webhook payloads
- Document security settings (IP whitelist, HMAC, etc.)

### Step 4: Extract Configuration Details

Parse the `config_schema` from `doover_config.json` to generate the Configuration table:

1. List all properties from the schema
2. Use `title` for the Setting Name
3. Use `description` for the Description
4. Use `default` for the Default value (or "*Required*" if in required array)
5. For nested objects/arrays, provide additional examples

### Step 5: Document Tags and UI

**For processors/integrations:**
- Look for `await self.set_tag()` calls in `application.py`
- Document what each tag contains and when it's updated

**For device apps:**
- Parse `app_ui.py` for Variables, Parameters, Actions, and StateCommands
- Document each UI element's purpose

### Step 6: Write the README

Write the completed README.md to `{app-directory}/README.md`.

Ensure:
- All placeholder values are replaced with actual content
- Links use the correct GitHub URL from PHASE.md
- Icon URL is correct
- Configuration table is complete and accurate
- Examples are valid JSON

### Step 7: Update State

Update `.appgen/PHASE.md`:

- Set current phase to "Phase 6 - Document"
- Set status to "completed"
- Add Phase 6 to completed phases list
- Note that README.md was generated

## Section Requirements

Each section must be filled out completely:

| Section | Required | Notes |
|---------|----------|-------|
| Header with icon | Yes | Use icon_url from PHASE.md |
| Badges | Yes | Version 0.1.0, Apache 2.0 license |
| Quick links | Yes | Getting Started, Configuration, Developer, Need Help |
| Overview | Yes | 2-3 paragraphs minimum |
| Features | Yes | At least 3-5 bullet points |
| Getting Started | Yes | Prerequisites and quick start steps |
| Configuration | Yes | Complete table from config_schema |
| Tags/UI Elements | Yes | Document all exposed data |
| How It Works | Yes | Numbered workflow (4-6 steps) |
| Integrations | Yes | What this works with |
| Need Help | Yes | Standard support links |
| Version History | Yes | v0.1.0 with feature list |
| License | Yes | Apache 2.0 |

## Completion Criteria

Phase 6 is complete when:
- [ ] README.md exists with all required sections
- [ ] Configuration table is complete and accurate
- [ ] Tags or UI elements are documented
- [ ] How It Works section explains the workflow
- [ ] All links are correct (GitHub URL, icon URL)
- [ ] `.appgen/PHASE.md` updated with status "completed"

## Summary for Orchestrator

When returning to the orchestrator, include:
- **Status**: success or failure
- **README sections**: List of sections generated
- **Configuration items documented**: Count of config options
- **Tags/UI elements documented**: Count of items
- **Errors**: Any issues encountered
