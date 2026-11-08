# 03 Storage and Historian

Status: design draft v0.1. Conforms to [../00-contracts/](../00-contracts/README.md), which is binding. Where this area needs something the contracts do not yet allow, it is proposed as an **additive extension** or flagged as a contract question under Open questions. Nothing here silently deviates from a contract. Claims about TimescaleDB or PostgreSQL features that were not checked against documentation are marked **UNVERIFIED**.

This area owns: the PostgreSQL schema (all tables the other areas reference), time-series history and its policies (chunking, compression, rollups, retention), the write path from the tag engine to the database, the disk-backed store-and-forward queue, configuration revision storage, the journals (events, alarms, audit) at the storage level, backup and restore, behavior when the disk fills or the power fails, capacity math, trending query patterns, and the Go interface that keeps the database replaceable.

The author's rule applies here more than anywhere else: **trust and reliability are the product.** A SCADA historian is only useful if the answer to "what did the meter read at 03:12 last Tuesday" is right, and if the answer to "was there a gap" is honest. Every design choice below is tested against two questions: what happens to the data when the box loses power mid-write, and can the customer take the data and leave.

## Files

| File | Covers |
|---|---|
| [01-schema.md](01-schema.md) | Table ownership matrix, roles, migrations layout, DDL for `tag_catalog`, `history`, `events`, `alarm_state`, `alarm_journal`, `derived_state`, `memory_values`, `system_tags`, `discovered_devices`, `quality_codes`, public SQL views, PostgreSQL server settings |
| [02-timescale-and-policies.md](02-timescale-and-policies.md) | Hypertables and chunk sizing, compression, the 15-minute continuous aggregate, retention and downsampling tiers, the plain-PostgreSQL fallback, licensing note |
| [03-write-path.md](03-write-path.md) | `HistorySink` to database: queue, batching, idempotent inserts, deadband and on-change vs periodic (what is upstream, what is here), writer state machine, error classes, flood guard, shutdown, metrics |
| [04-store-and-forward.md](04-store-and-forward.md) | Disk-backed spool format and recovery, replay, size caps and thinning, dead letters, the cloud forward queue, relation to the "store-and-forward is out of scope" contract line |
| [05-config-storage.md](05-config-storage.md) | `config_revisions`, content-addressed documents, atomic apply, drafts, restore, export/import fidelity, the catalog, snapshot file and git mirror |
| [06-events-alarms-audit.md](06-events-alarms-audit.md) | Event store, alarm state and journal storage, audit log placement and write discipline, retention, query shapes |
| [07-backup-restore.md](07-backup-restore.md) | What is backed up, logical vs physical, verification, restore runbook, what is lost and what self-heals, exit export |
| [08-disk-power-corruption.md](08-disk-power-corruption.md) | Disk-full ladder, ballast file, power loss on a cheap box, corruption detection and response, clock hazards, the pull-plug test |
| [09-capacity.md](09-capacity.md) | Assumptions and arithmetic for the water meter, 10,000 tags and 100,000 tags: rows, bytes, disk, WAL, spool, throughput |
| [10-queries.md](10-queries.md) | Trending query patterns as SQL: raw window, boundary sample, bucketed aggregates, rollup re-bucketing, carry-forward delta, local-day consumption, gaps, export, pagination, concurrency limits |
| [11-go-interfaces.md](11-go-interfaces.md) | Go storage abstraction: `Store` and its repositories, transactions, errors, capabilities, backend registry, conformance suite |

Read in order. 09 is the file to check before quoting hardware.

## The area in one picture

```
 tag engine (single writer)                                     REST /api/v1  (history, events, config, audit)
   │ HistorySink.Offer()   non-blocking                              │
   ▼                                                                 ▼
 ┌────────────── historian writer ──────────────┐        ┌───────────────── query layer ─────────────────┐
 │ ring (bounded) ─► batcher ─► DB insert       │        │ planner: raw | 15m rollup + raw tail          │
 │                        │  (idempotent,       │        │ per-request timeout, 4-way concurrency cap     │
 │                        │   unnest + ON       │        └───────────────┬───────────────────────────────┘
 │                        │   CONFLICT DO NOTHING)                       │
 │          DB unavailable│            ▲ replay (throttled)             │
 │                        ▼            │                                 │
 │             spool/historian/*.seg ──┘   (CRC-framed, fsync'd,         │
 │             dead/  (poison, bad clock)   torn-tail safe)              │
 └────────────────────────┬─────────────────────┘                       │
                          ▼                                              ▼
              PostgreSQL 17 (+ TimescaleDB on the bundled and recommended path)
   ┌──────────────────────────────────────────────────────────────────────────────────────────┐
   │ history (hypertable) ─► history_15m (continuous aggregate)       events (hypertable)    │
   │ alarm_journal (hypertable)   alarm_state   derived_state   memory_values   tag_catalog   │
   │ config_revisions + config_objects + config_head + config_drafts   audit_log (hash chain) │
   └──────────────────────────────────────────────────────────────────────────────────────────┘
              │ nightly backup (07)           │ pub.* SQL views (customer BI, exit)
              ▼                               ▼
     /var/lib/skyranger/backups        sr_readonly role, plain SQL, no Timescale types
```

State transitions and config apply never go through the history writer: they use ordinary PostgreSQL transactions (`WithTx`, see 11) so "persist, then announce" (alarms README) and "change and audit in one transaction" (07-security/08) both hold.

## Summary of key decisions

1. **One hypertable, three value columns, offsets for the secondary timestamps.** `history(tag_id, ts, gw_offset_ms, src_offset_ms, quality, flags, v_f, v_i, v_s)` with primary key `(tag_id, ts)`. `float64` goes in `v_f`; `int64`, `bool` and `uint64` (up to 2^63-1) in `v_i`; strings, datetimes and the rare `uint64` above 2^63-1 in `v_s`. This answers the tag-model open question on `uint64` without `numeric(20,0)`.
2. **Idempotent writes everywhere.** History rows are keyed `(tag_id, ts)` and inserted with `ON CONFLICT DO NOTHING`; events by `(time, id)`; alarm transitions by `(alarm_id, seq)`. That makes the spool at-least-once and safe to replay after any crash, restore or failover.
3. **No write-ahead log in front of a healthy database.** The disk spool is used only when PostgreSQL is unavailable or the writer falls behind. When the DB is up, a power cut can lose at most the unflushed batch (default flush `500ms`). This is a documented, bounded window, and cumulative counters recover from it.
4. **One rollup: `history_15m`.** Fifteen minutes divides every real UTC offset, so local-day and local-month totals roll up from it. Hourly and daily views are queries over `history_15m`, not more materialized tiers (until measured otherwise).
5. **Retention is global and drops whole chunks.** `Gateway.spec.history.default_retention` governs raw history; rollups keep longer (proposed `aggregate_retention`, default `3y`). No per-tag `DELETE`.
6. **Config is content-addressed.** One `config_revisions` row per revision (contract wording) listing `(kind, name, sha256)`; documents live once in `config_objects`. History of a 10,000-tag site costs kilobytes per revision, not megabytes.
7. **Disk-full is a ladder, not a cliff.** 80% warn, 90% emergency-prune raw history down to a floor (default `30d`) or stop history (customer choice), 95% shed history entirely, with a 1 GiB ballast file to give PostgreSQL room to recover. Config, audit and alarms are never shed first.
8. **Go interface split by repository** (`HistoryStore`, `ConfigStore`, `EventStore`, `AlarmStore`, `StateStore`, `AuditStore`, `CatalogStore`, `Admin`) with a backend registry and a conformance suite, so a second backend can implement history only, or everything.

## Cross-area requests (not yet agreed)

These touch other areas and are listed so their owners can accept or reject them:

| To | Request |
|---|---|
| Tag engine (`HistorySink`) | Carry `flags` (bit 0 `keepalive`, bit 1 `out_of_order`) and both timestamps per sample; `Offer` returns a result, never blocks. See 03. |
| Alarms (07, 09) | Adopt the `alarm_state`, `alarm_journal` DDL in 01 and the shared spool library in 04 instead of a private JSONL spool. |
| Deployment (02, 03, 07) | Sizing numbers in 03-packaging-and-hardware.md (50 B/row, 10x to 20x) are more optimistic than 09; reconcile. Add `POSTGRES_INITDB_ARGS=--data-checksums`, `max_slot_wal_keep_size`, ballast file creation in `skyrangerctl init`. |
| Security (08) | `audit_log` DDL stays owned there; 06 adds write-discipline details only. |
| Contracts | See Open questions below. |

## Decisions made
1. The storage area owns all DDL. Other areas own the semantics of their rows but not the table definitions, so there is one migration history.
2. Tables live in schema `public`, named exactly as the contracts name them (`events`, `audit_log`, `config_revisions`, `derived_state`, `memory_values`, `system_tags`, `discovered_devices`). Customer-facing views live in schema `pub`.
3. Backend selection, DB URL, spool paths and caps are host-level settings (env or `site.yaml`), not `Gateway` config, because the configuration itself lives in the database.
4. The bundled path is PostgreSQL 17 + TimescaleDB (decision 2). Plain PostgreSQL and other history databases are supported by design (interface and a schema sketch) and implemented only when a customer needs them.
5. History is never part of a multi-table transaction.

## Alternatives rejected and why
- **A dedicated time-series database (InfluxDB, QuestDB, VictoriaMetrics, ClickHouse) as the primary store**: better raw ingest on paper, but a second database engine to operate, back up and teach on every edge box, plus a new query language. Config, audit and alarms need transactions anyway. The interface in 11 leaves the door open for history only.
- **SQLite on the edge box**: boring and robust for one writer, but no compression, retention by `DELETE`, and a second schema dialect from the central server. Decision 2 already chose PostgreSQL.
- **Wide table (one column per tag)**: breaks at 10k tags, cannot represent config changes.
- **Writing history through MQTT**: contract decision (messages.md). Broker outage or QoS 0 loss would create gaps.

## Open questions
1. **Contract: durations with `d`.** tag-model.md uses `default_retention: 400d` and api.md uses `1d` intervals, but README.md says durations are Go duration strings, and `time.ParseDuration` has no day unit. Storage parses `Nd` (24 h) for retention and interval fields until the contract says whether to allow `d` or to write `9600h`.
2. **Contract: `delta` aggregate.** api.md defines `delta` as last minus first within the bucket. On on-change data that loses the increment between the last row of one bucket and the first row of the next, which is a systematic undercount (see 10, section 7). Proposed: `delta` carries the previous bucket's last value forward. Both are implemented; the literal one is selectable.
3. **Contract: `avg` on on-change data** is the mean of stored samples, not time-weighted. Propose an additive `agg=twavg`.
4. **Contract: multi-series `limit` and `cursor`** in `/history` (per series or total) is unspecified. Storage assumes per-series limit with a cursor map.
5. **Tag engine:** history decoding assumes a tag's `datatype` never changes for a given `tag_id`. If the registry allows an in-place datatype change, history needs either a new `tag_id` or a `datatype` epoch.
6. **Events:** `skyranger.tag.counter_discontinuity.v1` with `kind: manual_adjust` has no `total_before`, so a consumption report cannot net the jump out of a delta. Proposed additive field.
7. Whether `Gateway.spec.history` should grow `aggregate_retention` (default `3y`), `disk` policy (`prune_oldest | stop_history`, `min_raw_retention`) as additive fields, or whether these stay host-level in `site.yaml`. This doc assumes `Gateway.spec.history` for retention and `site.yaml` for disk policy.
8. TimescaleDB version and licence facts in 02 (UNVERIFIED) must be checked against the pinned image before milestone 1.
