---
name: doover
description: Index of Doover skills - start here to understand available skills and load only what you need
user_invocable: true
---

# Doover Skills Index

This is the entry point for Doover development. Use this index to understand available skills and load only the ones you need for your task.

## Available Skills

| Skill | Use When |
|-------|----------|
| `doover-app-workflow` | Creating a new app end-to-end (start here for new apps) |
| `doover-device-apps` | Building device apps (Docker on devices) |
| `doover-cloud-apps` | Building processors or integrations (serverless) |
| `doover-cli` | CLI command reference |
| `doover-data-management` | Channels, tags, inter-agent communication |
| `doover-app-explorer` | Finding existing apps to integrate with |
| `pydoover` | Python library API reference |
| `doover-admin` | Platform administration |

## App Types Summary

| Type | Runs On | Use For |
|------|---------|---------|
| Device App (`DEV`) | Docker on device | Hardware control, local logic |
| Processor (`PRO`) | Cloud (serverless) | React to channels/schedules |
| Integration (`INT`) | Cloud (serverless) | Receive external webhooks |

## Quick Decision Guide

**"I want to create a new Doover app"**
→ Load `doover-app-workflow` skill

**"I need to find existing apps"**
→ Load `doover-app-explorer` skill

**"How do I use the pydoover library?"**
→ Load `pydoover` skill

**"How do channels and tags work?"**
→ Load `doover-data-management` skill

**"I'm working on a device app"**
→ Load `doover-device-apps` skill

**"I'm working on a processor or integration"**
→ Load `doover-cloud-apps` skill

**"I need CLI command help"**
→ Load `doover-cli` skill

## Creating a New App - Quick Reference

1. **Research**: Search https://admin.doover.com/app-explorer for existing apps
2. **Create project**:
   - Device apps: `doover app create --name NAME --description "..."`
   - Processors/Integrations: Use GitHub template https://github.com/getdoover/app-template
3. **Implement**: Modify the template files
4. **Test**: `doover app run`
5. **Publish**: `doover app publish --profile dv2`

## Loading a Skill

To get detailed information, read the specific skill file:
- `~/.claude/skills/doover-app-workflow/SKILL.md`
- `~/.claude/skills/doover-cloud-apps/SKILL.md`
- etc.

Only load the skills you need for the current task to keep context focused.
