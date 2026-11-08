# `sparkplug` Driver: Sparkplug B Host Application

Milestone 1 driver. It consumes ESP32-class edge nodes (profile `skyranger.pulse_meter.v1`) and any other Sparkplug B edge node. Normative inputs: [messages.md](../00-contracts/messages.md) sections 1, 2 and 7, and the `spb://` address syntax in [driver-interface.md](../00-contracts/driver-interface.md) section 5. Spec facts marked UNVERIFIED in messages.md stay UNVERIFIED here and must be checked against the Sparkplug 3.0 specification and the Eclipse Tahu `sparkplug_b.proto` before code is finalized.

## 1. Capabilities

```go
driver.Info{
    Name:          "sparkplug",
    AddressSyntax: "spb://{group_id}/{edge_node_id}[/{device_id}]#{metric_name}  (metric '@online' is reserved)",
    Capabilities:  driver.Capabilities{Browse: true, Discovery: true, Write: true, Push: true, SourceTime: true},
}
```

## 2. Settings

```yaml
apiVersion: skyranger/v1
kind: Connection
metadata:
  name: local-sparkplug
spec:
  driver: sparkplug
  enabled: true
  timestamp_source: auto
  discovery: { enabled: true }
  settings:
    broker_url: mqtt://mosquitto:1883          # mqtts:// for TLS
    client_id: skyranger-gw01-spb
    credentials: secret://mqtt/gateway         # resolved by host, never logged
    group_ids: [skyranger]                     # subscribes spBv1.0/{g}/# for each
    host_id: skyranger-gw01                    # see Open questions: injected from Gateway.spec.sparkplug.host_id
    keep_alive: 30s
    tls:                                       # optional
      ca: secret://mqtt/ca
      insecure_skip_verify: false              # schema forbids true unless ...allow_insecure: true
    rebirth:
      min_interval: 30s                        # per node, contract value
      on_connect: true                         # request rebirth from all known nodes after reconnect
    write:
      confirm: report_back                     # report_back | publish_only (section 7)
      confirm_timeout: 10s
    flush_interval: 50ms
```

## 3. Internal model

```go
type nodeKey struct{ group, node string }            // device-level state hangs off the node
type node struct {
    online      bool
    bdSeq       uint64          // from last NBIRTH
    nextSeq     uint8           // expected seq of the next message (wraps 255 -> 0)
    born        bool
    aliases     map[uint64]string           // alias -> metric name, from NBIRTH
    metrics     map[string]metricInfo       // name -> datatype, last value
    devices     map[string]*device          // device_id -> device (DBIRTH)
    lastRebirth time.Time
}
type device struct {
    online  bool
    born    bool
    aliases map[uint64]string
    metrics map[string]metricInfo
}

// Subscription index: address -> handles. More than one tag may bind to the same metric.
type subs map[string][]uint32  // "spb://skyranger/esp32-meter01#pulse_count" -> [17, 42]
```

All MQTT callbacks put messages on one channel, and one goroutine processes them in order. Node state has no locks and the processing order matches the broker's delivery order.

## 4. Message handling

Topic `spBv1.0/{group_id}/{type}/{edge_node_id}[/{device_id}]`. Payloads are decoded with code generated from the Tahu `.proto`, used verbatim (messages.md).

```text
NBIRTH(node):
    reset node: aliases, metrics, devices = {}, born = true, online = true
    bdSeq = metric "bdSeq"; nextSeq = payload.seq + 1
    for m in metrics: record name, alias, datatype
    emit samples for every subscribed metric of this node (quality good, ts_source = metric/payload timestamp)
    emit @online = true
    subscribed metrics missing from the birth -> sample bad_config_error   (metric no longer exists)
    emit DiscoveryUpdate{device_key: spb://g/n, online: true, profile, attributes, points}

NDATA(node):
    if !born                       -> uncertain_last_known for node's subscribed items, request rebirth, drop
    if payload.seq != nextSeq      -> uncertain_last_known for node, request rebirth, still apply this payload
                                      (see note), nextSeq = payload.seq + 1
    for m in metrics:
        name = m.name or aliases[m.alias]; unknown alias -> request rebirth, skip metric
        if m.is_historical         -> skip, count historical_dropped (contract: store-and-forward deferred)
        if m.is_null               -> sample with null value and quality bad_device_failure (see Open questions)
        else                       -> sample good
    nextSeq++

NDEATH(node):
    if bdSeq metric != node.bdSeq  -> ignore (stale Will from an older session)
    online = false; all node and device items -> uncertain_last_known; @online = false (good quality)
    DiscoveryUpdate{online: false}

DBIRTH / DDATA / DDEATH: same rules scoped to the device, sharing the node's seq counter.
    DDEATH: device items uncertain_last_known, device @online = false.
NCMD / DCMD from others: ignored (we only publish them).
STATE from other hosts: ignored in v1 (we do not support multi-host failover).
```

Notes:
- **Applying a payload after a seq gap**: the payload is internally consistent, and cumulative counters lose nothing. The tags are first marked `uncertain_last_known` and then go back to `good` when their values in this payload are applied. Repeating `uncertain_last_known` for the whole node therefore only affects metrics not present in this payload, which really are of unknown freshness until rebirth. UNVERIFIED: the spec may require the host to discard data after a seq gap. If so, discard and wait for rebirth.
- **The `@online` pseudo-metric** is a `bool` with quality `good`. It reports health and is not a measurement, the same way as `liveness` (tag-model section 7). That way the `liveness` derived tag gets a good `online=false` and outputs `0 offline` instead of copying a bad input.
- **Rebirth request**: publish `NCMD` to `spBv1.0/{g}/NCMD/{n}` with metric `Node Control/Rebirth` Boolean `true`. Rate-limited per node to `rebirth.min_interval`. Sent with QoS 0 (UNVERIFIED: the spec's QoS rule for NCMD).
- **Timestamps**: metric `timestamp` (ms since epoch) to `ts_source`; if absent, payload `timestamp`; if both absent, zero. Plausibility checking is the tag engine's job (tag-model section 5).

### Datatype mapping (Sparkplug native to natural)

| Sparkplug | Natural | Notes |
|---|---|---|
| Int8, Int16, Int32, Int64 | `int64` | Sparkplug carries these in `int_value`/`long_value` unsigned fields, so they must be sign-extended according to declared width (UNVERIFIED: verify the encoding in the spec, this is a known interop pitfall) |
| UInt8, UInt16, UInt32, UInt64 | `uint64` | `pulse_count`, `boot_count` |
| Float, Double | `float64` | NaN/Inf: `bad_type_mismatch` |
| Boolean | `bool` | |
| String, Text, UUID | `string` | |
| DateTime | `datetime` | ms since epoch, UTC |
| DataSet, Bytes, File, Template, arrays | none | discovered, not subscribable (`unsupported`), contract |

## 5. Connection and session

```text
connect:
    MQTT CONNECT (3.1.1, clean session = true, keep_alive)
      Will: spBv1.0/STATE/{host_id}  {"online": false, "timestamp": ...}  retained QoS 1   (UNVERIFIED 3.0 format)
    on CONNACK:
      SUBSCRIBE spBv1.0/{g}/# for g in group_ids  (QoS 0)
      on SUBACK: PUBLISH spBv1.0/STATE/{host_id} {"online": true, ...} retained QoS 1
      state = connected
      if rebirth.on_connect: request rebirth from every node in the known table (rate-limited, spread over 5 s)
on connection lost:
    state = disconnected  (host sets all items bad_not_connected)
    keep the node table (aliases are invalid until rebirth anyway: mark all nodes born = false)
    reconnect with sdk/backoff (reliability.md)
```

Clean session is used: with a persistent session, the broker would replay queued QoS 1 messages after reconnect, and those would arrive with seq values we would treat as gaps. Rebirth gives a clean picture.

On reconnect, nodes are not marked online or offline until their birth or death arrives. Their items stay at whatever the host set (`bad_not_connected`) until a birth refreshes them. NDEATH is not a retained message (UNVERIFIED), so a node that died while we were disconnected never answers the rebirth request, and its items would stay `bad_not_connected` after we reconnect. That is wrong, because the link is up. Rule: **30 s after reconnect** (`rebirth.unanswered_after`, default 30 s), any node that has not sent a birth has its items set to `uncertain_last_known` and `@online = false`. This matches "we can hear the network, the device is quiet".

## 6. Discovery and Browse

- Every NBIRTH/DBIRTH sends a `DiscoveryUpdate` with `device_key = spb://{g}/{n}[/{d}]`, `profile` from the `profile` metric if present, `attributes` from `fw_version` and any metric listed in `discovery.attribute_metrics` (default `[fw_version]`), and `points` for every scalar metric (plus `@online`). Non-scalar metrics are included with `datatype` unspecified, so the HMI can show them as not mappable.
- The driver does not filter by approval. Unapproved devices produce discovery updates only. The driver sends samples only for subscribed handles, so data from unapproved devices never reaches the tag engine (messages.md section 7).
- NDEATH sends `DiscoveryUpdate{online: false}`. `Removed` is never sent by this driver. Removal is an operator action.
- `Browse(start)` returns the current in-memory node table filtered by prefix. It does no I/O. To make a silent node appear, the operator can call Browse with `start = spb://g/n`, which also requests a rebirth from that node (rate-limited).

## 7. Writes (NCMD/DCMD)

Sparkplug has no command acknowledgement. The contract says `Write` "must not return before the device acknowledged or ctx expired". The driver implements this with a **report-back confirm**:

```text
Write(addr=spb://g/n#setpoint, value v):
    node not online                    -> not_connected
    metric not in birth                -> not_found
    metric datatype cannot hold v      -> type_mismatch
    publish NCMD {metric: name, datatype: from birth, value: v}  (QoS 0, UNVERIFIED)
    write.confirm = report_back:
        wait until NDATA/DDATA carries that metric with value == v   -> ok
        or NDEATH                                                     -> not_connected
        or confirm_timeout / ctx                                      -> timeout (outcome unknown)
    write.confirm = publish_only:
        ok once the publish has been written to the socket (documented as "sent, not confirmed")
```

The ESP32 pulse-meter profile has no writable metrics in v1. `Node Control/Rebirth` is not writable through tags. It is driven only by the driver.

## 8. Quality mapping summary

| Event | Quality |
|---|---|
| Broker unreachable / connection lost | `bad_not_connected` (host, by state) |
| Broker rejects credentials (CONNACK not authorized) | `faulted`, `bad_config_error` |
| Node or device not yet born after subscribe | `bad_waiting_for_initial_data` (left as set by the engine) |
| Metric absent from latest birth | `bad_config_error` |
| NDEATH / DDEATH / no birth 30 s after reconnect | `uncertain_last_known`; `@online = false` with `good` |
| Seq gap, unknown alias, NDATA before NBIRTH | `uncertain_last_known` until rebirth |
| `is_null` metric | `bad_device_failure` with last value |
| Value not representable | `bad_type_mismatch` |
| Normal NDATA / birth | `good` |

## 9. Test cases specific to this driver

1. Reconnect without reboot (new `bdSeq`, same `boot_count`, count continues) gives no `counter_discontinuity`. This is end-to-end with `counter_total`.
2. Reboot (new `boot_count`, count from 0) gives exactly one `reset` event.
3. Stale Will: an NDEATH with an old `bdSeq` arriving after a new NBIRTH is ignored.
4. Seq wrap 255 to 0 is not a gap.
5. Alias-only NDATA after birth decodes. Unknown alias triggers exactly one rebirth per 30 s.
6. Broker restart: all items `bad_not_connected`, then `good` after births. A node that died during the outage goes to `uncertain_last_known` after 30 s.
7. 1 000 simulated nodes at 1 msg/s each for 1 hour: no growth in RSS, flush batches never over 1000 samples.

## Decisions made
1. One ordered processing goroutine for all MQTT messages. Node state needs no locks.
2. Clean MQTT session plus rebirth on reconnect, not persistent sessions.
3. Seq gaps make the node `uncertain_last_known` and trigger a rate-limited rebirth. The payload itself is applied (subject to the UNVERIFIED spec check).
4. `@online` is a `good`-quality bool, so `liveness` can tell offline apart from a broken link.
5. Nodes that do not rebirth within 30 s after a reconnect are treated as dead (`uncertain_last_known`).
6. `is_historical` metrics are dropped and counted until the store-and-forward path exists.
7. Writes are confirmed by the value being reported back, with `publish_only` as an explicit and documented weaker option.

## Alternatives rejected and why
- **Persistent MQTT session (clean session false)**: replays old messages after reconnect and confuses seq tracking.
- **MQTT 5 client**: Sparkplug 3.0 targets MQTT 3.1.1 compatibility (UNVERIFIED for 5.0 specifics), Paho 3.1.1 Go client is the most used. MQTT 5 brings nothing we need yet.
- **Writing our own MQTT client**: MQTT is easy to get mostly right and hard to get fully right (keep-alive, inflight, reconnect). Use Paho.
- **Hand-written Sparkplug protobuf**: contract forbids. The Tahu `.proto` is used verbatim.
- **Treating NDATA before NBIRTH as good**: aliases and datatypes are unknown, so values could be decoded incorrectly.

## Open questions
- `host_id` lives in `Gateway.spec.sparkplug.host_id`, but drivers only get `Connection.spec.settings`. Proposal: the config service injects `host_id` into the sparkplug driver's settings at apply time if absent. This needs a note in the contract.
- Quality for `is_null`: `bad_device_failure` is a choice. An alternative is `uncertain`. Needs agreement with the tag engine author.
- Sparkplug seq-gap handling, NCMD QoS, NDEATH retain flag, STATE 3.0 payload, signed integer encoding: all UNVERIFIED and must be checked against the spec.
- Multi-host (primary host) support: v1 ignores other hosts' STATE. Is that acceptable if a customer also runs Ignition on the same broker? It should be, as long as host ids differ.
- Should the driver warn when a device's `heartbeat_interval_s` exceeds the template's `heartbeat_stale`? That would need the driver to know tag config, which it should not. It is better done in the discovery service.
