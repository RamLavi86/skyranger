# Performance Targets, Memory Budget, Failure Modes

Status: design draft v0.1. Inputs: [pipeline.md](pipeline.md), [derived.md](derived.md), [subscriptions.md](subscriptions.md), [go-design.md](go-design.md), [driver-interface.md](../00-contracts/driver-interface.md) section 6.

**Honesty note.** No benchmark has been run. Every per-operation cost below is an engineering estimate used to size the design and is marked **(est.)**. They are to be replaced by measured numbers from the benchmarks named in go-design.md section 6; until then, treat them as UNVERIFIED. The targets are requirements, the estimates are the reasoning that says the design should meet them.

## 1. Targets and reasoning

Reference box (an assumption, not a purchase decision): 4 cores, 4 GB RAM, SSD, x86-64 or arm64 mini PC, running `skyrangerd`, PostgreSQL/TimescaleDB, Mosquitto and the drivers in containers.

| # | Target | Value | Reasoning |
|---|---|---|---|
| T1 | Configured tags | 50,000 | registry.md sizes lookups for 50k. A single-site SCADA with a few PLCs rarely exceeds 20k; 50k leaves headroom without designing for a cloud |
| T2 | Sustained ingest | 10,000 samples/s, of which at most 5,000/s are accepted changes | A 1 s poll of 10k points is 10k samples/s; most analog values change, many digital ones do not. pipeline.md's garbage estimate (under 1 MB/s) corresponds to about 5,000 changes/s x ~120 B |
| T3 | Burst ingest | 50,000 samples/s for 10 s | Driver reconnect (full re-read of every item) after an outage; absorbed by credits and coalescing, may be delayed, must not drop quality transitions |
| T4 | Sample to store latency | p99 under 5 ms at T2 | Dominated by queueing behind a batch of up to 1,000 samples at roughly a microsecond each (est.) |
| T5 | Sample to WebSocket delivery | p99 under 400 ms at T2 | 100 ms WS batch (api.md) + `min_interval` default 0 + JSON encoding and network. Not a control loop |
| T6 | Physical event to dashboard (water meter) | under 2 s typical | Device sends within 1 s (profile) + under 10 ms engine + 100 ms WS batch + render |
| T7 | Derived evaluation | Full dirty set (all derived tags) under 20 ms at 5,000 derived tags | About 1 to 4 microseconds per built-in evaluation (est.), so 5,000 is 5 to 20 ms |
| T8 | Registry swap | Under 50 ms at 50k tags, ingest keeps queuing | O(tags) pointer work and map construction (est.) |
| T9 | Cold start to first value served | Under 15 s with 50k tags | Compile and restore dominate; driver connect is separate |
| T10 | Engine loop busy ratio at T2 | Under 20% of one core | Leaves 5x headroom before the loop saturates; alert at 60% |
| T11 | Checkpoint | Derived state at most 1 s old; commit under 100 ms for 1,000 dirty rows | One transaction, one upsert statement; PostgreSQL on SSD (est.) |

**Why a single goroutine is enough.** Per accepted sample the pipeline does a handle lookup, a conversion, a scale (one multiply-add), a range check, a change comparison, one 120 B allocation, an atomic store, a loop over dependents and a few non-blocking hand-offs. That is on the order of 0.5 to 2 microseconds (est.), about 0.5 to 2 million samples/s per core, roughly 100 times T2. The design buys correctness (single writer, no locks, deterministic order) with headroom we do not need to spend. If measurement contradicts this by more than 10x, the escape hatch is sharding the loop by connection with derived tags confined to one shard; it is not designed here because it is not needed (Alternatives).

**Milestone 1 scale** is about 10 tags per meter and a handful of meters (tens of samples per second). Nothing here is stressed; the targets exist so that the structure does not need to change later.

**Garbage and GC.** Allocation per accepted change is one `tag.Value` (~120 B incl. strings pointing at registry-owned path, est.). At T2 that is under 1 MB/s, trivial for Go's collector. The path string is shared from the registry, not copied. JSON encoding allocates on consumer goroutines, not in the loop.

## 2. Memory budget

Per tag (est.), 50k tags:

| Item | Bytes per tag | Notes |
|---|---|---|
| `TagDef` incl. strings (path ~60 B, address ~60 B, display name) | 500 to 800 | Shared by registry snapshots in flight; two registries coexist briefly during a swap |
| `slot` (atomic pointer, lastRaw `Variant` ~80 B, flags, timers) | 250 | `Variant` as defined in the contract has all six fields inline (about 80 B) |
| Current `tag.Value` | 120 | |
| `lastPropagated`, `lastHistory` values | 0 to 240 | Only for tags that changed; share pointers with `cur` when equal |
| Index entries (`ByPath`, `ByID`, `sorted`, `ByHandle`) | 150 | Map overhead dominates |
| Subscription mailbox entries | 0 to 40 per subscribing consumer | Bounded by subscribed tags |
| Derived function state | 100 to 20,000 | `rate` buffer: up to ~1,000 points x 16 B (merge rule, derived.md 6.2) |
| **Total** | **about 1.2 to 1.6 KB** | **50k tags: 60 to 80 MB; transient 2x registry during swap adds about 30 MB** |

Other budget lines: bus mirror (10,000 msgs buffer x ~300 B = 3 MB); history sink buffer (100,000 x ~150 B = 15 MB); alarm ring (65,536 x 16 B = 1 MB); per WebSocket client up to 1 MiB send queue (20 connections per user x say 10 users = 200 MiB worst case is **the** dominant risk; limit total send-queue memory, Open questions). Target: engine plus sinks under 250 MB RSS at T1 with typical clients. A Go memory limit (`GOMEMLIMIT`) set to 70% of the container limit is advised (the knob exists in Go 1.19 and later; verify exact behavior before relying on it, UNVERIFIED).

## 3. Failure modes and responses

Principle: **the engine degrades by marking quality and counting, never by guessing, and never by blocking.** Every row names the signal an operator sees.

| # | Failure | Detection | Engine response | Visible as |
|---|---|---|---|---|
| F1 | Driver process crash | Driver host (gRPC stream break, health) | Host reports `disconnected`; all tags of the connection get overlay `bad_not_connected`, last values held (pipeline.md section 8); driver respawned with backoff, same handles | Tag quality; `_system/connections/{c}/state`, `restarts`; `connection.state_changed` event |
| F2 | Driver hangs (alive, not delivering) | `stale_after` on tags; `Health` timeouts (driver contract) | Tags with `stale_after` go `uncertain_last_known`; host kills after 3 failed health checks | Quality, `liveness` tags |
| F3 | MQTT broker down (Sparkplug source) | Sparkplug driver reports `disconnected` | As F1. Counts continue on devices (cumulative); on recovery the next sample carries them; no `counter_total` loss | `bad_not_connected` |
| F4 | Device dead, network up | NDEATH (Will) or heartbeat missing | Tags `uncertain_last_known`; `liveness` 0 or 1; `rate` freezes (derived.md 6.2) | Water meter: `health`, quality |
| F5 | Device reboot | `boot_count` change | `counter_total` reset path; event; total continues | `counter_discontinuity` |
| F6 | Counter wraps | `raw < last_raw`, `modulus > 0` | Rollover path; event | event |
| F7 | Implausible jump | `delta > max_delta` | Delta ignored, `last_raw` updated, quality `uncertain` for that sample, event | event |
| F8 | PostgreSQL unavailable | Checkpoint/history writer errors | History spills to its WAL (historian doc); checkpoints retried, latest state wins, engine keeps running; memory and `counter_total` **writes** fail with `internal`; config apply fails. A crash during the outage loses derived state back to the last good checkpoint, which for `counter_total` is harmless without a reboot (derived.md section 7) | `_system/engine/checkpoint_errors`, `checkpoint_lag`; `/system/health` degraded |
| F9 | History sink full | `Offer` returns false | Count `history_dropped`, force next row for that tag; never block | `_system/engine/history_dropped` |
| F10 | Alarm ring full | `Notify` false | Overflow flag; alarm engine re-reads current state from the store (subscriptions.md section 6) | `_system/engine/alarm_resyncs` |
| F11 | Slow WebSocket client | Send queue over 1 MiB | `resync` with snapshot (api.md); repeated overflow closes the session | Client sees `resync`; `ws_overflows` (API metric) |
| F12 | Engine overloaded | Loop busy ratio, intake credits exhausted | Back-pressure to drivers, which coalesce latest-wins (quality transitions preserved); history down-sampled, nothing stale or unbounded | `loop_busy_ratio`, driver `dropped_samples` |
| F13 | Panic in a derived function | `recover` in `evalOne` | Tag set to quality `bad` (128), value held, `derived_panics++`, error logged with tag path and input values; after 3 panics in 60 s the function is disabled until the next config revision (`bad_config_error`). The loop continues | `_system/engine/derived_panics`; quality |
| F14 | Panic elsewhere in core | process crash | Supervisor restarts; startup restores state; driver processes restart too | Container restart count |
| F15 | Gateway wall-clock step | Wall vs monotonic delta over 1 s | Counted and logged; device-timestamped samples keep the order rule, gateway-stamped are accepted in arrival order; `rate`/`liveness`/`continuous_activity` use monotonic receipt time so are unaffected | `_system/engine/clock_steps` |
| F16 | Gateway has no NTP and a wrong clock | Large skew rejections on all devices | Devices' timestamps rejected (`ts = ts_gateway`), history is internally consistent but wrongly dated | `clock_skew_rejections` per connection |
| F17 | Device without time sync reports 1970 | skew rule | `ts = ts_gateway`, counter incremented | `clock_skew_rejections` |
| F18 | Duplicate or replayed delivery (QoS 1 redelivery, rebirth) | Unchanged value | Unchanged repeats are no-ops; `counter_total` delta is zero for equal raw | none (by design) |
| F19 | Samples for unknown handle | Lookup miss | Dropped and counted (a late sample after unsubscribe) | `unknown_handle_samples` |
| F20 | Bad config applied (valid but wrong, for example wrong `multiplier`) | Operator | Revision N+1 applies; restore of revision N is a draft + apply; `counter_total` keeps state across both (registry.md section 9); history recorded under each revision | config revision events |
| F21 | Checkpoint row for unknown fn/version at startup | Load | Discard that tag's state with a log and event; tag restarts at `initial_total` | event, log |
| F22 | Memory pressure | RSS vs `GOMEMLIMIT`, mailbox sizes | Bounded structures only: nothing grows with message rate. Remaining growth is per configured tag and per connected client | RSS; refuse new WS subscriptions at a cap |
| F23 | Registry swap fails to compile | Compiler errors | Revision rejected before the engine sees it; engine keeps running the old registry | API `422` with pointers |
| F24 | Driver subscribe fails for some items on swap | Per-item result | Those tags `bad_config_error` in the new revision; swap proceeds | quality; `_system/connections/{c}/last_error` |
| F25 | Event or write-completion queue full | `Emit` false | Count `events_dropped`, log body; `write_completed` recoverable from `tag_writes` | `_system/engine/events_dropped` |

## 4. Engine self-monitoring tags (`_system/engine/...`)

Created by the compiler in every registry (registry.md section 8), updated by the engine every 5 s through the `stats` timer (a direct store write that bypasses pipeline stages 1 to 3). `kind` is system (reported as `source` until the contract adds `system`, registry.md Open questions). History: `sampled` at `1m` for gauges, `on_change` with `max_interval: 1h` for counters.

| Path | Type | Unit | Meaning |
|---|---|---|---|
| `_system/engine/config_revision` | int64 | | Active revision |
| `_system/engine/tags` | int64 | `1` | Tags in registry (by kind in sibling tags `tags_source`, `tags_derived`, `tags_memory`) |
| `_system/engine/samples_per_sec` | float64 | `1/s` | Accepted samples (incl. unchanged repeats), 10 s mean |
| `_system/engine/changes_per_sec` | float64 | `1/s` | Accepted changes |
| `_system/engine/loop_busy_ratio` | float64 | `1` | Busy time over wall time, 10 s |
| `_system/engine/batch_latency_p99` | float64 | `s` | Intake enqueue to applied (UCUM `s`) |
| `_system/engine/flush_duration_p99` | float64 | `s` | Derived flush time |
| `_system/engine/unknown_handle_samples` | uint64 | `1` | Counter |
| `_system/engine/clock_steps` | uint64 | `1` | Counter |
| `_system/engine/derived_panics` | uint64 | `1` | Counter |
| `_system/engine/history_dropped` | uint64 | `1` | Counter |
| `_system/engine/alarm_resyncs` | uint64 | `1` | Counter |
| `_system/engine/events_dropped` | uint64 | `1` | Counter |
| `_system/engine/checkpoint_lag` | float64 | `s` | Age of oldest unflushed derived state |
| `_system/engine/checkpoint_errors` | uint64 | `1` | Counter |
| `_system/engine/subscriptions` | int64 | `1` | Active subscriptions |
| `_system/engine/write_pending` | int64 | `1` | Writes in flight |

The names are an engine-owned list; a rename is allowed between versions (contract: `_system` tags may change). Alarms on `loop_busy_ratio` and `checkpoint_lag` are suggested defaults shipped as a template (Open questions).

## 5. How the targets get verified

1. `core` benchmarks (go-design.md section 6) on the reference box class; record real numbers here.
2. Soak test with the simulator: 20,000 tags, 10,000 samples/s with `burst` fault injection, 24 h; assert zero growth in RSS after warm-up, `history_dropped == 0` at nominal load.
3. Fault suite: each row F1 to F25 has a scenario or fault-hook test (go-design.md section 6) that asserts the response and the visible signal.
4. A 30 s reconnect storm (T3) with the sink blocked, asserting that quality transitions survive coalescing (driver contract conformance).

## Decisions made
1. Targets T1 to T11, with a single-goroutine core judged adequate by roughly 100x headroom, to be verified by benchmark, not assumed.
2. Memory is bounded by configured tags and connected clients, never by message rate.
3. Every queue in the system is bounded and has a defined, counted overflow behavior.
4. Failures are expressed as quality plus counters and events; the engine does not guess values or retry writes.
5. A derived-function panic degrades one tag and is contained; a core panic restarts the process.
6. Self-monitoring is exposed as ordinary `_system/engine` tags, so the same HMI, history and alarm machinery watch the engine.

## Alternatives rejected and why
- **Sharding the engine loop now**: complexity (cross-shard derived tags) for a 100x headroom problem we do not have.
- **Lock-free queues and custom allocators for the hot path**: premature; the allocation rate is tiny.
- **Dropping samples under load**: history gaps with no signal; coalescing at the driver preserves the latest state and quality transitions.
- **Unbounded buffering to avoid back-pressure**: converts an overload into an OOM kill.
- **A separate metrics system (Prometheus) as the only self-monitoring**: plants without a Prometheus still need to see the engine's health; Prometheus export can be added from the same tags.
- **Crash on any derived-function panic**: one bad user config would take the whole plant view down.

## Open questions
- All **(est.)** numbers are unmeasured. First action when code exists: run the benchmarks and rewrite sections 1 and 2 with measured values.
- Total WebSocket send-queue memory cap across clients (per-connection 1 MiB is in the contract, the sum is not). Proposal: global cap of 128 MiB, new subscriptions refused beyond it.
- `GOMEMLIMIT` semantics and recommended percentage: verify against Go documentation (UNVERIFIED).
- Default alarms for engine health tags: ship as a template or leave to the integrator.
- Is 50,000 tags the right ceiling for the reference box, or should the claim be 20,000 until soak-tested?
- Percentile gauges (`batch_latency_p99`) need a streaming quantile or a fixed-bucket histogram; choose the boring fixed-bucket one when implementing.
