---
name: doover-platform-docs
description: Doover platform development reference. Use when assisting with development of Doover applications — provides correct import paths, class structures, API patterns, and code examples for the pydoover SDK across all app types (Docker, Processor, Integration, Widget).
---

# Doover Platform Docs

Development reference for building applications on the Doover IoT platform using the `pydoover` SDK and the widget JavaScript framework (React + Module Federation).

## Your Role

You are assisting a developer who is building or maintaining a Doover application. The reference docs in this skill contain the correct patterns, class structures, import paths, and code examples for the `pydoover` SDK and the widget JS framework. Use them to give accurate, idiomatic answers — don't guess at APIs or invent patterns.

## When to Load Docs

Load relevant reference chunks when the developer is:

- **Writing or modifying application code** — load the chunk for their app type (Docker, Processor, Integration, or Widget) to get correct class structures and lifecycle methods
- **Working on configuration** — load `config-schema.md` for the `pydoover.config` schema API (types, defaults, constraints, nested objects)
- **Working on UI elements** — load `docker-ui.md` for the `pydoover.ui` component API (variables, parameters, actions, submodules, range coloring)
- **Building or modifying a widget** — load `widget-architecture.md` for the 3-layer component pattern, Module Federation config, and styling conventions; load `widget-hooks.md` for platform hooks and data access
- **Working with tags or channels** — load `tags-channels.md` for `get_tag`, `set_tag`, channel publishing, and inter-agent communication
- **Setting up a project** — load the relevant project setup chunk for Dockerfile/entry point (Docker) or build script/Lambda packaging (Cloud)
- **Debugging import errors or missing methods** — load the relevant chunk to verify correct import paths and method signatures
- **Asking "how do I..." questions about the platform** — scan the index for matching keywords and load the relevant chunks

## How to Load Docs

1. Read `references/index.md` — this contains a keyword registry mapping topics to specific chunks
2. Match the developer's current task to keywords in the index
3. Read only the chunks that are relevant — don't load everything upfront
4. If the developer's question spans multiple topics, load multiple chunks

## Reference Chunks

All chunks live in `references/` relative to this skill.

### Core (all app types)
| Chunk | File | Covers |
|-------|------|--------|
| Configuration Schema | `config-schema.md` | `pydoover.config` — Boolean, String, Integer, Number, Array, Object, Enum types, defaults, constraints, nested schemas, export |
| Tags & Channels | `tags-channels.md` | `get_tag`, `set_tag`, channel publishing, inter-agent messaging, throttling, state persistence |
| doover_config.json | `doover-config.md` | App metadata, schema references, platform interfaces, dependency declarations |

### Docker apps (device-based, containerised)
| Chunk | File | Covers |
|-------|------|--------|
| Application Class | `docker-application.md` | `pydoover.docker.Application` — setup, main_loop, callbacks, lifecycle |
| UI Components | `docker-ui.md` | `pydoover.ui` — Variable, Parameter, Action, Submodule, range coloring, alerts, deduplication |
| Advanced Patterns | `docker-advanced.md` | State machines, async workers, hardware I/O (GPIO, Modbus), data aggregation |
| Project Setup | `docker-project.md` | Entry point (`run_app`), Dockerfile, docker-compose, simulators, environment variables |

### Cloud apps (Processor & Integration, serverless Lambda)
| Chunk | File | Covers |
|-------|------|--------|
| Handler & Events | `cloud-handler.md` | `pydoover.cloud.processor.Application` — handler entry point, event methods (`on_message_create`, `on_schedule`, `on_ingestion`, `on_deployment`) |
| Project Setup | `cloud-project.md` | `build.sh`, Lambda packaging, cold start considerations, idempotency |
| Processor Features | `processor-features.md` | `ManySubscriptionConfig`, `ScheduleConfig`, `ui_manager.push_async`, connection status, processor-specific patterns |
| Integration Features | `integration-features.md` | `IngestionEndpointConfig`, `ExtendedPermissionsConfig`, payload parsing, device routing, HMAC/CIDR validation |

### Widget apps (React + Module Federation + Python processor)
| Chunk | File | Covers |
|-------|------|--------|
| Widget Architecture | `widget-architecture.md` | 3-layer component pattern, Module Federation config (rsbuild), shared modules from `customer_site`, ConcatenatePlugin, Tailwind CSS styling, processor-side `RemoteComponent` setup, name conventions |
| Widget Hooks & Data | `widget-hooks.md` | Platform hooks (`useAgent`, `useAgentChannel`, `useChannelUpdateAggregate`, `useChannelSendMessage`), `dataProvider` methods, `useParams`/`useRemoteParams`, permissions, WebSocket behavior, common read/write patterns |

## Key Principles

- **Always verify imports against the docs.** The `pydoover` SDK has specific import paths (e.g., `from pydoover.docker import Application` vs `from pydoover.cloud.processor import Application`). Don't guess — check the relevant chunk.
- **Docker and Cloud apps have different base classes and lifecycles.** Docker apps loop continuously (`main_loop`); Cloud apps respond to events (`on_message_create`, `on_schedule`, etc.). Don't mix patterns.
- **UI components work the same in Docker and Processor apps.** Both use `pydoover.ui`. Integration apps never have UI.
- **Widget apps have two sides.** The JavaScript widget (React + Module Federation) handles the UI; the Python processor (`RemoteComponent` + `on_aggregate_update`) registers the widget with the platform. Both chunks should be loaded together.
- **Widget JS must follow the 3-layer pattern.** RemoteComponentWrapper → Hooks wrapper → Inner component. The inner component uses `useParams()` from `react-router` for `agentId`. Check `widget-architecture.md` for the exact structure.
- **Widget hooks come from `customer_site/hooks`.** Don't invent hook signatures — check `widget-hooks.md` for the correct names, parameters, and return values.
- **Config schemas are shared across all app types.** The `pydoover.config` API is identical regardless of app type.
- **When in doubt, load the chunk and quote it.** It's better to show the developer the exact pattern from the docs than to paraphrase from memory.
