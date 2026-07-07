# Roadmap — Milestone 5 (Automation Engine)

Captures the concrete next steps and open design decisions from planning
this milestone, so a fresh session (especially a cloud session without
access to a prior session's local plan files) doesn't have to re-derive
them or start making blind decisions. Update this file as steps complete
or decisions get made — it should always reflect current reality, not a
frozen snapshot.

## Where things actually stand

`custom-telegraf` has a **working build toolchain** with two plugins
registered (`processors.rule`, `outputs.celery`) — verified end-to-end
against a real MQTT message. Both are currently **stubs**: `rule` passes
every metric through unchanged, `celery` just logs how many metrics it
would enqueue. Neither has real logic yet.

`IoTOps` has **not started** its side of this milestone at all — no
`automater/` backend module, no frontend, no Pipeline base class
extraction.

## Open design questions — resolve these before writing real plugin logic

These came up designing the Rule/Condition model and don't have a
settled answer yet:

1. **Single-table or cross-table conditions?** `IoTOps/docs/domain-models.md`
   defines `Condition` as flat — `{metric, operator, value}` — implicitly
   assuming every condition in a rule checks the *same* incoming metric
   stream (one measurement/table, matching one Automater's one MQTT
   input). A reference example from a prior, different project (shared
   during planning, not to be copied verbatim) showed rules whose
   conditions spanned *two different* measurement tables in one rule
   (e.g. a device's `devices_metrics` reading AND its `devices_status`
   state). Supporting that would mean the Rule Processor plugin
   correlates/caches state across multiple incoming metric streams, not
   just evaluates fields on one metric at a time — a materially bigger
   scope than the documented flat model implies. **Decide**: ship
   single-table-per-Automater for v1.1 (matches the documented model,
   much less plugin complexity), or take on cross-table correlation now.
2. **Exact config field shapes for both plugins** — not finalized. Needs:
   - `processors.rule`: how rules are passed (one JSON-string TOML field,
     like the reference example's `rules = '''...'''`, vs. TOML-native
     nested tables), Redis connection fields (`redis_host`/`redis_port`/
     `redis_password`/`redis_db`), `ttl` format, and what `identifiers`
     (which fields hash into the dedup key) look like per-rule.
   - `outputs.celery`: `task_name`, `target_queue`, Redis connection
     fields, and the actual Celery task message envelope shape (must
     match Celery's real wire protocol closely enough that a Python
     `celery` worker can consume it without a compatibility shim).
   - **This shape is a cross-repo contract** (see both repos'
     `CLAUDE.md`/READMEs) — finalize it once, in one place, then update
     both `custom-telegraf`'s plugin structs and `IoTOps`'s Pydantic
     `RuleProcessorConfig`/`CeleryOutputConfig` models to match exactly.
3. **Redis dedup mechanics** — the general approach (hash a rule match's
   identifying field values into a Redis key, `SETNX`-with-TTL on first
   match, suppress re-firing while the key exists) is agreed, but the
   exact hash construction (which fields, ordering, hash algorithm) isn't
   pinned down yet.

Already decided (don't re-litigate without a reason):
- Rules are flat (one `operator` AND/OR + a flat list of conditions), not
  a nested/recursive tree — matches the documented model.
- Rule evaluation is first-match-wins (rules checked in priority order,
  first enabled match wins) — no "evaluate all, emit multiple" mode yet.
- The matched rule's name goes into a **fixed** tag key (not a
  per-Automater configurable "Output Field") — one less moving part.
- Only one Celery action ships for v1.1: structured logging. Send-email/
  webhook/MQTT-publish stay explicitly deferred (matches
  `docs/architecture.md`'s own "Future versions may include additional
  actions" framing).

## Phase A — `custom-telegraf`: real plugin logic

1. Resolve the two open questions above.
2. Implement real rule evaluation + Redis dedup in
   `plugins/processors/rule/rule.go` (currently a passthrough stub).
3. Implement real Celery task enqueueing in
   `plugins/outputs/celery/celery.go` (currently logs a count only).
4. Update `docs/adding-a-plugin.md`'s config-shape section with the
   finalized fields once settled, so it's not just "ask IoTOps" anymore.

## Phase B — `IoTOps`: Automater backend module

1. Extract a shared `Pipeline` base (`project_id`, `name`, `description`,
   `enabled`, `inputs`, `outputs` + the "≥1 input, ≥1 output" validator)
   out of `collector/models.py`'s `CollectorPluginsBase`, per
   `docs/development-plan.md`'s "Internal Pipeline Abstraction" note.
   `CollectorPluginsBase(Pipeline)` adds back `processors` itself;
   `AutomaterPluginsBase(Pipeline)` adds `rules: list[Rule]` instead —
   Automater's Mongo document never stores a `processors` list; the one
   Rule Processor plugin instance is synthesized at deploy time from
   `rules`, never persisted.
2. Generalize `collector/generator.py`'s `generate_toml(collector,
   registry)` to take the three plugin lists directly
   (`generate_toml(inputs, processors, outputs, registry)`) rather than a
   whole `Collector` — needed so `AutomaterService.deploy()` can pass a
   transient synthesized processor list without a fake `Collector`-shaped
   object.
3. Add `Rule`/`Condition` Pydantic models and a
   `render_rule_config(rules) -> dict` (or similar) helper producing the
   validated config dict for the `rule` plugin, once Phase A's config
   shape is settled.
4. Add `RuleProcessorConfig`/`CeleryOutputConfig` (mirroring whatever
   `custom-telegraf`'s real plugin structs expect, 1:1) under a new
   `app/plugin/processors/` package and `app/plugin/outputs/celery.py`,
   registered into the existing shared `build_default_registry()`.
5. New `app/automater/{models,service,api,repository,docker}.py`
   mirroring `collector/`'s structure and CRUD+deployment-lifecycle API
   shape. `docker.py` is `CollectorDockerManager` with renamed literals
   (`iotops-automater-{id}`, `runtime/automaters/{id}/`) — no other
   behavioral changes needed (the Celery broker URL goes into
   `CeleryOutputConfig`'s own config fields, not a container-level env
   var, so `AutomaterDockerManager` stays a true copy).
6. New `settings.automater_telegraf_image` pointing at the
   `custom-telegraf` image tag, alongside the existing
   `settings.telegraf_image`. New `get_automater_docker_manager()`/
   `get_automater_service()` singletons in `dependencies.py`, sharing the
   existing `get_plugin_registry()`. New `automater_router` in `main.py`.

## Phase C — `IoTOps`: frontend

1. `types/automater.ts` + `api/automater.ts`, mirroring `collector.ts`'s
   shapes/thin-wrapper pattern.
2. `pages/AutomaterList.tsx` mirroring `CollectorList.tsx`.
3. `pages/AutomaterEditor.tsx` — a 5-step wizard (`Basic Info → Input →
   Rules → Output → Review`). Input/Output steps reuse the existing
   generic `PluginRows`/`PluginConfigForm` machinery as-is (same pattern
   Collector already uses, just "mqtt" and "celery" instead of "mqtt" and
   "timescaledb"). **Not a near-zero-diff port** — `CollectorEditor.tsx`
   hardcodes step numbers in four places (`canAdvance()`,
   `stepErrorMessage()`, `goNext()`'s auto-preload, the JSX step gates),
   all needing renumbering for the inserted step.
4. New bespoke `RuleBuilder` component for the Rules step — one AND/OR
   operator selector + a repeatable list of `{metric, operator, value}`
   condition rows, modeled on the existing repeatable-row add/remove/
   update pattern already used for dual-axis series in `PanelEditor.tsx`.
   No prior art exists for this in the frontend otherwise. `canAdvance()`
   for this step requires `rules.length > 0`, matching a corresponding
   backend validator (an Automater with zero rules does nothing).
5. Enable the already-present disabled Sidebar entry
   (`components/Sidebar.tsx`'s `NAV_ITEMS` already has `{ label:
   "Automater", icon: AutomaterIcon, disabled: true }` — add `to:
   "/automaters"`, drop `disabled`).

## Phase D — Beekeeping showcase integration

1. `examples/beekeeping-simulator/seed.py` gains an `ensure_automater()`
   step (idempotent, same lookup-by-name-and-reuse pattern as the
   existing Project/Collector/Dashboard steps): one Automater, MQTT input
   on `beekeeping/hive`, one Rule named `"swarm-alert"` (`temperature >
   36` — chosen because the simulator's existing random walk crosses that
   threshold on its own, so the demo fires from genuinely-simulated
   conditions, not a forced spike).
2. Verify end-to-end: real hive data triggers a real Celery task, visible
   in `celery-worker` logs, matching the milestone's acceptance criteria
   verbatim ("Beekeeping showcase demonstrates a live swarm-alert
   triggered by simulated conditions").
