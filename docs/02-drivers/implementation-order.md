# Implementation Order

Phased build order for the driver area, with exit criteria. It matches the catalogue in [README.md](README.md). Effort figures are guesses for one developer with AI agent help, to order work and not to promise dates. Milestone 1 (decisions.md entry 6) needs only phases 0 and 1.

## 1. Dependency graph

```text
 contracts (00) ──► Phase 0: SDK + host + sim driver + conformance suite
                          │
                          ├──► Phase 1: sparkplug    ──► MILESTONE 1 (ESP32 meter -> tag engine -> history -> dashboard)
                          │
                          ├──► Phase 2: modbus       ──► first integrator projects (most small sites)
                          │        │
                          │        └─ bench: OpenPLC, ESP32 Modbus slave
                          │
                          ├──► Phase 3: opcua client ──► modern PLCs (S7-1500, CODESYS, Beckhoff), gateways (Kepware)
                          │        (spike first)
                          │
                          ├──► Phase 4: enip         ──► Rockwell sites           (spike: libplctag build, gologix)
                          │
                          ├──► Phase 5: s7           ──► older Siemens sites       (spike: gos7)
                          │
                          └──► Phase 6: opcua server (northbound; separate process, uses the public API)
```

Phases 2 to 5 can run in parallel once phase 0 is done, because drivers share only the SDK and the contract. The order below is the recommended **priority** when effort is limited, and a customer's actual hardware overrides it.

## 2. Phases

### Phase 0: foundation (about 3 weeks)

| Deliverable | Notes |
|---|---|
| `pkg/driver` contract types, `driver.proto`, `buf` lint and breaking checks | owned by the contract; implement exactly |
| `pkg/driver/sdk` (`Main`, gRPC server, coalescer, state reporter, backoff, scheduler, convert, addr helpers) | framework.md |
| `internal/driverhost` (spawn, supervise, health, RSS kill, `InProcess`) | framework.md section 4 |
| `sim` driver | generators and faults per the `sim://` addresses; the full multi-protocol `skyranger-sim` server is owned by 09-testing/03 |
| Conformance suite (lifecycle restart re-subscribe, coalescing, quality transitions, per-item errors, idempotent Close, no I/O in Configure, SIGTERM in 5 s) | runs in-process and over gRPC |
| Licence gate (libraries-and-licensing.md section 4) in CI from day one | cheap now, painful later |

**Exit criteria**
1. `sim` passes the conformance suite in both modes.
2. Killing the `sim` process: all its tags `bad_not_connected` within 1 s of detection, restart with backoff, same handles re-subscribed, tags `good` again.
3. A blocked sink for 60 s: memory flat, `dropped_samples` rises, latest values correct after release, no quality transition lost.
4. A hung driver (test hook that stops answering Health): killed and restarted by the host.
5. Licence gate fails a PR that adds a GPL dependency (test with a throwaway module).

### Phase 1: `sparkplug` (about 2 to 3 weeks) - milestone 1

Per sparkplug.md: birth/death, seq checking, aliases, rebirth requests (rate-limited), STATE, discovery into the approval flow, counters and reboots as represented in messages.md.

**Exit criteria**
1. Passes the shared suite and the Sparkplug add-on suite (09-testing/02).
2. ESP32 pulse meter on the bench rig: oracle pulse count equals gateway `total_volume` after 24 h including 3 device reboots and 1 broker restart (rig and oracle: 09-testing/08).
3. NDEATH gives `uncertain_last_known`; a pulled network cable gives `bad_not_connected`; the health tag tells them apart (decisions.md entry 6).
4. Soak: 72 h with the simulator publishing 200 edge nodes, no memory growth, no sequence-gap storm after a broker restart.

### Phase 2: `modbus` (about 1.5 to 2 weeks)

Own client, address grammar and all worked examples as unit tests, planner with gap merging and bad-address isolation, exception mapping, per-unit settings (modbus-tcp.md).

**Exit criteria**
1. Every row of the modbus-tcp.md address table is a passing unit test.
2. Against a simulator: delete a register mid-run, only that tag goes `bad_config_error`, neighbours stay `good`, state `degraded`.
3. Against OpenPLC on a Raspberry Pi (test-bench.md): 24 h at 1 s scan of 200 registers, zero unexplained quality flips, reconnect after a PLC reboot in under 35 s (30 s backoff ceiling plus one scan).
4. Byte-order check against one real device or a documented test vector for each of the four orders.
5. Write test with the no-retry rule: kill the link during a write, result `timeout` (unknown outcome), no second write observed on the wire.

Decision gate before phase 3: is Modbus RTU/serial needed by a first customer (README open question)? If yes, RTU-over-TCP and serial transport slot in here (same PDU code, about 1 week).

### Phase 3: `opcua` client (about 2 weeks including the spike)

Spike (time-box 3 to 5 days, written result in the repo, per opcua.md section A1). Questions:
- Do subscriptions survive 72 h with server restarts, link cuts and an expired session?
- Can we disable or observe the library's own reconnect?
- Which security policies and user token types work against open62541, an S7-1500 (if available), CODESYS?
- Do timeouts honour `context`? Any goroutine leaks?

**Spike pass/fail**: if more than one of the four fails and cannot be worked around in one day each, switch to open62541 through cgo (adds a C build stage, MPL-2.0 pinned, libraries-and-licensing.md) or a thin separate process.

**Exit criteria**
1. Passes the shared suite against an open62541 server and the simulator's OPC UA server.
2. Namespace index drift handled (`namespace_uris`), certificate trust flow documented.
3. 72 h soak with 2000 monitored items and fault injection (link cuts every 10 min): no leak, all items recover, quality mapping matches opcua.md.
4. Rejected certificate gives `faulted` and no retry loop.

### Phase 4: `enip` (about 2 weeks, plus 1 to 3 days for the cgo and gologix spikes)

Spikes: (a) vendored libplctag static build in the CI image stage with the pinned MPL-2.0 licence record; (b) gologix against `ab_server`, to decide whether cgo can be dropped.

**Exit criteria**
1. Passes the shared suite against `ab_server` (read and write, arrays elements, wrong name, link cut).
2. Handle leak test (10 000 subscribe/unsubscribe cycles) is flat.
3. One packet capture against a real Logix-family or Micro800 bench PLC proves CIP connection sharing and request packing assumptions, or the doc is corrected.
4. Browse (4b) lists controller and program tags on the bench PLC, if the library supports it. Otherwise 4b is deferred and documented.

### Phase 5: `s7` (about 1 to 2 weeks, plus a 2-day `gos7` spike)

**Exit criteria**
1. Address table in siemens-s7.md is a passing unit test.
2. Planner property test: no request or response over the negotiated PDU, none over 20 items.
3. Against the snap7 server and one real S7 CPU (1200 or LOGO! on the bench): 24 h, 200 bytes across 3 DBs at 1 s, no flips. PUT/GET-disabled case gives `degraded` with the checklist hint.
4. Link cut test; no parallel sessions ever opened (checked in the capture).

### Phase 6: OPC UA server, northbound (about 2 to 3 weeks)

Separate process, client of the public REST/WS API (opcua.md part B). Library decision after the phase 3 spike results. No HistoryRead, no Alarms and Conditions in v1; writes off by default.

**Exit criteria**
1. UaExpert (free client) and one SCADA client browse and read tags with correct quality mapping.
2. Core stays up when the server process is killed or fuzzed (09-testing/09).
3. Username auth mapped to Sky Ranger users.

## 3. Cross-phase rules

- **A driver ships only if** it passes the conformance suite in-process and over gRPC, has an address table as tests, a quality mapping table as tests, a fault-injection run (test-bench.md section 5) and its libraries are recorded in the dependency register.
- **Spike results are written down** in the driver doc (replace the UNVERIFIED mark with "checked on date, version"), not left in someone's head.
- **Order of risk retirement**: licence and build risk (cgo, libplctag) before feature work; library maturity (gopcua, gos7) before building the driver on it.
- **One driver per PR train**, each with its own doc update. Agents may not weaken conformance tests (09-testing/12).

## 4. Why this order

1. `sim` and the host first: every other driver needs the supervision and the suite, and they are testable with no hardware.
2. `sparkplug` second: it is the milestone and the project's reason to exist, and it exercises push, discovery, quality and the backpressure path.
3. `modbus` third: highest customer reach (small water/wastewater and metering sites) for the lowest effort and risk; also the first polled driver, which proves the scheduler and planner that `s7` and `enip` reuse.
4. `opcua` fourth: broad reach, and for new Siemens/Beckhoff/CODESYS PLCs it replaces the need for `s7`. It has the biggest library risk, so a spike runs early (it can start during phase 2).
5. `enip` before `s7`: a Rockwell site has no good alternative except a gateway, while many Siemens sites can use OPC UA. If the first customer has older Siemens PLCs, swap them.
6. OPC UA server last: customers can use the REST API and Sparkplug northbound until someone asks to connect a third-party client.

## Decisions made
1. Order: 0 foundation and `sim`, 1 `sparkplug`, 2 `modbus`, 3 `opcua` client, 4 `enip`, 5 `s7`, 6 OPC UA server. Customer hardware can reorder phases 3 to 5.
2. Every phase has written exit criteria that include fault behavior, not only happy-path reads.
3. Library spikes are time-boxed, have explicit pass/fail, and their result is written into the doc.
4. The licence gate is in CI from phase 0.
5. Phases 2 to 5 may overlap after phase 0, but milestone 1 is not delayed by them.

## Alternatives rejected and why
- **OPC UA client first because it is universal**: biggest library risk and unneeded for the milestone. See README.
- **All protocols in parallel from the start**: spreads thin, delays the milestone, and the shared SDK would be designed against guesses.
- **Hardware before simulators**: slow, unrepeatable, and not runnable in CI. Simulators first, hardware validates them (test-bench.md).
- **Skipping spikes to save days**: a failed library discovered after the driver is built costs weeks.

## Open questions
- Effort numbers are guesses; recalibrate after phase 2 and update this file.
- Whether a first customer needs Modbus RTU (serial) before OPC UA.
- Whether the OPC UA spike should start during phase 1 so its result is available for phase 3 planning.
- Which real PLCs will actually be purchased (test-bench.md section 3), which determines when phases 4 and 5 can be exit-tested on hardware.
