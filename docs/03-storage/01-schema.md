# 01 Schema

Status: draft. The DDL below is the intended migration content. It is written for PostgreSQL 17 (the bundled version, [08-deployment/02-containers.md](../08-deployment/02-containers.md)) and avoids features newer than PostgreSQL 14 outside the `timescale/` migrations (UNVERIFIED: the core set is to be proven on 14, 15, 16, 17 in CI). TimescaleDB specifics are isolated in 02.

## 1. Table ownership

One migration history, many semantic owners.

| Table | DDL here | Semantics owned by | Hot or cold | Notes |
|---|---|---|---|---|
| `history` | yes | storage (this area) | hot, hypertable | section 4 |
| `history_15m` | 02 | storage | derived | continuous aggregate |
| `tag_catalog` | yes | tag engine registry, written by config apply | cold | resolved tag definitions incl. removed tags |
| `system_tags` | yes | tag engine ([registry.md](../01-tag-engine/registry.md) section on `_system`) | cold | `path`, `tag_id` as the registry specifies |
| `derived_state` | yes | tag engine (tag-model.md section 7) | warm, 1 s upserts | `counter_total` state |
| `memory_values` | yes | tag engine ([pipeline.md](../01-tag-engine/pipeline.md) section 9) | cold | operator-written values |
| `events` | yes | messages.md section 4 | warm, hypertable | every CloudEvent |
| `alarm_state` | yes | alarm engine | warm | one row per alarm |
| `alarm_journal` | yes | alarm engine | warm, hypertable | transitions |
| `notification_outbox` | no | alarm engine (06-notifications) | warm | DDL stays in the alarm area, but it is created by this area's migration runner |
| `config_revisions`, `config_objects`, `config_head`, `config_drafts` | [05](05-config-storage.md) | tag-model.md section 9, api.md | cold | |
| `audit_log` | [07-security/08](../07-security/08-audit-logging.md) | security | cold | not redefined here, see [06](06-events-alarms-audit.md) |
| `forward_cursors` | [04](04-store-and-forward.md) section 8 | storage / cloud sync | cold | per-destination forward position |
| `discovered_devices` | yes | messages.md section 7 | cold | |
| `users`, `api_keys`, `sessions` | no | security (02-authn-authz) | cold | |
| `quality_codes` | yes | tag-model.md section 4 | static | reference data for SQL users |

Rule: no table other than `history` stores per-sample data. Everything else is low volume.

## 2. Roles and schemas

```sql
-- migrations connect as sr_owner; skyrangerd connects as sr_app.
CREATE ROLE sr_owner       NOLOGIN;
CREATE ROLE sr_app         LOGIN;   -- password from secret db/skyranger (deployment doc)
CREATE ROLE sr_audit_writer NOLOGIN; -- per 07-security/08 section 3; sr_app SET ROLE-switches or a second pool uses it
CREATE ROLE sr_readonly    NOLOGIN; -- granted to customer BI users; see section 8

CREATE SCHEMA IF NOT EXISTS pub AUTHORIZATION sr_owner;   -- stable customer-facing views

-- sr_app: DML on everything except audit_log, no DDL.
-- Granted per table in each migration, never ALL TABLES IN SCHEMA, so a new table is private by default.
```

Statement timeouts are set per session by the pool, not per role: writer pool none (it is bounded by batch size), query pool `statement_timeout = 30s`, export pool unlimited with a row cap in Go.

## 3. Migrations layout

```
migrations/
  core/        0001_roles.sql 0002_catalog.sql 0003_history_table.sql 0004_events.sql ...   plain SQL, any PostgreSQL
  timescale/   0001_extension.sql 0002_hypertables.sql 0003_compression.sql 0004_caggs.sql 0005_policies.sql
  plainpg/     0001_partitions.sql 0002_rollup_table.sql                                    alternative to timescale/ (02 section 8)
```

- Embedded in the binary (`embed.FS`), applied by `skyrangerd migrate` (also run by `skyrangerctl` before start). Policy (expand/contract, one-way upgrades, pre-migration backup) is owned by [08-deployment](../08-deployment/README.md) file 07.
- Which set applies on top of `core/` is decided by the backend (`timescale` or `plainpg`) and recorded in `schema_migrations(version, backend, applied_at, checksum)`. Switching backend on an existing database is not supported; use export and import (07).
- TimescaleDB steps that cannot run in a transaction (creating a continuous aggregate with data, `refresh_continuous_aggregate`) are separate non-transactional migration files, flagged by a `-- +no-transaction` first line. Creating the aggregate `WITH NO DATA` is expected to be transaction-safe (UNVERIFIED).
- Migration tool: a small in-house runner over `embed.FS` (about 150 lines: lock with `pg_advisory_lock`, one transaction per file, checksum check). `goose` or `golang-migrate` are acceptable replacements; the file layout above works with either. The choice is not worth a decision entry until it hurts.

## 4. History

```sql
-- core/0003_history_table.sql
CREATE TABLE history (
  tag_id        uuid             NOT NULL,
  ts            timestamptz      NOT NULL,   -- effective timestamp (tag-model section 5), UTC, ms values
  gw_offset_ms  bigint           NOT NULL DEFAULT 0,  -- ts_gateway - ts, milliseconds
  src_offset_ms bigint,                      -- ts_source  - ts, milliseconds; NULL = no ts_source
  v_f           double precision,            -- datatype float64
  v_i           bigint,                      -- int64, bool (0/1), uint64 <= 9223372036854775807
  quality       smallint         NOT NULL,   -- tag-model section 4 code
  flags         smallint         NOT NULL DEFAULT 0,  -- bit0 keepalive (max_interval re-emit), bit1 out_of_order (history-only sample)
  v_s           text,                        -- string; datetime as RFC 3339 UTC ms; uint64 > 2^63-1 as decimal
  CONSTRAINT history_pk        PRIMARY KEY (tag_id, ts),
  CONSTRAINT history_one_value CHECK (num_nonnulls(v_f, v_i, v_s) <= 1),
  CONSTRAINT history_finite    CHECK (v_f IS NULL OR (v_f > '-infinity'::float8 AND v_f < 'infinity'::float8)),
  CONSTRAINT history_quality   CHECK (quality BETWEEN 0 AND 191)   -- codes 192+ are reserved (tag-model section 4)
);
```

Column order is deliberate: 8-byte columns first, the `smallint` pair together, the variable-length column last, to minimize alignment padding.

### Encoding rules (normative, implemented in one Go function pair `encodeValue`/`decodeValue`)

| `datatype` | Stored in | Rule |
|---|---|---|
| `float64` | `v_f` | NaN and infinity never reach storage (tag-model section 3); the CHECK rejects them as a backstop |
| `int64` | `v_i` | exact |
| `bool` | `v_i` | `0` or `1` |
| `uint64` | `v_i` if value <= 2^63-1, else `v_s` (decimal text) | answers the tag-model open question on `uint64`. A raw device counter reaches 2^63 only in theory (at one billion counts per second it takes 292 years); the escape hatch exists so the encoder is total, not because we expect it. Values in `v_s` are excluded from rollups and sort as text |
| `string` | `v_s` | UTF-8, max 4096 bytes (tag-model) |
| `datetime` | `v_s` | RFC 3339 UTC with `Z` and millisecond precision |
| null (bad quality, no last value) | all three NULL | `quality` carries the reason |

A row is self-describing by which column is non-null, except `bool` vs `int64` (both `v_i`): decode uses `tag_catalog.datatype`. Hence the open question in the README: a `tag_id` must not change datatype.

Why offsets instead of `ts_source`/`ts_gateway` columns: the contract requires all three timestamps to be retrievable (api.md `?detail=true`), but the effective `ts` is usually the source or the gateway timestamp, so the other is within milliseconds of it. Two extra `timestamptz` columns of unique-millisecond values compress poorly; small signed offsets compress to about a byte each (compression gain UNVERIFIED, measure in milestone 1). The public view (section 8) reconstructs the canonical columns.

Why `flags`: `keepalive` lets a report count real changes (pump starts) separately from `max_interval` re-emit rows, and lets a gap analysis ignore them. It costs no extra space (it shares the 4-byte slot with `quality`). It needs the tag engine to pass the flag (README cross-area requests).

No foreign key to `tag_catalog`: a per-row FK check is wasted work on the hot path, and orphan rows (a removed tag) are legitimate history. `skyrangerctl doctor` reports `tag_id`s present in `history` and absent from `tag_catalog` (should be zero).

Hypertable conversion, compression and the rollup are in 02.

## 5. Catalog

```sql
-- core/0002_catalog.sql
CREATE TABLE tag_catalog (
  tag_id           uuid        PRIMARY KEY,
  path             text        COLLATE "C" NOT NULL,   -- byte-exact comparison, tag-model section 1
  kind             text        NOT NULL CHECK (kind IN ('source','derived','memory','system')),
  datatype         text        NOT NULL CHECK (datatype IN ('bool','int64','uint64','float64','string','datetime')),
  unit             text,                                -- UCUM
  display_name     text,
  history_enabled  boolean     NOT NULL,
  history          jsonb       NOT NULL,                -- resolved {mode, interval, max_interval, deadband}
  first_revision   bigint      NOT NULL,
  last_revision    bigint      NOT NULL,                -- last revision in which the tag existed
  removed_revision bigint,                              -- NULL = live
  removed_at       timestamptz,
  updated_at       timestamptz NOT NULL DEFAULT now()
);
-- Paths are unique among LIVE tags only: a path may be reused by a new tag_id after a delete or rename.
CREATE UNIQUE INDEX tag_catalog_live_path ON tag_catalog (path) WHERE removed_revision IS NULL;
CREATE INDEX tag_catalog_path_prefix ON tag_catalog (path COLLATE "C");   -- prefix scans, section below

CREATE TABLE system_tags (
  path    text COLLATE "C" PRIMARY KEY,
  tag_id  uuid NOT NULL UNIQUE,
  created_at timestamptz NOT NULL DEFAULT now()
);
```

Segment-wise prefix match (README.md of contracts: prefix `a/b` matches `a/b` and `a/b/...`, never `a/bc`) as an indexable predicate, with no `LIKE` (underscore is a `LIKE` wildcard and is legal in paths):

```sql
-- $1 = prefix without trailing slash. '/' is 0x2F and '0' is 0x30, so [p || '/', p || '0') is exactly "under p/".
WHERE path = $1 OR (path >= $1 || '/' AND path < $1 || '0')      -- column is COLLATE "C"
```

The catalog is written in the same transaction as the config revision (05). It is not consulted on the write path (the engine knows `tag_id`s); it serves path-to-id resolution for tags that are no longer in the registry, the public views, and `doctor`.

## 6. Engine state tables

```sql
CREATE TABLE derived_state (
  tag_id          uuid PRIMARY KEY,
  fn              text        NOT NULL,         -- 'counter_total', ...
  schema_version  integer     NOT NULL DEFAULT 1,
  state           jsonb       NOT NULL,         -- counter_total: {"last_raw":"60","last_epoch":"7","total":1234627.0}
  updated_at      timestamptz NOT NULL,
  orphaned_since  timestamptz                   -- set when the tag leaves the registry; purge after default_retention
) WITH (fillfactor = 70, autovacuum_vacuum_scale_factor = 0.0, autovacuum_vacuum_threshold = 200);

CREATE TABLE memory_values (
  tag_id     uuid PRIMARY KEY,
  value      jsonb       NOT NULL,              -- wire encoding of the tag datatype (int64/uint64 as strings)
  ts         timestamptz NOT NULL,
  written_by text        NOT NULL DEFAULT 'system:skyrangerd'
) WITH (fillfactor = 70);
```

- `derived_state` follows tag-model.md section 7 exactly: `last_raw` (uint64), `last_epoch` (Variant), `total` (float64), checkpointed at most every `1s` and at shutdown. The checkpoint writer sends one multi-row `INSERT ... ON CONFLICT (tag_id) DO UPDATE` per second containing only dirty tags, in one transaction. `jsonb` preserves `float64` exactly because Go emits the shortest round-trip decimal and `jsonb` stores numerics as exact decimals (UNVERIFIED for extreme exponents; unit-test with random doubles).
- `fillfactor = 70` and a zero-scale-factor autovacuum threshold exist because these rows are updated in place every second; without them the table bloats and every upsert touches more pages. Heap-only updates require the updated columns not to be indexed; only the primary key is.
- `orphaned_since` answers the registry open question ("purge orphaned derived state with history retention"): a daily maintenance job deletes rows with `orphaned_since < now() - default_retention`. A restore of an old revision that re-adds the `tag_id` clears the mark and keeps the state, as the registry requires.

## 7. Events, alarms, discovery

```sql
-- Hypertable (02). Every CloudEvent from messages.md section 4.
CREATE TABLE events (
  id       uuid        NOT NULL,                 -- CloudEvents id, UUIDv7
  time     timestamptz NOT NULL,
  type     text        COLLATE "C" NOT NULL,     -- 'skyranger.alarm.state_changed.v1'; C collation so prefix ranges work
  source   text        NOT NULL,                 -- '/gateways/gw01/alarm-engine'
  subject  text        COLLATE "C",              -- tag path or resource id
  tag_id   uuid,                                 -- extracted from data when present, for tag-scoped queries
  data     jsonb       NOT NULL,
  PRIMARY KEY (time, id)                         -- idempotent replay: ON CONFLICT (time, id) DO NOTHING
);
CREATE INDEX events_type_time    ON events (type, time DESC);
CREATE INDEX events_subject_time ON events (subject, time DESC);        -- C collation: prefix range scans
CREATE INDEX events_tag_time     ON events (tag_id, time DESC) WHERE tag_id IS NOT NULL;

-- Current alarm state, one row per alarm. Mirrors messages.md section 5.
CREATE TABLE alarm_state (
  alarm_id       uuid PRIMARY KEY,
  tag_id         uuid        NOT NULL,
  name           text        NOT NULL,
  severity       text        NOT NULL CHECK (severity IN ('low','medium','high','critical')),
  state          text        NOT NULL CHECK (state IN ('normal','active_unacked','active_acked','cleared_unacked')),
  shelved_until  timestamptz,
  enabled        boolean     NOT NULL DEFAULT true,
  message        text        NOT NULL,
  value          jsonb,                          -- wire encoding
  quality        smallint,
  active_at      timestamptz,
  cleared_at     timestamptz,
  acked_at       timestamptz,
  acked_by       text,
  updated_at     timestamptz NOT NULL,
  last_seq       bigint      NOT NULL,           -- last applied journal seq: makes replay idempotent
  UNIQUE (tag_id, name)
) WITH (fillfactor = 80);

-- Transition journal. Hypertable (02). Retention: Gateway.spec.alarms.journal_retention (alarm area).
CREATE TABLE alarm_journal (
  ts          timestamptz NOT NULL,
  alarm_id    uuid        NOT NULL,
  seq         bigint      NOT NULL,              -- per-alarm, gapless, assigned by the alarm engine
  tag_id      uuid        NOT NULL,
  path        text        COLLATE "C" NOT NULL,  -- path at the time (paths are mutable)
  alarm_name  text        NOT NULL,
  severity    text        NOT NULL,
  transition  text        NOT NULL,              -- activate | clear | ack | shelve | unshelve | disable | enable | ...
  from_state  text        NOT NULL,
  to_state    text        NOT NULL,
  value       jsonb,
  quality     smallint,
  actor       text,                              -- 'user:alice' or 'system:alarm-engine'
  comment     text,
  event_id    uuid,                              -- the CloudEvent written in the same transaction
  PRIMARY KEY (alarm_id, seq, ts)                -- hypertable unique key must include ts
);
CREATE INDEX alarm_journal_ts      ON alarm_journal (ts DESC);
CREATE INDEX alarm_journal_path_ts ON alarm_journal (path, ts DESC);

CREATE TABLE discovered_devices (
  discovery_id        uuid PRIMARY KEY,
  connection          text        NOT NULL,
  device_key          text        NOT NULL,
  status              text        NOT NULL CHECK (status IN ('pending','approved','ignored')),
  profile             text,
  suggested_template  text,
  first_seen          timestamptz NOT NULL,
  last_seen           timestamptz NOT NULL,
  online              boolean     NOT NULL,
  attributes          jsonb       NOT NULL DEFAULT '{}',
  points              jsonb       NOT NULL DEFAULT '[]',
  linked_instance     text,
  UNIQUE (connection, device_key)
);
```

Notes:
- `events` and `alarm_journal` overlap on alarm transitions by design (messages.md requires every event in `events`; the alarm area keeps a narrower, longer-retained, query-optimized journal). The journal row carries `event_id` so the two can be joined. Retention differs (events follow history retention per 07-security/08; the journal follows `journal_retention`), which is the reason for two tables.
- `alarm_journal` primary key `(alarm_id, seq, ts)` includes `ts` only because TimescaleDB requires the partitioning column in every unique index. Idempotent replay uses `ON CONFLICT (alarm_id, seq, ts) DO NOTHING`, and `ts` is the transition time stored in the spool record, so a replay carries the identical key.
- `discovered_devices` is small and updated on every heartbeat that changes `last_seen`; the discovery service must rate-limit `last_seen` writes to once per `60s` per device (otherwise the 60 s ESP32 heartbeat becomes a steady stream of dead tuples on a table that is not tuned for it).

## 8. Reference data and public views

```sql
CREATE TABLE quality_codes (
  code   smallint PRIMARY KEY,
  name   text     NOT NULL UNIQUE,
  class  text     NOT NULL CHECK (class IN ('good','uncertain','bad'))
);
INSERT INTO quality_codes (code, name, class) VALUES
  (0,'good','good'), (1,'good_manual','good'),
  (64,'uncertain','uncertain'), (65,'uncertain_last_known','uncertain'),
  (66,'uncertain_eng_range_exceeded','uncertain'), (67,'uncertain_initial_value','uncertain'),
  (128,'bad','bad'), (129,'bad_not_connected','bad'), (130,'bad_comm_failure','bad'),
  (131,'bad_device_failure','bad'), (132,'bad_config_error','bad'), (133,'bad_out_of_service','bad'),
  (134,'bad_waiting_for_initial_data','bad'), (135,'bad_type_mismatch','bad');
```

This table is generated from the Go constants in a unit test (the test fails if the table and `pkg/tag` disagree), so the contract stays the single source.

Customers who want direct SQL (Grafana, Power BI, an exit migration) get **views, not tables**, so the physical schema can change without breaking them:

```sql
CREATE VIEW pub.history AS
SELECT h.tag_id,
       c.path,
       h.ts,
       h.ts + make_interval(secs => h.gw_offset_ms / 1000.0)              AS ts_gateway,
       h.ts + make_interval(secs => h.src_offset_ms / 1000.0)             AS ts_source,   -- NULL if none
       h.v_f                                                              AS value_float,
       h.v_i                                                              AS value_int,
       h.v_s                                                              AS value_text,
       h.quality                                                          AS quality_code,
       q.name                                                             AS quality,
       (h.flags & 1) = 1                                                  AS keepalive
FROM history h
JOIN tag_catalog c ON c.tag_id = h.tag_id
LEFT JOIN quality_codes q ON q.code = h.quality;

CREATE VIEW pub.tags AS
SELECT tag_id, path, kind, datatype, unit, display_name, history_enabled,
       (removed_revision IS NULL) AS live
FROM tag_catalog;

CREATE VIEW pub.events AS SELECT id, time, type, source, subject, tag_id, data FROM events;
CREATE VIEW pub.alarm_journal AS SELECT * FROM alarm_journal;

GRANT USAGE ON SCHEMA pub TO sr_readonly;
GRANT SELECT ON ALL TABLES IN SCHEMA pub TO sr_readonly;
```

`pub.*` is a stable interface: additive changes only, versioned by a comment (`COMMENT ON SCHEMA pub IS 'skyranger pub v1'`). A customer's Grafana dashboards survive upgrades. The `sr_readonly` role has no access to `audit_log`, `users`, secrets or config.

## 9. PostgreSQL server settings

The compose file ([02-containers.md](../08-deployment/02-containers.md)) already sets `shared_preload_libraries=timescaledb`, `synchronous_commit=on`, `full_page_writes=on`, `shared_buffers=512MB`, `max_wal_size=1GB`. This area adds, to be merged into that file by the deployment owner:

| Setting | Value | Why |
|---|---|---|
| `initdb --data-checksums` (`POSTGRES_INITDB_ARGS`) | on | detects silent page corruption on a cheap SSD. Cost is small; it can only be set at init (offline conversion exists with `pg_checksums`) |
| `fsync` | on (default; never off) | |
| `wal_compression` | `on` | less WAL volume, modest CPU. Good on slow flash |
| `random_page_cost` | `1.1` | SSD |
| `max_connections` | `50` | pools total about 20 (section 10) |
| `log_checkpoints`, `log_lock_waits`, `log_min_duration_statement=1s` | on | support bundle material |
| `idle_in_transaction_session_timeout` | `60s` | a stuck API handler must not pin the vacuum horizon |
| `max_slot_wal_keep_size` | `2GB` | T3 standby ([01-topologies.md](../08-deployment/01-topologies.md)): a dead standby with a replication slot would otherwise grow `pg_wal` until the disk is full. PG13+ parameter (UNVERIFIED exact semantics on the pinned version) |
| `timescaledb.max_background_workers` | `4` | compression, retention and refresh jobs; a small box should not run 16 (default is UNVERIFIED) |
| `max_worker_processes` | `16` | headroom for the above plus parallel query |
| `timescaledb.telemetry_level` | `off` | an edge gateway with no internet must not try to phone home, and customers should not have to discover that it does. Setting name UNVERIFIED |
| `autovacuum_naptime` | `30s` | the hot small tables (section 6) |

`synchronous_commit` stays `on` for every session, including the history writer. Asynchronous commit would shave fsyncs but would let the writer believe a batch is durable (and delete it from the spool) when a crash could still discard it. The saving is irrelevant at our row rates (09).

## 10. Connection pools

| Pool | Size | Used by | Settings |
|---|---|---|---|
| `writer` | 2 | historian writer, spool replay | no statement timeout |
| `state` | 2 | derived_state/memory_values checkpoint, config, alarms, audit | `statement_timeout=10s` |
| `query` | 4 | REST `/history`, `/events`, `/alarms/history` | `statement_timeout=30s`, `default_transaction_read_only=on` |
| `export` | 1 | CSV export, `skyrangerctl export` | read only, no timeout, row cap |
| `admin` | 1 | maintenance (policies, prune, verify) | |

Total 10 of `max_connections=50`; the rest is headroom for `psql` on a support call and the backup job.

## Decisions made
1. All DDL is owned here; tables keep the contract names; customer access is through `pub.*` views.
2. `history` has three nullable value columns plus `quality`, `flags`, and offsets for the secondary timestamps. `uint64` is stored in `bigint` up to 2^63-1 with a text escape hatch.
3. No foreign keys from `history`; orphan detection is a `doctor` check.
4. Paths are `COLLATE "C"`; prefix queries use a half-open byte range, never `LIKE`.
5. `synchronous_commit=on`, `fsync=on`, data checksums on, always.
6. Hot small tables (`derived_state`, `memory_values`, `alarm_state`) get low `fillfactor` and aggressive per-table autovacuum.
7. Orphaned `derived_state` is purged after `default_retention`.

## Alternatives rejected and why
- **`numeric(20,0)` for `uint64`**: exact but 3x the size, no efficient compression, slow aggregation, for a range nobody reaches.
- **Reinterpreting `uint64` as two's-complement `bigint`**: compact, but ordering and aggregates become silently wrong above 2^63. Silent wrongness is the thing to avoid.
- **One table per datatype** (`history_f`, `history_i`, `history_s`): fewer NULLs and slightly smaller rows, but every cross-tag query becomes a `UNION` and the planner/rollup code doubles. The NULL overhead is a few bytes before compression and about zero after.
- **Storing `ts_source` and `ts_gateway` as full timestamps**: simpler SQL, worse compression.
- **JSONB for the value** (`{"v":1.5}`): flexible, 2x to 4x larger and unaggregatable.
- **Surrogate `bigint` tag key** (smaller index): UUIDv7 is the contract key everywhere; a mapping table adds a lookup to every write and a failure mode on restore. 16 bytes per row compress away after the first week.
- **Foreign keys with `ON DELETE`** from `history` to `tag_catalog`: would make deleting a tag either expensive or impossible, and history must outlive tags.
- **Giving customers direct access to base tables**: couples their dashboards to our schema.

## Open questions
- Is the `flags` column worth a cross-area change to `HistorySink`? If the tag engine cannot carry it, drop the column in the first migration (cheap now, a rewrite later).
- `pub.history` joins `tag_catalog` on every row. For `sr_readonly` users doing large scans, a materialized path column or a per-query `tag_id IN (SELECT ...)` rewrite may be needed. Measure with Grafana before optimizing.
- Does PostgreSQL 14 support everything in `core/` (for customers on an old server in topology T2)? Needs the CI matrix. The sizing of `max_connections` and pool counts assumes one gateway per database.
- `last_seen` rate limit (60 s) belongs to the discovery area; confirm they accept it.
- `timescaledb.telemetry_level` name and effect UNVERIFIED.
