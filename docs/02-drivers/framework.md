# Driver Framework and Plugin Model

Builds on [driver-interface.md](../00-contracts/driver-interface.md). The contract fixes the Go `driver.Driver` interface, the gRPC service, the lifecycle, the error model and backpressure. This doc covers what driver authors and the driver host build around that contract: the SDK, how drivers are found and launched, supervision, resource limits, and type conversion.

## 1. Repository layout

```
pkg/driver/                 # contract types (Driver, Sink, Item, Sample, Error ...). Owned by contract.
pkg/driver/sdk/             # wraps a Driver in the gRPC server; author-facing helpers
pkg/driver/sdk/coalesce/    # latest-value-wins queue (contract section 6)
pkg/driver/sdk/scan/        # poll scheduler + scan classes (scanning.md)
pkg/driver/sdk/backoff/     # reconnect backoff with jitter (reliability.md)
pkg/driver/sdk/connstate/   # connection state machine helper (reliability.md)
pkg/driver/sdk/convert/     # native -> engine type conversion, exactness checks (section 6)
pkg/driver/sdk/addr/        # shared URI-ish address parsing helpers
pkg/driver/conformance/     # shared conformance suite (contract section 8)
internal/driverhost/        # in skyrangerd: spawn, supervise, gRPC client, InProcess()
drivers/sim/                # each driver: package + cmd
drivers/sparkplug/
drivers/modbus/
drivers/opcua/
drivers/enip/               # cgo (libplctag); built in a separate build stage
drivers/s7/
cmd/skyranger-driver-sim/main.go
cmd/skyranger-driver-sparkplug/main.go
...
proto/skyranger/driver/v1/driver.proto
```

Every driver `main.go` is the same three lines:

```go
package main

import (
    "skyranger/drivers/modbus"
    "skyranger/pkg/driver/sdk"
)

func main() { sdk.Main(modbus.New) } // parses --socket, --connection, sets up slog, serves gRPC, handles SIGTERM
```

## 2. SDK responsibilities

`sdk.Main(factory func() driver.Driver)` does all of this so driver authors never repeat it:

| Concern | What the SDK does |
|---|---|
| Flags | `--socket`, `--connection` (contract launch line). Also `--info` (prints `Info` as JSON and exits; see Open questions) |
| Logging | `log/slog` JSON handler to stderr, level from `SKYRANGER_LOG_LEVEL`, every record carries `connection` and `driver` |
| gRPC | Listens on the UDS (mode `0600`), registers `skyranger.driver.v1.Driver`, converts Go types to protobuf and back |
| Event stream | Implements `Sink` on top of the `StreamEvents` server stream, using the `coalesce` queue. `Values` blocks only when the stream is flow-controlled |
| Panics | Each RPC handler has `recover()`. It logs the stack and returns `internal`. A panic in a driver goroutine that the SDK did not start still kills the process; the host restarts it, which is the intended behavior |
| Shutdown | On `SIGTERM`: stop accepting RPCs, call `Driver.Close` with a 4 s context, flush the event stream, exit 0. A hard `os.Exit(1)` at 4.8 s keeps us inside the contract's 5 s |
| Health | Wraps `Driver.Health` with a 1.5 s internal deadline so a blocked driver still answers inside the host's 2 s timeout with `last_error = internal "health handler blocked"` |
| Stats | Counters for `samples_total`, `dropped_samples` (coalescing), `overruns` (scheduler) feed `HealthResponse` |

Driver authors also get these optional helpers (they do not have to use them, but every first-party driver does):

```go
// connstate: one place that emits ConnectionState and avoids duplicate transitions.
type Reporter struct{ /* sink, current state, last error, mutex */ }
func (r *Reporter) Set(s driver.ConnState, err *driver.Error) // no-op if unchanged (state and error code)

// backoff: see reliability.md for parameters.
type Backoff struct{ Initial, Max time.Duration; Multiplier, Jitter float64 }
func (b *Backoff) Next() time.Duration
func (b *Backoff) Reset()

// scan: see scanning.md.
type Scheduler struct{ /* ... */ }
func NewScheduler(read ReadFunc, sink driver.Sink, opts SchedulerOptions) *Scheduler
```

## 3. Plugin model

### 3.1 Bundled drivers (default)

- Driver binaries are installed in the `skyrangerd` image at `/usr/lib/skyranger/drivers/skyranger-driver-{name}`.
- At start, the host lists that directory, runs each binary once with `--info` (5 s timeout) and caches the `Info`. That cache feeds `GET /api/v1/drivers` and validates `Connection.spec.driver` and `settings` (against `Info.ConfigSchema`) at config apply time without a running connection.
- For each enabled `Connection`, the host spawns `skyranger-driver-{spec.driver} --socket /run/skyranger/drivers/{name}.sock --connection {name}`.

### 3.2 External drivers (third party, own container)

The contract allows a driver in its own container with the socket directory bind-mounted. The host does not spawn it. It runs in **attach mode**:

```yaml
apiVersion: skyranger/v1
kind: Connection
metadata:
  name: bacnet1
spec:
  driver: bacnet               # must match Info.Name reported by the driver
  enabled: true
  settings: { ... }            # validated by the schema the driver returns from GetInfo
  # Proposed additive field, see Open questions:
  # host: { mode: attach }     # default: spawn
```

In attach mode the host dials `/run/skyranger/drivers/bacnet1.sock`, calls `GetInfo`, and checks that `Info.Name == spec.driver`. Supervision is "reconnect to the socket with backoff". The host cannot restart the process; the container runtime's restart policy does that. All items show `bad_not_connected` while the socket is unreachable.

### 3.3 Why not Go `plugin`, go-plugin or WASM

The contract already rejects Go `plugin` and go-plugin. WASM (for example wazero) was also considered for third-party drivers. It is rejected because drivers need raw sockets, serial ports and C libraries (libplctag), and WASI networking is not mature enough to depend on (UNVERIFIED for current state). It could be revisited for sandboxed user code.

## 4. Driver host supervision

The host keeps one supervisor goroutine per connection. Process states and backoff are fixed by the contract (section 3). This section adds the implementation detail.

```go
package driverhost

type Supervisor struct {
    conn     config.Connection
    bin      string
    items    map[uint32]driver.Item   // desired subscriptions, source of truth for re-subscribe
    client   pb.DriverClient
    cmd      *exec.Cmd
    restarts int
    backoff  backoff.Backoff          // 1s, 2s, 4s ... 60s; Reset after 5 min in running
}

func (s *Supervisor) run(ctx context.Context) {
    for ctx.Err() == nil {
        s.markAll(tag.QualityBadNotConnected) // until the driver says otherwise
        if err := s.startOnce(ctx); err != nil {
            s.recordErr(err)
            s.sleep(ctx, s.backoff.Next())
            continue
        }
        stable := time.AfterFunc(5*time.Minute, s.backoff.Reset)
        err := s.serve(ctx)   // pumps StreamEvents, runs the Health ticker; returns on exit/health failure
        stable.Stop()
        s.restarts++
        s.recordErr(err)
        s.kill()              // SIGTERM, 5s, SIGKILL; reap
        s.sleep(ctx, s.backoff.Next())
    }
}

func (s *Supervisor) startOnce(ctx context.Context) error {
    s.cmd = exec.CommandContext(ctx, s.bin, "--socket", s.sock(), "--connection", s.conn.Name)
    s.cmd.SysProcAttr = &syscall.SysProcAttr{
        Setpgid:   true,             // own process group: we can kill the driver and its children
        Pdeathsig: syscall.SIGKILL,  // Linux: driver dies if skyrangerd dies (no orphans holding PLC sessions)
    }
    s.cmd.Env = s.env()              // SKYRANGER_LOG_*, GOMEMLIMIT, GOMAXPROCS
    s.cmd.Stderr = s.logPipe()       // JSON lines, re-emitted with connection=<name>
    // 1. start  2. wait for socket (poll 50ms, max 5s)  3. dial
    // 4. GetInfo (check Name)  5. Configure(settings)  6. Connect
    // 7. open StreamEvents     8. Subscribe(all s.items, same handles), in chunks of 1000
    return nil
}
```

Notes:
- `Pdeathsig` is tied to the thread that started the child in Go. The host must spawn from a goroutine locked with `runtime.LockOSThread` that stays alive, or accept the edge case. UNVERIFIED: re-check the current Go docs on `Pdeathsig` and thread exit before relying on it. A belt-and-braces measure: the SDK also exits when its parent PID changes (checked every 1 s with `os.Getppid()`).
- The stream is opened **before** `Subscribe`, so initial values sent during subscribe are not lost.
- Stale socket files are removed before spawning.
- `POST /api/v1/connections/{name}/restart` triggers `kill()` with no backoff delay and does not count towards the backoff.
- Disabling a connection (`enabled: false`) stops the process. All its tags become `bad_out_of_service` (tag-model section 4), not `bad_not_connected`.

### 4.1 Config changes

| Change | Host action |
|---|---|
| Items added/removed only | `Subscribe` / `Unsubscribe` on the running process |
| Item `Interval` changed | `Unsubscribe` + `Subscribe` with the same handle. A value gap is acceptable |
| `settings` changed | `Configure`. On `ErrRestartRequired`, restart. First-party drivers return `ErrRestartRequired` for any endpoint/credential change. That is simpler and safer than hot-swapping a session |
| `driver` changed | Stop old process, start new one |
| Connection removed | Stop process, remove socket, remove `_system/connections/{name}/...` tags |

## 5. Isolation and resource limits

Process-per-connection gives crash isolation. These limits stop one driver from starving the box:

| Limit | Mechanism | Default | Effect when hit |
|---|---|---|---|
| Memory (soft) | `GOMEMLIMIT` env | 128 MiB | Go GC works harder |
| Memory (hard) | Host reads `VmRSS` from `/proc/{pid}/status` every 10 s alongside `Health` | 256 MiB | Host kills and restarts the driver, `last_error = resource_exhausted` |
| CPU | `GOMAXPROCS=2` env; `nice` +5 for drivers relative to `skyrangerd` | 2 | Driver can't monopolize cores |
| File descriptors | `RLIMIT_NOFILE` via `prlimit` after start | 1024 | Socket leak becomes an error, not a box-wide failure |
| Item count | Driver rejects beyond `max_items` with `resource_exhausted` per item | 20 000 | Per-item error |

Limits are overridable per connection under `spec.settings.limits` (a driver setting the SDK reads before the driver sees settings). cgroup v2 per-driver limits inside the container would be stronger but need cgroup delegation into the container; deferred to the deployment doc. `RLIMIT_AS` is deliberately not used: Go reserves large virtual ranges and would fail spuriously.

Sandboxing (separate UID, seccomp, no network except the target) is listed in the contract as a security-doc item. The drivers area asks for one thing: drivers must work with read-only root filesystem and no capabilities. No first-party driver needs `CAP_NET_RAW`. (PROFINET DCP discovery would, which is one reason it is out of scope.)

## 6. Type conversion (shared `convert` package)

Contract rule: widening only, inexact values become `bad_type_mismatch`. The conversion is in two steps:

1. **Native to natural type** (`AddressInfo.DataType`): fixed per native type.
2. **Natural to requested** (`Item.DataType`): checked at `Subscribe` for type compatibility (incompatible types give `ItemResult` `type_mismatch`), and per value for range (an out-of-range value gives a `Sample` with `bad_type_mismatch` and the last good value).

| Native (any protocol) | Natural | May be requested as | Per-value check |
|---|---|---|---|
| bool / coil / BOOL / Boolean | `bool` | `bool`, `int64`, `uint64` (0/1) | none |
| int8/16/32/64, SINT/INT/DINT/LINT | `int64` | `int64`; `uint64` if value >= 0; `float64` if abs <= 2^53 | yes |
| uint8/16/32/64, USINT/UINT/UDINT/ULINT, WORD/DWORD | `uint64` | `uint64`; `int64` if <= 2^63-1; `float64` if <= 2^53 | yes |
| float32 / REAL | `float64` | `float64` only | NaN/Inf: `bad_type_mismatch` (tag-model section 3) |
| float64 / LREAL / Double | `float64` | `float64` only | same |
| string / STRING / Text | `string` | `string` | > 4096 bytes: `bad_type_mismatch` |
| DateTime / DATE_AND_TIME / DTL | `datetime` | `datetime` | must be a valid UTC instant |
| arrays, structs, DataSet, Bytes, UDT | none | not subscribable in v1 | `ItemResult` `unsupported` |

Float to integer is never allowed, because it can lose information. If a PLC stores a counter in a REAL, the tag must be `float64`.

```go
package convert

// To converts a natural-typed Variant to the requested type.
// ok=false means the value cannot be represented exactly: caller emits bad_type_mismatch.
func To(v tag.Variant, want tag.DataType) (out tag.Variant, ok bool)

// Compatible is checked once at Subscribe.
func Compatible(natural, want tag.DataType) bool
```

## 7. Configuration conventions for `settings`

Every first-party driver's `settings` schema shares these optional keys so the HMI can render them consistently:

```yaml
settings:
  # protocol-specific keys first (host, port, broker_url ...)
  timeouts:
    connect: 5s
    request: 2s
  reconnect:                 # reliability.md section 2
    initial: 1s
    max: 60s
  limits:                    # section 5
    memory: 256MiB
    max_items: 20000
  scan:                      # scanning.md section 3
    default_interval: 1s
    max_concurrent_requests: 1
```

Secrets are always references (`secret://...`), resolved by the host and passed in `Configure` in resolved form over the private UDS. They are never logged. The SDK redacts any settings key named `password`, `credentials`, `token` or `private_key` from logs.

## Decisions made
1. One SDK (`pkg/driver/sdk`) provides gRPC serving, coalescing, scheduling, backoff, state reporting and conversion. Each first-party driver's `main` is `sdk.Main(New)`.
2. Bundled drivers are discovered by listing `/usr/lib/skyranger/drivers/` and running `--info`. Third-party containerized drivers use an attach mode where the host dials but does not spawn.
3. Drivers get their own process group and `Pdeathsig=SIGKILL`, and the SDK exits if its parent changes. A dead host never leaves orphan PLC sessions.
4. Memory is limited by `GOMEMLIMIT` plus a host-enforced RSS kill. CPU is limited by `GOMAXPROCS` and nice. `RLIMIT_AS` is not used.
5. A shared two-step type conversion with a fixed table. Float to integer is never allowed.
6. Any change to endpoints or credentials restarts the driver process rather than hot-swapping the session.

## Alternatives rejected and why
- **Discover drivers via a manifest file per binary**: a second source of truth that can drift from the binary. `--info` asks the binary itself.
- **cgroup v2 per driver now**: needs delegation and privileged container setup on customer boxes. Too much deployment risk for v1.
- **Hot-reconfigure sessions in place**: adds a class of half-reconfigured bugs. Restarting costs a second of `bad_not_connected`.
- **WASM drivers**: no reliable raw socket, serial or C library access.
- **Let drivers choose any logging library**: the host relies on JSON lines on stderr. The SDK fixes this.

## Open questions
- `--info` and attach mode (`spec.host.mode`) are **additions** to the contract's launch line and `Connection` schema. They are additive, but they need a contract update via decisions.md. Without `--info` the host would have to start a throwaway process and call `GetInfo` to validate configs. That works but is slower.
- `spec.settings.limits` is read by the SDK, not the driver. Should limits instead be a first-class `Connection.spec.limits` field in tag-model.md section 9?
- Go `Pdeathsig` thread semantics: UNVERIFIED, see section 4.
- Should the host cap total driver memory on small boxes (for example 1 GiB on a 2 GiB Pi) and refuse to start more connections?
