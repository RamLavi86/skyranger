# Derived Tags: Functions, Dependency Graph, Evaluation, State

Status: design draft v0.1. Binding inputs: [tag-model.md](../00-contracts/tag-model.md) sections 2, 4, 7 and 11; [messages.md](../00-contracts/messages.md) sections 4 and 6. Pipeline context: [pipeline.md](pipeline.md). Compile-time checks that call into this file: [registry.md](registry.md) section 6.

## 1. Expression language decision

**v1 has no expression language.** The contract fixes a closed set of Go built-ins (tag-model.md section 7). This file does not reopen that. What it adds is a seam: every built-in is an implementation of one Go interface (section 2), and nothing in the pipeline, registry or persistence knows which functions exist. A future `fn: expr` or `kind: script` is therefore a new registration, not an engine change.

When the question is reopened, the candidates and the bar they must clear are:

| Requirement | Why |
|---|---|
| Not Turing-complete, or at least guaranteed to terminate (step or time budget) | An evaluation runs on the engine goroutine; an infinite loop stops the plant view |
| Static type check at config apply time | A bad expression must fail the draft, not log errors at 3 a.m. |
| No I/O, no clock, no randomness except what `Ctx` passes in | Deterministic tests with a fake clock |
| Allocation-light | Runs on the hot path |

Candidates to evaluate then (all UNVERIFIED as to current maturity, licence and performance; we have not benchmarked any): `expr-lang/expr`, `google/cel-go` (CEL), `go.starlark.net` (Starlark). Python scripting (decision 2 in decisions.md) stays out of process and talks to the REST API; it is not a candidate for the hot path.

Even with an expression language, `counter_total`, `rate`, `liveness` and `continuous_activity` stay Go: they hold state and have restart semantics that must be tested, not user-written.

## 2. The `Func` interface

```go
package derived // skyranger/internal/tagengine/derived

// Spec is the static description of a built-in. The registry compiler uses it for
// validation (input names, types, params, eval_interval); the engine uses it for
// scheduling. One Spec per fn name, registered at init.
type Spec struct {
    Name          string
    Inputs        []InputSpec   // ordered; declaration order is the quality tie-break order
    Output        func(in []tag.DataType) (tag.DataType, error) // output type, or error if inputs mismatch
    NewFunc       func(p Params, out tag.DataType) (Func, error) // strict param decoding; unknown param = error
    NeedsTimer    bool          // rate, liveness, continuous_activity: eval_interval required
    Writable      bool          // only counter_total
    QualityMode   QMode
    StateVersion  int           // bumped when the persisted state layout changes
}

type InputSpec struct {
    Name     string
    Optional bool              // epoch on counter_total
    Types    []tag.DataType    // accepted types; empty = any
    OnSample bool              // also evaluate on unchanged repeats (section 4)
}

type QMode uint8
const (
    QDegrade  QMode = iota // engine: worst input quality, then Degrade(worst, Output.Q). Default.
    QAbsolute              // Output.Q is final (liveness: always good)
    QFromInput             // quality of input Output.Pick (select)
)

type Ctx struct {
    Now     time.Time  // injected clock; carries a monotonic reading in production
    Trigger Trigger    // TrigInput | TrigTimer | TrigLoad | TrigWrite (bit set)
    TagID   uuid.UUID
    Path    string
}

type Input struct {
    Name       string
    Bound      bool          // false for an unbound optional input
    V          tag.Variant
    Q          tag.Quality
    TS         time.Time     // effective ts of the current value
    ChangedAt  time.Time     // gateway receive time of the last accepted change (monotonic where available)
    SampledAt  time.Time     // gateway receive time of the last accepted sample, changed or not
    Changed    bool          // value or quality changed since this function last ran
    Sampled    bool          // a sample was accepted since this function last ran (superset of Changed; only for OnSample inputs)
    Generation uint64        // bumped by the producing function on a discontinuity it declares (manual_adjust)
}

type Output struct {
    V        tag.Variant
    Q        tag.Quality    // 0 = no opinion
    Pick     int            // QFromInput: index into inputs
    Hold     bool           // no new value; keep the previous output as is
    Bump     bool           // increment this tag's Generation (consumers drop history-derived buffers)
    Events   []Discontinuity
    StateDirty bool         // persisted state changed; engine schedules a checkpoint
}

type Func interface {
    Eval(ctx Ctx, in []Input) Output
    // State returns the serialized persistent state, or nil for stateless functions.
    State() []byte
    Restore(state []byte) error
    // Rebaseline is called when an input is re-pointed to a different tag_id (registry.md section 9).
    Rebaseline()
}

// WritableFunc is implemented only by counter_total.
type WritableFunc interface {
    Func
    Write(ctx Ctx, v tag.Variant) (Output, error)
}

type Discontinuity struct {
    Kind         string // reset | rollover | implausible | manual_adjust (contract); rebaseline pending contract change
    PrevRaw, NewRaw        *uint64
    PrevEpoch, NewEpoch    *tag.Variant
    DeltaApplied float64
    TotalAfter   float64
}
```

Rules for implementers (enforced by review and by a `derived/conformance` test helper):
- `Eval` is pure with respect to `Ctx` and its own state: no I/O, no goroutines, no `time.Now()`, no locks, no logging on the hot path.
- A bad-quality input never mutates persisted state (contract rule in the `counter_total` algorithm); the function returns `Hold` or its current value.
- `Eval` is idempotent for the same inputs: running twice with no new input changes the output by nothing. This lets the engine evaluate defensively (after a registry swap, at load).
- A function never improves quality (pipeline.md section 3). `Degrade` is applied by the engine, so a function cannot get this wrong by forgetting.
- Panics are recovered by the engine per evaluation (performance-and-failures.md section 3), not by functions.

The engine, not the function, chooses the output `ts`: the latest `TS` among inputs with `Changed` (pipeline.md section 4), or `Ctx.Now` for timer-only runs. The engine, not the function, applies `eng_range`, change detection, history and propagation to the output.

## 3. Dependency graph, ranks, cycle detection

Computed by the compiler (registry.md section 1, step 7). Inputs: every derived tag's resolved input `TagIndex`es. Output: `Registry.Dependents` (input to dependent derived tags), `TagDef.Rank`, `Registry.MaxRank`.

```go
// Rank: source, memory, system tags are rank 0. A derived tag is 1 + max(rank of its inputs).
// Iterative DFS with white/grey/black colouring; on meeting a grey node, the grey stack
// slice from that node is the cycle, reported as "a -> b -> c -> a" with paths.
func rank(r *Registry) (maxRank int, errs []ValidationError)
```

- **Cycle detection**: DFS over edges `dependent -> input`. Grey-node hit means a cycle; the error message lists the path (registry.md: code `cycle`). A self reference (`inputs: {x: total_volume}` on `total_volume`) is the one-node cycle. All cycles are reported, not just the first. Cross-instance absolute references (`/site1/other/thing`) are in the same graph, so a cycle across two template instances is caught the same way.
- **Depth limit** 32 (`too_deep`): bounds the work per cycle and the length of any pathological chain. The water meter has depth 2.
- **Why cycles are never allowed, even with a timer in the loop**: a feedback loop between derived tags has no defined evaluation order and would make results depend on timer phase. A user who needs feedback (a PID, a latch) is a scripting case, not a derived tag.
- **Evaluation order inside a rank** is ascending `TagIndex`; `TagIndex` is assigned in sorted-path order (registry.md section 2), so it is the same on every restart. Order inside a rank cannot affect results, because tags in the same rank do not read each other; fixing it anyway makes logs and tests reproducible.

Water meter ranks: raw tags 0; `total_volume` 1 (reads `raw/pulse_count`, `raw/boot_count`); `health` 1; `flow_rate` 2; `leak` 2.

## 4. Evaluation: dirty set, batch level, glitch-free

The engine keeps, per registry, `dirty [][]*slot` (one bucket per rank, 1..MaxRank) and a `queued` flag on each slot to dedupe.

```go
// markDirty is called by pipeline stage 9 (store) and, for OnSample inputs, by stage 8 on unchanged repeats.
func (e *Engine) markDependents(s *slot, repeatOnly bool) {
    for _, d := range e.reg.Dependents[s.idx] {
        ds := e.slots[d]
        if repeatOnly && !ds.fnWantsSample(s) { continue }
        if !ds.queued { ds.queued = true; e.dirty[ds.def.Rank-1] = append(e.dirty[ds.def.Rank-1], ds) }
        ds.pendingTrigger |= TrigInput
    }
}

// flush runs at the end of every loop iteration that applied anything
// (a sample batch, a connection state change, a write, a registry swap, timers).
func (e *Engine) flush(now time.Time) {
    for r := 0; r < e.reg.MaxRank; r++ {          // rank 1 first
        bucket := e.dirty[r]; e.dirty[r] = e.dirty[r][:0]
        for _, s := range bucket {
            s.queued = false
            e.evalOne(s, now)                       // reads inputs from the store, may mark higher ranks dirty
        }
    }
}
```

`evalOne` builds `[]Input` from the store (each input's `*tag.Value` plus the engine-private change/sample times), calls `Eval`, merges quality (`QMode`), selects `ts`, then runs the output through pipeline stages 4, 8, 9, 10, 11. If the output changed it marks its own dependents, which are in a strictly higher rank, so the loop above visits them later in the same `flush`.

**Glitch-freedom.** A derived tag at rank r runs once per `flush`, after every input of rank below r has been finalized for this iteration. Consequences:
- Diamond: `a` feeds `b` and `c`, both feed `d`. `d` runs once, after `b` and `c`, never between them. No transient value computed from new `b` and old `c` is ever stored, propagated or written to history.
- Same-message dependencies: NBIRTH delivers `boot_count` and `pulse_count` in one `SampleBatch` (the Sparkplug driver must put all metrics of one MQTT message in one batch; see Open questions). `counter_total` runs once with both new, which is what the reset rule needs.

**What is not guaranteed:** consistency across batches. A derived tag may see `boot_count` from batch N and `pulse_count` from batch N-1. `counter_total` is written to tolerate exactly this (section 6, pending epoch), because the transport cannot promise otherwise.

**Triggers.** A function runs when (a) a declared input changed, (b) an input declared `OnSample` received any accepted sample, (c) its `eval_interval` timer fired, (d) at load or after a registry swap that created or changed it (`TrigLoad`, once), (e) a write (`TrigWrite`). Trigger bits accumulate until the run, so an input change and a timer firing in one iteration cause one evaluation.

**OnSample inputs** (this is the one addition to pipeline.md stage 8: an unchanged repeat that stops there still calls `markDependents(s, repeatOnly=true)`). Used by `counter_total` (`count`) and `liveness` (`heartbeat`). Cost: one cheap evaluation per heartbeat per meter.

**Eval cost bound**: each tag evaluates at most once per `flush`; a `flush` evaluates at most the dirty tags. The worst case is every derived tag dirty, bounded by `derived_count` evaluations at about a microsecond each (an estimate; UNVERIFIED until benchmarked, see performance-and-failures.md section 1).

## 5. Timers and `eval_interval`

- `eval_interval` is required for `rate`, `liveness`, `continuous_activity` (`Spec.NeedsTimer`; error `eval_interval_required`). Optional for others (e.g. a `select` that should react to quality changes needs none, because quality changes are input changes).
- Why these three need it: their output depends on time passing with no input event. `rate` decays when pulses stop; `liveness` becomes `stale` when heartbeats stop; `continuous_activity` becomes false when gaps exceed `max_gap`.
- Timers live in the engine's timer heap (go-design.md section 3). Scheduling is **fixed-rate with skip**: `next = prev_due + interval`, and if that is already in the past, `next = now + interval`. A stalled loop therefore never replays a backlog of missed evaluations.
- Timer evaluations set output `ts` and `ts_gateway` to the evaluation time and omit `ts_source` (contract).
- Phase: the first timer is armed at load time plus a per-tag offset `hash(tag_id) % interval`, so 200 meters do not all evaluate in the same millisecond. Per-tag offsets do not change results; they flatten CPU spikes.
- Granularity is the interval: `continuous_activity` with `eval_interval: 30s` can raise the leak up to 30 s after `duration` elapses. That is acceptable for a 2 h duration; a function that needs exact boundaries would request a wake time (not in v1; Open questions).

## 6. The six built-ins

### 6.1 `counter_total`

Spec: inputs `count` (uint64 or int64), `epoch` (optional, any); output float64; `OnSample` on `count`; writable; `QDegrade`. The normative algorithm is tag-model.md section 7. This is its implementation shape, including two points the contract leaves to the engine.

```go
type counterState struct {
    HasState     bool         `json:"has_state"`
    LastRaw      uint64       `json:"last_raw,string"`
    LastEpoch    *tag.Variant `json:"last_epoch"`      // nil if epoch unbound
    Total        float64      `json:"total"`
    PendingEpoch *tag.Variant `json:"pending_epoch"`   // epoch changed, its count sample not seen yet
}

func (c *counterTotal) Eval(ctx Ctx, in []Input) Output {
    cnt, ep := in[0], in[1]
    // 1. Validity. Bad class on count, or on a bound epoch: hold, do not touch state.
    if cnt.Q.Class() == tag.ClassBad || (ep.Bound && ep.Q.Class() == tag.ClassBad) {
        return Output{V: tag.Float(c.st.Total), Hold: !c.st.HasState}
    }
    raw := cnt.V.Uint()                       // int64 input: negative is bad_type_mismatch upstream
    // 2. First sample ever: baseline, emit initial_total.
    if !c.st.HasState {
        c.st = counterState{HasState: true, LastRaw: raw, LastEpoch: epochPtr(ep), Total: c.p.InitialTotal}
        return Output{V: tag.Float(c.st.Total), StateDirty: true}
    }
    // 3. Epoch handling (section 6.1.1).
    epochChanged := ep.Bound && !variantEqual(*c.st.LastEpoch, ep.V)
    if epochChanged || c.st.PendingEpoch != nil {
        if !cnt.Sampled {                     // epoch arrived before its count sample
            e := ep.V; c.st.PendingEpoch = &e
            return Output{V: tag.Float(c.st.Total), Q: tag.QualityUncertain, StateDirty: true}
        }
        return c.applyReset(raw, ep)          // delta = raw (zero) or 0 (first_sample); event reset
    }
    if !cnt.Sampled { return Output{V: tag.Float(c.st.Total)} } // timer or load run, nothing new
    // 4. Same epoch: normal delta, rollover, backwards, plausibility (contract algorithm).
    ...
}
```

Rules, all additive to the contract algorithm:

**6.1.1 Pending epoch.** The contract algorithm assumes `epoch` and `count` change together. The batch-level flush gives that for one Sparkplug message. If they arrive in different batches the two orders both break a naive implementation: epoch first would add the old count as a reset delta and then the new count again; count first would apply the backwards rule and then the epoch rule, double-counting `raw`. So: when `epoch` changes and no `count` sample has been accepted in this run, the function records `PendingEpoch`, holds the total, and degrades quality to `uncertain`. The next accepted `count` sample (changed or not, hence `OnSample`) is treated as the first post-reboot count and applies exactly one reset delta. An unchanged count after an epoch change is valid and common (a device with count 0 reboots; 0 to 0): that is why `OnSample`, not `Changed`, is the trigger.

**6.1.2 Backwards without epoch.** With `epoch` unbound, or bound and unchanged, `raw < last_raw` is a rollover if `modulus > 0` and a reset otherwise (contract). If `modulus > 0` and the device really rebooted without an epoch input, the "rollover" delta is about `modulus` and `max_delta` rejects it (`implausible`). That is why the template sets both. Binding `epoch` is the only unambiguous configuration.

**6.1.3 Arithmetic.** `total` is float64 accumulating `delta * multiplier`. Integer-valued totals are exact to 2^53 (about 9e15 L). With a fractional multiplier each addition carries relative error of about 1e-16; over 1e9 additions that is on the order of 1e-7 relative (a reasoned bound, not measured). The raw history is the audit source if anyone disputes a total.

**6.1.4 Writes.** `Write(v)`: `total = v`, `PendingEpoch = nil`, event `manual_adjust` with `delta_applied = v - old_total` (signed) and `prev_raw == new_raw == last_raw`; `Bump = true`; the engine publishes the value with quality `good_manual` for this emission when the computed quality would be `good`, and `good` again from the next input-driven run. The write is acknowledged only after a synchronous checkpoint (writes.md section 5).

**6.1.5 Registry carry-over** is in registry.md section 9: params change keeps state; input re-point calls `Rebaseline()` which clears `LastRaw`, `LastEpoch`, `PendingEpoch` but keeps `Total`, and the next sample sets the baseline with delta 0. `initial_total` applies only when `HasState` is false.

Events produced are `skyranger.tag.counter_discontinuity.v1` payloads (messages.md section 4). They are not emitted directly; they go through the checkpoint outbox (section 7) so an event exists if and only if the state change that caused it was committed.

### 6.2 `rate`

Spec: input `total` (numeric); params `window` (5m), `per` (1m), `min_samples` (2); output float64; needs timer; `QDegrade`.

State: a buffer of `(t, total)` points, not persisted (a restart means a fresh warm-up; honest and cheap). `t` is the gateway receive time (`ChangedAt` or `Ctx.Now`), **not** the device `ts`, so a wall-clock step cannot corrupt a window and device clock skew is irrelevant over minutes.

```text
on Eval(now, total):
  if total.Generation != gen:  buf.clear(); gen = total.Generation      # manual_adjust: history-derived data is invalid
  frozen := total.Q.Class() == bad || total.Q == uncertain_last_known    # we cannot see the source
  if frozen:  mark gapStart if not set; return Hold (value unchanged; engine degrades quality from input)
  if gapStart set:  gapStart = nil          # recovery: no step point, so the delta since the last live point is smeared over the gap
  else if total.Changed and buf non-empty:  append (ChangedAt - 1ms, previous total)   # was flat until the change
  append (ChangedAt if Changed else now, total.V) when Changed, or when last point older than eval_interval/2
  merge points closer than window/1024; prune points older than now-window except the newest one at or before it
  ws := now - window
  if buf has a point at or before ws:   a := interpolate(buf, ws);          quality = none
  else:                                 a := buf[0].total; span := now - buf[0].t
        if len(buf) < min_samples or span < 1s:  return 0.0, uncertain_initial_value
        quality = uncertain_initial_value        # window not yet filled: rate over the span we have
  rate := (total.V - a) / span_or_window * per ; clamp to >= 0
```

Properties:
- Linear interpolation between points, flat after the newest point. While the source is live the buffer gets a flat point at least every `eval_interval/2`, so "no pulses for 15 minutes" is known, not interpolated away.
- **Freezing during an outage** (value held, quality inherited as `uncertain_last_known` or bad) rather than decaying: a decay toward zero while the device is unreachable would present a computed "0" as information we do not have.
- **Recovery spreads the catch-up.** Device counted 600 L while the broker was down, rebirth delivers `+600` in one step: the linear segment from the last live point to now spreads it over the outage, so the flow shows the average over the gap instead of a 120 L/min spike for 5 minutes. For the contract trace (3 L after a 2 minute outage) this gives 0.6 L/min, quality `good`, as the contract table shows.
- Resolution: one pulse per `window` is `per/window` units, i.e. 0.2 L/min at 1 L per pulse and a 5 min window. The contract's open question (time-between-pulses mode for low flows) stays open; `continuous_activity` exists because of this.
- A `Bump` from the input (manual_adjust) clears the buffer; output goes back to `uncertain_initial_value` for up to `window`.

### 6.3 `liveness`

Spec: inputs `heartbeat` (any, `OnSample`), `online` (bool); params `stale_after`; output int64; needs timer; `QAbsolute` (always `good`).

```text
online.Q.Class() == bad || online.V == false   -> 0   offline
now - heartbeat.SampledAt > stale_after          -> 1   stale   (also if heartbeat.Q.Class() == bad)
otherwise                                         -> 2   ok
```

Superset of the contract: the contract says the heartbeat must change every period; using `SampledAt` (last accepted sample, changed or not) also works if a heartbeat value repeats, for example `uptime_s` equal before and after a reboot. At startup `SampledAt` is the load time, so a device that never reports is `offline` (online input bad) immediately, and `stale` is only reachable if `online` is true. State: none.

### 6.4 `continuous_activity`

Spec: input `total` (numeric); params `max_gap`, `duration`; output bool; needs timer; `QDegrade`.

State (persisted, wall-clock times): `last_increase`, `active_since` (both nullable).

```text
on Eval(now, total):
  if total.Generation changed: ignore the step (manual_adjust is not activity); update baseline
  frozen (bad class or uncertain_last_known): return Hold
  if total increased since last run:                 # Changed and value > previous
      if last_increase == nil or ChangedAt - last_increase > max_gap:  active_since = ChangedAt
      last_increase = ChangedAt
  if last_increase != nil and now - last_increase > max_gap:  active_since = nil     # gap broke the streak
  output = active_since != nil and now - active_since >= duration
```

- "Increased" compares against the previous value it saw, kept in memory; after a restart the first run only establishes it, so a restart is never counted as activity. Because the times are persisted, a gateway restart shorter than `max_gap` does not reset a leak timer that is already running.
- A catch-up step after an outage counts as one increase at recovery time. If the outage exceeded `max_gap`, the streak restarts; if not, it continues. Both are conservative: the function never claims continuity it did not observe.
- Slow leak example: one litre every 10 minutes, `max_gap` 15 min, `duration` 2 h. `rate` reads about 0 or 0.2 L/min; `continuous_activity` becomes true two hours after the first drip, up to `eval_interval` late.

### 6.5 `select`

Inputs `a`, `b` of the same type (validated); output that type; `QFromInput`. Output `a` if `a.Q` is good class, else `b`; quality is the picked input's. If both are bad, picks `a` (first in declaration order, matching the tie-break rule).

### 6.6 `scale`

Input `x` (numeric); params as tag-model.md section 6 `scale`; output float64; stateless; `QDegrade`. Reuses `Scale.Apply` from pipeline.md section 5, so a derived `scale` and a source `scale` can never round differently. Uses the same `clamp` and invertibility rules, except that `scale` is not writable.

## 7. State, checkpointing, restore

Which functions persist: `counter_total` (above) and `continuous_activity`. `rate`, `liveness`, `select`, `scale` are rebuilt from live inputs.

```sql
CREATE TABLE derived_state (
    tag_id        uuid PRIMARY KEY,
    fn            text        NOT NULL,
    state_version int         NOT NULL,
    state         jsonb       NOT NULL,
    updated_at    timestamptz NOT NULL,
    orphaned_at   timestamptz            -- set when the tag leaves the registry (registry.md section 9)
);
```

Checkpoint flow (go-design.md section 3 for the goroutines):
1. After `Eval`, `StateDirty` puts the slot on the engine's checkpoint set. Nothing is serialized per sample.
2. A 1 s engine timer takes the set, calls `State()` on each (in the loop, since functions are not goroutine-safe), and sends one `CheckpointBatch{states, outboxEvents}` to the checkpoint writer. Channel capacity 4. If the writer is behind (PostgreSQL slow), the engine keeps tags dirty and sends the latest state next tick; a backlog never queues.
3. The writer commits a batch in **one transaction**: upsert into `derived_state`, insert the batch's discontinuity events into `events` (**transactional outbox**). After commit it publishes the events to the bus. Result: a `counter_discontinuity` event exists if and only if the state that produced it is durable. Without this, a crash between event and checkpoint would replay the same reset after restart and emit a second event with a different UUIDv7 `id`.
4. `FlushNow(ctx)` is a synchronous variant used by writes (writes.md section 5) and at shutdown.

Why losing up to 1 s of checkpoints loses nothing for `counter_total`: the checkpoint is a consistent pair `(last_raw, total)`. Restoring an older pair and then applying the device's current cumulative raw yields the correct total, as long as no reboot or double wrap happened in between (contract reasoning). A reboot inside the lost window loses the counts of that window, which the contract already accepts for any reboot.

Restore at startup (before the loop runs): for every derived tag with a row, check `fn` and `state_version`; mismatch means discard with a log line and an event, not a crash. Valid rows go to `Func.Restore`. Then a `TrigLoad` evaluation runs once. Inputs are `bad_waiting_for_initial_data`, so the output is the restored value (held) with the inherited bad quality: a dashboard shows the last known total, greyed, until the device reports. No special-casing is needed because "bad quality carries the last value" already is the rule (tag-model.md section 3).

Orphan handling: on tag removal the row is kept with `orphaned_at` set (registry.md section 9); proposal to purge with history retention is in registry.md Open questions.

## Decisions made
1. No expression language in v1; built-ins sit behind a `Spec` + `Func` seam so adding one later touches only registration.
2. Ranks and a per-rank dirty set give glitch-free, once-per-flush evaluation; cycles and depth over 32 are compile errors with the cycle path reported.
3. The engine owns quality merge, `ts`, `eng_range`, change detection and propagation for derived outputs; functions only compute value, an optional quality opinion and events.
4. `OnSample` input triggers so `counter_total` and `liveness` see unchanged repeats; this lets reboot-to-same-count and repeated heartbeats be handled correctly.
5. `counter_total` tolerates epoch and count arriving in different batches via a persisted pending-epoch state, so a transport reordering cannot double count.
6. `rate`, `liveness`, `continuous_activity` use gateway receive times, never device timestamps, so device clock error and wall-clock steps cannot distort them.
7. `rate` freezes during outages and interpolates across them, instead of decaying to zero or spiking on catch-up.
8. State is checkpointed as one consistent row per tag, at most every 1 s, in a transaction with its discontinuity events (outbox).
9. Per-tag timer phase offsets and fixed-rate-with-skip scheduling flatten CPU spikes and prevent backlog replay.

## Alternatives rejected and why
- **Evaluate each derived tag immediately when an input changes (depth-first propagation)**: glitches in diamonds and double evaluation of `counter_total` on a two-input NBIRTH.
- **Topological sort at runtime per change**: the order is static per registry, so compute it once at compile.
- **Persist the whole `rate` buffer**: restart warm-up is honest and the buffer is derived from live data; persisting adds write volume and a stale-window hazard.
- **`rate` decaying toward zero while the source is unreachable**: invents information.
- **Step-function `rate` with a buffer reset on recovery**: contradicts the contract trace (flow `good` right after rebirth) and throws away valid history; the interpolation rule keeps the trace and removes the spike.
- **Device timestamps for `rate`/`liveness`**: one unsynced ESP32 clock would corrupt the window; the skew rule already protects history but not time-window logic.
- **Emitting discontinuity events directly from `Eval`**: duplicates after a crash; the outbox removes it.
- **Per-function goroutines**: ordering and snapshot consistency become the hard part for no gain at this scale.

## Open questions
- CONTRACT: `rebaseline` (input re-point) has no `counter_discontinuity.kind` (already raised in registry.md); also needed for the `PendingEpoch` hold, which is silent.
- CONTRACT: the contract's `liveness` says the heartbeat "must change every heartbeat"; this design relaxes it to "any accepted sample". Wording could follow.
- CONTRACT: `rate` interpolation rule and freeze-on-outage are not in tag-model.md section 7; add a sentence so other implementers (a future Python port, tests) match.
- CONTRACT: `epoch_wait`-style behavior is engine-internal (hold until the count sample arrives). If an operator needs a timeout, a `counter_total` param is needed.
- The Sparkplug driver must deliver all metrics of one NBIRTH/NDATA message in one `SampleBatch` and the SDK coalescer must not split them (driver-interface.md section 6 coalesces by handle; behavior across metrics of one message is unspecified). Needs a statement in the driver contract; this design is safe either way but is cleaner with it.
- Sums, averages and arithmetic across tags (for example "total of all meters in the pumphouse") are not in the six built-ins. Variable-arity inputs are not expressible with the current fixed `inputs` map. Proposal (additive): `sum`, `min`, `max`, `avg` over an input list, quality by the worst-input rule.
- A function wake-time request (`Output.WakeAt`) for exact duration boundaries is not in v1.
- Should `continuous_activity` count a post-outage catch-up step as activity at all?
