# 04 Store-and-Forward: the Disk Spool

Status: draft. Two queues are in scope and they are deliberately built differently:

1. **Gateway to database** (database or container down, restarting, upgrading, disk trouble): a **disk-backed spool** owned by the historian, described here.
2. **Gateway to cloud or central** (uplink down): **the database itself is the queue**, read through persisted cursors (section 8). No second disk queue.

Relation to the contracts: tag-model.md section 5 says "store-and-forward (backfilled historical data) is out of scope for v1" and driver-interface.md section 6 defers "store-and-forward samples (`is_historical` in Sparkplug)". Both are about **devices** that buffer samples while offline and send them later with old timestamps; the skew rule would reject those. The spool here sits **downstream of the tag engine**: it holds rows the engine has already accepted, with their final effective `ts`, on their way to the database. It bypasses nothing in the engine and creates no new path for device-originated backfill. No contract is contradicted.

## 1. Requirements

| # | Requirement |
|---|---|
| R1 | A PostgreSQL outage of hours or days loses no history rows (up to the size cap), and the engine never notices. |
| R2 | A crash or power cut at any instant leaves a spool that opens cleanly, loses at most the unfsynced tail, and never replays garbage. |
| R3 | Replay is idempotent. Any duplicate delivery is harmless (history key `(tag_id, ts)`, events `(time, id)`, alarms `(alarm_id, seq)`). |
| R4 | Disk use is bounded and the behavior at the bound is explicit, counted, and reported as a gap, never silent. |
| R5 | One small library, no external dependency, that other streams can reuse (events, alarm transitions). |
| R6 | Boring: append-only segment files, CRC per frame, atomic cursor file. No embedded database. |

## 2. Layout

```
/var/lib/skyranger/spool/                       (SKYRANGER_SPOOL_DIR, deployment 02-containers)
  historian/                                    stream "history"
    LOCK                                        flock(2), one writer process per directory
    00000000000000000041.seg                    sealed segment (immutable)
    00000000000000000042.seg                    sealed
    00000000000000000043.open                   active segment, appended to
    cursor.replay                               consumer position (atomic replace)
    dead/
      history-20261107-0001.jsonl               human-readable dead letters (section 6)
  events/                                       stream "events" (same library, section 9)
  alarms/                                       stream "alarms"
```

The directory is bind-mounted into the `skyrangerd` container (deployment doc), on the data volume, so it survives container replacement and is included in neither backups nor exports (it is transient by definition).

## 3. File format

All integers little-endian. Checksums are CRC-32C (Castagnoli), which has hardware support on x86-64 and arm64 and is in the Go standard library (`hash/crc32` with the Castagnoli table).

```
segment := header frame* [footer]

header (32 bytes)
   0   4   magic        "SRSP"
   4   2   version      1
   6   2   stream_id    1 history, 2 events, 3 alarms
   8   8   segment_seq  uint64, equals the file name
  16   8   created_ms   unix milliseconds
  24   4   header_crc   CRC-32C of bytes 0..23
  28   4   reserved     zero

frame (16-byte header + payload)
   0   4   sync         A5 5A C3 3C        resynchronization marker
   4   1   kind         1 data, 2 footer
   5   3   reserved
   8   4   length       payload bytes, max 1 MiB
  12   4   crc          CRC-32C over kind || length || payload
  16   n   payload      protobuf

data payload   := Batch { uint32 schema = 1; repeated Sample samples = 2; }       (history)
                  EventBatch / AlarmBatch                                          (other streams)
footer payload := Footer { uint64 samples; int64 min_ts_ms; int64 max_ts_ms; uint32 frames; }
```

- A frame holds a batch (up to `flush_max_samples`, about 1,000 to 5,000 samples), so per-frame overhead (16 bytes) is negligible. A `Sample` in protobuf is about **48 bytes** (16-byte tag id, varint timestamp deltas, value, quality, flags). This is the figure used in 09.
- Sync marker plus length plus CRC: a reader can find the next valid frame after corruption by scanning for the marker, which a bare length-prefixed format cannot do.
- The **footer** is written when a segment is sealed (rolled at `16 MiB` or `1 hour`, whichever first, if non-empty). It carries the time range, so when the oldest segment must be discarded (section 7) the exact gap is known without reading the file. Sealing renames `*.open` to `*.seg` and fsyncs the directory.
- Protobuf, not JSON or a custom binary format: schema evolution through field numbers (`schema` is also bumped for incompatible changes), and the project already ships protobuf for gRPC. The `.proto` for spool payloads is internal and not a public contract.

## 4. Append, durability, and the fsync rule

- Writer API (in `package spool`): `Append(frame) error`, `Sync() error`. The historian calls `Append` per batch while in `spooling` state.
- **Fsync policy:** a background syncer calls `fdatasync` every `100 ms` if there are unsynced frames, or immediately after `1,000` unsynced samples, and always at seal and at graceful stop. A caller may request `SyncNow` for a stream where loss is not acceptable (alarm transitions use it).
- **Creating a file** is followed by an fsync of the directory; so is every rename. Without that, a power cut can leave a segment present in the cache but absent from the directory after reboot.
- **Loss window:** at most `100 ms` of appended frames when the DB is down and power fails; zero for `SyncNow` streams.
- **fsync failure policy (important):** if `fdatasync` returns an error, the kernel may already have dropped the dirty pages and a later `fsync` on the same file can return success without the data being safe (the PostgreSQL community learned this the hard way; PostgreSQL itself now panics on fsync failure). We therefore **never retry fsync on the same file.** The appender keeps all frames since the last *successful* sync in memory (at most 100 ms or 1,000 samples); on error it marks the segment `.bad` (kept for forensics, never replayed past its last confirmed frame), opens a new segment, re-appends the in-memory frames there, and raises `_system/storage/spool_io_errors`. Two consecutive failures put the stream in `shedding` state (08).

## 5. Open and recovery after a crash

On start, before the writer accepts anything:

1. Take `flock` on `LOCK` (a second process, for example a stale container, fails fast).
2. For the `.open` segment (there is at most one): verify the header; scan frames sequentially. At the first invalid frame:
   - If a valid frame follows later in the file (marker plus a valid length and CRC), the damage is **mid-file corruption** (bad block, bit rot): keep the file, record the damaged byte range, continue after it. The skipped range is counted (`spool_corrupt_frames_total`) and reported as an event with the segment seq.
   - Otherwise it is a **torn tail** (power cut during a write): `ftruncate` to the end of the last good frame, `fsync`, continue. This is the normal case and is silent apart from a counter.
3. Seal the recovered `.open` (compute the footer from the recovered frames) so it becomes an ordinary `.seg`.
4. Delete nothing else. Sealed segments are verified lazily during replay (each frame's CRC is checked as it is read; the same skip-to-next-marker logic applies).
5. Read `cursor.replay`: `{segment_seq, offset, crc}`. A missing or CRC-bad cursor means "start from the oldest segment" (safe: replay is idempotent, just slower).

The cursor is written with the atomic pattern: write `cursor.replay.tmp`, `fsync`, `rename` over `cursor.replay`, `fsync` the directory. It is updated after a replayed batch commits, at most once per second. A crash between a batch commit and the cursor update causes a duplicate replay of at most one second of batches, which dedupes (R3).

A sealed segment is deleted only when the cursor has passed its end **and** the replayer has observed the commit of every batch in it. Deletion order is oldest first.

## 6. Replay and dead letters

- **Replayer:** one goroutine, started in `draining` state (03, section 5). Reads frames in segment order, decodes, and calls the same `HistoryStore.Write` as the live path in chunks of at most `max_rows_per_statement`. Throttle: 5,000 rows/s default, yielding to live batches. After the last sealed segment and the open one are drained and the open one is empty, state returns to `direct`.
- **Ordering:** none required for history (key is `(tag_id, ts)`). For the `alarms` stream order matters per alarm: the stream is replayed strictly in append order, and the alarm upsert only applies a transition whose `seq` is greater than `alarm_state.last_seq` (01, section 7).
- **Poison rows:** a batch that fails with a data error is bisected (03, section 6). The offending rows are appended to `dead/history-YYYYMMDD-NNNN.jsonl`, one JSON object per line using the canonical value object plus `reason` and `error` text. Human-readable on purpose: this is what an engineer opens at 2 a.m. Dead letters are capped at 100 MiB per stream (oldest file removed first, counted) and are never replayed automatically. Rows rejected by the timestamp sanity window (03, section 8) go to the same place with `reason: "ts_out_of_window"`.
- **Re-delivery tool:** `skyrangerctl history redeliver <file> [--shift <duration>]` (proposed) reads a dead-letter file and inserts through the normal idempotent path after the operator fixed the cause.
- After a replay touching data older than the rollup refresh window, the replayer triggers `RefreshAggregates` for the range (02, section 5).

## 7. Size caps, thinning, and what is dropped

The cap exists because the spool lives on the same disk as the database, and a spool that eats the disk takes the database down with it.

| Profile | `max_bytes` default | Constraint |
|---|---|---|
| `meter` | 512 MiB | and at most 10% of the data volume |
| `small` | 2 GiB | same |
| `medium` | 4 GiB | same |
| `large` | 8 GiB | same |

Outage capacity at 48 B per sample (arithmetic in 09, section 7):

| Scenario | Rows/day | Spool bytes/day | Cap | Outage the cap holds |
|---|---|---|---|---|
| Water meter, low (5 m3/day) | 21.3 k | 1.0 MB | 512 MiB | about 500 days |
| Water meter, high (100 m3/day) | 182.7 k | 8.8 MB | 512 MiB | about 61 days |
| 10,000 tags, quiet | 2.3 M | 111 MB | 2 GiB | about 19 days |
| 10,000 tags, typical | 11.5 M | 553 MB | 2 GiB | about 3.9 days |
| 10,000 tags, busy | 69.1 M | 3.3 GB | 4 GiB | about 31 hours |
| 100,000 tags, typical | 115 M | 5.5 GB | 8 GiB | about 37 hours |
| 100,000 tags, busy | 691 M | 33 GB | 8 GiB | about 6 hours |

Behavior as the spool fills (percentages of `max_bytes`):

| Level | Behavior |
|---|---|
| < 70% | normal: every row is spooled |
| 70% to 100%: **thinning** | the spool keeps (a) every quality-change row, (b) every keepalive row, (c) at most one other row per tag per `60 s` (the latest in that minute). The rest are dropped and counted (`spool_thinned_total`). This preserves the shape and the state transitions of the data at a small fraction of the bytes, and it is the same philosophy as the driver SDK's coalescing ("down-sampled but never stale or unbounded", driver-interface.md section 6) |
| 100%: **discard oldest** | the oldest sealed segment is deleted (its footer says exactly which time range and how many samples), a gap record `{from, to, samples, reason: "spool_full"}` is kept in memory and in `gaps.jsonl`, and when the database returns an event `skyranger.storage.history_gap.v1` (proposed) is written per gap. The **newest** data is kept because it is what makes the current picture right after recovery; the oldest data is what the operator will least miss. Reports covering the range show the gap flag (10, section 8) |
| write errors or disk shed (08) | `shedding`: `Offer` returns `Dropped`; the count is exported |

Nothing is dropped without a counter, a gap record, and an alarm (`_system/historian/spool_bytes` over 80% is in the default system alarm set).

## 8. The cloud and central queue: the database is the queue

The central-sync design belongs to a later area (08-deployment topology T4 describes the shape). The storage side commits to this mechanism so that no second durable queue has to be built, sized, backed up and debugged:

```sql
CREATE TABLE forward_cursors (
  destination text        NOT NULL,        -- 'central-main'
  stream      text        NOT NULL,        -- 'history_15m' | 'history_raw' | 'events' | 'alarm_journal'
  position    timestamptz NOT NULL,        -- everything with time < position has been acknowledged by the destination
  updated_at  timestamptz NOT NULL DEFAULT now(),
  PRIMARY KEY (destination, stream)
);
```

- The forwarder reads from the database **since `position - overlap`** (default overlap `1h` for raw history and rollups, `10m` for events), sends in batches, and advances `position` only after the destination acknowledges. The destination de-duplicates on `(gateway_id, tag_id, ts)` for raw history, `(gateway_id, tag_id, bucket)` for rollups, and CloudEvents `id` for events.
- The overlap covers rows that were written after the cursor passed their time (spool replay after a database outage, out-of-order history-only samples within an hour). Rows older than the overlap are caught by an explicit `skyrangerctl forward resync --from <time>`.
- The queue is therefore bounded by **database retention** (400 days raw, 3 years rollups), not by a separate cap. An uplink outage of weeks queues nothing extra on disk: the data is already in `history`.
- Default forward set is `history_15m` plus `events` plus `alarm_journal` (small). Raw forwarding is opt-in per destination, because 10,000 tags at the typical rate is 11.5 M rows/day (about 550 MB/day on the wire at the spool's density).
- If the database is down, forwarding pauses (nothing to read); the spool holds the data; after replay the cursor and overlap deliver it.
- Messages to the destination are labeled historical (Sparkplug `is_historical`, or the HTTPS ingest equivalent), which is a decision for the cloud-sync design and compatible with the contracts: `is_historical` is a northbound concern there, not a device concern here.

## 9. Other streams using the library

| Stream | Used by | Durability | Replay | Cap |
|---|---|---|---|---|
| `history` (1) | historian | `100 ms` fsync | idempotent, any order | per profile above |
| `events` (2) | event writer | `SyncNow` | idempotent by `(time, id)` | 64 MiB; at cap the oldest low-severity events are discarded and counted |
| `alarms` (3) | alarm engine | `SyncNow` | strictly ordered, idempotent by `seq` | 64 MiB; at cap the alarm engine keeps state in memory and alarms the condition (alarm area 09 decides) |

The alarm area's draft says "DB down: fsync'd JSONL spool, replayed later, idempotent by event id" (04-alarms README). It is the same design; this library is offered so there is one implementation of torn-tail recovery and fsync discipline rather than two. JSONL is rejected for the machine path (no CRC, no resync, ambiguous torn lines) but kept for dead letters.

Audit and config apply do **not** spool: they fail closed when the database is down (07-security/08 section 4, 05).

## 10. Go API sketch

```go
package spool

type Options struct {
    Dir          string
    Stream       StreamID
    SegmentBytes int64         // 16 MiB
    SegmentAge   time.Duration // 1h
    MaxBytes     int64
    SyncEvery    time.Duration // 100ms
    SyncSamples  int           // 1000
    FS           FS            // real FS, or a fault-injecting FS in tests
}

type Log interface {
    Append(frame []byte, kind uint8, opts ...AppendOption) error // AppendOption: SyncNow()
    NewReader(consumer string) (Reader, error)                    // persisted cursor
    Stats() Stats                                                 // bytes, segments, oldest ts, corrupt frames, gaps
    Close() error
}

type Reader interface {
    Next(ctx context.Context) (frame []byte, pos Pos, err error)  // io.EOF when drained
    Commit(pos Pos) error                                         // advances cursor, may delete segments
}
```

`FS` is an interface over the five syscalls the library uses (`OpenFile`, `Write`, `Sync`, `Rename`, `Truncate`, plus directory sync) so the fault-injection tests (section 11) can fail or tear any of them deterministically.

## 11. Tests (feed [09-testing](../09-testing/04-fault-injection.md))

- **Torn tail at every offset:** for a recorded segment of N frames, truncate to each byte length 0..size, reopen, assert: opens, recovered frames equal the prefix of complete frames, no panic.
- **Bit flips:** flip each bit in a window; assert the damaged frame is skipped, neighbours survive, counters increase.
- **Crash during append and during seal** with a fault-injecting `FS` that stops after k syscalls.
- **fsync error injection:** assert the segment is abandoned, in-memory frames re-appended, no further `fsync` on the bad file.
- **ENOSPC injection** during append, seal, cursor write.
- **Idempotent replay property test:** random batches, random crash points between commit and cursor write, replay, assert the final database equals the union with no duplicates and no loss.
- **Soak:** database stopped for 10 minutes at `typical` rates, restarted, assert zero loss, replay throttle respected, live latency unaffected.
- **Real power cut** (08, section 8): relay-switched supply, 200 cycles.

## Decisions made
1. The gateway-to-database queue is an append-only, segmented, CRC-framed, sync-marked disk spool with a footer per sealed segment and an atomically replaced cursor file.
2. It is only used when the database is unavailable or the writer is behind; there is no write-ahead log in front of a healthy database.
3. Replay is idempotent by the tables' own keys, so the system never needs exactly-once delivery.
4. fsync errors are fatal for the file: abandon the segment, re-append from memory, never retry fsync.
5. At the size cap, thin first (keep state changes and one row per tag per minute), then discard the oldest segment, and always record the gap.
6. The cloud queue is the database plus persisted cursors with an overlap window; there is no second disk queue.
7. The spool library is shared with the events and alarm streams.

## Alternatives rejected and why
- **An embedded KV store as the spool (BoltDB/bbolt, Badger, SQLite, LevelDB)**: each brings its own crash-recovery semantics, compaction, and write amplification. A spool that is append-only and read-once does not need any of it. Fewer moving parts, and corruption behavior is ours and testable.
- **Using PostgreSQL `UNLOGGED` tables or a local second PostgreSQL as the spool**: they exist exactly when PostgreSQL does not; the failure we are protecting against is PostgreSQL being down.
- **Kafka, NATS JetStream, MQTT persistent sessions as the queue**: one more server on the box, and MQTT persistence would put the broker in the history path, which messages.md forbids.
- **JSONL spool**: human-friendly but no checksum, ambiguous torn lines, larger (about 3x). Used only for dead letters.
- **Dropping the newest data when full**: leaves the system with a stale picture after recovery. Dropping the oldest segments and recording the gap is the better trade.
- **A separate cloud queue on disk**: duplicates what the database already is and adds a cap, a policy and a second gap model.
- **Unbounded spool**: would fill the data volume and kill PostgreSQL. Cap plus explicit gap is strictly safer.
- **Exactly-once delivery machinery** (two-phase commit, transactional outbox for history): idempotent keys give the same result at none of the cost.

## Open questions
- Is a 100 ms fsync interval right for the spool on a consumer SSD without power-loss protection? On such a device the interval is moot (the drive's volatile cache lies), which is another reason the hardware guidance requires PLP ([03-packaging-and-hardware.md](../08-deployment/03-packaging-and-hardware.md)).
- Thinning keeps one row per tag per minute: confirm with an integrator that this is acceptable for process tags, and whether `sampled` mode tags should be exempt from thinning (they are already at a deliberate rate).
- Proposed contract additions: events `skyranger.storage.history_gap.v1`, `skyranger.storage.state_changed.v1`; `_system/storage/spool_io_errors`; `skyrangerctl history redeliver`, `skyrangerctl forward resync`.
- Sparkplug `is_historical` semantics for the northbound forward (cloud-sync design): UNVERIFIED how Ignition and other hosts treat historical metrics.
- Should the alarm engine adopt this library now, or only converge on it if its own spool shows problems? Needs an owner decision from the alarm area.
