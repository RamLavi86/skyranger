# Value Store and Update Pipeline

Status: design draft v0.1. Binding inputs: [tag-model.md](../00-contracts/tag-model.md) sections 3 to 6; [driver-interface.md](../00-contracts/driver-interface.md) sections 2, 3, 6.

## 1. Value store

The store is a dense slice indexed by `TagIndex` (registry.md section 2). Each slot holds the published current value behind an atomic pointer, plus engine-private state that only the engine goroutine touches.

```go
package tagengine

type slot struct {
    // Published. Written only by the engine goroutine, read by anyone.
    cur atomic.Pointer[tag.Value]   // never nil after load; *Value is immutable once stored

    // Engine-private (no locks, single writer; see go-design.md section 3).
    lastRaw        tag.Variant      // last raw sample value (source tags), for re-scaling on config change
    sampleQuality  tag.Quality      // quality the driver or function produced, before stale/conn overlay
    lastSampleMono time.Time        // gateway receive time (monotonic) of last sample, changed or not
    stale          bool
    connOverlay    tag.Quality      // 0 = none, else bad_not_connected / bad_config_error / bad_out_of_service
    lastPropagated *tag.Value       // for propagation deadband
    lastHistory    *tag.Value       // for history deadband and max_interval
    lastHistoryMono time.Time
    staleTimer     timerID
    fn             derived.Func     // derived tags only (derived.md)
    writeInFlight  uuid.UUID        // writes.md section 3
}

type Store struct {
    slots []*slot                   // replaced on each revision; the same *slot pointers carry over (go-design.md section 2)
}
```

Readers (REST `GET /tags/values`, WS snapshots, the alarm engine's initial load) do `s.slots[i].cur.Load()` against the registry snapshot they took. Each `*tag.Value` is allocated fresh per change and never mutated, so a reader can never see a half-written value. The cost is one small allocation per change (about 120 bytes); at the targets in performance-and-failures.md that is under 1 MB/s of garbage, which Go's GC handles easily.

Initial value of every tag at load: value `null`, quality `bad_waiting_for_initial_data`, `ts = ts_gateway = load time`, unless restored (memory tags and derived state, section 9 and derived.md section 7).

## 2. Source sample pipeline

Input: a `SampleBatch` from the driver host for one connection, already stamped with one `ts_gateway` (driver-interface.md: the host stamps at receive). For each `Sample`, the engine goroutine runs these stages in order:

| # | Stage | Rule |
|---|---|---|
| 1 | Resolve | `ByHandle[conn][handle]` to `TagIndex`. Unknown handle: drop, count `_system/engine/unknown_handle_samples`. Tag disabled: drop. |
| 2 | Convert | If `Variant.Type` is not the tag's raw type, try an **exact widening** conversion (int64 to float64 only if `|v| <= 2^53`; uint64 to int64 only if `v <= MaxInt64`; any int to uint64 only if non-negative). Failure: quality `bad_type_mismatch`, raw value not updated. Drivers should already deliver the requested type; this is defense in depth. |
| 3 | Scale | Section 5. Produces the engineering value. NaN or Inf result: quality `bad_type_mismatch`, last good engineering value held (see Open questions). |
| 4 | Range | If `eng_range` set and value outside `[low, high]`: quality becomes `uncertain_eng_range_exceeded` if it was good class. `range` scale with `clamp: true` clamps first, so a clamped value is not flagged. |
| 5 | Timestamp | Section 4 selects `ts`; `ts_source` and `ts_gateway` kept as received/stamped. |
| 6 | Overlay | Clear `stale`, re-arm stale timer, set `lastSampleMono`. Compute effective quality (section 3). |
| 7 | Order | If `ts < cur.TS` **and** `ts` came from the device clock: history-only (go to stage 10 with the sample, skip 8, 9, 11). |
| 8 | Change detect | Changed if value differs (exact, type-aware; floats compared bitwise after scaling) or effective quality differs. Unchanged repeat: stop here (stale timer already reset; history `max_interval` handled by the history timer), except that dependents whose function declared this input `OnSample` are still marked dirty (derived.md section 4). |
| 9 | Store | Build a new `*tag.Value`, `cur.Store()`. Mark every `Dependents[i]` dirty (derived.md section 4). |
| 10 | History gate | Section 7. Non-blocking hand-off to `HistorySink`. |
| 11 | Propagate | Deadband (section 6). If it passes: record `lastPropagated`, enqueue to the fan-out (subscriptions.md). Quality changes always pass. |

After the whole batch is applied, the engine runs derived evaluation for the dirty set (derived.md section 4), whose outputs go through stages 4, 6 (quality only), 8, 9, 10, 11.

Notes:
- **Derived tags read the store, not the propagated stream.** Deadband affects only what leaves the engine. `counter_total` must see every raw count; `rate` must see every total.
- **Bad samples** from a driver carry the driver's last value (driver-interface.md section 4). If the sample value is null, the tag keeps its last engineering value; otherwise the carried raw value is scaled as usual. Either way `lastRaw` is not updated from a bad sample, so derived re-scaling uses the last good raw.
- **Unchanged repeats** do not change `cur` and so do not change `cur.TS`. `cur.TS` therefore means "time of last change". A separate read-only field is not added to the canonical value object (contract), but `_system`-level staleness is visible through quality.

## 3. Quality

### Source tags: effective quality

Evaluated in this order; the first rule that applies wins.

1. Tag disabled, or its connection disabled: `bad_out_of_service`.
2. `connOverlay` set by a connection state change (section 8): that code.
3. Sample quality class bad (from driver or stage 2/3): that code.
4. `stale` set: `uncertain_last_known` (replaces `good` and any uncertain code, because "the source has gone quiet" is the more important fact for an operator).
5. Sample quality uncertain, or range flagged (stage 4): that code.
6. Otherwise the sample quality (`good`).

### Derived tags

Contract rule: worst input quality by class, ties to the first input in declaration order, specific code copied; the function may degrade but never improve.

```go
// WorstQuality returns the code of the first input with the highest class.
func WorstQuality(inputs []tag.Quality) tag.Quality {
    w := inputs[0]
    for _, q := range inputs[1:] {
        if q.Class() > w.Class() { w = q }
    }
    return w
}

// Degrade returns fnQ only if it is strictly worse by class than inQ, or the
// same class while inQ is good. A function never improves quality.
func Degrade(inQ, fnQ tag.Quality) tag.Quality
```

Exceptions defined by the contract and implemented exactly there: `liveness` output is always `good`; `select` picks `a` when `a` is good, otherwise `b`, and takes the chosen input's quality. Optional inputs that are not bound (e.g. `counter_total` without `epoch`) do not take part.

### Memory tags
Operator writes and restored values: `good_manual`. A memory tag with no `initial_value` and no persisted value: `bad_waiting_for_initial_data`.

## 4. Timestamps

```go
func selectTS(src tsSource, sourceTS, gwTS time.Time, maxSkew time.Duration) (ts time.Time, fromDevice bool) {
    switch src {
    case TSGateway:
        return gwTS, false
    case TSDevice:
        if !sourceTS.IsZero() { return sourceTS, true }
        return gwTS, false
    default: // TSAuto
        if !sourceTS.IsZero() && !sourceTS.Before(epoch2020) && absDur(sourceTS.Sub(gwTS)) <= maxSkew {
            return sourceTS, true
        }
        if !sourceTS.IsZero() { skewRejections[conn]++ } // _system/connections/{name}/clock_skew_rejections
        return gwTS, false
    }
}
```

- All times are truncated to milliseconds once, at this stage, so the value in memory equals the value on the wire and in the database. Equality checks and history keys `(tag_id, ts)` then behave identically everywhere.
- **Order rule (stage 7)** applies only when `fromDevice` is true. Gateway-stamped samples are accepted as current in arrival order even if a wall-clock step made `ts` go backwards; otherwise one backward chrony step of 2 s would freeze every gateway-stamped tag for 2 s, and a large step (bad RTC at boot, then NTP sync) could freeze it for hours. A backward step greater than `1s` (detected by comparing wall and monotonic deltas between batches) emits a log line and increments `_system/engine/clock_steps`. See Open questions: this refines the contract rule.
- Derived values take the `ts` (and `ts_source`) of the triggering input sample. When several inputs changed in one batch, the latest `ts` among the changed inputs is used, so a derived `ts` is never earlier than any input it reflects. Timer evaluations use the evaluation time for `ts` and `ts_gateway`, no `ts_source` (contract).
- A history-only (out-of-order) sample of a source tag does **not** trigger derived evaluation. `counter_total` cannot take a count from the past without corrupting `last_raw`. For cumulative counters the next in-order sample carries the information anyway.

## 5. Units and scaling

- `unit` is a label. The engine does no unit conversion in v1; a template converting pulses to liters does it with `multiplier`.
- `scale` requires tag `datatype: float64` in v1 (validation, registry.md section 6). The engine subscribes with `Item.DataType = RawType` (the address's natural type) so the driver does not pre-convert.

```go
func (s *Scale) Apply(raw float64) (eng float64) {
    switch s.Type {
    case ScaleLinear: return raw*s.Multiplier + s.Offset
    case ScaleRange:
        eng = s.EngLow + (raw-s.RawLow)*(s.EngHigh-s.EngLow)/(s.RawHigh-s.RawLow)
        if s.Clamp { eng = clamp(eng, min(s.EngLow, s.EngHigh), max(s.EngLow, s.EngHigh)) }
        return eng
    }
}
// Inverse is used by writes (writes.md). Linear with multiplier 0 is not invertible:
// validation rejects writable tags with such a scale.
func (s *Scale) Inverse(eng float64) (raw float64, err error)
```

- Raw `uint64`/`int64` above 2^53 lose precision when converted to float64 for scaling. Stage 2 flags that as `bad_type_mismatch` rather than silently rounding.
- Unscaled tags: the value passes through with the tag's `datatype`.

## 6. Propagation deadband

```go
func passesDeadband(db *Deadband, span float64, last, next *tag.Value) bool {
    if last == nil || last.Quality != next.Quality { return true }
    if db == nil { return !variantEqual(last.V, next.V) }
    a, b := last.V.Float(), next.V.Float()
    if math.IsNaN(a) || math.IsNaN(b) { return true }
    limit := db.Value
    if db.Type == DeadbandPercent { limit = db.Value / 100 * span }
    return math.Abs(b-a) > limit
}
```

- Compared against the **last propagated** value, not the last stored value, so a slow drift accumulates and eventually propagates (no "creeping" loss).
- Deadband applies to numeric types only. Non-numeric tags propagate on every change.
- The history deadband (`history.deadband`) is a separate, independent filter with the same function, applied in the history gate.
- Rate limiting by time (as opposed to by value) is a consumer concern (subscriptions.md section 4), not a tag property, in v1.

## 7. History gate

Hands values to the historian through `HistorySink` (go-design.md section 2), never blocking. The historian doc owns storage, batching and the on-disk WAL.

| `history.mode` | Row written when |
|---|---|
| `on_change` (default) | a stored change passes the history deadband (quality changes always pass); **or** `max_interval` elapsed since the last row (re-emits the current value with `ts = now`, `ts_gateway = now`, no `ts_source`, so a flat line is distinguishable from a gap); **or** an out-of-order history-only sample arrives (written as is) |
| `sampled` | every `interval` on a timer aligned to wall-clock multiples of `interval` (e.g. :00, :01), current value with `ts = tick time`; plus every quality change |
| `enabled: false` | never |

The `max_interval` and `sampled` timers live in the engine's timer wheel (go-design.md section 3). Alignment of sampled ticks to wall-clock multiples makes multi-tag reports line up.

## 8. Connection state effects

The driver host turns `Sink.ConnectionState` into an engine command (it never touches tag state directly).

| New state | Engine action on every tag of the connection |
|---|---|
| `disconnected` | `connOverlay = bad_not_connected`; last value held; `ts = now`; propagate |
| `connecting` | no change |
| `connected`, `degraded` | clear `connOverlay` but **do not** recompute quality to good: effective quality stays the overlay code until the first new sample for that tag arrives. A reconnected link says nothing about whether the old value is still true. Sparkplug sends a rebirth on reconnect (messages.md), so values recover within seconds. |
| `faulted` | `connOverlay = bad_config_error` |
| driver process crash | driver host reports `disconnected` (driver-interface.md section 3) |

Implementation detail: clearing the overlay sets `sampleQuality` to the overlay code, so stage 6 on the next sample computes the true quality.

## 9. Memory tags

- Writes go through writes.md section 5. The value is persisted (table `memory_values(tag_id, value jsonb, ts)`) by the checkpoint writer before the write is reported `ok`.
- On startup: persisted value if present, else `initial_value`, else `bad_waiting_for_initial_data`. A changed `initial_value` in config does not overwrite a persisted operator value (the operator's last word wins, which is the PLC retentive-memory convention).

## Decisions made
1. Dense slot store with atomic pointers to immutable values; engine-private state alongside, single writer.
2. A fixed 11-stage pipeline per source sample, then batch-level derived evaluation.
3. Derived tags read the store (all accepted changes); deadband filters only outbound propagation.
4. Effective quality is computed from a fixed precedence list: out of service, connection overlay, bad sample, stale, uncertain sample/range, good.
5. Timestamps truncated to milliseconds at ingest so memory, wire and DB agree.
6. Out-of-order rejection only for device-clock timestamps; out-of-order samples never feed derived functions.
7. After reconnect, tags stay bad until their own next sample.
8. No unit conversion in v1; `scale` outputs `float64` only.
9. Persisted memory-tag values win over config `initial_value`.

## Alternatives rejected and why
- **`sync.RWMutex` per tag or a global map with a lock**: either contention under fan-out reads or lock-ordering bugs in derived evaluation. Atomic pointers give lock-free reads for free with a single writer.
- **Mutating values in place**: torn reads for readers; the allocation saved is negligible.
- **Deadband before derived inputs**: would make `counter_total` totals depend on display deadbands, an unacceptable coupling.
- **Restoring good quality immediately on reconnect**: shows values that may be hours old as good. Rejected on trust grounds.
- **Unit conversion engine (UCUM arithmetic)**: real but non-trivial; not needed for milestone 1.
- **Updating `cur.TS` on every unchanged repeat**: would make every heartbeat a "change" for consumers, multiplying bus and WS traffic for nothing.

## Open questions
- CONTRACT: tag-model.md section 3 says NaN/Inf makes "value `null`", while the same section says a bad sample carries the last known value. This design holds the last value (null only if there is none) for consistency with the HMI greying rule. Contract wording should be aligned.
- CONTRACT: tag-model.md section 5's out-of-order rule is refined here to apply only to device-clock timestamps (gateway clock steps must not freeze tags). Needs a contract note.
- CONTRACT: `scale` "output datatype is the tag datatype" is restricted here to `float64`. If `int64` scaled outputs are wanted, define the rounding rule first.
- Should the canonical value object gain an optional `last_seen` (time of last sample, changed or not)? Useful for the HMI; additive. Not proposed for v1.
- Whether `eng_range` violations should also apply to derived tags (flow_rate has an `eng_range` in the worked example). Proposal: yes, stage 4 runs for derived outputs too; that is what this doc does.
