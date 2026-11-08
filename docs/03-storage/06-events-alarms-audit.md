# 06 Events, Alarm Journal and Audit Storage

Status: draft. This file covers how the three journals are stored and written, not what they mean. Meaning: events in messages.md section 4, alarm states in messages.md section 5 and the alarm area ([04-alarms](../04-alarms/README.md)), audit in [07-security/08](../07-security/08-audit-logging.md), which owns the `audit_log` DDL, the hash chain, the checkpoint signing and the event list. Nothing here changes those.

## 1. Comparison

| Stream | Table | Volume | Writer | If PostgreSQL is down | Retention | Tamper evidence |
|---|---|---|---|---|---|---|
| Events (CloudEvents) | `events`, hypertable on `time` | tens per day (meter) to thousands per day (large plant), bursts in alarm floods | event writer, batched | spool stream `events` (04), replay idempotent | = raw history retention (07-security/08 section 4) | none (operational data) |
| Alarm state | `alarm_state`, plain | one row per configured alarm | alarm engine persister | spool stream `alarms`, ordered; in-memory state continues | n/a (current state) | n/a |
| Alarm journal | `alarm_journal`, hypertable on `ts` | one row per transition | alarm engine persister | same spool | `journal_retention` (alarm config, `1095d` placeholder) | none |
| Audit | `audit_log`, plain | tens to hundreds per day | request handlers, in the same transaction as the change | **fail closed**: the mutating request fails | 7 years default, prefix pruning | hash chain, signed checkpoints, off-box copies |

## 2. Events

### Write

- Every event emitted by any component goes through one `EventWriter` (a bounded channel and a single goroutine, same pattern as the historian writer, 03). It batches up to 200 events or 200 ms and runs:

```sql
INSERT INTO events (id, time, type, source, subject, tag_id, data)
SELECT * FROM unnest($1::uuid[], $2::timestamptz[], $3::text[], $4::text[], $5::text[], $6::uuid[], $7::jsonb[])
ON CONFLICT (time, id) DO NOTHING;
```

- `time` is the CloudEvents `time` attribute as emitted, not the insert time, so a replayed row has the identical key and dedupes. `id` is the CloudEvents `id` (UUIDv7).
- `tag_id` is extracted from `data.tag_id` when the event type's schema has one (`counter_discontinuity`, `write_completed`, `alarm.state_changed`); this makes "all events for this tag" an index lookup and keeps working after a tag rename (the `subject` path is the path at the time).
- `data` is capped at 64 KiB by the writer (events are small by contract; a larger payload is truncated into `{"truncated":true,"original_size":N}` and counted). This protects the table from a buggy emitter.
- **Alarm-originated events are not written by this writer.** They are written in the alarm persister's transaction (section 3) so that state, journal and event commit together.
- The **MQTT publish** of an event happens after the database write (or after the spool append), never before. A subscriber must not see an event that a restart would then lose (consistent with "persist, then announce", alarm README).

### Read (api.md `GET /events`)

Parameters: `type` (prefix, repeated), `subject_prefix`, `start`, `end`, `cursor`, `limit` (default 100, max 1000). Keyset pagination on `(time DESC, id DESC)`; the cursor is an opaque encoding of the last `(time, id)`.

```sql
SELECT id, time, type, source, subject, data
FROM events
WHERE time >= $start AND time < $end
  AND (cardinality($types) = 0 OR EXISTS (SELECT 1 FROM unnest($types) p WHERE type >= p AND type < p || chr(1114111)))
  AND (cardinality($subj)  = 0 OR subject = $subj OR (subject >= $subj || '/' AND subject < $subj || '0'))
  AND ($cursor_time IS NULL OR (time, id) < ($cursor_time, $cursor_id))
ORDER BY time DESC, id DESC
LIMIT $limit + 1;
```

`type` and `subject` are `COLLATE "C"` so the half-open range is a byte prefix match and uses `events_type_time` and `events_subject_time` (01). Row-value comparison `(time, id) < (...)` gives a stable, index-friendly keyset. A `+1` row tells whether `next_cursor` is needed. `chr(1114111)` is U+10FFFF, the highest code point, so `p <= type < p || U+10FFFF` is every string starting with `p` (UNVERIFIED that the database encoding treats this bytewise as intended under C collation; unit-test with multi-byte subjects).

### Size

Row about 300 to 600 bytes with its three indexes. At 5,000 events/day (an active 10,000-tag plant, assumption) and 400 days: 2 M rows, about 1 GB uncompressed. Meter: 20 events/day, negligible. Compression is not enabled in v1 (02); it would be a two-line change.

## 3. Alarm persistence (storage side of "persist, then announce")

The alarm engine produces `Transition` records with a per-alarm gapless `seq` (04-alarms README, 02-state-machine). The **persister** commits batches within 50 ms (group commit) in one transaction:

```sql
-- 1. state: only moves forward
INSERT INTO alarm_state (alarm_id, tag_id, name, severity, state, shelved_until, enabled, message,
                         value, quality, active_at, cleared_at, acked_at, acked_by, updated_at, last_seq)
SELECT * FROM unnest(...)
ON CONFLICT (alarm_id) DO UPDATE SET
  state = EXCLUDED.state, shelved_until = EXCLUDED.shelved_until, enabled = EXCLUDED.enabled,
  message = EXCLUDED.message, value = EXCLUDED.value, quality = EXCLUDED.quality,
  active_at = EXCLUDED.active_at, cleared_at = EXCLUDED.cleared_at,
  acked_at = EXCLUDED.acked_at, acked_by = EXCLUDED.acked_by,
  updated_at = EXCLUDED.updated_at, last_seq = EXCLUDED.last_seq
WHERE alarm_state.last_seq < EXCLUDED.last_seq;         -- stale replay cannot move state backwards

-- 2. journal: idempotent
INSERT INTO alarm_journal (...) SELECT * FROM unnest(...) ON CONFLICT (alarm_id, seq, ts) DO NOTHING;

-- 3. the CloudEvent skyranger.alarm.state_changed.v1
INSERT INTO events (...) SELECT ... ON CONFLICT (time, id) DO NOTHING;

-- 4. notification_outbox rows (alarm area DDL), same transaction
```

- **Idempotent replay.** The spool replays `alarms` strictly in append order (04). A transition replayed after it already committed hits `ON CONFLICT DO NOTHING` in the journal, the guarded `WHERE last_seq <` leaves state alone, and the event insert dedupes. All three are idempotent by the transition's own keys, so a crash between commit and cursor update is harmless.
- **`SyncNow` spool durability for DB-down:** an alarm transition is acknowledged to the alarm engine as persisted once it is in the fsync'd spool, which is the same promise as a commit (04, section 9). The engine announces (WS, MQTT, notifications) only after that point.
- **Startup:** the alarm engine loads `alarm_state` for all alarms in one query, then replays the `alarms` spool (if any), then evaluates. Orphan rows (alarm removed from config) are marked by the config apply (alarm area) and cleared after `journal_retention`.
- **Hot row concern:** an alarm that chatters (hundreds of transitions per minute) updates one `alarm_state` row repeatedly. Fillfactor 80 and the alarm area's chatter suppression (04-alarms/04) keep this cheap; at the extreme (10,000 alarms flooding in 10 minutes) the persister's group commit turns it into a handful of transactions of a few thousand rows each.
- Retention of `alarm_journal` is a Timescale retention policy at `journal_retention` (02, section 6). The alarm area's KPI queries (04-alarms/08) read the journal and the events table; their SQL belongs there. Indexes provided: `(ts DESC)` for time windows and `(path, ts DESC)` for per-area history. A per-alarm index is the primary key prefix `(alarm_id, seq, ts)`.

### `GET /alarms/history` shape

```sql
SELECT ts, alarm_id, seq, path, alarm_name, severity, transition, from_state, to_state, value, quality, actor, comment
FROM alarm_journal
WHERE ts >= $start AND ts < $end
  AND ($prefix IS NULL OR path = $prefix OR (path >= $prefix || '/' AND path < $prefix || '0'))
  AND ($cursor_ts IS NULL OR (ts, alarm_id, seq) < ($cursor_ts, $cursor_alarm, $cursor_seq))
ORDER BY ts DESC, alarm_id DESC, seq DESC
LIMIT $limit + 1;
```

## 4. Audit storage discipline

The DDL, the event list, the hash chain, signing and verification are owned by [07-security/08](../07-security/08-audit-logging.md). Storage adds the following operational rules, which that file's section 4 requires ("fail closed for changes, same transaction where possible") but does not specify mechanically:

1. **Lock order.** Every transaction that needs both takes the **config lock (`7301`) before the audit lock (`7302`)**. Anything else can deadlock a config apply against a user login. A request that only audits takes only `7302`.
2. **Append procedure** (inside the caller's transaction):
   ```
   SELECT pg_advisory_xact_lock(7302);
   SELECT seq, hash FROM audit_log ORDER BY seq DESC LIMIT 1;          -- O(1) via the primary key
   compute canonical JSON, then hash := SHA-256(prev_hash || canonical)  -- in Go (package canonjson, shared with 05)
   INSERT INTO audit_log (seq = last+1, prev_hash = last.hash, hash, ...);
   ```
   The lock is held until commit, so `seq` is gapless and the chain cannot fork. A rollback after the insert releases the number with no gap, because the row never committed.
3. **Throughput.** The lock serializes audit writes. At the plausible peak of tens of audited actions per second (a bulk alarm acknowledge produces one record per alarm, up to the number of active alarms), a transaction of a few hundred inserts with one lock is milliseconds. This is ample; the design is **not** intended for per-read auditing (07-security/08 section 1 excludes reads).
4. **Two writers, one chain.** `sr_audit_writer` does the insert (07-security/08 section 3). The application connects as `sr_app` and uses `SET LOCAL ROLE sr_audit_writer` for the audit insert statement within the transaction, or a dedicated `state` sub-pool authenticated as that role if `SET ROLE` is not acceptable to the security design (UNVERIFIED interaction with `pgx` connection reuse; either works). `sr_app` itself has no `INSERT` on `audit_log`, so a bug in a normal handler cannot write or alter audit rows.
5. **Backup and verify.** `audit_log` is included in every backup (07). After any restore, `skyrangerctl audit verify` runs automatically before `skyrangerd` starts accepting writes (07-security/08 section 3, point 5). A mismatch refuses start in "safe mode" (read-only API, no new audit records) rather than extending a possibly forked chain. The deployment area owns safe mode.
6. **Pruning.** Retention pruning deletes whole chain prefixes. It needs `DELETE` on `audit_log`, which `sr_audit_writer` does not have, so pruning is done by a third role `sr_audit_pruner`, used only by `skyrangerctl audit prune` (admin, itself audited as `audit.pruned`). It deletes `seq < K` in one transaction after writing the `audit.pruned` record whose `after` carries `K` and the `prev_hash` of the first remaining row, as 07-security/08 section 4 specifies. The trigger that rejects `DELETE` makes one exception: it allows the statement only when `current_user = 'sr_audit_pruner'` and the transaction-local setting `skyranger.audit_prune` is `on`, which only the prune command sets. (Proposed mechanism; the security area owns the final design.)
7. **Size and partitioning.** About 1 KB per record. At 1,000 records/day, 7 years is 2.5 M rows, about 2.5 GB. Not a hypertable and not partitioned: an unpartitioned table of this size is routine, and keeping it ordinary avoids Timescale-specific tooling in the one table that must be verifiable with plain SQL on any PostgreSQL. If a deployment audits at 100,000 records/day, range-partitioning by `seq` is the escape hatch (partition drop would then implement pruning); it is not needed within this design's scale.
8. **Sensitive reads** (`audit.read`, `history.exported`) write audit records on the **query** side: the export handler appends its record before it starts streaming.

## 5. When PostgreSQL is down: summary

| Operation | Behavior | API response while down |
|---|---|---|
| Live values, WS, MQTT, drivers, alarm evaluation | continue in memory | normal |
| History | spooled (04) | `/history` returns `503` |
| Alarm transitions | spooled with `SyncNow`, announced after the spool append | `/alarms` current state served from memory; `/alarms/history` `503` |
| Events | spooled | `/events` `503` |
| Write to a `memory` tag | **rejected** (must commit before `ok`, pipeline.md section 9) | `503` |
| Tag write to a device | allowed if its audit record `tag.write_requested` can be committed first; **not allowed** while the DB is down (fail closed, 07-security/08) | `503` |
| Config draft/apply, user and key admin, ack and shelve | rejected (audit fail closed). **Alarm acknowledge is the one to reconsider**: an operator unable to acknowledge an alarm because the historian database is down is a safety-relevant annoyance | `503` |
| `GET /system/health` | `status: degraded` with `checks.database: "down"`; unauthenticated, no tag data (api.md) | `200` |

api.md has no 503 code for "database unavailable" other than `not_connected` (which means a device connection) or `internal`. A new additive code `unavailable` (HTTP 503, `Retry-After`) is proposed.

## Decisions made
1. Events, alarm transitions, config apply and audit each write in their own transaction shape, but all use idempotent keys, so spool replay and crash recovery are safe by construction.
2. Alarm state, journal and event commit together; the alarm announces only after that point or after an fsync'd spool append.
3. Audit writes take a transaction-scoped advisory lock with a fixed lock order (config before audit), execute as a restricted role, and fail closed.
4. `type` and `subject` are byte-collated; all prefix queries use half-open byte ranges and keyset pagination on `(time, id)`.
5. `audit_log` stays an ordinary table; pruning uses a dedicated, audited role.
6. Events and the alarm journal are not compressed in v1.

## Alternatives rejected and why
- **Putting events and the journal in one table**: different retention and different query shapes; two tables with a shared `event_id` costs a few hundred bytes per alarm transition.
- **Audit as a hypertable**: the chain and prefix-pruning want a plain, auditable table; the volume is tiny.
- **`LISTEN/NOTIFY` for announcing committed events**: the application already knows when it committed; adding a database-side notification path creates a second ordering to reason about.
- **Offset pagination**: api.md rejects it; keyset is also faster.
- **Audit via logical decoding or triggers capturing every table change**: records what changed but not who and why (no actor, comment, request id) and would audit internal churn.
- **Letting alarm acknowledge succeed while the DB is down and reconcile later**: tempting for operators, but it creates an unaudited state change. Left as an open question for the alarm and security owners rather than decided here.

## Open questions
- **Alarm acknowledge when the database is down** (section 5). Options: allow it into the `alarms` spool with a synthesized audit record spooled alongside, replayed in order; or keep fail-closed. Needs a joint decision (alarm, security).
- `SET LOCAL ROLE sr_audit_writer` versus a dedicated pool; trigger-bypass design for the pruner. Security owns the final mechanism.
- New additive API code `unavailable` (503) and event/alarm contract additions.
- Does `events` need `tag_id` extraction for events other than the three named types (e.g. `device_announced`)? Add per type as needed.
- Exact U+10FFFF range behavior under `COLLATE "C"` (UNVERIFIED), versus using `starts_with(type, p)` with a trigram or SP-GiST index.
