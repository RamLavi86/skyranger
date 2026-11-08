# Polling, Subscription, Scan Classes and Batching

How values get from devices into `Sink.Values`. Contract inputs: `Item.Interval` (poll/publish hint, `0` = driver default), `Capabilities.Push`, overrun rule ("skip, don't queue, increment `overruns`"), `SampleBatch` limits (1000 samples or 64 KiB), tag-model `source.interval` (default `1s`).

## 1. Polling vs subscription per protocol

| Driver | Model | Why | What `Item.Interval` means |
|---|---|---|---|
| `sparkplug` | push (report by exception from device) | Devices publish; the host cannot poll a Sparkplug node | Ignored. Accepted for validation only |
| `opcua` | subscribe (MonitoredItems), poll fallback | Server-side sampling and change detection is what OPC UA is built for | Requested `SamplingInterval`; the subscription `PublishingInterval` is derived (opcua.md) |
| `modbus` | poll | Protocol has no unsolicited messages | Scan period |
| `s7` | poll | PUT/GET has no subscription for our use (S7 "cyclic data" services exist but are poorly documented, UNVERIFIED) | Scan period |
| `enip` | poll | Explicit messaging read; implicit I/O (Class 1) is out of scope | Scan period |
| `sim` | internal timer | | Generator tick |

Push drivers still report unchanged values when the protocol gives them (Sparkplug heartbeat NDATA, OPC UA keep-alive does **not** carry values). This matters for `stale_after`: for OPC UA, a slow-changing value produces no notifications, so `stale_after` on OPC UA tags is usually wrong. See opcua.md section 4.

## 2. Polled drivers send every sample

A polled driver sends every read result to the host, changed or not. The tag engine already discards unchanged repeats and uses them to reset `stale_after` (tag-model section 6). Suppressing repeats in the driver would break `stale_after` and would hide a stuck PLC from the "last update" timestamp.

Cost check: 20 000 items at 1 s is 20 000 samples/s, which is about 20 `SampleBatch` messages/s over a Unix socket. That is small. If profiling shows a problem, an additive driver-side option `report: { on_change: true, max_interval: 30s }` can be added later.

## 3. Scan classes

A **scan class** is the set of items on one connection that share the same effective interval. Scan classes are not configured by name in v1. They come from the distinct `source.interval` values on the tags:

```
effective_interval = Item.Interval if > 0 else settings.scan.default_interval (default 1s)
effective_interval = max(effective_interval, settings.scan.min_interval)          (default 100ms)
effective_interval = round up to a multiple of settings.scan.granularity            (default 100ms)
```

Rounding stops 1 000 tags with slightly different intervals from producing 1 000 scan classes. Example: tags at `1s`, `1s`, `950ms`, `5s` and `0` (default) produce classes `1s` (4 items) and `5s` (1 item).

### Phase alignment

Each scan class ticks at wall-clock multiples of its interval (`next = truncate(now, interval) + interval`), plus a fixed per-connection offset (hash of connection name mod interval, max 1 s). Effects:
- `1s` and `5s` classes on the same connection line up every 5 s, so they can share one request when they overlap (the planner merges due classes).
- Different connections do not all fire at `:00.000`, which spreads CPU and network load.
- History samples from polled devices land at predictable times, which makes trends easy to read.

### Example connection settings

```yaml
apiVersion: skyranger/v1
kind: Connection
metadata:
  name: plc1
spec:
  driver: modbus
  enabled: true
  timestamp_source: gateway
  settings:
    host: 192.168.10.20
    port: 502
    scan:
      default_interval: 1s
      min_interval: 200ms          # protect a weak device
      granularity: 100ms
      max_concurrent_requests: 1   # in-flight requests on this connection
    batching:
      max_registers_per_read: 120  # under the protocol limit of 125, see modbus-tcp.md
      max_gap_registers: 10        # read through gaps up to 10 unused registers
      max_coils_per_read: 1968
```

## 4. Scheduler

`pkg/driver/sdk/scan` is shared by `modbus`, `s7`, `enip` and the OPC UA poll fallback.

```go
package scan

// ReadFunc executes one planned request. It returns one Sample per item in the request.
// It must honour ctx (request timeout). Errors become per-sample qualities inside ReadFunc.
type ReadFunc func(ctx context.Context, req Request) []driver.Sample

// Planner turns the items due this tick into protocol requests (block reads).
type Planner interface {
    Plan(items []driver.Item) []Request // deterministic, cached until item set changes
}

type Request struct {
    Key   string          // e.g. "1/hr/40001-40120" for logs and stats
    Items []driver.Item   // items served by this request
    // protocol-specific payload set by the Planner (start, count, tag list ...)
    Payload any
}

type SchedulerOptions struct {
    MaxConcurrent  int           // settings.scan.max_concurrent_requests
    RequestTimeout time.Duration // settings.timeouts.request
    Granularity    time.Duration
    MinInterval    time.Duration
}

type Scheduler struct {
    classes map[time.Duration]*class   // one per effective interval
    planner Planner
    read    ReadFunc
    sink    driver.Sink
    sem     chan struct{}              // MaxConcurrent
    overruns atomic.Uint64
}

func (s *Scheduler) Add(items []driver.Item)       // re-plans affected classes
func (s *Scheduler) Remove(handles []uint32)
func (s *Scheduler) Pause()                        // on disconnect: stop ticking, keep plan
func (s *Scheduler) Resume()                       // on connect: immediate full scan, then ticks
func (s *Scheduler) Run(ctx context.Context)
```

Per-tick algorithm for one class:

```text
on tick(class):
  if class.inFlight:                      # previous cycle not finished
      overruns++ ; return                  # skip, do not queue (contract section 6)
  class.inFlight = true
  for req in class.plan (in address order):
      acquire sem (bounded by MaxConcurrent)
      samples += read(ctx with RequestTimeout, req)
      release sem
  sink.Values(ctx, samples)                # one flush per completed cycle, split into 1000-sample batches
  class.inFlight = false
  class.lastDuration = elapsed             # exported in Health details as scan_ms{interval}
```

Notes:
- **One flush per cycle** keeps related values (for example the 4 registers of a pump) in one batch, so the tag engine sees them together.
- `MaxConcurrent: 1` is the default for all serial-minded devices (most Modbus TCP gateways in front of RTU networks and small PLCs handle exactly one request at a time). S7 and Logix may allow more (see their docs).
- **Resume does an immediate full scan** so tags leave `bad_not_connected` within one request round-trip after reconnect, not one interval later.
- A request failing does not end the cycle. Its items get the mapped bad quality (reliability.md) and the next request runs. If N consecutive requests fail with timeout, the driver treats the connection as lost (reliability.md section 3).
- If a class's `lastDuration` is above 80 % of its interval for 10 cycles in a row, the driver logs a warning and sets connection state `degraded` with `last_error` `resource_exhausted` "scan class 1s near overrun". This warning comes before missed cycles.

## 5. Request batching (block planning)

The planner merges items into as few requests as possible, within protocol limits.

### Address-range protocols (Modbus, S7)

```text
group items by (unit/db/area, table)     # never mix in one request
sort by start address
for each group:
    block = first item range
    for next item:
        if next.start - block.end - 1 <= max_gap  and  (next.end - block.start + 1) <= max_per_read:
            extend block
        else:
            emit block; start new block
emit block
```

- Items that overlap (two tags on the same register, a float32 and its high word as uint16) share a block and are decoded from the same response bytes, so they are consistent with each other.
- **Bad-address isolation:** if a block read returns an "illegal address" style exception, the planner marks the block *suspect*, and the next cycle splits it in half recursively, down to single items. Individual bad items get `bad_config_error`; the good ones keep working. Split results are cached until the item set changes or the connection reconnects. Without this, one deleted register in a PLC would take out 120 neighbours.
- `max_gap` trades bandwidth for request count. The default of 10 registers is reasonable for TCP. Some devices fault on reads of unmapped gaps, so gap-reading can be disabled per connection with `max_gap_registers: 0`.

### Symbolic protocols (Logix via libplctag, OPC UA Read)

- Logix: one libplctag tag handle per item; libplctag batches requests into CIP Multiple Service Packets itself (UNVERIFIED: verify current libplctag request packing behavior). The planner only groups by scan class.
- OPC UA poll fallback: one `Read` service call per class with up to `settings.max_nodes_per_read` (default 500, also capped by the server's `MaxNodesPerRead` operational limit if it publishes one).

## 6. Push drivers: flush policy

Push drivers receive values one message at a time. They do not call `Sink.Values` per value. They buffer for at most `flush_interval` (default `50ms`) or 1000 samples, whichever comes first, then flush. One Sparkplug NDATA is always flushed as one batch so metrics in one payload stay together. The coalescing queue (contract section 6) sits behind this and applies only when the host stream is blocked.

## Decisions made
1. Polled drivers send every read result, changed or not. Change filtering belongs to the tag engine.
2. Scan classes are implicit: the distinct effective intervals on a connection, after clamping to `min_interval` and rounding to `granularity`. There are no named scan classes in v1.
3. Scan ticks are phase-aligned to wall-clock multiples, with a per-connection offset.
4. One `Sink.Values` flush per completed scan cycle. Overrun cycles are skipped. Resume after reconnect starts with an immediate full scan.
5. A shared block planner with gap merging, protocol caps, and recursive splitting to isolate bad addresses.
6. `max_concurrent_requests` defaults to 1 per connection.

## Alternatives rejected and why
- **Named scan classes (`fast`, `slow`) in config**: familiar from Ignition and Kepware, but `source.interval` is a duration in the contract. Durations are self-describing and need no extra config kind. This could come back as an additive alias.
- **Driver-side deadband/change-only reporting**: breaks `stale_after` semantics and duplicates tag-engine logic.
- **Fixed-rate tickers started at subscribe time (no alignment)**: random phase makes merged requests impossible and trends noisy.
- **Queueing overrun cycles**: builds unbounded latency, which is the classic "SCADA shows 40 s old data" failure.
- **Tiny per-item requests**: simple, but a 200-tag Modbus device would need 200 round trips per second.

## Open questions
- Should a "leased" scan class (fast only while an HMI screen is showing the tag) exist? It is useful for big PLCs, but it needs the tag engine to signal interest to drivers, which the contract has no message for.
- S7 cyclic services and Logix CIP connected messaging could reduce load. Both are UNVERIFIED and deferred.
- `scan_ms` per class is reported in Health `details`, but `HealthResponse` in the contract has no details map. Either add `map<string,string> details` (additive) or keep these in logs only.
