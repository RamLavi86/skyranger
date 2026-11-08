# Reconnect, Backoff, Failure Behavior and Quality Mapping

Contract inputs: five `ConnState`s and their item-quality effect, error `Code`s with `Retryable`, "value-path failures become quality", "writes never retried", host process backoff (1 s to 60 s, reset after 5 min stable). This doc defines the **driver-side** connection state machine and the cross-driver failure rules. Per-protocol mapping tables are in each driver doc.

There are two independent loops, and they must not be confused:

| Loop | Owner | Triggers | Backoff |
|---|---|---|---|
| Process restart | driver host | crash, health timeouts, RSS limit | contract: 1 s, 2 s, 4 s ... 60 s, reset after 5 min |
| Connection reconnect | driver (inside the process) | socket error, session timeout, broker drop | this doc, section 2 |

A PLC being unplugged must **not** cause process restarts. The process stays healthy (Health answers) and reports `disconnected` while it retries the connection.

## 1. Driver connection state machine

```
                    Connect()
   (process start) ─────────► connecting ──────── success ───────► connected ◄─────┐
                                 ▲   │                                │  │          │
                                 │   │ retryable failure              │  │ some     │ all items
                     backoff     │   ▼                                │  │ items    │ recover
                     elapsed ────┴─ disconnected ◄── link lost ───────┘  │ failing  │
                                     ▲    │                              ▼          │
                                     │    │ non-retryable failure     degraded ─────┘
                                     │    ▼ (auth, bad settings)          │
                                     │  faulted ──(Configure with new settings)──► connecting
                                     └───── link lost ────────────────────┘
```

Transition rules:

| From | Event | To | Item quality effect (contract) |
|---|---|---|---|
| any | `Connect()` called | `connecting` | unchanged (`bad_waiting_for_initial_data` on fresh start) |
| `connecting` | session established, first successful read or broker CONNACK | `connected` | items updated as samples arrive |
| `connecting` | retryable failure (refused, timeout, DNS, TLS handshake I/O) | `disconnected`, schedule retry | all `bad_not_connected` |
| `connecting` | non-retryable (`unauthorized`, `invalid_config`, TLS cert rejected) | `faulted` | all `bad_config_error` |
| `connected` | link lost (socket error, N consecutive request timeouts, keep-alive lost) | `disconnected` | all `bad_not_connected` |
| `connected` | some items return per-item errors while others succeed | `degraded` | per-item quality |
| `degraded` | a full scan cycle (or `30s` for push drivers) with no per-item errors | `connected` | |
| `faulted` | `Configure` with different settings | `connecting` | |
| `faulted` | 1 hour elapsed | `connecting` (one attempt) | see below |

`faulted` and auto-retry: the contract says "no auto-retry until config changes". A password changed on the PLC side while the gateway is unattended for a weekend would otherwise need a site visit. We stay inside the contract by default, and offer `reconnect.faulted_retry: 1h` (default **off**) as a driver setting. With it, the driver tries once per period and goes straight back to `faulted` on the same error. This setting must never be used against devices that lock accounts after N failed logins. This is listed in Open questions.

"Connected" means proven: the driver reports `connected` only after a successful protocol exchange (Modbus: first successful read; S7: COTP + Setup Communication; OPC UA: session activated; Sparkplug: CONNACK + subscriptions acknowledged + STATE published). A TCP connect alone is not enough.

Implementation uses `sdk/connstate.Reporter`, which suppresses duplicate transitions. It treats an error with a different `Code` as a new event (it updates `last_error`) but does not send a new state change.

## 2. Reconnect backoff

```go
// sdk/backoff defaults for in-driver reconnect.
backoff.Backoff{
    Initial:    1 * time.Second,
    Max:        30 * time.Second,  // settings.reconnect.max
    Multiplier: 2,
    Jitter:     0.2,               // +/- 20 %, "full jitter" not used: we want bounded worst case
}
// Reset() is called after the connection has stayed connected/degraded for 60 s.
```

Sequence while a PLC is off: 1 s, 2 s, 4 s, 8 s, 16 s, 30 s, 30 s ... (each +/- 20 %). Recovery is noticed within 30 s of the device coming back.

Why 30 s and not the host's 60 s: the in-driver retry is cheap (one TCP SYN), and operators expect a PLC that comes back from a power cycle to show data quickly. The jitter stops 50 Modbus connections behind one switch that just rebooted from retrying in lockstep.

Device-specific rule: **never hammer a device that is overloaded.** If a connect succeeds but the very first request times out, that counts as a failure for backoff (it does not reset it). Some Modbus gateways accept TCP but have no free request slot.

Per-request timeouts while connected are not backoff. The scheduler retries on the next cycle.

## 3. Detecting link loss

| Driver | Link loss signal |
|---|---|
| `modbus` | socket read/write error; or `consecutive_timeouts` (default 3) request timeouts in a row with no success in between |
| `s7` | socket error; 3 consecutive timeouts; PDU-level "connection aborted" |
| `enip` | libplctag status codes for connection loss (UNVERIFIED: the exact codes; see ethernet-ip.md) |
| `opcua` | library session/secure channel state change, subscription keep-alive missing past `MaxKeepAliveCount * PublishingInterval` |
| `sparkplug` | MQTT client connection-lost callback; MQTT keep-alive is the detector |

TCP keep-alive is enabled on every driver socket (`net.Dialer{KeepAlive: 15 * time.Second}`) so that half-open connections (a cable pulled while the device is idle) are detected even for push protocols that have no traffic.

## 4. Error classification

Every driver classifies raw errors into the contract `Code` with one function, so that the classification is unit-testable:

```go
// Each driver implements this; tests feed it recorded errors.
func classify(err error) (code driver.Code, retryable bool, linkLost bool)
```

Generic Go network errors (all drivers):

| Go error | `Code` | Retryable | Link lost |
|---|---|---|---|
| `context.DeadlineExceeded`, `net.Error.Timeout()` | `timeout` | yes | counts toward `consecutive_timeouts` |
| `syscall.ECONNREFUSED` | `not_connected` | yes | yes |
| `syscall.ECONNRESET`, `io.EOF`, `syscall.EPIPE` | `not_connected` | yes | yes |
| `syscall.EHOSTUNREACH`, `ENETUNREACH` | `not_connected` | yes | yes |
| DNS `*net.DNSError` (not found) | `not_connected` | yes | yes. Log it clearly: often a config typo |
| TLS `x509` verification failure | `unauthorized` | **no** | goes to `faulted` |
| protocol decode error (bad frame) | `device_error` | yes | yes (stream is no longer in sync, reconnect) |

## 5. Quality mapping (all drivers)

Fixed rules on top of the contract table (tag-model section 4):

| Situation | Quality | Value carried |
|---|---|---|
| Item never read yet | `bad_waiting_for_initial_data` (set by tag engine) | `null` |
| Successful read | `good` (or the device-supplied quality for OPC UA and Sparkplug) | new value |
| Connection `disconnected` / process down | `bad_not_connected` (set by host on state change) | last value |
| Connection `faulted` | `bad_config_error` | last value |
| Request timeout, protocol error, connection still up | `bad_comm_failure` | last value |
| Device exception: point does not exist / illegal address | `bad_config_error` | last value |
| Device exception: device busy, gateway path failed, device fault | `bad_device_failure` (busy and gateway-path failures: `bad_comm_failure`, see modbus-tcp.md) | last value |
| Value cannot be converted exactly | `bad_type_mismatch` | last value |
| Device online but source silent (Sparkplug NDEATH, OPC UA `Uncertain*`) | `uncertain_last_known` | last value |

Who sets what:
- On connection state changes the **host** sets the quality of every item on the connection (contract section 3). The driver does not need to send one bad sample per item for `disconnected` or `faulted`. This keeps 20 000-item disconnects cheap.
- For per-item failures the **driver** sends a `Sample` with the bad quality. Carrying "the last value" (contract section 4) requires the driver to remember the last good value per handle. The SDK's coalescing store already has it, so drivers send `Sample{Handle: h, Quality: q}` with a null `Value`, and the SDK fills in the last value before sending. If no last value exists, the value stays null.
- Recovery: the first good sample after a bad period restores `good`. No hysteresis. Flapping is visible in history, which is the honest answer.

## 6. Failure scenario table

| # | Scenario | Driver behavior | What operator sees |
|---|---|---|---|
| F1 | PLC powered off | socket errors, state `disconnected`, backoff 1 s to 30 s | all tags `bad_not_connected` with last value, connection state tag `disconnected` |
| F2 | PLC back on | reconnect within the current backoff step, immediate full scan | tags `good` within about 30 s at worst |
| F3 | Cable pulled, idle push protocol | TCP keep-alive / MQTT keep-alive detects in about 15 s to 1.5 x keep-alive | as F1 |
| F4 | One register deleted in PLC program | block gets illegal-address exception, split, isolate | that tag `bad_config_error`, rest `good`, state `degraded` |
| F5 | PLC overloaded, requests time out | per-request `bad_comm_failure`; after 3 in a row, reconnect | items `bad_comm_failure`, then `bad_not_connected` if it persists |
| F6 | Wrong password / OPC UA cert rejected | `faulted`, no retry | all `bad_config_error`, `last_error` names the cause |
| F7 | Driver deadlock (library hangs) | Health times out 3 times, host kills it | `bad_not_connected` during restart (about 30 s detection + backoff), `restarts` increments |
| F8 | Driver panic | process exits, host restarts with backoff | as F7, but immediate |
| F9 | Tag engine slow | `Sink.Values` blocks, coalescing, `dropped_samples` increments | trend resolution drops, values stay current |
| F10 | Gateway clock jumps (NTP step) | no driver effect; schedule uses monotonic time for intervals, wall-clock only for alignment | possible one shortened or lengthened cycle |
| F11 | Broker restarts (Sparkplug) | MQTT reconnect, re-subscribe, republish STATE, request rebirth from all known nodes (rate-limited) | `bad_not_connected` then `good` after births |
| F12 | Device sends float NaN | `bad_type_mismatch` on that item | greyed last value |
| F13 | Write while disconnected | immediate `WriteResult` `not_connected`, no queueing | API 503 `not_connected` |
| F14 | Write times out | `WriteResult` `timeout`, outcome unknown, no retry | API 504; event `write_completed` with error `timeout` |

## 7. Writes

- `Write` is sent through the same connection and the same concurrency semaphore as reads. Writes take priority: the scheduler lets a write take the next free request slot before the next queued read.
- Read-modify-write (bit inside a register, Modbus FC22 or read-then-write) is **not** supported in v1. Writing one bit of a 16-bit holding register through read-modify-write can overwrite bits the PLC changed in between. Writable bit tags must use coils (Modbus) or BOOL tags (S7, Logix). Open question for later.
- The driver range-checks the raw value against the native type before sending (writing 70 000 to an int16 fails with `type_mismatch` and nothing is sent).
- After a successful write, the driver triggers an immediate read of that item (outside the scan schedule) so the HMI sees the effect without waiting for the next cycle. That read is not a verification and does not change the write result.

## Decisions made
1. Two loops: host restarts processes (contract backoff), drivers reconnect connections (1 s to 30 s, x2, +/- 20 % jitter, reset after 60 s stable).
2. `connected` is reported only after a successful protocol exchange, not after a TCP connect.
3. On connection-level changes the host sets item quality in bulk. Drivers send per-item bad samples only for per-item failures, and the SDK fills in the last value.
4. TCP keep-alive on every driver socket. Link loss after 3 consecutive request timeouts.
5. `faulted` does not auto-retry by default. An opt-in `faulted_retry` period exists for unattended sites.
6. Writes go first in the request queue, are never queued while disconnected, are never retried, and are followed by an immediate read-back for display.
7. No read-modify-write bit writes in v1.

## Alternatives rejected and why
- **One backoff for both loops**: conflates "PLC off" with "driver broken" and causes needless restarts.
- **Quality hysteresis (N bad reads before showing bad)**: hides real faults. Operators trust honest data more than smooth data.
- **Queue writes during disconnect and send on reconnect**: a delayed "start pump" command is dangerous.
- **Unbounded backoff (doubling forever)**: a site that comes back after a 2-hour outage would wait up to an hour for data.

## Open questions
- `faulted_retry` slightly extends the contract's "no auto-retry until config changes". Is this acceptable as an opt-in, or should it need a contract change?
- Does the host also need per-item quality from the driver at disconnect (for example for items on a Modbus gateway that fronts several RTU slaves, where one slave can be down while the TCP link is up)? Current answer: that is `degraded` plus per-item `bad_comm_failure`, which already works.
- Read-modify-write for bit-in-register writes (Modbus FC22 Mask Write Register would be atomic, but device support varies).
