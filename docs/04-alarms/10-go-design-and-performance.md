# 10 Go Design and Performance

Module path is a placeholder (`skyranger/...`, contracts README open question). Packages live inside `skyrangerd`; nothing here is a separate process.

## 1. Package layout

```
internal/alarm/
  model/        State, Flags, Definition, Transition, Tri, Severity  (pure types, no deps)
  state/        Next(): the pure transition function and its table tests
  cond/         condition evaluators per alarm type (high, low, equal, ..., rate_of_change)
  engine/       the single-goroutine loop, routing index, timer heap, command handling
  persist/      batching persister, PostgreSQL writer, JSONL spool, replayer
  notify/       outbox dispatcher, channels (smtp, webhook, ntfy, modem, ...), templates, heartbeat
  journal/      row building, queries, retention
  kpi/          aggregator, _system tag publisher
  apihook/      adapters for REST/WS handlers (ack, shelve, list)
```

Dependency rule: `model` imports nothing; `state` and `cond` import only `model`; `engine` imports `state`, `cond`, `model`; `persist`, `notify`, `journal`, `kpi` import `model` and consume `Transition` values. **No package imports `engine` except `apihook` and the daemon wiring**, so the engine can be tested without a database.

## 2. Core types

```go
package model

type Tri uint8

const (
    Unknown Tri = iota // input not usable (quality hold) or warming up
    False
    True
)

type AlarmState string

const (
    Normal          AlarmState = "normal"
    ActiveUnacked   AlarmState = "active_unacked"
    ActiveAcked     AlarmState = "active_acked"
    ClearedUnacked  AlarmState = "cleared_unacked"
)

// State is the persisted, published object (messages.md s5 + 02 s1).
type State struct {
    AlarmID, TagID       uuid.UUID
    Path, Name           string
    Severity             Severity
    Class                Class
    State                AlarmState
    ShelvedUntil         *time.Time
    Suppressed           bool
    SuppressedReason     string
    Chattering           bool
    Held                 bool
    Enabled              bool
    Message              string
    Value                any          // json-serializable sample value
    Quality              tag.Quality  // from the tag-model contract package
    ActiveAt, ClearedAt  *time.Time
    AckedAt              *time.Time
    AckedBy              string
    UpdatedAt            time.Time
    Seq                  uint64       // JSON: string
    ActivationCount      uint64       // JSON: string
}

type Definition struct { // resolved config, after template expansion and reference resolution
    AlarmID, TagID uuid.UUID
    Path, Name     string
    Type           Type
    SetpointConst  *tag.Variant
    SetpointRef    *uuid.UUID
    RefTag         *uuid.UUID
    Deadband       float64
    OnDelay, OffDelay time.Duration
    Window, Per    time.Duration
    QualityMatch   QualityMatch
    Severity       Severity
    Class          Class
    AckRequired    bool
    AckRole        Role
    Shelvable      bool
    MaxShelve      time.Duration
    SuppressWhen   []SuppressCond
    SuppressRelease time.Duration
    Chatter        ChatterCfg
    Console        string
    Routes         []string
    Message        string
    Enabled        bool
}

// Transition is what the engine emits; the persister turns it into DB rows.
type Transition struct {
    Event   EventID           // UUIDv7, becomes the CloudEvent id
    Before  *State            // nil on create
    After   State
    Cause   Cause             // activated, acked, shelved, ...
    Actor   string            // "system:engine", "user:alice"
    Comment string
    TS      time.Time         // sample ts or command time
    CondSince *time.Time      // raw condition start (on-delay visible)
    Setpoint any
    ConfigRevision int
    Notify  []NotifyIntent    // computed from routes by the engine; persister expands to outbox rows
}
```

## 3. The pure transition function

```go
package state

type Input struct {
    Kind    InputKind // CondTrue, CondFalse, Ack, Shelve, Unshelve, ShelveExpired, SuppressOn, ...
    TS      time.Time
    Actor   string
    Comment string
    Until   time.Time // Shelve
    Reason  string    // SuppressOn
}

type Effects struct {
    Journal        bool
    Notify         NotifyKind // none | activate | clear | ack | ...
    CancelTiers    bool
    ArmTiers       bool
    Cause          model.Cause
}

// Next never reads the clock, never does I/O, never panics on any (State, Input).
func Next(s model.State, def *model.Definition, in Input) (model.State, Effects, bool /*changed*/) {
    switch in.Kind {
    case CondTrue:
        switch s.State {
        case model.Normal, model.ClearedUnacked:
            cause := model.CauseActivated
            if s.State == model.ClearedUnacked {
                cause = model.CauseReactivated
            }
            s.State = model.ActiveUnacked
            s.ActiveAt = ptr(in.TS)
            s.ClearedAt, s.AckedAt, s.AckedBy = nil, nil, ""
            s.ActivationCount++
            return bump(s, in.TS), Effects{Journal: true, Notify: NotifyActivate, ArmTiers: true, Cause: cause}, true
        }
        return s, Effects{}, false // active already: no-op
    case Ack:
        switch s.State {
        case model.ActiveUnacked:
            s.State, s.AckedAt, s.AckedBy = model.ActiveAcked, ptr(in.TS), in.Actor
            return bump(s, in.TS), Effects{Journal: true, Notify: NotifyAck, CancelTiers: true, Cause: model.CauseAcked}, true
        case model.ClearedUnacked:
            s.State, s.AckedAt, s.AckedBy = model.Normal, ptr(in.TS), in.Actor
            return bump(s, in.TS), Effects{Journal: true, Cause: model.CauseAcked}, true
        }
        return s, Effects{}, false
    // CondFalse, flag inputs, Disable ...: one case per row of 02 s3
    }
    return s, Effects{}, false
}
```

(`bump` increments `Seq` and sets `UpdatedAt`. Every row of 02 s3 gets one table test.)

## 4. Condition evaluators

```go
package cond

type Sample struct {
    Value   tag.Variant
    Quality tag.Quality
    TS      time.Time
}

type Evaluator interface {
    // Eval returns the raw condition for the sample; it must be a pure function of
    // (Sample, per-alarm memory held in the evaluator). rawActive is the previous raw result.
    Eval(s Sample, ref *Sample, setpoint float64, rawActive bool) Tri
    // Reset clears per-alarm memory (after hold, config change).
    Reset()
}

// New returns the evaluator for a definition's type. Types: high, low, equal, not_equal,
// quality, deviation, rate_of_change.
func New(def *model.Definition) Evaluator

func Usable(t model.Type, q tag.Quality) bool { // 02 s4
    switch q {
    case tag.QualityUncertainLastKnown, tag.QualityUncertainInitialValue:
        return false
    }
    return q.Class() != 2 // bad_*
}
```

`rate_of_change` owns a ring buffer (`[]struct{ts int64; v float64}`, pre-sized by `window / min_interval`, capped 4096); all other evaluators are stateless.

## 5. The engine loop

```go
package engine

type Engine struct {
    in       chan TagUpdate       // from the tag engine; bounded (default 100_000)
    cmds     chan Command         // ack, shelve, config apply, flag inputs
    byTag    map[uuid.UUID][]*rt  // routing index: tag_id -> runtimes whose input or ref or setpoint_ref or suppress tag is it
    heap     timerHeap            // (deadline, alarm, kind) min-heap; one time.Timer for the earliest
    out      chan<- model.Transition // to the persister, bounded; engine blocks here only if persister is far behind
    clock    Clock                // monotonic abstraction; real and fake
    overflow overflowCoalescer
}

func (e *Engine) Run(ctx context.Context) {
    t := e.clock.NewTimer(far)
    for {
        e.armTimer(t)
        select {
        case u := <-e.in:
            e.onSample(u)                // drains up to N more without re-selecting (batch)
        case c := <-e.cmds:
            e.onCommand(c)               // replies on c.Reply after the transition is *queued*; durable ack: see section 7
        case <-t.C():
            e.onTimers(e.clock.Now())    // pops all due entries
        case <-ctx.Done():
            e.shutdown()
            return
        }
    }
}

func (e *Engine) onSample(u TagUpdate) {
    for _, a := range e.byTag[u.TagID] {
        a.apply(e, u)                    // 1 quality gate 2 evaluate 3 deadband 4 on/off-delay via heap 5 state.Next
    }
}
```

Per-alarm runtime:

```go
type rt struct {
    def      *model.Definition
    st       model.State
    ev       cond.Evaluator
    rawActive bool           // previous raw condition (deadband)
    condSince time.Time      // raw condition start
    onTimer, offTimer *timerEntry
    chatter  ring[time.Time] // activation times
    lastQuality tag.Quality
}
```

Rules:
- **One goroutine owns all `rt`s.** No locks. Reads for `GET /alarms` come from `alarm_state` (the DB) or a snapshot map the engine republishes (copy-on-write pointer swap), never from `rt` directly.
- **Commands and samples are serialized** by the same select; ack against an `expected_seq` is a simple comparison inside the loop (05 s1.1).
- **Timers**: a single `time.Timer` set to the earliest heap deadline; on fire pops every due entry. On/off-delay, shelve expiry, suppress release, escalation tiers, chatter hold, and flood windows are all heap entries with a kind and a `generation` number (cancelling = bumping the generation; stale entries are skipped when popped, so cancel is O(1)).
- **Fake clock** in tests drives the whole engine deterministically: `clock.Advance(61*time.Second)`.
- **Backpressure and overflow**: `in` is bounded and the tag engine **never blocks** on it. On a full queue the tag engine's producer calls `overflow.Coalesce(update)`, which keeps only the **latest** update per `tag_id` in a side map that the engine drains when it catches up. Information lost: intermediate values between those samples (edges shorter than the backlog). Visible: `_system/alarms/engine/input_overflow` tag true while coalescing, counter, critical `overflow` system alarm (01 s6). Under normal operation the queue is lossless; coalescing is the explicit degraded mode.
- **Panics**: each `apply` is wrapped in `recover`; a panic in an evaluator holds that alarm (`Held = true`, `cause: engine_error`), logs the stack, increments `_system/alarms/engine/errors`, and the loop continues. One broken alarm never stops the others.

## 6. Persister

```go
type Persister struct {
    in      <-chan model.Transition
    db      *pgxpool.Pool
    spool   *Spool
    flush   time.Duration // 50ms group commit
    maxBatch int          // 500
}
```

Loop: collect transitions until `flush` elapsed or `maxBatch` reached, then one transaction:

```sql
BEGIN;
INSERT INTO alarm_state (...) VALUES (...) ON CONFLICT (alarm_id) DO UPDATE SET ... WHERE alarm_state.seq < EXCLUDED.seq;
INSERT INTO alarm_journal (...) VALUES ...;
INSERT INTO events (...) VALUES ...;
INSERT INTO notification_outbox (...) VALUES ... ON CONFLICT DO NOTHING;
COMMIT;
```

After commit the persister publishes to the post-commit fan-out: WS hub, bus publisher, notifier wake-up, KPI aggregator. The `WHERE alarm_state.seq < EXCLUDED.seq` guard makes replays and retries harmless.

On DB error: write the batch to the spool (fsync), continue; a background replayer retries. See 09 s3.

## 7. Durable ack

The API handler for ack sends a `Command` and waits for the result. To keep "an ack that returned `200` survives a power cut", the result is returned **after the persister commits** (or after the spool fsync), not when the engine queues it:

```go
type Command struct {
    Kind  CommandKind
    Alarm uuid.UUID
    Actor string
    Comment string
    ExpectedSeq *uint64
    Reply chan<- Result       // engine sends Result{Transition, Err}; handler then waits on Transition.Durable
}
// Transition.Durable is a chan struct{} closed by the persister after commit/spool-fsync.
```

Handler timeout 5 s; on timeout the handler returns `504`-style "unknown outcome" and the client retries (idempotent).

## 8. Performance targets and budget

Reference scale for design (not a promise): 1 gateway, 10,000 alarms on 5,000 tags, 20,000 tag updates/s peak, 500 alarm transitions/s in a worst-case plant upset, sustained 5/s typical. Hardware: a small industrial x86 or 4-core ARM box with an SSD (e.g., a NUC class machine). All figures below are **design targets to be validated by benchmark**, not measured results.

| Stage | Target | Reasoning |
|---|---|---|
| Route sample to alarms | `O(alarms per tag)` map lookup, < 100 ns + per-alarm cost | `byTag` is a map keyed by 16-byte UUID; typical 1-3 alarms per tag |
| Evaluate one alarm | < 300 ns, 0 allocations | float compare, a few branches; `rate_of_change` < 1 µs with ring buffer |
| Engine throughput | >= 200,000 evaluations/s on one core | 20,000 updates/s x ~3 alarms = 60,000/s, so 3x headroom |
| Timer heap | O(log n); 10,000 pending entries worst case | one `time.Timer`; pops are batched |
| Engine to persister handoff | < 50 µs | buffered channel, bounded 4096 |
| Commit latency (group commit) | p99 < 100 ms (50 ms window + commit) | PostgreSQL on SSD, batch inserts via `COPY` or multi-row `INSERT` |
| Activation to WS/bus visible | p99 < 250 ms from sample `ts_gateway` | tag engine + engine + commit + fan-out |
| Activation to SMS handed to modem | p99 < 3 s (transmit time is the carrier's) | outbox dispatcher poll `250 ms` plus wake-up channel |
| Memory per alarm runtime | < 600 B + ring buffer for `rate_of_change` | 10,000 alarms < 8 MB |
| Restart recovery (10,000 alarm_state rows) | < 5 s to first evaluation | one `SELECT`, no replay |

Why this is enough: the dominant cost is never evaluation; it is persistence and the flood case. A flood of 500 transitions/s is 25 transitions per 50 ms batch, a single transaction. The engine never waits on the database (bounded handoff plus spool), so alarm **evaluation latency is independent of DB speed**; only durability and announcement are.

Avoiding allocation and GC pauses: update structs passed by value; `Variant` is a tagged struct, not `interface{}` on the hot path (`Value any` only on `Transition`, which is cold); sample-to-alarm fan-out iterates a slice; timers use pooled entries.

### Benchmarks to write first (test harness, docs/09-testing)
1. `BenchmarkEvaluateHigh` (no alloc), `BenchmarkRoute` with 10,000 alarms.
2. `BenchmarkEngineFlood`: 500 transitions/s for 60 s with a real PostgreSQL in testcontainers; assert commit p99 and zero lost transitions (compare journal count with expected).
3. Soak: 24 h simulated with fault injection (stale values, bad quality bursts, chattering signal) at 50x clock speed.
4. Restart: kill -9 the process mid-flood; assert invariants of 09 s2.1.
5. Fuzz `state.Next` and `cond` evaluators with random sequences against the properties in 02 s7.

### Observability of the engine itself
`_system/alarms/engine/{queue_depth, eval_p99_us, commit_p99_ms, errors, input_overflow, spool_bytes}` (08 s3). The engine measures its own loop time per batch with a histogram; no external metrics system.

## Decisions made
1. One goroutine owns all alarm runtime state; commands, samples and timers are serialized in one `select`. No locks.
2. Pure `state.Next` and `cond` packages with no clock or I/O; `engine` is the only place with timers, and it takes an injectable clock.
3. The engine never blocks on the database: bounded handoff to the persister, spool on failure.
4. Input queue is lossless in normal operation; overflow coalesces to latest-per-tag and raises a critical system alarm.
5. Ack replies after durability (commit or spool fsync).
6. Timers are one heap with generation-based cancellation, one `time.Timer`.
7. Panic isolation per alarm; a broken alarm holds itself and raises an engine error counter.
8. Performance claims are targets with named benchmarks, none asserted as measured.

## Alternatives rejected and why
- **Goroutine per alarm**: 10,000 goroutines, locks around shared state, nondeterministic ordering. The single loop is fast enough by two orders of magnitude.
- **Sharded engine loops by tag hash**: premature; needed only if one core cannot keep up (not at this scale). The `byTag` structure shards trivially later, except cross-tag references (deviation, suppress), which is why it is deferred.
- **Evaluating alarms inside the tag engine**: couples two components, makes alarm logic hard to test and failures contagious.
- **Blocking the tag engine when the queue is full**: stalls history and the HMI for the sake of alarms; coalescing with a loud alarm is safer.
- **Reply to ack immediately, persist later**: an operator can believe an alarm is acked and see it reappear after a crash.
- **`interface{}` for all values on the hot path**: allocations and GC pressure with no benefit.

## Open questions
- Where do the shared types `tag.Variant`, `tag.Quality` live (the contract package)? This doc assumes `skyranger/internal/tag`. Needs confirming with the tag engine design.
- Does the tag engine expose a per-update `byTag` subscription or a firehose the alarm engine filters? A subscription by `tag_id` set (updated on config apply) is assumed.
- Is 100,000 the right mailbox size, and should coalescing be per-alarm-aware (for example never coalesce a tag with a `quality` alarm)? Probably not needed; revisit after the flood benchmark.
- PostgreSQL write path: multi-row `INSERT` vs `COPY` for the journal; decide with a benchmark.
- Reference hardware is unspecified in the deployment area.
