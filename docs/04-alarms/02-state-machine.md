# 02 Alarm State Machine

The state names are fixed by the contract (messages.md section 5): `normal`, `active_unacked`, `active_acked`, `cleared_unacked`. Everything else the operator cares about (shelved, suppressed, out of service, chattering, input not trusted) is an **orthogonal flag** on the alarm, not a state. That is what lets an alarm clear while shelved and still record the true state.

Two rules shape everything below:
1. **State records what the plant did and what the operator did. Flags record what the system decided to show.** The state machine never forgets a transition because it was hidden.
2. **The transition function is pure.** `Next(state, input) -> (state, effects)` has no clock, no I/O, no goroutines. The engine (10) feeds it; tests table-drive it.

## 1. The state object (contract plus proposed additive fields)

Contract fields are unchanged (messages.md section 5). `P` fields are additive and ignored by contract-only consumers.

| Field | | Meaning |
|---|---|---|
| `alarm_id`, `tag_id`, `path`, `name`, `severity`, `message`, `value`, `quality` | C | as contract. `value`/`quality` are the input sample that caused the last change, not a live value |
| `state` | C | one of the four states |
| `shelved_until` | C | RFC 3339 or `null`. Non-null means the shelved flag is set |
| `active_at`, `cleared_at`, `acked_at`, `acked_by`, `updated_at` | C | see section 5 for exact semantics |
| `seq` | P | uint64 as JSON **string**. Increments by one on every change of the object. Optimistic-concurrency token for ack (05) and sync cursor (09) |
| `class` | P | `safety` ... `maintenance` (01) |
| `enabled` | P | from config; `false` = out of service |
| `suppressed` | P | bool, suppressed by condition; `suppressed_reason` string, e.g. `connection_down` |
| `chattering` | P | bool |
| `held` | P | bool, input not trusted (section 4); the alarm is frozen, not evaluated |
| `activation_count` | P | activations since alarm creation (string, uint64) |
| `annunciation` | P | derived, one of `annunciated`, `shelved`, `suppressed`, `chatter_digest`, `out_of_service` (section 6) |

## 2. States

| State | Condition now | Operator has acked this activation | Shows in operator active list |
|---|---|---|---|
| `normal` | false (or alarm never fired) | n/a | no |
| `active_unacked` | true | no | yes, flashing/audible class |
| `active_acked` | true | yes | yes, steady |
| `cleared_unacked` | false (returned to normal) | no | yes, as "returned, needs ack" |

"Condition" is the alarm's condition after deadband, on-delay and off-delay have been applied (04). The state machine only sees the debounced boolean.

### Text state diagram

```
                         cond_true (after on_delay)
        ┌─────────┐ ─────────────────────────────────► ┌────────────────┐
        │ normal  │                                     │ active_unacked │ ◄──┐
        └─────────┘ ◄───────┐                           └────────────────┘    │
             ▲              │                              │           │      │
             │ ack          │ cond_false                   │ ack       │      │ cond_true
             │              │ (after off_delay)            ▼           │      │ (re-activation:
   ┌─────────────────┐      │                      ┌──────────────┐    │      │  seq+1, count+1)
   │ cleared_unacked │ ─────┼──────────────────────│ active_acked │    │      │
   └─────────────────┘      └──────────────────────└──────────────┘    │      │
             ▲                                                         │      │
             └─────────────────────────────────────────────────────────┴──────┘
                          cond_false (after off_delay) from active_unacked

   In words:
     normal          --cond_true-->  active_unacked
     active_unacked  --ack-------->  active_acked
     active_unacked  --cond_false->  cleared_unacked   (or normal if ack_required: false)
     active_acked    --cond_false->  normal            (acknowledged alarms return silently)
     cleared_unacked --ack-------->  normal
     cleared_unacked --cond_true-->  active_unacked    (new activation)
     any state       --disabled--->  normal            (cause out_of_service, journaled)
```

Flags never appear in the diagram because they never move the state. They are drawn separately:

```
   flags (independent, any combination, each journaled on change)
     shelved     operator set, expires, max max_shelve          gates annunciation
     suppressed  proven condition (suppress_when), no expiry    gates annunciation
     chattering  detected by count/window                       digests annunciation
     held        input quality not usable                       freezes evaluation
     enabled     config (false = out of service)                removes the alarm entirely
```

## 3. Full transition table

Inputs to `Next`: `cond_true`, `cond_false` (debounced), `ack`, plus flag inputs. `*` = any state. "no-op" means no change, no event, no `seq` increment.

| # | From | Input | To | Effects |
|---|---|---|---|---|
| 1 | `normal` | `cond_true` | `active_unacked` | `active_at = ts`, clear `cleared_at`/`acked_*`, `activation_count++`, journal `activated`, notify `activate` |
| 2 | `active_unacked` | `ack` | `active_acked` | `acked_at`, `acked_by`, journal `acked`, cancel escalation timers, notify `ack` |
| 3 | `active_unacked` | `cond_false` | `cleared_unacked` | `cleared_at = ts`, journal `cleared`, notify `clear`. If `ack_required: false`: to `normal` instead (journal `cleared`) |
| 4 | `active_acked` | `cond_false` | `normal` | `cleared_at = ts`, journal `cleared`, notify `clear` |
| 5 | `cleared_unacked` | `ack` | `normal` | `acked_at`, `acked_by`, journal `acked` |
| 6 | `cleared_unacked` | `cond_true` | `active_unacked` | re-activation, same effects as row 1. The previous unacked return is superseded; journal records `reactivated` with `previous_state` |
| 7 | `normal` | `ack` | `normal` | no-op (result `noop` to the caller, not an error) |
| 8 | `active_acked` | `ack` | `active_acked` | no-op |
| 9 | `active_unacked` | `cond_true` | same | no-op |
| 10 | `active_acked` | `cond_true` | same | no-op |
| 11 | `cleared_unacked` | `cond_false` | same | no-op |
| 12 | `normal` | `cond_false` | same | no-op |
| 13 | `*` | `disable` (config `enabled: false`) | `normal` | journal `out_of_service` with `disabled_reason`; flags cleared; notify nothing (a revision event already exists) |
| 14 | `*` | `remove` (alarm deleted by revision) | gone | final journal row `removed`; state row archived |
| 15 | `normal` | `enable` | `normal` | journal `returned_to_service`; evaluation restarts under startup rules (09) |

Flag inputs (state unchanged, `seq++`, journal row, `state_changed` event with `cause`):

| Input | Allowed when | Effect on flag | Side effects |
|---|---|---|---|
| `shelve(duration, comment, actor)` | `shelvable`, actor role `operator`, `duration <= min(max_shelve, gateway cap)`, comment non-empty | `shelved_until = now + duration` | pending escalation timers cancelled; no notifications while shelved |
| `unshelve(actor)` | shelved | `shelved_until = null` | if still `active_unacked`: re-annunciate (notify `unshelve`); if `cleared_unacked`: moved to `normal` with `acked_by = system:unshelve` (nobody should ack an alarm they chose not to see) |
| `shelve_expired` (timer) | shelved | as unshelve | actor `system:shelve_expiry`; as above |
| `suppress_on(reason)` | `suppress_when` evaluates true | `suppressed = true` | annunciation withdrawn; escalation cancelled |
| `suppress_off` | `suppress_release_delay` elapsed | `suppressed = false` | if still `active_unacked`: re-annunciate with `cause: unsuppressed`; if `cleared_unacked`: moved to `normal` with `acked_by = system:unsuppress` (the operator never saw it, as with unshelve) |
| `chatter_on` / `chatter_off` | detection (04) | `chattering` | one `chatter_start` notification, then digest |
| `hold_on` / `hold_off` | input quality unusable / usable | `held` | see section 4 |

Rules of the table:
- **Ack is an operator acknowledgement of the current activation.** Row 6 means an operator who acked activation 1 cannot accidentally "pre-ack" activation 2.
- **An alarm cannot be acked into `normal` while the condition is true.** Ack only moves `active_unacked` to `active_acked`.
- **Unacked returns accumulate.** If 5 alarms are `cleared_unacked`, the operator sees 5 "returned" items. This is the point: someone must see that it happened. Escape hatches for noise are in 04, not in the state machine.
- **Disabled flushes.** Out of service forces `normal`; the journal row keeps the prior state in `previous_state`, so "acked or not" is not lost.
- Rows 9 to 12 produce no events. In particular a value that keeps exceeding the setpoint does not generate transitions.

## 4. The quality-hold rule

Each alarm evaluates its condition as **tri-state**: `true`, `false` or `unknown`. `unknown` means the input quality is not usable for this alarm type.

| Input quality | Limit-style types (`high`, `low`, `equal`, `not_equal`, `deviation`, `rate_of_change`) | `quality` type |
|---|---|---|
| `good`, `good_manual` | evaluate | evaluate (condition false) |
| `uncertain` (64), `uncertain_eng_range_exceeded` (66) | evaluate (the value exists and is plausible as a reading) | evaluate (condition true if `quality_match: not_good`) |
| `uncertain_last_known` (65), `uncertain_initial_value` (67) | **hold** | evaluate (true) |
| any `bad_*` (128-135) | **hold** | evaluate (true) |

`hold` means:
1. `held = true`. State is frozen: an active alarm stays active, a normal alarm stays normal. A held alarm **neither activates nor clears.**
2. Any pending on-delay or off-delay timer is cancelled. On leaving hold the condition must hold continuously from scratch.
3. No notification is generated for hold itself. Loss of data is the job of a `quality` alarm or the device health alarm; a limit alarm going silent must never be the only sign.
4. When quality becomes usable again, the engine evaluates the **current** value (not the value from before) and `held = false`.

Why: an alarm that clears because the sensor died looks exactly like a recovery. Conversely a high alarm that fires on a `bad_comm_failure` last-known value says nothing about the plant. Both are lies. The water meter leak alarm benefits directly: if the meter drops off the network at 01:00 during a leak, the leak alarm stays active (not silently cleared) and the `not_ok` alarm tells the operator why the data stopped (11).

Held alarms are counted in KPIs (08) as "not evaluable": an alarm that has been held for days is a sign that something else is broken.

## 5. Timestamps and ordering

| Field | Set to |
|---|---|
| `active_at` | the `ts` of the sample at which the debounced condition became true (on-delay satisfied), or the timer evaluation time for timer-driven transitions. The journal row also carries `condition_since` (when the raw condition first became true), so the on-delay is visible in reports |
| `cleared_at` | likewise for the debounced false |
| `acked_at` | gateway receive time of the ack command (`ts_gateway` semantics): an ack is an operator action, not a sample |
| `updated_at` | time of the last change of any field |

Ordering: the engine processes inputs for one tag in `ts` order (tag engine guarantee, messages.md section 3). Commands (ack, shelve) are serialized in the same loop with samples, so "ack arrives just as the alarm clears" has exactly one outcome per interleaving, never a race. A late sample (`ts` older than the tag's current `ts`) is history-only and never reaches the alarm engine (tag-model section 5).

Clock jumps: durations (on-delay, shelve expiry, escalation) use the monotonic clock. Wall-clock time is used only for stored timestamps. A shelve that was stored as `shelved_until` wall time is re-armed on restart against the current wall clock (09).

## 6. Annunciation

`annunciation` is a function of flags, not stored independently:

```go
func Annunciation(s State) string {
    switch {
    case !s.Enabled:                       return "out_of_service"
    case s.ShelvedUntil != nil:            return "shelved"
    case s.Suppressed:                     return "suppressed"
    case s.Chattering && s.DigestOnly:     return "chatter_digest"
    default:                               return "annunciated"
    }
}
```

Only `annunciated` alarms go in the operator's main active list, trigger sounds and generate immediate notifications. Others are in their own lists (shelved list, suppressed list) which the HMI must make one click away. **Everything is always in the journal and on the bus**, whatever the annunciation.

## 7. Table-driven test sketch

```go
var cases = []struct {
    name  string
    from  State
    in    Input
    want  State
    fx    []EffectKind
}{
    {"activate", St(Normal), CondTrue(t0), St(ActiveUnacked, Active(t0), Count(1)), FX(Journal, Notify)},
    {"ack", St(ActiveUnacked), Ack("user:alice", t1), St(ActiveAcked, AckedBy("user:alice", t1)), FX(Journal, CancelEscalation, Notify)},
    {"clear unacked", St(ActiveUnacked), CondFalse(t1), St(ClearedUnacked, Cleared(t1)), FX(Journal, Notify)},
    {"reactivate", St(ClearedUnacked), CondTrue(t2), St(ActiveUnacked, Active(t2), Count(2)), FX(Journal, Notify)},
    {"clear while shelved", St(ActiveUnacked, Shelved(t9)), CondFalse(t1), St(ClearedUnacked, Shelved(t9)), FX(Journal)}, // no Notify
    {"ack no-op", St(Normal), Ack("user:alice", t1), St(Normal), nil},
    // ... one case per row of section 3, plus property tests: no sequence of inputs reaches
    // active_acked without passing active_unacked; seq strictly increases; flags never change state.
}
```

Property tests (rapid or `testing/quick`): random input sequences preserve (a) state is always one of four, (b) `active_acked` is reachable only via `ack` from `active_unacked`, (c) flag inputs never change `state`, (d) `cond_true`/`cond_false` no-ops produce no `seq` change.

## Decisions made
1. The four contract states are the whole state machine. Shelved, suppressed, held, chattering, disabled are flags.
2. Pure transition function, serialized with commands in one loop.
3. Re-activation from `cleared_unacked` goes to `active_unacked` as a new activation (row 6); the earlier unacked return is superseded, not lost (journal keeps both).
4. Out of service forces `normal` and journals the previous state.
5. Quality hold: unusable input freezes the alarm and cancels timers; it never activates or clears. `uncertain` and `uncertain_eng_range_exceeded` count as usable.
6. A cleared-while-shelved alarm is auto-closed on unshelve with `acked_by = system:unshelve`, so nobody acknowledges an alarm they chose not to see, and the record shows why.
7. `ack_required: false` skips `cleared_unacked` only; activation still needs an ack (matches 01).
8. Ack of an alarm that cannot be acked is a `noop` result, not an HTTP error, so bulk and retry paths are idempotent.

## Alternatives rejected and why
- **Adding `shelved` and `disabled` as states** (tag-model section 8 lists them as states, messages.md section 5 corrects this to flags): states multiply (`shelved_active_unacked`, ...) and an alarm clearing while shelved would lose its true state. Flags keep four states.
- **A separate "pending" or "on-delay" state in the contract state set**: the contract fixes four names. Pending is an engine-internal timer, not operator-visible and not persisted.
- **Latching alarms with a separate reset action**: `cleared_unacked` already forces a human to see the return. A second reset step adds clicks without information. Revisit for safety classes if a customer needs it.
- **Auto-clearing unacked returns after N minutes**: hides events from operators on a quiet night shift. The journal keeps the record, but the list must not lie about what was acknowledged.
- **Quality-hold on every `uncertain` code**: would blind alarms on values flagged only `uncertain_eng_range_exceeded`, where the reading is plausible and often exactly the dangerous one (a tank level above its range).
- **Letting a held alarm clear after a timeout**: a timeout makes lost data indistinguishable from recovery; `quality` alarms are the explicit, visible path.

## Open questions
- Contract wording: tag-model.md section 8 still calls `shelved` and `disabled` states in prose; messages.md section 5 says flags. Propose a one-line correction in tag-model.md.
- Is treating `uncertain_eng_range_exceeded` as usable right for every customer? A mis-scaled 4-20 mA sensor also produces it. Per-alarm override `hold_on: [codes]` is possible but not proposed yet.
- ISA-18.2 defines specific state names and their transitions (including shelving, suppression by design and out-of-service as states of an alarm). Mapping our flags to those names for reports is UNVERIFIED and should wait for the standard text.
- Should a re-activation of a `cleared_unacked` alarm notify again, or only update? Proposed: notify (row 6), with the flood and chatter rules (04) protecting against spam. Needs field experience.
