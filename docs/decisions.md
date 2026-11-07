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

## Open items
- Tag engine data model and interface contracts
- Deployment and updates
- Security
- Testing strategy details
- Agent team for Claude Code
- Roadmap
