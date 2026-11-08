# 09 Capacity

Status: draft. Everything here is arithmetic on stated assumptions. **No number below is a measurement.** The two bytes-per-row constants are estimates, and replacing them with measured values from the simulator is a milestone 1 exit criterion (section 10). Where this file disagrees with the sizing rule of thumb in [03-packaging-and-hardware.md](../08-deployment/03-packaging-and-hardware.md), this file is more conservative and the deployment doc should be updated (section 10).

## 1. Assumptions

| Symbol | Value | Basis |
|---|---|---|
| `B_unc` | **140 B** per uncompressed history row, heap plus primary-key index | heap tuple: 32-byte header (23 bytes plus a 2-byte null bitmap for 9 columns, padded to 8) + 60 bytes of fixed columns (16+8+8+8+8+8+2+2) = 92, rounded to 96, plus a 4-byte line pointer = 100; B-tree entry for `(uuid, timestamptz)`: about 8 + 16 + 8 = 32 plus pointer, at about 90% fill = 40. Total about 140. UNVERIFIED until `pgstattuple` on a real chunk |
| `B_cmp` | **16 B** per compressed history row | planning figure, about 9x. The deployment doc quotes 10x to 20x on 50 B rows (3 to 5 B/row); on-change timestamps are irregular and floats noisy, so this area plans at 16. UNVERIFIED, measure |
| `B_agg` | **25 B** per compressed `history_15m` row | about 100 B heap + 40 B index uncompressed, assumed about 5x |
| `B_spool` | **48 B** per sample in the spool | protobuf frame, section 3 of 04 |
| `B_csv` | **95 B** per exported CSV row | uuid-free path column plus timestamps; estimate |
| `ret_raw` | **400 d** | `Gateway.spec.history.default_retention` default (tag-model.md) |
| `ret_agg` | **3 y** (1,095 d) | proposed default, 02 section 6 |
| `compress_after` | **3 d** (meter: 14 d) | 02 section 4 |
| History-enabled share | **80%** of configured tags | the rest are memory tags, disabled, or `raw/uptime_s`-style points |
| Keepalive floor | `max_interval = 15 min` gives 96 rows/day per history tag | tag-model.md default |
| Rollup rows | 96 per history tag per day (upper bound: every bucket occupied) | 15-minute buckets |
| WAL and write amplification (WAF) | **3x** of `B_unc` per row | heap + index + WAL record + full-page images after checkpoints. UNVERIFIED; measure with `pg_stat_wal` |

## 2. Row-rate model

```
rows_per_day = history_tags x (changes_per_tag_per_day + keepalive_floor_per_day)       (changes dominate above 1 per 15 min)
```
For fleets we state the **average stored rows per history tag per period**: a tag that changes beyond its deadband once per `P` seconds averages `86400 / P` rows/day. Three activity profiles, applied to all history-enabled tags:

| Profile | `P` | Meaning |
|---|---|---|
| quiet | 300 s | slow process values with sensible deadbands, mostly flat |
| typical | 60 s | mixed plant: levels, flows, statuses |
| busy | 10 s | fast signals with tight or no deadbands (the rate used in the deployment doc's example) |

A configuration with `mode: sampled`, `interval: 1s` on every tag is **86,400 rows/day/tag** and is outside all three profiles; it is the thing the flood guard and the commissioning check exist to catch.

## 3. The water meter (milestone 1)

One ESP32 meter (1 pulse = 1 L) instantiating `pulse_water_meter`, plus simulated devices counted as zero here (add one meter's worth per simulated meter). History-enabled tags from the template: `raw/pulse_count`, `raw/boot_count`, `raw/online`, `raw/rssi_dbm`, `total_volume`, `flow_rate`, `health`, `leak` (8; `raw/uptime_s` is disabled by the template). Plus 20 `_system` tags assumed at `max_interval: 1h`.

| Series | Low use: 5 m3/day | High use: 100 m3/day | Derivation |
|---|---|---|---|
| `raw/pulse_count` rows | 5,000 | 86,400 | one row per device message; the device sends at most 1 msg/s (messages.md section 2). Low: pulses are sparse, 5,000 L = 5,000 messages. High: 100,000 pulses/day exceed 86,400 seconds, so coalescing caps it at 1 row/s |
| `total_volume` rows | 5,000 | 86,400 | `counter_total` emits on each sample |
| `flow_rate` rows | 10,000 | 8,640 | `rate` over a 5-minute window changes when a pulse enters and again when it leaves (2 per pulse); at high use, bounded by `eval_interval: 10s` (8,640/day). Upper bounds |
| keepalive floor, 8 tags | 768 | 768 | 8 x 96 (conservative: `rssi` uses 1 h, 24/day) |
| `_system` tags | 480 | 480 | 20 x 24 |
| other changes (`boot_count`, `online`, `health`, `leak`, `rssi`) | 50 | 50 | assumed |
| **Rows per day** | **21,298** | **182,738** | sum |
| Rows per second (average) | 0.25 | 2.1 | |
| Uncompressed per day (`B_unc`) | 3.0 MB | 25.6 MB | |
| Compressed per day (`B_cmp`) | 0.34 MB | 2.9 MB | |
| 14-day uncompressed window | 42 MB | 358 MB | `compress_after = 14 d` on the `meter` profile |
| Compressed, 400 days | 136 MB | 1.17 GB | |
| Rollup, 3 years (28 tags x 96 x 1,095 x `B_agg`) | 74 MB | 74 MB | upper bound; most buckets of a quiet meter are empty |
| Raw CSV, 400 days | 0.8 GB | 6.9 GB | `B_csv` |
| Spool at 48 B/sample | 1.0 MB/day | 8.8 MB/day | section 6 |

Conclusion: **the milestone 1 database is tiny.** A 128 GB disk is hundreds of times more than needed; the real risks at this scale are correctness, restore, and power loss, not capacity. The fixed overheads (WAL up to 1 GiB, ballast 256 MiB, spool cap 512 MiB) are larger than the data for the first year. Steady state with overheads: 2.2 GB (low), 3.8 GB (high) (section 9).

Two observations worth keeping:
- At the low profile, **keepalive and `_system` rows (about 1,250/day) are 6% of the data and `flow_rate` is 47%.** The cheapest storage reduction is a coarser `flow_rate` deadband (for example 0.5 L/min), not compression.
- At the high profile the device's 1 msg/s coalescing cap, not the water usage, bounds the rate.

## 4. 10,000 tags

8,000 history-enabled tags. Rows/day = `8,000 x 86,400 / P`.

| | quiet (300 s) | typical (60 s) | busy (10 s) |
|---|---|---|---|
| Stored rows per second | 26.7 | 133.3 | 800 |
| Rows per day | 2.30 M | 11.52 M | 69.12 M |
| Uncompressed per day | 323 MB | 1.61 GB | 9.68 GB |
| Compressed per day | 36.9 MB | 184 MB | 1.11 GB |
| Resident uncompressed window (3 d) | 0.97 GB | 4.84 GB | 29.0 GB |
| Compressed, 400 days | 14.7 GB | 73.7 GB | 442 GB |
| Rollup, 3 years (768,000 rows/day x `B_agg`) | 21.0 GB | 21.0 GB | 21.0 GB |
| Chunk profile (02, section 3) | `small` (1 day) | `small` (1 day) | `medium` (6 h) |
| Write volume per day (WAF 3) | 0.97 GB | 4.84 GB | 29.0 GB |
| Write volume per year | 353 GB | 1.77 TB | 10.6 TB |
| Spool bytes per day | 111 MB | 553 MB | 3.32 GB |

The rollup size is the same for all three because it depends on tags, not on the row rate (upper bound: every 15-minute bucket occupied).

**Verdict.** Typical fits a 256 GB industrial SSD with room. Busy needs 1 TB at 400 days or a shorter raw retention (section 9 gives the days). The rollup is a significant part of the footprint for quiet sites (21 GB against 15 GB of raw): for those, shorten `aggregate_retention`.

## 5. 100,000 tags

80,000 history-enabled tags. Rows/day = `80,000 x 86,400 / P`.

| | quiet (300 s) | typical (60 s) | busy (10 s) |
|---|---|---|---|
| Stored rows per second | 267 | 1,333 | 8,000 |
| Rows per day | 23.0 M | 115.2 M | 691.2 M |
| Uncompressed per day | 3.2 GB | 16.1 GB | 96.8 GB |
| Compressed per day | 369 MB | 1.84 GB | 11.1 GB |
| Resident uncompressed window (3 d) | 9.7 GB | 48.4 GB | 290 GB |
| Compressed, 400 days | 147 GB | 737 GB | 4.42 TB |
| Rollup, 3 years | 210 GB | 210 GB | 210 GB |
| Chunk profile | `medium` | `large` | `large` (1 h; 3 d window is 290 GB) |
| Write volume per year (WAF 3) | 3.5 TB | 17.7 TB | 106 TB |
| Spool bytes per day | 1.1 GB | 5.5 GB | 33.2 GB |

**Verdict.** 100,000 tags is **not an edge-box target for v1.** Quiet and typical are server-class (T2 with a customer's database server, or central) with 0.5 to 2 TB. Busy is beyond this design: it needs a different recording strategy (coarser deadbands, `sampled` at 10 s for the few tags that need it, or a specialized history backend behind the interface in 11). The rollup alone at 100,000 tags is 210 GB for three years; `aggregate_retention` and which tags are rolled up (a per-tag opt-out, proposed) become real design levers. Single-node PostgreSQL write capability at 1,333 rows/s is comfortable and at 8,000 rows/s is plausible with batching on SSD but must be benchmarked (UNVERIFIED). Any statement beyond that would be invention.

## 6. Spool and outage capacity

Spool bytes per day = rows per day x 48 B. Caps by profile (04, section 7): `meter` 512 MiB, `small` 2 GiB, `medium` 4 GiB, `large` 8 GiB, at most 10% of the volume.

| Scenario | Rows/day | Spool/day | Cap | Outage held by the cap |
|---|---|---|---|---|
| Meter low | 21.3 k | 1.0 MB | 512 MiB | 525 days |
| Meter high | 182.7 k | 8.8 MB | 512 MiB | 61 days |
| 10k quiet | 2.30 M | 111 MB | 2 GiB | 19.4 days |
| 10k typical | 11.52 M | 553 MB | 2 GiB | 3.9 days (93 h) |
| 10k busy | 69.1 M | 3.32 GB | 4 GiB | 1.3 days (31 h) |
| 100k quiet | 23.0 M | 1.1 GB | 8 GiB | 7.8 days (derived from 8 GiB / 1.1 GB per day) |
| 100k typical | 115.2 M | 5.53 GB | 8 GiB | 1.55 days (37 h) |
| 100k busy | 691 M | 33.2 GB | 8 GiB | 0.26 days (6 h) |

`compress_after` should be at least the outage hours the spool can hold, or replay lands in compressed chunks (02, section 4). For 10k typical that is 3.9 days against the 3-day default: the 3-day default is slightly **less** than the spool's capacity. Either accept the slow replay path for outages longer than 3 days or raise `compress_after` to 4 days (costs 1.6 GB more uncompressed window). Recommended: `compress_after = max(2 x chunk interval, 3 d)` stays the default; sites with a documented multi-day outage risk raise it.

## 7. Throughput, WAL, memory, wear

- **Insert rate.** Peak design target is **5x the profile's average** (bursts at start-up, replay, config apply, many tags changing at once). Meter: negligible. 10k typical: 133 rows/s average, 670 rows/s peak. Replay is throttled at 5,000 rows/s by design (03), so the database sees at most that plus live traffic.
- **Index working set.** The primary key `(tag_id, ts)` appends per tag: only the rightmost leaf page of each tag's key range is hot, so the hot set is about `history tags x 8 KB`: 64 MB for 8,000 tags, 640 MB for 80,000. This is why a chunk's whole index does not need to fit in RAM, and why the "25% of RAM" chunk rule of thumb (02) is conservative.
- **Full-page writes.** After each checkpoint the first touch of each hot leaf page writes an 8 KB image: for 8,000 tags about 64 MB per checkpoint cycle (about 220 KB/s at the 5-minute default), included in the WAF 3 assumption; `wal_compression=on` reduces it (01, section 9).
- **SSD wear.** Write volume per year in sections 4 and 5 (WAF 3). 10k typical at 1.77 TB/yr is far below the endurance of an industrial 256 GB drive (endurance ratings are in hundreds of terabytes written; exact figures UNVERIFIED and vendor-specific); busy at 10.6 TB/yr is still modest. 100,000 busy at 106 TB/yr is the case where wear becomes the constraint, another reason it is out of scope.
- **Compression CPU.** A day of 10k typical is 1.6 GB uncompressed per chunk to compress once; on a 4-core box expect minutes per chunk (UNVERIFIED).
- **Memory.** `shared_buffers = 512 MB` (deployment default) is right for the meter and 10k quiet; 10k typical and above want 2 to 4 GB with 16 GB RAM (the deployment doc's recommendation). The Go process is small by comparison (tag engine memory is in its own performance doc); the ring is about 12 MB at `small`.
- **Trend queries** (10) at these sizes: one tag over 24 hours is at most thousands of rows; 30 days from rollups is 2,880 rows per tag. A dashboard of 20 tags is tens of thousands of rows. Not a capacity concern; the concurrency cap protects the writer.

## 8. Small tables, configuration, journals

| Item | Estimate | Basis |
|---|---|---|
| Config, 10,000 tags | about 2.5 MB YAML, about 1 MB as `jsonb` after TOAST | about 250 B of YAML per tag |
| Config, 100,000 tags | about 25 MB, about 10 MB `jsonb` | same; split across `TagGroup`s of at most 5,000 tags |
| Each config revision | tens of KB (changed documents) + manifest rows | content addressing (05) |
| `tag_catalog`, 100,000 tags | about 25 MB with indexes | about 250 B per row |
| `derived_state` | tens to hundreds of rows, KB | counters only |
| `events` | meter 20/day; plant 5,000/day at 500 B = 2.5 MB/day, 1 GB per 400 days | 06 |
| `alarm_journal` | similar to events, 3 years retention | alarm area |
| `audit_log` | about 1 KB per record; 1,000/day for 7 years = 2.5 GB | 07-security/08 |

None of these affects hardware choice.

## 9. Steady-state size formula (used by `skyrangerctl doctor`)

```
steady_state =  rows_day x B_cmp x ret_raw                       compressed raw history at full retention
              + rows_day x B_unc x compress_after_days            resident uncompressed window
              + history_tags x 96 x B_agg x ret_agg_days          rollups
              ) x 1.2                                             working space: compression, vacuum, index builds, bloat
              + spool_cap + max_wal_size (1 GiB) + ballast (1 GiB; 256 MiB meter)
              + local_backups (if kept on the same volume; usually 2 x the last backup)
require  steady_state <= 0.70 x volume_size                        the 70% rule: leaves the whole disk ladder (08) unused
```

Computed with the constants above (backups on another volume):

| Scenario | Steady state | Volume needed at 70% |
|---|---|---|
| Meter low | 2.2 GB | 3.1 GB |
| Meter high | 3.8 GB | 5.4 GB |
| 10k quiet | 48 GB | 69 GB |
| 10k typical | 124 GB | 177 GB |
| 10k busy | 597 GB | 853 GB |
| 100k quiet | 452 GB | 645 GB |
| 100k typical | 1.21 TB | 1.72 TB |

**Raw retention (days) that fits at 70%, same formula solved for `ret_raw`:**

| Volume | 10k quiet | 10k typical | 10k busy | 100k quiet | 100k typical |
|---|---|---|---|---|---|
| 128 GB | 1,331 | 245 | 17 | 0 | 0 |
| 256 GB | 3,357 | 650 | 84 | 0 | 0 |
| 512 GB | 7,408 | 1,460 | 219 | 189 | 16 |
| 1 TB | 15,130 | 3,005 | 477 | 961 | 171 |

Read: 10k typical with the default 400 days needs 177 GB, so a 256 GB drive works and a 128 GB drive allows 245 days. 10k busy on 256 GB allows 84 days of raw history. 100,000 tags do not fit small disks at any useful retention. (Entries of 0 mean the fixed costs alone exceed the budget; 15,130 days is just "unbounded" arithmetic.)

## 10. Reconciliation and what must be measured

**Against the deployment doc** ([03-packaging-and-hardware.md](../08-deployment/03-packaging-and-hardware.md)): its example (1,000 tags, 1 change/10 s, `~430 MB/day` uncompressed at 50 B, `25 to 45 MB/day` compressed, `10 to 16 GB` for 400 days) uses 50 B/row and 10x to 20x. This file's constants give for the same example: 8.64 M rows/day (1,000 tags at `P = 10 s`, no 80% factor), 1.2 GB/day uncompressed, **138 MB/day compressed**, **55 GB for 400 days**: roughly **3.5x to 5.5x** the deployment doc's figure. The deployment doc says the historian doc owns the real numbers and its section marks the figures UNVERIFIED; this is the more conservative estimate and the doc should adopt it until measured. Its hardware recommendations (256 to 1000 GB for 10,000 tags at 1 s) remain adequate for the typical profile.

**Milestone 1 measurements (exit criteria):**
1. `B_unc`: `pgstattuple` and `pg_total_relation_size` on a day-old uncompressed chunk with realistic data.
2. `B_cmp`: `hypertable_compression_stats('history')` after the policy has compressed a week of simulator data at the typical rate, for the `meter` and `small` profiles (UNVERIFIED function name).
3. `B_agg`: size of `history_15m` for the same period.
4. Rows per day of the real meter and of the simulator mix, to replace the assumed 5 and 100 m3/day.
5. Insert throughput at 1x, 5x, 10x the typical profile (single writer, `unnest` insert) and replay speed.
6. WAL bytes per row from `pg_stat_wal` (replaces WAF 3).
7. Pull-plug recovery time and `doctor`'s days-to-full prediction against the observed growth.

`skyrangerctl doctor` reports the measured `B_unc`, `B_cmp`, rows/day, projected days to full and projected steady state, so every real site becomes a data point that corrects this file.

## Decisions made
1. Plan with `B_unc = 140 B`, `B_cmp = 16 B`, `B_agg = 25 B`, `B_spool = 48 B`, WAF 3, until measured; they are more conservative than the deployment doc's.
2. Three activity profiles (quiet 300 s, typical 60 s, busy 10 s) over 80% of tags, plus explicit water meter low and high cases.
3. The 70% rule: configuration must fit in 70% of the data volume so the disk ladder stays unused.
4. 100,000 tags is a server-class (T2 or central) target, not an edge-box target, for v1.
5. The first lever for storage reduction is deadbands and `max_interval`, not hardware or compression.

## Alternatives rejected and why
- **Sizing from the deployment doc's 50 B and 10x to 20x**: optimistic for irregular on-change data, and an undersized disk is the failure that is hardest to fix after commissioning.
- **A single average rate for all tags**: hides that a handful of badly configured tags dominate storage. The flood guard and `doctor`'s top-offender listing handle that.
- **Sizing for 1 s `sampled` history on every tag**: 86,400 rows/day/tag; it is a configuration mistake, not a capacity plan.
- **Including a safety factor silently in each figure**: instead each assumption is visible and the 20% working space and 70% rule are explicit.

## Open questions
- Every constant in section 1 is an estimate until section 10's measurements exist. If `B_cmp` is closer to 4 B (the deployment doc's number), every compressed figure above falls by 4x and the 10,000-tag busy case needs about 285 GB at 70% (a 512 GB drive) instead of 853 GB; if it is 25 B, 10k typical needs about 250 GB at 70% (tight on a 256 GB drive).
- The 80% history-enabled share and the profile periods (300/60/10 s) are assumptions about plants we have not met; the first real project should replace them with measured distributions per tag class.
- A per-tag rollup opt-out and a per-tag `history.guard` are contract questions (tag-model.md `history` fields).
- Whether `aggregate_retention` of 3 years should instead be sized per site from the 70% rule automatically.
- Should the quiet-site rollup be 1 hour instead of 15 minutes (4x smaller) with the half-hour-timezone caveat of 02, or is correctness for all timezones worth the 21 GB?
