# CLAUDE.md — iotops-workspace root

This is a thin coordination repo, not a monorepo — `IoTOps/` and
`custom-telegraf/` are **independent sibling git repos** living inside
this one as subdirectories (`IoTOps/`, `custom-telegraf/` relative to
this file — nested under this workspace repo on disk, not filesystem
siblings of it), each with their own GitHub remote and history. This repo
holds only what genuinely spans both: cross-repo routing guidance for
AI-assisted sessions, and (later) shared deploy tooling once there's an
actual deployment target. Read this file first when a task touches both
repos, then read the relevant repo's own `CLAUDE.md`/`README.md` before
touching code there.

## Repo map

| Directory | What it is | Toolchain |
|---|---|---|
| `IoTOps/` | Self-hosted IoT Operations Platform — Collector/Automater/Dashboard, FastAPI + React | Python (FastAPI) + TypeScript (React) |
| `custom-telegraf/` | Custom Telegraf build bundling IoTOps's Automater plugins (`processors/rule`, `outputs/celery`) alongside upstream Telegraf | Go |

## Routing: which repo to work in

**IoTOps work** — touches `IoTOps/`:
- Collector/Automater/Dashboard/Project backend modules (FastAPI, Pydantic, Mongo, TimescaleDB)
- Frontend (React/TS) — any page, component, or API client
- Dashboard charts, Variable Builder, AI SQL builder
- Docker Compose orchestration of the whole platform (`docker-compose.yml` lives here)
- Anything under `examples/` (demo simulators/showcases)
- The Automater's deploy mechanics (running a container from `custom-telegraf`'s image + generated config) — this repo owns that, not `custom-telegraf`

**custom-telegraf work** — touches `custom-telegraf/`:
- The two custom Go plugins (`plugins/processors/rule/`, `plugins/outputs/celery/`) — real rule evaluation, Redis-backed dedup, Celery task envelope
- Anything about the Telegraf build itself (`cmd/telegraf/`, `go.mod`, `Dockerfile`)
- Adding a new custom plugin — see `custom-telegraf/docs/adding-a-plugin.md` first

**Cross-repo work** — touches both:
- Changing the config shape a plugin accepts (`RuleProcessorConfig`/`CeleryOutputConfig` in IoTOps's Pydantic models + TOML generator) must stay in lockstep with the corresponding Go struct's `toml:"..."` tags in custom-telegraf — **IoTOps's generator is the interface contract; custom-telegraf's plugins must parse whatever it produces, not the other way around**. Changing one without checking the other will silently break config generation or plugin parsing.
- Bumping the custom Telegraf image version/tag that IoTOps's Automater deploy mechanics reference.

## Key shared facts

- **IoTOps only ever consumes custom-telegraf's output as a pre-built Docker image, by tag** — it never imports custom-telegraf's Go code, and custom-telegraf never imports IoTOps's Python/TS code. No shared build context, no shared dependency graph.
- **Why two repos, not one**: Go plugin development is a genuinely different toolchain/CI/release cadence than IoTOps's Python+TS stack — see custom-telegraf's own README for the fuller reasoning.
- **Why this workspace repo exists, not a merged monorepo**: solves "one Claude Code session needs filesystem access to both repos" without permanently coupling two unrelated build pipelines. Modeled on the `agritwin` project's own root-repo pattern (`~/personal/agritwin`); each sub-repo is nested as a nested-but-independently-git-tracked subdirectory rather than a true filesystem sibling.
- **Milestone status**: IoTOps v1 (Milestones 0-4) is shipped. Milestone 5 (Automation Engine, v1.1) is substantially done — real rule/Redis/Celery logic in both custom-telegraf plugins, IoTOps's Automater backend + frontend, automated tests in both repos, a persisted events feature (Mongo-backed, SSE-delivered, with sidebar activity-bar redesign and Panel-chart overlay) are all shipped. See `IoTOps/docs/development-plan.md` for the authoritative status and `ROADMAP.md` below for what's actively in flight.

## Bandwidth is sometimes scarce — check local caches before downloading

The user is sometimes on a metered hotspot. Before any command that could
pull data over the network, check whether it's actually needed first:

- **Docker**: `docker images` to check an image isn't already present
  before `docker pull`/`docker build` triggers a fresh pull; prefer
  layer-cache-friendly rebuilds (don't touch dependency-manifest files
  unless the dependency set actually changed, so `RUN pip install`/`go
  mod download`/`npm install` layers stay cached); never `docker system
  prune`, `--no-cache` rebuilds, or other cache-busting operations unless
  there's a real reason.
- **Package installs**: check what's already installed/cached before
  installing (`pip show`, `go list -m all`, `npm ls`) rather than
  reflexively running the install command. Reuse an existing venv/
  `go env GOMODCACHE`/`node_modules` instead of recreating one if it's
  already there and current.
- **New dependencies**: adding one to `requirements.txt`/`go.mod`/
  `package.json` is a real (if usually small) download — worth a beat of
  "is this actually needed" rather than reaching for a library reflexively.
- If a task is genuinely going to need a nontrivial download (a new base
  image, a large package), say so and confirm before proceeding, the same
  way you'd confirm before any other consequential action.

## Next steps

**Read [`ROADMAP.md`](ROADMAP.md) before starting any Milestone 5 work** —
it has the concrete phased plan (custom-telegraf plugin logic → IoTOps
backend → IoTOps frontend → beekeeping showcase wiring) and, critically,
the open design questions that don't have a settled answer yet (whether
Rule/Condition needs cross-table correlation, the exact config field
shapes, Redis dedup key mechanics). Update it as steps complete or
decisions get made — treat it as living state, not a frozen plan.
