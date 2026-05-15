# CoAnalyst — Handoff

This document captures a design conversation. The reader is the next agent
(or future me) picking up work on CoAnalyst. The repo is fresh and empty:
https://github.com/PatelVro/CoAnalyst.git

Do not start coding. Read this top to bottom, then resolve the open
questions at the end with the user before any implementation.

---

## 1. The problem CoAnalyst exists to solve

When users build features with AI coding agents (Claude, Cursor, etc.),
agents routinely:

- Write code.
- Write tests.
- Run the tests; they pass.
- Declare the task done.

…while the actual end goal is not achieved. The user trusts the agent and
discovers later (sometimes much later) that the thing doesn't really work.

Concrete example from the user: an auto-repricer was "set up" by an agent.
Tests passed. The scheduler never reliably fired in production. The user
only found out because prices weren't moving.

The failure decomposes into three gaps:

1. **Spec gap** — the goal was never written as a checkable condition.
2. **Evidence gap** — tests pass via mocks; nothing exercised the real
   runtime path.
3. **Self-audit gap** — agents skim their own logs for "no error" rather
   than asking "what would I see if this were silently broken?"

This is especially severe for **scheduled and event-triggered work**:
cron expressions, timezone bugs, schedulers registered but never started,
double-registration, locks preventing fire, retries swallowing errors.
Unit tests almost never catch these — you can test the handler all day
without proving the scheduler calls it on Tuesday at 3am.

The user works across many projects and wants this problem solved
generally, not patched per-project.

---

## 2. What CoAnalyst is

An **agent-readiness toolkit** that drops into any project and makes it
verifiable by — and accountable to — whatever agent the user happens to
be using. Agent-agnostic, project-agnostic.

It does four things:

1. Forces declared **behaviors** (machine-readable acceptance criteria)
   before code is written.
2. Standardises **runtime evidence** (structured event log) so behaviors
   can be observed.
3. Provides a **verify** step distinct from "tests pass" — the agent must
   produce live evidence of the behavior firing.
4. Surfaces **analytics** so detailed that drift, silent failures, and
   skipped verification are visible to both the user and the next agent.

What CoAnalyst is *not*: a test framework, an APM, a logging library, an
agent. It's the connective tissue between the agent's claims and the
project's runtime reality.

---

## 3. The architectural spine (three artifacts)

Only one piece is stack-specific. The other two are plain data.

### 3a. `behaviors.yaml`  (`.coanalyst/behaviors/*.yaml`)

A per-feature contract. The agent (or user) fills this in **before**
writing code. If a behavior has no entry, the feature isn't ready to
start. Fields (draft — lock the schema before writing code):

- `id` — stable identifier, e.g. `repricer-hourly-tick`
- `goal` — one sentence in plain English
- `signal` — the observable proof of success:
  - log event name (preferred) + required fields
  - or DB state assertion
  - or HTTP response shape
  - or filesystem change
- `trigger` — how to fire this behavior manually:
  - `{ type: shell, cmd: "..." }`
  - `{ type: http, method, url, body }`
  - `{ type: time_compress, real_cron: "...", test_interval: "10s" }`
- `expected_frequency` — how often this should fire in normal operation
  (used by analytics to flag staleness)
- `owner` — agent or human who declared it
- `created_at`, `last_verified_at`

This single artifact closes the spec gap. It is the discipline.

### 3b. Event log schema (canonical JSON)

One schema, written to disk (NDJSON) or shipped over a socket. Every
adapter conforms. The analytics layer reads only this — never raw stdout,
never framework-specific logs.

Draft fields:

- `ts` — ISO-8601 UTC
- `event_type` — `registration` | `tick` | `decision` | `outcome` | `error`
- `behavior_id` — links back to behaviors.yaml
- `correlation_id` — to trace a single fire across multiple events
- `decision` — for decision events, the choice made
- `inputs` — what the code saw
- `outcome` — `success` | `noop` | `failure` (no-ops MUST be logged;
  silent success looks identical to silent failure otherwise)
- `meta` — free-form

Without a canonical schema, "advanced analytics" becomes regex
archaeology. The schema is the single most important design artifact.
Lock it on paper before writing any logger code.

### 3c. The `coanalyst` CLI

Written once, in one language. Commands (draft):

- `coanalyst init` — scaffold `.coanalyst/` in the target project
- `coanalyst declare <id>` — interactive prompt to author a behavior
- `coanalyst verify <id>` — run the behavior's trigger, watch the event
  log for the declared signal within a timeout, report pass/fail/silent
- `coanalyst report` — produce the analytics view:
  - behaviors declared but never observed firing
  - behaviors firing without their expected decisions
  - behaviors stale beyond expected_frequency (catches dead schedulers)
  - decisions made without logged inputs
  - last verify per behavior
- `coanalyst hook install` — drop in the Claude Code hooks bundle
  (SessionStart loads behaviors.yaml into context; Stop blocks if any
  touched behavior lacks a passing verify run)

The CLI does not need to be written in the target project's language.
It only reads JSON. Pick one host language for the CLI.

### 3d. Per-stack logger adapter (the thin part)

`coanalyst-py`, `coanalyst-js`. Each ~100–200 lines. One job: emit a
canonical event with the right fields. No framework integration. No
auto-instrumentation. No magic.

**Resist** every temptation to make these smart. The whole point is that
**declaration is the discipline** — if the agent doesn't write the
`coanalyst.event("repricer.tick", ...)` call, the agent didn't think
about the behavior. Auto-capture defeats the mechanism.

---

## 4. Build order

1. **Lock the schemas on paper first.** `behaviors.yaml` fields and the
   event JSON schema. Everything else depends on them. Getting these
   wrong later means rewriting every adapter.
2. **Build the CLI** in one host language. Probably Node/TS (because
   `npx coanalyst init` works for Python and JS users alike; Python devs
   won't `pip install` a JS tool but JS devs won't `pip install`
   anything). Not a strong opinion — flip if the user prefers Python.
3. **Ship the Python logger first** (the user's primary stack). Roughly:
   `coanalyst.event(event_type, behavior_id=..., outcome=..., **meta)`.
4. **Dogfood against one real project** — ideally one where a
   scheduler/trigger has already silently failed on the user. The point
   of the first cycle is to catch one repricer-class bug end-to-end. If
   you don't catch a real bug, you haven't validated the toolkit.
5. **JS adapter second**, once Python has caught something real.
6. Generalise from there.

---

## 5. Principles (do not violate)

- **Declaration is the discipline.** Loggers stay dumb on purpose.
- **Evidence, not assertion.** "Done" requires a verify-pass with a
  captured event, not just green tests.
- **Log no-ops explicitly.** Silent success and silent failure look
  identical otherwise.
- **Compress time for verification.** Schedulers get a 10-second test
  interval (or manual trigger endpoint) so verify can actually observe
  a fire. Real schedule is restored after.
- **Log the schedule itself at startup.** "Registered job X with cron
  `0 */6 * * *`, next run at 2026-05-14T18:00Z." This separates
  *registration* health from *handler* health.
- **Inverted prompt.** Verification asks "what would I see if this were
  silently broken?" not "does this work?" Adversarial framing flips the
  default skim.
- **Don't ship "works on any project, any agent" before catching one
  real bug.** That's how this becomes a meta-project that never proves
  itself.

---

## 6. Open questions — resolved with user

Resolved 2026-05-15:

1. **GitHub access.** Agent pushes directly to `PatelVro/CoAnalyst`
   on branch `claude/coanalyst-handoff-review-L4nFB`; PRs opened as
   drafts for review.
2. **"Maybe Windows."** User's **dev machine is Windows**. Hooks must
   work in PowerShell; CLI must handle Windows paths; the dogfood
   project may or may not be Windows — confirm when target is chosen.
3. **CLI host language.** Node/TS. `npx coanalyst <cmd>` is the install
   ergonomics target.
4. **Dogfood target.** User will name a specific project (not the
   repricer example). Pending — does not block schema lock.
5. **Auto-CoPylot.** Separate project, untouched. This handoff was
   originally written there; it now lives in `CoAnalyst/HANDOFF.md`.

---

## 7. What was explicitly NOT decided

- The exact `behaviors.yaml` field list (drafted above, not locked).
- The exact event JSON schema (drafted above, not locked).
- Whether CoAnalyst supports anything beyond schedulers/triggers in v1.
  The conversation was scheduler-heavy because that's the user's pain.
  Don't over-generalise the schema for UI flows, batch jobs, etc.
  until v1 has caught a real bug.
- Storage of the event log (NDJSON file? SQLite? Postgres? local first,
  probably).
- Whether `coanalyst verify` runs in CI or only locally (probably both,
  but not designed yet).
- Any UI for the analytics report (CLI text is fine for v1).

---

## 8. First concrete deliverable for the next session

After resolving the open questions above:

1. Write `SCHEMAS.md` in CoAnalyst with locked `behaviors.yaml` and
   event JSON schemas. No code yet.
2. Get user sign-off on the schemas.
3. Then scaffold the CLI.

Do not skip step 1. The schemas are the contract; rushing them
guarantees rework.
