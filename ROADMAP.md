# Roadmap — Milestone 5 (Automation Engine)

Captures the concrete next steps and open design decisions from planning
this milestone, so a fresh session (especially a cloud session without
access to a prior session's local plan files) doesn't have to re-derive
them or start making blind decisions. Update this file as steps complete
or decisions get made — it should always reflect current reality, not a
frozen snapshot.

## Multi-table Automaters — DONE 2026-07-10

**The bug that surfaced this**: a user testing two rules in the same
project — one on `env_readings`, one on `device_health` (from the new
"Rule Testing Sandbox" / "Rule Testing Collector" fixture, see below) —
attached both to the *same* existing Automater via the UI. The Automater
had been created with a single mqtt input (whichever table the first rule
targeted); the second rule's table didn't match it. Two symptoms: (1)
Dedup Identifiers silently auto-populated from the *wrong* table's tag
keys (the Automater's existing input's, not the new rule's actual
table's) with no warning, since the frontend's `derivedInput` correctly
resolves for existing Automaters but never checked the rule's table
against it; (2) had the rule been submitted as-is, the deployed
`telegraf.conf` would have had an mqtt input subscribed to the *first*
rule's topic only, so the second rule's `table` would never equal any
incoming metric's name (`processors.rule`'s `evaluateConditions` requires
`rule.Table == metric.Name()`) — the rule would deploy successfully and
never fire, silently, forever.

**Decision**: rather than force "one Automater = one table" (which would
just mean directing the user to create a second Automater), extend
Automater to support multiple mqtt inputs, mirroring how Collector
already can. One deployed Telegraf process can now host rules across
several tables — each table gets its own `[[inputs.mqtt_consumer]]`
block, same as a Collector's config already looks when it has 2+ mqtt
inputs. User explicitly chose this over the smaller alternative
(strictly enforce single-table-per-Automater + block the mismatched
combination) when asked.

**Backend** (`app/automater/service.py`): `create_rule`'s existing-
Automater branch now checks `_has_input_for_table(automater, rule.table)`
before appending the rule — if no input already covers it, requires
`collector_id` (same requirement as the new-Automater path) and appends
a *new* mqtt input (via a shared `_find_mqtt_input(collector, table)`
helper) rather than rejecting or silently deploying a dead rule. Already-
covered tables are unaffected — no `collector_id` needed, exactly as
before. `update_rule` intentionally NOT touched — no UI path lets a rule's
`table` be edited post-creation yet (only enabled/disabled), so there's
nothing to retrofit there today; worth revisiting if/when rule-field
editing ships.

**Frontend** (`AutomaterEditor.tsx`): new `existingAutomaterTables` memo
(the set of tables an existing Automater's mqtt inputs already cover) and
`needsNewInputForTable` (true once a table's picked and that Automater
doesn't already cover it). When an existing Automater is selected, a
banner now states which tables it already watches. The Collector picker
— previously shown only for `+ New Automater` — now also appears when
`needsNewInputForTable`, labeled with the specific table it's being
picked for (e.g. "Collector (its MQTT input is reused) for
device_health"), and `collector_id` is included in the submit payload in
both cases. `derivedInput` now branches: existing-Automater-with-matching-
input reuses it directly (as before); everything else (new Automater, or
existing one needing a new table) derives from the chosen Collector,
same multi-input-disambiguation logic as the Collector-input fix from
earlier today.

**New fixture for exercising this and future rule-condition work**: a
dedicated "Rule Testing Sandbox" project / "Rule Testing Collector" (2
mqtt inputs: `env_readings` from `ruletest/env` — tags `sensor_id`/
`zone`, numeric `temperature`/`pressure`, string field `mode`; and
`device_health` from `ruletest/device` — tags `device_id`/`location`,
numeric `battery_pct`/`rssi`, string field `state`) plus a new
`examples/rule-testing-publisher` (mirrors `examples/mqtt-publisher`'s
pattern, `docker compose --profile tools up -d rule-testing-publisher`),
publishing continuously with per-sensor phase-shifted sine waves so
numeric thresholds cross repeatedly (real match/clear cycling, not a
one-shot match). Deliberately covers tags + numeric fields + string
fields across two distinct topics/tables in one Collector, specifically
so multi-input-Collector and now multi-table-Automater scenarios have
real data to verify against without touching the beekeeping showcase or
the original `mqtt-publisher`'s `device_metrics`/`device_status` fixture.

**Verified live**: attached a second rule (`device_health`, mixed OR
chain `battery_pct>28 OR rssi>25 OR state=="critical"`) to an existing
Automater that only had an `env_readings` input. Deployed config grew a
second `[[inputs.mqtt_consumer]]` block; both rules' matches showed up
in the real Celery worker logs from the one container, including correct
match/clear transitions on `device_health`'s `state` field. Also verified
the UI flow directly (Collector field correctly hidden until a table is
chosen, then appears scoped to that table; banner correctly lists an
existing Automater's already-covered tables).

## Per-condition AND/OR chains, tag/field lookup fix, multi-input Collector fix — DONE 2026-07-10

**The ask**: user wanted rules like `a==1 AND b>3 OR c<5` — a genuine
mixed chain, not one operator applied uniformly across all conditions.
The "Combine with" UI polish item (below, in yesterday's list) was
initially built as a single rule-wide selector reusing the old
`Rule.operator` field; user corrected this — neither the model nor the
Go plugin supported per-condition joins at all, so this was a real
three-layer redesign, not just a UI tweak.

**Model change**: `Rule.operator` removed entirely. `Condition` gained
`join: RuleOperator = "AND"` (Python), `Join string \`toml:"join"\``
(Go), `join: RuleOperator` (TypeScript) — the join is what connects a
condition to the *previous* one in the list; the first condition's `join`
is unused (nothing precedes it). Evaluation is a strict left-to-right
fold, no operator precedence/parentheses:
```go
result := evaluateCondition(rc.Conditions[0], m)
for _, c := range rc.Conditions[1:] {
    ok := evaluateCondition(c, m)
    if strings.EqualFold(c.Join, "OR") { result = result || ok } else { result = result && ok }
}
```
Deliberately no precedence rules (`a AND b OR c` always means
`(a AND b) OR c`, never `a AND (b OR c)`) — matches how a non-programmer
reads the chain top-to-bottom, and matches what the UI can express (no
grouping/parentheses control exists). `SchemaConditionBuilder.tsx` keeps
`conditions` sorted in schema-column order (not insertion order) since
fold order is now semantically meaningful, not just cosmetic.

**Bug found and fixed along the way**: `evaluateCondition()` only ever
called `m.GetField()`, never `m.GetTag()` — a condition on a tag-typed
column (e.g. `apiary_id`) always evaluated false regardless of the real
value. Fixed with a `columnValue()` helper mirroring the existing
tag-then-field lookup `identifierValue()` already used for dedup hashing.
Verified live with a real mixed AND/OR rule referencing both a tag and a
numeric field — match/clear oscillated correctly against real
temperature crossings.

**String field verification** (user's direct follow-up question: "are we
covering string fields too, not just tags/numbers"): traced
`equalValues()`'s fallback (`fmt.Sprintf("%v", a) == fmt.Sprintf("%v", b)`
when both sides fail to parse as numeric) and then verified live against
a real plain string *field* (`device_status.connection`, values
`"online"/"offline"/"degraded"` — not a tag, `tag_keys` on that input is
`["device_id"]` only). Confirmed working: `connection == "offline"`
correctly fired `flag=match` and cleared (`flag=clear`) on a real
`offline → degraded` transition observed from live simulator traffic.

**Bug found during that verification, fixed**: `AutomaterService
.create_rule` derived a new Automater's MQTT input via
`next(i for i in collector.inputs if i.plugin_type == "mqtt")` — the
*first* mqtt input, unconditionally. A Collector with more than one mqtt
input (one per table/topic — the "Test" collector has one for
`device_metrics`, one for `device_status`) could silently get the wrong
one: a rule targeting `device_status` got deployed subscribed to
`device_metrics`'s topic instead, so it could never see matching data.
Fixed by matching on `configuration.get("name_override") == rule.table`
instead of taking the first match; raises `InvalidOperationError` if no
mqtt input on the chosen Collector targets the rule's table. Same fix
applied to `derivedInput` in `AutomaterEditor.tsx` (only affects the
`+ New Automater` path — an *existing* Automater's input is already
fixed at its own creation time, no ambiguity there).

`custom-telegraf/docs/adding-a-plugin.md`'s sample config updated too
(now shows a real two-condition `AND`/`OR` example), matching the
in-plugin `sample.conf`.

## Automater/Rule architecture redesign — DONE 2026-07-10

**The bug that surfaced this**: testing on 2026-07-09 created two
Automaters ("test", "test temp") in what the user expected to be *one*
project's automation setup. Both showed up as separate
`iotops-automater-*` containers. Root cause, confirmed by reading the
code: `AutomaterEditor.tsx` *always* called `createAutomater()` — no
lookup for "does this project already have an Automater?" before
creating a new one. The backend model was actually already right for
multiple rules (`Automater.rules: list[Rule]` always supported it) — the
bug was purely that nothing ever reused an existing Automater, and there
was no independently-addressable Rule resource at all.

**Revised the mental model once more before implementing** (2026-07-10):
the first draft of this plan (below, superseded) assumed *one* Automater
per project, auto-created implicitly. User pushed back — Collector was
never restricted to one per project either, so forcing that on Automater
would have been a new, unjustified inconsistency, not a fix. Landed on:
**Automater** = a deployed service (one Telegraf container); a project
can have *as many as the user wants*, for whatever reason (different
purposes, different Collectors, different TTL/severity profiles,
whatever). **Rule** = an independent resource with its own lifecycle,
living inside one Automater. When creating a rule, the user explicitly
picks which Automater it joins via a dropdown (scoped to the selected
project) or `+ New Automater`, rather than it being resolved implicitly.
Creating a new Automater also requires picking which of the project's
Collectors to derive its MQTT input from (a project can have more than
one Collector too — the original "first Collector found" simplification
was dropped in favor of an explicit picker once multiplicity was
embraced project-wide).

Action sets, as originally agreed and unchanged by the revision:
- Automater: deploy, stop, delete (unchanged, existing).
- Rule: activate, deactivate, delete (new).

**Lifecycle rules, as implemented:**
- Creating a rule against an existing Automater → append it, redeploy.
- Creating a rule with `+ New Automater` → derive `inputs` from the
  chosen Collector's MQTT input, static Celery `outputs`, create the
  Automater with this one rule, deploy.
- Deactivating/reactivating a rule (`enabled` toggle) or deleting one →
  redeploy if the Automater still has ≥1 *enabled* rule left, else stop
  it (skipped entirely — no-op — if it was never deployed in the first
  place, so `_redeploy_or_stop` can't crash calling `stop()` on a
  container that never existed). Automaters are never auto-deleted;
  that's still only a manual, explicit action.
- Deleting a rule refuses if it's the Automater's *last* rule
  (`InvalidOperationError`, 400) — "delete the Automater instead" is the
  explicit path for that, keeping `Automater.rules` non-empty (the
  Pydantic model's own validator already requires ≥1 rule structurally).
- One deliberate simplification vs. the original plan: deactivated rules
  stay *in* the generated config with `enabled = false` rather than being
  stripped out — `processors.rule`'s `Apply()` already skips disabled
  rules entirely (see "Evaluate every rule independently" above), so the
  behavior is identical either way and this needed zero extra filtering
  logic in `_synthesize_rule_processor`.

**Backend**: `AutomaterService.create_rule(project_id, rule, automater_id,
automater_name, automater_description, collector_id)` /
`update_rule(automater_id, rule_id, rule)` / `delete_rule(automater_id,
rule_id)`, all funneling through a shared `_redeploy_or_stop()` helper.
New endpoints: `POST /api/automater/rules`, `PUT
/api/automater/{automater_id}/rules/{rule_id}`, `DELETE
/api/automater/{automater_id}/rules/{rule_id}`. New
`InvalidOperationError` exception (400) for the business-rule violations
above (wrong project, missing collector_id/automater_name, last-rule
delete) — existing exception types (`EntityNotFoundError`,
`PluginConfigurationError`, etc.) didn't fit. `AutomaterService` now
takes a `CollectorRepository` dependency to resolve the chosen Collector's
input when creating a new Automater.

**Frontend**: `AutomaterEditor.tsx` is now a rule-creation page (not an
Automater-creation page) — Project dropdown → Automater dropdown
(existing automaters for that project + `+ New Automater`) → if new,
Name/Description + a Collector dropdown appear (progressive disclosure)
→ rule metadata + `SchemaConditionBuilder` unchanged from Phase C.
`AutomaterList.tsx` is now automater-cards-with-nested-rule-tables:
automater-level deploy/stop/delete unchanged, each rule row gets
Activate/Deactivate (toggles `enabled` via the update endpoint) and
Delete (disabled with a tooltip when it's the last rule). Rule *field*
editing (conditions/message/etc, not just the enabled toggle) has no UI
yet — only activate/deactivate/delete, matching what was actually asked
for; full edit is a later addition if needed, the backend `update_rule`
endpoint already supports it (it's a full-rule replace, not a
partial-enabled-only patch).

**Verified live** end-to-end, both via direct API calls and a real
browser: created a new Automater with rule 1 (auto-deployed, real
container); added rule 2 to the *same* Automater via the existing-automater
dropdown path (confirmed still exactly one container, both rules present
in its deployed config); created a second, fully independent Automater
in the *same* project (confirmed two separate containers); deactivated
an Automater's only rule through the real UI (confirmed automater status
flipped `running` → `stopped`, rule badge flipped `Active` → `Inactive`,
action buttons swapped correctly); confirmed deleting a last rule is
rejected with a 400 and a clear message.

**Not done**: rule field editing beyond activate/deactivate (see above);
whether `update_rule` should redeploy on *every* field change including
ones that don't affect the deployed config (e.g. `description`) wasn't
revisited — it still does, unconditionally, same as originally sketched.

## Open issues / questions — surfaced 2026-07-09, not yet actioned

Raised while reviewing the match/clear + multi-rule work above. None of
these are blocking, but all are real gaps worth resolving deliberately
rather than forgetting about. Not prioritized/sequenced beyond the
explicit note on the last one.

- **Stale mqtt inputs on multi-table Automaters aren't garbage collected
  (deliberate, 2026-07-10).** After "Multi-table Automaters" (above), an
  Automater with rules on two tables that loses its last rule for one of
  them (deleted, or deactivated) keeps the now-unused mqtt input in its
  deployed config indefinitely — it'll keep subscribing to that topic
  with nothing consuming it. Harmless (no incorrect behavior, just a
  wasted subscription), and deliberately left alone rather than adding
  input-GC logic to `delete_rule`/`set_rule_enabled` — consistent with the
  existing "deactivated rules stay in config" simplification (see
  "Evaluate every rule independently"/Automater-Rule redesign notes
  above), and re-adding a removed input if the same table's rule comes
  back would need to re-derive it from a Collector all over again.
  Revisit only if stale inputs actually become an operational nuisance in
  practice, not preemptively.
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
- ~~Zero automated tests, either repo.~~ **Addressed 2026-07-10**, now
  that the Automater/Rule shape has settled. `custom-telegraf`:
  `rule_test.go` (pure functions — condition evaluation, the left-to-right
  AND/OR fold, tag-then-field lookup, firingKey uniqueness), plus
  `rule_redis_test.go`/`celery_test.go` using `miniredis` (new dependency)
  for real match/clear/TTL/multi-rule-independence behavior without a live
  Redis. `IoTOps`: `tests/backend/automater/` (models, service, API),
  mirroring `tests/backend/collector/`'s existing pattern (`mongomock-motor`
  + a fake Docker client, `pytest.ini`/`requirements-dev.txt` were already
  set up, just unused for this module until now).

  Along the way, found the *existing* `collector`/`plugin` test suites had
  silently rotted: the `Pipeline` extraction (Phase B, days ago) moved
  `InputPlugin`/`OutputPlugin` out of `app.collector.models` and changed
  `generate_toml`'s signature, but nothing had actually run `pytest` since,
  so 7 pre-existing tests were failing on import/signature errors — fixed
  as a prerequisite to get a clean baseline before adding anything new.

  Coverage is backend/plugin-only — no frontend tests, and Go coverage
  stops at `processors.rule`/`outputs.celery`'s own logic (doesn't spin up
  a real Telegraf binary or exercise `cmd/telegraf`). Good enough to catch
  the kind of regression that bit `collector`/`plugin` above; not
  exhaustive.

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
- **Single-table-per-Rule, still true.** A prior-project reference
  example showed *one rule's own conditions* spanning two measurement
  tables (`devices_metrics` AND `devices_status`), but that needs
  cross-stream state correlation in the Go plugin — materially bigger
  scope than the documented flat `Condition` model implies. Still
  deferred; not ruled out for a later milestone. Distinct from
  "single-table-per-*Automater*", which is no longer true as of
  2026-07-10 — see "Multi-table Automaters" below: an Automater can now
  have *several* rules, each pinned to its own table, sharing one
  deployed Telegraf process (one mqtt input per table it covers).
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
      ttl = "5m"                # Go duration string, time.ParseDuration
      identifiers = ["device_id"]  # tag/field names hashed into the dedup key, in order

      [[processors.rule.rules.conditions]]
        column = "temperature"   # column/field name on `table`
        operator = ">"
        value = 36.0
        join = "AND"              # joins this condition to the *previous* one; ignored on the first condition — see "Per-condition AND/OR chains" below

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
- ~~Rules are flat (one `operator` AND/OR + a flat list of conditions)~~
  **Superseded 2026-07-10** — see "Per-condition AND/OR chains" below.
  Conditions are still a flat list (not a nested/recursive tree), but the
  AND/OR join is now per-condition, not one rule-wide operator.
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
