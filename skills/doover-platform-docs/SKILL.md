---
name: doover-platform-docs
description: Doover platform development reference. Use when building, modifying, or debugging Doover applications — covers configuration schemas, tags/channels, UI components, Docker app patterns, cloud handler patterns, and project setup for all app types (Docker, Processor, Integration).
---

# Doover Platform Docs

Reference documentation for developing Doover applications, organized by topic as loadable chunks.

## When to Use

- Building or modifying a Doover application (Docker, Processor, or Integration)
- Understanding Doover platform concepts (tags, channels, config schemas, UI components)
- Debugging issues in Doover app code
- Looking up correct import paths, class structures, or API patterns from `pydoover`

## How to Use

1. Read the index: `references/index.md`
2. Identify relevant chunks by app type and keywords
3. Read only the chunks you need — don't load everything

The index contains a keyword registry that maps topics to specific documentation chunks.

## Directory Structure

```
doover-platform-docs/
├── SKILL.md              # This file
└── references/
    ├── index.md              # Chunk registry with keywords and selection guide
    ├── config-schema.md      # Configuration types (all app types)
    ├── tags-channels.md      # Tags and channels (all app types)
    ├── doover-config.md      # doover_config.json structure
    ├── docker-application.md # Application class (Docker)
    ├── docker-ui.md          # UI components (Docker/Processor)
    ├── docker-advanced.md    # State machines, workers, hardware I/O
    ├── docker-project.md     # Entry point, Dockerfile, simulators
    ├── cloud-handler.md      # Handler and events (Processor/Integration)
    ├── cloud-project.md      # Build script, best practices
    ├── processor-features.md # Processor-specific (subscriptions, schedules, UI)
    └── integration-features.md # Integration-specific (ingestion, routing)
```

## Quick Reference by App Type

### Docker Apps
Start with: `config-schema.md`, `docker-application.md`, `docker-project.md`
Add if has UI: `docker-ui.md`
Add if needed: `tags-channels.md`, `docker-advanced.md`, `doover-config.md`

### Processor Apps
Start with: `config-schema.md`, `cloud-handler.md`, `cloud-project.md`, `processor-features.md`
Add if has UI: `docker-ui.md`
Add if needed: `tags-channels.md`, `doover-config.md`

### Integration Apps
Start with: `config-schema.md`, `cloud-handler.md`, `cloud-project.md`, `integration-features.md`
Add if needed: `tags-channels.md`, `doover-config.md`
