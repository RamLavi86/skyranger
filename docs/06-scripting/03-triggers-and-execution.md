# 03 Triggers and execution model

Status: design draft. Conforms to [../00-contracts/](../00-contracts/README.md). Depends on P1 to P6 (see [README.md](README.md)). Runtime mechanics in [01-runtime.md](01-runtime.md), API in [02-script-api.md](02-script-api.md).

## 1. Config kinds (P2)

Scripts are configuration: `apiVersion: skyranger/v1`, Kubernetes-style envelope, in config revisions (tag-model.md section 9). `metadata.name` follows the segment grammar.

```yaml
apiVersion: skyranger/v1
kind: Script
metadata:
  name: leak_webhook
spec:
  display_name: "Leak webhook"
  description: "POST to the on-call webhook when a leak alarm activates"
  enabled: true                   # default true
  pool: default                   # default "default"; only one pool in v1
  api_level: 1                    # script API level this source was written for (see 10)
  entrypoint: handler             # default "handler"
  timeout: 10s                    # default 10s; capped by pool ceiling (max 60s in v1)
  concurrency: 1                  # default 1; max 4
  queue:
    max_depth: 100                # default 100
    on_overflow: drop_oldest      # drop_oldest | drop_newest | coalesce; default depends on trigger kind
  ignore_own_writes: true         # tag_change triggers skip changes this script wrote (default true)
  failure_policy:
    quarantine_after: 10          # consecutive failures; 0 = never quarantine; default 10
  triggers:
    - event:
        types: [skyranger.alarm.state_changed.v1]
        subject_prefix: site1/pumphouse
  permissions:
    tags: { read: [site1/pumphouse] }
    http:
      - host: hooks.example.com
        methods: [POST]
        secret_headers: { Authorization: secret://webhooks/leak }
  source: |
    from skyranger import http, log
    def handler(ctx):
        ...
```

```yaml
apiVersion: skyranger/v1
kind: ScriptLibrary
metadata:
  name: units
spec:
  source: |
    def l_to_m3(l: float) -> float:
        return l / 1000.0
```

- `source` is inline text, or `source_file: scripts/leak_webhook.py` in the split export layout (06). Exactly one of the two.
- `kind: ScriptPool` is v2 (several pools with different ceilings). In v1 the single pool `default` is configured in the scripts container, not in tag configuration (04 section 2).
- Validation at draft time (06): YAML schema, name grammar, trigger paths syntactically valid, permission prefixes valid paths, `source` compiles (`compile()`), entrypoint exists as a top-level function, every `libs.*` import resolves, every import is on the allowlist. Dangling tag references in triggers and permissions are warnings, not errors (tags can be created in the same or a later revision), except exact `paths:` in `tag_change`, which must resolve (they would never fire).

## 2. Triggers

A script has one or more triggers. A trigger fires a **run request** (script, trigger details). Decisions happen in Go, in the script host; Python is not consulted.

### 2.1 `tag_change`

```yaml
- tag_change:
    paths: [site1/pumphouse/meter01/flow_rate]     # exact paths
    prefixes: [site1/pumphouse/meter02]             # segment-wise prefix (README naming)
    fire_on: value_or_quality                        # value | quality | value_or_quality (default)
    debounce: 0s                                     # trailing-edge: fire once after quiet period
    min_interval: 0s                                 # leading-edge rate limit per tag
```

- Source: the tag engine's propagated stream (after deadband, in `ts` order per tag), the same stream feeding alarms and the WebSocket, delivered in-process (P1). Not the MQTT mirror.
- One run per accepted change per tag. `ctx.trigger.new` and `.old` carry the canonical value object.
- Values with `ts` older than the tag's current `ts` are history-only and **do not** trigger (tag-model.md section 5).
- Semantics are **at-most-once with latest-wins coalescing**: if the script's queue is full, older pending changes for the *same tag* are replaced by the newer one (consistent with driver and WebSocket backpressure). For cumulative values, this loses nothing; for events like "pulse", use an event trigger.
- No initial fire on subscribe or startup; use a `startup` trigger plus `tags.read` for initial state.
- `debounce` and `min_interval` are exclusive options per trigger.
- Prefix triggers on large subtrees are expensive: a draft warning appears when a prefix matches more than 500 tags without `min_interval` or `debounce`.

### 2.2 `timer`

```yaml
- timer: { every: 1m }                               # Go duration, min 1s; anchored to script start
- timer: { cron: "5 0 * * *", tz: Europe/London }    # 5-field cron; tz default Gateway.spec.timezone
```

- `every` uses the monotonic clock: unaffected by wall-clock steps. `cron` uses wall-clock in the given IANA zone.
- Missed ticks (gateway down, queue full) are **skipped, not replayed** by default. `catch_up: true` (cron only) runs once at startup if a tick was missed while down, with `ctx.trigger.late` set. Rationale: replaying a backlog of "every second" runs after an outage is how plants get hurt.
- A tick that arrives while the previous run is still running is skipped and counted (`skipped_overlap`), unless `concurrency > 1`.
- DST: cron expressions in a DST zone follow the cron library's behavior; the chosen library's handling of skipped and repeated local times must be tested (UNVERIFIED which library, open question 3).
- The `scheduled_for` instant is passed to the script so reports use the schedule time, not `now()`.

### 2.3 `event`

```yaml
- event:
    types: [skyranger.alarm.state_changed.v1, skyranger.user.daily_report_done.v1]   # prefix match, like GET /events
    subject_prefix: site1/pumphouse
    where: { "data.name": leak }          # optional equality filters on top-level paths of data; v1 keeps this tiny
```

- Source: the in-process event stream (messages.md section 4) for system events, and user events emitted by scripts.
- Delivery is at-most-once in v1 (events are dropped while the pool is unavailable or the queue overflows; drops are counted and visible). Every event is in the `events` table, so a script can catch up with `events.query(start=...)`. A durable mode is v2 (open questions).
- De-duplication: the host drops an event whose `id` it already dispatched to this script (QoS 1 redelivery on the internal bus does not apply in-process, but replays from the API must not re-fire).
- `where` is deliberately not a query language: equality on `data.<key>` only. Anything more is Python.
- A script cannot be triggered by events it emitted itself (see loop protection).

### 2.4 `startup`

```yaml
- startup: { delay: 5s, on_apply: false }
```

- Fires once after `skyrangerd` has loaded config, drivers have had `delay` to connect, and the pool is attached. `reason: gateway_start`.
- `on_apply: true`: also fires when a new revision changes this script (`reason: config_applied`). Useful for "re-initialize my memory tags".
- Runs are best-effort: `startup` does not block the gateway from reporting ready.
- There is no `shutdown` trigger in v1: a shutdown handler cannot be reliable under power loss, and a script that "must run on shutdown" has a design problem. Open question 5.

### 2.5 `manual`

```yaml
- manual:
    min_role: engineer                     # operator | engineer | admin; default engineer
    params_schema:                         # JSON Schema (2020-12) subset: object of scalars
      type: object
      properties: { target_l: { type: number, minimum: 0 } }
      required: [target_l]
```

- Every script can be run manually by an `engineer` through `POST /api/v1/scripts/{name}/run` (05); declaring a `manual` trigger additionally lets lower roles run it and attaches a `params_schema`. A script with no `manual` trigger still accepts engineer test runs flagged `dry_run`.
- `requested_by` in the run record is the human principal (`user:alice`); the script's own permissions still apply (a manual run never gains the caller's rights).
- Params are validated by the host against the schema before dispatch; `ctx.params` is the validated object.
- Emits `skyranger.script.run_requested.v1` (P3) with the principal and params for audit.

## 3. Execution model

```
trigger source ──► per-script queue ──► dispatcher ──► scriptd (worker) ──► Complete ──► run record
   (Go)             bounded, policy        concurrency       timeout, memory      events, _system tags
```

- A run is **one call** of the entrypoint with a fresh `ctx`. It runs to completion or is stopped. No suspension, no resumption, no checkpointing.
- **Per-script FIFO**, concurrency 1 by default: two runs of the same script never overlap, so scripts need no locking for their own tags. `concurrency > 1` is opt-in and the script must be written for it.
- **Across scripts**: no ordering guarantee. Fairness: the dispatcher round-robins scripts with pending work, so one noisy script cannot starve others. Pool workers are the shared resource.
- **Queue overflow policies:** `coalesce` (tag_change default: keep latest per tag), `drop_oldest` (event default), `drop_newest`. Every drop increments `dropped` and writes at most one summary log line per minute (no event storm).
- **Backpressure never reaches the tag engine.** Triggers use a bounded, non-blocking offer. The tag engine never waits for a script (same rule as the historian, driver-interface.md section 6).
- **Ordering of effects inside a run** is call order. Effects are not transactional: if a script writes three tags and dies after two, two writes happened (audited). Scripts should write in the order that leaves the plant consistent. No rollback in v1.
- **Deadline propagation**: a host call inherits `min(call timeout, remaining run time)`. `tags.write` default wait (10s) is clamped to the remaining time.
- **Config changes**: a new revision swaps the immutable `ScriptRev` snapshot atomically. A run in flight finishes (or times out) on the revision it started with; its record names that revision. Queued runs execute on the revision current at dispatch time. Removing a script cancels queued runs and kills the in-flight run if it exceeds a `cancel_grace` of 5s.

## 4. Limits

Defaults; the pool ceiling (04 section 2) caps every value, a script may only lower them.

| Limit | Default | Ceiling v1 | On breach |
|---|---|---|---|
| Wall-clock per run (`timeout`) | 10s | 60s | soft `ScriptTimeout`, then `SIGKILL` at +2s; run `timeout` |
| Memory per worker | 256 MiB | 512 MiB | worker killed; run `killed` (`memory`) |
| Calls per run | 1000 | 10000 | `BudgetExceeded` |
| Tag writes per run | 100 | 1000 | `BudgetExceeded` |
| Tag writes per second per script (token bucket) | 20 sustained, burst 50 | 100 | `BudgetExceeded` |
| History points returned per run | 200000 | 1000000 | `BudgetExceeded` |
| HTTP requests per run | 10 | 100 | `BudgetExceeded` |
| HTTP response body | 1 MiB | 10 MiB | request fails (`BudgetExceeded`), never silently truncated |
| Log lines per run | 1000 | 10000 | further lines dropped, one `log truncated` marker |
| Event payload | 16 KiB | 64 KiB | `BadRequest` |
| Queue depth | 100 | 1000 | overflow policy |
| Trigger rate per script (accepted) | 50/s | 200/s | overflow policy |
| CPU | container cpu limit shared by the pool | | slow runs time out |

All numbers are starting points to be validated on the target edge hardware (UNVERIFIED).

## 5. Write-loop and cascade protection

The classic SCADA scripting bug: a tag-change script writes a tag that triggers itself.

1. **Origin tagging.** Every write carries an internal `origin` (`script:{pool}/{name}`, or `user:...`, `driver:...`), not part of the wire value object. With `ignore_own_writes: true` (default), a `tag_change` trigger of script X drops changes whose origin is X.
2. **Causation depth.** A change caused by a script write carries `depth = writer_depth + 1`; script runs triggered by it inherit that depth. Runs at depth greater than 8 are rejected (`rejected: cascade_depth`) and recorded. This catches A writes tag that triggers B that writes tag that triggers A.
3. **Rate budgets** (section 4) cap what survives both rules.
4. **Quarantine** (section 6) stops a script that keeps failing.

Events emitted by a script with a `subject` do not trigger that script (same origin rule for `event` triggers).

## 6. Failure states and quarantine

States of a script (mirrored in `_system/scripts/{name}/state`, P5):

| State | Meaning | Triggers accepted? |
|---|---|---|
| `ok` | enabled, pool available | yes |
| `disabled` | `enabled: false` | no |
| `quarantined` | `failure_policy.quarantine_after` consecutive failed runs (error, timeout, killed) | no |
| `unavailable` | pool absent or not attached | queued up to depth, then policy |
| `invalid` | source did not validate on a pool restart (e.g. library removed) | no |

- Quarantine exits on: a new revision touching the script, `POST /api/v1/scripts/{name}/reset`, or gateway restart (configurable: `failure_policy.reset_on_restart`, default true).
- Entering and leaving a state emits `skyranger.script.state_changed.v1` and updates the `_system` tags. Alarming on `_system/scripts/{name}/state` needs P6.
- Timers of quarantined scripts keep ticking silently as skipped; they are counted as `skipped_quarantine`.

## 7. Execution trace of one run

```
10:00:00.000 tag engine propagates flow_rate change              (Go, in-process)
10:00:00.001 script host: tag_change trigger matches low_flow_flag, origin check passes, offer to queue
10:00:00.002 dispatcher: pick idle worker #1, mint run_id + run_token, RunAssignment on Attach stream
10:00:00.004 scriptd: relay run to worker fd 3
10:00:00.006 worker: call tags.write -> scriptd -> Call(run_id, run_token, "tags.write", ...)
10:00:00.008 host: token ok, permission tags.write covers path, budget ok, same service function as REST
10:00:00.030 driver ack (write wait) -> result back through scriptd -> worker
10:00:00.031 worker: done ok -> scriptd Complete -> host: run record ring buffer, _system counters updated
```

(Timings are illustrative, not measured.)

## Decisions made
1. Five trigger kinds: `tag_change`, `timer`, `event`, `startup`, `manual`. Declarative in config, evaluated in Go.
2. Run-to-completion handlers; per-script FIFO with concurrency 1 by default; no long-running user loops, no `shutdown` trigger in v1.
3. Delivery is at-most-once with explicit, counted drops; tag changes coalesce latest-wins per tag; timers skip missed ticks rather than replaying them.
4. Backpressure never propagates into the tag engine; trigger queues are bounded and non-blocking.
5. Limits are per run and per second, defaults conservative, capped by a pool ceiling, all breaches become explicit errors with the API `code` vocabulary.
6. Write-loop protection by origin tagging, causation depth (max 8), rate budgets and quarantine.
7. Quarantine after 10 consecutive failures, visible as a `_system` tag and an event; leaves on new revision, manual reset or restart.
8. A manual run uses the script's permissions, never the caller's.
9. Config changes swap an immutable script revision atomically; in-flight runs finish on the revision they started with.

## Alternatives rejected and why
- **Long-running scripts with their own loops** (`while True: ... sleep`): cannot be timed out meaningfully, hold workers forever, make shutdown and config changes hard. Timers cover the real needs. Revisit as an explicit `service` kind in v2 (open question 4).
- **Script-defined subscriptions at runtime** (`tags.subscribe(callback)`): invisible to config diff, dry-run and permission review. Declarative triggers are inspectable.
- **At-least-once with an unbounded durable queue**: replays stale actions after an outage, grows without bound, and the tag engine would have to care. Choose durable delivery per event type later, bounded.
- **Replay missed timer ticks**: dangerous side effects after downtime; `catch_up` is opt-in and single-shot.
- **Run the script on the triggering thread/goroutine**: couples tag engine latency to user code.
- **Unlimited concurrency**: needs locking in user scripts; the default must be the safe, boring one.
- **Cron library with seconds fields and non-standard extensions**: five-field cron plus `every` covers it.
- **Retry on failure automatically**: a retried script that wrote half its tags may duplicate effects; retries are the script author's explicit choice.

## Open questions
1. Is `coalesce` the right default for every `tag_change` script? A script that must see every `bool` edge can opt into `drop_oldest` with a larger queue, but edge-triggered semantics might deserve an `edges: true` guarantee.
2. Per-script `priority` for dispatch (alarm-handling scripts over report scripts)? v1 relies on fair round-robin and the pool ceiling.
3. Choice of cron library and its DST behavior (skipped and repeated local times): UNVERIFIED, needs tests.
4. A `service` kind (long-running, supervised) for state machines: v2 candidate, likely a plugin extension point (09) rather than scripts.
5. A `shutdown` trigger with a short deadline for graceful `SIGTERM` stops; excluded now because it cannot be relied on.
6. Durable event delivery (bounded, per type) for scripts that must not miss events during pool restarts.
7. `where` filters on events: stay equality-only, or adopt a standard expression (CEL) later? Keeping it tiny avoids a third language.
8. Per-tag rate caps for `tag_change` fan-in on large prefixes: draft warnings only for now.
