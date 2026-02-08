# Mini-Docs Index

Reference documentation for Doover app development, organized by topic.

## How to Use

**Plan phase:**
1. Read this index to understand available documentation
2. Select relevant chunks based on app requirements
3. Output a "Documentation Chunks" section in PLAN.md listing required/recommended chunks

**Build phase:**
1. Read chunks listed in PLAN.md
2. Scan PLAN.md content for keywords from this index
3. Auto-load any additional matching chunks

## Chunk Registry

### Core (All App Types)

| Chunk | File | App Types | Keywords |
|-------|------|-----------|----------|
| Configuration Schema | `config-schema.md` | all | config, boolean, string, integer, number, array, object, enum, application, schema, x-name, x-hidden, format |
| Tags & Channels | `tags-channels.md` | all | tag, state, persist, get_tag, set_tag, channel, publish, stream, inter-agent, coordinator, worker, throttle, rate limit |
| doover_config.json | `doover-config.md` | all | metadata, schema, export, key, visibility, depends_on, platform_interface, device_agent, modbus_interface |

### Docker Apps

| Chunk | File | Required | Keywords |
|-------|------|----------|----------|
| Application Class | `docker-application.md` | **Yes** | application, setup, main_loop, callback, loop_target_period, lifecycle |
| UI Components | `docker-ui.md` | If has_ui | ui, variable, parameter, action, button, warning, alert, submodule, range, color, statecommand, deduplication |
| Advanced Patterns | `docker-advanced.md` | Situational | state machine, worker, async, thread, hardware, gpio, debounce, timeout, transition, recovery, modbus, analog, rolling, statistics, aggregation |
| Project Setup | `docker-project.md` | **Yes** | entry, init, dockerfile, simulator, docker compose, pyproject, environment, APP_KEY, publish, version |

### Cloud Apps (Processor & Integration)

| Chunk | File | App Types | Keywords |
|-------|------|-----------|----------|
| Handler & Events | `cloud-handler.md` | PRO, INT | handler, lambda, on_message_create, on_schedule, on_ingestion, on_deployment, event |
| Project Setup | `cloud-project.md` | PRO, INT | build.sh, package.zip, cold start, idempotent, serverless |

### Processor-Specific

| Chunk | File | Keywords |
|-------|------|----------|
| Processor Features | `processor-features.md` | subscription, schedule, cron, rate, ui_manager, push_async, ping_connection, connection, ManySubscriptionConfig, ScheduleConfig |

### Integration-Specific

| Chunk | File | Keywords |
|-------|------|----------|
| Integration Features | `integration-features.md` | ingestion, endpoint, permission, extended, payload, parse, device routing, hmac, cidr, IngestionEndpointConfig, ExtendedPermissionsConfig |

## Chunk Selection Guide

### For Docker Apps

**Always include:**
- `config-schema.md`
- `docker-application.md`
- `docker-project.md`

**Include if has_ui:**
- `docker-ui.md`

**Include if relevant keywords found:**
- `tags-channels.md` - if app uses tags, publishes to channels, or coordinates with other apps
- `docker-advanced.md` - if app needs state machines, workers, hardware I/O (GPIO, Modbus), or data aggregation
- `doover-config.md` - if customizing app metadata or using system dependencies

### For Processor Apps

**Always include:**
- `config-schema.md`
- `cloud-handler.md`
- `cloud-project.md`
- `processor-features.md`

**Include if has_ui:**
- `docker-ui.md` (UI components work the same in processors)

**Include if relevant keywords found:**
- `tags-channels.md` - if app uses tags
- `doover-config.md` - if customizing app metadata

### For Integration Apps

**Always include:**
- `config-schema.md`
- `cloud-handler.md`
- `cloud-project.md`
- `integration-features.md`

**Include if relevant keywords found:**
- `tags-channels.md` - if app uses tags
- `doover-config.md` - if customizing app metadata
