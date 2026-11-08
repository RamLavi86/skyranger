# 03 Device Simulator

Decision 7 calls for a device simulator (MQTT, Modbus TCP, OPC UA) with fault injection. This file designs it. There are two delivery forms built on one library:

| Form | What it is | Used in |
|---|---|---|
| `pkg/sim` | Go library: device models, generators, seeded RNG, scenario engine, oracle bookkeeping | unit/property tests (oracle), L3 harness, both forms below |
| `skyranger-sim` | Standalone binary/container: many virtual devices speaking **real protocols** on real sockets (Sparkplug B over MQTT, Modbus TCP server, OPC UA server) | L4 integration, L5 e2e, L6 soak/perf, demos, customer FAT |
| `sim` driver (`skyranger-driver-sim`) | A driver that generates values internally from `sim://` addresses, no network | quick demos, HMI development, perf tests of the engine without protocol cost |

All three share generators and device models, so `sim://pulse_meter?...`, a simulated Sparkplug pulse meter and the property-test oracle are the same code.

## 1. Requirements

1. **Many devices at once**: at least 10,000 Sparkplug edge-node/device instances and 100,000 metrics from one `skyranger-sim` process on a 4-core CI runner (target; to be measured), so it is never the bottleneck in perf tests (file 07).
2. **Real protocols**: the gateway must not be able to tell a simulated device from a real one at the protocol level.
3. **YAML scenarios**: devices, value behavior, and a timeline of events and faults.
4. **Deterministic**: same scenario + same seed = same device-side event sequence (values, timestamps in virtual time, fault times). Logged seed reproduces a failure.
5. **Time control**: virtual (as fast as possible), scaled, and real time.
6. **Oracle output**: the simulator records what it truly did (pulses emitted, reboots, which messages were dropped) so tests can compare the gateway's results against ground truth.
7. **Control API**: tests drive faults at runtime, not only from the scenario.
8. **Faithful**: the `pulse_meter` model must pass the device-profile conformance suite PM-01..PM-07 (file 02).

## 2. Architecture

```
                         scenario.yaml + seed
                                │
                      ┌─────────▼─────────┐
                      │  Scenario engine  │  discrete-event scheduler, priority queue of (vtime, seq, action)
                      │  clock.Clock      │  Fake (virtual) | Scaled | Real
                      └─────────┬─────────┘
            ┌───────────────────┼─────────────────────────┐
     ┌──────▼──────┐     ┌──────▼──────┐            ┌──────▼──────┐
     │ Device      │ ... │ Device      │            │ Oracle log  │  JSONL: emitted, dropped, reboots, truth
     │ models      │     │ models      │            └─────────────┘
     └──────┬──────┘     └──────┬──────┘
            │ points (name, type, value, quality hint)
   ┌────────┼──────────────────┬──────────────────────┐
┌──▼───────────────┐ ┌─────────▼────────┐ ┌───────────▼────────┐
│ Sparkplug edge   │ │ Modbus TCP server│ │ OPC UA server      │  protocol adapters
│ nodes (MQTT      │ │ (one listener per│ │ (one endpoint,     │
│ clients)         │ │ device or unit-id│ │ namespace per dev) │
└──┬───────────────┘ └─────────┬────────┘ └──────────┬─────────┘
   │ fault layer: per-adapter wire faults (drop, dup, reorder, delay, corrupt, half-open)
   ▼                           ▼                     ▼
 MQTT broker              TCP sockets           TCP sockets
                                               (optionally via Toxiproxy in L4)
            Control API: HTTP on :9900 (JSON) for tests; Prometheus metrics on :9901
```

- **Device model**: a set of points plus behaviors (generators) plus lifecycle (power, reboot, network). Models: `pulse_meter` (the ESP32 profile exactly), `generic` (any points from generators), `modbus_plc` (register map), `opcua_server` (node tree).
- **Generators** (from driver-interface.md section 5 list): `const`, `sine`, `ramp`, `counter`, `random_walk`, `pulse_meter`. Each is a pure function of `(vtime, rng stream, params)` plus explicit state, so it can be replayed.
- **RNG**: one root seed; each device gets its own stream `PCG(seed, hash(device_id))`. Adding a device to a scenario does not change the values of other devices. Go's `math/rand/v2` PCG is used (stable algorithm across Go versions; UNVERIFIED that its output is guaranteed stable across releases, so the simulator vendors a fixed PCG implementation to be safe).
- **Scheduler**: single-threaded discrete-event loop for model state. Protocol I/O runs on goroutines but takes values from the loop through channels, so model state is deterministic even though network timing is not.

### Determinism: what is and is not guaranteed

| Mode | Deterministic | Notes |
|---|---|---|
| Virtual time, in-process transports (L3) | Fully: device events, gateway processing order, results | Gateway and simulator share one `clock.Fake`; transports are in-memory |
| Virtual or scaled time, real sockets (L4) | Device-side event sequence only | Network and OS scheduling are not deterministic; tests assert with tolerances and oracle comparisons, not exact traces |
| Real time | Device-side sequence given the seed | Used for soak |

## 3. Time modes

| Mode | Flag | Semantics | Where valid |
|---|---|---|---|
| `virtual` | `--time=virtual` | Clock jumps to the next scheduled event; runs as fast as CPU allows | L3 only, where the gateway shares the fake clock. **Never with a separate gateway process**, whose real clock would see a 2 h scenario take 2 s |
| `scaled` | `--time=scaled --speed=10` | Device time runs N times faster than real time | Only with a **compressed config profile** (below) |
| `real` | default | 1:1 | L4, L6, L7, demos |

**Speed-up and a separate gateway process.** A Go gateway in a container cannot be sped up from outside (Go reads time through the vDSO, not libc, so `libfaketime` is not expected to work; UNVERIFIED but treat as true). Speed-up at L4 is therefore done by **compressing the scenario and the gateway config by the same factor**, as an explicit test profile:

```yaml
# test/compose/profiles/compressed-x10.yaml  (applied with the config API before the scenario)
# Production: heartbeat 60s, heartbeat_stale 150s, leak_max_gap 15m, leak_duration 2h
# Compressed x10: every scenario-owned duration / 10. Contract-fixed constants are NOT scaled:
#   Sparkplug rebirth limit 30s, driver Health 10s, restart backoff, WS 100ms batching.
params:
  heartbeat_stale: 15s
  leak_max_gap: 90s
  leak_duration: 12m
device:
  heartbeat_interval_s: 6
```

Rule: compressed results are evidence of logic, never of timing. Evidence-pack timing claims (file 11) come from real-time runs only. L3 virtual time covers the logic with full production durations.

## 4. Scenario file format

`apiVersion: skyranger-test/v1` is deliberately different from the product's `skyranger/v1`, so a scenario can never be applied as product config.

```yaml
apiVersion: skyranger-test/v1
kind: Scenario
metadata:
  name: water-meter-reboot-and-leak
  description: "Meter at 12 L/min, reboots twice, one rollover, then a slow leak"
  tags: [water_meter, counter, faults, milestone1]
  faults_covered: [F-DEV-01, F-DEV-02, F-CNT-01]        # IDs from 04-fault-injection.md
spec:
  seed: 42                    # overridable with --seed; logged in every report
  start: 2026-11-07T10:00:00Z # virtual start time (wall)
  duration: 6h
  time: { mode: virtual }     # virtual | scaled (speed) | real

  brokers:
    - name: local
      url: mqtt://mosquitto:1883          # L3 harness substitutes an in-memory transport
      credentials: { username: sim, password_env: SIM_MQTT_PASSWORD }

  devices:
    - id: meter01
      model: pulse_meter
      protocol: { sparkplug: { broker: local, group_id: skyranger, edge_node_id: esp32-meter01 } }
      params:
        heartbeat_interval: 60s
        boot_count: 7                     # NVS value at scenario start
        start_pulse_count: 0
        mqtt_keepalive: 20s               # broker fires the NDEATH Will roughly 1.5 x keepalive after power loss (UNVERIFIED factor)
        flow:                             # liters per minute profile, 1 pulse = 1 L
          generator: schedule
          points:
            - { at: 0s,    lpm: 12 }
            - { at: 5m,    lpm: 0 }
            - { at: 1h,    lpm: 12 }      # flow so the counter crosses 2^32 after the `set` below (296 pulses, ~25 min)
            - { at: 1h50m, lpm: 0 }
            - { at: 2h,    lpm: 0.2 }     # slow leak: 1 pulse every 5 min
        clock:
          sntp_synced_after: 30s          # before that, device timestamps are 1970-based
          drift_ppm: 40

    - id: fleet
      model: pulse_meter
      count: 500                          # templated fleet: fleet-000 .. fleet-499
      protocol: { sparkplug: { broker: local, group_id: skyranger, edge_node_id: "esp32-f{index:03d}" } }
      params:
        heartbeat_interval: 60s
        flow: { generator: random_walk, mean_lpm: 5, sigma: 2, min: 0, max: 40 }
        start_jitter: 60s                 # spread connects so births do not all land in one second

    - id: plc1
      model: modbus_plc
      protocol: { modbus_tcp: { listen: ":5020", unit_ids: [1] } }
      points:
        - { addr: "hr/40001", type: float32, order: cdab, generator: sine, params: { mean: 50, amplitude: 10, period: 10m } }
        - { addr: "ir/30001", type: int16, generator: random_walk, params: { mean: 215, sigma: 1 } }  # 21.5 Cel x10
        - { addr: "co/00001", type: bool, writable: true, generator: const, params: { value: false } }

    - id: opc1
      model: opcua_server
      protocol: { opcua: { listen: ":4840", security: none } }    # security modes: see file 09
      points:
        - { node: "ns=2;s=Line1.Flow", type: Double, generator: ramp, params: { from: 0, to: 100, period: 30m } }

  timeline:
    - at: 30m
      action: reboot                # power-cycle: boot_count+1, pulse_count from 0, MQTT Will -> NDEATH
      target: meter01
      params: { offline_for: 5m }
    - at: 1h
      action: set
      target: meter01
      params: { pulse_count: 4294967000 }   # jump near 2^32 to force a rollover soon (test-only action)
    - at: 1h05m
      action: fault
      target: meter01
      fault: drop_messages
      params: { types: [NDATA], probability: 0.3, for: 20m }
    - at: 3h
      action: fault
      target: broker:local
      fault: broker_restart
      params: { down_for: 45s }
    - at: 4h
      action: reboot
      target: fleet[0:50]           # slice of a fleet
      params: { offline_for: 10s, stagger: 5s }

  expect:                           # asserted by the runner after (or during) the run
    - tag: site1/pumphouse/meter01/total_volume
      equals_oracle: total_liters_received_by_gateway   # oracle-computed, exact
    - tag: site1/pumphouse/meter01/leak
      value: true
      within: { after: 4h05m, before: 4h10m }   # leak_duration 2h after slow flow started at 2h + gap logic
    - alarm: site1/pumphouse/meter01/leak/leak
      state: active_unacked
      by: 4h10m
    - event: skyranger.tag.counter_discontinuity.v1
      subject: site1/pumphouse/meter01/total_volume
      count: { kind: reset, eq: 1 }
    - event: skyranger.tag.counter_discontinuity.v1
      subject: site1/pumphouse/meter01/total_volume
      count: { kind: rollover, eq: 1 }
    - tag: site1/pumphouse/meter01/health
      sequence: [2, 0, 2]           # ok, offline (NDEATH ~30 s after power loss, before 150 s staleness), ok again
      between: [29m, 37m]
    - invariant: no_history_gaps_longer_than
      params: { tag_prefix: site1/pumphouse/meter01, max: 16m }    # max_interval 15m + slack
```

### Scenario schema rules

- JSON Schema at `test/scenarios/schema/scenario-v1.json`; every scenario file is validated in CI.
- `metadata.faults_covered` links scenarios to the fault catalogue; the catalogue coverage report (file 04) is computed from this.
- `expect` entries are evaluated by `pkg/sim/expect` against the gateway's public API only (REST `/api/v1/history`, `/api/v1/events`, `/api/v1/alarms`, WS). Tests never read the database directly for assertions. That keeps tests valid when internals change and checks the API at the same time.
- Times in `timeline` and `expect` are offsets from `spec.start` (Go durations).
- Templated device IDs use `{index:03d}` and the result must satisfy the segment grammar.
- Test-only actions (`set` on a counter) are flagged in the schema as `test_only: true` and cannot be used in the `demo` scenario set shown to customers.

### Actions and faults available in `timeline`

| Action | Targets | Params |
|---|---|---|
| `reboot` | device, fleet slice | `offline_for`, `stagger`, `clean` (false = no NDEATH sent by device; broker Will fires on keepalive timeout) |
| `power_off` / `power_on` | device | as reboot, split |
| `network_down` / `network_up` | device | device keeps counting, MQTT disconnects (no reboot) |
| `set` | device point | direct value (test-only) |
| `flow` / `generator` | device | change generator params at runtime |
| `fault` | device, broker, adapter | any wire fault in file 04 (`drop_messages`, `duplicate`, `reorder`, `delay`, `corrupt`, `half_open`, `seq_skip`, `bad_alias`, `burst`, `stale`, `bad_quality`, `clock_jump`) |
| `broker_restart` | broker | `down_for`, `persist` (retained messages kept or not) |
| `checkpoint` | runner | snapshot oracle and gateway state for mid-run asserts |

Gateway-side faults (DB down, disk full, kill `skyrangerd`) are not simulator actions; the L4 fault runner executes them (file 04). The scenario can reference them with `action: external` and a fault ID, and the runner dispatches them.

## 5. `pulse_meter` model (the oracle that matters most)

Implements messages.md section 2 exactly:

- State: `boot_count` (persisted across reboots, like NVS), `pulse_count` (uint32, from 0 per boot, wraps at 2^32), `uptime_s`, `true_pulses_total` (oracle only, never published).
- Pulses are generated from the flow profile as a **Poisson-free deterministic accumulator**: `acc += lpm/60 * dt`; emit `floor(acc)` pulses; keep the fraction. Optional `jitter` param uses the device RNG stream.
- NDATA on change at most 1/s, heartbeat every `heartbeat_interval`, NBIRTH on connect and on Rebirth NCMD, NDEATH as MQTT Will with the current `bdSeq`; `bdSeq` increments per MQTT session.
- Clock: device wall time = virtual time + drift, or 1970-based until `sntp_synced_after`.
- **Oracle bookkeeping** per device:
  - `true_pulses_total`
  - `pulses_lost_to_reboots`: on each reboot, `pulse_count_at_reboot - last_pulse_count_delivered_to_gateway_in_this_epoch`. "Delivered" is known exactly in L3 (in-memory transport acks) and is taken from the broker-side tap in L4 (file 05 capture tap, which sees what the broker forwarded to the gateway's subscription).
  - `expected_total_volume = initial_total + (true_pulses_total - pulses_lost_to_reboots) * liters_per_pulse`, under the `reset_baseline: zero` rule.
  - Expected discontinuity events by kind.

The oracle runs its own minimal reimplementation of the contract algorithm only for the "lost pulses" accounting. It deliberately does not reuse `pkg/tag/derived`, otherwise a bug there would also be in the oracle.

## 6. `sim://` driver addresses (details owned here)

Grammar extends driver-interface.md section 5: `sim://{generator}?{params}`. Query parameters use Go durations and plain numbers.

| Generator | Params | Output type |
|---|---|---|
| `const` | `value`, `type` (`bool,int64,uint64,float64,string`) | as `type` |
| `sine` | `mean`, `amplitude`, `period`, `phase` | float64 |
| `ramp` | `from`, `to`, `period`, `mode` (`sawtooth`\|`triangle`) | float64 |
| `counter` | `start`, `step`, `every`, `modulus` | uint64 |
| `random_walk` | `mean`, `sigma`, `min`, `max`, `revert` (mean-reversion 0..1) | float64 |
| `pulse_meter` | `lpm`, `point` (`pulse_count`\|`boot_count`\|`uptime_s`\|`online`), `device` (shared state key), `reboot_every`, `heartbeat` | per point |

Common fault params on any address:

| Param | Example | Effect |
|---|---|---|
| `interval` | `interval=1s` | update period (default `Item.Interval`) |
| `seed` | `seed=7` | per-item stream; default derived from connection seed + address |
| `drop` | `drop=0.1` | probability a scheduled update is not delivered |
| `stale` | `stale=5m/2m` | every 5 m, stop updating for 2 m (tests `stale_after`) |
| `bad_quality` | `bad_quality=bad_comm_failure:0.05` | probability of a sample with that quality code name (must be a contract name) |
| `burst` | `burst=100x10ms/1m` | every 1 m, 100 updates 10 ms apart |
| `nan` | `nan=0.01` | probability of NaN for float outputs (tests `bad_type_mismatch` handling) |

Examples:

```
sim://sine?mean=50&amplitude=10&period=10m
sim://pulse_meter?device=m1&point=pulse_count&lpm=12&reboot_every=1h
sim://counter?start=4294967000&step=1&every=1s&modulus=4294967296
sim://random_walk?mean=20&sigma=0.5&bad_quality=bad_comm_failure:0.02&stale=30m/5m
```

`ValidateAddress` rejects unknown generators, unknown params and quality names outside the contract list.

## 7. Control API (`skyranger-sim` :9900)

Plain HTTP+JSON, unauthenticated, **binds to localhost or the compose network only**; refuses to start bound to `0.0.0.0` without `--i-know-this-is-a-test-tool`.

| Method | Path | Body | Purpose |
|---|---|---|---|
| GET | `/v1/devices` | | list devices, state, protocol endpoint |
| POST | `/v1/devices/{id}/actions` | `{"action": "reboot", "params": {...}}` | same actions as timeline |
| POST | `/v1/faults` | `{"target": "...", "fault": "duplicate", "params": {...}}` → `{"fault_id"}` | start fault |
| DELETE | `/v1/faults/{fault_id}` | | stop fault |
| GET | `/v1/oracle` | | oracle snapshot for all devices (JSON) |
| GET | `/v1/oracle/stream` | | JSONL stream of oracle records |
| POST | `/v1/scenario/pause` / `resume` | | |
| GET | `/v1/time` | | current scenario time, mode, speed |

Go tests use a typed client in `pkg/sim/simclient`.

## 8. Scale design notes

- One MQTT connection per Sparkplug edge node is required for realism (Will per node). 10,000 nodes means 10,000 TCP connections to Mosquitto. The simulator raises `RLIMIT_NOFILE`; the compose file sets `ulimits.nofile`. Mosquitto's connection limit config (`max_connections`) must be raised in the perf profile (UNVERIFIED default value).
- For pure tag-engine load without MQTT connection count, use Sparkplug **devices** under fewer edge nodes (e.g. 100 nodes x 100 devices x 10 metrics), and the `sim` driver for no-network load.
- MQTT client library for the simulator: candidates are `eclipse/paho.golang` (MQTT 5) and `eclipse/paho.mqtt.golang` (3.1.1). Deliberately **use a different client library from the gateway's Sparkplug driver** where practical, so a bug in one library cannot hide on both sides of the test. UNVERIFIED which library the Sparkplug driver doc picks.
- Modbus TCP server: candidate Go libraries exist (for example `simonvetter/modbus` offers a server; UNVERIFIED maturity). Fallback is a small hand-written Modbus TCP server, which is about 300 lines for function codes 1-6, 15, 16.
- OPC UA server: weakest part of the Go ecosystem. Option A: gopcua server (UNVERIFIED feature level). Option B: run `open62541`-based or Python `asyncua` server in a separate container driven by the same scenario through the control API (slower, but mature). Decide when the OPC UA driver is built; milestone 1 does not need it.
- Protobuf encoding for Sparkplug uses the Tahu `.proto` compiled with Go protobuf, the same file as the gateway (that is a contract artifact, not a library choice, so sharing it is correct).

## 9. Scenario library (initial, `test/scenarios/`)

| Scenario | Purpose | Tier |
|---|---|---|
| `wm-trace-contract.yaml` | Exactly the trace table in tag-model.md section 11; asserts every cell | L3 per commit, L4 nightly |
| `wm-reboot-storm.yaml` | 20 reboots in 1 h with traffic | L3, L4 |
| `wm-rollover.yaml` | start near 2^32, cross twice | L3 |
| `wm-no-flow-vs-dead.yaml` | heartbeats with zero flow, then silent device, then network down | L3, L4, L5 (HMI shows the difference) |
| `wm-slow-leak.yaml` | 0.2 L/min for 3 h; leak alarm timing | L3 |
| `wm-clock-unsynced.yaml` | 1970 timestamps then SNTP sync; skew rejections | L3 |
| `fleet-500.yaml` | 500 meters, random flows, staggered reboots | L4 nightly |
| `mixed-plant.yaml` | meters + Modbus PLC + OPC UA | L4 nightly (OPC UA when available) |
| `perf-10k.yaml`, `perf-100k.yaml` | file 07 | L6 |
| `soak-plant.yaml` | file 07 | L6 |
| `demo-pumphouse.yaml` | customer demo, no test-only actions | demos, FAT |

## Decisions made
1. One Go library `pkg/sim` behind three forms: library oracle, `skyranger-sim` multi-protocol binary, and the `sim` driver.
2. Scenario YAML with `apiVersion: skyranger-test/v1`, timeline actions, linked fault IDs, and `expect` blocks evaluated through the public API only.
3. Determinism via a single-threaded discrete-event model loop with per-device RNG streams from one seed.
4. Virtual time only in-process (L3); speed-up across process boundaries only via explicit compressed config profiles, never claimed as timing evidence.
5. The pulse meter oracle tracks true pulses and pulses lost to reboots, and does not reuse the production `counter_total` code.
6. The simulator uses a different MQTT client library from the gateway where practical.

## Alternatives rejected and why
- **Existing simulators only** (for example Node-RED flows, Modbus slave GUIs, Prosys OPC UA simulation server): useful for manual exploration, but none give us seeded determinism, Sparkplug pulse-meter semantics with an oracle, and a fault control API in one tool. They remain allowed for ad-hoc tests.
- **Python simulator**: author knows Python well and asyncua is mature, but the Go `pkg/sim` must exist anyway for the oracle and L3 harness; two implementations would drift. Python is kept as a fallback only for the OPC UA server.
- **libfaketime to speed up containers**: not expected to affect Go binaries; also would desynchronize the database and broker clocks.
- **Random fault injection without seeds (chaos-monkey style) in CI**: unreproducible failures. Randomness is allowed, but always seeded and logged.

## Open questions
- Simulator MQTT, Modbus server and OPC UA server library choices are UNVERIFIED and need a short spike each.
- The oracle's notion of "delivered to gateway" at L4 relies on the broker-side capture tap (file 05). If that tap misses messages under load, conservation checks give false failures; the tap must be validated at the perf target rate.
- Whether `sim://` fault params should live in the address (as the contract example implies) or in Connection settings. Address params are kept for contract compatibility.
- Leak alarm timing expectation in the example (`within 4h05m..4h10m`) depends on `continuous_activity` edge semantics (first increase after a gap starts `active_since`); the exact window must be derived once the tag engine implements it and then frozen.
