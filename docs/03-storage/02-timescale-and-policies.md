# 02 TimescaleDB, Chunking, Compression, Rollups, Retention

Status: draft. Every TimescaleDB function name and option below is written from memory of the 2.x documentation. The pinned image tag is not yet chosen ([02-containers.md](../08-deployment/02-containers.md) marks it UNVERIFIED), so **all version-dependent behavior is marked UNVERIFIED and must be checked against the pinned release before milestone 1.** The CI conformance suite (11, section 9) contains a test for every UNVERIFIED behavior that this design depends on; a failing test means the design changes, not the test.

## 1. What TimescaleDB is used for, and what it is not

Used for exactly four things:
1. Time partitioning of `history`, `events`, `alarm_journal` (hypertables), so retention is `DROP` of a file, not `DELETE` of millions of rows.
2. Native compression of old `history` chunks.
3. One continuous aggregate, `history_15m`.
4. Background jobs (compression, retention, aggregate refresh) that run inside PostgreSQL even when `skyrangerd` is down.

Not used: space partitioning, distributed hypertables, the Toolkit extension (`timescaledb_toolkit`, time-weighted averages and LTTB: believed not to be in the stock image, UNVERIFIED), hierarchical continuous aggregates, real-time aggregation (we union raw ourselves, section 5), or any proprietary column type. (`time_bucket`, `first` and `last` are TimescaleDB functions that we do use; their availability in the stock image is part of the UNVERIFIED list.) Everything stored is plain `uuid`, `timestamptz`, `bigint`, `double precision`, `smallint`, `text`, `jsonb`, so a `COPY` to CSV and a load into stock PostgreSQL always works (the exit plan).

**Licence note (UNVERIFIED, must be checked before we promise anything to a customer).** To my recollection, compression, continuous aggregates and the job scheduler are under the Timescale License (source-available, free to self-host and embed, but restricting hosting TimescaleDB as a managed service to third parties), while the core hypertable code is Apache 2.0, and the stock `timescale/timescaledb` image ships the full community build. Consequences if that is right: shipping it inside the gateway is fine; a customer's IT department running a stripped Apache-only build would lack compression and rollups; and a customer who forbids source-available licences needs the plain PostgreSQL backend (section 8). The integrator's contract template should state which licence applies. Verify with the actual licence text.

## 2. Hypertables

```sql
-- timescale/0002_hypertables.sql   (chunk interval is chosen at init from the site profile, section 3)
SELECT create_hypertable('history',       by_range('ts',   INTERVAL '1 day'), create_default_indexes => false);
SELECT create_hypertable('events',        by_range('time', INTERVAL '7 days'));
SELECT create_hypertable('alarm_journal', by_range('ts',   INTERVAL '30 days'));
```

UNVERIFIED: the `by_range` form (believed to arrive in 2.13; the older `create_hypertable('history','ts', chunk_time_interval => ...)` is the fallback and is believed still accepted), and that `create_default_indexes` is accepted in the new form. The PostgreSQL 17 requirement implies a recent TimescaleDB (support for PG17 is believed to start at 2.17, UNVERIFIED), so `by_range` should be available.

`create_default_indexes => false` matters: the default creates a `(ts DESC)` index on every chunk. History is always queried by `tag_id` and a time range, which the primary key `(tag_id, ts)` serves. The extra index costs write time and 30% more disk for no query we run. (`events` and `alarm_journal` keep it: they are queried by time alone.)

Primary key constraints on a hypertable must include the partitioning column; ours do: `(tag_id, ts)`, `(time, id)`, `(alarm_id, seq, ts)`.

## 3. Chunk interval

A chunk is a table plus its indexes. Too-large chunks do not fit in memory while being written (index pages thrash on a small box). Too-small chunks multiply planning cost and metadata, and defeat compression (compression works per chunk and per `tag_id` segment). Rule of thumb from the TimescaleDB guidance, from memory: size chunks so the **active** chunk with indexes fits in about 25% of RAM (UNVERIFIED).

Using the uncompressed row estimate of 140 B/row from 09:

| Profile | Rows/day (all tags) | Uncompressed per day | Chunk interval | Uncompressed per chunk | Example |
|---|---|---|---|---|---|
| `meter` | < 1 M | < 140 MB | `7 days` | < 1 GB | the water meter milestone (21 k to 183 k rows/day) |
| `small` | 1 M to 20 M | 140 MB to 2.8 GB | `1 day` | 0.14 to 2.8 GB | 10,000 tags quiet or typical (2.3 M, 11.5 M) |
| `medium` | 20 M to 100 M | 2.8 to 14 GB | `6 hours` | 0.7 to 3.5 GB | 10,000 tags busy (69 M), 100,000 quiet (23 M) |
| `large` | > 100 M | > 14 GB | `1 hour` | 0.6 GB and up | 100,000 typical (115 M): a server, not an edge box |

- `site.yaml` carries `storage.profile: meter|small|medium|large` (default `meter` until a site grows). The profile selects chunk interval, compression delay and queue sizes. `skyrangerctl doctor` recommends a profile from measured rows/day.
- The interval can be changed later with `set_chunk_time_interval('history', ...)`; it applies to newly created chunks only (believed, UNVERIFIED), so changing profile needs no migration of existing data.
- With 3 compression-delay chunks resident in uncompressed form, the **resident uncompressed window** is the dominant disk and RAM cost: `compress_after x rows/day x 140 B` (09 shows 4.8 GB for 10,000 typical).

## 4. Compression

```sql
-- timescale/0003_compression.sql
ALTER TABLE history SET (
  timescaledb.compress,
  timescaledb.compress_segmentby = 'tag_id',
  timescaledb.compress_orderby   = 'ts DESC'
);
SELECT add_compression_policy('history', compress_after => INTERVAL '3 days');   -- profile: max(2 x chunk, 3 days)
```

Why `segmentby = tag_id`: every query we run filters on `tag_id`, and values of one tag compress well together (similar magnitudes, delta-encoded timestamps). Compression ratio depends on rows per `(tag_id, chunk)`: TimescaleDB packs up to 1000 rows into one compressed batch, so a tag with fewer than about 100 rows in a chunk compresses badly (UNVERIFIED threshold). The profile table therefore prefers longer chunks for sparse data (the meter profile uses 7 days).

Expected ratio: the deployment doc quotes 10x to 20x (UNVERIFIED). This area plans with **16 bytes per compressed row against 140 uncompressed (about 9x)**, deliberately conservative because on-change data has irregular timestamps and noisy floats. Milestone 1 must measure the real figure with `hypertable_compression_stats('history')` on the simulator and replace the planning constant in 09.

`compress_after` is also the **late-data tolerance**: rows arriving for a time older than that land in a compressed chunk. Recent TimescaleDB versions allow `INSERT` into compressed chunks (believed 2.11 and later, with limits around unique constraints and `ON CONFLICT`; UNVERIFIED), but it is slower and decompresses batches. Our idempotent insert uses `ON CONFLICT DO NOTHING` against the unique primary key, so **whether `INSERT ... ON CONFLICT DO NOTHING` works against compressed chunks is a gating UNVERIFIED item.** Fallbacks if it does not:
1. Spool replay for data older than `compress_after` first calls `decompress_chunk` for the affected chunks, inserts, and lets the policy recompress. Cost: transient disk equal to the uncompressed chunk (bounded by profile, at most a few GB).
2. Increase `compress_after` to cover the maximum outage the spool can hold (09 section 6 gives hours per profile).

Replay is expected rarely and mostly within hours, so the slow path matters only after a multi-day DB outage.

Compression of `events` and `alarm_journal`: not enabled in v1 (low volume; 09). Revisit if a site shows more than a few million rows.

## 5. The rollup: `history_15m`

Purpose: long-range trends and daily/monthly reports without scanning raw rows, and **downsampled history that outlives raw retention.**

```sql
-- timescale/0004_caggs.sql   (non-transactional migration; create WITH NO DATA, then policy and initial refresh separately)
CREATE MATERIALIZED VIEW history_15m
WITH (timescaledb.continuous, timescaledb.materialized_only = true) AS
SELECT
  tag_id,
  time_bucket(INTERVAL '15 minutes', ts)                                              AS bucket,
  count(*)                                                                            AS n_total,
  count(CASE WHEN quality < 128 THEN 1 END)                                           AS n_usable,
  count(CASE WHEN (flags & 1) = 0 THEN 1 END)                                         AS n_changes,   -- non-keepalive rows
  max(CASE WHEN quality < 128 THEN quality END)                                       AS worst_q,     -- NULL if nothing usable
  min(CASE WHEN quality < 128 THEN COALESCE(v_f, v_i::double precision) END)          AS v_min,
  max(CASE WHEN quality < 128 THEN COALESCE(v_f, v_i::double precision) END)          AS v_max,
  sum(CASE WHEN quality < 128 THEN COALESCE(v_f, v_i::double precision) END)          AS v_sum,
  first(CASE WHEN quality < 128 THEN COALESCE(v_f, v_i::double precision) END, ts)    AS v_first,
  last (CASE WHEN quality < 128 THEN COALESCE(v_f, v_i::double precision) END, ts)    AS v_last,
  min(ts)                                                                             AS ts_first,
  max(ts)                                                                             AS ts_last
FROM history
WHERE v_s IS NULL                      -- numeric and boolean only; strings, datetimes, huge uint64 are not rolled up
GROUP BY tag_id, time_bucket(INTERVAL '15 minutes', ts)
WITH NO DATA;

SELECT add_continuous_aggregate_policy('history_15m',
  start_offset      => INTERVAL '3 days',
  end_offset        => INTERVAL '15 minutes',
  schedule_interval => INTERVAL '5 minutes');
```

How the columns implement the contract (api.md section "History"): "aggregates use only `good` and `uncertain` samples" is `quality < 128` (class `bad` starts at 128). "A bucket's quality is the worst included quality" is `worst_q`, where larger code means a worse class (good 0..1, uncertain 64..67); within a class the numeric order is not a badness order, so the query layer reports the class's generic representative only when it must pick (a `good_manual` and a `good` row in one bucket report `good_manual`, the larger code, which is the more informative and still class good). "A bucket with no usable samples has `value: null, quality: bad`" is `n_usable = 0`, `worst_q IS NULL`.

UNVERIFIED items this view depends on (each has a CI test):
1. Continuous aggregates accept `CASE` inside aggregates, `WHERE` on the source, and `first`/`last` with a `CASE ... END` argument. In particular **whether `first()`/`last()` ignore NULL arguments** (they must, for the first and last *usable* value to be right when a bucket starts with a bad row). If they do not, replace `v_first`/`v_last` with two separate columns `ts_first_usable = min(CASE ... THEN ts END)` and a join back to `history` at query time.
2. `timescaledb.materialized_only` as a view option (the default of real-time aggregation changed across versions, believed 2.13 made materialized-only the default; we set it explicitly).
3. `CREATE MATERIALIZED VIEW ... WITH NO DATA` inside a transaction block.
4. Compression of the continuous aggregate's own storage (believed available since 2.6).

### Why materialized-only and a union in Go

Real-time aggregation (the view silently unions un-materialized raw rows) is convenient but its exact semantics changed between versions (UNVERIFIED), and it hides a performance cliff: after a long spool replay the un-materialized region is large and a "cheap" query becomes a raw scan. Instead the query layer asks `SELECT max(bucket) FROM history_15m WHERE bucket > now() - INTERVAL '3 days'` (an index range read) to learn the **materialization boundary**, serves buckets below it from the rollup and everything above from raw rows aggregated on the fly (10, section 5). The raw tail is at most about 20 to 35 minutes of data, so it is cheap and explicit.

### Late data and refresh

The refresh policy covers `[now - 3 days, now - 15 minutes]` every 5 minutes. Rows that arrive for a bucket already materialized invalidate it and the next refresh recomputes it (believed behavior, UNVERIFIED). Rows older than `start_offset` (a spool replay after a multi-day outage) are **not** picked up by the policy. After any replay that touched data older than 3 days the writer calls `Admin().RefreshAggregates(ctx, from, to)`, which runs `CALL refresh_continuous_aggregate('history_15m', from, to)` outside a transaction. The same call is made after a restore.

`start_offset` must stay below raw retention: if a refresh window reached into chunks already dropped by retention it would re-materialize those buckets as empty. With 3 days against 400 days there is no overlap; the config validator rejects `default_retention < 7d` for this reason.

### Why 15 minutes, and only one level

- 15 minutes divides every UTC offset in current use, including the half-hour (India) and quarter-hour (Nepal) zones, so a **local calendar day** is always a whole number of 15-minute buckets. Daily and monthly water consumption in the plant's timezone (`Gateway.spec.timezone`) therefore rolls up exactly from `history_15m` with `date_trunc('day', bucket AT TIME ZONE tz)`. A 1-hour rollup would misalign by 30 minutes for those zones, and a UTC-day rollup is wrong for everyone.
- On-change data does not shrink when bucketed finer than its natural rate: a tag with one row per minute produces one row per 1-minute bucket. A 15-minute rollup cuts that by about 15x at the typical rate (09), where a 1-minute rollup would save nothing.
- Hourly, daily, and monthly series are queries over `history_15m` (96 rows per tag per day; a year of one tag is 35 k rows). If dashboard latency demands it, a second cagg `history_1d` or a hierarchical aggregate can be added later. It is cheap to add and impossible to remove from a customer's reports once they depend on it, so we start with one.
- A time-weighted mean cannot be built from the columns above; see 10, section 4 for the on-demand computation and the open question about `agg=twavg`.

## 6. Retention and downsampling tiers

| Data | Policy | Source of the number | Notes |
|---|---|---|---|
| Raw `history` | `add_retention_policy('history', drop_after => <default_retention>)` | `Gateway.spec.history.default_retention` (contract, default `400d`) | whole chunks only, so effective retention is up to one chunk interval longer |
| `history_15m` | `add_retention_policy('history_15m', drop_after => <aggregate_retention>)` | proposed `Gateway.spec.history.aggregate_retention`, default `3y` | placeholder; regulatory need is UNVERIFIED |
| `events` | follows raw history retention | 07-security/08 section 4 | |
| `alarm_journal` | `journal_retention` | alarm area config (`1095d` placeholder) | |
| `audit_log` | `7y` default, min `1y`, prefix pruning only | 07-security/08 | not a hypertable |
| `derived_state` orphans | `default_retention` after `orphaned_since` | 01 section 6 | daily Go job |
| config revisions | never deleted in v1 | 05 | tiny |

- **Downsampling is automatic and one-directional:** raw rows become 15-minute rollups continuously (within minutes), raw rows are dropped at `default_retention`, rollups live on. After retention the system can answer "daily consumption in March two years ago" but not "the exact flow at 14:03:11".
- **Retention is global, never per tag.** The contract has no per-tag retention field, and per-tag retention on compressed chunks would be a `DELETE` that decompresses and rewrites data, the exact cost hypertables avoid. If one customer needs 5 years for 10 tags and 90 days for the rest, the answer is `aggregate_retention`, or a second `Connection`/gateway; this is an open question for the contract, not a storage feature to improvise.
- **Retention changes** arrive with a config revision (`Gateway.spec.history.default_retention`). The apply hook calls `Admin().EnsurePolicies(ctx, Policies{...})`, which is idempotent: it removes and re-adds the policy only if the interval differs (`remove_retention_policy` then `add_retention_policy`, believed names, UNVERIFIED). Shortening retention drops data at the next job run: the apply response carries a warning stating how much raw history will be dropped (computed from chunk ranges), and the audit record notes it. Lengthening takes effect immediately for data not yet dropped.
- `drop_chunks` is irreversible. The drop job is never run by hand on a customer box without a backup less than a day old (07).

## 7. Background job hygiene on a small box

- Compression and refresh are CPU- and IO-heavy on a 4-core box. The Timescale job scheduler runs them whenever due; Running them at a lower OS priority is not possible from inside PostgreSQL, so the controls against colliding with a trend-query storm are: `timescaledb.max_background_workers=4`, `compress_after` large enough that each run compresses one chunk, and the query-layer concurrency cap (10, section 10).
- `skyrangerd` records job status from `timescaledb_information.job_stats` (believed view name, UNVERIFIED) into `_system/historian/*` tags (03, section 11) so a job that has been failing for hours raises an alarm instead of a full disk.

## 8. Plain-PostgreSQL fallback (designed, not built for milestone 1)

For a customer who cannot or will not run TimescaleDB (licence, an existing PostgreSQL service where extensions cannot be installed, or a corporate standard). Same `core/` migrations, with `plainpg/` in place of `timescale/`:

```sql
-- plainpg/0001_partitions.sql   (history only; events and alarm_journal follow the same pattern)
CREATE TABLE history (
  ... same columns and constraints as 01 section 4, but declared as:
) PARTITION BY RANGE (ts);
-- PRIMARY KEY (tag_id, ts) is allowed because ts is the partition key.
-- Partitions are created by the Go Maintainer, 14 days ahead, one per day:
--   CREATE TABLE history_20261107 PARTITION OF history FOR VALUES FROM ('2026-11-07') TO ('2026-11-08');
-- Retention: DROP TABLE history_YYYYMMDD (or DETACH then DROP) by the Maintainer.

-- plainpg/0002_rollup_table.sql
CREATE TABLE history_15m (
  tag_id uuid NOT NULL, bucket timestamptz NOT NULL,
  n_total int NOT NULL, n_usable int NOT NULL, n_changes int NOT NULL, worst_q smallint,
  v_min double precision, v_max double precision, v_sum double precision,
  v_first double precision, v_last double precision, ts_first timestamptz NOT NULL, ts_last timestamptz NOT NULL,
  PRIMARY KEY (tag_id, bucket)
) PARTITION BY RANGE (bucket);
-- Filled by the Go Maintainer every 5 minutes:
--   INSERT INTO history_15m SELECT ...same SELECT as the cagg... WHERE ts >= $from AND ts < $to
--   ON CONFLICT (tag_id, bucket) DO UPDATE SET ...;
-- over a sliding window, plus an explicit range call after replay (same Admin().RefreshAggregates).
```

What changes: no compression (TOAST does not help rows this small), so about **9x more disk** at the same retention (09 shows about 645 GB for 10,000 tags typical over 400 days versus 74 GB). The backend is therefore suitable for meters, small plants and short retention, not for 10,000 tags and a year. The query layer is identical (it only needs `time_bucket` replaced by `date_bin`, PG 14+, which the generic SQL already uses where it does not need cagg-specific functions). `Capabilities()` reports `Compression: false, NativeAggregates: false`, and `skyrangerctl doctor` projects days-to-full accordingly.

This backend is designed so a second implementation exists on paper and the interface (11) is proven against two shapes. It is implemented the first time a customer needs it.

## 9. Conformance tests for this file

Run in CI against the pinned Timescale image (and, once built, plain PostgreSQL):
- Insert-with-conflict into a compressed chunk (gating item, section 4).
- `first`/`last` NULL handling in the cagg (section 5).
- Retention drops exactly whole chunks; effective retention is at most `default_retention + chunk interval`.
- Aggregate refresh after back-dated insert, and after a manual `refresh_continuous_aggregate` call.
- `set_chunk_time_interval` leaves existing chunks alone.
- Hypertable plus compression plus cagg survive `pg_dump`/`pg_restore` using the documented Timescale restore procedure (07).
- Quality semantics: a bucket of [bad, good, uncertain] reports `n_usable=2`, `worst_q=64`; a bucket of [bad] reports `n_usable=0`.

## Decisions made
1. TimescaleDB is used for hypertables, compression, one continuous aggregate and policies, nothing else. No Toolkit, no real-time aggregation, no hierarchical aggregates.
2. Chunk interval and compression delay come from a four-step `storage.profile` (`meter`, `small`, `medium`, `large`), changeable later for new chunks.
3. `segmentby = tag_id`, `orderby = ts DESC`. Compression ratio is planned at 9x until measured.
4. One rollup, `history_15m`, materialized-only, with the union against raw done by the query layer using the measured materialization boundary.
5. Raw retention is global and chunk-granular; rollups outlive raw (default `3y`); no per-tag retention.
6. After a replay or restore older than the refresh window, the writer explicitly refreshes the affected range.
7. A plain-PostgreSQL backend is designed (partitions plus a Go-maintained rollup) and built on demand.

## Alternatives rejected and why
- **Native PostgreSQL partitioning plus `pg_partman`/`pg_cron` as the primary path**: boring and fully open, but no compression (9x disk) and more moving parts to maintain ourselves. It stays as the fallback (section 8).
- **A 1-minute rollup**: no gain on on-change data (section 5), 15x the rows of the 15-minute rollup.
- **A UTC daily rollup**: wrong local-day totals for billing.
- **Real-time aggregation views**: version-dependent semantics and a hidden raw scan after replays.
- **Time-weighted aggregates through the Toolkit extension**: not in the stock image (UNVERIFIED), another extension to version and pin, and not portable to the fallback.
- **Per-tag retention via `DELETE`**: rewrites compressed data, bloats, and no contract field asks for it.
- **Space partitioning by `tag_id` hash**: helps multi-node write throughput we do not have; costs planner overhead on every query.
- **Short chunks for everyone** (one hour): good for the 100,000-tag server, 168 chunks per week of nearly-empty tables for the water meter.

## Open questions
- Every UNVERIFIED item above. The gating ones: `ON CONFLICT DO NOTHING` on compressed chunks, `first`/`last` NULL handling, `WITH NO DATA` in a transaction, licence text.
- Real compression ratio on our row shape. Measure at milestone 1 with the simulator running at the `typical` rate for a week of compressed chunks.
- Whether `history_15m` should also be compressed in v1 (its size is 21 GB for 10,000 tags at 3 years, 09), and when to add a daily aggregate.
- `aggregate_retention` default of `3y` is a placeholder; water utilities may require more or less (UNVERIFIED).
- Should the writer, rather than a policy, own compression scheduling so that compression pauses while a spool replay is running? (Lower contention on small boxes; more code.)
