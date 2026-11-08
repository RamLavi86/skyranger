# 03 Write Path

Status: draft. Boundary: the tag engine decides **which** values become history rows ([pipeline.md](../01-tag-engine/pipeline.md) section 7, the "history gate": mode, deadband, `max_interval`, quality changes). The historian decides **how** those rows reach disk safely. This file starts at the `HistorySink` hand-off and ends at committed rows. History never travels over MQTT (messages.md), and the tag engine never blocks on this component (driver-interface.md section 6, tag-engine README principle 4).

## 1. What is upstream, what is here

| Concern | Owner | Where defined |
|---|---|---|
| `history.enabled`, `mode: on_change \| sampled`, `interval`, `max_interval`, `deadband` | tag engine | tag-model.md section 9; pipeline.md section 7 |
| Choice of effective `ts`, skew rule | tag engine | tag-model.md section 5 |
| Which samples are history-only (out of order) | tag engine | pipeline.md stage 7 |
| Timers for `max_interval` and `sampled` ticks | tag engine timer wheel | pipeline.md section 7 |
| Queue, batching, dedupe, retries, spool, replay, dead letters | **storage** | this file and 04 |
| Sanity window on `ts` (garbage clock) | **storage** (backstop) | section 8 |
| Per-tag flood guard | **storage** (backstop) | section 7 |

The rules the engine applies, restated only so the storage consequences are visible:

| Mode | Row produced when | Storage consequence |
|---|---|---|
| `on_change` (default) | value passes `history.deadband` (compared to the last **stored** row), or quality changes, or `max_interval` (default `15m`) elapsed | `max_interval` rows carry `ts = now`, no `ts_source`, and (proposed) `flags` bit 0 `keepalive`. A flat line is distinguishable from a gap. Each tag costs at least `86400 / max_interval` = 96 rows/day, so `max_interval` is the knob that sets the **floor** of storage (09) |
| `sampled` | every `interval` on wall-clock aligned ticks, plus every quality change | one row per tick per tag, regardless of change. Predictable, and the most expensive mode for slow tags |
| history-only | out-of-order device-clock sample | `flags` bit 1 `out_of_order`; goes through the same pipeline, lands in an older (possibly compressed) chunk |

Two storage-relevant behaviors follow from the engine design:
- **After a gateway restart** the engine's `lastHistory` is empty, so every history-enabled tag writes one row on its first sample. That row marks "gateway was back at this time". Combined with the audit events `system.stopped` and `system.started` and the quality rows (`bad_waiting_for_initial_data` is not stored; the first good sample is), a report can show the downtime window honestly.
- **Quality changes always pass the deadband**, so a device that goes quiet produces exactly one `uncertain_last_known` row, not a stream.

## 2. The hand-off

The engine calls a non-blocking function. The contract with the engine (to be mirrored in the tag-engine Go design doc, which does not exist yet, README cross-area request):

```go
package historian

// Sample is the storage view of one history row. The engine builds it from tag.Value.
type Sample struct {
    TagID     uuid.UUID
    TS        time.Time   // effective timestamp, ms precision, UTC
    TSGateway time.Time
    TSSource  time.Time   // zero if absent
    Quality   tag.Quality
    Flags     Flags       // KeepAlive, OutOfOrder
    V         tag.Variant // zero Variant = null
}

type Sink interface {
    // Offer never blocks and never performs I/O. It reports what happened to the sample.
    Offer(s Sample) OfferResult
}

type OfferResult uint8
const (
    Accepted  OfferResult = iota // in the ring; will be written or spooled
    Dropped                      // ring full AND spool refused (disk shed state); counted
)
```

`Offer` is a single atomic ring-buffer push (multi-producer is not needed: the engine goroutine is the only producer, so a single-producer ring with an atomic head is enough). It returns `Dropped` only in the shed state of 08 (disk at 95%) or when the ring is full and the writer is itself blocked on a full disk. Everything else is accepted.

## 3. Pipeline

```
 engine goroutine ──Offer──► ring (65,536 samples, ~12 MB) ──► writer goroutine
                                                               │
                              flush when:  >= 2,000 samples  OR  500 ms since first unflushed sample
                                                               ▼
                                          batch (sorted by tag_id, ts)
                                           │ state == direct               │ state == spooling
                                           ▼                               ▼
                                  INSERT ... unnest ... ON CONFLICT   spool.Append(frame)  (04)
                                  DO NOTHING   (one transaction)
                                           │ unavailable / timeout
                                           └──► state = spooling, batch goes to spool (04)
```

Why a ring and one writer goroutine: simple, bounded memory (the ring is a fixed array), ordered per producer, and trivially testable with a fake clock. Parallel writers buy nothing at these rates (09: 800 rows/s busy on 10,000 tags) and would reorder inserts into the same index pages.

### Batch parameters (profile defaults, overridable in `site.yaml storage.historian`)

| Parameter | `meter` | `small` | `medium` and `large` | Meaning |
|---|---|---|---|---|
| `ring_size` | 16,384 | 65,536 | 262,144 | samples held before the engine-facing hand-off must shed |
| `flush_max_samples` | 500 | 2,000 | 5,000 | batch size trigger |
| `flush_max_wait` | `500ms` | `500ms` | `250ms` | time trigger (bounds the power-cut loss window with the DB up) |
| `write_timeout` | `10s` | `10s` | `15s` | per-batch deadline; exceeding it counts as unavailable |
| `max_rows_per_statement` | 5,000 | 5,000 | 5,000 | large replays are split |

At the water meter's tens of rows per minute a "batch" is usually one to five rows flushed on the 500 ms timer, one transaction each. That is about two commits per second at most while flow is active, which is trivial even on a cheap SSD.

## 4. The insert statement

```sql
INSERT INTO history (tag_id, ts, gw_offset_ms, src_offset_ms, v_f, v_i, quality, flags, v_s)
SELECT * FROM unnest($1::uuid[], $2::timestamptz[], $3::bigint[], $4::bigint[],
                     $5::float8[], $6::bigint[], $7::smallint[], $8::smallint[], $9::text[])
ON CONFLICT (tag_id, ts) DO NOTHING;
```

- Driver: `pgx` v5, nullable array elements via `[]*float64` and friends. One round trip per batch, no per-row statements. The batch is sorted by `(tag_id, ts)` before the call, which improves index page locality and removes lock-order differences between the live writer and a concurrent replay.
- `ON CONFLICT DO NOTHING` makes every insert idempotent: a batch whose commit outcome is unknown (connection died after `COMMIT` was sent) can be resent. The command tag gives the inserted row count; `submitted - inserted` is exported as `historian_dedup_rows_total`. It is expected to be zero outside replays.
- **This deliberately differs from the `COPY` mention in driver-interface.md section 6.** `COPY` cannot express `ON CONFLICT`; staging into a temp table and `INSERT ... SELECT` adds a second write. `unnest` inserts are fast enough (10,000s of rows/s on one connection, UNVERIFIED, benchmark at milestone 1) for the largest profile this area targets on an edge box. If a benchmark shows otherwise at `large`, the writer gains a `COPY` into an unlogged staging table path, with no change to the interface. The contract sentence is a description, not a requirement; it should be edited to "batch inserts".
- Two different samples for the same tag in the same millisecond collapse to the first one. The engine truncates to milliseconds and evaluates derived tags once per batch, so this needs two source changes within 1 ms of the same tag, which for physical signals is noise. It is counted (dedup counter) and not hidden.

## 5. Writer state machine

```
          start (spool non-empty or empty)
             │
             ▼
        ┌─────────┐   N=2 consecutive ErrUnavailable, or one connection loss
        │ direct  │──────────────────────────────────────────────┐
        └─────────┘                                               ▼
             ▲                                             ┌──────────┐
             │ probe SELECT 1 ok (backoff 1s..30s)         │ spooling │  batches ──► spool, fsync
             │                                             └────┬─────┘
             │                                                  │ probe ok
        ┌──────────┐◄────────────────────────────────────────────┘
        │ draining │  live batches go direct; a background replayer reads the spool, throttled
        └──────────┘
             │ spool empty  ──► direct
```

Rules:
- **Live data always goes direct when the DB is reachable, even while draining.** Row order in the database is irrelevant (key `(tag_id, ts)`), so there is no ordering invariant to preserve and no reason to queue new data behind old.
- **Replay is throttled** (default 5,000 rows/s, at most 50% of the writer pool) so a long backlog cannot starve live inserts or the query pool. Replay runs at lower priority: a live batch that has waited longer than `flush_max_wait` pre-empts the replayer between statements.
- **Unknown outcome is safe.** If a batch was sent and the connection dropped, the batch is spooled whole; if it had in fact committed, replay finds every row already present (dedup). The writer never tries to determine which happened.
- A **health probe** runs only in `spooling`: `SELECT 1` on a fresh connection with backoff `1s, 2s, 4s ... 30s`. It is not run in `direct`, where real writes are the probe.
- Transition to or from `spooling` emits event `skyranger.storage.state_changed.v1` (proposed) and updates `_system/historian/db_state` (0 direct, 1 draining, 2 spooling, 3 shedding). **An alarm on `db_state != 0` for more than 60 s is part of the default system alarm set** (alarm area owns the definition).
- After a replay that touched data older than the cagg `start_offset` (3 days), the replayer calls `Admin().RefreshAggregates` for the touched range (02, section 5).

## 6. Error classification

The writer maps PostgreSQL errors to the storage error kinds (11, section 5). SQLSTATE classes are from the PostgreSQL manual (error codes appendix, from memory, low risk):

| Condition | SQLSTATE | Kind | Writer action |
|---|---|---|---|
| connection refused, reset, DNS, TLS failure, `Class 08` | `08xxx`, network errors | `ErrUnavailable` | spool, probe with backoff |
| server shutting down or starting | `57P01`, `57P02`, `57P03` | `ErrUnavailable` | same |
| too many connections, out of memory | `53300`, `53200` | `ErrUnavailable` | same, plus warning (pool misconfigured) |
| disk full or quota | `53100`, ENOSPC from the spool | `ErrDiskFull` | disk ladder (08); history shed |
| deadlock, serialization failure | `40P01`, `40001` | retryable | retry the batch up to 3 times with jitter, then spool |
| statement timeout, our deadline | `57014`, `context.DeadlineExceeded` | `ErrUnavailable` | spool (a slow DB is an unavailable DB for history) |
| data error, constraint violation, bad text encoding | `22xxx`, `23xxx` | `ErrInvalid` (poison) | **bisect** the batch (halves) down to the offending rows, write the rest, move the poison rows to `spool/historian/dead/` (04), count, emit event. Never retry forever, never block |
| data or index corruption | `XX001`, `XX002` | `ErrCorrupt` | stop writes to history, spool continues (bounded), critical alarm `_system/storage/corruption` (08) |
| permission denied, undefined table | `42501`, `42P01` | `ErrInvalid` | treated as fatal config error: log, critical alarm, spool (a migration or role problem, not data) |

A retry policy that is always "spool and probe" is intentional. A SCADA historian that tries clever things on an unhealthy database (for example, retrying inside the engine's goroutine) is what turns a database fault into a control-plane outage.

## 7. Flood guard (storage backstop)

A tag with `deadband` unset on a noisy sensor can produce hundreds of rows per second. The engine owns deadband; storage protects itself.

- Per-tag token bucket: sustained `20 rows/s`, burst `200`, per tag, in the writer (a `map[uuid.UUID]*bucket` sized by live history tags; rows refilled lazily). Quality-change rows are **exempt**. Out-of-order and keepalive rows are not.
- Rows over the limit are dropped and counted per tag: `historian_guard_dropped_total{tag_id}` is exported to `_system/historian/guard_dropped` (total) and the top five offenders by `tag_id` appear in an event `skyranger.storage.history_flood.v1` (proposed), emitted at most once per tag per hour. This is the same philosophy as the alarm flood handling (04-alarms): the number must be visible, the system must stay up, the fix is configuration.
- Defaults are conservative (20 rows/s per tag is above any sensible trend); `site.yaml storage.historian.guard: off` disables it for a site that really wants raw 100 Hz data. Not a contract change: dropping happens after the engine's deliberate decisions and is reported.

## 8. Sanity window and the clock

A cheap box without a working RTC can boot with the clock in 1970 or in the BIOS build date; chrony then steps it forward minutes later. The engine already rejects bad **device** timestamps (tag-model section 5), but `ts_gateway` and gateway-stamped `ts` come from the gateway's own clock.

- **Start-up gating** (deployment): `skyrangerctl preflight` and the unit ordering wait for a plausible clock before `skyrangerd` starts ([04-install-and-first-boot.md](../08-deployment/04-install-and-first-boot.md)).
- **Storage backstop:** a sample with `ts < 2020-01-01T00:00:00Z` (the constant tag-model.md uses) or `ts > now + 5 min` is **not inserted**. It is written to `spool/historian/dead/` with a reason code, counted in `historian_rejected_ts_total`, and an event is emitted (rate-limited). Nothing is lost permanently: an operator can re-time dead-letter rows after a known clock fault with `skyrangerctl history redeliver --shift <duration>` (proposed, not specified here).
- A **clock step during operation** (chrony step of more than a few seconds) is a known hazard that this layer cannot repair: rows written before the step have the old time. The writer records the step as an event when the monotonic-versus-wall offset jumps by more than `max_clock_skew` (default 10 s, the tag-model constant), so reports can flag the interval. Prevention (slew only, `makestep` limited to start-up) is the deployment area's chrony configuration (`11-time-and-power`, referenced by the deployment README but not yet written).

## 9. Startup, shutdown, restart

- **Start:** open the spool, truncate any torn tail (04), set state `draining` if non-empty, `direct` otherwise; the first DB write decides. The ring starts empty.
- **Graceful stop (SIGTERM, deployment gives 30 s):** stop accepting from the engine after the engine has flushed its last batch; flush the ring to the DB with a 5 s budget; whatever is not committed by then is appended to the spool and fsync'd; close. Nothing acknowledged is lost on a graceful stop.
- **Crash or power cut with DB up:** lost = samples in the ring and the unflushed batch, at most `flush_max_wait` (500 ms) of data plus whatever was in flight. This window is the price of not writing every sample twice (rejected: an always-on write-ahead log, 4 and the alternatives below). For cumulative counters (the water meter) the loss is invisible: the next sample's delta recovers it (tag-model section 7). For gauge values it is a half-second hole.
- **Crash or power cut with DB down:** lost = at most the unfsynced tail of the spool (fsync interval `100ms` or every 1,000 samples, 04).
- **PostgreSQL restart** (container update, crash): treated as unavailable; the spool bridges it; the engine never notices.

## 10. Other write paths (not history)

These do not use the historian writer. All use the `state` pool and ordinary transactions:

| Writer | Frequency | Statement shape | Durability rule |
|---|---|---|---|
| derived/memory checkpoint | at most 1 s, dirty rows only | one multi-row upsert per tick | a memory-tag write is reported `ok` only after commit (pipeline.md section 9). `counter_total` checkpoints are lossless by construction (cumulative counter) |
| alarm transitions | event-driven, group commit within 50 ms | one transaction: `alarm_state` upsert, `alarm_journal` insert, `events` insert, outbox rows | persist, then announce (alarms README) |
| events (other) | event-driven | batched insert, `ON CONFLICT (time, id) DO NOTHING` | spool on DB-down (04, stream `events`) |
| config apply | rare | one transaction (05) | fail closed |
| audit | per mutating request | one insert under advisory lock | fail closed for the change (07-security/08 section 4) |
| discovery | 60 s per device at most | upsert | best effort |

## 11. Observability (proposed `_system` tags and metrics)

Gateway-generated tags under `_system/historian/` (additive; `_system` paths may change between versions per tag-model.md), also exported as Prometheus-style metrics on the loopback metrics endpoint if one exists (not in the contracts yet):

| Tag | Type | Meaning |
|---|---|---|
| `db_state` | int64 enum | 0 direct, 1 draining, 2 spooling, 3 shedding |
| `queue_depth` | uint64 | samples in the ring |
| `rows_written_total` | uint64 | inserted (non-duplicate) rows |
| `rows_per_s` | float64 | 1-minute rate |
| `batch_ms_p95` | float64 | write latency |
| `spool_bytes`, `spool_oldest_age_s` | uint64 | backlog size and age |
| `dropped_total` | uint64 | `Dropped` offers (disk shed, ring overflow) |
| `guard_dropped_total` | uint64 | flood guard |
| `rejected_ts_total` | uint64 | sanity window |
| `dedup_rows_total` | uint64 | conflicts on insert |
| `rollup_lag_s` | uint64 | now minus the newest materialized `history_15m` bucket end (10, section 5; alarm above 2 h) |
| `job_failing` | bool | any Timescale compression, retention or refresh job failing for over 1 h (02, section 7) |
| `dead_letter_rows_total` | uint64 | poison rows |
| `db_bytes`, `disk_used_pct` | uint64, float64 | 08-disk (`disk_used_pct` is already proposed by 07-security/08) |

These tags are history-enabled with `max_interval: 1h` to keep their own storage floor low (09 counts 20 system tags). Alarms on them are in the system alarm set, not hand-written per site.

## Decisions made
1. The tag engine decides what is stored; the historian decides how. The seam is a non-blocking `Offer` into a single-producer ring.
2. Batched `INSERT ... SELECT FROM unnest(...) ON CONFLICT (tag_id, ts) DO NOTHING`, sorted, one transaction per batch, one writer goroutine. Idempotency is the safety property everything else (spool, replay, restore, failover) relies on.
3. Live writes bypass the spool whenever the DB answers, even during replay; replay is throttled and pre-emptible.
4. "Spool and probe" is the only reaction to an unhealthy DB; no clever retries on the engine's path.
5. Poison rows are bisected and dead-lettered, never retried forever and never allowed to block the queue.
6. A per-tag flood guard (20 rows/s sustained) and a timestamp sanity window protect the database from upstream mistakes; both are counted and evented, never silent.
7. Power-cut loss window with the DB up is bounded by `flush_max_wait` and documented.
8. `synchronous_commit` stays on for the writer.

## Alternatives rejected and why
- **Always-on write-ahead log in front of the DB** (write every sample to disk, then to Postgres): closes the half-second loss window but doubles writes on flash and adds a second durable store whose state must be reconciled. The window is harmless for counters and tolerated for gauges. The decision is revisitable per site if a customer needs zero loss, in which case the answer is an industrial box with a UPS and power-fail signal ([03-packaging-and-hardware.md](../08-deployment/03-packaging-and-hardware.md)).
- **`COPY` for everything**: no `ON CONFLICT`; see section 4.
- **Per-row `INSERT` through the pgx batch API**: more round trips and statement parsing for the same effect.
- **Multiple parallel writers**: no benefit at 800 rows/s, reorders index inserts, harder to reason about shutdown.
- **`synchronous_commit = off` for history**: tempting, but the writer would delete spooled data that a crash could still lose. Correctness over a saving we do not need.
- **Dropping the oldest ring entries when full** (as the driver SDK coalesces): history is not latest-value-wins; each row is an event in time. The correct response to overload is to spool (disk is large and sequential) and, as a last resort, to count and drop at the edge of the system.
- **Writing history from the driver host or over MQTT**: contract decisions.
- **Storing unapproved devices' data**: messages.md says it is dropped.

## Open questions
- `Sink` name and shape must be mirrored in the tag-engine Go design (`go-design.md` is referenced by pipeline.md but not yet written). In particular whether `OfferResult` is useful to the engine or should be fire-and-forget with counters.
- Is 20 rows/s per tag the right guard default? It would clip a deliberately sampled 50 Hz vibration tag. The answer may be a per-tag `history.guard` field instead of a site switch.
- `skyranger.storage.state_changed.v1`, `skyranger.storage.history_flood.v1`, the `_system/historian/*` tags, and `skyrangerctl history redeliver` are contract additions (event catalog, `_system` tags) and need agreement.
- Benchmark `unnest` insert throughput and the effect of `ON CONFLICT` index lookups on the `large` profile (UNVERIFIED claims in section 4).
- Should the engine's `max_interval` keepalive rows be omitted from the DB and synthesized at read time (saves up to the entire floor of 96 rows/day/tag)? Rejected for now because a stored row is proof the gateway was alive; revisit if the floor dominates storage at 100,000 tags.
