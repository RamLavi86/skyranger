# 10 Trending Query Patterns

Status: draft. These are the query shapes behind `GET /api/v1/history`, `/history/export`, and the reports the integrator will build. They are written against the schema in 01 and the rollup in 02. SQL uses `time_bucket` where the Timescale backend is assumed and notes the portable `date_bin` equivalent; function behaviors that are version-dependent are marked UNVERIFIED and covered by the conformance suite (11, section 9). All parameters are bound parameters, never string-built.

## 1. Principles

1. **Tags are addressed by `tag_id`.** The API accepts paths; they are resolved to `tag_id` in memory from the registry (live tags) or from `tag_catalog` (removed tags, 01 section 5) before any SQL. If a path maps to several `tag_id`s over time (a path reused after deletion), the live one wins, otherwise the most recently removed one. Old paths after a rename are not aliases (contract).
2. **Time ranges are half-open `[start, end)`.** api.md does not say; this is the only definition under which adjacent windows never double-count. (Open question 1.)
3. **Every history query filters `tag_id` and `ts`.** That is what the primary key and compression `segmentby` serve. A query without a `tag_id` filter is a bug and the query layer refuses to build one.
4. **Aggregates use usable samples only**: `quality < 128` (good and uncertain), per api.md. A bucket with none is `value: null, quality: bad`.
5. **Raw data is for short ranges; rollups are for long ones.** The planner (section 5) chooses; the client does not.
6. **Cumulative totals are aggregated by carrying the value across bucket boundaries** (section 7), never by `last - first` inside a bucket alone.
7. **The query pool is isolated** (01, section 10): read-only role, 30 s statement timeout, 4 connections, concurrency cap (section 10). A trend storm must not slow the writer.

## 2. Raw window, many series

For `agg=raw`, up to 50 series, per-series `limit` (default 10,000, max 100,000), keyset pagination per series. One `LATERAL` subquery per tag keeps each series a pure index range scan (and a segmentby-filtered decompression on old chunks) and makes `LIMIT` apply per series:

```sql
SELECT t.tag_id, h.ts, h.gw_offset_ms, h.src_offset_ms, h.v_f, h.v_i, h.v_s, h.quality, h.flags
FROM unnest($1::uuid[], $2::timestamptz[]) AS t(tag_id, after_ts)         -- after_ts = '-infinity' on the first page
CROSS JOIN LATERAL (
  SELECT *
  FROM history h
  WHERE h.tag_id = t.tag_id
    AND h.ts >= $3 AND h.ts < $4                                          -- [start, end)
    AND h.ts >  t.after_ts                                                -- keyset
  ORDER BY h.ts
  LIMIT $5 + 1                                                            -- +1 to know whether a next page exists
) h
ORDER BY t.tag_id, h.ts;
```

The `+1` row is not returned; it only sets `next_cursor`. `ts_source` and `ts_gateway` are reconstructed as `ts + offset` only when `?detail=true` (01, section 4); `value` is decoded from `v_f`/`v_i`/`v_s` by the catalog datatype.

**Cursor:** `base64url(JSON{"v":1,"q":"<hash of the normalized query parameters>","after":{"<tag_id>":"<ts>"}})`. `after` lists only series that were truncated; the next request returns only those. The `q` hash rejects a cursor presented with different parameters (`400 bad_request`). Not signed: it contains positions only. (Multi-series `limit` semantics are an open question for api.md; this is the assumption.)

## 3. Boundary sample (step charts and `delta`)

On-change data draws as steps, and the first segment of a window needs the value in force at `start`. Bounded lookback so an idle tag cannot scan all chunks:

```sql
SELECT ts, v_f, v_i, v_s, quality
FROM history
WHERE tag_id = $1
  AND ts <  $2                      -- start
  AND ts >= $2 - $3::interval       -- lookback = max(2 x max_interval, 1 h), from tag_catalog.history
ORDER BY ts DESC
LIMIT 1;
```

The result is returned as a point at `ts = start` only if the HMI requests `boundary=true` (proposed additive query param), never mixed silently into data points. If no row exists within the lookback the tag had no known value at `start`: that is a gap, and it is reported as such (section 8).

## 4. Bucketed aggregates from raw rows, and time-weighted average

Used when the planner picks raw (short ranges, intervals not aligned to 15 minutes, or the live tail). `date_bin`'s origin is fixed to `2000-01-03 00:00:00+00` and the same origin is passed to `time_bucket`, so both produce identical bucket edges (the TimescaleDB default origin is believed to be the same Monday, UNVERIFIED, which is why it is passed explicitly):

```sql
SELECT
  tag_id,
  time_bucket($interval, ts, origin => TIMESTAMPTZ '2000-01-03 00:00:00+00')                    AS bucket,   -- portable: date_bin($interval, ts, TIMESTAMPTZ '2000-01-03 00:00:00+00')
  count(*)                                                                                      AS n_total,
  count(*) FILTER (WHERE quality < 128)                                                         AS n_usable,
  max(quality) FILTER (WHERE quality < 128)                                                     AS worst_q,
  min(COALESCE(v_f, v_i::float8)) FILTER (WHERE quality < 128)                                  AS v_min,
  max(COALESCE(v_f, v_i::float8)) FILTER (WHERE quality < 128)                                  AS v_max,
  sum(COALESCE(v_f, v_i::float8)) FILTER (WHERE quality < 128)                                  AS v_sum,
  first(COALESCE(v_f, v_i::float8), ts) FILTER (WHERE quality < 128)                            AS v_first,
  last (COALESCE(v_f, v_i::float8), ts) FILTER (WHERE quality < 128)                            AS v_last
FROM history
WHERE tag_id = ANY($1) AND ts >= $2 AND ts < $3 AND v_s IS NULL
GROUP BY tag_id, bucket
ORDER BY tag_id, bucket;
```

Mapping to `agg`: `avg = v_sum / n_usable` (arithmetic mean of stored samples), `min`, `max`, `first = v_first`, `last = v_last`, `count = n_usable`, `delta` as section 7. Output quality: `n_usable = 0` gives `value null, quality bad`; otherwise the quality name of `worst_q` (01, `quality_codes`). A generic (non-Timescale) path replaces `first`/`last` with `(array_agg(v ORDER BY ts) FILTER (...))[1]` and `time_bucket` with `date_bin`; the results are identical.

**`avg` is the mean of stored samples, not time-weighted.** For on-change data a value that persisted for an hour and one that persisted for a minute count once each, which misleads (a pump that ran at 50 Hz all day and spiked to 80 Hz for one minute averages high). The contract's `avg` is implemented as written and documented as such. A time-weighted mean exists as a separate, on-demand computation over raw rows (proposed `agg=twavg`; open question), because it cannot be built from the rollup's columns:

```sql
WITH pts AS (                                  -- rows in [start - lookback, end), all qualities, ordered
  SELECT tag_id, ts, quality, COALESCE(v_f, v_i::float8) AS v
  FROM history
  WHERE tag_id = ANY($1) AND ts >= $2 - $5::interval AND ts < $3 AND v_s IS NULL
), spans AS (                                  -- each row holds its value until the next row, but never longer than the hold limit
  SELECT tag_id, v, quality, ts AS t0,
         LEAST(COALESCE(lead(ts) OVER (PARTITION BY tag_id ORDER BY ts), $3),
               ts + $6::interval) AS t1        -- hold limit = 1.5 x max_interval: a silent gateway is not "holding" a value
  FROM pts
), cut AS (                                    -- split spans at bucket edges, clip to [start, end)
  SELECT s.tag_id, b.bucket, s.v,
         GREATEST(s.t0, b.bucket, $2) AS a,
         LEAST(s.t1, b.bucket + $4::interval, $3) AS z
  FROM spans s
  CROSS JOIN LATERAL generate_series(
       date_bin($4::interval, GREATEST(s.t0, $2), TIMESTAMPTZ '2000-01-03 00:00:00+00'),
       date_bin($4::interval, LEAST(s.t1, $3) - interval '1 microsecond', TIMESTAMPTZ '2000-01-03 00:00:00+00'),
       $4::interval) AS b(bucket)
  WHERE s.quality < 128 AND s.t1 > s.t0
)
SELECT tag_id, bucket,
       sum(v * extract(epoch FROM (z - a))) / nullif(sum(extract(epoch FROM (z - a))), 0) AS twavg
FROM cut WHERE z > a GROUP BY tag_id, bucket ORDER BY tag_id, bucket;
```

Spans of bad-quality rows contribute nothing (their time is excluded from numerator and denominator), so a comms outage does not drag the average toward the last good value. It is restricted to ranges whose estimated row count is below 2 M (section 10), is not available beyond raw retention, and is the reason `history_15m` does not try to be time-weighted.

## 5. Source selection and the rollup tail

For `agg != raw` the planner chooses per request:

| Condition | Source |
|---|---|
| `interval` is not a multiple of 15 minutes | raw (section 4) |
| `end - start` is short: estimated raw rows <= 200,000 (see below) and the request is not for older-than-raw data | raw (section 4); exact and cheap |
| any part of `[start, end)` is older than `now - default_retention` (raw dropped) | rollup for that part; if `interval < 15 min` the request fails with `422` (`code: out_of_range`, detail "requested resolution is no longer retained") |
| otherwise (long range, aligned interval) | rollup, plus raw tail |

Estimated raw rows = `sum over tags (range seconds x recent rows/s for that tag)`, using an in-memory per-tag rate (an exponentially weighted rows-per-hour kept by the writer, so no `count(*)` is ever issued).

**Materialization boundary and tail.** The rollup is materialized-only (02, section 5). The boundary is read once per second and cached:

```sql
SELECT max(bucket) + INTERVAL '15 minutes' AS boundary
FROM history_15m
WHERE bucket > now() - INTERVAL '3 days';
```

Buckets before `boundary` come from the rollup; `[boundary, end)` is aggregated from raw rows with the section 4 query at 15-minute resolution. The two result sets are concatenated and then re-bucketed together (section 6). The tail is at most about 20 to 35 minutes of data unless the refresh job is behind, in which case the tail grows and the query slows but stays **correct**. A refresh job that is more than 2 hours behind raises `_system/historian/rollup_lag_s` and an alarm (02, section 7).

## 6. Re-bucketing the rollup to the requested interval

`interval` is any multiple of 15 minutes (for example `1h`, `6h`, `24h`):

```sql
WITH base AS (
  SELECT tag_id, bucket, n_total, n_usable, worst_q, v_min, v_max, v_sum, v_first, v_last, ts_first, ts_last
  FROM history_15m
  WHERE tag_id = ANY($1) AND bucket >= $2 AND bucket < LEAST($3, $boundary)
  UNION ALL
  SELECT ... FROM <section 4 query at 15 minutes over [boundary, end)>
)
SELECT tag_id,
       time_bucket($interval, bucket, origin => TIMESTAMPTZ '2000-01-03 00:00:00+00')              AS b,
       sum(n_usable)                                                                               AS n_usable,
       max(worst_q)                                                                                AS worst_q,
       min(v_min)                                                                                  AS v_min,
       max(v_max)                                                                                  AS v_max,
       sum(v_sum) / nullif(sum(n_usable), 0)                                                       AS v_avg,
       first(v_first, ts_first) FILTER (WHERE v_first IS NOT NULL)                                 AS v_first,
       last (v_last,  ts_last)  FILTER (WHERE v_last  IS NOT NULL)                                 AS v_last
FROM base
GROUP BY tag_id, b
ORDER BY tag_id, b;
```

`min` of minima, `max` of maxima, `sum` of sums over `sum` of counts, first of firsts, last of lasts are all exact re-aggregations, so the result equals what a raw query would have returned. The conformance suite asserts equality of the two paths on random data (rollup path versus raw path), which is the most valuable single test of this file.

## 7. `delta` for cumulative totals

api.md: "`delta` = last minus first within the bucket, for monotonic totals (daily consumption). It is computed on `total_volume`, not on raw counts." Two implementations are provided; the literal one is selectable.

**Literal (`delta`, as the contract defines it):** `v_last - v_first` of the bucket. **It loses the increment between the last row of the previous bucket and the first row of this one.** On `total_volume` every row is about one pulse, so literal daily deltas lose about 1 L per day and literal 15-minute deltas about 1 L per bucket that has flow. That is small for daily water consumption. It is **not** small in general: for a total that updates every 15 minutes in steps of 5 units (an energy meter), literal hourly deltas lose one of every four increments (25%), and literal daily deltas lose 1 in 96 (about 1%). A billing report built on literal deltas is wrong by an amount that depends on update granularity.

**Carry-forward (`delta` as storage recommends, exposed as `agg=delta` with `?carry=true` until the contract is amended):** the value at the end of a bucket minus the value at the end of the previous bucket that has usable data, with the value before the first bucket taken from the boundary sample (section 3):

```sql
WITH series AS (                                  -- output of section 5/6, usable buckets only
  SELECT tag_id, b, v_last FROM rebucketed WHERE v_last IS NOT NULL
), start_v AS (                                   -- boundary sample per tag (section 3), as a one-row-per-tag set
  SELECT tag_id, v AS v0 FROM boundary
)
SELECT s.tag_id, s.b,
       s.v_last - COALESCE(lag(s.v_last) OVER (PARTITION BY s.tag_id ORDER BY s.b), sv.v0) AS delta
FROM series s LEFT JOIN start_v sv USING (tag_id)
ORDER BY s.tag_id, s.b;
```

- The sum of carry-forward deltas over any range equals `end_value - start_value` **exactly**; the sum of literal deltas does not. That identity is the property test.
- Buckets with no usable data stay in the output with `value: null, quality: bad` (api.md); the **next** bucket's delta covers the consumption across the empty bucket. To keep that honest the point after a gap is returned with quality `uncertain` (proposed; api.md does not define it).
- **Counter adjustments.** A `counter_discontinuity` with `kind: manual_adjust` is an operator writing `total_volume` (tag-model section 7); the jump is not consumption. A consumption report subtracts it: for each such event in range, `total_after - (last total_volume row before event.time)`. The event has `total_after` but not `total_before` (README open question 6), so the report reads the preceding history row. `reset` events need no correction: `counter_total` keeps the total monotonic across reboots by construction.

## 8. Gaps and coverage

A flat line and a gap must look different (tag-model: `max_interval` exists for exactly this). Per tag the **hold limit** is `1.5 x history.max_interval` (or `1.5 x interval` in `sampled` mode), stored in `tag_catalog.history`.

```sql
SELECT tag_id, prev_ts AS gap_from, ts AS gap_to
FROM (
  SELECT tag_id, ts, lag(ts) OVER (PARTITION BY tag_id ORDER BY ts) AS prev_ts
  FROM history
  WHERE tag_id = ANY($1) AND ts >= $2 - $4::interval AND ts < $3
) x
WHERE prev_ts IS NOT NULL AND ts - prev_ts > $4::interval;      -- $4 = hold limit (a single value per query; per-tag limits are joined in the generic form)
```

Causes of a gap, in the order a report should explain them: gateway stopped (audit `system.stopped` to `system.started`, 08), database or spool outage (`skyranger.storage.history_gap.v1` events, 04), device offline (quality rows `uncertain_last_known` or `bad_*` are stored, so these are *not* gaps; they are visible bad-quality spans), tag disabled (`bad_out_of_service`). In rollups a gap is an **absent bucket**; `n_total = 0` is never stored. Reports include a `coverage` figure: `1 - (sum of gap seconds over the period) / period`, and the water meter report prints it next to every daily consumption.

## 9. Local-day and local-month consumption

Billing and reports are in the plant's calendar (`Gateway.spec.timezone`, `tz` request parameter). Rollup buckets are 15 minutes, which always aligns with local midnight (02, section 5), so the calendar roll-up is exact and cheap:

```sql
WITH q AS (
  SELECT bucket, v_last
  FROM history_15m
  WHERE tag_id = $1
    AND bucket >= ($2::timestamptz - INTERVAL '2 days')       -- one extra day before start for the carry
    AND bucket <  $3
    AND v_last IS NOT NULL
), days AS (
  SELECT date_trunc('day', bucket AT TIME ZONE $4::text)                              AS local_day,   -- timestamp without time zone, DST-correct (23 or 25 hour days handled)
         (array_agg(v_last ORDER BY bucket DESC))[1]                                  AS v_end
  FROM q GROUP BY 1
)
SELECT local_day AT TIME ZONE $4::text                                               AS ts_utc,       -- back to UTC for the API
       v_end - lag(v_end) OVER (ORDER BY local_day)                                  AS consumption
FROM days
WHERE local_day >= date_trunc('day', $2::timestamptz AT TIME ZONE $4::text)
ORDER BY local_day;
```

For months use `date_trunc('month', ...)`. Because it is end-of-day minus end-of-previous-day it is the carry-forward delta of section 7 at a calendar boundary. The raw-tail union of section 5 applies to the current (incomplete) day. A day with no data at all is simply missing from `days`, and the following day's consumption spans it (flagged `uncertain` as in section 7). If the rollup has been dropped by `aggregate_retention`, the question cannot be answered, and the API says so (`out_of_range`).

## 10. Concurrency, limits, cancellation

| Control | Value | Why |
|---|---|---|
| Concurrent history queries | 4 (the query pool size); waiting room of 16, then `429 rate_limited` with `Retry-After: 5` | the writer and the compression job share the box |
| `statement_timeout` (query pool) | 30 s, returns `504 timeout` (api.md) | |
| Series per request | 50 (api.md) | |
| Raw rows per series per page | default 10,000, max 100,000 | |
| Estimated rows scanned per request (planner estimate, section 5) | refuse above 5 M (`422`, detail suggests a coarser `interval`) | a quiet sanity limit, not a hard SQL guard |
| `twavg` | estimated raw rows <= 2 M | window functions over raw rows |
| Export (`/history/export`) | `export` pool, no timeout, `export_max_rows` default 50 M, streamed (rows written to the response as they are read; one pooled connection, no result buffering) | |
| Cancellation | the request context is passed to every query; a client disconnect cancels the statement | an abandoned HMI tab must not hold a connection for 30 s |
| Priority | none inside PostgreSQL; the cap above is the control | |

`GET /history/export` writes the audit record `history.exported` **before** streaming when the estimated row count exceeds 1 M (07-security/08 section 1).

## 11. Plans must be checked, not hoped for

For each query shape above the CI suite stores `EXPLAIN (FORMAT JSON)` against a database with 30 days of simulator history (some chunks compressed) and asserts the **node types**, not costs:
- raw window: an index scan using `history_pk` (or a `ChunkAppend` over per-chunk index scans) with chunk exclusion by `ts`; on compressed chunks a `DecompressChunk` with the `tag_id` filter pushed into the segment scan (node names UNVERIFIED across versions);
- rollup query: scans of `history_15m` (or its materialization hypertable) with `bucket` range pruning, no `Seq Scan` over `history`;
- no query touches more chunks than the time range overlaps.

Design targets for the typical-profile database (10,000 tags) on the recommended 4-core box. These are targets to test against, not measurements: one tag, 24 h raw: p95 below 100 ms; one tag, 30 days from the rollup: p95 below 200 ms; 20 tags, 7 days at 1 h: p95 below 500 ms; one tag, 1 year of daily consumption: p95 below 300 ms.

## Decisions made
1. Tags are resolved to `tag_id` first; every history query filters `tag_id` and a half-open `[start, end)` time range; the query layer cannot construct an unfiltered history query.
2. Raw reads are per-series `LATERAL` index scans with per-series keyset cursors.
3. Aggregates use only usable samples; bucket quality is the worst usable class; empty usable sets are `null, bad`.
4. The planner picks raw or rollup by estimated rows and alignment; the rollup is unioned with a raw tail at the measured materialization boundary; results of both paths are tested equal.
5. `avg` is the arithmetic mean of stored samples as the contract says; time-weighted mean is a separate, bounded, raw-only computation (proposed `agg=twavg`).
6. `delta` is implemented in both literal and carry-forward forms; carry-forward is recommended for anything that is billed.
7. Local-day and local-month reports roll up from the 15-minute rollup with `AT TIME ZONE`, so DST days come out right.
8. Gaps are defined by a per-tag hold limit (`1.5 x max_interval`) and are reported with a coverage figure.

## Alternatives rejected and why
- **`WHERE tag_id = ANY(...) ORDER BY tag_id, ts LIMIT n`** for multi-series raw reads: a single limit across series and a cursor that cannot resume per series; the lateral form is simpler to paginate and plans better.
- **Real-time aggregation views** to hide the tail: version-dependent and hides cost (02).
- **Server-side interpolation or LTTB downsampling**: the HMI does min/max envelope per pixel column with two aggregate requests; interpolation is a presentation choice (api.md open question) and LTTB needs the Toolkit extension.
- **Computing daily consumption from `daily last - daily first`**: the literal-delta undercount of section 7.
- **`count(*)` estimates per request**: expensive on compressed hypertables; an in-memory per-tag rate is free.
- **Caching query results in the gateway**: invalidation (late data, replays, retention) is exactly the class of bug a SCADA historian cannot afford; queries are cheap enough without it.
- **A separate read replica for trends**: valid in T2 or T3 deployments (read from the standby); not needed on one box.

## Open questions
1. **Range convention:** api.md does not state whether `end` is inclusive. This file assumes half-open `[start, end)`; the contract should say so.
2. **Contract `delta` and `avg`** (README open questions 2 and 3): carry-forward delta and `twavg` are proposals. A `?carry=true` parameter is an interim, additive switch.
3. `boundary=true`, an `uncertain` quality after a gap, a `coverage` field in the response, and `out_of_range` (422) for "no longer retained" resolutions are additive API proposals.
4. Multi-series raw `limit` semantics and the cursor shape (section 2) need agreement with api.md.
5. First/last NULL handling and `time_bucket` origin defaults are UNVERIFIED (02).
6. Whether the HMI wants min and max as one request (`agg=min,max`), which would halve envelope-chart queries; additive, not decided.
7. Performance targets in section 11 are guesses until measured on the target box.
