# 11 Water Meter Alarms End to End

The first two alarms of Sky Ranger: **continuous flow (leak)** and **device offline**, on the ESP32 pulse-counter meter (1 pulse = 1 L). This file follows them from a pulse on the wire to a phone and a journal row, then through the failures that matter: operator does nothing, device power cut, gateway restart, broker loss, planned irrigation. It doubles as the acceptance script for the L3 harness (docs/09-testing).

Contract facts used (tag-model.md s7, s10): `total_volume` (counter_total), `flow_rate` (rate, 5 m window), `health` (liveness, `stale_after: 150s`, 0 offline / 1 stale / 2 ok), `leak` (continuous_activity, `max_gap: 15m`, `duration: 2h`, `eval_interval: 30s`). Heartbeat every 60 s. All times below are UTC on 2026-11-06/07.

## 1. Configuration

### 1.1 The alarms on the template (contract fields plus proposed additions)

The contract-only form (tag-model s10) is valid and works. The form below is what we ship in `pulse_water_meter`, with additive fields from 01:

```yaml
    - name: health
      # ... contract tag definition unchanged ...
      alarms:
        - name: not_ok
          type: not_equal
          setpoint: 2
          on_delay: 60s
          severity: medium
          class: equipment
          message: "Water meter ${edge_node_id} not reporting"
          suppress_when:
            - { tag: "_system/connections/${connection}/state", not_equal: connected }
          suppress_release_delay: 30s
          rationale:
            cause: "Meter lost power, Wi-Fi, or firmware crashed (NDEATH or heartbeat missing)"
            consequence: "No consumption data and no leak detection for this meter"
            action: "Check power and Wi-Fi at the meter; check RSSI history; power-cycle"
            response_time: 4h
    - name: leak
      # ... contract tag definition unchanged ...
      alarms:
        - name: leak
          type: equal
          setpoint: true
          severity: high
          class: environmental
          message: "Flow has not stopped for ${leak_duration}: possible leak"
          rationale:
            cause: "Burst or leaking pipe, fitting, or tap left open downstream of the meter"
            consequence: "Water loss and possible damage; cost grows with time"
            action: "Check for open taps or irrigation; isolate and inspect downstream pipework"
            response_time: 2h
```

Why these choices:
- `leak` has **no `on_delay`**. The 2 h condition is already in `continuous_activity`; adding delay would duplicate it (03 s4.3).
- `not_ok` uses `on_delay: 60s` (contract). The liveness tag already waits `stale_after: 150s` for the "stale" case; `online == false` (NDEATH) is immediate, so the 60 s delay is the debounce for a Wi-Fi blip.
- `not_ok` is suppressed while the **connection** is not `connected`, so a broker or driver failure creates one alarm (the link), not one per meter.
- The `leak` alarm has no `suppress_when` by default. A leak is the alarm that costs money; suppression is opt-in per instance (section 5).
- `severity: medium` for offline would not page at 03:00 in some policies; the `water_oncall` route in 01 includes `medium`, so it does page. Integrator decision per customer (Open questions).

### 1.2 Instance and routing

```yaml
apiVersion: skyranger/v1
kind: TagGroup
metadata: { name: pumphouse }
spec:
  base_path: site1/pumphouse
  instances:
    - name: meter01
      template: pulse_water_meter
      params: { connection: local-sparkplug, edge_node_id: esp32-meter01, initial_total_l: 1234567 }
```

Routing (01 s4, unchanged): console `water` owns `site1/pumphouse`; route `water_oncall` sends `medium`, `high`, `critical` activations, clears and escalations: tier 0 immediately to `oncall_phone` by `sms_modem` and `ntfy`; tier 1 after 30 min unacked to `manager` by `sms_modem` and `mail`. `everything_to_hq` posts every change to the HQ webhook.

## 2. Scenario A: a real leak, nobody answers, then someone does

Setup: a slow leak of about 0.5 L/min (a pulse every ~2 min) starts at 22:00. `max_gap 15m` is satisfied by a pulse every 2 min.

| Time (UTC) | What happens | Tags | Alarm `water/leak` | Notified |
|---|---|---|---|---|
| 22:00:20 | first pulse after quiet; `continuous_activity` starts `active_since` | `leak` false good | normal | |
| 22:00-00:00 | pulses every ~2 min; `flow_rate` reads 0 or 0.2 L/min in a 5 min window (1 L resolution; why the rate alarm would never have caught it, tag-model decision 8) | `leak` false | normal | |
| 00:00:50 | `leak` evaluation (30 s tick) finds 2 h of continuous activity: `leak = true` good | `leak` true good | `normal -> active_unacked`, `active_at 00:00:50`, `seq` 1 to n, journal `activated` | outbox rows for `water_oncall` tier 0 and `everything_to_hq` created in the same transaction |
| 00:00:52 | dispatcher picks rows | | | modem: `[gw01] HIGH LEAK site1/pumphouse/meter01 Flow has not stopped for 2h: possible leak. Active 00:00 #12` (SMS), ntfy push, HQ webhook |
| 00:03 | SMS arrives; nobody acks | | `active_unacked` | |
| 00:30:50 | tier 1 timer (anchored at `active_at`): still `active_unacked`, annunciated | | journal `escalated` tier 1 | manager: SMS plus email with rationale and action text |
| 00:41 | Alice (on call) acks in the HMI: `POST /alarms/{id}/ack {"comment": "Hydrant flushing? calling site", "expected_seq": "12"}` | | `active_unacked -> active_acked`, `acked_by user:alice` | HQ webhook; `water_oncall` has `notify_on` without `ack`, so no SMS |
| 01:10 | crew finds a stuck float valve, closes the isolation valve | | `active_acked` | |
| 01:10:20 | last pulse | | | |
| 01:25:20-01:25:50 | 15 min with no increase: `continuous_activity` goes false at the first tick after `last increase + max_gap` | `leak` false | `active_acked -> normal`, `cleared_at ~01:25:50`, journal `cleared` | SMS `[gw01] CLEARED LEAK ... 01:25 (active 1h25m) #13` to oncall and manager contacts that were notified |

Expected `GET /api/v1/alarms?state=active_acked` while acked:

```json
{
  "items": [{
    "alarm_id": "01933b81-0a1b-7c2d-8e3f-405162738495",
    "tag_id": "01933b7e-9a00-7c3d-9a2b-4f6e8d1c2b3a",
    "path": "site1/pumphouse/meter01/leak", "name": "leak",
    "severity": "high", "class": "environmental",
    "state": "active_acked", "shelved_until": null,
    "message": "Flow has not stopped for 2h: possible leak",
    "value": true, "quality": "good",
    "active_at": "2026-11-07T00:00:50.000Z", "cleared_at": null,
    "acked_at": "2026-11-07T00:41:12.000Z", "acked_by": "user:alice",
    "updated_at": "2026-11-07T00:41:12.000Z",
    "seq": "13", "activation_count": "3", "annunciation": "annunciated",
    "suppressed": false, "chattering": false, "held": false, "enabled": true
  }]
}
```

Journal rows for the episode: `activated` (condition_since = 00:00:50, value true, setpoint true), `escalated` (tier 1), `notified` x N (channel, latency), `acked` (comment), `cleared`.

If the leak had been **acked at 00:07** (before tier 1): no `escalated` row, the tier timer is cancelled (05 s2.1 rule 3), the manager is never messaged.

## 3. Scenario B: device offline (power cut) and recovery

Follows the tag-model trace. The leak alarm is normal throughout.

| Time | Event | `health` | `raw/*` and derived quality | Alarm `water/not_ok` |
|---|---|---|---|---|
| 10:20:00 | power cut; broker publishes NDEATH (Will) | `0` offline (good) | `uncertain_last_known` on raw and derived tags; `leak` also `uncertain_last_known` (worst input) so the leak alarm is **held** (no flag noise; held < 1 min is not journaled) | raw condition true (0 != 2), on-delay timer starts at 10:20:00 |
| 10:21:00 | 60 s of continuous 'not ok' | 0 | | `normal -> active_unacked` (`active_at 10:21:00`, `condition_since 10:20:00`); SMS "MEDIUM NOT_OK ... Water meter esp32-meter01 not reporting" |
| 10:22:00 | power back; NBIRTH with `boot_count` +1, `pulse_count` 3 | `2` ok | good; `counter_discontinuity{reset, delta_applied: 3}` event (tag area) | raw condition false; off-delay 0: `active_unacked -> cleared_unacked`, `cleared_at 10:22:00`; SMS "CLEARED" |
| 10:35 | Alice acks the return | 2 | | `cleared_unacked -> normal`, `acked_by user:alice` |

What the operator sees (HMI contract from the tag-model trace): `flow_rate = 0.0` good with `health 2` is "no flow"; non-good quality with `health 0` is "dead device". The alarm and the quality tell the same story.

If the device **flaps** (reboots every 90 s because of a brown-out): the on-delay of 60 s lets some cycles through; after 3 activations in 60 s the alarm becomes `chattering` (04 s4) and the oncall person receives one chatter message and a digest, not 20 SMS. The journal keeps `chatter_edges` rows; the KPI "top offenders" shows `meter01/not_ok`; the fix is a power supply, not a delay.

## 4. Scenario C: leak in progress, then the device dies

A leak is active (`active_unacked` since 00:00:50). At 01:00:00 power is cut.

| Time | Event | Result |
|---|---|---|
| 01:00:00 | NDEATH; `leak` quality `uncertain_last_known` | `water/leak` stays `active_unacked`, `held = true` (02 s4). **It does not clear.** |
| 01:01:00 | `not_ok` activates (60 s) | SMS: the operator now knows data has stopped. The leak alarm text on the HMI shows "held: input not trusted" |
| 01:01:00 | journal: `held` row (hold > 1 min) | |
| 01:30:00 | device returns, good quality | `held = false`; current `leak` value is evaluated |

Here the **contract leaves one thing open**: `continuous_activity` has `max_gap: 15m`, and the device was silent for 30 min. If the tag engine treats the silent time as a gap, `leak` becomes false at 01:30 and the alarm goes `cleared_unacked` although the pipe may still be leaking; the leak is then re-detected only after another 2 h of flow. The recommended behavior (a request to the tag-model owner, Open questions, and 09) is that `continuous_activity` **freezes** while its input quality is not good: neither advancing nor resetting. Then the alarm stays active and the operator, who got the `not_ok` SMS, decides. Until the contract says so, this scenario's expected result is marked **depends on tag engine** in the L3 script.

## 5. Scenario D: gateway restart during the leak

At 01:00:00 the gateway process is killed (power blip), back at 01:01:30 (downtime 90 s).

| Step | Behavior | Invariant (09 s2.1) |
|---|---|---|
| Restart | `alarm_state` loaded: `leak` `active_unacked`, `seq` 12, `active_at 00:00:50`. Startup hold begins (alarm held until its tag has a post-start sample) | restart does not change state |
| Journal | `engine_gap` row for the `leak` alarm: `from 01:00:00 (last_alive_at) to 01:01:30` | downtime explicit |
| Notifications | the 00:00:52 SMS was `sent`, so no re-send. The `recovered` summary is sent only if downtime exceeded `recovery_notify_after` (2 m): not here | no storm |
| Tiers | alarm unacked, tier 1 due 00:30:50 is in the past: the `(alarm_id, seq, tier 1)` outbox row exists if it was sent; if the downtime had straddled 00:30:50, tier 1 is enqueued immediately at startup, with the text "gateway restarted, alarm active since 00:00" | 05 s2.2 |
| `leak` tag value | depends on whether `continuous_activity` state (`active_since`, last increase) is checkpointed (Open questions). If yes: the tag resumes `true` after warm-up; alarm unchanged | |
| If not | the derived tag restarts, reads `uncertain_initial_value` (held), then false; the alarm would clear wrongly. That is why checkpointing is requested | |

Counter totals are safe regardless: `counter_total` is checkpointed and the delta in the next sample covers the downtime (tag-model s7).

A **long** downtime (the gateway is off for 3 h and the leak stops during it): after restart the alarm stays active until the first usable post-restart sample proves the condition false (with the tag state either way), then clears through the normal path. The journal's `engine_gap` row marks the uncertainty.

## 6. Scenario E: MQTT broker loss (Mosquitto crash) with the leak active

The broker is both the Sparkplug transport and the internal bus (README of the contracts). The alarm engine is in-process and keeps working.

| Time | Event | Result |
|---|---|---|
| 01:00:00 | Mosquitto dies. Driver connection leaves `connected`; every tag on it becomes `bad_not_connected` | `leak` alarm `held`; stays `active_unacked` |
| 01:00:00 | `meter01/health` goes `0` (liveness: `online` input bad) | `not_ok` raw true; `suppress_when` condition (connection not `connected`) is true, so `suppressed = true` at once |
| 01:00:30 | `_system/connections/local-sparkplug/state#not_connected` active after its 30 s delay | `active_unacked`, severity from config (high; 01 example overrides to `critical`). SMS via `system_health` route to Alice, and (per 01 override) `water_oncall`. **One** alarm for the link |
| 01:01:00 | `not_ok` on-delay satisfied | state machine runs: `active_unacked` with `suppressed = true`, journal `activated` (suppressed true), **no** notification. Shows in the HMI "suppressed" list with reason `connection_down` |
| 01:00-01:20 | the internal bus is down: no MQTT publishes, no retained alarm updates. **DB, WS (HMI), SMS modem continue**: the outbox and the SMS path do not use the broker (the reason for first-class `sms_modem`, README) | |
| 01:20:00 | broker back, driver reconnects, devices re-birth | `not_ok` raw condition false at 01:20:10 and it goes `cleared_unacked` (while still suppressed). Bus publisher republishes the current state of all alarms (07 s1.4) |
| 01:20:05 | `not_connected` clears (`cleared_unacked`); Alice acks later | |
| 01:20:40 | `suppress_release_delay` (30 s after the connection returned to `connected`) elapsed | `suppressed = false`; alarm was `cleared_unacked`, so it closes: `normal`, `acked_by system:unsuppress`, journal. Operator never got 40 offline alarms |
| after | `leak` evaluation resumes. The same `max_gap` question as scenario C applies: 20 min without observed increase | depends on tag engine |

If the flap is short (broker restart in 10 s), the link alarm's 30 s on-delay never completes and nothing is raised; the HMI shows `bad_not_connected` for 10 s. That is the intended debounce.

## 7. Scenario F: planned irrigation (opt-in suppression of the leak alarm)

An integrator may choose, for a customer whose meter feeds irrigation that runs for hours, to suppress `leak` during scheduled irrigation. The operator or schedule sets the memory tag `site1/pumphouse/irrigation_active` (bool, `good_manual`).

```yaml
instances:
  - name: meter01
    template: pulse_water_meter
    params: { connection: local-sparkplug, edge_node_id: esp32-meter01 }
    overrides:
      leak:
        alarms:
          leak: { suppress_when: [ { tag: site1/pumphouse/irrigation_active, equal: true } ] }
```

| Time | Event | Result |
|---|---|---|
| 06:00 | irrigation_active = true | no effect yet (alarm normal) |
| 08:00:50 | 2 h of continuous flow, `leak` true | alarm `active_unacked` with `suppressed = true`, journal `activated`, **no SMS**; shown in the suppressed list with reason `irrigation_active` |
| 09:00 | irrigation_active = false; flow continues (a real leak starts, or the valve is stuck) | `suppress_release_delay 30s` |
| 09:00:30 | suppression ends; alarm still `active_unacked` | **re-annunciated**: SMS "HIGH LEAK ... (unsuppressed)", tiers armed from 09:00:30 (05 s2.1 rule 5) |
| (alt) 09:00 | flow stops with irrigation | `leak` false 15 min later, alarm closes silently (cleared while suppressed, journaled) |

The risk is explicit: during irrigation, a real leak elsewhere on that meter's line is not announced until irrigation ends (up to 3 h in this example). That is a customer/integrator decision, hence opt-in and written in the rationale. The safer default is no suppression and a longer `leak_duration` for the irrigation instance (a parameter change, not a logic change).

## 8. L3 acceptance script

The harness (docs/09-testing) runs `skyrangerd` with a simulated Sparkplug device (fault injection per decisions.md item 7), a **fake modem** (PTY speaking a minimal AT subset: `AT`, `AT+CMGF`, `AT+CMGS`, `AT+CSQ`, `AT+CREG?`), an SMTP sink, an HTTP webhook sink and fake time where the engine supports it. Real-time scenarios use a test instance with short parameters: `leak_duration: 2m`, `leak_max_gap: 30s`, `heartbeat_stale: 10s`, route tiers `after: 0s` and `after: 1m`.

| ID | Inject | Expect |
|---|---|---|
| T1 leak happy path | pulses every 10 s for 2 m 10 s | within 5 s of `leak=true`: alarm `active_unacked`; fake modem got 1 SMS with `HIGH LEAK` and `#seq`; webhook got 1 `state_changed` with `cause: activated`; journal 1 `activated` |
| T2 escalate | no ack for 60 s | `escalated` journal row; SMS to `manager` and 1 email; SMS count for manager = 1 |
| T3 ack cancels | ack with correct `expected_seq` at 30 s | no tier 1 SMS; second ack returns `noop`; ack with stale `expected_seq` returns `409` |
| T4 clear | stop pulses | after `max_gap` plus one tick, alarm `normal` (acked) or `cleared_unacked` (unacked); CLEARED SMS sent |
| T5 offline | simulator drops the device (NDEATH) | `health 0`; `not_ok` active after 60 s (test: 5 s on-delay param); SMS once; NBIRTH returns: cleared |
| T6 chatter | drop and return device 4 times in 40 s | `chattering` true; exactly 1 chatter SMS; journal `chatter_edges`; alarm not shelved |
| T7 hold | leak active, kill device | leak alarm stays active, `held` true after hold; no CLEARED SMS; `not_ok` SMS sent |
| T8 broker loss | stop Mosquitto 60 s | one link alarm SMS; `not_ok` suppressed (no SMS); SMS still delivered while broker down (modem path); after return, no residual alarms except unacked link return |
| T9 restart | `kill -9` mid-leak after the first SMS | restart: same `seq` and `active_at`; no duplicate activation SMS; one `engine_gap` row; ack still works |
| T10 restart + tier | kill before tier 1, restart after it | tier 1 SMS sent once at startup with the restart note |
| T11 modem failure | fake modem returns `+CMS ERROR` | outbox `retry` with backoff; email path still delivers; `_system/alarms/notifier/sms_modem/failing` true after 3 failures plus 5 min (test: shortened); failing alarm routed via mail |
| T12 modem recovery | modem works again | queued SMS sent, prefixed `(delayed ...)`; activation and clear combined if the alarm already cleared |
| T13 DB outage | stop PostgreSQL during activation | spool written; SMS still sent; PostgreSQL restored: journal row present exactly once (idempotent replay), outbox status consistent |
| T14 shelve | shelve with 60 s duration and comment; leak active | no notifications while shelved; `shelved` list; at expiry re-annunciation SMS if still `active_unacked` |
| T15 irrigation | set `irrigation_active` then drive T1 | alarm active and suppressed, no SMS; clear flag with flow still on: SMS `unsuppressed` 30 s later |
| T16 heartbeat | run 10 min | webhook received >= 2 heartbeats; stopping the gateway leads the sink to flag absence |
| T17 config change | change `setpoint`-less alarm severity via draft | alarm object updated in place, `seq++`, `reconfigured` journal row, no spurious SMS |

Pass criteria for the first milestone: T1-T5 and T9 green; the rest tracked.

## Decisions made
1. The shipped template carries `class`, `rationale`, and for `not_ok` a `suppress_when` on connection state; the contract-only form remains valid.
2. `leak` has no `on_delay` and is not suppressed by default; suppression of it is an opt-in instance override with the risk written down.
3. A device dying during a leak holds the alarm, it does not clear it; the offline alarm carries the explanation.
4. A broker or link failure produces one alarm (the link); per-device offline alarms are suppressed and close silently on recovery.
5. The restart must not change state or resend notifications; downtime is shown by `engine_gap`.
6. The L3 script uses shortened parameters rather than fake time for the first milestone; fake clock is used at unit and engine level.

## Alternatives rejected and why
- **A rate threshold for the leak** (`flow_rate > x` for N minutes): 1 L pulses make small-window rate read zero in a slow leak (tag-model decision 8); the gap-based derived tag is the contract choice.
- **A single combined alarm "meter problem"** for offline and leak: different response times, owners and text; the operator needs to know which.
- **Clearing the leak alarm when the device goes offline**: indistinguishable from a repair.
- **Paging per offline meter during a broker failure**: 50 SMS for one cause.
- **Auto-ack of the device offline alarm on recovery**: the unacked return is the evidence that it happened; the operator closes it with one click.
- **Testing only with a mocked notifier**: the modem path is the point of the design; a PTY fake modem exercises the real dispatcher.

## Open questions
- `continuous_activity` semantics under bad or stale input (freeze vs gap) and checkpointing of its state across gateway restart: needed for scenarios C, D and E. Request to the tag-model owner.
- Should the offline alarm be `medium` and page at night? Integrator decision per customer; a `maintenance` route that sends offline alarms only to email during off-hours is possible with time-of-day routing, which is not in 01 yet.
- Exact `connection` state names and which count as `connected` for `suppress_when` (driver-interface.md section 3). 01 says `degraded` counts as connected.
- Suppression conditions that reference `_system` tags (needs the clarification raised in 01).
- Whether the modem approach has a real-hardware gotcha (SIM PIN, registration after boot): to be verified with the first hardware, UNVERIFIED here.
- Sparkplug-specific: whether NDEATH is delivered when the broker (not the device) dies. It is not; the broker's Will mechanism cannot fire if the broker itself is gone, so loss of the broker is seen only through the driver connection (UNVERIFIED against the Sparkplug spec).
