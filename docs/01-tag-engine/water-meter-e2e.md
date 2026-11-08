# Water Meter End to End (acceptance walkthrough)

Status: design draft v0.1. This is the milestone 1 vertical slice seen from the tag engine: one ESP32 pulse-counter water meter (1 pulse = 1 liter, decision 6) through the pipeline to history, alarms, the bus and the dashboard. It uses the template and instance from [tag-model.md](../00-contracts/tag-model.md) section 11 and the device profile in [messages.md](../00-contracts/messages.md) section 2 **unchanged**, and extends the contract's six-row trace to the cases it does not cover. Mechanisms referenced: [registry.md](registry.md), [pipeline.md](pipeline.md), [derived.md](derived.md), [subscriptions.md](subscriptions.md), [writes.md](writes.md).

Each scenario below is written as a table that becomes one scenario file for the `enginetest` harness (go-design.md section 6). Times are `2026-11-07` gateway time unless stated. Numbers are exact consequences of the algorithms, not measurements.

## 0. Setup

Instance `site1/pumphouse/meter01`, connection `local-sparkplug`, `edge_node_id: esp32-meter01`, `initial_total_l: 1234567` (mechanical register at commissioning), `liters_per_pulse: 1.0` (`multiplier`), `heartbeat_stale: 150s`, `leak_max_gap: 15m`, `leak_duration: 2h`, `modulus: 4294967296`, `max_delta: 100000`.

**Compile result (registry revision 43).** Nine tags. Ranks (derived.md section 3):

| Tag (under `meter01/`) | Kind | Rank | Handle (illustrative) | Depends on |
|---|---|---|---|---|
| `raw/boot_count`, `raw/online`, `raw/pulse_count`, `raw/rssi_dbm`, `raw/uptime_s` | source | 0 | 1 to 5 | |
| `total_volume` | derived `counter_total` | 1 | | `raw/pulse_count` (count), `raw/boot_count` (epoch) |
| `health` | derived `liveness` | 1 | | `raw/uptime_s`, `raw/online` |
| `flow_rate` | derived `rate` | 2 | | `total_volume` |
| `leak` | derived `continuous_activity` | 2 | | `total_volume` |

`MaxRank` is 2 (plus `_system` tags at rank 0). All nine tags start with `null`, quality `bad_waiting_for_initial_data`. After apply, the driver host `Subscribe`s the five handles and the Sparkplug driver (host application) is waiting for the node.

## 1. Commissioning: first NBIRTH

The ESP32 has just booted and has **not** got SNTP time yet. Gateway receive time `10:00:00.120`. One MQTT message, so the driver delivers one `SampleBatch`: `boot_count=7`, `online=true`, `pulse_count=0`, `rssi_dbm=-61`, `uptime_s=2`. The metrics carry timestamps near 1970.

| Stage (pipeline.md) | Effect |
|---|---|
| 1-3 resolve, convert, scale | Handles resolve; all unscaled; `uint64`/`bool`/`int64` pass |
| 5 timestamp | `|ts_source - ts_gateway|` far over `10s` and before 2020: `ts = ts_gateway = 10:00:00.120`; `_system/connections/local-sparkplug/clock_skew_rejections` += 5 |
| 6-9 | Every tag changed (`null` to value): stored, `quality = good`, dependents marked dirty |
| 10-11 | History rows for tags with history enabled (not `raw/uptime_s`); propagate to bus (retained topics) and WS |
| flush rank 1 | `total_volume`: no state, first sample: baseline, `total = 1234567`, `good`, state checkpointed; `health`: online true and heartbeat sampled now: `2` |
| flush rank 2 | `flow_rate`: buffer has one point (fewer than `min_samples`): `0.0`, `uncertain_initial_value`; `leak`: no activity yet: `false`, `good` |

The first value of `total_volume` is the register value you commissioned with, not 0 and not the device's pulse count. That is `initial_total_l`; it applies once (derived.md 6.1.5).

## 2. Normal flow: 12 L/min

Pulses every 5 s from 10:00:05. One `NDATA` per pulse, one batch each (the device rate limit is 1 message/s). Each batch: `pulse_count` changes, `counter_total` runs (delta 1, `+1.0 L`), then `rate` and `continuous_activity` run in rank 2 in the same flush.

| Time | `pulse_count` | `total_volume` | `flow_rate` | `leak` |
|---|---|---|---|---|
| 10:00:05 | 1 | 1234568 | 2 points, span about 5 s: about `12.0`, `uncertain_initial_value` | false |
| 10:00:30 | 6 | 1234573 | `6/30*60 = 12.0`, `uncertain_initial_value` | false (`active_since = 10:00:05`, 25 s) |
| 10:05:00 | 60 | 1234627 | window filled: `60/300*60 = 12.0`, **`good`** | false |

Notes:
- `flow_rate` leaves `uncertain_initial_value` when the buffer reaches back `window` (5 m), at 10:05:00. The values are right earlier; the quality says the window was short.
- Every 10 s the `eval_interval` timer also evaluates `rate`, adding flat points while the source is live (derived.md 6.2).
- Each accepted change goes through deadband (none on `total_volume`), is offered to history, queued to the alarm ring if alarms exist, and dropped into the WebSocket mailboxes. A dashboard subscribed with `min_interval: 500ms` sees at most one update per tag per 500 ms.
- `leak` stays false: `continuous_activity` needs 2 h of unbroken activity.

## 3. Flow stops, heartbeats continue ("no flow")

Last pulse at 10:05:00 (`pulse_count = 60`). The device sends a heartbeat `NDATA` every 60 s: `uptime_s` (changes), `pulse_count` (unchanged), `rssi_dbm` (usually unchanged).

| Event | Engine behavior |
|---|---|
| Heartbeat `pulse_count=60` | Unchanged repeat: no store, no history, no propagation (stage 8); `stale_after` timer reset; `counter_total` is evaluated (`OnSample`), computes delta 0, output unchanged |
| Heartbeat `uptime_s` | Changed: stored (history disabled for it), `health` evaluated: `2` |
| 10:05:10 timer | `rate` window start 10:00:10: 58 pulses inside: about 11.6 L/min and falling by 0.2 for each pulse that leaves the window |
| 10:10:00 timer | window `[10:05:00, 10:10:00]` has no pulses: **`0.0`, quality `good`** (history row via deadband 0.1 when it crosses) |
| 10:20:00 | gap since last increase exceeds `leak_max_gap` (15 m): `active_since` cleared. `leak` was never true |

State at 10:15: `flow_rate = 0.0 good`, `health = 2`, `pulse_count` quality `good` (never stale: heartbeats reset `stale_after = 150s`). This is **"no flow"**: the meter is alive and reports zero. A history row for `total_volume` is still written at `max_interval: 15m` (flat line, not a gap).

## 4. Dead device: NDEATH

10:20:00 power cut. The broker publishes the Last Will `NDEATH`. The Sparkplug driver delivers one batch: `@online = false` (`good`) and every node metric with quality `uncertain_last_known` and its last value.

| Tag | Result | Why |
|---|---|---|
| `raw/pulse_count`, `raw/boot_count`, `raw/uptime_s`, `raw/rssi_dbm` | last value, `uncertain_last_known` | sample quality uncertain; link is up, device quiet |
| `raw/online` | `false`, `good` | driver pseudo-metric |
| `total_volume` | `1234627`, `uncertain_last_known` | worst input; no delta |
| `flow_rate` | `0.0`, `uncertain_last_known` | `rate` frozen (derived.md 6.2); quality inherited |
| `leak` | `false`, `uncertain_last_known` | frozen |
| `health` | **`0` (offline), quality `good`** | `online == false`; `liveness` quality is absolute |
| alarm `health/not_ok` | `active_unacked` at 10:21:00 | `not_equal 2` held for `on_delay 60s` |

**The two cases the HMI must distinguish** (decision 6, heartbeat):

| | `flow_rate` | `health` | `raw/*` quality | HMI |
|---|---|---|---|---|
| No flow (10:15) | `0.0` `good` | 2 | `good` | "0 L/min", normal |
| Dead device (10:21) | `0.0` `uncertain_last_known` | 0 | `uncertain_last_known` | greyed value, "device offline", alarm |

## 5. Reboot and rebirth

10:22:00 the ESP32 is back. NVS `boot_count` is 8. `NBIRTH`: `boot_count=8`, `pulse_count=3` (3 pulses since boot), `online=true`, `uptime_s=14`, new `bdSeq`. One batch.

| Step | Result |
|---|---|
| Raw tags | `good` again; `boot_count 7 to 8`, `pulse_count 60 to 3` |
| Flush rank 1, `total_volume` | epoch changed and a count sample is present: reset path, `delta = raw = 3` (`reset_baseline: zero`), `total = 1234627 + 3 = 1234630` |
| Event | `skyranger.tag.counter_discontinuity.v1` `{kind: "reset", prev_raw: "60", new_raw: "3", prev_epoch: "7", new_epoch: "8", delta_applied: 3.0, total_after: 1234630.0}`; written to `events` in the same transaction as the checkpoint (derived.md section 7), then published to the bus |
| `health` | online true, heartbeat sampled now: `2`; alarm `not_ok` goes `cleared_unacked` |
| `flow_rate` | recovery: a point `(10:22:00, 1234630)` is added with no step before it, so the 3 L is spread over the unobserved gap since the last live point; window start 10:17:00 total `1234627`: `0.6 L/min`, quality `good` (matches the contract trace) |
| `leak` | increase after a gap over 15 m: `active_since = 10:22:00` (a catch-up counts as activity; conservative) |

Counts between the last sample received from the old epoch and the actual reboot are lost; the event marks the period (messages.md section 6). Reports flag it as possibly under-counted by at most the pulses in that interval. What if `boot_count` and `pulse_count` arrived in two batches? `PendingEpoch` makes the result identical (derived.md 6.1.1): epoch first holds the total with `uncertain`; the count sample then applies one delta of 3.

A reboot **without** a flow change and with count 0 to 0 (idle meter reboots): epoch changes, the count sample is unchanged but `OnSample` delivers it: `delta = 0`, event `reset` with `delta_applied: 0.0`, total unchanged.

## 6. Rollover and implausible jumps

Simulator `sim://pulse_meter?start_count=4294967290` (fault: wrap), epoch constant:

| Raw `pulse_count` | Rule | `delta` | Event |
|---|---|---|---|
| 4294967294 (first after baseline 4294967290) | normal | 4 | none |
| 4294967295 | normal | 1 | none |
| 3 | `raw < last_raw`, `modulus > 0`, epoch unchanged | `(4294967296 - 4294967295) + 3 = 4` | `rollover` `{prev_raw: "4294967295", new_raw: "3", delta_applied: 4.0}` |

A real wrap on a 1 L/pulse meter needs 4.29 million m3: about 13.6 years at a constant 10 L/s (4,294,967,296 L / 10 L/s). It is a test case, not an operations case; the same template can serve a faster-pulsing meter, where 1,000 pulses/s would wrap in about 49.7 days.

Glitch: firmware bug sends `pulse_count = last_raw + 2,000,000` in one message. `max_delta = 100000` exceeded: `delta = 0`, event `implausible` (`prev_raw`, `new_raw`), output quality `uncertain` for that sample, `last_raw` updated to the new raw, so the next normal sample counts from there. If the jump was real, those counts are recorded in raw history and the event; the total is not corrupted by an obvious fault.

## 7. Broker loss (the device is fine)

14:00:00 Mosquitto restarts; back 14:03:00. The device keeps counting (36 L in 3 min). The Sparkplug driver reports `disconnected`, then `connected`.

| Time | Tag result |
|---|---|
| 14:00:00 | Overlay `bad_not_connected` on all five raw tags, values held. `total_volume`, `flow_rate`, `leak`: `bad_not_connected` (worst input), values held. `health`: `online` input is bad: `0`. Alarm `not_ok` activates at 14:01:00 (the gateway cannot see the device; the same alarm covers both causes, and the raw quality tells them apart) |
| 14:03:00 connected | Overlay cleared **but quality stays bad** until each tag's own sample (pipeline.md section 8) |
| 14:03:02 | Device reconnects, publishes `NBIRTH` (new `bdSeq`, **same** `boot_count 8`): `pulse_count` is `previous + 36`. Same epoch, so normal delta 36: `total_volume` gains all 36 L, nothing lost. `flow_rate` spreads them across the outage (36 L smeared over the gap and averaged over the 5 min window: about 7.2 L/min, quality `good`) instead of a spike. `health 2`, alarm clears |

An MQTT reconnect never looks like a reset because `bdSeq` is not used as an epoch (messages.md section 2).

## 8. Gateway restart

16:00:00 `skyrangerd` stops (SIGTERM): `FlushNow` commits `total_volume` and `leak` state. Restart at 16:00:20.

| Step | Result |
|---|---|
| Startup (go-design.md section 5) | Registry compiled; `counter_total` state restored: `last_raw`, `last_epoch = 8`, `total`. `leak` state (`last_increase`, `active_since`) restored |
| First seconds | All raw tags `bad_waiting_for_initial_data` (null). `total_volume`: restored value, quality `bad_waiting_for_initial_data` (inherited), shown greyed. Driver connects; meter's next `NDATA` arrives before any `NBIRTH`: the driver requests a rebirth (messages.md section 1, at most once per 30 s) |
| `NBIRTH` (about 16:00:23) | Raw tags `good`. `total_volume`: same epoch, `raw - last_raw` includes every pulse counted during the 23 s of downtime: nothing lost. `health 2` |
| `flow_rate` | Buffer is not persisted: a fresh warm-up, `uncertain_initial_value` until 5 minutes of data. Honest, not a bug |
| `leak` | Persisted streak continues if the downtime was shorter than `leak_max_gap`; otherwise it restarts |

If PostgreSQL was down during the previous run so the last checkpoint is older than the last second: the restored `(last_raw, total)` is an older, consistent pair, and the first sample applies the full difference. The only loss is a device reboot inside that window (derived.md section 7).

## 9. Manual register alignment

17:00 an engineer compares the mechanical register (1,234,900) with `total_volume` (1,234,877).

| Step | Result |
|---|---|
| `POST /tags/{total_volume id}/write {"value": 1234900.0, "comment": "align to register"}` | Role operator. Validation in the loop: writable (`writable: true` on the template tag, `counter_total` has write semantics), float64, no `write_limits`. `202 {write_id, pending}` |
| In the loop | `counter_total.Write`: `total = 1234900`; event `manual_adjust` `{prev_raw == new_raw == last_raw, delta_applied: 23.0, total_after: 1234900.0}`; synchronous checkpoint including the event; on commit the write completes `ok` and `write_completed` is emitted |
| Value | Published once with quality `good_manual`; next pulse returns it to `good` |
| Dependents | `Generation` bumps: `rate` clears its buffer and warms up again (`uncertain_initial_value`); `continuous_activity` ignores the step (an alignment is not activity) |
| History | `total_volume` row jumps by 23 L. A `delta` aggregate for the day includes it; the `manual_adjust` event is listed next to the report, so the discrepancy is explained |

## 10. Slow leak the rate cannot see

From 18:00 a dripping fixture produces one pulse every 10 minutes after a long quiet period (last activity before that is more than 15 min ago).

| Time | Event | `flow_rate` | `leak` state |
|---|---|---|---|
| 18:00:00 | pulse 1 | `0.2`, then back to `0.0` when it leaves the 5 min window | `active_since = 18:00:00`, `last_increase = 18:00:00` |
| 18:10:00 | pulse 2 | same flip, `good` | gap 10 min is under 15 min: streak continues |
| ... | every 10 min | 0.0 or 0.2 | unchanged `active_since` |
| 20:00:00 | pulse 13 | | `now - active_since = 2h`: next `eval_interval` tick (at most 30 s later) sets **`leak = true`** |
| 20:00:30 | | | alarm `leak/leak` (`equal true`, no delay): `active_unacked`, severity `high` |

`flow_rate` is `good` and reads 0.0 to 0.2 L/min the entire time: with 1 L pulses a 5 minute window cannot resolve this. This is why the contract uses `continuous_activity` (tag-model.md decision 8). A normal daily pattern (taps in bursts with gaps over 15 minutes) never accumulates 2 h.

## 11. What lands in history, bus and the dashboard

| Tag | History (default for template) | Bus topic (retained, QoS 0) |
|---|---|---|
| `raw/pulse_count`, `raw/boot_count`, `raw/online` | on change, `max_interval 15m` | `.../raw/pulse_count` etc. |
| `raw/uptime_s` | none | published, not stored |
| `raw/rssi_dbm` | on change, deadband 3, `max_interval 1h` | published |
| `total_volume` | on change, `max_interval 15m` | `skyranger/v1/gw01/tags/site1/pumphouse/meter01/total_volume` |
| `flow_rate` | on change with absolute deadband 0.1, `max_interval 15m` | published after the same deadband |
| `health`, `leak` | on change, `max_interval 15m` | published |

Daily consumption for a report: `GET /api/v1/history?id=<total_volume>&agg=delta&interval=1d` (api.md). It uses `total_volume`, so reboots, rollovers and alignments are already inside the series; the discontinuity events in the same day explain any anomaly. The dashboard subscribes by prefix `site1/pumphouse/meter01` and renders: `total_volume` (L, `good` shows black, anything else greyed with the quality name), `flow_rate`, `health` via `enum_labels` (`0 offline`, `1 stale`, `2 ok`), `leak`, and the alarm banner.

## 12. Acceptance checklist

Each line is an assertion in the scenario suite; the water meter slice is done when all pass against the real stack (Sparkplug driver, Mosquitto, PostgreSQL) driven by the simulator, and against `core` with a fake clock.

1. NBIRTH from an unsynced device: `ts = ts_gateway`, skew counter increments, first `total_volume == initial_total`.
2. Steady 12 L/min for 5 min: `flow_rate == 12.0` within one pulse (0.2 L/min), `good` from the window boundary.
3. Idle with heartbeats: `flow_rate` reaches `0.0 good`, `health == 2`, no history spam.
4. NDEATH: raw `uncertain_last_known`, `health == 0`, `flow_rate` and `total_volume` `uncertain_last_known`, alarm after 60 s.
5. Reboot with epoch change: exactly one `reset` event, `delta_applied == new_raw`.
6. Reboot where `boot_count` arrives before `pulse_count` in a separate batch: same total as in the same batch.
7. Idle meter reboot (count 0 to 0): one event, delta 0.
8. Rollover: `delta = 4`, one `rollover` event.
9. Implausible jump: total unchanged, one `implausible` event, `uncertain` for that sample.
10. Broker loss and recovery without reboot: all counts present after, no event, no flow spike.
11. Gateway restart (clean and with `kill -9`): total correct after the next sample; `leak` streak preserved when shorter than `max_gap`.
12. Manual alignment: durable before `ok`; event with signed delta; `rate` warm-up; failed commit leaves total consistent (writes.md section 5).
13. Slow leak: `leak == true` within `duration + eval_interval`; alarm raised; `flow_rate` never above 0.2.
14. Fault injection from the simulator (drops, stale values, bad quality, bursts): never a wrong `good` value, never a total decrease except `manual_adjust`.
15. Property test: for random event sequences, `total_volume` equals the model's expected total (go-design.md section 6).

## Decisions made
1. The contract's template and device profile are used unchanged; all new behavior lives in the engine.
2. The scenario tables double as the test specification, one scenario file each.
3. "No flow" (`0.0 good`, health 2) and "dead device" (non-good quality, health 0 or 1) are different states by construction and the HMI renders them differently.
4. A reconnect or catch-up never produces a rate spike (interpolation across the gap), and a reboot or rollover never produces a wrong total (epoch plus modulus plus plausibility limit).
5. The slow-leak case is handled by `continuous_activity`, not by lowering a rate threshold.

## Alternatives rejected and why
- **Showing `flow_rate` as 0.0 with `good` while the device is dead**: indistinguishable from no flow; the decaying value would invent information.
- **Treating any MQTT reconnect as a reset**: loses nothing but would double count the pre-reboot pulses (the device count continues); the epoch makes it unambiguous.
- **Writing the device's pulse count directly into the total on the first sample**: ignores the commissioning register; `initial_total_l` exists for exactly this.
- **Making the gateway ask the device for a rebirth on every restart before trusting data**: not needed because cumulative counts are self-correcting; the rebirth request on `NDATA` before `NBIRTH` is already in the driver contract.

## Open questions
- Heartbeat 60 s versus `stale_after` 150 s versus `not_ok` `on_delay` 60 s means a silent but connected device raises the alarm at about 3.5 minutes (150 s + 60 s); acceptable? (Contract numbers, restated for sign-off.)
- After a long broker outage, is spreading the catch-up evenly the right flow estimate? It is an estimate; the quality is `good`. Alternative: show `uncertain` for one window after any gap longer than N.
- Does a catch-up after reboot count as "activity" for `continuous_activity`? Chosen: yes, conservatively.
- The `not_ok` alarm fires for both device death and gateway-to-broker loss. Should broker loss use a separate connection-level alarm instead (so a broker restart does not page the plant for every meter)? Alarm doc.
- The contract trace shows `flow_rate` `good` right after rebirth; this design reproduces it through interpolation. If the contract owner prefers a warm-up after outages, `derived.md` section 6.2 changes in one place.
