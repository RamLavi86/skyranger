# Go Design: Packages, Types, Concurrency, Lifecycle

Status: design draft v0.1. Binding inputs: [driver-interface.md](../00-contracts/driver-interface.md) (Sink, backpressure), [tag-model.md](../00-contracts/tag-model.md) section 3 (`Value`, `Variant`, `Quality`), README.md in the contracts for naming. Module path is a placeholder (`skyranger/...`), as in the contracts.

## 1. Package layout

```
skyranger/
  pkg/tag/                 public: DataType, Variant, Value, Quality (contract types)
  pkg/driver/              public: driver.Driver, Sink, Sample (contract)
  internal/tagengine/
    registry/              Registry, TagDef, compiler, template expansion, diff   (registry.md)
    derived/               Spec, Func, built-ins (counter_total, rate, ...)      (derived.md)
    subs/                  Hub, Subscription, Mailbox                            (subscriptions.md)
    checkpoint/            derived_state, memory_values, tag_writes, outbox writer (PostgreSQL)
    clock/                 Clock interface, real and fake implementations
    core/                  State machine: store, pipeline, dirty set, timers, writes. No goroutines.
    engine/                Runner: goroutines, channels, intake credits, wiring to sinks
    enginetest/            Harness: fake clock, fake sinks, scenario runner, golden files
```

Dependency direction (no cycles, enforced with `go vet` plus a `depguard`-style lint rule, UNVERIFIED tool choice):

```
pkg/tag  <-  derived  <-  registry  <-  core  <-  engine
                  ^                      ^   ^       |
              (Spec only)       subs ────┘   └─ clock, sinks (interfaces)
checkpoint implements core's StateStore interface; engine wires it.
```

- `core` is the heart and is **single-threaded by construction**: it has no goroutines, channels, mutexes (apart from `subs.Mailbox` and atomic pointers) or `time.Now()`. Every input is a method call (`HandleBatch`, `HandleConnState`, `HandleTimers`, `HandleWrite`, `ApplyRegistry`), every output goes through an interface.
- `engine` is a thin shell: it owns the goroutines and channels and calls `core` methods one at a time. This split exists for testing (section 6): the full pipeline, derived evaluation and timers run in a plain unit test with a fake clock, no sleeping, no data races to reason about.
- `pkg/tag` additions needed (additive, see Open questions): class constants `ClassGood=0`, `ClassUncertain=1`, `ClassBad=2` next to `Quality.Class()`.

## 2. Key types and interfaces

Types for the registry and slots are in registry.md section 2 and pipeline.md section 1. Additional ones:

```go
package core

// Consumer-defined interfaces (Go idiom): core depends on these, others implement them.

type HistorySink interface {
    // Offer must not block. false = dropped (caller counts it).
    Offer(e HistoryEntry) bool
}
type HistoryEntry struct {
    TagID  uuid.UUID
    V      *tag.Value
    Reason HistoryReason // Change | MaxInterval | Sample | OutOfOrder
}

type AlarmSink interface {
    Notify(idx registry.TagIndex, v *tag.Value) bool // false = ring full; core sets Overflow
    Overflow() *atomic.Bool
}

type EventSink interface{ Emit(ev tagevent.Event) bool } // non-blocking

type StateStore interface { // implemented by checkpoint
    Load(ctx context.Context) (LoadedState, error) // memory values, derived state rows
    Submit(b CheckpointBatch) bool                  // non-blocking, false = writer busy
    FlushNow(ctx context.Context, b CheckpointBatch) error
}

type DriverHost interface { // implemented by the driver host
    Subscribe(conn string, items []driver.Item) ([]driver.ItemResult, error)
    Unsubscribe(conn string, handles []uint32) error
    Write(ctx context.Context, conn string, w driver.WriteRequest) (driver.WriteResult, error)
}

type Clock interface {
    Now() time.Time                    // wall + monotonic
    NewTimer(d time.Duration) Timer    // fake clock advances them deterministically
}

type Core struct { /* registry snapshot, slots, dirty buckets, timer heap, subs hub, sinks, clock */ }

func (c *Core) HandleBatch(conn string, b driver.SampleBatch, gw time.Time)
func (c *Core) HandleConnState(conn string, s driver.ConnState, err *driver.Error, at time.Time)
func (c *Core) HandleTimers(now time.Time)            // runs everything due
func (c *Core) NextTimer() (time.Time, bool)
func (c *Core) ApplyRegistry(next *registry.Registry, diff registry.Diff) // section 4
func (c *Core) Write(cmd writeCmd) writeAccepted       // validation; side effects started via interfaces
func (c *Core) Flush(now time.Time)                    // derived dirty set, called at the end of each Handle*
func (c *Core) Snapshot() *Snapshot                    // atomic read view for any goroutine
```

**Published snapshot.** Readers need the registry and the values to match. Slots are heap objects shared across revisions so that a reader holding an older snapshot still sees live values for tags that survive a revision:

```go
type Snapshot struct {
    Reg   *registry.Registry
    Slots []*slot            // index = TagIndex of Reg; the same *slot pointers carry over between revisions
}
// Core.snap atomic.Pointer[Snapshot]; replaced only in ApplyRegistry.
```

This refines pipeline.md section 1 (`slots []slot`) to `[]*slot`. Carry-over becomes copying pointers, and a removed tag's slot simply stops being updated. Each slot also keeps its `idx` (engine-private, updated at swap) and a `removed` flag so timers holding a `*slot` after a swap can tell.

`Value.Path` in a stored `*tag.Value` is the path at store time. After a rename, stored values keep the old path until the next change; `ApplyRegistry` re-stamps the current value of each renamed tag with the new path (same value, quality, `ts`) so subscribers never see the old path after the swap.

## 3. Concurrency model

### Goroutines

| Goroutine | Count | Owns | Talks to |
|---|---|---|---|
| **Engine loop** | 1 | All mutable tag state (`core`) | Everything below, via channels and interfaces |
| Driver sink callers | 1+ per connection (gRPC stream receiver) | Nothing | Engine `inbox` (credits) |
| Driver write dispatch | 1 per in-flight source write | One write | Driver host, then `writeDone` to loop |
| Checkpoint writer | 1 | PostgreSQL for derived state, memory values, outbox | Engine (`Submit`), loop (completion commands) |
| Historian writer | 1+ (historian doc) | History | `HistorySink` queue |
| Alarm engine | 1+ (alarm doc) | Alarm state machines | `AlarmSink` ring |
| Bus mirror | 1 | MQTT client | `subs.Subscription` |
| WebSocket hub per session | 1-2 per client (API) | Socket | `subs.Subscription` |
| REST handlers | per request | Nothing | `Snapshot()` for reads; `post(cmd)` for writes and drains |

Rule: **only the engine loop mutates `core`**. Everything else either reads through `Snapshot` (atomic pointers, immutable values) or posts a command.

### Intake and the loop

```go
type Engine struct {
    core   *core.Core
    clock  clock.Clock
    inbox  chan intake      // sample batches from all connections; capacity 256
    cmds   chan command     // priority: writes, subscribe, registry swap, writeDone, shutdown; capacity 1024
    credit map[string]*semaphore.Weighted // per connection, 64 (driver-interface.md section 6)
}

// Called by the driver host's Sink.Values for connection conn. Blocks (back-pressure to the gRPC stream) when the connection has used its 64 credits.
func (e *Engine) Values(ctx context.Context, conn string, b driver.SampleBatch, gw time.Time) error {
    if err := e.credit[conn].Acquire(ctx, 1); err != nil { return err }
    select {
    case e.inbox <- intake{conn, b, gw}: return nil
    case <-ctx.Done(): e.credit[conn].Release(1); return ctx.Err()
    }
}

func (e *Engine) loop() {
    for {
        // Priority: drain commands before taking more samples. One item per iteration so neither starves.
        select {
        case c := <-e.cmds: e.apply(c); e.core.Flush(e.clock.Now()); continue
        default:
        }
        timerC := e.nextTimerChan()
        select {
        case c := <-e.cmds:  e.apply(c)
        case in := <-e.inbox: e.core.HandleBatch(in.conn, in.batch, in.gw); e.credit[in.conn].Release(1)
        case <-timerC:       e.core.HandleTimers(e.clock.Now())
        }
        e.core.Flush(e.clock.Now())
    }
}
```

- **Per-connection bounded queue as credits, one shared FIFO channel**: a connection can have at most 64 batches outstanding anywhere (in the channel or being applied), which is the README's "per-connection bounded queues (64)" without a `reflect.Select` over a dynamic number of channels. The shared FIFO gives fairness: a chatty connection cannot get more than 64 positions ahead of anyone else.
- `ts_gateway` is stamped by the host at receive (driver-interface.md), before the credit wait, so queueing delay does not alter timestamps.
- **No blocking calls in the loop**: PostgreSQL, gRPC, MQTT and file I/O all happen on other goroutines. The loop's blocking primitives are channel receives and mailbox mutexes held for pointer swaps.
- **Commands carry their own reply path** (a buffered channel of 1, or a callback executed on the caller's goroutine) so the loop never waits for the replier.
- **Panic policy**: derived `Eval` calls are wrapped in `recover` inside `evalOne` (performance-and-failures.md section 3). A panic elsewhere in `core` crashes the process on purpose: the supervisor restarts it, state is checkpointed, and a half-applied pipeline state is worse than a restart.

### Timers

One min-heap of `timerEntry{due time.Time, kind, slot *slot, gen uint32}` and **one** `clock.Timer` armed for the earliest entry. Kinds: `eval` (derived `eval_interval`), `stale`, `historyMax`, `historySampled`, `checkpoint` (1 s), `stats` (5 s, `_system/engine` tags).

- **Lazy deadlines** for `stale` and `historyMax`: a sample does not touch the heap. The entry fires at the old deadline, checks `lastSampleMono + stale_after` against now, and re-arms for the remainder if a sample arrived in between. The per-sample cost is therefore O(1) (a timestamp store), not O(log n). At 5,000 samples/s this removes 5,000 heap operations per second for nothing.
- **Monotonic time**: deadlines and durations use `time.Time` values from `Clock.Now()` and `Sub`, which use the monotonic reading, so an NTP step cannot make a tag stale or fresh. Wall time is used only for output `ts`. Wall-clock steps are detected by comparing wall and monotonic deltas between iterations (pipeline.md section 4).
- `gen` is bumped when a slot's timers are reset (config change); stale heap entries are skipped on pop (lazy deletion).
- `sampled` history ticks are scheduled on wall-clock multiples of `interval` (pipeline.md section 7) using wall time to find the next multiple, then monotonic for the wait.
- When the loop is behind, `HandleTimers` processes everything due in one go and fixed-rate-with-skip scheduling (derived.md section 5) prevents a replay storm.

### Memory ordering and visibility

`slot.cur` is an `atomic.Pointer[tag.Value]`; Go's sync/atomic operations are sequentially consistent, so a reader that loads a pointer sees a fully initialized `Value`. Multi-tag reads are **not** a consistent cut: a REST `GET /tags/values?prefix=` can return `total_volume` from one flush and `flow_rate` from the next. Callers needing consistency compare `ts`. This is documented in the API layer; if a consistent cut ever matters, the loop can publish a generation counter that readers check before and after.

## 4. Applying a new registry (hot reload)

The compiler runs on the config service's goroutine and produces `(*Registry, Diff)`. Applying it is one command, executed in the loop, bounded in time because it only moves pointers and arms timers:

1. **Driver side first, off the loop**: for added or rebound source tags the engine host computes `Subscribe` and `Unsubscribe` item lists from the diff (handles are already allocated by the compiler, registry.md section 7). The driver calls happen **before** the swap command is posted for additions (so values do not arrive for handles the registry does not know; they would be dropped and counted) and **after** it for removals (so late samples still resolve). If a driver call fails per-item, those tags get `bad_config_error` in the new revision; the swap is not blocked.
2. **In the loop**, `ApplyRegistry`:
   a. build the new `[]*slot`: reuse the `*slot` for every tag_id present in both revisions, create slots for added tags (value `bad_waiting_for_initial_data`, or restored state);
   b. for each change class (registry.md section 9) perform its state action: re-derive current value from `lastRaw` for `processing` changes; reset for `source rebind`/`datatype`; call `Func.Rebaseline()` for derived input changes; `Restore` function state for re-added ids; mark removed slots `removed`;
   c. rebuild `ByHandle`, `Dependents`, per-rank dirty buckets, fan-out index, and re-expand subscriptions (subscriptions.md section 3);
   d. re-arm timers (bump `gen` for changed slots);
   e. mark every new or changed derived tag dirty with `TrigLoad` and run `Flush`;
   f. store the new `Snapshot` with one `atomic.Store`.
3. Post-swap notifications: `TopicsToClear` to the bus mirror, `config_changed` to the API hub, the alarm engine's new definitions (`AlarmSink.Reconfigure`), and the `revision_applied` event is emitted by the config service, not the engine.

Time bound: the swap is O(tags). At 50,000 tags it is a few milliseconds of pointer work plus map construction (estimate, UNVERIFIED); the intake keeps queuing during it, covered by credits.

## 5. Startup and shutdown

### Startup (order matters)

1. Load the current config revision from PostgreSQL; compile it (same compiler as drafts). Failure here is fatal and loud; the previous revision number is logged. Opening the database is a hard dependency of `skyrangerd` itself (contract decision: PostgreSQL holds configuration).
2. `StateStore.Load`: memory values, derived state rows, `system_tags` ids.
3. Build `Core`: slots with restored memory values (`good_manual`) and restored function state; derived tags get a `TrigLoad` evaluation (derived.md section 7).
4. Start sinks' consumers first: historian writer, alarm engine (it takes its initial state from the store's current values, then follows `AlarmSink`), checkpoint writer, bus mirror.
5. Start the engine loop.
6. Start the driver host: spawn driver processes, `Configure`, `Connect`, `Subscribe` all items with the handles from the registry. Samples start arriving.
7. Start the API listener last. `/system/health` reports `degraded` until step 6 completes.

Why this order: the loop must exist before any driver can deliver; sinks must exist before the loop can offer; the API must not serve a snapshot of tags the engine has not initialized.

### Shutdown (SIGTERM; budget 10 s total, UNVERIFIED that the container runtime default stop timeout is 10 s: configure it explicitly)

1. API stops accepting new connections; WebSocket sessions are closed with `1000` (api.md close codes). In-flight writes are cancelled and completed as unknown-outcome (writes.md section 3).
2. Driver host `Close`s all drivers (5 s each in parallel per driver-interface.md). No new samples after this.
3. Engine drains `inbox`, runs a last `Flush` and `HandleTimers`.
4. `StateStore.FlushNow` with all dirty derived state and memory values (limit 3 s).
5. Historian and alarm sinks flush (historian doc).
6. Bus mirror publishes `status` with `online: false` and disconnects cleanly; the Will covers the unclean case.
7. Exit 0. Anything left past the budget is abandoned: the 1 s checkpoint cadence means at most 1 s of derived state is lost, which for `counter_total` loses nothing (derived.md section 7).

## 6. Testing hooks

- **Deterministic core tests**: `enginetest.New(t, registryYAML)` builds a `Core` with a fake `Clock`, in-memory sinks that record, an in-memory `StateStore` and `DriverHost`. A test feeds `HandleBatch`, advances the clock (`h.Advance(5*time.Minute)` runs `HandleTimers` in due order), and asserts on recorded values, events and checkpointed state. No `time.Sleep`.
- **Scenario files**: `testdata/*.scenario.yaml` describe timed device messages and expected tag values and events. `water-meter-e2e.md` is written so each of its tables becomes one scenario file; the sim driver's `pulse_meter` generator produces the same sequences for system tests.
- **Golden files** for the compiler (registry.md section 1): `testdata/{name}/input/*.yaml` to `expected/registry.json` and `diff.json`.
- **Property tests** (`testing/quick` or Go native fuzzing): `counter_total` against a reference model. For any sequence of device events (count steps, wraps at `modulus`, reboots with epoch +1, gateway restarts with checkpoint restore, duplicated and reordered-by-one batches), the total equals initial plus pulses counted by the model minus declared lost pulses.
- **Fault hooks**: `core` accepts a `FaultInjector` interface in tests only (drop checkpoint, fail `FlushNow`, panic in `Eval`, full history sink) to prove the failure rows in performance-and-failures.md section 3.
- **Race detector**: all `engine` tests run with `-race` in CI; `core` tests also run without goroutines, so there is nothing to race.
- **Benchmarks**: `BenchmarkHandleBatch` (1,000 samples, 10% changed, derived fan-out 1:3) and `BenchmarkFlushWorstCase` back the targets in performance-and-failures.md section 1. Results go in that file as measured numbers when they exist.

## Decisions made
1. Split into a goroutine-free `core` and a thin `engine` shell, so the entire data path is deterministic under test.
2. One engine goroutine mutates state; everyone else reads atomic pointers or posts commands.
3. Per-connection back-pressure by credits over one shared FIFO channel, instead of a dynamic multi-channel select.
4. Slots are shared `*slot` across revisions; a `Snapshot{Reg, Slots}` is swapped atomically.
5. Lazy timer deadlines so a sample costs O(1); one heap, one OS timer.
6. Panics inside derived functions are recovered per evaluation; panics elsewhere crash and restart on purpose.
7. Registry swap runs in the loop as pointer work only; driver subscribe calls happen outside it, additions before and removals after.
8. Startup order: store, sinks, loop, drivers, API; shutdown reverses it with a bounded budget.

## Alternatives rejected and why
- **Actor per tag or goroutine per connection mutating shared state under locks**: ordering across derived tags becomes the hard part, and locks reappear on the hottest path.
- **`sync.RWMutex` around one big map**: contended by every reader, no benefit from single-writer knowledge.
- **Channel per connection with `reflect.Select`**: slow and awkward with a changing connection set.
- **Timer per tag (`time.AfterFunc`)**: 50k runtime timers and a goroutine hop each, for a job one heap does.
- **Rebuilding the store as a new slice of values on each revision**: breaks readers holding atomic pointers and costs a copy of every tag.
- **Running the compiler in the loop**: a large revision could stall ingest; compile is pure and runs elsewhere.
- **Unbounded command channel**: a stuck consumer could grow memory silently; commands are bounded and callers see back-pressure.

## Open questions
- `pkg/tag` additions: `ClassGood/ClassUncertain/ClassBad` constants (additive to the contract, which only shows `Class() uint16`).
- Whether `core` should be exported from `internal` for the scripting runtime's tests. Probably not.
- Lint tooling for package dependency rules: choose when CI is set up (UNVERIFIED which tool).
- Consistent multi-tag reads (generation counter) are deferred; confirm the HMI does not need them.
- Credit size (64) is the driver contract's number; the shared channel capacity (256) and command capacity (1024) are guesses to be validated by the benchmarks.
- Shutdown budget (10 s) versus the container stop timeout: a deployment doc decision.
