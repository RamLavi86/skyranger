# 04 Fault Injection Catalogue

Every fault the platform must survive, with a stable ID, how it is injected, and the **expected behavior in contract terms** (quality codes from tag-model.md section 4, connection states from driver-interface.md section 3, events from messages.md section 4). The machine-readable index is `test/faults/faults.yaml`; CI computes a coverage report from it (section 4).

Tiers: **L1/L3** in-process (virtual time, every commit), **L4** docker compose (nightly), **L6** soak (randomized schedule), **L7** real hardware (nightly subset / release).

## 1. Injection mechanisms

| Mechanism | Tier | Faults it can produce |
|---|---|---|
| `clock.Fake` (`Advance`, `SetWall`) | L1, L3 | timeouts, staleness, clock jumps |
| In-memory transport with fault hooks (`pkg/sim/transport`) | L3 | drop, duplicate, reorder, delay, corrupt, disconnect, half-open |
| `skyranger-sim` timeline and control API | L3, L4, L6 | all device-side faults |
| **Toxiproxy** (Shopify, TCP proxy with toxics: latency, bandwidth, slow_close, timeout, slicer, limit_data, reset_peer) placed between gateway and broker, gateway and PLC sims, gateway and PostgreSQL | L4, L6 | network faults on any TCP link. Toxic names from memory, UNVERIFIED |
| `docker compose stop/kill/pause/restart` | L4, L6 | process crash, hang (pause = SIGSTOP), restarts |
| Privileged `netem` sidecar sharing the gateway's network namespace (`tc qdisc ... netem loss/delay/duplicate/reorder`, `iptables -j DROP`) | L4, L7 | packet-level faults Toxiproxy cannot make (true half-open, packet reorder, duplicate) |
| Small loop/tmpfs filesystems mounted at the data dirs (`--tmpfs size=`, or loop device) | L4 | disk full |
| `chronyd`/`date -s` in privileged container, or on the HIL box | L4, L7 | wall clock jumps |
| Relay-controlled power (file 08) | L7 | power loss, brownout loops, mid-write cuts |
| Managed switch port shutdown / cable pull relay | L7 | link loss, Wi-Fi loss for ESP32 (AP power relay) |

## 2. The catalogue

Columns: **ID**, fault, injection, expected behavior (pass criteria), tiers.

### 2.1 Network and connection (F-NET)

| ID | Fault | Injection | Expected | Tiers |
|---|---|---|---|---|
| F-NET-01 | Clean connection drop gateway-to-broker | Toxiproxy disable / transport close | Sparkplug connection `disconnected`; every tag on it `bad_not_connected` within 1 s of detection; `connection.state_changed.v1` emitted; reconnect with backoff; on reconnect gateway republishes `STATE` online and requests or receives births; tags back to `good` | L3, L4, L7 |
| F-NET-02 | **Half-open TCP** (peer gone, no FIN/RST) | netem `iptables DROP` both ways; transport `Blackhole` | Detected within the driver's documented bound (<= 60 s, file 02 DC-CONN-04); then as F-NET-01. Never shows `good` + stale data indefinitely | L3, L4, L7 |
| F-NET-03 | Connection reset (RST) mid-message | Toxiproxy `reset_peer` | as F-NET-01; no panic; partial protobuf discarded | L4 |
| F-NET-04 | High latency (500 ms, jitter 200 ms) | Toxiproxy `latency` | no state change; values arrive late but `ts` from device if within skew; WS clients see values within latency + 200 ms | L4 |
| F-NET-05 | Bandwidth limit (64 kbit/s, cellular-like) | Toxiproxy `bandwidth` | no crash, backpressure at broker; `dropped_samples` may rise; counters stay exact (cumulative) | L4, L6 |
| F-NET-06 | Flapping link: down 2 s / up 5 s for 10 min | Toxiproxy schedule | bounded reconnect rate; no goroutine/memory growth; no duplicate alarm activations beyond actual state changes; total conserved | L4, L6 |
| F-NET-07 | Modbus device stops answering one unit id | sim point fault timeout | per-item `bad_comm_failure`, connection `degraded`; other unit ids unaffected | L3, L4 |
| F-NET-08 | Modbus exception response (code 02 illegal address) | sim | item `bad_config_error` or `bad_device_failure` per driver mapping doc; `Details` has `modbus_exception` | L3 |
| F-NET-09 | DNS name of broker fails to resolve | compose network alias removed | `connecting`/`disconnected`, retry, no crash; `last_error` set | L4 |

### 2.2 Device behavior (F-DEV)

| ID | Fault | Injection | Expected | Tiers |
|---|---|---|---|---|
| F-DEV-01 | **ESP32 reboot (power cycle)** | sim `reboot`; HIL relay | NDEATH (Will) -> node tags `uncertain_last_known`, `@online` false, `health` 0; new NBIRTH with `boot_count+1`, `pulse_count` small; `counter_discontinuity{kind: reset}` once; `total_volume` = oracle exactly; `health` back to 2 | L3, L4, L7 |
| F-DEV-02 | MQTT reconnect **without** reboot | sim `network_down/up` | new `bdSeq`, same `boot_count`, `pulse_count` continues: **no** discontinuity event; total conserved | L3, L4, L7 |
| F-DEV-03 | Device silent but link up (hung firmware, no Will yet) | sim stop publishing, keep TCP | after `stale_after` (150 s): raw tags `uncertain_last_known`; `health` 1 (stale); then broker keepalive timeout fires Will: `health` 0 | L3, L4 |
| F-DEV-04 | Device dies without Will (broker never notices, e.g. half-open on device side) | netem on device link | `health` 1 at 150 s; stays 1 until broker detects; `health/not_ok` alarm after `on_delay` 60 s | L4, L7 |
| F-DEV-05 | Reboot loop (brownout) 10 x in 5 min | sim; HIL relay | 10 reset events (one per epoch change seen; reboots with no delivered sample between them may collapse, oracle accounts); no crash; health alarm once (not 10 activations if never back to ok for `on_delay`) | L3, L7 |
| F-DEV-06 | `boot_count` NVS corrupted: goes backwards or resets to 0 | sim `set boot_count` | epoch differs -> treated as reset (contract compares `!=`); one reset event; total conserved by oracle rule | L3 |
| F-DEV-07 | Firmware update changes metric set | sim device with new birth | `discovery.device_changed.v1`; existing tags untouched; new metric shown unmapped | L3, L4 |
| F-DEV-08 | Unapproved device publishing | sim new node | discovery record `pending`; **no tag, no history row** created (assert via `/api/v1/tags` and history count) | L3, L4 |
| F-DEV-09 | Two devices claim the same `edge_node_id` | sim duplicate | each connect kicks the other (MQTT client-id clash if same client id) or births alternate; gateway must not crash; discontinuity events appear and an operator-visible warning (UNVERIFIED desired behavior, see Open questions) | L4 |

### 2.3 Values and quality (F-VAL)

| ID | Fault | Injection | Expected | Tiers |
|---|---|---|---|---|
| F-VAL-01 | **Stale value**: device keeps heartbeats off, data unchanged | sim `stale` | after `stale_after`, `uncertain_last_known`, last value held; history `max_interval` rows still written (with that quality) | L1, L3 |
| F-VAL-02 | **Bad quality from source** (OPC UA bad status, sim `bad_quality`) | sim | quality copied; derived outputs worst-quality with tie rule; aggregates exclude bad samples; quality alarm activates after `on_delay` | L1, L3 |
| F-VAL-03 | NaN / Inf float | sim `nan` | `bad_type_mismatch` and `null` value (see file 01 OQ3); NaN never reaches JSON, DB or WS (assert serialized output) | L1, L3 |
| F-VAL-04 | Type mismatch (string where float expected; uint64 > int64 max into int64) | sim | `bad_type_mismatch`, no truncation | L1, L3 |
| F-VAL-05 | Out of engineering range | sim | `uncertain_eng_range_exceeded` (or clamped if `clamp: true`) | L1 |
| F-VAL-06 | **Burst**: 1000 updates/s for 10 s on 1000 tags | sim `burst` | no unbounded memory; coalescing; `dropped_samples` counted; quality transitions all delivered; WS `seq` contiguous or explicit `resync`; counters exact | L3, L4, L6 |
| F-VAL-07 | **Duplicate messages** (QoS 1 redelivery, same seq) | transport `duplicate`; netem duplicate | no false seq gap, no double count (cumulative), events de-duplicated by `id` at consumers | L3, L4 |
| F-VAL-08 | **Out-of-order messages** | transport `reorder` | late samples go to history only and do not replace current value (tag-model.md section 5); counter not over-counted (depends on contract fix, file 01 OQ1); Sparkplug seq out-of-order treated as gap -> rebirth | L1, L3 |
| F-VAL-09 | Sparkplug `seq` skip | sim `seq_skip` | file 02 SP-04 | L3 |
| F-VAL-10 | Unknown alias in NDATA | sim `bad_alias` | file 02 SP-09 | L3 |
| F-VAL-11 | Malformed protobuf / truncated payload | sim `corrupt` | message dropped, counter `_system/...` (error counter) increments, node marked `uncertain_last_known`, rebirth requested; no panic | L3, L4 |

### 2.4 Counters (F-CNT)

| ID | Fault | Injection | Expected | Tiers |
|---|---|---|---|---|
| F-CNT-01 | **Counter rollover** at 2^32 | sim `set pulse_count` near 2^32 | one `rollover` event; total exact | L1, L3 |
| F-CNT-02 | Rollover while a sample is lost | rollover + drop the sample right before wrap | still one rollover, total exact (delta computed across wrap) | L1, L3 |
| F-CNT-03 | Implausible jump (bit flip, wiring noise) | sim `set` +10^6 | `implausible`, delta 0, that sample `uncertain`, next normal sample counts from the new raw (see Open questions) | L1, L3 |
| F-CNT-04 | Reboot and count back past old value before first sample | sim: reboot, 100 pulses offline-buffered? (device counts while MQTT down, then connects) | epoch change detected via `boot_count`; delta = new raw (baseline zero); exact | L1, L3 |
| F-CNT-05 | Gateway restart (state from checkpoint) during flow | kill `skyrangerd` mid-flow | after restart, total equals oracle exactly (cumulative counter + checkpointed `last_raw`) | L3 (simulated restart), L4, L7 |
| F-CNT-06 | Gateway down across a device reboot | stop gateway, reboot device, start gateway | reset detected from epoch change; loss bounded per contract rule; exact vs oracle | L4 |
| F-CNT-07 | Manual total write | API write | `manual_adjust` event, audit row, total equals written value, subsequent deltas add | L3, L5 |

### 2.5 Time (F-CLK)

| ID | Fault | Injection | Expected | Tiers |
|---|---|---|---|---|
| F-CLK-01 | ESP32 unsynced, reports 1970 | sim `clock.sntp_synced_after` | `ts = ts_gateway`; `clock_skew_rejections` increments; history not polluted with 1970 rows | L1, L3 |
| F-CLK-02 | Device clock drift beyond `max_clock_skew` | sim `drift_ppm` large | switch to `ts_gateway` per sample; counter increments | L1, L3 |
| F-CLK-03 | **Gateway wall clock jumps forward 1 h** (bad NTP, manual change) | `clock.Fake.SetWall`; L7 `date -s` | timers based on monotonic time do not fire en masse (no false stale, no false leak alarm); history has a 1 h forward gap but no crash; `rate` does not spike (uses monotonic or sample ts consistently, engine doc must decide); event logged (proposal: `_system/gateway/clock_jumps` counter) | L1, L3, L7 |
| F-CLK-04 | **Gateway wall clock jumps backward 10 min** | as above | new samples have `ts` older than current value: by contract they become history-only, so **current values freeze for 10 min**. This is a contract consequence that must be tested and probably fixed (see Open questions); minimum pass: no crash, no data loss in history, no duplicate key failure on `(tag_id, ts)` insert | L1, L3, L7 |
| F-CLK-05 | DST change at gateway timezone | `Gateway.spec.timezone` Europe/London, run across last Sunday of October | daily `delta` aggregation buckets 23 h / 25 h correct; storage unaffected (UTC) | L1, L3 |
| F-CLK-06 | Gateway boots with no RTC battery (time 1970 or build time) before chrony sync | L7 | gateway must not write history until clock plausible (proposal), or must mark; at minimum detect and alarm | L7 |

### 2.6 Storage and database (F-DB)

| ID | Fault | Injection | Expected | Tiers |
|---|---|---|---|---|
| F-DB-01 | **PostgreSQL down 10 min** | `docker compose stop timescaledb`; Toxiproxy | tag engine, alarms, WS live values keep working (tag engine never blocks on historian, contract section 6); historian spills to local WAL; `/system/health` `degraded` with `database` not ok; config apply returns error (503-style); after DB back: WAL drained, **zero history rows lost** vs oracle | L4, L6, L7 |
| F-DB-02 | DB down longer than WAL capacity | DB stop + small WAL limit | documented behavior (historian doc owns: drop oldest vs stop accepting); gauge/alarm raised; no crash; never silent | L4 |
| F-DB-03 | DB slow (2 s per query) | Toxiproxy latency | WAL absorbs, API history queries slow but bounded by timeouts, no tag engine stall | L4 |
| F-DB-04 | DB connection reset mid-COPY | Toxiproxy reset_peer | batch retried from WAL; no duplicates (idempotent insert on `(tag_id, ts)` or dedupe), no loss | L4 |
| F-DB-05 | **Disk full** on the data volume (DB and WAL) | tmpfs/loop device of fixed size, fill with `fallocate` | `skyrangerd` stays up, live values continue, alarm/health raised (`_system/gateway/disk_free`), writes rejected cleanly; after space freed, recovery without manual DB repair | L4, L7 |
| F-DB-06 | Disk full on the log volume | as above | logging drops, process unaffected | L4 |
| F-DB-07 | **Power loss mid-write** | L7 relay during heavy history write and during config apply | after boot: PostgreSQL recovers by itself (WAL replay); config revision is either N or N+1, never partial; `counter_total` state consistent (checkpoint atomic); historian WAL file truncated at last valid record, no crash on corrupted tail | L7 (plus L4 approximation with `docker kill -s KILL`) |
| F-DB-08 | DB restored from backup (older) | restore older dump | gateway notices config revision mismatch; documented behavior (deployment doc) | L4 release |

### 2.7 Broker (F-BRK)

| ID | Fault | Injection | Expected | Tiers |
|---|---|---|---|---|
| F-BRK-01 | **Broker restart** (clean, persistence on) | `docker compose restart mosquitto` | all Sparkplug tags `bad_not_connected` during outage; devices and gateway reconnect; births; values `good`; retained internal bus topics restored (republished by `skyrangerd` on reconnect); history continuity: no missed counts (cumulative), gap marked by quality rows | L4, L6, L7 |
| F-BRK-02 | Broker restart without persistence (retained lost) | restart with empty volume | gateway republishes `STATE` and internal retained topics; nothing depends on stale retained data | L4 |
| F-BRK-03 | Broker killed (SIGKILL) | `docker kill` | as F-BRK-01; device Wills are **not** delivered (broker died), so gateway must rely on its own disconnect detection, not NDEATH | L4 |
| F-BRK-04 | Broker paused (SIGSTOP) 90 s | `docker pause` | half-open from both sides; detection by keepalive; as F-NET-02 | L4 |
| F-BRK-05 | Broker ACL rejects gateway (misconfig) | wrong credentials | Sparkplug connection `faulted`, `bad_config_error`, no reconnect loop | L4 |

### 2.8 Gateway processes (F-GW)

| ID | Fault | Injection | Expected | Tiers |
|---|---|---|---|---|
| F-GW-01 | Driver process crash | `kill -9` the driver PID | file 02 DC-LIFE-01; `restarts` counter; other connections unaffected | L3 (in-process sim of host), L4 |
| F-GW-02 | Driver hang | SIGSTOP driver PID | killed after 3 failed Health calls | L4 |
| F-GW-03 | Driver memory leak (test driver allocates) | test-only driver | process killed by memory limit (cgroup/rlimit per driver: security/deployment doc), restarted; gateway fine | L4 |
| F-GW-04 | `skyrangerd` SIGKILL | `docker kill` | container restart policy brings it back; `counter_total` from checkpoint; alarms state restored; WS clients reconnect and resync | L4, L7 |
| F-GW-05 | `skyrangerd` OOM | memory limit in compose | as F-GW-04; post-mortem: perf test should have prevented it | L4 |
| F-GW-06 | Config apply with driver `ValidateConfig` failure | API | revision rejected, no change, no restart of running connections | L3 |
| F-GW-07 | Config apply under load | apply while F-VAL-06 burst | unchanged tags keep state; no lost counts | L3, L4 |

### 2.9 HMI and API clients (F-API)

| ID | Fault | Injection | Expected | Tiers |
|---|---|---|---|---|
| F-API-01 | Slow WS consumer | test client stops reading | `resync{reason: overflow}` then snapshot; other clients unaffected; server memory bounded (1 MiB per connection) | L3, L4 |
| F-API-02 | WS connection drop / gateway restart | proxy cut | HMI reconnects with backoff, shows "connection lost" banner, values greyed, then fresh snapshot | L5 |
| F-API-03 | Token revoked while connected | API logout | WS closed with `4401` | L3, L5 |
| F-API-04 | 20+ connections from one user | test client | `4429` on the 21st | L3 |

## 3. Expected-behavior assertions as code

Each fault test asserts through the public surfaces only. Helper sketch (L3 harness):

```go
func TestFault_F_DEV_01_ESP32Reboot(t *testing.T) {
    h := harness.New(t, harness.Scenario("test/scenarios/wm-trace-contract.yaml"), harness.Seed(1))
    meter := h.Sim.Device("meter01")
    base := "site1/pumphouse/meter01/"

    h.RunUntil(10 * time.Minute)
    h.Sim.Reboot(meter, sim.OfflineFor(2*time.Minute))

    h.Eventually(2*time.Minute, func(c *harness.C) {
        c.TagQuality(base+"raw/pulse_count", tag.QualityUncertainLastKnown)
        c.TagValue(base+"health", int64(0))
    })
    h.RunFor(5 * time.Minute)

    h.AssertTagEqualsOracle(base+"total_volume", meter.Oracle().ExpectedTotal())
    h.AssertEventCount("skyranger.tag.counter_discontinuity.v1",
        harness.Subject(base+"total_volume"), harness.DataField("kind", "reset"), 1)
    h.AssertTagValue(base+"health", int64(2))
    h.AssertNoEvent("skyranger.tag.counter_discontinuity.v1", harness.DataField("kind", "implausible"))
}
```

`h.Eventually` advances the fake clock in small steps and re-checks, failing with the full timeline of the tag (value, quality, ts) if the condition never holds. No real sleeps.

## 4. Machine-readable index and coverage

```yaml
# test/faults/faults.yaml (protected)
- id: F-DEV-01
  title: ESP32 reboot (power cycle)
  severity: critical          # critical faults must be covered at L3 AND (L4 or L7) before release
  tiers_required: [L3, L4, L7]
  contract_refs: [tag-model.md#7, messages.md#2, messages.md#6]
- id: F-CLK-04
  title: Gateway wall clock jumps backward
  severity: high
  tiers_required: [L1, L3, L7]
  known_issue: "contract freezes current values; see 04-fault-injection.md OQ"
```

Tests declare what they cover with a Go comment directive `//fault:F-DEV-01` above the test function (or `faults_covered` in scenario metadata). `tools/testguard faults` cross-references and prints:

```
fault      severity  L1  L3  L4  L6  L7  status
F-DEV-01   critical  -   ok  ok  ok  ok  covered
F-CLK-04   high      ok  ok  -   -   --  MISSING L7 (required)
```

Release gate: every `critical` and `high` fault is covered at all its `tiers_required`, and passed in the release run (file 10). The table is also published in the evidence pack (file 11).

## 5. Randomized fault schedules (soak)

In L6, a **chaos scheduler** picks faults from the catalogue entries tagged `soak_safe: true` (recoverable without operator action) using a logged seed, with Poisson inter-arrival (mean 20 min) and a cool-down between faults so each recovery is measurable. Faults that need operator action (F-BRK-05, F-DB-02) are excluded from soak and tested deterministically.

## Decisions made
1. Every fault has an ID, a severity, required tiers, and contract-referenced expected behavior; the YAML index plus `//fault:` directives produce a coverage report that gates releases.
2. Expected behavior is asserted only through public surfaces (REST, WS, events, internal bus), never by reading internal state.
3. Toxiproxy for TCP-level faults, a privileged netem/iptables sidecar for packet-level faults (true half-open), compose for process faults, tmpfs/loop devices for disk full, relays for power on real hardware.
4. Soak uses a seeded random schedule of only self-recovering faults.

## Alternatives rejected and why
- **Chaos Mesh / Litmus (Kubernetes chaos tools)**: we do not run Kubernetes on the edge; compose plus Toxiproxy plus netem covers the same faults with less machinery.
- **Pumba (Docker chaos tool)**: workable, but netem via a sidecar is the same mechanism with fewer layers to debug. Can be revisited.
- **Only testing faults manually before releases**: not repeatable, not evidence.

## Open questions
1. **Contract issue (F-CLK-04).** tag-model.md section 5 says a sample with `ts` older than current `ts` is history-only. After a backward wall-clock jump on the gateway, every new sample is "older" for the jump duration, so live values freeze and alarms stop evaluating. Proposal for the tag-model owner: compare by monotonic receive order for current-value replacement when `ts` was taken from `ts_gateway`, and only apply the "older ts" rule to `ts_source`-derived timestamps.
2. **Contract issue (F-CNT-03).** After an `implausible` sample the algorithm sets `last_raw` to the implausible raw. If the jump was a single corrupted reading, the next correct sample is lower and is treated as rollover or reset, producing a second wrong delta. Is that intended? An alternative is to not advance `last_raw` on implausible samples unless N consecutive samples agree.
3. F-DEV-09 duplicate `edge_node_id`: desired behavior is not specified anywhere. Proposal: discovery service detects alternating births with different `boot_count` sequences and raises an event.
4. F-CLK-06: should the gateway refuse to timestamp history before its clock is plausible? Belongs to the deployment/historian docs.
5. Toxiproxy toxic names and Mosquitto Will timing (1.5 x keepalive) are from memory, UNVERIFIED.
6. F-DB-02 behavior (WAL overflow) is owned by the historian doc and not yet known.
