# Decisions Log

Each entry records a choice and why it was made.

## 1. Edge-first architecture
The gateway runs fully on a single on-prem box with no internet. Cloud sync is an optional add-on, never required for the gateway to function.
Why: plants need control that survives network outages, and edge-first keeps cloud as an add-on rather than a rewrite.

## 2. Stack
- Go for the gateway core and drivers
- Python for user scripting
- TypeScript + React for the web HMI
- MQTT with Sparkplug B as the message backbone
- PostgreSQL + TimescaleDB for configuration and history
- Everything containerized

## 3. Open standards, no lock-in
OPC UA, MQTT/Sparkplug, plain SQL, and configuration in exportable JSON/YAML.
Why: customers must be able to leave easily, which is a selling point against Siemens/Rockwell lock-in.

## 4. Tag discovery: hybrid
Devices announce themselves and the gateway lists them as discovered, but nothing becomes a live tag until it is approved and configured (units, scaling, alarm limits). Configuration is the source of truth.

## 5. Business sequencing
Integrator first, product later. Licensing model deferred until after two or three real projects.

## 6. First milestone and first sensor
One ESP32 pulse-counter water meter (1 pulse = 1 liter) plus simulated devices.
- The device publishes the raw cumulative count; the gateway computes flow rate.
- Handle counter rollover and reboots.
- Heartbeat distinguishes "no flow" from "dead device".
- Initial tags: total volume, flow rate, device health, plus a continuous-flow (leak) alarm.

## 7. Testing
A device simulator (MQTT, Modbus TCP, OPC UA) with fault injection: dropped connections, stale values, bad quality, bursts.

## 8. Architecture drafted (status: DRAFT, not yet approved)
A full architecture was drafted on 2026-11-08 by AI agents under the author's direction, in docs/00-contracts (binding contracts) and docs/01 to docs/10 (one area each), reviewed in docs/11-review. Nothing below is final until the author approves it; the docs lead, code follows.
- Contracts: one shared value object, UUIDv7 tag_id plus renameable path, uint16 quality codes, versioned YAML config (Gateway, Connection, Template, TagGroup), REST + WebSocket API under /api/v1, drivers as one process per connection over gRPC.
- Counters: device sends pulse_count plus boot_count; the gateway's counter_total handles rollover and reboot.
- Scripting: Python in separate sandboxed worker processes; plugins out of process and signed.
- Storage: idempotent history writes, disk-backed store-and-forward only when PostgreSQL is down, one 15-minute rollup.
- Deployment: PostgreSQL is the config store mirrored to git; health-gated updates with rollback; export-everything exit plan.
- Drivers: libplctag for EtherNet/IP, spike gos7 for S7, licence gate blocks GPL/AGPL in shipped binaries; s7: and enip: address schemes still need an entry here.
- Delivery: Claude Code agents own directories; contracts first; M1 vertical slice, alarms in M2.

## 9. Pending decisions (need the author)
The independent review (docs/11-review/contract-issues.md) found 57 issues: 12 blocking, 31 should-fix, 14 minor. Ten must be settled before any code; the review recommends an answer for each but none is adopted yet:
1. Duration grammar (the contract's own 400d example is invalid Go)
2. counter_total v1.1 (lost boot_count write can look like a rollover; double counting)
3. Ordering rule for gateway-timestamped values
4. History key uniqueness at millisecond precision
5. History aggregates (delta undercounts, avg not time-weighted)
6. One DDL owner, no duplicate table definitions
7. _system tag registry
8. TimescaleDB licence check and a plain-PostgreSQL CI path
9. MQTT broker TLS topology for M1 versus production
10. M1 API additions
Also open: disk-full policy, retention promise, driver isolation level in v1, supported-tag claim, scripting access model (decide by M4).

## Open items
- Approve or amend the pending decisions above, then update the contracts and golden files.
- Verify everything marked UNVERIFIED in the docs (licences, library maturity, Sparkplug and OPC UA details, IEC 62443 clause numbers).
- Create the Go module and make targets, then replace the TODO build commands in CLAUDE.md.
