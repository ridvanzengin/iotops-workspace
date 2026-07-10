# Roadmap — Milestone 5 (Automation Engine)

Captures the concrete next steps and open design decisions from planning
this milestone, so a fresh session (especially a cloud session without
access to a prior session's local plan files) doesn't have to re-derive
them or start making blind decisions. Update this file as steps complete
or decisions get made — it should always reflect current reality, not a
frozen snapshot.

## Next session — Automater/Rule architecture redesign (not started)

**The bug that surfaced this**: testing yesterday created two Automaters
("test", "test temp") in what the user expected to be *one* project's
automation setup. Both showed up as separate `iotops-automater-*`
containers. Root cause, confirmed by reading the code: `AutomaterEditor.tsx`
*always* calls `createAutomater()` — there's no lookup for "does this
project already have an Automater?" before creating a new one. The
backend model was actually already right for multiple rules
(`Automater.rules: list[Rule]` always supported it) — the bug is purely
that nothing ever reused an existing Automater, and there's no
independently-addressable Rule resource at all (a Rule today only exists
as a sub-document inside `Automater.rules`, editable only by replacing
the whole list via `PUT /api/automater/{id}`). Both stray test automaters
were manually deleted by the user; nothing to clean up.

**The target mental model**, agreed 2026-07-09: **Automater** = the
deployed service (one Telegraf container). **Rule** = an independent
resource with its own lifecycle, living inside an Automater. They have
different action sets:
- Automater: deploy, stop, delete (existing, already implemented,
  unchanged — stays available as a manual override).
- Rule: activate, deactivate, delete (new — doesn't exist yet).

**Lifecycle rules agreed:**
- First rule created in a project → creates the project's Automater (if
  none exists yet) *and* the rule (active by default) → Automater is
  auto-deployed.
- Additional rule created in the same project → reuses the existing
  Automater, just adds the rule (active by default) → Automater is
  auto-*re*deployed to pick up the new rule.
- Rule deactivated or deleted → if the Automater still has ≥1 active rule,
  redeploy (to drop the deactivated/deleted rule from the live config);
  if none remain active, stop the Automater (don't delete it — it can be
  redeployed later when a rule gets activated again).
- Editing a rule's fields (conditions, message, etc.) → redeploy, same as
  above, *only if* the Automater is currently `running` — don't silently
  flip a deliberately-`stopped` Automater back to running just because
  its rule config changed underneath it (agreed direction, not yet
  implemented/confirmed against real usage).

**Backend implementation sketch** (not started):
- `AutomaterService` gains rule-scoped methods: `create_rule(project_id,
  rule_payload)` (finds-or-creates the project's one Automater — deriving
  its `inputs` from the project's Collector's MQTT input and its
  `outputs` from the static Celery output, same derivation logic
  `AutomaterEditor.tsx` does client-side today, which should move
  server-side since Automater creation becomes an implicit side effect
  rather than a user-facing step), `update_rule(automater_id, rule_id,
  rule_payload)` (replaces the rule's fields — doubles as activate/
  deactivate via its `enabled` field and as a full field edit — then
  redeploys-or-stops per the lifecycle rules above), `delete_rule(automater_id,
  rule_id)` (same redeploy-or-stop logic).
- New endpoints: `POST /api/automater/rules`, `PUT
  /api/automater/{automater_id}/rules/{rule_id}`, `DELETE
  /api/automater/{automater_id}/rules/{rule_id}`.
- Deleting an Automater still cascades to its embedded rules (nothing
  new needed there — they're sub-documents).

**Frontend implementation sketch** (not started):
- `AutomaterEditor.tsx` becomes a rule editor — create *and* edit. Note:
  edit mode doesn't exist anywhere in this app yet (Collector doesn't
  have it either — checked `CollectorEditor.tsx`, it's create-only, no
  `useParams`/GET-then-populate/PUT path), so this is genuinely new UI
  work, not mirroring an existing pattern. Drops the Automater Name/
  Description fields entirely (Automater becomes implicit/derived, named
  from its project) — keeps the Project selector (now used to find/
  imply the target Automater, not to name a new one) plus everything
  already built for rule metadata + `SchemaConditionBuilder`.
- `AutomaterList.tsx` becomes automater-rows-with-nested-rules:
  automater-level deploy/stop/delete stays as today, each rule nested
  underneath gets activate/deactivate/delete/edit actions.

Open items to resolve when picking this up: exact derived Automater
`name` (e.g. `f"{project.name} Automater"`?), whether `update_rule` should
really always redeploy on *any* field change (including edits that don't
affect the deployed config, like `description`) or only when the deployed-
relevant fields change, and the route shape for the new rule editor page
(`/automaters/rules/new?project=<id>` for create vs.
`/automaters/:automaterId/rules/:ruleId/edit` for edit, or similar).

## Open issues / questions — surfaced 2026-07-09, not yet actioned

Raised while reviewing the match/clear + multi-rule work above. None of
these are blocking, but all are real gaps worth resolving deliberately
rather than forgetting about. Not prioritized/sequenced beyond the
explicit note on the last one.

- **Asymmetric Redis-error handling in `rule.go`.** `trySetFiring` fails
  *open* on a Redis error (treats it as a new match — risk: duplicate
  `match`). `clearFiring` fails *closed* (no clear emitted — risk: a
  clear silently drops, and the firing key just expires later via TTL
  with no explicit `clear` ever logged). A consumer relying on
  match/clear pairing for state could see a permanently "unresolved"
  occurrence that quietly self-heals with no signal. Worth fixing before
  calling match/clear done-done — probably means giving `clearFiring` a
  bounded retry, or at minimum logging loudly enough that it's
  operationally visible when it happens.
- **Unexplained ~90-minute MQTT reconnect gap** from 2026-07-08. Stock
  Telegraf `inputs.mqtt_consumer`, not custom code, resolved by a
  container restart, never root-caused. Hasn't recurred since, but
  nothing detects or prevents it if it does (no healthcheck, no
  alerting on "container up but not receiving").
- **Disabling a firing rule never clears it, at the plugin level.**
  Tomorrow's redeploy-on-deactivate plan (rule removed from the
  generated config entirely) sidesteps this in practice, but the
  underlying gap — ship a config with a rule flipped to `enabled=false`
  while it was firing, and its firing key just expires silently, no
  `clear` ever emitted — exists independently of how the product
  happens to use `enabled` today.
- **No flapping/hysteresis protection.** A value oscillating right at a
  threshold (59.9, 60.1, 59.9...) fires a match/clear pair on every
  single crossing. Probably fine for now; alerting systems usually need
  this eventually.
- **The `tag_keys` mismatch warning needs to survive the Automater/Rule
  rework above.** `AutomaterEditor.tsx` currently warns when a rule's
  `identifiers`/`message` reference a field missing from the input's
  `tag_keys` — real and useful, already built (see Phase C). The
  redesign above is going to substantially rewrite this page into a rule
  create/edit flow; easy to lose this warning in the rewrite if not
  deliberate about carrying it over.
- **No server-side validation that `identifiers`/message placeholders
  are actually in `tag_keys`.** Today it's a frontend-only warning —
  anyone hitting the API directly (or a future non-UI client) gets no
  protection at all.
- **No image build/publish pipeline for `custom-telegraf`.**
  `custom-telegraf:latest` is built by hand locally; already caused one
  real outage (the "action failed" deploy bug from 2026-07-08, deploy
  500'd because the image didn't exist). Fine for local dev, a real gap
  before this goes anywhere beyond one machine.
- **Beekeeping showcase seeding (`ensure_automater()`) still not
  started.** Phase D, see below — reconfirming it's still open, not a
  new finding.
- **Zero automated tests, either repo.** Rule evaluation, dedup/firing-
  state transitions, TOML generation, Pydantic validation — all verified
  by hand against the live stack this whole milestone, never by a test
  suite. `rule.go`'s pure functions (`evaluateCondition`, `firingKey`,
  etc.) are straightforward to unit test and have none. Python side has
  pytest + mongomock-motor already set up in the repo, unused for any of
  this. **Do this one after the Automater/Rule work above ships**, not
  before — no point writing tests against an API/service shape
  (`create_rule`/`update_rule`/etc.) that's about to change.

## Match/clear flag semantics — DONE 2026-07-09

Implemented the firing/resolved pattern flagged 2026-07-08: a `flag` tag
(`"match"` / `"clear"`) alongside the existing `matched_rule` tag,
resolving all the open questions from that note:
- **Same occurrence across match→clear**: yes, the existing
  `identifiers`-derived SHA-256 hash, unchanged.
- **Redis state**: the dedup key was repurposed (and renamed
  `automater:dedup:...` → `automater:firing:...`, a clean break so no
  stale keys from before this change carry over any meaning) into a
  firing-state key. First match: `SETNX` succeeds → emit `match`. Still
  matching on a later metric: `SETNX` fails (key exists) → refresh its
  TTL as a heartbeat, suppress the repeat (same "don't re-fire every
  flush" behavior as before). First non-match after a match: `DEL` the
  key → if it actually existed (`DEL` returned >0), emit `clear`; if it
  didn't exist (this occurrence was never firing), emit nothing — so a
  fresh deploy doesn't spam clear events for every currently-non-matching
  device. This was explicitly checked before implementing (see git
  history if you want the back-and-forth) since it's the obvious failure
  mode for this kind of feature.
- **Celery envelope**: same task (`automater.tasks.log_rule_match`), no
  new task name — both match and clear are still just structured logging,
  distinguished by the `flag` tag the task already receives via kwargs.
- **TTL interaction**: TTL is now a firing-state heartbeat (refreshed on
  every still-matching observation), not a suppression window — it only
  matters as a safety-net auto-forget if the metric stream stops
  reporting entirely (no explicit clear fires in that case, the key just
  silently expires).
- **Per-rule opt-in**: no, universal — every rule gets match/clear, no new
  `Rule` model field. Matches the user's recollection that this was just
  how the mechanism always worked, not an extra mode.

Verified live against the real beekeeping stack (two real deployed
Automaters, real simulator data, real Celery worker) — confirmed actual
log lines with both `flag: 'match'` and `flag: 'clear'` for the same
device as its humidity crossed the threshold in both directions, e.g.
`apiary-2/hive-6`: `match` at humidity=60.21, `clear` shortly after at
humidity=59.63.

**Superseded same day** by "Evaluate every rule independently" and
"Unique rule ID in the firing key" below — `firstMatch()`/`firstApplicable()`
(the "first enabled rule wins, one event per metric" framing mentioned
here originally) no longer exist; see those sections for what replaced
them and why.

## Evaluate every rule independently — DONE 2026-07-09

The original "first-match-wins, one event per metric" rule (documented
below under "already decided") turned out not to fit real usage: a user
creating a temperature rule *and* a humidity rule on the same table
expects both to be evaluated and both to potentially fire — they're
unrelated conditions, not competing alternatives. Reversing that decision:

- `Apply()` in `custom-telegraf/plugins/processors/rule/rule.go` now
  loops every enabled rule whose `table` matches, independently, against
  each incoming metric — not stopping at the first one with something to
  report.
- One input metric can now produce **multiple** output metrics (one per
  rule that matched or cleared). Mutating the same `telegraf.Metric`
  object per rule would have each rule's tags stomp the previous one's,
  so each event now uses `m.Copy()` (confirmed present on the
  `telegraf.Metric` interface) before `annotate()`.
- `Priority` no longer decides which rule "wins" — every enabled rule
  gets its chance regardless of priority. The field and its sort in
  `Init()` are left in place (harmless, gives stable iteration/log
  order) but no longer gate evaluation. Nothing currently reads
  `Priority` for anything else; consider dropping it later if it stays
  unused after the frontend rework below ships.
- This also fixes the multi-rule-same-table short-circuit gap noted in
  the match/clear section above (a suppressed-repeat match on one rule
  no longer blocks a same-table lower-priority rule's legitimate clear)
  — same root cause, same fix.

Verified live: a single incoming metric (`apiary-2/hive-4`,
humidity=60.3, temperature=34.1) produced **two independent `match`
events at the same timestamp**, correctly tagged `rule_category=humidity`
and `rule_category=temperature` respectively, from two separately
configured rules on the same table.

## Unique rule ID in the firing key — DONE 2026-07-09

`firingKey()` used to be `automater:firing:{rule_name}:{identifiers_hash}`
— the rule's *Name* was the only thing distinguishing one rule's firing
state from another's, and nothing enforces Name uniqueness anywhere
upstream (no validator on the Pydantic `Rule.name`, and no Automater-scoping
in the key either). Two rules sharing a name — within one Automater, or
even across two completely unrelated ones — would have silently shared
firing state.

Fixed by adding `ID string \`toml:"id"\`` to `RuleConfig` (the value was
already being sent — every generated config already includes
`id = "<uuid>"` from the Pydantic `Rule.id` — the Go struct just had no
field to catch it, so TOML unmarshaling silently discarded it) and
requiring it non-empty in `Init()`. `firingKey()` is now
`automater:firing:{name}:{id}:{identifiers_hash}` — `id` is what actually
guarantees no collision (globally unique UUID); `name` stays purely for
humans reading `redis-cli keys` output.

Verified live: two rules deliberately both named `"test"` (one on
`humidity`, one on `temperature`, same table, same `identifiers`) produced
completely separate `automater:firing:test:{id-A}:...` /
`automater:firing:test:{id-B}:...` keys and never interfered with each
other's match/clear state.

## Where things actually stand

`custom-telegraf`'s `processors.rule` and `outputs.celery` now have real
logic (rule evaluation, Redis dedup, Celery protocol v2 task envelope) and
have been verified end-to-end 2026-07-08 against a real Mosquitto +
Redis: an MQTT message matching a rule got tagged
(`matched_rule`/`rule_category`/`rule_severity`/`rule_event_type`/
`rule_message`) and a valid protocol-v2 task landed in Redis's `celery`
list (body decodes to `[args, kwargs, embed]` as expected); a repeat of
the same match was suppressed by the `SETNX`-with-TTL dedup key; a
non-matching metric produced neither a dedup key nor a queue entry.

**Full production loop verified live 2026-07-08** against the real
beekeeping showcase stack (real simulator data, real deployed Automater,
real Celery worker — not a scratch/isolated test): `app/automater/tasks.py`
(new) defines `celery_app`/`log_rule_match`, run via a new `celery-worker`
docker-compose service reusing the backend image
(`celery -A app.automater.tasks worker`). `settings.redis_uri` was
repurposed as the Celery broker URL (was dead/unused config; now
`redis://redis:6379/1`, matching `CeleryOutputConfig`'s own `redis_db`
default — **also update `.env`'s `REDIS_URI`**, since it overrides the
Python default and had the stale `/0` value). Confirmed: real hive
humidity crossing the rule's threshold → tagged/deduped by
`processors.rule` → enqueued by `outputs.celery` → consumed and logged by
the real Python worker within milliseconds, repeatedly, using the actual
production image (not a debug build).

**False-alarm debugging note, worth remembering**: mid-verification the
queue appeared permanently stuck at 0 with no new events for ~90 minutes,
which looked like a serious regression. Root cause turned out to be
*not* a bug — the Celery worker was already running continuously and
draining the queue via `BRPOP` within milliseconds of each enqueue, so
`redis-cli llen celery` reading 0 was actually the **expected signature
of a healthy real-time consumer**, not evidence nothing was being
produced. Check the consumer's own logs (`docker logs
iotops-celery-worker-1`), not queue length, to verify production is
happening. Separately, a real ~90-minute `inputs.mqtt_consumer`
disconnect-without-reconnect *did* occur once during this session
(mosquitto's log showed a clean client disconnect with no reconnect
attempt until the container was manually restarted) — cause unconfirmed
(stock Telegraf plugin, not custom code), container restart resolved it,
not chased further. Worth watching for recurrence.

`IoTOps`'s Phase B backend is now done (2026-07-08): shared `Pipeline`
base (`app/shared/models.py`), `RuleProcessorConfig`/`CeleryOutputConfig`
registered in `build_default_registry()`, generalized `generate_toml`,
and the full `app/automater/{models,repository,docker,service,api}.py`
stack wired into `dependencies.py`/`main.py`. Verified for real: a
Python-generated `Automater` (via `AutomaterService._synthesize_rule_processor`
+ `generate_toml`, not hand-written TOML) was fed into the actual Go
`test-telegraf` binary from Phase A, which parsed the 3-level-deep nested
config, connected to Redis, matched a real MQTT message, and enqueued a
real Celery task — closing the loop between the two repos' halves of this
feature. `Collector`'s existing behavior was regression-checked
(`generate_toml`, the ≥1-input/≥1-output validator) and is unaffected by
the `Pipeline` extraction.

Phase C (frontend) is also done — see below. The real Python `celery`
worker (originally the last open item here) is done too, see the
"Full production loop verified live" note above. **Not yet done**: the
beekeeping showcase's own `ensure_automater()` seeding step (Phase D) —
today's live verification reused a manually-created test Automater
against the existing beekeeping simulator data, not a seeded showcase
fixture.

One gotcha surfaced during verification, worth remembering for the
frontend rule builder: Telegraf's JSON input parser silently drops any
*string*-valued JSON key that isn't listed in the MQTT input's
`tag_keys` (already documented in `mqtt.py`, rediscovered here the hard
way — a rule's `message` placeholder for a dropped field silently renders
empty, e.g. `"Hive  swarm risk"` instead of `"Hive hive1 swarm risk"`).
Any column a `Rule.identifiers` or `Rule.message` placeholder references
that isn't also a numeric field must be in the Automater's MQTT input's
`tag_keys`, or it silently vanishes before the rule processor ever sees
it. Not currently validated anywhere — worth a cross-field check in
`AutomaterService`/the frontend wizard eventually.

## Open design questions

None blocking Phase A anymore — all three resolved 2026-07-08 (see
"Already decided" below for the outcomes and concrete config shapes).

Already decided (don't re-litigate without a reason):
- **Single-table-per-Automater for v1.1.** A prior-project reference
  example showed rules spanning two measurement tables
  (`devices_metrics` AND `devices_status`), but that needs cross-stream
  state correlation in the Go plugin — materially bigger scope than the
  documented flat `Condition` model implies. Deferred; not ruled out for
  a later milestone.
- **Config shape: TOML-native nested tables, not a JSON-string blob.**
  Every other IoTOps plugin config already flows Pydantic model → dict →
  `tomli_w.dumps()`, which renders arbitrary nesting with zero
  special-casing (`app/plugin/outputs/timescaledb.py`'s list-of-dict
  `create_templates` etc. is the existing precedent). Go's toml library
  (BurntSushi-based, what Telegraf uses) unmarshals nested slices-of-
  structs the same way, so neither side needs one-off (de)serialization
  code for this plugin specifically. Finalized shape:
  ```toml
  [[processors.rule]]
    redis_host = "redis"
    redis_port = 6379
    redis_password = ""
    redis_db = 0

    [[processors.rule.rules]]
      name = "swarm-alert"
      description = "..."
      category = "hive-health"        # free text, UI grouping/filtering only
      event_type = "threshold_breach" # free text
      severity = "high"               # low | medium | high | critical
      message = "Hive {device_id} swarm risk: temperature {temperature}"
      enabled = true
      priority = 0
      table = "hive_metrics"    # measurement/hypertable this rule evaluates against
      operator = "AND"          # or "OR"
      ttl = "5m"                # Go duration string, time.ParseDuration
      identifiers = ["device_id"]  # tag/field names hashed into the dedup key, in order

      [[processors.rule.rules.conditions]]
        column = "temperature"   # column/field name on `table`
        operator = ">"
        value = 36.0

  [[outputs.celery]]
    redis_host = "redis"
    redis_port = 6379
    redis_password = ""
    redis_db = 0
    target_queue = "celery"
    task_name = "automater.tasks.log_rule_match"
  ```
  No `output_field` (fixed tag key, see below) and no `evaluation_mode`
  field (only one evaluation mode exists — see "Evaluate every rule
  independently" for what it is today — so nothing to select between
  yet) — both dropped from the config surface entirely rather than
  exposed with one legal value.
- **Redis firing-state mechanics** (originally described here as "dedup",
  before the 2026-07-09 match/clear addition superseded that framing —
  see "Match/clear flag semantics" above for the current mechanics and
  key name). Identifier hashing (`identifiers` values, in order, `|`-
  joined, SHA-256'd) is unchanged from the original dedup design.
- **Celery task envelope must match Celery's real Redis-broker wire
  protocol** (protocol v2: base64 JSON `body` + `headers` + `properties`,
  `LPUSH`ed onto a list keyed by `target_queue`) closely enough that an
  unmodified Python `celery` worker can consume it — no compatibility
  shim on the consuming side. This was already settled in
  `custom-telegraf/docs/adding-a-plugin.md`; restated here so Phase A
  doesn't have to rediscover it.
- **`Rule.table` + `Condition.column`, not `Condition.metric`.** A single
  Automater's MQTT input can carry more than one measurement if it
  subscribes to more than one topic (each topic maps to exactly one
  TimescaleDB hypertable — see `custom-telegraf`'s reference
  `outputs.postgresql` config, which never creates more than one table per
  topic). `table` lives once per `Rule` (not repeated per `Condition`,
  unlike the prior-project reference example) because every condition in
  a rule already shares one table under the no-cross-table-correlation
  decision above — repeating it per condition would just be redundant.
  `column` (not `metric`) matches the actual DB-schema vocabulary: the
  Automater rule-builder UI works the same way the Dashboard's schema
  browser does — pick table, then column, then operator, then value. The
  Go plugin refuses to match a rule against a metric whose measurement
  name (`telegraf.Metric.Name()`) doesn't equal the rule's `table`.
- **Rule carries `category`/`event_type`/`severity`/`message`** (kept from
  the prior-project reference example, minus its `kpi_name` — that concept
  is now `Automater.project_id`, mirroring `Collector`/`Dashboard`'s
  existing relationship to `Project`, not a field on `Rule` itself).
  `severity` is a fixed enum (`low`/`medium`/`high`/`critical`);
  `category`/`event_type` are free text for UI grouping/filtering only,
  not consumed by any plugin logic. `message` is a template string with
  `{field}` placeholders (e.g. `"Hive {device_id} swarm risk..."`),
  interpolated by the Go rule processor against the matched metric's
  tags/fields (same lookup as `identifiers`) — the *rule processor* does
  the interpolation, not the Celery consumer, so the logged message is
  fully formed by the time it reaches Celery. On a match, the rule
  processor stamps `matched_rule`/`rule_category`/`rule_severity`/
  `rule_event_type` tags and a `rule_message` field onto the metric, which
  flow through to the Celery task automatically (its kwargs already
  include the full tag/field set).
- Rules are flat (one `operator` AND/OR + a flat list of conditions), not
  a nested/recursive tree — matches the documented model.
- ~~Rule evaluation is first-match-wins~~ **Superseded 2026-07-09** — see
  "Evaluate every rule independently" above. Every enabled rule on a
  matching table is now evaluated on its own merits; a metric can produce
  more than one event.
- The matched rule's name goes into a **fixed** tag key (not a
  per-Automater configurable "Output Field") — one less moving part.
- Only one Celery action ships for v1.1: structured logging. Send-email/
  webhook/MQTT-publish stay explicitly deferred (matches
  `docs/architecture.md`'s own "Future versions may include additional
  actions" framing).

## Phase A — `custom-telegraf`: real plugin logic — DONE 2026-07-08

1. ~~Implement real rule evaluation + Redis dedup in
   `plugins/processors/rule/rule.go`~~ Done: table/column-scoped AND/OR
   evaluation, SHA-256 identifier hashing, `SETNX`-with-TTL dedup,
   `matched_rule`/`rule_category`/`rule_severity`/`rule_event_type` tags +
   interpolated `rule_message` field on match.
2. ~~Implement real Celery task enqueueing in
   `plugins/outputs/celery/celery.go`~~ Done: real protocol-v2 envelope,
   `LPUSH`ed onto `target_queue`.
3. ~~Update `docs/adding-a-plugin.md`'s config-shape section~~ Done.

Verified end-to-end against real Mosquitto + Redis (see "Where things
actually stand" above); not yet verified against a real Python `celery`
worker (Phase D).

## Phase B — `IoTOps`: Automater backend module — DONE 2026-07-08

1. ~~Extract a shared `Pipeline` base~~ Done: `app/shared/models.py` now
   has `Pipeline`/`InputPlugin`/`ProcessorPlugin`/`OutputPlugin`/
   `DockerConfig`. `CollectorPluginsBase(Pipeline)` adds `processors`;
   `AutomaterPluginsBase(Pipeline)` adds `rules: list[Rule]`. Automater's
   Mongo document never stores a `processors` list; the one Rule
   Processor plugin instance is synthesized at deploy time.
2. ~~Generalize `generate_toml`~~ Done:
   `generate_toml(inputs, processors, outputs, registry)` in
   `collector/generator.py`, called by both `CollectorService.deploy()`
   and `AutomaterService.deploy()`.
3. ~~Add `Rule`/`Condition` Pydantic models~~ Done (see Phase A's config
   shape above). No separate `render_rule_config` helper turned out to be
   needed — `RuleProcessorConfig.rules: list[Rule]` flows through the
   exact same `registry.validate_configuration()` path every other plugin
   config uses; `AutomaterService._synthesize_rule_processor()` just
   builds the transient `ProcessorPlugin` wrapper.
4. ~~Add `RuleProcessorConfig`/`CeleryOutputConfig`~~ Done, under
   `app/plugin/processors/rule.py` and `app/plugin/outputs/celery.py`,
   registered in `build_default_registry()`. Default `redis_db`s differ
   (rule: 0, celery: 1) so the two purposes don't collide on one Redis
   instance by default.
5. ~~New `app/automater/{models,service,api,repository,docker}.py`~~
   Done, mirroring `collector/`'s structure 1:1.
6. ~~New `settings.automater_telegraf_image`~~ Done
   (`custom-telegraf:latest`, placeholder until a real tag gets
   published), alongside `get_automater_docker_manager()`/
   `get_automater_service()` in `dependencies.py` and `automater_router`
   in `main.py`.

**Local dev gotcha hit 2026-07-08**: the first real UI-driven deploy
(`POST /api/automater/{id}/deployment`) 500'd with
`docker.errors.ImageNotFound` — `custom-telegraf:latest` had never
actually been built locally (only a stale `custom-telegraf:test`,
predating the real plugin logic, existed). Fixed by `docker build -t
custom-telegraf:latest .` in `custom-telegraf/`. This isn't a code bug,
but it'll recur on any fresh environment or whenever `custom-telegraf`'s
plugin source changes without a rebuild — same "images COPY source at
build time, no live mount" gotcha the `run-app` skill already documents
for `backend`/`frontend`. Worth remembering: rebuild
`custom-telegraf:latest` after editing anything under
`custom-telegraf/plugins/`, the same way `backend`/`frontend` need
rebuilding after editing `app/`/`src/`.

Verified: real `Automater` → real `generate_toml` output → fed into the
actual Go `test-telegraf` binary (not hand-written TOML) → real MQTT
message matched → real Celery task enqueued. `Collector`'s existing
behavior regression-checked against the `Pipeline` extraction.

## Phase C — `IoTOps`: frontend — DONE 2026-07-08

Redesigned mid-implementation, replacing the originally-planned 5-step
wizard — see "already decided" below for why and what shipped instead.

1. ~~`types/automater.ts` + `api/automater.ts`~~ Done, mirroring
   `collector.ts`'s shapes/thin-wrapper pattern.
2. ~~`pages/AutomaterList.tsx`~~ Done, mirroring `CollectorList.tsx`.
3. ~~`pages/AutomaterEditor.tsx`~~ Done as a **single-page, two-column
   form**, not a wizard (see below) — `components/SchemaConditionBuilder.tsx`
   is the bespoke schema-browser-with-inline-conditions piece, no prior
   art existed for it in the frontend. The `identifiers`/`message`
   tag_keys gotcha (Phase A/B note above) is surfaced live in the UI, not
   just a doc comment — flagged only for columns that actually resolve to
   a non-numeric schema type, to avoid false-positiving on numeric
   placeholders like `{temperature}`.
4. ~~Enable the disabled Sidebar entry~~ Done
   (`components/Sidebar.tsx`'s `NAV_ITEMS` now has `{ label: "Automater",
   to: "/automaters", icon: AutomaterIcon }`, `disabled` dropped).

Already decided (don't re-litigate without a reason):
- **Single-page, two-column layout, not a wizard.** Left column: Basic
  Info (project/name/description) + the one Rule's metadata
  (name/category/event_type/severity/AND-OR/identifiers/ttl/message).
  Right column: `SchemaConditionBuilder` — an extended `SchemaBrowser`
  where each table's columns get a checkbox + operator + value inline,
  modeled on a reference `rule_form.html` from a prior project. Checking
  a column on a different table than the one currently selected replaces
  the whole condition set (enforces the existing no-cross-table-
  correlation decision declaratively, without needing the reference's
  accordion-click-clears-other-tables event handling).
- **One rule per Automater, not a repeatable list.** The reference this
  was modeled on only ever creates one rule per submission, and neither
  it nor the redesign request described a "+ Add Rule" affordance.
  `AutomaterInputPayload.rules` is still a list on the wire (backend
  unchanged, Go plugin unchanged — both already handle N rules), so
  multi-rule support is a pure frontend addition later if ever needed,
  not a breaking change.
- **No Input step.** An Automater's MQTT input is derived automatically
  from the first Collector found for the selected project (matching
  `project_id`, first `plugin_type === "mqtt"` input) — not re-asked of
  the user, since it must observe the same telemetry stream the Collector
  already writes to TimescaleDB anyway. A project with no such Collector
  yet blocks submission with an explicit message rather than falling back
  to a generic default, since silently guessing at broker/topic would
  produce an Automater that watches the wrong (or no) data.
- **No Output step.** Output is always exactly one `celery` plugin
  instance, `task_name` fixed to `automater.tasks.log_rule_match` (the
  only Celery action in v1.1, per the earlier "already decided" list) and
  every other field left at its `RuleProcessorConfig`/`CeleryOutputConfig`
  schema default.

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
