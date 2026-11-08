# `s7` Driver: Siemens S7comm

Reads and writes Siemens S7-300/400/1200/1500 data blocks, merkers, inputs and outputs over ISO-on-TCP (port 102) using the classic S7comm "Read Var / Write Var" services (often called PUT/GET). Contract inputs: [driver-interface.md](../00-contracts/driver-interface.md), [tag-model.md](../00-contracts/tag-model.md). Scan and batching: [scanning.md](scanning.md) section 5 (address-range planner, shared with Modbus). Link loss and backoff: [reliability.md](reliability.md).

**Read this first: when not to use this driver.** Siemens has a better door for new PLCs. S7-1500 (and S7-1200 on recent firmware, UNVERIFIED which versions and whether a licence is needed) has a built-in OPC UA server with real authentication, encryption, symbolic names and subscriptions. If the customer's PLC has it, use the `opcua` driver ([opcua.md](opcua.md)). The `s7` driver exists for three cases: S7-300/400 and older S7-1200 firmware (no OPC UA), PLCs where the OPC UA server is not licensed or the integrator will not touch the TIA project, and cheap test benches (LOGO!, snap7 server). A decision table is in section 9.

All S7comm facts below are from memory of the public Snap7 documentation and Wireshark dissector behaviour. Siemens does not publish S7comm as an open specification. Everything marked UNVERIFIED must be checked against Snap7 sources, a packet capture from a real PLC, or both, before code is finalised.

## 1. Capabilities and settings

```go
driver.Info{
    Name:          "s7",
    AddressSyntax: "s7:{DBn.DBX|DBB|DBW|DBD}{byte}[.{bit}] | s7:{I|Q|M}{X|B|W|D}{byte}[.{bit}] ?type={bool|int8|uint8|int16|uint16|int32|uint32|float32|float64|string}[&len={n}]",
    Capabilities:  driver.Capabilities{Browse: false, Discovery: false, Write: true, Push: false, SourceTime: false},
}
```

```yaml
apiVersion: skyranger/v1
kind: Connection
metadata:
  name: s71200-pumps
spec:
  driver: s7
  enabled: true
  timestamp_source: gateway            # S7comm has no per-value timestamps
  settings:
    host: 192.168.20.10
    port: 102
    cpu: s7-1200                        # s7-300 | s7-400 | s7-1200 | s7-1500 | logo8 (sets rack/slot defaults)
    rack: 0                             # optional override
    slot: 1                             # optional override (1200/1500: 1; 300: 2; 400: depends on rack, check HW config)
    timeouts: { connect: 3s, request: 1s }
    consecutive_timeouts: 3
    scan: { default_interval: 1s, min_interval: 200ms, max_concurrent_requests: 1 }
    batching:
      pdu_size: auto                    # use the size the PLC grants during Setup Communication
      max_gap_bytes: 16                 # read through gaps of up to 16 unused bytes inside one DB
    write:
      enabled: false                    # writes need an explicit per-connection opt-in (section 8)
```

`max_concurrent_requests` stays at 1. Small Siemens CPUs have a limited number of simultaneous connections and a limited number of in-flight jobs per connection (the count negotiated at Setup Communication, often 1 on small CPUs, UNVERIFIED). Never open a second TCP session to "go faster". It can lock out the engineer's TIA Portal or another HMI.

## 2. Address grammar

URI-like, using Siemens' own notation, which engineers already know from TIA Portal and STEP 7. The scheme is `s7:` (no `//`), like `opcua:`. The datatype is not derivable from the notation (`DBD4` may be DINT, UDINT, REAL or TIME), so `?type=` is mandatory except where the notation fixes it.

```text
address  = "s7:" area "?type=" type [ "&len=" n ] [ "&order=" order ]
area     = db_addr | io_addr
db_addr  = "DB" db_number "." size byte [ "." bit ]       ; db_number 1..65535
io_addr  = ( "I" | "Q" | "M" ) size byte [ "." bit ]      ; inputs, outputs, merkers
size     = "X" | "B" | "W" | "D"                          ; bit, byte, word (2 bytes), dword (4 bytes)
bit      = 0..7                                            ; only with size X
```

| Address | Meaning | Natural type |
|---|---|---|
| `s7:DB10.DBX0.3?type=bool` | DB10, byte 0, bit 3 | `bool` |
| `s7:DB10.DBW4?type=int16` | DB10, byte 4..5 | `int64` |
| `s7:DB10.DBD8?type=float32` | DB10, byte 8..11 (REAL) | `float64` |
| `s7:DB10.DBD12?type=uint32` | DB10, byte 12..15 (DWORD / UDINT) | `uint64` |
| `s7:DB10.DBB20?type=string&len=16` | DB10, byte 20..37: STRING[16], 2 header bytes (max len, actual len) + 16 | `string` |
| `s7:M10.0?type=bool` | merker bit | `bool` |
| `s7:MW20?type=uint16` | merker word | `uint64` |
| `s7:I0.0?type=bool` | process-image input bit | `bool` |
| `s7:Q4.1?type=bool` | process-image output bit | `bool` |
| `s7:DB10.DBX0?type=bool` | **invalid**: `X` requires a bit number | |
| `s7:DB10.DBW5?type=float32` | **invalid**: `W` is 2 bytes, float32 needs 4 (size must match type width) | |
| `s7:DB10.DBD8?type=float32&order=cdab` | **invalid**: S7 data is always big-endian, `order` is not accepted in v1 | |

Rules:
- Size letter and type width must agree (`B`=1, `W`=2, `D`=4 bytes; `float64` has no size letter and uses `DB10.DBD8?type=float64`, which spans 8 bytes. The driver reads `8` bytes when type is `float64`. The size letter `D` is only the start-address spelling in this one case). UNVERIFIED for LREAL layout alignment rules on S7-1500 optimized blocks (not reachable by this driver anyway, see section 3).
- Odd byte offsets for `W`/`D` are legal in the protocol. The driver does not enforce alignment, but warns in `ValidateAddress` output when a `W`/`D` is on an odd offset, because that is almost always a mistake in an unoptimized DB.
- `Q` writes are allowed only when `write.enabled: true` and are warned about in the HMI (writing a process-image output that the PLC program also drives is overwritten on the next PLC scan, UNVERIFIED but standard PLC behaviour).
- All multi-byte values are big-endian. There is no byte-order switch.
- `ValidateAddress` returns `AddressInfo.NativeType` in the form `DB10.DBD8:REAL@byte=8,len=4`.
- Bit-in-word access to a word-typed value is expressed as `DBX` on the byte address (`DB10.DBX4.2` is bit 2 of byte 4), never `DBW4.2`.
- Not in v1: counters/timers areas (`C`, `T`), `S5TIME`, `DATE_AND_TIME` (BCD), `DTL`, `WSTRING`, arrays, UDTs. Arrays and UDTs are addressed by their elements. Return `unsupported` for any unknown syntax.

## 3. The PLC-side switches (commissioning checklist)

The most common reason an S7 driver "does not work" is PLC configuration, not the driver. The driver documents this in its `last_error` text and the HMI shows the checklist.

| PLC family | Needed on the PLC | Symptom if missing |
|---|---|---|
| S7-300/400 | Usually nothing. Check rack/slot and that the CP/CPU Ethernet port is configured | connect fails or `Setup Communication` rejected |
| S7-1200 / S7-1500 | In TIA Portal CPU properties: *Protection and security > Connection mechanisms > Permit access with PUT/GET communication from remote partner* enabled; and the CPU protection level must not forbid it | connection works but every read returns an access-denied item code |
| S7-1200 / S7-1500 | Target DBs must have **Optimized block access disabled** (standard, absolute addressing), because the driver addresses by byte offset | access-denied or "object does not exist" item codes on optimized DBs |
| all | Enough free connection resources on the CPU (each client takes one) | connect succeeds then drops, or connect refused |

Both PLC settings reduce the security of the PLC (anyone on the network can read and write without a password). That is a property of the protocol, not a bug we can fix. See section 8 and the security doc.

Item return codes (S7 read/write item response, from memory): `0xFF` success, `0x0A` object does not exist (DB missing or out of range), `0x05` address out of range, `0x03` access denied, `0x07` data type inconsistent, `0x06` data type not supported. UNVERIFIED: confirm the full list against Snap7 `s7_types.h`.

## 4. Connection sequence and PDU planning

```text
 TCP connect :102
   -> COTP Connection Request (src/dst TSAP derived from rack+slot)  -> COTP Connection Confirm
   -> S7 "Setup Communication" (max AmQ calling/called, PDU size request 480)
   <- granted PDU size (e.g. 240 on S7-300, 480 on S7-1200/1500, up to 960 on some CPUs; UNVERIFIED)
   -> state = connected, immediate full scan
```

"Connected" means Setup Communication succeeded (reliability.md section 1), not merely TCP.

```text
connection states for this driver (reliability.md owns the generic machine):

 disconnected --dial--> connecting --COTP CC + Setup OK--> connected
      ^                      | refused / TSAP rejected          |  item codes all 0x03/0x0A for
      |                      v                                  |  every item for 3 cycles
      |                 (backoff 1s..30s)                       v
      +<---- socket error / 3 timeouts ----------------- degraded (PUT/GET or optimized DB hint)
```

PDU planning (the S7 version of the block planner):

- Granted PDU size `P`. A Read Var request carries one 12-byte item spec per item, with a fixed header (about 12 bytes plus 2 for function and item count, UNVERIFIED exact overhead). The response carries per item a 4-byte data header plus the data, padded to even length, with a response header of about 14 bytes. The planner computes both directions and keeps each request and response within `P`. The maximum number of items per request is also capped at 20 (a well-known limit of the S7 Read Var service, UNVERIFIED).
- Contiguous or near-contiguous bytes in the same DB are merged into one item (`max_gap_bytes`), which is far cheaper than many small items.
- Items across different DBs go in the same request (up to the caps above), because each item carries its own DB number.
- An item larger than the data capacity of one PDU is split into consecutive items and reassembled. Reassembly happens only within one scan cycle, so two halves of a `float64` are always from the same request unless the planner had to split it (very large `string`), which is documented as non-atomic.
- Bad-address isolation (scanning.md section 5): an item return code `0x0A`/`0x05` marks that item `bad_config_error`; other items in the response are unaffected, because S7 returns a return code **per item**. That makes S7 easier than Modbus. If the whole request is rejected (header-level error), the planner falls back to the split-in-half rule.

Concurrency: one request at a time per connection (section 1). Typical cycle time for a 200-byte DB on a LAN is a few milliseconds to tens of milliseconds, so a 1 s class has large headroom. UNVERIFIED: measure on the real bench (test-bench.md).

## 5. Quality and error mapping

| Condition | `Code` | Item quality | Notes |
|---|---|---|---|
| TCP refused / reset / COTP refused | `not_connected` | host sets `bad_not_connected` | backoff |
| Setup Communication rejected | `device_error` | `faulted` is **not** used: retry with backoff, since a busy CPU can reject temporarily | |
| Request timeout | `timeout` | `bad_comm_failure` | 3 in a row means link lost |
| Item `0xFF` | ok | `good` | |
| Item `0x0A` object does not exist | `not_found` | `bad_config_error` | DB deleted or too short |
| Item `0x05` address out of range | `not_found` | `bad_config_error` | |
| Item `0x03` access denied | `unauthorized` per item | `bad_config_error` | PUT/GET disabled or DB optimized; connection `degraded` with a checklist hint, not `faulted`, because other areas may still work |
| Item `0x07` / `0x06` type errors | `type_mismatch` | `bad_type_mismatch` | |
| CPU in STOP | reads of DBs still usually succeed (values frozen). The driver cannot see this without extra work. Optional read of CPU state via SZL is **not** in v1 | `good` (stale-looking) | Section 7 explains the consequence: use `stale_after` on a PLC heartbeat tag |
| NaN/Inf REAL | | `bad_type_mismatch` | convert rule |

Why `0x03` does not fault the connection: contract rule 7 (reliability.md) says auth and config rejections go to `faulted` and stay. Access-denied on S7 is per area and could be one bad DB. We keep the connection running, set `degraded`, and put the hint in `last_error`. If **every** item on the connection returns `0x03` for 3 consecutive cycles, the driver moves to `faulted` with `invalid_config` "PUT/GET disabled or optimized block access enabled".

## 6. Library choice (UNVERIFIED throughout, see libraries-and-licensing.md)

| Option | Licence (UNVERIFIED) | Notes |
|---|---|---|
| `github.com/robinson/gos7` | believed BSD-style | Pure Go port of the Snap7 client concepts. Last-known activity unclear. Candidate for the spike |
| Snap7 C library via cgo | believed LGPL-3.0 (plus a GPL option). **Avoid in a shipped binary**: static Go + LGPL needs a relinking story | Excellent as a **test server** (it includes a server), see test-bench.md |
| `libnodave` | GPL | Rejected by policy |
| Own implementation | ours | Read Var / Write Var / Setup Communication / COTP: roughly 600 to 900 lines of Go |

Plan (phase 5, implementation-order.md): a time-boxed spike (2 days) tests `gos7` against a real S7-1200 and the snap7 server for: reads across several DBs, reads larger than one PDU, reconnect behaviour, context cancellation (can a request be abandoned without leaking a goroutine), thread safety and error typing. If it fails any of the five, write our own thin client modelled on the documented message layouts and verified with packet captures. Because the driver runs as its own process (framework.md), a library defect cannot take down the core either way.

## 7. Behaviour notes

- **The PLC is the authority on its own run state.** A CPU in STOP still answers reads. Values look "good" but are frozen. Guidance for integrators: add a heartbeat counter incremented in OB1 / cyclic OB and read it as a tag with `stale_after: 10s` in the tag model. This is the same pattern as the water meter heartbeat in decisions.md and needs no driver feature.
- **Atomicity.** Values in one request are consistent with each other (one PLC snapshot per request, not guaranteed across requests, UNVERIFIED for S7-1500 interrupt behaviour). Related values should be placed adjacent in one DB so the planner reads them in one request.
- **Time.** No timestamps in S7comm. `timestamp_source: gateway`.
- **Optimized access / symbolic.** Symbolic access to optimized blocks is S7CommPlus (S7-1200/1500), a different, proprietary and partly reverse-engineered protocol. Out of scope for v1 and likely forever; use OPC UA for those.

## 8. Writes

Writes are disabled by default per connection (`write.enabled: false`) and each write is rejected with `not_writable` until enabled, so that enabling S7 writes is a deliberate act in the connection config and is visible in the audit log (security doc). Rules from the contract apply: no retry, timeout means unknown outcome.

- Bit writes: S7comm Write Var supports bit-sized items, so `DBX`/`MX` writes are atomic at the PLC and **do not** need read-modify-write (unlike Modbus holding-register bits, reliability.md section on RMW). UNVERIFIED: confirm bit-write transport size on S7-300 and 1200 with a capture.
- Multi-byte writes are written in one item, atomic within the PLC's request handling (UNVERIFIED).
- A write to a DB the PLC program also writes is a race. The driver cannot know. The tag model's `writable` flag and the API's role checks are the guard.

## 9. S7 or OPC UA? (decision table for the integrator)

| Situation | Use |
|---|---|
| S7-1500, or S7-1200 with OPC UA server available and licensed | `opcua` |
| S7-300/400, S7-1200 without OPC UA | `s7` |
| Customer will not allow PUT/GET (security policy) and has no OPC UA | neither; add a Siemens-approved gateway or use Modbus TCP via a CP/program block |
| Need symbolic names, structured types, browse | `opcua` |
| Need minimum PLC load and no extra licence on an old fleet | `s7` |
| LOGO! 8 and tiny automation | `s7` (UNVERIFIED which LOGO! versions and TSAP values) or Modbus TCP |

## 10. Conformance additions

On top of the shared suite ([driver-interface.md](../00-contracts/driver-interface.md) section 8 and the conformance doc in 09-testing):

- Address table in section 2: every row is a unit test, including the invalid ones.
- PDU planner property test: random item sets at PDU sizes 240, 480, 960 never produce a request or response over the PDU size or over 20 items, and every item is covered exactly once.
- Per-item return code mapping table (section 5), with recorded responses.
- Reconnect test against the snap7 server with the TCP link cut by Toxiproxy (test-bench.md).
- A recorded capture from each real PLC model used on the bench is stored for replay (09-testing record/replay).

## Decisions made
1. The `s7` driver speaks S7comm Read Var / Write Var (PUT/GET) only. S7CommPlus is out of scope.
2. Address notation is Siemens' own (`DB10.DBD8`) in an `s7:` scheme with a mandatory `?type=`. Always big-endian. No byte-order option.
3. One TCP session per connection, one request in flight. Never open parallel sessions.
4. Access-denied item codes degrade the connection with a checklist hint and fault it only when every item is denied for three cycles.
5. Writes are off by default per connection and never retried. Bit writes are native, not read-modify-write.
6. Library: spike `gos7`; otherwise own thin client. No LGPL or GPL code in the shipped binary.
7. Prefer the `opcua` driver whenever the PLC offers a real OPC UA server.

## Alternatives rejected and why
- **Snap7 via cgo in the shipped binary**: mature, but LGPL with static linking is a licence burden for a sold product, and cgo adds a build stage. Used as a test server only.
- **libnodave**: GPL.
- **S7CommPlus implementation**: proprietary, partly reverse-engineered, changes with firmware, legal and maintenance risk.
- **Opening several sessions for speed**: exhausts the CPU's connection resources and can lock out engineering tools.
- **Treating access-denied as `faulted` immediately**: one misconfigured DB would blank every tag of the PLC.
- **Reading CPU run state in v1**: needs SZL parsing, which is poorly documented; the heartbeat-tag pattern covers the need.

## Open questions
- Exact PDU overhead numbers, the 20-item cap, grant sizes per CPU, and item return codes: UNVERIFIED, to be fixed against Snap7 sources and captures.
- Whether `gos7` is maintained and whether its licence is acceptable (spike).
- Address grammar addition: `s7:` is a new scheme not in driver-interface.md section 5; additive, but needs a decisions.md entry (also noted in [README.md](README.md)).
- LOGO! 8 TSAP and rack/slot conventions, and S7-1200 OPC UA availability by firmware, need checking on hardware.
- Whether to read CPU state (RUN/STOP) via SZL later and surface it as `_system/connections/{name}/plc_run_state`. That tag name would need a contract entry.
- Counters/timers, DTL and DATE_AND_TIME types: add on first customer demand.
