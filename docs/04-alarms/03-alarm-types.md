# 03 Alarm Types, Priority and Classification

Each alarm type is a function from the input sample (and a little per-alarm memory) to a tri-state condition: `true`, `false`, `unknown` (02 section 4). Deadband, on-delay and off-delay (04) are applied by the engine on top, identically for every type. A type only answers "is the raw condition true right now?".

Contract types: `high`, `low`, `equal`, `not_equal`, `quality`. Proposed additive types: `deviation`, `rate_of_change`. Comms loss, stale data and duration/continuous-flow alarms are **patterns built from these types**, not new types (section 4).

## 1. Type summary

| Type | | Tag datatypes | Raw condition becomes true when | Clears when (after deadband) | Quality handling |
|---|---|---|---|---|---|
| `high` | C | numeric | `v > setpoint` | `v <= setpoint - deadband` | hold on unusable (02 s4) |
| `low` | C | numeric | `v < setpoint` | `v >= setpoint + deadband` | hold |
| `equal` | C | any | `v == setpoint` | `v != setpoint` | hold |
| `not_equal` | C | any | `v != setpoint` | `v == setpoint` | hold |
| `quality` | C | any | quality matches `quality_match` | quality does not match | never held; the input is the quality |
| `deviation` | P | numeric | `abs(v - ref) > setpoint` | `abs(v - ref) <= setpoint - deadband` | hold if **either** tag unusable |
| `rate_of_change` | P | numeric | rate beyond `setpoint` (below) | rate back inside by `deadband` | hold; also `unknown` until window filled |

Notes on the contract types:
- The contract says `high: v > sp`. Clear at exactly `sp - deadband` is inclusive (`<=`), so `deadband: 0` means "clear as soon as the raw condition is false". 01 phrases the clear point as "below 34.0"; read as the same threshold, with equality clearing.
- `equal`/`not_equal` compare after datatype conversion: `bool` to bool, integers exactly, `string` byte-exact. For `float64` the config validator emits a **warning** ("equality on float is rarely what you want"); the engine compares with `==` and the integrator is responsible.
- `deadband` on `equal`/`not_equal` is rejected (01).

## 2. Evaluation formulas

### 2.1 high / low

```go
// hys = deadband; active reports current alarm condition (debounced upstream or raw, see 04)
func condHigh(v, sp, hys float64, rawActive bool) Tri {
    if v > sp          { return True }
    if rawActive && v > sp-hys { return True }   // inside the deadband band: stay as we were
    return False
}
func condLow(v, sp, hys float64, rawActive bool) Tri {
    if v < sp          { return True }
    if rawActive && v < sp+hys { return True }
    return False
}
```

`rawActive` is the previous **raw** condition (before delays), kept per alarm. The deadband applies to the raw condition, so the sequence "above limit, dips into the band, goes back above" never produces a clear that the off-delay would then have to absorb.

`setpoint_ref` is re-read on every evaluation: when the referenced memory tag changes, the engine re-evaluates the alarm with the last value of its own tag. A `setpoint_ref` tag with unusable quality makes the alarm `unknown` (held).

### 2.2 deviation

`ref` is the `reference` tag. Evaluate whenever **either** tag changes. Both quality gates apply; if one is unusable the result is `unknown`. Typical use: setpoint vs process value of a controller, or two redundant sensors.

```yaml
- name: level_mismatch
  type: deviation
  reference: tank01/level_b
  setpoint: 0.15          # metres of disagreement
  deadband: 0.03
  on_delay: 2m
  severity: medium
  class: equipment
  message: "Level sensors A and B disagree"
```

`setpoint` is always a positive magnitude; the validator rejects `<= 0`.

### 2.3 rate_of_change

Rate over a sliding window, same definition as the tag-model `rate` built-in:

```
rate = (v(now) - v(now - window)) / window * per        # e.g. window 5m, per 1m -> units per minute
```

- `setpoint > 0`: true when `rate > setpoint` (rising too fast). `setpoint < 0`: true when `rate < setpoint` (falling too fast). Clear when the rate is back within `abs(setpoint) - deadband`, on the same side.
- `unknown` until `window` of samples exists after start or after a hold (no alarm on a warming-up buffer).
- The buffer is the alarm's own ring (`window` worth of post-deadband samples, plus one sample before the window edge, interpolated linearly). Memory is bounded by `window` divided by the tag's propagation interval, capped at 4096 samples per alarm.
- Prefer `rate_of_change` for process values (temperature, level). For monotonic totals (water meter) use the tag engine `rate` or `continuous_activity` derived tags and a plain alarm on the result, so there is one implementation of windowing.

```yaml
- name: fast_drop
  type: rate_of_change
  window: 5m
  per: 1m
  setpoint: -0.5          # m per minute: reservoir losing level fast (burst main)
  deadband: 0.1
  on_delay: 1m
  severity: high
  class: process
  message: "Reservoir level falling fast"
```

### 2.4 quality

`quality_match: not_good` (default, contract behavior): true when quality class is uncertain or bad. `quality_match: bad`: true only for class bad (128-135). Proposed third value `stale`: true only for `uncertain_last_known` (65). Alarm `value` in the state object is the last known value.

The quality alarm is the **explicit** path for loss of data: it activates after `on_delay`, clears when quality is good (class 0, including `good_manual`) for `off_delay`. It is never itself held.

## 3. Priority (severity) and classification

`severity` is the contract's priority field (`low`, `medium`, `high`, `critical`). `class` is the proposed classification (`safety`, `environmental`, `process`, `equipment`, `system`, `maintenance`). They are independent: priority answers "how fast must someone respond", class answers "what kind of problem is it, and which policy applies".

Priority policy (defaults, all overridable in `Gateway.spec.alarms`; response times are targets for rationalization, not guarantees):

| Severity | Meaning | Typical response target | Notification default | Ack comment | Shelvable | Flood bypass |
|---|---|---|---|---|---|---|
| `critical` | Loss or harm imminent; act now, day or night | minutes | SMS + push, repeat hourly until acked, escalation tier at 30 min | required (01 default) | yes, max 24 h, comment required | yes (never digested) |
| `high` | Serious; respond this shift | tens of minutes | SMS + push, escalate at 30 min | optional | yes | no |
| `medium` | Needs attention; no immediate harm | hours | push + email | no | yes | no |
| `low` | Information that needs a human eventually | next working day | email digest | no | yes | no |

Classification effects:

| Class | Default `shelvable` | Default route hint | Report grouping |
|---|---|---|---|
| `safety` | **false** | highest tier | safety alarms (never in nuisance KPIs as "removable") |
| `environmental` | true | duty manager included | regulatory reporting |
| `process` | true | by console | operations |
| `equipment` | true | maintenance console | maintenance backlog |
| `system` | true | integrator contact | platform health (the `_system` alarms) |
| `maintenance` | true | email only | planned work |

Guidance on distribution (how many alarms of each severity a healthy system has) exists in the alarm-management literature, with concrete percentage splits. Quoted numbers are **UNVERIFIED** and not hard-coded; 08 reports the actual distribution and flags a very top-heavy one (for example more than half of configured alarms `critical`) as a warning.

Rationalization: `rationale` (`cause`, `consequence`, `action`, `response_time`) is how an alarm answers "why does this exist". The validator warns for `high` and `critical` without it. An alarm with no action an operator can take is an event, not an alarm (README): the lint suggests `class: maintenance` or removing the alarm.

## 4. Patterns for the cases you will actually use

### 4.1 Comms loss (device offline)

Three layers, each with a different meaning, so one failure does not become fifty alarms:

| Layer | Source | Alarm | What it says |
|---|---|---|---|
| Link | `_system/connections/{name}/state` | `not_connected` (gateway-defined, 01 s6) | Gateway cannot talk to the broker/PLC/server. Every tag on it is `bad_not_connected` |
| Device | liveness derived tag (`health`) | `not_equal 2` for 60 s | This device went quiet or offline while the link is fine, or the link is down |
| Tag | any tag | `quality` | This one value is not good (finer grain, optional) |

Device-level alarms get `suppress_when: [connection down]` (04 section 7), so a broker or LAN failure raises exactly one alarm (the link) rather than one per device. The `health` tag distinguishes `0 offline` and `1 stale` for display; one alarm covers both because the operator action is the same (go look at the device).

```yaml
- name: not_ok
  type: not_equal
  setpoint: 2
  on_delay: 60s
  severity: medium
  class: equipment
  message: "Water meter ${edge_node_id} not reporting"
  suppress_when:
    - { tag: "_system/connections/${connection}/state", not_equal: connected }
```

### 4.2 Stale data

Source tags set `stale_after` (tag-model section 6); on expiry quality becomes `uncertain_last_known`. Alarm with `type: quality`, `quality_match: stale`, `on_delay: 0s` (the tag engine already waited `stale_after`). Used on process values read over Modbus where the PLC keeps answering but a field device behind it stopped updating. For a device with a heartbeat, prefer the liveness pattern, because it distinguishes "no flow" from "dead device" explicitly.

### 4.3 Duration / continuous-flow

Hours-long conditions belong in derived tags (README decision 5, 01). The alarm is a plain `equal true`.

```yaml
- name: leak
  type: equal
  setpoint: true
  severity: high
  class: environmental
  message: "Flow has not stopped for ${leak_duration}: possible leak"
```

The temporal logic (`max_gap`, `duration`) lives in `continuous_activity`, tested once in the tag engine, and visible as a tag in trends. Alarm `on_delay` is capped at `1h` for the same reason: a 2 h condition held only in a timer is invisible, not queryable, and lost on restart (09).

For other "condition for a long time" needs (pump running dry for 20 min): build a derived tag from existing built-ins when one fits; if not, ask for a new built-in rather than stretching `on_delay`.

### 4.4 Digital alarms

A `bool` tag with `equal true` (or `not_equal`). Enumerations (`health`, valve state) use `equal`/`not_equal` with the numeric value; the HMI shows `enum_labels`. No special type is needed.

## 5. What a type must never do

- Read the clock itself (all time comes from the sample `ts` or the engine's timer service, so tests are deterministic).
- Cache a value across a hold: after `unknown`, evaluation restarts from the current sample.
- Throw on a datatype it cannot compare: validation prevents it at apply time; at runtime a mismatch yields `unknown` plus an engine error counter, never a panic (10).

## Decisions made
1. No `duration` or `comms_loss` alarm types in v1; they are patterns over derived tags, `quality` and `not_equal`.
2. `deadband` is applied to the raw condition with the previous raw state; delays are applied afterward by the engine (04).
3. `deviation` and `rate_of_change` are proposed as additive types; both are small, pure, and documented formulas.
4. `rate_of_change` setpoint sign encodes direction; no separate `direction` field.
5. `quality_match: stale` is proposed alongside `not_good` and `bad`.
6. Safety-class alarms default to `shelvable: false`.
7. Severity distribution is reported, not enforced, and no percentage target is hard-coded.

## Alternatives rejected and why
- **A `duration` type with an `hours` field**: duplicates `continuous_activity`, loses state on restart unless it checkpoints too, and invisible in trends.
- **A `comms_loss` type tied to connection state**: the liveness derived tag already carries the right semantics (offline vs stale vs ok) and works for devices without a driver connection concept (heartbeat-based).
- **Percent deadband and percent setpoints**: engineering units are what gets reviewed (01).
- **Expression-based conditions** (`v1 > v2 * 1.1`): rejected in 01 and tag-model for the same reasons.
- **Evaluating limit alarms on `uncertain_last_known`**: gives false activations and false clears (02).
- **Separate rising and falling rate types**: sign of the setpoint is enough and halves the validation surface.

## Open questions
- `deviation`, `rate_of_change`, `quality_match: stale`, and the `window`/`per` fields need the contract amendment listed in 01 (01's `quality_match` table lists only `not_good` and `bad`; `stale` is an extension of that table).
- Multi-state or enum "bad state set" alarms (`in: [3, 4, 5]`) are common on PLC state words. Not proposed; `equal` plus a derived `select` or a small `in_set` built-in may be needed. Ask the tag-model owner.
- Whether to support alarms on `string` tags beyond equality. Not needed so far.
- Specific ISA-18.2 alarm type taxonomy (absolute, deviation, rate of change, discrepancy, and so on) is UNVERIFIED from memory; the names here are ours.
