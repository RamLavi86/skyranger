# 05 Errors, run records and logging

Status: design draft. Conforms to [../00-contracts/](../00-contracts/README.md). Depends on P3 to P6 (see [README.md](README.md)). Related: [03-triggers-and-execution.md](03-triggers-and-execution.md), [../07-security/08-audit-logging.md](../07-security/08-audit-logging.md).

## 1. Principle

A failing script must be **visible** to an engineer within seconds, **harmless** to the plant, and **reconstructable** later: which revision, which trigger, what it called, what it wrote, why it died.

## 2. What happens when a handler raises

1. The worker catches the exception at the top of the run, formats `{type, message, traceback}` (traceback trimmed to 8 KiB and with frames for the bootstrap removed), sends `done ok=false`.
2. scriptd sends `Complete` with status `error`. The host writes a run record (section 3), increments counters, updates `_system/scripts/{name}/...` (section 4), emits `skyranger.script.run_failed.v1` (P3) and logs at `error`.
3. After `failure_policy.quarantine_after` consecutive failures the script enters `quarantined` (03 section 6).
4. **Nothing is retried automatically.** Effects already done (writes, events, HTTP requests) are not undone; they are in the audit trail.

Run statuses:

| `status` | Meaning |
|---|---|
| `ok` | handler returned |
| `error` | handler raised (`error.type` is the Python class) |
| `timeout` | soft timeout fired or hard kill after deadline |
| `killed` | worker killed by scriptd: `reason` in `memory`, `crash`, `cancelled` (config removed) |
| `rejected` | never started: `cascade_depth`, `quarantined`, `pool_unavailable`, `invalid` |
| `skipped` | trigger accepted but not run: `overlap` (timer), `queue_overflow` |

Only `ok` and `error|timeout|killed` runs executed code. `rejected` and `skipped` are counted, and individually recorded only when they are failures of the system rather than of the script (see section 3 sampling).

## 3. Run records

Table `script_runs` (PostgreSQL, a regular table with time-based retention; not a hypertable unless volume demands it):

| Column | Type | Notes |
|---|---|---|
| `run_id` | uuid (v7) | primary key |
| `script` | text | name |
| `pool` | text | |
| `config_revision` | bigint | the revision the run executed (traceability, 06) |
| `source_sha256` | text | hash of the source (library hashes in `libs`) |
| `trigger` | jsonb | kind + details; for `tag_change`: `path`, new value+quality+ts |
| `requested_by` | text | `system` or `user:alice` for manual |
| `status` | text | section 2 |
| `started_at`, `finished_at` | timestamptz | |
| `duration_ms` | integer | internal column; API renders a Go duration string |
| `calls` | jsonb | per method counts: `{"tags.read": 3, "tags.write": 2, "http.request": 1}` |
| `writes` | jsonb | list of `{path, value, write_id, result}` for writes (capped at 100 entries) |
| `error` | jsonb | `{type, message, traceback}` or null |
| `dry_run` | boolean | |

**Sampling to keep volume sane.** A tag-change script on a fast tag can run thousands of times a day. Rules:
- Every non-`ok` run is stored in full.
- `ok` runs: the last 100 per script are kept in a memory ring (served by the API), and `ok` runs are written to `script_runs` only for `manual` and `dry_run` triggers, `timer` and `startup` runs (low volume), and 1 in `N` of `tag_change`/`event` runs (default `N = 100`, plus the first run after each revision change).
- **Every write is audited regardless of sampling** (audit_log, 08-audit-logging.md), so "who changed this tag" never depends on the sample.
- Retention default: 30 days (`Gateway`-level setting, schema to be added with P2). Counters live forever in `_system` tags and history.

## 4. `_system` tags (P5)

Created by the host, read-only, may change between versions (tag-model.md section 1). Per script:

| Path | datatype | Unit | Meaning |
|---|---|---|---|
| `_system/scripts/{name}/state` | string | | `ok`, `disabled`, `quarantined`, `unavailable`, `invalid` (03 section 6) |
| `_system/scripts/{name}/runs` | uint64 | `1` | executed runs since gateway start |
| `_system/scripts/{name}/failures` | uint64 | `1` | failed runs since start |
| `_system/scripts/{name}/consecutive_failures` | int64 | `1` | |
| `_system/scripts/{name}/last_run_at` | datetime | | |
| `_system/scripts/{name}/last_duration` | float64 | `s` | seconds of the last run |
| `_system/scripts/{name}/last_error` | string | | `Type: message` of the last failure, max 256 bytes |
| `_system/scripts/{name}/queue_depth` | int64 | `1` | |
| `_system/scripts/{name}/dropped` | uint64 | `1` | queue overflow drops |

Per pool: `_system/script_pools/{pool}/state` (`ok`, `degraded`, `unavailable`), `workers` (int64), `restarts` (uint64), `kills_timeout`, `kills_memory`, `rss_bytes` (uint64, unit `By`; the UCUM code for bytes is UNVERIFIED, tag-model.md already notes UCUM checks for dBm), `run_queue_depth`.

Alarming on these requires P6 (an alarm binding to `_system` tags). Until then the HMI shows them directly and the event `skyranger.script.run_failed.v1` is the notification hook (a script can even watch for failures of other scripts via an event trigger, carefully).

## 5. Events (P3)

| `type` | `data` |
|---|---|
| `skyranger.script.run_failed.v1` | `{"script", "run_id", "config_revision", "status": "error"\|"timeout"\|"killed", "error": {"type","message"}, "trigger": {"kind"}, "consecutive_failures"}` |
| `skyranger.script.state_changed.v1` | `{"script", "state", "previous_state", "reason"}` |
| `skyranger.script.run_requested.v1` | `{"script", "run_id", "requested_by", "params"}` (manual runs) |
| `skyranger.user.{name}.v1` | script-defined JSON object, max 16 KiB |

`source` for all of them is `/gateways/{gateway_id}/script-host` (P3 adds the component). `subject` is the script name for the first three, the user-supplied subject for user events. A flood guard limits `run_failed` events to 1 per script per 10 s (with a suppressed-count field in the next one) so a failing 50-per-second trigger does not flood the event table.

## 6. Logging

Three streams, each with a clear owner:

1. **Script logs** (what authors write: `log.info(...)` and captured `print`/stderr): structured `{ts, level, msg, fields, run_id, script, stream}`. Line cap 4 KiB; per run caps in 03. Stored in a per-script ring in the host (last 1000 lines, served by `GET /scripts/{name}/logs`), written to the container's stdout as `slog` JSON with `component=script`, and persisted in `script_runs.error`/an attached log excerpt (last 50 lines) for non-ok runs. Not retained long-term in PostgreSQL in v1 (log volume is user-controlled; use the container log driver and the syslog forwarding the security doc defines).
2. **Platform logs** (scriptd and host): `log/slog` JSON, `SKYRANGER_LOG_LEVEL`, same conventions as drivers (driver-interface.md section 1): fields `pool`, `script`, `run_id`, `worker`, `event`. Worker kills log the reason and RSS.
3. **Audit** (what the script *did*): tag writes, alarm acks, event emits, HTTP requests, state changes of scripts, config changes to scripts. Principal `script:{pool}/{name}`, with `run_id` in the record. Owned by 08-audit-logging.md; this area supplies the fields.

Rules:
- Logs never contain secrets: scripts have none; the host never logs headers or bodies of HTTP calls.
- Script log level is per script (`spec.log_level`, default `info`); `debug` lines are dropped at the worker when disabled.
- All log timestamps RFC 3339 UTC ms, durations Go strings (contracts).

## 7. REST endpoints (P4)

Additive to api.md, same auth, pagination, error format (RFC 9457), cursor pagination. Roles: `viewer` read, `engineer` run and reset.

| Method | Path | Role | Notes |
|---|---|---|---|
| GET | `/scripts` | viewer | `{"items": [{"name","display_name","enabled","state","pool","config_revision","last_run_at","last_status","triggers":[kinds]}], "next_cursor"}` |
| GET | `/scripts/{name}` | viewer | Definition (without `source` unless `?include_source=true`) plus live counters |
| POST | `/scripts/{name}/run` | per `manual.min_role`, default engineer | Body `{"params": {...}, "dry_run": false}`. `202` `{"run_id"}`. `?wait=10s` blocks and returns the run record |
| POST | `/scripts/{name}/reset` | engineer | Leave quarantine |
| GET | `/scripts/{name}/runs` | viewer | Run records. Query `status`, `start`, `end`, `cursor` |
| GET | `/scripts/runs/{run_id}` | viewer | One record |
| GET | `/scripts/{name}/logs` | viewer | Ring buffer lines. Query `run_id`, `level`, `limit` |
| POST | `/scripts/dry-run` | engineer | Body `{"document": <Script>, "trigger": {...}, "params": {...}}` for an unsaved draft; see [07-testing.md](07-testing.md) |
| GET | `/script-pools` | viewer | `[{"name","state","workers","version","rss_bytes"...}]` |

Run record JSON (field names are the `script_runs` columns, `duration` as a Go string):

```json
{
  "run_id": "01933b90-4c1d-7e2f-8a9b-0c1d2e3f4a5b",
  "script": "daily_report",
  "config_revision": 43,
  "source_sha256": "9f2c...e1",
  "status": "ok",
  "trigger": { "kind": "timer", "scheduled_for": "2026-11-08T00:05:00.000Z" },
  "requested_by": "system",
  "started_at": "2026-11-08T00:05:00.003Z",
  "finished_at": "2026-11-08T00:05:00.211Z",
  "duration": "208ms",
  "calls": { "history.read": 1, "tags.write": 1, "events.emit": 1, "state.set": 1 },
  "writes": [ { "path": "site1/pumphouse/meter01/yesterday_l", "value": 4821.0, "result": "ok" } ],
  "error": null,
  "dry_run": false
}
```

Config for scripts goes through the existing `/config` endpoints (06); there are no script CRUD endpoints, so there is one place to edit and one audit path.

HMI: a "Scripts" admin page (state, last run, errors, run button) belongs to the HMI area; this contract is enough for it.

## Decisions made
1. Failure is visible three ways at once: run record, `_system/scripts/...` tags, and `skyranger.script.run_failed.v1`; nothing is retried.
2. Run records for failures are always stored; successful high-volume runs are sampled; **every write is audited regardless**.
3. Script logs are structured, bounded, ring-buffered in the host, mirrored to container stdout; long-term retention belongs to the platform's log forwarding, not PostgreSQL, in v1.
4. Events and tags follow the contract naming (`skyranger.script.*.v1`, `_system/scripts/{name}/...`), with flood guards.
5. No script CRUD endpoints: scripts are edited through config drafts only.
6. `run_failed` events are rate-limited per script with a suppressed counter.

## Alternatives rejected and why
- **Store every run**: a 5-per-second trigger would produce 400k rows a day for no operational value.
- **Auto-restart/retry failed runs**: repeats half-done side effects.
- **Send script logs to MQTT**: no consumer needs it, adds broker load.
- **Script CRUD REST API**: second path around config revisions, diff and validation.
- **Alert by email from the platform directly**: notification channels are an extension point (09); in v1 failures are events and `_system` tags, so any alarm or script can react.
- **Persist full tracebacks forever**: they can include user data; bounded size and retention instead.

## Open questions
1. Retention setting location: a `Gateway.spec.scripts` block (additive field to the Gateway kind) is assumed; needs sign-off with P2.
2. Should sampled `ok` runs record the sample rate so dashboards can extrapolate? Currently counters in `_system` tags give exact totals.
3. Run record `writes` capture: values may be large (strings); cap size per entry.
4. Whether `_system` bytes tag should declare unit `By` (UCUM bytes code UNVERIFIED).
5. Event flood guard window (10 s) is arbitrary; tune on real failure patterns.
6. Where script logs go for customers that want them in an external system: syslog forwarding via the platform (security area) or a log-shipping sidecar.
