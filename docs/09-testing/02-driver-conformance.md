# 02 Driver Conformance Suite

driver-interface.md section 8 requires a shared conformance suite in `skyranger/pkg/driver/conformance`. This file specifies it. **A driver that does not pass every mandatory case cannot be listed in `GET /api/v1/drivers` of a release build** (the release pipeline runs the suite for each bundled driver and fails otherwise). Third-party driver authors run the same suite.

## 1. How a driver plugs into the suite

The suite cannot know a protocol, so each driver provides a small **fixture** that owns a protocol-level test peer (the simulator in a mode for that protocol, or a protocol test server) and can be told to do things to it.

```go
package conformance

// Fixture is implemented once per driver, in that driver's _test package.
type Fixture interface {
    // NewDriver returns a fresh, unconfigured driver instance.
    NewDriver(t testing.TB) driver.Driver
    // Settings returns valid Connection.spec.settings pointing at the peer.
    Settings(t testing.TB) json.RawMessage
    // InvalidSettings returns settings that must fail Configure, with reasons.
    InvalidSettings() []NamedJSON
    // Peer controls the protocol-side test peer.
    Peer() Peer
    // Addresses for the standard point set (section 3).
    Points() StandardPoints
}

type Peer interface {
    Start(t testing.TB)
    Stop()                               // clean stop (FIN), clients see disconnect
    Kill()                               // abrupt: socket closed with RST where possible
    Blackhole(on bool)                   // stop delivering bytes both ways, keep TCP open (half-open)
    SetValue(addr string, v tag.Variant) // change a point's value on the device side
    SetPointFault(addr string, f PointFault) // FaultTimeout, FaultDeviceError, FaultGone
    RejectCredentials(on bool)           // makes the next connect fail auth, if protocol has auth
    Writes() []PeerWrite                 // writes the device received, in order
}

type StandardPoints struct {
    Bool, Int, Uint, Float, String string // readable points of each type (empty if protocol lacks one)
    Counter      string                   // monotonic uint counter
    Writable     string                   // writable numeric point; "" if driver has no Write capability
    ReadOnly     string                   // exists, not writable
    Missing      string                   // syntactically valid address, no such point
    InvalidAddrs []string                 // must fail ValidateAddress
}

// Run executes every case. Opt-outs must be justified by a Capabilities flag.
func Run(t *testing.T, f Fixture, opts ...Option)
```

Each driver adds one test:

```go
// drivers/modbus/conformance_test.go
func TestConformance(t *testing.T) {
    conformance.Run(t, &modbusFixture{}, conformance.Hosting(conformance.InProcess, conformance.GRPC))
}
```

**Both hostings are mandatory.** Contract section 1 says in-process and gRPC behavior must be identical. With `Hosting(InProcess, GRPC)` the suite runs every case twice: once via `driverhost.InProcess(d)` and once by building and spawning the real `skyranger-driver-{name}` binary on a UDS in `t.TempDir()`, talking through the real host. Cases that need a process (SIGTERM, crash) only run in the gRPC hosting.

Peers per v1 driver:

| Driver | Peer |
|---|---|
| `sparkplug` | embedded MQTT broker in the test (`mochi-mqtt/server` as a Go library, UNVERIFIED that it suits; fallback Mosquitto container in the integration variant) plus `skyranger-sim` Sparkplug edge nodes in-process |
| `modbus` | `skyranger-sim` Modbus TCP server in-process (built on a Go Modbus server library, choice open, see file 03) |
| `opcua` | `skyranger-sim` OPC UA server (built on `gopcua/opcua` server support if adequate, UNVERIFIED; fallback: open62541 example server in a container, nightly only) |
| `sim` | itself; the peer is the generator state |

Time: the suite runs drivers on real time but with **short contract timeouts injected by the host test configuration** (for example `Health` every 200 ms instead of 10 s) where the contract allows host-configurable values. Contract-fixed values (SIGTERM within 5 s) are tested at their real value.

## 2. Case list

Each case has an ID used in reports and the evidence pack. "M" = mandatory for all drivers, "C" = mandatory if the capability flag is set.

### Info and configuration

| ID | M/C | Case | Pass criterion |
|---|---|---|---|
| DC-INFO-01 | M | `Info()` before `Configure` | non-empty `Name` matching segment grammar, semver `Version`, `ConfigSchema` is valid JSON Schema 2020-12 |
| DC-INFO-02 | M | `Settings()` validates against `ConfigSchema`; every `InvalidSettings()` entry fails the schema or `Configure` | |
| DC-CFG-01 | M | **Configure does no I/O.** Peer is stopped; also runs with a network namespace with no routes (gRPC hosting, Linux, CI) | `Configure` returns within 100 ms; peer observes zero connection attempts |
| DC-CFG-02 | M | `Configure` with invalid settings | returns `*Error{Code: invalid_config}`, no panic |
| DC-CFG-03 | M | Second `Configure` with changed settings while running | returns nil and applies, or returns `ErrRestartRequired`; never leaves the driver half-configured |
| DC-ADDR-01 | M | `ValidateAddress` on `InvalidAddrs` and the golden `addresses.yaml` reject list for this scheme | `invalid_address`, no I/O |
| DC-ADDR-02 | M | `ValidateAddress` on standard points | `AddressInfo.DataType` matches the expected engine type |

### Connection lifecycle

| ID | M/C | Case | Pass criterion |
|---|---|---|---|
| DC-CONN-01 | M | `Connect` returns quickly even when peer is down | returns within 1 s; state reported `connecting` then (later) `connected` after peer starts |
| DC-CONN-02 | M | Peer `Stop()` while connected | state `disconnected` reported within the driver's documented detection time; all subscribed items' samples (or host-derived) quality `bad_not_connected` |
| DC-CONN-03 | M | Peer comes back | driver reconnects by itself with backoff, state `connected`, values resume with `good` quality, **same handles** |
| DC-CONN-04 | M | Peer `Blackhole(true)` (half-open TCP) | driver detects within its documented keepalive/timeout bound (must be documented, must be <= 60 s for v1 drivers) and reports `disconnected` or `degraded` with items `bad_comm_failure`/`bad_not_connected`; it does not report `connected` forever |
| DC-CONN-05 | C (auth) | `RejectCredentials(true)` | state `faulted`, items `bad_config_error`, **no reconnect attempts** until `Configure` is called again (peer counts attempts) |
| DC-CONN-06 | M | Reconnect backoff | attempts to a dead peer over 30 s are <= 10 (no hot loop); interval grows |
| DC-CONN-07 | M | `Close` while connecting, while connected, twice | returns nil each time, no goroutine leak (checked with `go.uber.org/goleak`), peer connection closed |

### Subscriptions and values

| ID | M/C | Case | Pass criterion |
|---|---|---|---|
| DC-SUB-01 | M | Subscribe standard points of each type | one `ItemResult` per item, all nil errors; first good sample per handle within 2 s (polled: within 2 x interval) |
| DC-SUB-02 | M | Subscribe mixed batch incl. `Missing` and an invalid address | **per-item errors in results, call returns nil error**; good items still deliver values |
| DC-SUB-03 | M | Value change at peer | sample with new value delivered; `Handle` correct; value exact (no float rounding for exactly representable values) |
| DC-SUB-04 | M | Unsubscribe | no more samples for that handle after `Unsubscribe` returns (allowing in-flight batch, 1 batch grace) |
| DC-SUB-05 | M | Requested `DataType` narrower than native, value out of range (e.g. uint32 4e9 as int64 is fine; string into bool is not) | sample quality `bad_type_mismatch`, never a silently truncated value |
| DC-SUB-06 | M | Point fault `FaultTimeout` | that item `bad_comm_failure` carrying last value; other items stay `good`; connection state `degraded` or `connected` |
| DC-SUB-07 | M | Point fault `FaultDeviceError` | `bad_device_failure`, last value held |
| DC-SUB-08 | M | Point fault `FaultGone` | `bad_config_error` |
| DC-SUB-09 | C (SourceTime) | Peer supplies a source timestamp | `TSSource` set and equal to peer time (ms precision); without capability, `TSSource` is zero |
| DC-SUB-10 | M | Driver never sets `ts_gateway` | (structural: `Sample` has no such field; the host test asserts host stamps it on receipt) |

### Lifecycle restart (contract: restart re-subscribes the same handles)

| ID | M/C | Case | Pass criterion |
|---|---|---|---|
| DC-LIFE-01 | M (gRPC) | Kill driver process with SIGKILL | host reports items `bad_not_connected` within 1 health interval; respawns with backoff; re-runs `Configure`, `Connect`, `Subscribe` with **identical handle-to-address map**; values resume |
| DC-LIFE-02 | M (gRPC) | Driver hangs (test hook `SKYRANGER_TEST_HANG_HEALTH=1` makes Health block) | after 3 failed Health calls the host kills and restarts it |
| DC-LIFE-03 | M (gRPC) | SIGTERM | process exits 0 within 5 s, closes peer connection |
| DC-LIFE-04 | M (gRPC) | Driver logs | every stderr line is valid JSON with `level`, `msg`, `time` (slog JSON keys) and never contains the secret value from settings |
| DC-LIFE-05 | M | Crash loop | backoff sequence 1s, 2s, 4s ... capped at 60 s (host-level; checked once per driver because the panic path differs) |

### Backpressure (contract section 6)

| ID | M/C | Case | Pass criterion |
|---|---|---|---|
| DC-BP-01 | M | Sink blocked for 10 s while peer changes every point at high rate | driver RSS growth < 10 MiB (or < 1 KiB x items, whichever larger); after unblock, last value per handle is delivered and equals peer's last value; `dropped_samples` > 0 |
| DC-BP-02 | M | Quality transition during block: point goes `good` -> fault -> `good` | both transitions delivered in order after unblock (never coalesced away) |
| DC-BP-03 | M | Batch limits | no `SampleBatch` over 1000 samples or 64 KiB |
| DC-BP-04 | C (polled) | Poll overrun: peer latency > interval | next cycle skipped, `overruns` increments, no queue growth |

### Read, write, browse

| ID | M/C | Case | Pass criterion |
|---|---|---|---|
| DC-READ-01 | M | `Read` mixed valid/missing | per-item samples with qualities; `Handle` 0, `Address` set |
| DC-WR-01 | C (Write) | Write to `Writable` | returns after peer acknowledged; `Peer.Writes()` contains exactly **one** write with the exact raw value |
| DC-WR-02 | C (Write) | Write to `ReadOnly` | `not_writable`, peer receives nothing |
| DC-WR-03 | C (Write) | **Write never retried**: peer drops the first write's response (blackhole after request) | result `timeout` after ctx deadline; `Peer.Writes()` has exactly one entry, even after reconnect |
| DC-WR-04 | C (Write) | Write while disconnected | `not_connected`, no queued write delivered later on reconnect |
| DC-BR-01 | C (Browse) | Browse root, paged with `PageSize` 2 | concatenated pages equal unpaged result; tokens opaque; no duplicates |
| DC-DISC-01 | C (Discovery) | Device announces | one `DiscoveryUpdate` with stable `device_key`, points with `native_type` and `datatype` |

### Health and robustness

| ID | M/C | Case | Pass criterion |
|---|---|---|---|
| DC-HEALTH-01 | M | `Health` under load (DC-BP-01 running) | returns within 50 ms |
| DC-ROB-01 | M | Peer sends malformed protocol data (from the driver's fuzz corpus, file 09) | no panic, no process exit; item or connection quality reflects error |
| DC-ROB-02 | M | 1000 subscribe/unsubscribe cycles | no goroutine or memory leak (goleak, RSS delta) |

## 3. Sparkplug add-on suite (`conformance/sparkplug`)

The Sparkplug driver has protocol state the generic suite cannot see. Extra mandatory cases, all from messages.md section 1:

| ID | Case | Pass criterion |
|---|---|---|
| SP-01 | Driver publishes `STATE` online retained at connect, with Will offline | broker retained message matches golden `sparkplug/state_online.json` |
| SP-02 | NBIRTH then NDATA by alias only | values mapped to correct metrics |
| SP-03 | NDATA before any NBIRTH | node tags `uncertain_last_known`, one `Rebirth` NCMD sent |
| SP-04 | `seq` gap (skip one) | `uncertain_last_known` for the node, Rebirth sent; after NBIRTH back to `good` |
| SP-05 | Rebirth rate limit | 10 gaps in 5 s produce exactly 1 Rebirth; another after 30 s |
| SP-06 | `seq` wrap 255 -> 0 | **no** gap detected |
| SP-07 | NDEATH with matching `bdSeq` | node tags `uncertain_last_known`, `@online` false |
| SP-08 | Stale NDEATH (old `bdSeq`, arrives after a newer NBIRTH, typical when broker delivers a delayed Will) | ignored; tags stay `good` |
| SP-09 | Unknown alias | `uncertain_last_known` + Rebirth |
| SP-10 | Broker disconnect | all tags `bad_not_connected` (not `uncertain_last_known`) |
| SP-11 | `DataSet`/`Bytes`/`File`/`Template` metrics in birth | reported in discovery, subscribe yields `bad_type_mismatch` |
| SP-12 | Metric timestamp vs payload timestamp | metric ts used as `TSSource`, payload ts as fallback |
| SP-13 | Duplicate NDATA (same seq, QoS 1 redelivery) | no double-processing effect on counters (cumulative, so value repeats) and no false gap |

## 4. Device-profile conformance (`skyranger.pulse_meter.v1`)

A separate suite checks **devices** (the real ESP32 firmware on the HIL rig, and the simulator's `pulse_meter` model) against messages.md section 2. It runs as a Sparkplug host observer.

| ID | Check |
|---|---|
| PM-01 | NBIRTH contains every normative metric with the normative Sparkplug type |
| PM-02 | `profile` = `skyranger.pulse_meter.v1`; `edge_node_id` satisfies segment grammar |
| PM-03 | `boot_count` increments by exactly 1 per power cycle; unchanged by MQTT reconnect |
| PM-04 | `pulse_count` starts at 0 after boot; unchanged by MQTT reconnect |
| PM-05 | Heartbeat NDATA every `heartbeat_interval_s` ±10% with `uptime_s`, `pulse_count`, `rssi_dbm`, even with no pulses |
| PM-06 | NDATA on change at most 1 per second |
| PM-07 | Rebirth NCMD produces NBIRTH with same `boot_count`, current `pulse_count` |
| PM-08 | Pulse accuracy against generator (HIL only, file 08) |

The simulator model must pass PM-01..PM-07 too. That keeps the simulator honest against the real device, so simulated results stay meaningful.

## 5. Reporting

`conformance.Run` writes `conformance-{driver}-{hosting}.json` (case ID, pass/fail, duration, driver version, commit). The release pipeline aggregates these into the evidence pack's driver conformance table (file 11).

## Decisions made
1. Driver plugs in a `Fixture` with a controllable protocol `Peer`; the suite is protocol-agnostic and owns all assertions.
2. Every case runs in both in-process and gRPC hosting; process-only cases in gRPC.
3. Case IDs are stable and appear in reports and the evidence pack.
4. Sparkplug and the pulse-meter device profile get add-on suites; the simulator must pass the device-profile suite so it stays faithful to the real firmware.
5. Half-open TCP detection bound must be documented per driver and be at most 60 s in v1.

## Alternatives rejected and why
- **Per-driver hand-written lifecycle tests**: each driver would interpret the contract differently, and agents writing new drivers would copy the weakest one.
- **Conformance only in-process**: would miss gRPC serialization, handle mapping and process supervision bugs, which the contract says must behave identically.
- **Mocked peers (interfaces mocked with gomock)**: test the driver against our assumptions of the protocol, not the protocol. Real protocol peers (simulator, test servers) only.

## Open questions
- Contract says on broker disconnect "all tags go `bad_not_connected`" and also that the host maps `disconnected` state to `bad_not_connected`. Which side emits the samples (driver per item, or host on state change) should be stated; the suite currently accepts either but requires the observable tag quality.
- DC-CONN-04: 60 s bound is my proposal. MQTT keepalive defaults and Modbus TCP have no keepalive of their own; Modbus driver must rely on request timeouts. Needs agreement with driver authors.
- Go embedded MQTT broker for in-process Sparkplug tests: `mochi-mqtt/server` maturity and MQTT 3.1.1 Will behavior UNVERIFIED.
- OPC UA test server choice UNVERIFIED (gopcua server support level).
- Whether `ValidateAddress` may be called before `Configure` (the suite assumes yes, since it does no I/O).
