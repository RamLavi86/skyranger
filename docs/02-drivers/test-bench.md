# Driver Test Bench

How the drivers are tested: simulators in CI, protocol test servers in containers, and a small hardware bench with cheap real devices. This file is the driver-specific view. It does not duplicate the test area: the simulator design is in [09-testing/03-simulator.md](../09-testing/03-simulator.md), the fault catalogue in [09-testing/04-fault-injection.md](../09-testing/04-fault-injection.md), the conformance suite in [09-testing/02-driver-conformance.md](../09-testing/02-driver-conformance.md), and the hardware-in-the-loop rig and power-cut relays in 09-testing/08 (planned file, listed in the testing README). Where this file names a fault, it uses the 09-testing ids only when they exist there and otherwise says "new".

Principle (from 09-testing): **simulators prove logic and run in CI; real hardware proves the simulators are honest.** A simulator written by us encodes our understanding of the protocol, so it can share our misunderstanding. Every protocol therefore gets at least one real device, and a packet capture from it is kept for replay.

All product names, licences, features and prices below are from memory and UNVERIFIED. Prices are given only as relative cost bands.

## 1. Tiers

```text
 T1  in-process        driver + sim device in one Go test binary, virtual clock          every commit
 T2  protocol server   driver process (gRPC) -> real TCP -> server in a container          every commit / nightly
     in a container     (+ Toxiproxy in between for network faults)
 T3  bench hardware    driver on a bench gateway -> real PLC / MCU on a switch              weekly + before release
 T4  customer site     recorded captures replayed in T2                                    when available
```

| Tier | Gives you | Cannot give you |
|---|---|---|
| T1 | determinism, speed, exact oracle | real TCP behaviour, real device quirks |
| T2 | real sockets, real stack of an independent implementation, network faults | vendor quirks, timing of real CPUs |
| T3 | vendor reality: odd PDU sizes, busy CPUs, firmware behaviour, power cycles | repeatability, parallel runs |
| T4 | the customer's actual device | control |

## 2. T2: protocol servers per driver

Each is a container in the integration compose stack (09-testing/06). Licences matter only for tools we **ship**; these are test-only (libraries-and-licensing.md section 5), but they stay out of shipped images.

| Driver | Primary T2 server | Second opinion | Notes |
|---|---|---|---|
| `sparkplug` | Mosquitto + `skyranger-sim` Sparkplug edge nodes | Eclipse Tahu Java/Python example edge node (UNVERIFIED availability) | an independent Sparkplug implementation guards against a shared misreading of the spec |
| `modbus` | `skyranger-sim` Modbus server (built on `simonvetter/modbus` server, UNVERIFIED) | `pymodbus` server; OpenPLC runtime | the sim must be able to: return exceptions 01 to 04, 06, 0A, 0B on demand, delay responses, send late/duplicate responses, close mid-frame, serve many units on one socket |
| `opcua` | open62541 example/server container | `asyncua` server; sim OPC UA server | server-side faults: bad status codes, session expiry, certificate change, namespace reorder, restart with new subscription state |
| `enip` | libplctag `ab_server` (UNVERIFIED which tag types and PLC models it simulates) | none free that I know of (Rockwell's emulator is paid, Windows) | enough for DINT, REAL, BOOL, arrays; gaps covered by T3 |
| `s7` | Snap7 server (small C++ program with its own server API) | none | DB layout set by the test; return-code faults (0x0A, 0x03) injected by a wrapper or by not creating the DB; PDU size configurable (UNVERIFIED) |

Network faults at T2 use Toxiproxy (reset_peer, latency, bandwidth, timeout, slicer) in front of every server, exactly as 09-testing/04 describes, plus the netem sidecar for true half-open links. Slicer matters most for the Modbus and S7 parsers: split frames at every byte boundary.

```yaml
# docker-compose.drivers.yaml  (sketch, test area owns the real file)
services:
  toxiproxy: { image: shopify/toxiproxy }          # tag pinned in the real file
  modbus-pymodbus:  { build: ./bench/pymodbus, networks: [bench] }
  opcua-open62541:  { build: ./bench/open62541, networks: [bench] }
  enip-abserver:    { build: ./bench/abserver,  networks: [bench] }   # built from pinned libplctag source
  s7-snap7:         { build: ./bench/snap7,     networks: [bench] }
  mosquitto:        { image: eclipse-mosquitto }
  skyranger-sim:    { build: ./cmd/skyranger-sim, networks: [bench] }
  gateway:          { build: ., depends_on: [toxiproxy], networks: [bench] }
```

## 3. T3: hardware bench

Goal: one real device per protocol, cheap, always powered, on a managed switch with a controllable power relay each, so a test script can cut power and cable. Cost bands: **$** under about 50 USD, **$$** about 50 to 300, **$$$** several hundred or more. UNVERIFIED all.

| Device | Protocols it gives us | Cost band | Why it earns a place | Caveat |
|---|---|---|---|---|
| Raspberry Pi (3/4/5) running **OpenPLC Runtime** | Modbus TCP server (and, in recent versions, possibly S7 via Snap7 and EtherNet/IP via a stack, UNVERIFIED) | $ to $$ | the standard cheap soft-PLC; IEC 61131-3 program of our choosing; reboot and power-cut it freely | GPL tool, lab only. Its Modbus behaviour is one implementation's behaviour |
| **ESP32 pulse-meter rig** (the milestone device) with a ground-truth pulse generator | Sparkplug B over MQTT | $ | the real thing for milestone 1 | rig design is owned by 09-testing/08 |
| ESP32 or Arduino running a Modbus TCP **slave** (esp-modbus or similar, UNVERIFIED) | Modbus TCP | $ | cheap second Modbus implementation with real timing and tiny buffers; good for slow, overloaded device behaviour | not representative of industrial PLC firmware |
| Allen-Bradley **Micro820/Micro850** (or any used CompactLogix / MicroLogix on the second-hand market) | EtherNet/IP CIP, Modbus TCP, (Micro800: its own tag model, UNVERIFIED) | $$ to $$$ | the only way to see real CIP behaviour; used CompactLogix is closer to the v1 target (Logix) but costs more and needs Studio 5000 to program | Studio 5000 / CCW tools are licensed software (CCW has a free edition, UNVERIFIED); v1 scope is Logix only, so Micro800 tests the library path but not the exact tag model |
| Siemens **LOGO! 8** (0BA8) | S7 (TSAP-based access) and Modbus TCP (UNVERIFIED versions) | $$ | cheapest real Siemens device with Ethernet | not a full S7: addressing differs (VM addresses), quirks. Gives connection and PDU behaviour only |
| Used **S7-1200** (preferably one with a CPU firmware that has an OPC UA server, UNVERIFIED) | S7comm (PUT/GET), maybe OPC UA | $$$ | the main real target for `s7`; tests the PUT/GET and optimized-DB commissioning traps | TIA Portal is licensed; OPC UA server may need a licence on the CPU |
| **CODESYS runtime** on a Raspberry Pi (or another CODESYS device) | OPC UA server, Modbus TCP, EtherNet/IP adapter (UNVERIFIED which need licences) | $ to $$ | a real OPC UA server stack used by many PLC vendors, to test `opcua` beyond open62541 | the runtime has a per-device licence after a trial period (UNVERIFIED) |
| Managed switch with port control, smart power strip / relays | n/a | $$ | repeatable cable-pull and power-cycle | |

Minimum viable bench (phase 2 to 3): Raspberry Pi with OpenPLC + the ESP32 pulse rig + a Pi with CODESYS or open62541. Add the Micro800 and an S7-1200 or LOGO! when phases 4 and 5 start (implementation-order.md). If the first customer owns a specific PLC, buying a used unit of that model beats any of the above.

```text
                     ┌──────────── managed switch (port control) ────────────┐
                     │                                                        │
  bench gateway ─────┤──  Pi: OpenPLC (Modbus)        ◄── relay 1             │
  (skyrangerd +      │──  ESP32 pulse rig (Sparkplug) ◄── relay 2             │
   drivers, real     │──  Pi: CODESYS (OPC UA)        ◄── relay 3             │
   time)             │──  Micro800 (CIP)              ◄── relay 4             │
                     │──  S7-1200 / LOGO! (S7)        ◄── relay 5             │
                     │──  Toxiproxy host (optional, in path for chosen links) │
                     └────────────────────────────────────────────────────────┘
  control PC / CI runner: drives relays, Toxiproxy, ground-truth generator, collects oracle logs
```

Safety and hygiene: bench network is isolated (no route to real plants), PLC programs are simple and non-actuating (outputs go to LEDs or nothing), all device programs and settings are stored in the repo (`bench/`) so the bench can be rebuilt, and every real-device finding becomes either a simulator change or a recorded capture.

## 4. What each real device must answer

| Question | Device | Method |
|---|---|---|
| Does our Modbus addressing/byte order match a vendor manual? | OpenPLC + one real device with a manual | read documented registers, compare with the manual's example values |
| What PDU size does the real CPU grant, what is the per-request time? | S7-1200 / LOGO! | packet capture during Setup Communication; compare with siemens-s7.md section 4 (UNVERIFIED numbers) |
| Does libplctag pack reads and share one CIP connection? | Micro800 / CompactLogix | Wireshark capture of 200 tags; count TCP sessions and CIP packets |
| Do OPC UA subscriptions survive a CPU restart and a certificate change? | CODESYS | restart runtime during soak |
| Does an ESP32 reboot produce the Sparkplug sequence the driver expects? | ESP32 rig | oracle comparison, 24 h with reboots |
| What happens when the PLC runs out of connections? | S7-1200 / Logix | open connections with other tools until it refuses; check that the driver backs off and does not hammer |

## 5. Driver fault matrix

Every driver runs these. The **expected behaviour** is in contract terms and defined by reliability.md. Fault ids refer to 09-testing/04 where they exist (F-NET-01, 03, 04, 05, 06 are quoted there). Rows marked "new" should be added to that catalogue.

| # | Fault | T1 | T2 | T3 | Expected |
|---|---|---|---|---|---|
| D1 | Clean connection drop (FIN) | yes | yes (F-NET-01) | relay/cable | `disconnected`, items `bad_not_connected`, backoff, reconnect, immediate full scan |
| D2 | Reset (RST) mid-frame | yes | yes (F-NET-03) | | as D1, no panic, partial frame discarded |
| D3 | Half-open link (cable pulled while idle) | | netem | cable pull | detected by TCP keep-alive or protocol keep-alive within 15 s to 1.5 x keep-alive, then as D1 |
| D4 | Latency 500 ms +/- 200 ms | | yes (F-NET-04) | | no state change; no overrun storm; one warning at most |
| D5 | Latency above request timeout | yes | yes | | `bad_comm_failure`, after 3 in a row link lost; no pile-up of requests (concurrency 1) |
| D6 | Fragmented frames (1 byte at a time) | yes | slicer | | parsers correct (Modbus MBAP, S7 TPKT/COTP) |
| D7 | Late response after timeout | yes | proxy delay | | Modbus: reconnect (stream out of sync). Other drivers per their doc |
| D8 | Duplicate / wrong transaction id | yes | | | protocol error, reconnect |
| D9 | One bad address in a block | yes | yes | | only that item `bad_config_error`; neighbours `good`; `degraded` |
| D10 | Device exceptions (Modbus 01 to 04, 06, 0A, 0B; S7 item codes; CIP status) | yes | yes | where reproducible | mapping tables in each driver doc, with `Details` |
| D11 | NaN/Inf, out-of-range for requested type | yes | yes | | `bad_type_mismatch` with last value |
| D12 | Flapping link: down 2 s / up 5 s for 10 min (F-NET-06) | | yes | | bounded reconnect rate; no goroutine/memory growth |
| D13 | Auth / certificate rejected (OPC UA, MQTT) | | yes | | `faulted`, no retry loop, clear `last_error` |
| D14 | Device reboot (power cut) | simulated | simulated | relay | `bad_not_connected`, recover within backoff ceiling plus one scan; Sparkplug: new birth, bdSeq, counter reset handled |
| D15 | Slow device (response 90% of timeout) | yes | | ESP32 slave | no flapping; overrun counter and `degraded` warning per scanning.md |
| D16 | Driver process killed (SIGKILL) | n/a | yes | | host restarts, same handles, all items `bad_not_connected` during outage (new) |
| D17 | Driver hung (stops answering Health) | test hook | yes | | killed and restarted within about 30 s plus backoff (new, reliability F7) |
| D18 | Sink blocked 60 s | yes | yes | | memory flat, `dropped_samples` up, latest values on release, quality transitions kept (new) |
| D19 | Connection resource exhaustion on device | | | yes | backoff, no hammering, no parallel sessions (new) |
| D20 | Write with link cut mid-write | yes | yes | | result `timeout`, outcome unknown, no retry observed on the wire (new) |
| D21 | PLC-side config trap (S7 PUT/GET off, optimized DB; CIP path wrong; tag external access none) | | yes where emulated | yes | `degraded` or retrying configuration error with the checklist hint as in each driver doc |

## 6. Soak tests

Run on the bench gateway (T3) and in compose (T2), per 09-testing/07 (planned file; this section states driver-specific targets only).

| Test | Duration | Load | Pass criteria |
|---|---|---|---|
| Modbus soak | 72 h | OpenPLC + sim, 5 connections, 1000 items, 1 s | RSS growth below 10% after warm-up, goroutines flat, zero unexplained quality flips, `dropped_samples` 0 under normal load |
| OPC UA soak | 72 h | 2000 monitored items, link cut every 10 min | all items recover; no subscription leak on the server (server-side count flat) |
| EtherNet/IP soak | 48 h | 1000 handles, 1 s | handle count and RSS flat, one CIP connection (capture sample) |
| S7 soak | 48 h | 200 bytes across 3 DBs, 1 s | flat memory; one session only |
| Sparkplug soak | 72 h | 200 edge nodes, broker restart every 6 h | no sequence-gap storm; counters match oracle exactly |
| Everything together | 7 days | all above on one gateway | box CPU under 50% on the target hardware, no restarts except injected |

The thresholds are initial guesses and are tightened after the first run produces real numbers.

## 7. Record and replay

Every T3 session records packets (`tcpdump` on the bench switch mirror port) with a short note of device, firmware, and the action performed. Captures are anonymised (09-testing/05, planned) and become **replay fixtures** for T2, so a real vendor quirk found once is tested forever. Captures are the main defence against the "shared misreading" risk of home-made simulators, and for S7comm (no public spec) they are the primary reference.

## Decisions made
1. Four tiers: in-process, protocol servers in containers with Toxiproxy, a small hardware bench, and replayed customer captures. CI runs T1 and T2 only.
2. Every protocol has at least one independent T2 server we did not write (pymodbus, open62541, `ab_server`, Snap7, Tahu/asyncua) plus at least one real T3 device.
3. Minimum bench: Raspberry Pi with OpenPLC, the ESP32 pulse rig, one real OPC UA server (CODESYS or open62541 Pi). Siemens and Allen-Bradley devices are bought when their phases start, preferring the customer's model.
4. One shared driver fault matrix (D1 to D21), results reported in contract terms.
5. Real-device findings become simulator behaviour or recorded fixtures. Captures are the reference for S7.
6. GPL/LGPL test tools run only in test containers outside shipped images.

## Alternatives rejected and why
- **Only simulators**: they encode our beliefs. Real devices catch the surprises.
- **Only hardware**: slow, serial, not CI-able, cannot inject precise faults.
- **Buying a full PLC (ControlLogix, S7-1500) first**: expensive. Micro800/CompactLogix and used S7-1200 cover the code paths; the customer's exact model is bought when a customer exists.
- **PLCSIM / Studio 5000 Logix Emulate in CI**: Windows, licensed, GUI-driven, not scriptable in a Linux pipeline (UNVERIFIED current state).
- **Commercial Modbus/OPC UA simulators**: licence and redistribution unclear; open servers do the job.
- **Kubernetes chaos tools**: same reasoning as 09-testing/04.

## Open questions
- Whether `ab_server` and the Snap7 server are good enough in practice: unknown until the spikes. If not, record/replay of real captures carries more weight.
- Which real PLC models to buy, and their current price, firmware and licence status (all UNVERIFIED).
- Whether OpenPLC's S7 and EtherNet/IP support is usable as a free extra server (UNVERIFIED).
- Whether the bench relays and the ESP32 rig share one control PC with the testing area's rig (09-testing/08), and who owns the bench repository folder.
- Fault ids D1 to D21 should be merged into the 09-testing/04 catalogue with proper ids (that is a change in another area, so noted here and not made).
- Time source on the bench gateway (NTP) for oracle comparisons.
