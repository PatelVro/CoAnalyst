# CoAnalyst — Locked Schemas (v1)

This document is the contract. Everything in CoAnalyst — CLI, adapters,
analytics — reads from these schemas. Changes here are breaking and
require a `schema_version` bump.

Scope of v1: scheduled and event-triggered work. We deliberately do not
yet model UI flows, batch pipelines, or long-running streams. Those wait
until v1 catches a real bug (see `HANDOFF.md` §5).

Status: **proposed**, awaiting user sign-off. No code is written against
this until sign-off lands.

---

## 1. `behaviors.yaml` schema

### 1a. File layout

One YAML document per behavior, one behavior per file:

```
.coanalyst/
  behaviors/
    repricer-hourly-tick.yaml
    repricer-price-decision.yaml
    inventory-sync-daily.yaml
```

Filename (minus `.yaml`) MUST equal the document's `id`. One-file-per-
behavior is locked because it keeps PR diffs small, lets reviewers
sign off per behavior, and avoids merge conflicts when two agents
declare in parallel.

### 1b. Fields

| Field | Type | Required | Notes |
|---|---|---|---|
| `schema_version` | int | yes | `1` for this spec. CLI rejects unknown values. |
| `id` | string | yes | `^[a-z][a-z0-9-]{0,63}$`. Must match filename. |
| `goal` | string | yes | One sentence, plain English. ≤ 200 chars. |
| `signal` | object | yes | See §1c. The observable proof of success. |
| `trigger` | object | yes | See §1d. How `coanalyst verify` fires the behavior. |
| `expected_frequency` | string | yes | ISO 8601 duration (`PT1H`, `P1D`). Used by analytics to flag staleness. Use `PT0S` for event-triggered (non-scheduled) behaviors. |
| `verify_timeout` | string | no | ISO 8601 duration. How long `coanalyst verify` waits for `signal`. Default `PT30S`. |
| `owner` | string | yes | Free-form. Agent name, human name, or team. |
| `created_at` | string | yes | ISO 8601 UTC, e.g. `2026-05-15T14:30:00Z`. Set on declare. |
| `last_verified_at` | string | no | ISO 8601 UTC. Written by `coanalyst verify`. Hand-edits are linted against. |
| `tags` | list<string> | no | Free-form labels for the analytics report. |
| `notes` | string | no | Free-form prose. Not parsed. |

### 1c. `signal` — discriminated union on `kind`

Exactly one of:

**`kind: log`** — preferred. The behavior emits a canonical CoAnalyst event.
```yaml
signal:
  kind: log
  event_type: tick           # or decision | outcome (typically outcome)
  required_outcome: success  # optional; only meaningful for event_type: outcome
  required_fields:           # dotted paths into the event's meta / inputs
    - meta.scheduled_for
    - meta.actual_fired_at
```

**`kind: db`** — assert a row count or value after the trigger fires.
```yaml
signal:
  kind: db
  driver: postgres           # postgres | sqlite | mysql
  dsn_env: DATABASE_URL      # env var holding the connection string
  query: "select count(*) from price_changes where created_at > now() - interval '1 minute'"
  expect: { op: ">=", value: 1 }
```

**`kind: http`** — assert a response from a local endpoint.
```yaml
signal:
  kind: http
  method: GET
  url: "http://localhost:8080/healthz/repricer"
  expect_status: 200
  expect_body_contains: "last_fired_at"   # optional substring check
```

**`kind: fs`** — assert a file appears or is updated.
```yaml
signal:
  kind: fs
  path: ".artifacts/last-repricer-run.json"
  expect: modified_within   # exists | modified_within
  within: PT2M              # required when expect=modified_within
```

### 1d. `trigger` — discriminated union on `kind`

Exactly one of:

**`kind: shell`** — run a command. Locked behavior on Windows: if `shell: pwsh` is set, run via `pwsh -NoProfile -Command`; otherwise run via the platform default (`bash -lc` on Linux/macOS, `cmd /c` on Windows). The user's dev machine is Windows, so `pwsh` is the supported scripting target.
```yaml
trigger:
  kind: shell
  shell: pwsh                # bash | pwsh | sh | cmd. Default = platform.
  cmd: "python -m repricer.cli tick --once"
  cwd: "."                   # optional. Relative to project root.
  env:                       # optional. Merged onto process env.
    COANALYST_TEST_INTERVAL: PT10S
```

**`kind: http`** — hit a manual-trigger endpoint exposed by the app.
```yaml
trigger:
  kind: http
  method: POST
  url: "http://localhost:8080/_coanalyst/fire/repricer"
  headers:
    Authorization: "Bearer ${COANALYST_LOCAL_TOKEN}"
  body: { reason: "verify" }
```

**`kind: time_compress`** — for schedulers that can be told to fire faster in dev. The app reads the test interval from `env_var` and falls back to `real_schedule` in production.
```yaml
trigger:
  kind: time_compress
  real_schedule: "0 */6 * * *"      # the production cron expression
  test_interval: PT10S              # ISO 8601 duration
  env_var: COANALYST_TEST_INTERVAL  # the env var the app reads
  startup_cmd: "python -m repricer"  # how to start the app under verify
```

Note: Windows Task Scheduler is intentionally **not** a first-class
trigger kind in v1. If the dogfood project uses Task Scheduler, model
the trigger as `kind: shell` with `shell: pwsh` and the equivalent
`Start-ScheduledTask` command. Promote to a first-class kind only if
the dogfood project demands it.

### 1e. Full example

```yaml
schema_version: 1
id: repricer-hourly-tick
goal: "Every 6 hours, the repricer evaluates all SKUs and may adjust prices."
signal:
  kind: log
  event_type: outcome
  required_outcome: success
  required_fields:
    - meta.skus_evaluated
trigger:
  kind: time_compress
  real_schedule: "0 */6 * * *"
  test_interval: PT10S
  env_var: COANALYST_TEST_INTERVAL
  startup_cmd: "python -m repricer"
expected_frequency: PT6H
verify_timeout: PT45S
owner: "claude"
created_at: "2026-05-15T14:30:00Z"
tags: [scheduler, pricing]
```

---

## 2. Event JSON schema

One event per line in NDJSON. UTF-8, LF line endings (even on Windows).
Producers must not include literal newlines inside an event.

### 2a. Common fields (every event)

| Field | Type | Required | Notes |
|---|---|---|---|
| `schema_version` | int | yes | `1`. |
| `ts` | string | yes | ISO 8601 UTC with millisecond precision and `Z` suffix, e.g. `2026-05-15T14:30:00.123Z`. |
| `event_type` | enum | yes | `registration` \| `tick` \| `decision` \| `outcome` \| `error`. |
| `behavior_id` | string | conditional | See §2c. Required for `registration`, `tick`, `outcome`. Optional for `decision` (inherits from correlation) and `error`. |
| `correlation_id` | string | conditional | UUID (v7 preferred for sortability, v4 accepted). Required for `tick`, `decision`, `outcome`, and any `error` that belongs to a fire. Omitted for `registration`. |
| `meta` | object | no | Free-form. See §2d for required keys by event type. |

### 2b. Event-type-specific fields

| `event_type` | Adds | Required extra fields |
|---|---|---|
| `registration` | — | `meta.schedule_kind` (`cron` \| `rate` \| `task_scheduler` \| `event`), `meta.schedule` (the expression or descriptor), `meta.next_run_at` (ISO 8601 UTC; may be `null` for event-triggered). |
| `tick` | — | `meta.scheduled_for` (ISO 8601 UTC or `null` for manual), `meta.actual_fired_at` (ISO 8601 UTC). |
| `decision` | `decision` (string), `inputs` (object) | `decision` is a short label like `"raise_price"`. `inputs` captures what the code saw — required, may be `{}` if literally nothing. |
| `outcome` | `outcome` (enum) | `outcome` ∈ `success` \| `noop` \| `failure`. **No-ops MUST be logged.** Closes the correlation chain. |
| `error` | `error` (object) | `error.kind` (string, e.g. `"TimeoutError"`), `error.message` (string), `error.stack` (string, optional). |

### 2c. Lifecycle of a single fire

```
[registration]            (once at startup, no correlation_id)
   ↓
[tick]                    (correlation_id assigned here)
   ↓
[decision]?               (0..n, same correlation_id)
   ↓
[outcome]                 (exactly one, same correlation_id)

[error] may appear at any point in the chain, sharing correlation_id.
A fire that errors before producing an outcome is considered failed.
```

A complete fire chain is `tick → outcome`. `decision` events are
optional but strongly encouraged: without them, the analytics layer
cannot tell *why* the outcome happened.

### 2d. Required `meta` conventions by event type

These keys are required by the analytics layer. Producers that omit
them won't crash anything, but the behavior will surface in
`coanalyst report` as "incomplete instrumentation."

- `registration`: `schedule_kind`, `schedule`, `next_run_at`.
- `tick`: `scheduled_for`, `actual_fired_at`. Drift between these flags timing bugs.
- `outcome`: nothing required, but conventional `duration_ms` is encouraged.

### 2e. Examples

```ndjson
{"schema_version":1,"ts":"2026-05-15T14:30:00.000Z","event_type":"registration","behavior_id":"repricer-hourly-tick","meta":{"schedule_kind":"cron","schedule":"0 */6 * * *","next_run_at":"2026-05-15T18:00:00.000Z","registered_by":"APScheduler-3.10"}}
{"schema_version":1,"ts":"2026-05-15T18:00:00.012Z","event_type":"tick","behavior_id":"repricer-hourly-tick","correlation_id":"01933b2c-1234-7abc-9def-000000000001","meta":{"scheduled_for":"2026-05-15T18:00:00.000Z","actual_fired_at":"2026-05-15T18:00:00.012Z"}}
{"schema_version":1,"ts":"2026-05-15T18:00:00.430Z","event_type":"decision","behavior_id":"repricer-hourly-tick","correlation_id":"01933b2c-1234-7abc-9def-000000000001","decision":"raise_price","inputs":{"sku":"ABC-123","current":19.99,"competitor_min":21.50,"floor":17.00}}
{"schema_version":1,"ts":"2026-05-15T18:00:00.890Z","event_type":"outcome","behavior_id":"repricer-hourly-tick","correlation_id":"01933b2c-1234-7abc-9def-000000000001","outcome":"success","meta":{"duration_ms":878,"skus_evaluated":412,"skus_changed":7}}
```

A silent-failure shape — no `tick` for hours after a `registration` —
is exactly what the analytics layer is built to surface.

---

## 3. Storage layout

```
.coanalyst/
  behaviors/                   # checked in
    <id>.yaml
  events/                      # gitignored by default
    2026-05-15.ndjson          # UTC calendar date
    2026-05-16.ndjson
  state/                       # gitignored
    last_verify.json           # per-behavior verify results
  config.yaml                  # checked in. Optional. Overrides defaults.
```

Default event path is locked; making it configurable can come later
without a schema bump.

---

## 4. Versioning

Both schemas carry `schema_version: 1`. Producers and the CLI both
write/read `schema_version`. The CLI refuses to operate on events or
behaviors with an unknown `schema_version` and surfaces a clear error.

Additive changes (new optional fields, new enum values gated behind a
new field) do not bump the version. Removing or repurposing a field
bumps it.

---

## 5. Explicitly deferred

These are out of scope for v1 schemas, deferred until v1 has caught a
real bug:

- UI flow behaviors (multi-step user journeys).
- Batch / streaming pipeline behaviors with no clear "fire" boundary.
- Distributed correlation across services (one `correlation_id` space
  per project for now).
- Sampling / rate-limiting at the producer.
- Remote event sinks (Postgres, S3). NDJSON file only in v1.
- Native Windows Task Scheduler trigger kind. Use `kind: shell` +
  `shell: pwsh` for now.
- A web UI for the analytics report. CLI text only in v1.

---

## 6. Sign-off checklist

Before any CLI code is written, the user needs to confirm:

- [ ] `behaviors.yaml` field list (§1b) is complete and the right level
      of strictness.
- [ ] `signal` kinds (§1c) cover the v1 use cases. (Note: `db`, `http`,
      `fs` are included as escape hatches — `log` is the preferred path.)
- [ ] `trigger` kinds (§1d), including the Windows/PowerShell decision
      to model Task Scheduler as `shell + pwsh` for v1.
- [ ] Event types and lifecycle (§2b, §2c).
- [ ] Required-vs-optional split (especially: `decision.inputs` required,
      `outcome` MUST be logged including no-ops).
- [ ] NDJSON-on-disk storage (§3) as the only v1 sink.
- [ ] The deferred list (§5) — anything that should be promoted into v1?
