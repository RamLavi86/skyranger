# Driver Interface Contract

Status: v1 contract. Depends on [tag-model.md](tag-model.md) (datatypes, quality, timestamps) and [messages.md](messages.md) (Sparkplug addressing, discovery).

A **driver** speaks one southbound protocol (Sparkplug B, Modbus TCP, OPC UA, simulator) and turns it into tag values. A driver knows addresses, native types and connections. It does not know tag paths, scaling, deadbands, alarms, or history; the tag engine owns those.

Terms:
- **Driver**: the code for one protocol (`sparkplug`, `modbus`, `opcua`, `sim`). Binary: `skyranger-driver-{name}`.
- **Connection**: one configured instance of a driver (`Connection` config kind), e.g. `local-sparkplug` or `plc1`. One driver process serves exactly one connection.
- **Driver host**: the component in `skyrangerd` that launches, supervises, and talks to driver processes.
- **Item**: one subscribed address, identified by a host-assigned `handle` (uint32).

## 1. Isolation boundary

- **Each connection runs as a separate OS process**, launched and supervised by the driver host. Transport: **gRPC over a Unix domain socket** at `/run/skyranger/drivers/{connection}.sock`. The host creates the socket directory; the driver listens; the host dials.
- Packaging: driver binaries ship inside the `skyrangerd` container image and run as child processes, so there is one container and many processes. A third-party driver may instead run in its own container with the socket directory bind-mounted; the protocol is the same.
- Why a process per connection: a hung PLC library, panic, or memory leak in one driver cannot take down the tag engine or other connections. Restart is cheap and per-connection.
- Process launch contract:

```
skyranger-driver-modbus --socket /run/skyranger/drivers/plc1.sock --connection plc1
env: SKYRANGER_LOG_LEVEL=info  SKYRANGER_LOG_FORMAT=json
```

  The driver logs JSON lines to stderr (`log/slog` JSON handler); the host captures and tags them with `connection`. It exits 0 on `SIGTERM` within `5s`, or it is killed.
- For tests and the simulator, a driver can be hosted **in-process** through the same Go interface (`driverhost.InProcess(d)`), with no gRPC. Behavior must be identical, and conformance tests run against both.

## 2. Go interface

Package `skyranger/pkg/driver`. Driver authors implement `Driver`. The SDK package `skyranger/pkg/driver/sdk` wraps a `Driver` in the gRPC server, so authors never touch protobuf.

```go
package driver

import (
    "context"
    "encoding/json"
    "time"

    "github.com/google/uuid"
    "skyranger/pkg/tag"
)

// Info is static metadata, available before Configure.
type Info struct {
    Name           string          // "modbus"
    Version        string          // semver of the driver
    ConfigSchema   json.RawMessage // JSON Schema (2020-12) for Connection.spec.settings
    AddressSyntax  string          // human help, e.g. "modbus://{unit}/{table}/{register}?type=..."
    Capabilities   Capabilities
}

type Capabilities struct {
    Browse      bool // can enumerate points
    Discovery   bool // devices announce themselves (pushes DiscoveryUpdates)
    Write       bool
    Push        bool // values arrive unsolicited (MQTT, OPC UA subscriptions) vs polled
    SourceTime  bool // supplies ts_source
}

type Driver interface {
    Info() Info

    // Configure validates and stores settings. Must not do network I/O.
    // Called once before Connect, and again on config change (driver may return
    // ErrRestartRequired to ask the host to restart the process instead).
    Configure(ctx context.Context, settings json.RawMessage) error

    // Connect starts the connection and returns quickly. Ongoing connection
    // management (retries, backoff) is the driver's job; it reports through
    // Sink.ConnectionState. Connect returning nil does NOT mean "connected".
    Connect(ctx context.Context, sink Sink) error

    // ValidateAddress parses an address without I/O. Used at config apply time.
    ValidateAddress(addr string) (AddressInfo, error)

    // Subscribe adds items. Values are delivered via Sink.Values.
    // Push drivers subscribe at the protocol level; polled drivers poll at
    // Item.Interval (driver may coalesce and batch). Per-item failures are
    // returned in the result slice, never as a whole-call error.
    Subscribe(ctx context.Context, items []Item) ([]ItemResult, error)
    Unsubscribe(ctx context.Context, handles []uint32) error

    // Read performs a one-shot read, bypassing subscriptions. Per-item results.
    Read(ctx context.Context, addrs []string) ([]Sample, error)

    // Write sends values. Must not return before the device acknowledged or
    // ctx expired. Drivers do NOT retry writes (see section 4).
    Write(ctx context.Context, writes []WriteRequest) ([]WriteResult, error)

    // Browse lists points under a starting address ("" = root). Paged.
    Browse(ctx context.Context, req BrowseRequest) (BrowseResponse, error)

    // Health returns a snapshot; must be cheap and non-blocking.
    Health(ctx context.Context) Health

    // Close disconnects and releases everything. Idempotent.
    Close(ctx context.Context) error
}

type AddressInfo struct {
    NativeType string       // "UInt32", "holding_register:int16"
    DataType   tag.DataType // natural engine type
    Writable   bool
}

type Item struct {
    Handle   uint32
    Address  string
    DataType tag.DataType   // requested engine type; driver converts or reports bad_type_mismatch
    Interval time.Duration  // poll/publish hint; 0 = driver default
}

type ItemResult struct {
    Handle uint32
    Err    *Error // nil = ok
}

// Sample is a raw (unscaled) value from the device.
type Sample struct {
    Handle    uint32         // 0 for Read results; Address is set instead
    Address   string
    Value     tag.Variant
    Quality   tag.Quality
    TSSource  time.Time      // zero if device gave none
}

type WriteRequest struct {
    WriteID uuid.UUID
    Address string
    Value   tag.Variant      // raw value, already unscaled by the tag engine
}

type WriteResult struct {
    WriteID uuid.UUID
    Err     *Error
}

type BrowseRequest struct {
    Start     string // address prefix or device_key
    PageSize  int    // default 500
    PageToken string
}

type BrowseResponse struct {
    Devices       []DiscoveredDevice // same shape as messages.md section 7 (minus status fields)
    NextPageToken string
}

// Sink is implemented by the host. Methods are safe for concurrent use.
type Sink interface {
    // Values delivers a batch. Blocks if the host is applying backpressure
    // (section 6); the driver must keep only the latest value per handle
    // while blocked.
    Values(ctx context.Context, batch []Sample) error
    ConnectionState(state ConnState, err *Error)
    Discovery(update DiscoveryUpdate)
}

type DiscoveryUpdate struct {
    Device  DiscoveredDevice
    Removed bool
}
```

The host stamps `ts_gateway` when a batch arrives at the driver host (gRPC receive or in-process call), not inside the driver.

## 3. Lifecycle

### Process lifecycle (owned by driver host)

```
          spawn           Info+Configure ok          Connect ok
 stopped ───────► starting ──────────────► configured ─────────► running
    ▲                │ fail                                       │
    │                ▼                                            │ crash / health timeout
    └──────── backoff (1s, 2s, 4s ... max 60s, reset after 5 min stable) ◄──┘
 running ──Close──► stopping ──exit──► stopped
```

- After a crash, the host respawns with backoff, re-runs `Configure`, `Connect`, and re-`Subscribe`s all items with the same handles. During the outage, all items on the connection are `bad_not_connected`.
- Config change on a running connection: if only items changed, use `Subscribe`/`Unsubscribe`. If `settings` changed, call `Configure`; on `ErrRestartRequired`, restart the process.
- Host calls `Health` every `10s`. Three consecutive failures or timeouts (`2s` each) mean the host kills and restarts the process.

### Connection state (reported by driver via `Sink.ConnectionState`)

| `ConnState` | Wire name | Item quality effect |
|---|---|---|
| `StateDisconnected` | `disconnected` | all items `bad_not_connected` |
| `StateConnecting` | `connecting` | unchanged (still bad until first connect) |
| `StateConnected` | `connected` | items updated as samples arrive |
| `StateDegraded` | `degraded` | connected, but some items failing; driver reports per-item quality |
| `StateFaulted` | `faulted` | configuration-level failure (auth rejected, bad settings): all items `bad_config_error`, no auto-retry until config changes |

The host mirrors this into tag `_system/connections/{connection}/state` (string) and emits `skyranger.connection.state_changed.v1` (messages.md).

`_system/connections/{connection}/...` tags created by the host: `state`, `items`, `samples_per_sec`, `dropped_samples`, `overruns`, `restarts`, `clock_skew_rejections`, `last_error`.

## 4. Error model

```go
type Code string

const (
    CodeInvalidAddress   Code = "invalid_address"    // syntax; caught at apply time
    CodeNotFound         Code = "not_found"          // address valid but point doesn't exist
    CodeTypeMismatch     Code = "type_mismatch"
    CodeNotWritable      Code = "not_writable"
    CodeNotConnected     Code = "not_connected"
    CodeTimeout          Code = "timeout"
    CodeDeviceError      Code = "device_error"       // device returned an exception
    CodeUnauthorized     Code = "unauthorized"       // device/server rejected credentials
    CodeInvalidConfig    Code = "invalid_config"
    CodeUnsupported      Code = "unsupported"        // capability not implemented
    CodeResourceExhausted Code = "resource_exhausted"
    CodeInternal         Code = "internal"
)

type Error struct {
    Code      Code
    Message   string            // human-readable, no secrets
    Retryable bool
    Details   map[string]string // e.g. {"modbus_exception": "02"}
}

func (e *Error) Error() string

var ErrRestartRequired = &Error{Code: "restart_required"}
```

Rules:
- **Per-item errors go in result slices**; a whole-call `error` means the call itself could not run (process shutting down, context cancelled).
- **Errors on the value path become quality, not errors.** A failed poll of an item is a `Sample` with quality `bad_comm_failure` (timeout, protocol error), `bad_device_failure` (device-reported fault), or `bad_config_error` (point gone), carrying the last value. Mapping is the driver's job and must be documented per driver.
- **Writes are never retried by the driver or the host.** A retried write to a PLC can repeat an action. The caller (API user, script) decides. Write `Timeout` means "unknown outcome" and is reported as such in `skyranger.tag.write_completed.v1`.
- Over gRPC, `Error` travels inside result messages. gRPC status codes are used only for transport-level failures (`UNAVAILABLE`, `DEADLINE_EXCEEDED`, `CANCELLED`).
- Error codes map to API error codes (api.md section 7).

## 5. Address syntax (per driver, normative for v1 drivers)

Addresses are URI-like strings with a driver scheme. Each driver documents its grammar and implements `ValidateAddress`.

| Driver | Address | Example |
|---|---|---|
| `sparkplug` | `spb://{group_id}/{edge_node_id}[/{device_id}]#{metric}` | `spb://skyranger/esp32-meter01#pulse_count` |
| `modbus` | `modbus://{unit_id}/{table}/{address}?type={t}[&order={o}]` with table `co`\|`di`\|`hr`\|`ir`, address as 1-based Modicon number, type `bool,int16,uint16,int32,uint32,float32,float64`, order `abcd`(default)\|`cdab`\|`badc`\|`dcba` | `modbus://1/hr/40001?type=float32&order=cdab` |
| `opcua` | `opcua:{node_id}` using the OPC UA string NodeId format (`ns=2;s=Pump1.Speed`, `ns=0;i=2258`) (UNVERIFIED: treat namespace URIs via `nsu=` in a later version for portability) | `opcua:ns=2;s=Line1.Flow` |
| `sim` | `sim://{generator}?{params}` with generators `const`, `sine`, `ramp`, `counter`, `random_walk`, `pulse_meter` and fault params `drop`, `stale`, `bad_quality`, `burst` (testing doc owns the details) | `sim://pulse_meter?lpm=12&reboot_every=1h` |

The Modbus addressing convention (1-based numbers with 4xxxx prefix vs 0-based offsets) is a frequent source of off-by-one bugs. We use the table plus 1-based register number, and the Modbus driver doc must include worked examples.

## 6. Backpressure and flow control

- Driver to host values travel over a **server-streaming gRPC call** opened by the host (`StreamEvents`). Each `DriverEvent.samples` message is a `SampleBatch` (max 1000 samples or 64 KiB, whichever is smaller). gRPC/HTTP2 flow control gives real backpressure: if the tag engine is slow, `Sink.Values` blocks.
- **Driver obligation while blocked: coalesce, latest value wins per handle.** The SDK implements this with a map of `handle -> latest Sample` plus a dirty set. Memory is bounded by the item count rather than by message rate. Every coalesced-away sample increments `dropped_samples`.
- Consequence: under overload, history is down-sampled but never stale or unbounded. For cumulative counters (water meter), coalescing loses no information.
- Exception: `Sample`s carrying a quality change are never coalesced away behind a later sample of the same quality. If the queue holds `good` and then a `bad` arrives, both are delivered. The SDK keeps at most one pending quality transition plus the latest value per handle.
- Host side: the tag engine processes batches on a bounded channel (default 64 batches per connection). The historian writer has its own bounded queue with batch inserts (COPY) and spills to a local on-disk WAL if PostgreSQL is unavailable (historian doc owns this). The tag engine never blocks on the historian.
- Polled drivers schedule by `Interval`; if a poll cycle overruns, the next cycle is skipped (not queued) and `_system/connections/{connection}/overruns` increments.
- Unary RPC deadlines (host-set): `Configure` 5s, `Subscribe` 10s, `Read` 10s, `Write` 10s (overridable per request up to 60s), `Browse` 30s, `Health` 2s.

## 7. gRPC service (`skyranger.driver.v1`)

```protobuf
syntax = "proto3";
package skyranger.driver.v1;

import "google/protobuf/timestamp.proto";
import "google/protobuf/duration.proto";

service Driver {
  rpc GetInfo(GetInfoRequest) returns (Info);
  rpc Configure(ConfigureRequest) returns (ConfigureResponse);
  rpc Connect(ConnectRequest) returns (ConnectResponse);
  rpc ValidateAddress(ValidateAddressRequest) returns (ValidateAddressResponse);
  rpc Subscribe(SubscribeRequest) returns (SubscribeResponse);
  rpc Unsubscribe(UnsubscribeRequest) returns (UnsubscribeResponse);
  rpc Read(ReadRequest) returns (ReadResponse);
  rpc Write(WriteRequest) returns (WriteResponse);
  rpc Browse(BrowseRequest) returns (BrowseResponse);
  rpc Health(HealthRequest) returns (HealthResponse);
  rpc Close(CloseRequest) returns (CloseResponse);
  // Host opens once after Connect; carries values, state, discovery.
  rpc StreamEvents(StreamEventsRequest) returns (stream DriverEvent);
}

enum DataType {
  DATA_TYPE_UNSPECIFIED = 0;
  DATA_TYPE_BOOL = 1;
  DATA_TYPE_INT64 = 2;
  DATA_TYPE_UINT64 = 3;
  DATA_TYPE_FLOAT64 = 4;
  DATA_TYPE_STRING = 5;
  DATA_TYPE_DATETIME = 6;
}

message Variant {
  oneof kind {
    bool bool_value = 1;
    int64 int64_value = 2;
    uint64 uint64_value = 3;
    double float64_value = 4;
    string string_value = 5;
    google.protobuf.Timestamp datetime_value = 6;
  }  // none set = null
}

message Sample {
  uint32 handle = 1;
  string address = 2;
  Variant value = 3;
  uint32 quality = 4;                       // tag-model.md section 4 code
  google.protobuf.Timestamp ts_source = 5;  // unset if unknown
}

message SampleBatch { repeated Sample samples = 1; }

enum ConnState {
  CONN_STATE_UNSPECIFIED = 0;
  CONN_STATE_DISCONNECTED = 1;
  CONN_STATE_CONNECTING = 2;
  CONN_STATE_CONNECTED = 3;
  CONN_STATE_DEGRADED = 4;
  CONN_STATE_FAULTED = 5;
}

message Error {
  string code = 1;            // section 4 Code values
  string message = 2;
  bool retryable = 3;
  map<string, string> details = 4;
}

message ConnectionStateChange { ConnState state = 1; Error error = 2; }

message DiscoveryUpdate {
  string device_key = 1;
  string profile = 2;
  bool online = 3;
  bool removed = 4;
  map<string, string> attributes = 5;
  repeated Point points = 6;
}

message Point {
  string address = 1;
  string name = 2;
  string native_type = 3;
  DataType datatype = 4;
  bool writable = 5;
}

message DriverEvent {
  oneof event {
    SampleBatch samples = 1;
    ConnectionStateChange state = 2;
    DiscoveryUpdate discovery = 3;
  }
}

message Item {
  uint32 handle = 1;
  string address = 2;
  DataType datatype = 3;
  google.protobuf.Duration interval = 4;
}
message ItemResult { uint32 handle = 1; Error error = 2; }
message SubscribeRequest { repeated Item items = 1; }
message SubscribeResponse { repeated ItemResult results = 1; }

message WriteItem { string write_id = 1; string address = 2; Variant value = 3; }
message WriteResultItem { string write_id = 1; Error error = 2; }
message WriteRequest { repeated WriteItem writes = 1; }
message WriteResponse { repeated WriteResultItem results = 1; }

message HealthResponse {
  ConnState state = 1;
  uint32 items = 2;
  uint64 samples_total = 3;
  uint64 dropped_samples = 4;
  uint64 overruns = 5;
  Error last_error = 6;
  google.protobuf.Timestamp last_error_at = 7;
}
// Remaining request/response messages mirror the Go types in section 2;
// the .proto file in proto/skyranger/driver/v1/driver.proto is authoritative
// once written and must not contradict this section.
```

Proto rules: never renumber or reuse field numbers; additive changes only within `v1`; managed with `buf` (`buf lint`, `buf breaking`).

## 8. Conformance

Every driver must pass the shared conformance suite (`skyranger/pkg/driver/conformance`) run against the simulator or a protocol test server: lifecycle restart re-subscribes the same handles, coalescing under a blocked sink, quality transitions never lost, per-item errors don't fail the batch, `Close` idempotent, `Configure` does no I/O, `SIGTERM` exit within 5 s.

## Decisions made
1. One OS process per connection, gRPC over a Unix domain socket; an in-process mode with the same Go interface for tests.
2. Drivers deal in raw values plus addresses. Scaling, paths, deadbands, alarms, and history belong to the tag engine.
3. Value-path failures become quality codes; per-item errors go in result slices; whole-call errors are only for transport-level failures.
4. Backpressure via a gRPC stream plus driver-side latest-value-wins coalescing that never drops quality transitions. Memory is bounded by item count.
5. Writes are never automatically retried.
6. `Connect` returns immediately; drivers own reconnect and report one of five connection states.
7. Driver addresses are URI-like strings with a per-driver scheme.

## Alternatives rejected and why
- **In-process Go plugins (`plugin` package)**: fragile (exact toolchain/dependency match), Linux-only, no crash isolation.
- **HashiCorp go-plugin**: proven, but adds handshake/netrpc layers we don't need; plain gRPC over UDS is simpler to debug and language-neutral (a Python or C driver is possible). Worth revisiting if we need its features.
- **Drivers publish to MQTT internal bus**: no request/response for writes/browse, no flow control, and it would put the broker in the history path.
- **Container per driver by default**: heavier on small edge boxes and complicates supervision; process per connection gives most of the isolation.
- **Unbounded queues or drop-oldest buffers**: either OOM or stale data; latest-value-wins is the standard SCADA answer.
- **Driver-side scaling**: duplicates logic across drivers and gives inconsistent rounding.

## Open questions
- Should one process be allowed to serve multiple connections of the same driver (memory on small boxes vs isolation)? v1 says no.
- Store-and-forward samples (`is_historical` in Sparkplug) need a separate, non-coalescing path. Deferred.
- Sandboxing driver processes (seccomp, separate UID) belongs to the security doc.
- OPC UA address `nsu=` form and Go OPC UA library choice (`gopcua/opcua` is the obvious candidate; its maturity for subscriptions should be evaluated) are open and the address syntax is UNVERIFIED.
- Whether `Write` needs a "write and read back" verification option for critical setpoints.
