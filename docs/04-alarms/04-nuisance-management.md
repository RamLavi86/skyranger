# 04 Nuisance Management

Nuisance alarms are alarms that fire without needing action: chattering around a limit, repeated by a sloppy signal, or a consequence of something already alarmed. They train operators to ignore the phone, which is the failure that eventually costs a plant something real. This file defines each tool, what it is for, and **what it must never do**.

Governing principle: **throttle the announcement, never the record.** Every tool below changes what is shown or sent. None of them removes a transition from the journal (07) or the bus. A nuisance tool that hides data is worse than the nuisance.

The tools, in the order a signal meets them in the engine:

```
 sample ─► quality gate (02 s4) ─► condition + DEADBAND ─► ON-DELAY ─► state machine ─► OFF-DELAY (before clear)
                                                                           │
                         flags that gate annunciation: OUT-OF-SERVICE, SUPPRESSION, SHELVING, CHATTER, FLOOD
```

## 1. Deadband (hysteresis)

Purpose: stop a value hovering around the limit from toggling the condition. Defined in 03 section 2.1; applied to the raw condition.

- Choose it from signal noise: at least the peak-to-peak noise of the sensor, in engineering units. The tag-level `deadband` (tag-model s6) filters what propagates at all and applies **before** the alarm; the alarm `deadband` is the clear hysteresis. Validation warns if the tag deadband is larger than the alarm's (01).
- Must never: be used to make a limit "mean" something else. If you want the alarm to fire at 35 and clear at 30, that is a 5 unit deadband and the rationale should say why.

## 2. On-delay

Purpose: ignore short excursions. The raw condition must be true **continuously** for `on_delay` before the alarm activates. Max `1h`.

- Timer starts when the raw condition becomes true; any `false` or `unknown` cancels it. The activation timestamp is the moment the delay is satisfied; the journal keeps `condition_since`.
- Timers are engine state (heap), not persisted (09): a restart restarts the delay, never shortens it.
- Must never: be the way to detect long conditions (hours). That is a derived tag (03 s4.3).
- Rule of thumb in rationalization: `on_delay` is no longer than a fraction of the alarm's `response_time`. A 30 s delay on an alarm with a 30 min response time costs nothing; a 30 min delay on it costs the response.

## 3. Off-delay

Purpose: do not return to normal on a momentary recovery. The raw condition must be false continuously for `off_delay` before `cond_false` is delivered to the state machine. Max `1h`.

- Applies to alarms in `active_unacked` and `active_acked`. A raw `true` during the delay cancels the timer; no events are produced (the alarm never left active).
- `unknown` during the off-delay cancels the timer and holds (02): an alarm does not clear while the sensor is blind.
- Must never: delay the **activation** side. It is a clear filter only.

## 4. Chattering

Purpose: stop an alarm that keeps going active/cleared from flooding the operator and the notification channels, while still telling someone that the signal is bad.

Detection (per alarm, config `chatter: {count, window, hold}`, default from `Gateway.spec.alarms.chatter` = 3 activations in 60 s, hold 10 m; the numbers are ours, UNVERIFIED against any standard):

```
activations := timestamps of activations in the last `window`
if len(activations) >= count and not chattering:
    chattering = true          # journal chatter_start; ONE notification (cause: chatter_start)
while chattering:
    chatter_hold_until = lastActivation + hold
    # behavior below
if now >= chatter_hold_until and no activation in last `window`:
    chattering = false         # journal chatter_end with counts; ONE notification
```

Behavior while `chattering`:
1. **Annunciation is digested.** Activations and clears do not notify individually. Chatter start and end each notify once. A periodic `chatter_digest` row reports the count.
2. **The state machine keeps running.** Raw transitions are journaled, but to bound volume, consecutive chatter transitions are coalesced into one journal row per `window` carrying `count`, `first_ts`, `last_ts` (journal kind `chatter_edges`). The state object and bus still update (retained topic is one message per change, QoS 1; a chattering alarm that updates 10 times a second is itself a defect, so the bus publisher rate-limits the **alarm topic** to 1 update per second per alarm while chattering. The journal is not rate-limited).
3. **The alarm stays on the operator list** as one entry with a chatter badge. It is not auto-shelved and not auto-disabled.
4. Chatter produces a KPI entry and a maintenance work item: the fix is a deadband, a delay, or a sensor, never silence (08).

Must never: delete, drop or silently shelve the alarm; swallow the chatter-start notification; apply to `safety` class (chatter on a safety alarm is still notified every activation, plus the chatter badge).

## 5. Flood

Purpose: when a plant upset triggers dozens of alarms, the operator and phones must not be buried. A flood is an *annunciation* condition per console, not an alarm.

Detection (per console from `Gateway.spec.alarms.consoles`): `count` activations in `window` (default 10 in 10 min, UNVERIFIED against standard guidance). Entering flood:
- `flood` flag set for the console, event `skyranger.alarm.flood_changed.v1` (proposed, 07), banner in the HMI, KPI tag `_system/alarms/flood/{console}` = `true`.
- Notifications for severities **not** in `bypass_severities` (default `[critical]`) are **batched**: instead of one message per alarm, a digest every `digest_interval` (default 5 min) listing counts per severity and the top 5 by severity then age. The individual outbox rows are created and marked `digested` so the audit trail shows exactly which alarms were covered by which digest message.
- Critical alarms keep individual notifications and escalation.
- The HMI shows alarms grouped by cause where a **suppression** relationship exists (section 7), so a root cause and its consequences read as one.

Leaving flood: when the activation rate stays below `count / 2` for `window`. Event, banner off, a final digest is sent if anything is pending.

Must never: drop an alarm from the journal, drop a critical notification, or auto-ack. Flood handling is not a substitute for rationalization; the flood KPIs (08) are the argument for fixing the root causes.

## 6. Shelving

Purpose: let an operator temporarily remove a nuisance alarm from their list without engineering involvement, under control and with a time limit.

| Rule | Value |
|---|---|
| Who | `operator` or above (api.md); alarm must be `shelvable` |
| Duration | required, `<= min(alarm max_shelve, Gateway.spec.alarms.max_shelve)`; contract cap default `24h` |
| Comment | **required** (api.md); stored in journal and audit |
| Effect | `shelved_until` set; annunciation withdrawn (02 s6); escalation timers cancelled; notifications for this alarm suppressed |
| Recording | state keeps changing and journaling underneath (02); journal rows get `shelved: true` |
| Expiry | automatic (`shelve_expired`); alarm re-annunciates if still `active_unacked`; if `active_acked` it just reappears as active on the list; if `cleared_unacked` it closes (02 s3) |
| Early return | `POST /alarms/{id}/unshelve`, any operator |
| Persistence | `shelved_until` is persisted (09). Gateway restart does not unshelve; elapsed shelves expire at startup |
| Visibility | HMI has a shelved list reachable in one click; KPI counts shelved alarms and shelve duration (08) |
| Re-shelve | allowed; each shelve is its own journal row, so "shelved 40 times this month" is a visible pattern |

Must never: be silent. A shelved alarm that **activates** while shelved generates a journal row and a bus update, so a flapping shelved alarm shows up in KPIs. Shelve must never be unlimited; a permanent exclusion is a config change (out-of-service, section 8) with a reason.

## 7. Suppression by condition ("suppressed by design")

Purpose: an alarm that is meaningless in a defined plant condition: pump stopped, device in maintenance, link down so every device on it reads offline. Engineered once, in config, reviewed with the alarm, no operator action, no expiry.

```yaml
suppress_when:                     # list; the alarm is suppressed while ANY item is true (OR)
  - { tag: irrigation_active, equal: true }
  - { tag: "_system/connections/${connection}/state", not_equal: connected }
  - { tag: pump_speed, below: 1.0 }
suppress_release_delay: 30s        # stay suppressed this long after the condition ends
```

Item grammar: `tag` (reference, same resolution rules as derived inputs) plus exactly one of `equal`, `not_equal`, `above`, `below` with a literal. No logic operators, no expressions (01 rejected them).

Evaluation:
- A suppression tag with **unusable quality** counts as `false` (not suppressed). A suppression that cannot be proven does not hide an alarm. Fail loud.
- `suppress_on` the instant any item is true; `suppress_off` only after all items are false for `suppress_release_delay`, so a restart sequence does not flash the alarm.
- While suppressed, the alarm's state machine **continues to run** (02). Suppressed alarms appear in a "suppressed" list, with `suppressed_reason` (the item that matched, human readable: `connection_down`, `irrigation_active`).
- When suppression ends and the alarm is still active, it re-annunciates (`cause: unsuppressed`, notifies as an activation). The operator is told about a condition that outlasted its excuse.
- Rationalization requirement: every `suppress_when` must be explainable in one sentence; the validator warns if `rationale` is missing on an alarm that has `suppress_when`.

Must never: suppress alarms of `class: safety` (validation error), hide the **cause** alarm (the `not_connected` link alarm is never suppressed by the thing it causes; the validator rejects suppression cycles and a suppression tag that is the alarm's own tag), or evaluate through a reference to another alarm's *state* (v1 uses tag values only, which is why the dependency is explicit and testable).

Cause and consequence example (water network): link `not_connected` is the root; per-device `not_ok` alarms are suppressed while the link is down; operators see one alarm, the journal shows all of them, the KPI counts all of them as "suppressed activations".

## 8. Out-of-service

Purpose: remove an alarm for a long, managed period (instrument being replaced, known fault awaiting parts). It is a **configuration** change, not an operator action.

- `enabled: false` with mandatory `disabled_reason` (01). Applied as a revision by an `engineer`, audited with before/after.
- The engine stops evaluating; state forced to `normal` (02 row 13); journal `out_of_service` with reason and previous state.
- It appears in the **out-of-service list** (HMI) and in KPIs with age. Out-of-service for more than 30 days is a KPI warning (threshold ours).
- Re-enable: revision with `enabled: true`; journal `returned_to_service`; evaluation restarts through startup rules (09).

Must never: be an operator button. If a night-shift operator needs to silence an alarm, that is shelving (max 24 h, comment). If it must stay silent for weeks, someone with engineering authority writes down why.

Optionally the same tag may also be set `enabled: false` at tag level; tag-level disable yields `bad_out_of_service` (tag-model quality 133), which the quality hold (02 s4) treats as unusable for every limit alarm on the tag, and which fires `quality` alarms. Engineers disabling a tag should also disable its `quality` alarm, or knowingly leave it.

## 9. Which tool for which problem

| Symptom | Tool | Not this |
|---|---|---|
| Value hovers around limit | deadband | on-delay |
| Short spikes, not real | on-delay | deadband |
| Brief dips back to normal during a real event | off-delay | deadband |
| Same alarm repeatedly flaps for minutes | (chatter flag handles it; then fix with deadband/delay) | shelving forever |
| 40 alarms during a power cut | suppression by condition on the cause + flood digest | acking in bulk |
| Operator needs peace for a shift | shelve | out-of-service |
| Instrument broken for a month | out-of-service | shelve renewed daily |
| A signal is just wrong | fix the signal, then rationalize | any of the above |

## Decisions made
1. Throttle annunciation, never recording; the journal is complete under every nuisance tool.
2. Chattering is a flag with digest behavior, never auto-shelve or auto-disable; safety class is exempt from digest.
3. Flood is a per-console annunciation mode with batched digests and a critical bypass, audited by `digested` outbox rows.
4. Shelving requires comment, has a hard cap, persists across restart, and re-annunciates on expiry.
5. Suppression by condition is config-only, fail-loud (unusable suppress input means not suppressed), OR across items, forbidden for safety class.
6. Out-of-service is a reviewed config revision with a mandatory reason, never an operator control.
7. Bus publication of a chattering alarm is rate-limited to 1/s; the journal is not.

## Alternatives rejected and why
- **Auto-shelve chattering alarms** (common in vendor products): silently moves the problem out of sight with no owner; replaced by the chatter flag, KPI and a maintenance item.
- **Dropping notifications under flood**: the dropped one is the critical one. Digest plus bypass instead.
- **Suppression via alarm-to-alarm dependencies** (suppress B while alarm A is active): creates hidden ordering and cycles, and makes tests depend on state. Tag-value conditions are visible in trends.
- **AND/OR/NOT expressions in `suppress_when`**: second language (01). OR of simple comparisons covers the cases; a derived tag covers the rest.
- **Operator-controlled out-of-service**: removes management of change; shelving (time-limited) is the operator tool.
- **Global alarm inhibit at the console** ("mute all"): the classic way an alarm gets missed.
- **Persisting on/off-delay timers**: complexity for no safety gain; restart only makes alarms later by at most the delay.

## Open questions
- Chatter defaults (3 in 60 s, 10 m hold) and flood defaults (10 in 10 min) are placeholders (UNVERIFIED). Tune on the first real project, then check against ISA-18.2 / EEMUA 191 text.
- `suppress_when` with `_system` tag references requires the one-line contract clarification already raised in 01.
- Should flood digests be a separate notification kind in routes (`notify_on: [digest]`), so a manager can receive only digests? Proposed yes in 06.
- Do we want a separate "latched until reviewed" state for chatter badge clearing (an operator must acknowledge that a chatterer was seen)? Not proposed.
- Bus rate limit of 1/s for chattering alarm topics: confirm it does not violate the messages.md retained-state expectation (last message always equals the current state; our rule keeps this by publishing the final state when the limiter drains).
