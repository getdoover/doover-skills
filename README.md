# Doover Skills

A collection of Claude Skills for developing applications on the Doover platform.

## Installation

Add this plugin to Claude Code:

```bash
/plugin marketplace add doover/doover-skills
```

## Available Skills

### doover-app-workflow

End-to-end workflow for creating new Doover apps:

- Research existing apps for integration
- Create GitHub repo from template
- Configure organization and registry
- Implement application logic
- Test with local simulators
- Publish to Doover platform

**Start here when building a new app.**

### doover-device-apps

Comprehensive guide for creating and developing Doover device applications. Covers:

- Application lifecycle and structure
- Configuration schemas
- UI components
- Simulators for local testing
- Platform interface for hardware I/O
- Best practices

### doover-cloud-apps

Guide for processors and integrations (cloud-based apps):

- Processor apps (channel and schedule triggered)
- Integration apps (HTTP ingestion endpoints)
- Event handlers (on_message_create, on_schedule, on_ingestion_endpoint)
- Cloud app configuration
- Integration + Processor patterns

### doover-cli

Complete reference for the Doover CLI:

- `doover app create` - Create new applications
- `doover app run` - Local development
- `doover app build` - Build Docker images
- `doover app publish` - Deploy to platform
- `doover app test/lint/format` - Code quality
- `doover app channels` - Debug data streams

### doover-data-management

Doover's channel-based data architecture:

- Channels as the foundation (publish/subscribe)
- Tags system (tag_values channel)
- UI system (ui_state, ui_cmds channels)
- Inter-agent communication
- Alerts and notifications

### pydoover

API reference for the pydoover Python library:

- Application class and lifecycle
- Configuration schema types
- UI components (Variables, Parameters, Actions)
- Hardware interfaces (Platform, Modbus)
- State machine support
- Utility functions

### doover-app-explorer

Search and explore existing Doover apps:

- Browse apps at https://admin.doover.com/app-explorer
- Query apps via API
- Find apps by functionality or dependencies
- Evaluate apps for integration
- Understand config schemas and app references

### doover-admin

Platform administration:

- Organization setup
- Device types
- Deployment configuration
- Monitoring and health checks

## Quick Start

1. Install the plugin in Claude Code
2. Describe what you want to build:
   - "I need a Doover app that pushes sensor data to Power BI"
   - "Create an app to control a pump based on tank level"
   - "Build an integration to receive webhooks from my fleet management system"
3. The `doover-app-workflow` skill will guide you through:
   - Searching for existing apps
   - Setting up a new project from the template
   - Implementing and testing
   - Publishing to Doover

## Usage

Invoke skills in Claude Code by asking questions about Doover development:

- "How do I create a new Doover app?"
- "Show me how to use state machines"
- "What CLI commands are available?"
- "How do I publish data to channels?"
- "What UI components are available in pydoover?"
- "Find apps that work with Modbus"
- "I need a Doover app that pushes tag data to Power BI"

## Resources

- [Doover Platform](https://doover.com)
- [pydoover Documentation](https://docs.doover.com)

## License

MIT License - see [LICENSE](LICENSE) for details.

## Support

For support, contact support@doover.com
