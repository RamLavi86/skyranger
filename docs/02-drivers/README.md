# 02 Drivers and Protocols

Status: design, v1. Binding inputs: [../00-contracts/driver-interface.md](../00-contracts/driver-interface.md), [../00-contracts/tag-model.md](../00-contracts/tag-model.md), [../00-contracts/messages.md](../00-contracts/messages.md). This area adds detail on top of those contracts. Where it needs something they do not define, it says so in Open questions and does not quietly change them.

A driver turns one southbound protocol into raw, quality-stamped samples for the tag engine. Drivers own sockets, sessions, retries, request batching and native-type decoding. They do not own paths, scaling, deadbands, alarms or history.

## Files

| File | What it covers |
|---|---|
| [framework.md](framework.md) | Driver SDK, plugin model (binaries, `--info`, third-party drivers), driver host supervision, process isolation and resource limits, type conversion rules, repo layout |
| [scanning.md](scanning.md) | Polling vs subscription per protocol, scan classes, the poll scheduler, request batching (block planner), sample flushing to the host |
| [reliability.md](reliability.md) | Connection state machine, reconnect and backoff, error classification, failure scenario table, quality mapping summary across all drivers, write failure semantics |
| [sparkplug.md](sparkplug.md) | `sparkplug` driver: Sparkplug B host application consuming ESP32-class edge nodes; birth/death, seq, aliases, rebirth, discovery, writes |
| [modbus-tcp.md](modbus-tcp.md) | `modbus` driver: own implementation, address grammar with worked examples, byte orders, exception mapping |
| [opcua.md](opcua.md) | `opcua` driver (client) and the northbound OPC UA server (`skyranger-opcua-server`) |
| [siemens-s7.md](siemens-s7.md) | `s7` driver: S7comm for S7-300/400/1200/1500 (PUT/GET), address grammar, PDU planning; when to use OPC UA instead |
| [ethernet-ip.md](ethernet-ip.md) | `enip` driver: Allen-Bradley Logix symbolic tags over EtherNet/IP via libplctag |
| [libraries-and-licensing.md](libraries-and-licensing.md) | Candidate libraries per protocol, licence policy for a sellable product, build-vs-buy per protocol |
| [implementation-order.md](implementation-order.md) | Phased build order with exit criteria |
| [test-bench.md](test-bench.md) | Simulators in CI, fault injection, hardware bench (OpenPLC, cheap real PLCs, ESP32 pulse rig), soak tests |

## Driver catalogue (v1 and planned)

| Driver name | Binary | Protocol | Model | Phase | Build or buy |
|---|---|---|---|---|---|
| `sim` | `skyranger-driver-sim` | none (generators + faults) | poll | 0 | build |
| `sparkplug` | `skyranger-driver-sparkplug` | MQTT 3.1.1 + Sparkplug B | push | 1 (milestone 1) | build on Paho MQTT + Tahu `.proto` |
| `modbus` | `skyranger-driver-modbus` | Modbus TCP | poll | 2 | build (small protocol, full control) |
| `opcua` | `skyranger-driver-opcua` | OPC UA client | subscribe (poll fallback) | 3 | buy (library `gopcua/opcua`) |
| `enip` | `skyranger-driver-enip` | EtherNet/IP CIP, Logix symbolic | poll | 4 | buy (libplctag via cgo) |
| `s7` | `skyranger-driver-s7` | S7comm (ISO-on-TCP) | poll | 5 | spike `gos7`, else build |
| (northbound) | `skyranger-opcua-server` | OPC UA server | n/a | 6 | buy (library), not a driver |

For protocols we do not implement (BACnet, DNP3, IEC 60870-5-104, PROFINET IO, Modbus RTU on serial), the answer for an integrator is a third-party protocol gateway that speaks OPC UA or Modbus TCP. See [libraries-and-licensing.md](libraries-and-licensing.md).

## The ten rules every driver follows

1. One process per connection. A crash or hang affects one connection only (contract section 1).
2. `Configure` does no I/O. `Connect` returns at once. The driver owns reconnecting and reports state through `Sink.ConnectionState`.
3. Value-path failures become quality codes, never call errors (contract section 4). Each driver doc has a mapping table.
4. Raw values only. Widening type conversions only. Anything that does not fit exactly becomes `bad_type_mismatch`.
5. Polled drivers schedule by `Item.Interval`, batch reads, and skip overrun cycles instead of queueing them.
6. Writes are not retried by anyone. A write timeout means "outcome unknown".
7. Backoff is exponential with jitter and has a ceiling. Auth and config rejections go to `faulted` and stay there.
8. The driver never stamps `ts_gateway`. It sets `ts_source` only when the device actually supplied a time.
9. Every driver passes the conformance suite in both in-process and gRPC modes before it ships.
10. No GPL/AGPL code in a shipped binary. Every dependency is listed in the licence table with its verification status.

## Decisions made
1. Six drivers in v1 scope (`sim`, `sparkplug`, `modbus`, `opcua`, `enip`, `s7`). The OPC UA server is a separate northbound process and not a driver.
2. Build order: sim, then sparkplug, modbus, opcua, enip, s7, OPC UA server. Milestone 1 needs only the first two.
3. Unsupported protocols are reached through third-party gateways speaking OPC UA or Modbus TCP. We do not write long-tail drivers.
4. The driver name for Allen-Bradley is `enip`, with a `plc` family setting. v1 supports only the Logix family.

## Alternatives rejected and why
- **One multi-protocol driver binary**: one bug would take down every protocol, and it would conflict with process-per-connection.
- **Starting with OPC UA client because it is "universal"**: milestone 1 is Sparkplug, and Modbus is the most common protocol on small water and utility sites.
- **Writing BACnet/DNP3 early**: each is a large protocol with a narrow customer base at this stage.

## Open questions
- Whether a demand-driven driver (for example Modbus RTU over a serial port on the gateway box) should come before S7. The answer depends on the first customers' hardware.
- The contracts list only `sparkplug`, `modbus`, `opcua` and `sim` address syntaxes in section 5. `s7` and `enip` add schemes `s7:` and `enip:`. Adding them is additive, but the contract table should be updated through a decisions.md entry.
