# Interface Contracts (00-contracts)

These five documents are the cross-cutting contracts for Sky Ranger. Every other design doc, every component, and every agent working in parallel MUST conform to them. If a design needs something these docs forbid, change the contract first (with a decisions.md entry), then build. Never fork a name.

| Doc | Defines | Primary consumers |
|---|---|---|
| [tag-model.md](tag-model.md) | Tag paths, datatypes, quality codes, timestamps, units/scaling, YAML config schema (`Gateway`, `Connection`, `Template`, `TagGroup`), templates (UDTs), derived-tag builtins, the water meter end to end | tag engine, historian, config service, HMI, everybody |
| [messages.md](messages.md) | Event envelope (CloudEvents 1.0 JSON), internal MQTT namespace `skyranger/v1/...`, Sparkplug B usage (`spBv1.0/...`), the `skyranger.pulse_meter.v1` device profile, discovery and approval flow, counter rollover/reset representation | firmware, Sparkplug driver, tag engine, scripting, integrations |
| [driver-interface.md](driver-interface.md) | Go `driver.Driver` interface, gRPC service `skyranger.driver.v1.Driver`, lifecycle and connection states, error model, isolation, backpressure | driver authors, core driver host |
| [api.md](api.md) | REST `/api/v1`, WebSocket `/api/v1/ws` (subprotocol `skyranger.v1`), auth, errors (RFC 9457), pagination, versioning | HMI, external clients, test harness |
| README.md (this file) | Index and naming conventions | everybody |

## System shape these contracts assume

```
 ESP32 / simulators ──Sparkplug B──┐
                                   ▼
                          MQTT broker (Mosquitto)  ◄──── internal bus  skyranger/v1/{gateway_id}/...
                                   ▲                         (observe, scripting, integrations)
 driver processes ──gRPC (UDS)──► skyrangerd (Go): tag engine, alarms, historian writer,
 (sparkplug, modbus, opcua, sim)   config, discovery, REST/WS API, Sparkplug northbound
                                   │
                                   ▼
                         PostgreSQL + TimescaleDB          HMI (React) ── REST + WS
```

- `skyrangerd` is one Go process. Inside it, components exchange the same Go structs that are serialized onto the internal bus. History writes do NOT go through MQTT.
- Drivers are separate processes talking gRPC over a Unix domain socket. The same Go interface can be hosted in-process for tests.
- The MQTT broker carries both Sparkplug B (`spBv1.0/...`) and the internal bus (`skyranger/v1/...`). They never share a topic root.

## Naming conventions (normative)

### Identifiers
All of these use the **segment grammar**: `^[a-z0-9][a-z0-9_-]{0,62}$` (lowercase ASCII letters, digits, `_`, `-`; starts with letter or digit; 1 to 63 chars).

| Identifier | Example | Notes |
|---|---|---|
| `gateway_id` | `gw01` | Unique per deployment. Also the northbound Sparkplug edge node id. |
| connection name | `local-sparkplug` | `metadata.name` of a `Connection` |
| template name | `pulse_water_meter` | `metadata.name` of a `Template` |
| tag path segment | `flow_rate` | see tag-model.md |
| alarm name | `leak` | unique per tag |
| driver name | `sparkplug` | binary is `skyranger-driver-{name}` |

The reserved root segment `_system` is the only place a leading underscore is allowed (`_system/connections/local-sparkplug/state`).

### Tag paths
`site1/pumphouse/meter01/flow_rate`: segments joined by `/`, no leading or trailing `/`, max 16 segments, max 255 bytes. Comparison is byte-exact. A **prefix** `a/b` matches `a/b` itself and anything under `a/b/`, never `a/bc`. Paths are mutable; `tag_id` (UUIDv7) is the immutable key.

### Field names and encodings
- JSON and YAML field names: `snake_case`. Go: `CamelCase` with `json:"snake_case"` tags. Protobuf: `snake_case` fields, `UPPER_SNAKE` enum values prefixed by the enum name.
- Enum values on any JSON/YAML wire: lowercase `snake_case` strings (`good`, `bad_not_connected`, `active_unacked`).
- Timestamps: RFC 3339, UTC, `Z` suffix, millisecond precision: `2026-11-07T14:03:22.123Z`. Protobuf uses `google.protobuf.Timestamp`. Database uses `timestamptz`.
- Durations: Go duration strings everywhere in JSON/YAML and query params (`"250ms"`, `"30s"`, `"2h"`). Protobuf uses `google.protobuf.Duration`. No `_ms` integer fields.
- 64-bit integers (`int64`, `uint64`) are JSON **strings** (proto3 JSON mapping); all other numbers are JSON numbers.
- IDs: UUIDv7 (RFC 9562) as lowercase hyphenated strings.
- Units: UCUM case-sensitive codes (`L`, `L/min`, `m3`, `Cel`, `kW.h`, `%`).

### The canonical value object
Everywhere a current or historical value appears (internal bus, REST, WebSocket, scripting), it is this shape. Do not invent variants.

```json
{
  "tag_id": "01933b7e-8f1a-7c3d-9a2b-4f6e8d1c2b3a",
  "path": "site1/pumphouse/meter01/flow_rate",
  "value": 12.0,
  "quality": "good",
  "ts": "2026-11-07T14:03:22.123Z",
  "ts_source": "2026-11-07T14:03:22.100Z",
  "ts_gateway": "2026-11-07T14:03:22.123Z"
}
```

`ts_source` is omitted when unknown. History rows may omit `tag_id`/`path` when the enclosing response already states them.

### Topic roots and versions
| Thing | Version marker |
|---|---|
| Internal MQTT bus | `skyranger/v1/{gateway_id}/...` |
| Event types | `skyranger.<area>.<name>.v1` |
| REST | `/api/v1/...` |
| WebSocket subprotocol | `skyranger.v1` |
| gRPC package | `skyranger.driver.v1` |
| Config files | `apiVersion: skyranger/v1` |
| Device profile | `skyranger.pulse_meter.v1` |

Breaking change = new major everywhere that changed. Additive fields are not breaking; all consumers MUST ignore unknown fields.

### Verification marker
Any claim about an external standard we have not checked against the spec text is marked **UNVERIFIED**. Before implementing code that depends on an UNVERIFIED item, check the spec and update the doc.

## Decisions made
1. One shared value object (`tag_id`, `path`, `value`, `quality`, `ts`, `ts_source`, `ts_gateway`) across bus, API, WS and scripting.
2. One identifier grammar for everything human-named, chosen to be safe in MQTT topics, URLs, file names, and SQL without quoting.
3. Durations are always Go duration strings; timestamps always RFC 3339 UTC ms; 64-bit ints as JSON strings.
4. `skyrangerd` is a single process; drivers are out-of-process over gRPC; the MQTT internal bus is for observation, scripting and integration, not for the history path.
5. Every versioned surface carries an explicit `v1`.

## Alternatives rejected and why
- **Case-insensitive or mixed-case paths**: invites `Pump1` vs `pump1` duplicates and DB collation bugs. Lowercase-only with a separate `display_name` is simpler.
- **Dots as path separators** (`site1.pumphouse.meter01`): would not map 1:1 onto MQTT topic levels.
- **Integer millisecond durations** (`_ms` fields): two formats for one concept; strings are readable in YAML and the API alike.
- **Microservices for tag engine/historian/alarms**: more failure modes and network hops for no benefit at single-gateway scale.

## Open questions
- Module path for Go packages (`github.com/<org>/skyranger`) is not chosen; docs use `skyranger/...` as a placeholder.
- Whether a public docs domain will exist for RFC 9457 `type` URIs (currently relative URIs served by the gateway).
- Multi-gateway addressing (`{gateway_id}:{path}`) is reserved syntax but not specified until cloud sync is designed.
