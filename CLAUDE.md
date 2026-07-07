# CLAUDE.md — iotops-workspace root

This is a thin coordination repo, not a monorepo — `IoTOps/` and
`custom-telegraf/` are **independent sibling git repos** living next to
this one (`../IoTOps`, `../custom-telegraf` relative to this file), each
with their own GitHub remote and history. This repo holds only what
genuinely spans both: cross-repo routing guidance for AI-assisted
sessions, and (later) shared deploy tooling once there's an actual
deployment target. Read this file first when a task touches both repos,
then read the relevant repo's own `CLAUDE.md`/`README.md` before touching
code there.

## Repo map

| Directory | What it is | Toolchain |
|---|---|---|
| `../IoTOps/` | Self-hosted IoT Operations Platform — Collector/Automater/Dashboard, FastAPI + React | Python (FastAPI) + TypeScript (React) |
| `../custom-telegraf/` | Custom Telegraf build bundling IoTOps's Automater plugins (`processors/rule`, `outputs/celery`) alongside upstream Telegraf | Go |

## Routing: which repo to work in

**IoTOps work** — touches `../IoTOps/`:
- Collector/Automater/Dashboard/Project backend modules (FastAPI, Pydantic, Mongo, TimescaleDB)
- Frontend (React/TS) — any page, component, or API client
- Dashboard charts, Variable Builder, AI SQL builder
- Docker Compose orchestration of the whole platform (`docker-compose.yml` lives here)
- Anything under `examples/` (demo simulators/showcases)
- The Automater's deploy mechanics (running a container from `custom-telegraf`'s image + generated config) — this repo owns that, not `custom-telegraf`

**custom-telegraf work** — touches `../custom-telegraf/`:
- The two custom Go plugins (`plugins/processors/rule/`, `plugins/outputs/celery/`) — real rule evaluation, Redis-backed dedup, Celery task envelope
- Anything about the Telegraf build itself (`cmd/telegraf/`, `go.mod`, `Dockerfile`)
- Adding a new custom plugin — see `../custom-telegraf/docs/adding-a-plugin.md` first

**Cross-repo work** — touches both:
- Changing the config shape a plugin accepts (`RuleProcessorConfig`/`CeleryOutputConfig` in IoTOps's Pydantic models + TOML generator) must stay in lockstep with the corresponding Go struct's `toml:"..."` tags in custom-telegraf — **IoTOps's generator is the interface contract; custom-telegraf's plugins must parse whatever it produces, not the other way around**. Changing one without checking the other will silently break config generation or plugin parsing.
- Bumping the custom Telegraf image version/tag that IoTOps's Automater deploy mechanics reference.

## Key shared facts

- **IoTOps only ever consumes custom-telegraf's output as a pre-built Docker image, by tag** — it never imports custom-telegraf's Go code, and custom-telegraf never imports IoTOps's Python/TS code. No shared build context, no shared dependency graph.
- **Why two repos, not one**: Go plugin development is a genuinely different toolchain/CI/release cadence than IoTOps's Python+TS stack — see custom-telegraf's own README for the fuller reasoning.
- **Why this workspace repo exists, not a merged monorepo**: solves "one Claude Code session needs filesystem access to both repos" without permanently coupling two unrelated build pipelines. Modeled on the `agritwin` project's own root-repo pattern (`~/personal/agritwin`), adapted for sibling (not nested) sub-repos since neither IoTOps nor custom-telegraf needed to move.
- **Milestone status**: IoTOps v1 (Milestones 0-4) is shipped. Milestone 5 (Automation Engine, v1.1) is in progress — custom-telegraf has a working build toolchain with both plugins as passthrough/logging *stubs* (not real rule/Redis/Celery logic yet); IoTOps's own Automater backend module (CRUD, deploy mechanics, frontend) hasn't been started. See `../IoTOps/docs/development-plan.md` for the authoritative status.
