# Tag Model Contract

Status: v1 contract. Conventions (identifier grammar, timestamp and duration formats, the canonical value object) are defined in [README.md](README.md) and are not repeated here.

A **tag** is a named, typed, quality-stamped value with history. Every value the platform shows, stores, alarms on, or exports is a tag value. Configuration is the source of truth: nothing becomes a tag until it is in an applied config revision (decision 4).

## 1. Tag identity and paths

| Field | Type | Mutable | Purpose |
|---|---|---|---|
| `tag_id` | UUIDv7 | never | Primary key. History, alarms, and audit reference this. |
| `path` | tag path | yes (rename) | Human address, used in HMI, API queries, MQTT topics. |
| `display_name` | string, any UTF-8, max 128 | yes | What the HMI shows. |

### Path grammar (ABNF)

```abnf
tag-path  = segment *15( "/" segment )            ; 1..16 segments, <= 255 bytes total
segment   = ( ALNUM-LC ) *62( ALNUM-LC / "_" / "-" )
ALNUM-LC  = %x61-7A / DIGIT                       ; a-z 0-9
system-path = "_system" 1*15( "/" segment )       ; reserved, gateway-generated only
```

Rules:
- Paths are lowercase, byte-compared, and never contain `+`, `#`, `/`-at-ends, spaces, or dots. This makes them safe as MQTT topic levels, URL query values, file names, and SQL text without escaping.
- Recommended (not enforced) hierarchy: `{site}/{area}/{equipment}/{signal}`, e.g. `site1/pumphouse/meter01/flow_rate`.
- Prefix match is segment-wise: prefix `site1/pump` does NOT match `site1/pumphouse/...`.
- `_system/...` tags are created by the gateway (connection state, driver stats, gateway health). They are read-only, never in user config, and may change between versions.
- Rename: changing a path keeps `tag_id`, so history continues. The old path is not an alias (no redirects); clients holding paths must re-resolve. The gateway clears the retained MQTT value at the old topic (see messages.md).

### Tag references inside config

Inside `derived.inputs` and template bodies, a **reference** (not a path) is used:
- `total_volume` or `raw/pulse_count`: relative to the containing instance root (template instance) or `base_path` (TagGroup).
- `/site1/other/thing`: leading `/` means absolute tag path `site1/other/thing`.

References are resolved to `tag_id` at apply time. A dangling reference fails validation; the revision is not applied.

## 2. Tag kinds

| `kind` | Value comes from | Writable? |
|---|---|---|
| `source` | a driver via a `Connection` + `address` | only if `writable: true` and driver supports write |
| `derived` | a built-in function of other tags (section 7), evaluated in the tag engine | only where the function defines write semantics (`counter_total`) |
| `memory` | operator/API writes; persisted; survives restart | yes (role `operator`) |

Python-scripted tags are deliberately not a v1 kind. When scripting lands, it will add `kind: script` without changing the others.

## 3. Data types

The tag engine has exactly six value types. Drivers map native types onto these (widening only; a value that cannot be represented exactly is delivered with quality `bad_type_mismatch`).

| `datatype` | Go | Protobuf (`Variant`) | JSON wire | Notes |
|---|---|---|---|---|
| `bool` | `bool` | `bool_value` | `true`/`false` | |
| `int64` | `int64` | `int64_value` | **string** `"-42"` | 64-bit ints as strings (proto3 JSON rule) |
| `uint64` | `uint64` | `uint64_value` | **string** `"4294967295"` | counters |
| `float64` | `float64` | `float64_value` | number | NaN/Inf never leave the engine: value becomes `null`, quality `bad_type_mismatch` |
| `string` | `string` | `string_value` | string | UTF-8, max 4096 bytes |
| `datetime` | `time.Time` (UTC) | `datetime_value` (`google.protobuf.Timestamp`) | RFC 3339 string | |

`value` is `null` only when quality class is `bad` and no last value is held, typically `bad_waiting_for_initial_data`. Otherwise a bad-quality sample carries the last known value so the HMI can show it greyed out.

Optional `enum_labels` (map of integer to label) can be attached to `int64` tags for display only: `{0: offline, 1: stale, 2: ok}`. Stored values remain integers.

Go representation (package `skyranger/pkg/tag`):

```go
type DataType uint8

const (
    TypeBool DataType = iota + 1
    TypeInt64
    TypeUint64
    TypeFloat64
    TypeString
    TypeDateTime
)

// Variant holds exactly one typed value. Zero Variant (Type == 0) means null.
type Variant struct {
    Type DataType
    b    bool
    i    int64
    u    uint64
    f    float64
    s    string
    t    time.Time
}

type Value struct {
    TagID     uuid.UUID
    Path      string
    V         Variant
    Quality   Quality
    TS        time.Time  // effective timestamp (see section 5)
    TSSource  time.Time  // zero if unknown
    TSGateway time.Time
}
```

## 4. Quality codes

A quality is a `uint16` code. **Class = code >> 6**: 0 good, 1 uncertain, 2 bad. Codes 192+ are reserved. Database column type: `smallint`. JSON wire: the name string.

| Code | Name | Class | Meaning | Likely OPC UA equivalent (UNVERIFIED mapping) |
|---|---|---|---|---|
| 0 | `good` | good | Fresh value from a live source | Good |
| 1 | `good_manual` | good | Value set by operator (memory tag, manual override) | GoodLocalOverride |
| 64 | `uncertain` | uncertain | Generic, use a specific code when possible | Uncertain |
| 65 | `uncertain_last_known` | uncertain | Link to the data source is up, but the source is offline or stale (`stale_after` exceeded, Sparkplug NDEATH). Value is the last one received | UncertainLastUsableValue |
| 66 | `uncertain_eng_range_exceeded` | uncertain | Engineering value outside `eng_range` | UncertainEngineeringUnitsExceeded |
| 67 | `uncertain_initial_value` | uncertain | Derived value still warming up (e.g. `rate` window not yet filled) | UncertainInitialValue |
| 128 | `bad` | bad | Generic | Bad |
| 129 | `bad_not_connected` | bad | The driver connection (gateway to broker/PLC/server) is down | BadNotConnected |
| 130 | `bad_comm_failure` | bad | Connection up but reads of this item fail (timeouts, exceptions) | BadCommunicationError |
| 131 | `bad_device_failure` | bad | Device reports its own fault for the point | BadDeviceFailure |
| 132 | `bad_config_error` | bad | Address invalid, point no longer exists on device, dangling input | BadConfigurationError |
| 133 | `bad_out_of_service` | bad | Tag or connection disabled in config | BadOutOfService |
| 134 | `bad_waiting_for_initial_data` | bad | Configured, no value received yet | BadWaitingForInitialData |
| 135 | `bad_type_mismatch` | bad | Value could not be represented in the tag datatype | BadTypeMismatch |

Rules:
- **Device dead vs link down.** `uncertain_last_known` means "we can still hear the network, the device went quiet". `bad_not_connected` means "we cannot hear the network". The water meter health tag relies on this distinction.
- **Derived propagation.** A derived output's quality is the worst input quality (bad > uncertain > good); ties go to the first input in declaration order, and that input's specific code is copied. A function may further degrade quality (e.g. `rate` emits `uncertain_initial_value`) but never improve it.
- **Alarms on quality** are possible (`type: quality`, section 8).
- **Mapping from OPC UA StatusCode** values to these codes is done by the OPC UA driver. The numeric OPC UA codes are not part of this contract. The symbolic names in the table above are from memory and must be checked against OPC UA Part 4/Part 8 before the OPC UA driver is written (UNVERIFIED).

```go
type Quality uint16

const (
    QualityGood              Quality = 0
    QualityGoodManual        Quality = 1
    QualityUncertain         Quality = 64
    QualityUncertainLastKnown Quality = 65
    QualityUncertainEngRangeExceeded Quality = 66
    QualityUncertainInitialValue     Quality = 67
    QualityBad               Quality = 128
    QualityBadNotConnected   Quality = 129
    QualityBadCommFailure    Quality = 130
    QualityBadDeviceFailure  Quality = 131
    QualityBadConfigError    Quality = 132
    QualityBadOutOfService   Quality = 133
    QualityBadWaitingForInitialData Quality = 134
    QualityBadTypeMismatch   Quality = 135
)

func (q Quality) Class() uint16 { return uint16(q) >> 6 } // 0 good, 1 uncertain, 2 bad
```

## 5. Timestamps

Every value carries up to three timestamps:

| Field | Set by | Meaning |
|---|---|---|
| `ts_source` | device/driver | When the device says it measured. Optional. Not trusted blindly. |
| `ts_gateway` | driver host, at receive | When `skyrangerd` received the sample. Always present. Gateway clock is assumed NTP-disciplined (the gateway should run chrony and serve NTP to the plant LAN, since there may be no internet). |
| `ts` | tag engine | **Effective timestamp.** Used for history ordering, aggregation, alarms, and display. |

Selection of `ts` per `Connection.spec.timestamp_source`:

- `gateway`: `ts = ts_gateway`.
- `device`: `ts = ts_source` if present, else `ts_gateway`.
- `auto` (default): `ts = ts_source` if present AND `ts_source >= 2020-01-01T00:00:00Z` AND `|ts_source - ts_gateway| <= Gateway.spec.timestamps.max_clock_skew` (default `10s`); otherwise `ts = ts_gateway` and the connection's `_system/connections/{name}/clock_skew_rejections` counter increments. This protects history from an ESP32 that booted without time sync and reports 1970.

Other rules:
- Derived values take `ts` of the input sample that triggered evaluation. Timer-driven evaluations (e.g. `rate` decaying toward zero) use the evaluation time as `ts` and `ts_gateway`, and omit `ts_source`.
- History is keyed on `(tag_id, ts)`. Late or out-of-order samples are accepted into history, but a sample with `ts` older than the tag's current `ts` does not replace the current value (it is history-only).
- All storage and wire timestamps are UTC. `Gateway.spec.timezone` affects display and calendar aggregation (e.g. daily totals) only.
- Store-and-forward (backfilled historical data) is out of scope for v1. The skew rule above would reject it; a future `is_historical` path will bypass it.

## 6. Units, scaling, ranges, deadbands

- `unit`: UCUM case-sensitive code (`L`, `L/min`, `m3`, `Cel`, `kPa`, `kW.h`, `%`, `1` for dimensionless, `s`). Free text is rejected so exports stay machine-readable.
- `scale` (source tags only) converts raw driver value to engineering value. Output datatype is the tag `datatype`.

```yaml
scale:
  type: linear          # eng = raw * multiplier + offset
  multiplier: 0.1
  offset: 0
# or
scale:
  type: range           # 4-20 mA style
  raw_low: 4000
  raw_high: 20000
  eng_low: 0
  eng_high: 10
  clamp: false          # if true, clamp to eng_low..eng_high instead of flagging
```

- `eng_range: {low, high}`: expected engineering range. Values outside it get `uncertain_eng_range_exceeded` (unless clamped). Also used by HMI for gauges.
- `deadband: {type: absolute | percent, value}`: a new sample is propagated (bus, WS, alarms) only if the value moved by more than the deadband from the last propagated value, or quality changed. `percent` is of `eng_range` span. Default: none (every change propagates). Unchanged repeats never propagate but do reset `stale_after`.
- `stale_after: <duration>`: if no sample (changed or not) arrives within this time, the tag's quality becomes `uncertain_last_known` with the last value held. Default: unset (never stale).

## 7. Derived tags

Derived tags use a **fixed set of built-in functions** implemented in Go and unit-tested. No expression language in v1.

```yaml
derived:
  fn: <builtin name>
  inputs: { <input name>: <tag reference>, ... }
  params: { ... }
  eval_interval: 10s     # optional timer evaluation in addition to on-input-change
```

### Built-ins for v1

| `fn` | Inputs | Params | Output | Behavior |
|---|---|---|---|---|
| `counter_total` | `count` (uint64 or int64), `epoch` (optional, any type) | `modulus` (uint64, 0 = never wraps), `multiplier` (float64, default 1), `max_delta` (uint64, plausibility limit per sample), `reset_baseline` (`zero` \| `first_sample`, default `zero`), `initial_total` (float64, default 0) | float64, monotonic | Turns a raw cumulative device counter into a gateway-owned total that survives device reboots and rollovers. Algorithm below. Writable: a write sets the total (meter register alignment), audited. |
| `rate` | `total` (numeric, monotonic) | `window` (duration, default `5m`), `per` (duration, default `1m`), `min_samples` (default 2) | float64 | `(total(now) - total(now - window)) / window * per`, from a sliding sample buffer. `uncertain_initial_value` until `window` of data exists. Needs `eval_interval` so it decays to 0 when the counter stops. |
| `liveness` | `heartbeat` (any, must change every heartbeat), `online` (bool) | `stale_after` (duration) | int64, `0` offline, `1` stale, `2` ok | `online == false` or online input bad: 0. Heartbeat not changed within `stale_after`: 1. Else 2. Its own quality is always `good` because it reports health rather than a measurement. |
| `continuous_activity` | `total` (numeric, monotonic) | `max_gap` (duration), `duration` (duration) | bool | True when `total` has increased at least once in every `max_gap` interval for at least `duration`. Tracks `active_since` (first increase after a gap > `max_gap`). Detects slow leaks that a 1-litre-resolution `rate` would read as zero. |
| `select` | `a`, `b` | none | input type | Output `a` if its quality is good, else `b`. Redundant sensors. |
| `scale` | `x` | same as section 6 `scale` | float64 | For scaling derived values. |

#### `counter_total` algorithm (normative)

State persisted per tag: `last_raw` (uint64), `last_epoch` (Variant), `total` (float64, in raw counts times multiplier). State is checkpointed atomically, at most every `1s` and at shutdown, to the `derived_state` table. Because the device count is cumulative, a crash that loses the last second of checkpoints loses nothing: the next sample's delta is computed from the checkpointed `last_raw` and includes the missing counts.

```text
on sample (raw, epoch) with good or uncertain quality:
  if no state:                                   # first sample after commissioning
      last_raw = raw; last_epoch = epoch; total = initial_total; emit total; return
  if epoch input bound and epoch != last_epoch:  # device rebooted, counter restarted
      delta = (reset_baseline == zero) ? raw : 0
      emit event counter_discontinuity{kind: reset}
  else if raw >= last_raw:
      delta = raw - last_raw
  else if modulus > 0:                           # wrapped
      delta = (modulus - last_raw) + raw
      emit event counter_discontinuity{kind: rollover}
  else:                                          # went backwards, no epoch change
      delta = (reset_baseline == zero) ? raw : 0
      emit event counter_discontinuity{kind: reset}
  if max_delta > 0 and delta > max_delta:
      delta = 0
      emit event counter_discontinuity{kind: implausible}
      output quality = uncertain (this sample only)
  total += delta * multiplier
  last_raw = raw; last_epoch = epoch
  emit total
on sample with bad quality: do not update state; output takes input quality, value = total
on write(v): total = v; emit event counter_discontinuity{kind: manual_adjust}; audit
```

Counts that occurred between the last sample received from the old epoch and the device reboot are lost and cannot be recovered. The `reset` event records this so reports can flag the period.

## 8. Alarms (tag-attached)

Alarm definitions live on tags. The alarm state machine (normal, `active_unacked`, `active_acked`, `cleared_unacked`, plus `shelved`, `disabled`) belongs to the alarm engine. Its wire shape is in messages.md and api.md.

```yaml
alarms:
  - name: leak                 # segment grammar, unique per tag
    type: equal                # high | low | equal | not_equal | quality
    setpoint: true             # high: v > sp, low: v < sp, equal/not_equal: v ==/!= sp
    deadband: 0                # numeric types, for high/low clear
    on_delay: 0s               # condition must hold this long before activation
    off_delay: 0s
    severity: high             # low | medium | high | critical
    message: "Continuous flow detected: possible leak"
    enabled: true
```

`type: quality` activates when the tag's quality class is not good for `on_delay`. Each configured alarm gets an immutable `alarm_id` (UUIDv7) the same way tags get `tag_id`.

## 9. Configuration schema (YAML)

Config is a set of YAML documents, each with `apiVersion: skyranger/v1`, `kind`, `metadata`, `spec` (Kubernetes-style envelope, chosen because it is familiar and validates cleanly with JSON Schema). Kinds in v1: `Gateway`, `Connection`, `Template`, `TagGroup`.

The authoritative store is PostgreSQL (`config_revisions`, one row per applied revision). YAML export/import is lossless: **export then import of the same content produces an empty diff.** Secrets never appear in config. They are referenced (`secret://mqtt/gateway`) and resolved from the secret store, and export never includes their values.

Apply semantics: a submitted set of documents is validated as a whole (schema, references, driver `ValidateConfig`, path collisions), diffed against the current revision, and applied atomically as revision N+1, or rejected with no change. The tag engine hot-reloads; unchanged tags keep their state and subscriptions.

### Gateway

```yaml
apiVersion: skyranger/v1
kind: Gateway
metadata:
  name: gw01                       # = gateway_id
spec:
  display_name: "Example Water Co. - Pumphouse"
  timezone: Europe/London          # IANA, display/calendar only
  timestamps:
    max_clock_skew: 10s
  history:
    default_retention: 400d
  sparkplug:
    host_id: skyranger-gw01        # primary host id used for STATE (see messages.md)
```

### Connection

```yaml
apiVersion: skyranger/v1
kind: Connection
metadata:
  name: local-sparkplug
spec:
  driver: sparkplug                # driver name, see driver-interface.md
  enabled: true
  timestamp_source: auto           # auto | device | gateway
  discovery:
    enabled: true
  settings:                        # driver-specific, validated by driver's JSON Schema
    broker_url: mqtt://mosquitto:1883
    client_id: skyranger-gw01-spb
    credentials: secret://mqtt/gateway
    group_ids: [skyranger]
```

### Tag definition fields (used in TagGroup `tags` and Template `tags`)

| Field | Required | Applies to | Notes |
|---|---|---|---|
| `name` | yes | all | Relative path, 1+ segments (`flow_rate`, `raw/pulse_count`) |
| `id` | no | all | `tag_id`. Present in exports. On import, if absent, match by full path; if no match, a new UUIDv7 is assigned |
| `kind` | yes | all | `source` \| `derived` \| `memory` |
| `datatype` | yes | all | section 3 |
| `display_name`, `description` | no | all | |
| `unit` | no | all | UCUM |
| `enum_labels` | no | int64 | |
| `source.connection` | source | source | Connection name |
| `source.address` | source | source | Driver-specific string (driver-interface.md section 5) |
| `source.interval` | no | source | Poll/publish interval hint for polled drivers (default `1s`) |
| `scale`, `eng_range`, `deadband`, `stale_after` | no | numeric | section 6 |
| `derived` | derived | derived | section 7 |
| `initial_value` | no | memory | |
| `writable` | no | source, memory | default `false` for source, `true` for memory |
| `write_limits` | no | numeric writable | `{low, high}`; out-of-range writes rejected by API |
| `history` | no | all | `{enabled: true, mode: on_change \| sampled, interval: 1m (sampled), max_interval: 15m, deadband: {...}}`. `max_interval` forces a row even if unchanged so gaps are distinguishable from flat lines. Default `{enabled: true, mode: on_change, max_interval: 15m}` |
| `alarms` | no | all | section 8 |
| `enabled` | no | all | default `true`; disabled tags report `bad_out_of_service` |

### TagGroup

```yaml
apiVersion: skyranger/v1
kind: TagGroup
metadata:
  name: pumphouse
spec:
  base_path: site1/pumphouse
  instances:                       # template instances
    - name: meter01                # instance root = site1/pumphouse/meter01
      template: pulse_water_meter
      params: { ... }
      overrides:                   # optional per-tag field overrides
        flow_rate:
          deadband: { type: absolute, value: 0.1 }
      tag_ids:                     # written by export, optional on import
        total_volume: 01933b7e-8f1a-7c3d-9a2b-4f6e8d1c2b3a
  tags:                            # plain (non-template) tags under base_path
    - name: pump_room_temp
      kind: source
      datatype: float64
      unit: Cel
      source: { connection: plc1, address: "modbus://1/ir/30001?type=int16" }
      scale: { type: linear, multiplier: 0.1, offset: 0 }
```

## 10. Templates (UDTs)

A `Template` is a reusable set of tag definitions plus typed parameters. An instance materializes tags at `{base_path}/{instance name}/{tag name}`.

- Params are typed: `string`, `int64`, `uint64`, `float64`, `bool`, `duration`. Each has `required: true` or a `default`.
- **Substitution** happens on the YAML node tree before decoding. A scalar whose entire text is `${param}` is replaced with the typed param value. `${param}` embedded inside a longer string is interpolated as text. There are no expressions, conditionals, or loops. Unknown `${...}` fails validation. Literal `${` is written `$${`.
- Instances are **live-linked**: editing a template changes all its instances in the next revision, and the apply diff shows every affected tag. Instance `overrides` win over template fields.
- Templates do not nest in v1.
- Tag ids of instance tags are stable per (instance, template tag name). Renaming a tag inside a template is a delete plus create (new `tag_id`) unless the export's `tag_ids` map is edited to carry the id over.

## 11. Worked example: ESP32 pulse water meter, end to end

**Device (see messages.md section 6, profile `skyranger.pulse_meter.v1`):** ESP32 is a Sparkplug B edge node `skyranger/esp32-meter01`. It publishes node metrics `pulse_count` (UInt32, wraps at 2^32, starts at 0 every boot), `boot_count` (UInt32, persisted in NVS, +1 each boot), `uptime_s`, `rssi_dbm`, `fw_version`, `profile`. Every 60 s it sends a heartbeat NDATA containing `uptime_s` and `pulse_count`, even when the count is unchanged.

**Template:**

```yaml
apiVersion: skyranger/v1
kind: Template
metadata:
  name: pulse_water_meter
  description: "Pulse-output water meter on device profile skyranger.pulse_meter.v1"
  annotations:
    skyranger.io/profile: skyranger.pulse_meter.v1   # discovery uses this to suggest the template
spec:
  params:
    connection:       { type: string, required: true }
    group_id:         { type: string, default: skyranger }
    edge_node_id:     { type: string, required: true }
    liters_per_pulse: { type: float64, default: 1.0 }
    initial_total_l:  { type: float64, default: 0 }
    heartbeat_stale:  { type: duration, default: 150s }   # 2.5 x 60 s heartbeat
    leak_max_gap:     { type: duration, default: 15m }
    leak_duration:    { type: duration, default: 2h }
  tags:
    # ---- raw device points (history kept for audit/debug) ----
    - name: raw/pulse_count
      kind: source
      datatype: uint64
      unit: "1"
      source: { connection: ${connection}, address: "spb://${group_id}/${edge_node_id}#pulse_count" }
      stale_after: ${heartbeat_stale}
    - name: raw/boot_count
      kind: source
      datatype: uint64
      source: { connection: ${connection}, address: "spb://${group_id}/${edge_node_id}#boot_count" }
    - name: raw/uptime_s
      kind: source
      datatype: uint64
      unit: s
      source: { connection: ${connection}, address: "spb://${group_id}/${edge_node_id}#uptime_s" }
      history: { enabled: false }
    - name: raw/online
      kind: source
      datatype: bool
      source: { connection: ${connection}, address: "spb://${group_id}/${edge_node_id}#@online" }
    - name: raw/rssi_dbm
      kind: source
      datatype: int64
      display_name: "Wi-Fi RSSI (dBm)"   # no unit: UCUM code for dBm is UNVERIFIED
      source: { connection: ${connection}, address: "spb://${group_id}/${edge_node_id}#rssi_dbm" }
      history: { enabled: true, mode: on_change, deadband: { type: absolute, value: 3 }, max_interval: 1h }

    # ---- engineering tags ----
    - name: total_volume
      display_name: "Total volume"
      kind: derived
      datatype: float64
      unit: L
      writable: true                       # align with mechanical register; audited
      derived:
        fn: counter_total
        inputs: { count: raw/pulse_count, epoch: raw/boot_count }
        params:
          modulus: 4294967296
          multiplier: ${liters_per_pulse}
          max_delta: 100000                # >100 m3 between samples = implausible
          reset_baseline: zero
          initial_total: ${initial_total_l}
    - name: flow_rate
      display_name: "Flow rate"
      kind: derived
      datatype: float64
      unit: L/min
      eng_range: { low: 0, high: 200 }
      derived:
        fn: rate
        inputs: { total: total_volume }
        params: { window: 5m, per: 1m }
        eval_interval: 10s
      history: { enabled: true, mode: on_change, deadband: { type: absolute, value: 0.1 }, max_interval: 15m }
    - name: health
      display_name: "Device health"
      kind: derived
      datatype: int64
      enum_labels: { 0: offline, 1: stale, 2: ok }
      derived:
        fn: liveness
        inputs: { heartbeat: raw/uptime_s, online: raw/online }
        params: { stale_after: ${heartbeat_stale} }
        eval_interval: 5s
      alarms:
        - name: not_ok
          type: not_equal
          setpoint: 2
          on_delay: 60s
          severity: medium
          message: "Water meter ${edge_node_id} not reporting"
    - name: leak
      display_name: "Continuous flow"
      kind: derived
      datatype: bool
      derived:
        fn: continuous_activity
        inputs: { total: total_volume }
        params: { max_gap: ${leak_max_gap}, duration: ${leak_duration} }
        eval_interval: 30s
      alarms:
        - name: leak
          type: equal
          setpoint: true
          severity: high
          message: "Flow has not stopped for ${leak_duration}: possible leak"
```

**Instance** (normally generated by the discovery approval, messages.md section 7):

```yaml
apiVersion: skyranger/v1
kind: TagGroup
metadata: { name: pumphouse }
spec:
  base_path: site1/pumphouse
  instances:
    - name: meter01
      template: pulse_water_meter
      params:
        connection: local-sparkplug
        edge_node_id: esp32-meter01
        initial_total_l: 1234567           # mechanical register at commissioning
```

**Resulting tags:** `site1/pumphouse/meter01/raw/pulse_count`, `.../raw/boot_count`, `.../raw/uptime_s`, `.../raw/online`, `.../raw/rssi_dbm`, `.../total_volume`, `.../flow_rate`, `.../health`, `.../leak`.

**Trace of one scenario:**

| Time | Device sends | `pulse_count` | `total_volume` | `flow_rate` | `health` |
|---|---|---|---|---|---|
| 10:00:00 | NBIRTH bdSeq=4, boot_count=7, pulse_count=0 | 0 good | 1234567 good (first sample; initial_total) | `uncertain_initial_value` | 2 ok |
| 10:00:00-10:05:00 | NDATA, 12 L/min | 60 | 1234627 | 12.0 good (window filled) | 2 |
| 10:05:00-10:20:00 | heartbeats only, no flow | 60 (repeat) | 1234627 | decays to 0.0 good | 2 |
| 10:20:00 | power cut, broker publishes NDEATH (Will) | 60 `uncertain_last_known` | 1234627 `uncertain_last_known` | 0.0 `uncertain_last_known` | 0 offline |
| 10:21:00 | alarm `health/not_ok` becomes `active_unacked` | | | | |
| 10:22:00 | NBIRTH bdSeq=5, boot_count=8, pulse_count=3 | 3 good | 1234630 good, event `counter_discontinuity{kind: reset, delta_applied: 3}` | good | 2 |

"No flow" is `flow_rate = 0.0` with quality `good` and `health = 2`. "Dead device" is non-good quality on `flow_rate` and `health` of 0 or 1. The HMI shows the two cases differently.

## Decisions made
1. Six engine datatypes only (`bool`, `int64`, `uint64`, `float64`, `string`, `datetime`); drivers widen native types.
2. Quality is a `uint16` with class in the top bits (`code >> 6`) and 14 named codes; distinct codes for "link down" (`bad_not_connected`) vs "device quiet" (`uncertain_last_known`).
3. Three timestamps; effective `ts` chosen by a skew-checked rule, defaulting to the device clock only when it is plausible.
4. `tag_id` UUIDv7 is the immutable key; path is a mutable, lowercase, MQTT-safe name.
5. Derived values come from a fixed set of Go built-ins. `counter_total` owns rollover/reboot logic and persists its state; the device stays dumb.
6. Config is Kubernetes-style YAML (`Gateway`, `Connection`, `Template`, `TagGroup`), applied atomically as numbered revisions in PostgreSQL, with lossless export/import.
7. Templates use whole-scalar typed `${param}` substitution, no expression language, live-linked instances.
8. Leak detection uses `continuous_activity` (gap-based), not a rate threshold, because 1 L pulse resolution makes small-window rates read zero during slow leaks.

## Alternatives rejected and why
- **OPC UA StatusCode as our quality type**: 32-bit codes with subcodes and flags are more than HMI and history need, and copying them would tie us to a spec we have not verified in detail. We map to and from them in the driver instead.
- **Expression language for derived tags** (e.g. `expr-lang/expr`, CEL): powerful, but adds a second user language next to Python and makes reboot/rollover semantics user-written and untested. Revisit with scripting.
- **Gateway-side rollover detection without an epoch input**: cannot tell "rebooted and counted back up past the old value" from "counted normally". `boot_count` makes resets unambiguous.
- **Device persisting the cumulative count in flash**: flash wear, and still lossy on power cut. Count-from-zero plus epoch is simpler and fully detectable.
- **Path as primary key**: renames would orphan history.
- **Jinja / Go text/template for templates**: too much power for config; loops and conditionals make diffs unreadable.
- **Free-text units**: not machine-checkable; UCUM is a real standard with a Go-parsable grammar.

## Open questions
- `rate` resolution: at 1 L/pulse, a 5 min window resolves 0.2 L/min. Should `rate` also support a time-between-pulses mode for low flows?
- Postgres has no unsigned 64-bit type. Historian must choose `numeric(20,0)` or a bigint with documented range for `uint64` tags (the historian doc owns this).
- Should `max_delta` default be derived from a configured max flow rate times sample gap, rather than a fixed count?
- Template versioning: live-linked instances are simple, but a customer may want "pin instance to template version N". Deferred.
- OPC UA quality name mapping (table above) is UNVERIFIED.
- Whether `memory` tags need per-tag role requirements (e.g. setpoints writable only by `engineer`).
