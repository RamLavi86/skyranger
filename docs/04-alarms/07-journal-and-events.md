# 07 Alarm Journal and Event Model

Three stores, each with one job:

| Store | Job | Written by | Retention |
|---|---|---|---|
| `alarm_state` | The **current** state of each alarm (one row per `alarm_id`). Source for `GET /alarms`, restart recovery, WS snapshots | persister (upsert) | forever (one row per alarm) |
| `alarm_journal` | The **immutable history**: every transition, flag change and operator action, one row each. Source for reports, KPIs, `GET /alarms/history` | persister (insert only) | `Gateway.spec.alarms.journal_retention` (placeholder `1095d`, 01) |
| `events` | The contract's generic CloudEvents table (messages.md s4). Every alarm change is also an event, so integrations and `GET /events` see one stream | persister | the events table's own retention (tag-model/historian owner) |

Why a journal beside `events`: alarm questions ("who acked it, how long to ack, how many times this month, what was the value") need typed, indexed columns and a longer retention than a generic event stream. The journal row and the event are written in the **same transaction** and share an id (`event_id`), so they cannot disagree.

## 1. Event model

### 1.1 Contract event reused

`skyranger.alarm.state_changed.v1`, `data` = alarm state object (02 s1) plus `previous_state` (contract), plus additive fields:

```json
{
  "specversion": "1.0",
  "id": "01933b81-5a6b-7c7d-8e9f-a0b1c2d3e4f5",
  "source": "/gateways/gw01/alarm-engine",
  "type": "skyranger.alarm.state_changed.v1",
  "time": "2026-11-07T00:00:30.000Z",
  "subject": "site1/pumphouse/meter01/leak",
  "datacontenttype": "application/json",
  "data": {
    "alarm_id": "01933b81-0a1b-7c2d-8e3f-405162738495",
    "tag_id": "01933b7e-8f1a-7c3d-9a2b-4f6e8d1c2b3a",
    "path": "site1/pumphouse/meter01/leak",
    "name": "leak",
    "severity": "high",
    "class": "environmental",
    "state": "active_unacked",
    "previous_state": "normal",
    "shelved_until": null,
    "suppressed": false, "chattering": false, "held": false, "enabled": true,
    "annunciation": "annunciated",
    "message": "Flow has not stopped for 2h: possible leak",
    "value": true, "quality": "good",
    "active_at": "2026-11-07T00:00:30.000Z", "cleared_at": null,
    "acked_at": null, "acked_by": null,
    "updated_at": "2026-11-07T00:00:30.000Z",
    "seq": "7", "activation_count": "7",
    "cause": "activated", "actor": "system:engine", "comment": null
  }
}
```

One event type covers **every** change of the alarm object: state transitions and flag changes alike. The additive field `cause` says why:

`activated`, `reactivated`, `cleared`, `acked`, `shelved`, `unshelved`, `shelve_expired`, `suppressed`, `unsuppressed`, `chatter_start`, `chatter_end`, `held`, `released`, `out_of_service`, `returned_to_service`, `reconfigured`, `removed`, `recovered` (restart, 09).

Consumers that do not know a `cause` must treat it as a generic update; the `state` field is always sufficient to know the state. `data.subject` of the event envelope is the tag path; the alarm name is in `data.name`.

### 1.2 Two proposed new event types

| `type` | `data` |
|---|---|
| `skyranger.alarm.flood_changed.v1` | `{"console": "water", "flood": true, "previous": false, "count": 14, "window": "10m", "top": [{"alarm_id", "path", "name", "severity"}]}` (top 5) |
| `skyranger.alarm.delivery_failed.v1` | `{"delivery_id", "alarm_id", "alarm_seq": "7", "channel": "sms_modem", "contact": "oncall_phone", "kind": "activate", "status": "dead"\|"expired", "attempts": 9, "last_error": "..."}` |

Deliveries that succeed do **not** produce events (volume); they are in the outbox and a compact `notified` journal row.

### 1.3 Events that are not alarms

Examples that stay events: connection state changes, counter discontinuities, config revisions, write completions (messages.md). They never enter the alarm state machine. A rule for authors: if an operator is expected to act, make it an alarm on a tag (for example a `quality` alarm for a connection); otherwise it stays an event. This is the README's alarm-versus-event definition applied.

### 1.4 Bus publication

State object retained on `skyranger/v1/{gateway_id}/alarms/{tag_path}/{alarm_name}` (messages.md s3), published after commit. `state_changed` events on `skyranger/v1/{gateway_id}/events/skyranger.alarm.state_changed.v1`. A removed alarm gets an empty retained message to clear the topic. Publication is after commit and retried by a bus publisher goroutine; if the broker is down, the DB is still correct, and on reconnect the publisher republishes the **current state of all alarms** (retained) in order to resynchronize subscribers; the event stream may lose events during a broker outage, so integrations that need completeness use `GET /events`, `GET /alarms/history` or the journal cursor (09).

## 2. Journal table

```sql
CREATE TABLE alarm_journal (
  id              uuid        NOT NULL,                 -- UUIDv7 (sync cursor, 09)
  event_id        uuid        NOT NULL,                 -- CloudEvent id; every row has one
  ts              timestamptz NOT NULL,                 -- when it happened (sample ts / command time)
  ts_gateway      timestamptz NOT NULL,                 -- when the gateway committed it
  alarm_id        uuid        NOT NULL,
  tag_id          uuid        NOT NULL,
  path            text        NOT NULL,                 -- path at the time (paths are mutable)
  alarm_name      text        NOT NULL,
  seq             bigint      NOT NULL,
  kind            text        NOT NULL,                 -- the `cause` values above, plus notified, escalated, notification_expired, chatter_edges, engine_gap
  from_state      text,
  to_state        text,
  severity        text        NOT NULL,
  class           text        NOT NULL,
  console         text,
  value           jsonb,                                -- the sample value at the time
  quality         smallint,
  setpoint        jsonb,                                -- resolved setpoint (setpoint_ref values are captured)
  condition_since timestamptz,                          -- start of the raw condition (on-delay visible)
  actor           text        NOT NULL,                 -- system:engine | user:alice | apikey:hq-scada | system:...
  comment         text,
  shelved         boolean     NOT NULL DEFAULT false,
  suppressed      boolean     NOT NULL DEFAULT false,
  config_revision int         NOT NULL,
  detail          jsonb,                                -- kind-specific (tier, channel, counts, gap bounds)
  PRIMARY KEY (ts, id)
);
SELECT create_hypertable('alarm_journal', 'ts');       -- TimescaleDB, chunk 7 days (UNVERIFIED tuning)
CREATE INDEX ON alarm_journal (alarm_id, ts DESC);
CREATE INDEX ON alarm_journal (console, ts DESC);
CREATE UNIQUE INDEX ON alarm_journal (event_id, ts);   -- idempotent replay; unique indexes on a hypertable must include ts (UNVERIFIED)
```

Rules:
- **Insert-only.** No updates, no deletes except retention. The application role has `INSERT, SELECT` only on this table; retention runs under a different role.
- **Self-describing.** `path`, `severity`, `class`, setpoint and `config_revision` are copied into the row. Reports from 2027 must still say what the alarm was in 2026 even if it was renamed, re-prioritized or deleted.
- **Idempotent.** unique `(event_id, ts)`; spool replay keeps the original `ts`, so duplicates collide; replay (09) ignores duplicates with `ON CONFLICT DO NOTHING`.
- **Compression**: chunks older than 30 days compressed (TimescaleDB feature; settings UNVERIFIED), segment by `alarm_id`.
- `kind = 'chatter_edges'` rows carry `detail: {"count": 41, "first_ts": "...", "last_ts": "..."}` (04 s4).
- `kind = 'engine_gap'` is written at startup and covers the interval the engine was not running (09).

### What gets a row

| Event | Row `kind` | Notes |
|---|---|---|
| Activation (also re-activation) | `activated` / `reactivated` | `condition_since` set |
| Return to normal | `cleared` | |
| Ack (including system acks) | `acked` | `comment`, `actor` |
| Shelve, unshelve, expiry | `shelved` / `unshelved` / `shelve_expired` | `detail: {"until"}`, comment |
| Suppression on/off | `suppressed` / `unsuppressed` | `detail: {"reason"}` |
| Quality hold on/off | `held` / `released` | `detail: {"quality"}`; held rows are emitted only after the hold has lasted `> 1m`, with `condition_since` = hold start, to avoid journaling flickers |
| Out of service, back | `out_of_service` / `returned_to_service` | |
| Config change affecting the alarm | `reconfigured` | `detail: {"revision", "diff"}` |
| Notification success, escalation | `notified`, `escalated` | compact |
| Engine downtime | `engine_gap` | one per restart per gateway, written as one row per **active** alarm (so each alarm's timeline shows the gap) |

## 3. Queries

All use the journal. REST mapping: `GET /alarms/history?start&end&prefix&cursor` (api.md) is a keyset-paginated select by `(ts, id)` where cursors are opaque.

Activation history of an alarm:
```sql
SELECT ts, kind, to_state, actor, comment, value
FROM alarm_journal
WHERE alarm_id = $1 AND ts >= $2 AND ts < $3
ORDER BY ts, id;
```

Time to acknowledge (per activation):
```sql
SELECT a.alarm_id, a.seq, a.ts AS active_at, k.ts AS acked_at, k.ts - a.ts AS time_to_ack
FROM alarm_journal a
JOIN LATERAL (SELECT ts FROM alarm_journal k
              WHERE k.alarm_id = a.alarm_id AND k.kind = 'acked' AND k.ts >= a.ts
              ORDER BY k.ts LIMIT 1) k ON true
WHERE a.kind IN ('activated','reactivated') AND a.ts >= now() - interval '30 days';
```

"Sequence of events" report across alarms for an incident (SOE), ordered by `ts` with sub-ordering `id` (UUIDv7 is time-ordered, same ms ties resolved by id).

Event stream cursor for integrations: `GET /events?type=skyranger.alarm.&start=...` returns events in `time, id` order.

## 4. Retention and export

- Journal retention is configured, not hard-coded; deletion by dropping chunks. Any customer-specific regulatory period is UNVERIFIED (01).
- Export: `GET /alarms/history` supports `Accept: text/csv` (same convention as history export, api.md) with columns `ts,path,alarm,kind,from_state,to_state,severity,actor,comment,value,quality`.
- Config export (decision 3) includes alarm definitions but not the journal. A journal dump is a plain SQL/CSV export (no lock-in).
- Backup: the journal is in the same PostgreSQL as config; the restore procedure (deployment doc) must restore journal and `alarm_state` from the same point.

## Decisions made
1. One event type (`state_changed`) for every change of the alarm object, discriminated by `cause`; two new types only (`flood_changed`, `delivery_failed`).
2. A typed, self-describing, insert-only journal table beside the generic events table; written in the same transaction, sharing `event_id`.
3. Journal rows copy path, severity, class, setpoint, config revision so history survives renames and deletions.
4. Hold rows are only emitted for holds longer than one minute; chatter edges are coalesced.
5. Bus publication is after commit and best effort; the DB and `GET /events` are the complete record.
6. Restart gaps are explicit journal rows so reports never imply evaluation during downtime.

## Alternatives rejected and why
- **Only the generic `events` table**: loses typed columns, longer retention and query speed; reports would parse JSON.
- **One event type per transition** (`activated.v1`, `acked.v1`, ...): consumers must subscribe to many types and keep ordering across them. One type with `cause` and `seq` is simpler and the contract already has `state_changed`.
- **Updating a single row per alarm activation** (start and end times on one row, as many historians do): cannot represent shelve/suppress/hold interleavings and invites mutable history.
- **Event sourcing as the primary store** (derive `alarm_state` by replay): elegant, but restart time grows with history and a corrupted journal row would corrupt state. We keep a state table that is also journaled.
- **Journal in a separate SQLite file**: second database to back up and keep consistent with the first.

## Open questions
- New event types and the additive fields on `state_changed` (`seq`, `cause`, `actor`, `comment`, flags, `class`, `annunciation`) need the contract amendment.
- `engine_gap` writes one row per active alarm: confirm the volume is acceptable on a large system (it is one row per active alarm per restart, normally dozens).
- Time zone and DST in reports: journal is UTC; report layer converts using `Gateway.spec.timezone`.
- Hypertable unique-index constraints (partition column in unique indexes) and chunk interval settings are UNVERIFIED and should be validated against the TimescaleDB version used.
- Does the `events` retention policy delete alarm events earlier than the journal? Acceptable (journal is authoritative), but HMI links from events to journal should handle "not found".
