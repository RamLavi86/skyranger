# `modbus` Driver: Modbus TCP

The most common protocol on small water, wastewater and utility sites. It is also the driver most likely to be blamed for bad data, so the address handling has to be exactly right. Normative address grammar: [driver-interface.md](../00-contracts/driver-interface.md) section 5.

Build decision: **write our own client** (about 1 000 lines of Go). The protocol is small and stable. We need control over transaction ids, timeouts, one-request-at-a-time behavior, and exception details. See [libraries-and-licensing.md](libraries-and-licensing.md).

## 1. Capabilities and settings

```go
driver.Info{
    Name:          "modbus",
    AddressSyntax: "modbus://{unit_id}/{co|di|hr|ir}/{1-based register}?type={bool|int16|uint16|int32|uint32|float32|float64}[&order={abcd|cdab|badc|dcba}]",
    Capabilities:  driver.Capabilities{Browse: false, Discovery: false, Write: true, Push: false, SourceTime: false},
}
```

```yaml
apiVersion: skyranger/v1
kind: Connection
metadata:
  name: plc1
spec:
  driver: modbus
  enabled: true
  timestamp_source: gateway          # Modbus has no timestamps
  settings:
    host: 192.168.10.20
    port: 502
    timeouts: { connect: 3s, request: 1s }
    consecutive_timeouts: 3          # then treat link as lost
    inter_request_delay: 0s          # some serial gateways need 20-50ms
    scan: { default_interval: 1s, min_interval: 100ms, max_concurrent_requests: 1 }
    batching:
      max_registers_per_read: 120
      max_coils_per_read: 1968
      max_gap_registers: 10
      max_gap_coils: 64
    unit_overrides:                  # per-unit tuning behind a TCP-to-RTU gateway
      "3": { timeouts: { request: 3s }, max_registers_per_read: 32 }
    write:
      single_register_fc: 6          # 6 (FC06) for single-register writes, or 16 (FC16) always
      single_coil_fc: 5              # 5 (FC05) or 15 (FC15)
```

## 2. Address grammar and worked examples

`modbus://{unit_id}/{table}/{address}?type={t}[&order={o}]`

- `unit_id`: 0 to 255. Unit 255 or 0 is common for native TCP devices, 1 to 247 for devices behind an RTU gateway. We pass the value through unchanged.
- `table`: `co` coils (FC01 read, FC05/15 write), `di` discrete inputs (FC02), `hr` holding registers (FC03 read, FC06/16 write), `ir` input registers (FC04).
- `address`: **1-based Modicon number**, either with the table prefix digit (5- or 6-digit form) or without it. The table in the URI is authoritative, and the prefix must agree with it if present.

Normalisation rule (normative):

```text
let n = address as integer (leading zeros allowed: "000017" == 17)
prefix P of table: di=1, ir=3, hr=4   (coils have prefix 0, which is invisible)

table == co:
    n in 1..65536                       -> register = n
table in (di, ir, hr):
    n in 1..9999                        -> register = n                 # short form
    n in P*10000+1   .. P*10000+9999    -> register = n - P*10000       # 5-digit, e.g. 40001..49999
    n in P*100000+1  .. P*100000+65536  -> register = n - P*100000      # 6-digit, e.g. 400001..465536
    anything else                       -> invalid_address
        (includes a wrong prefix such as ir/40001, and short-form numbers >= 10000:
         registers 10000..65536 must be written in 6-digit form)

protocol offset (PDU address) = register - 1                            # 0..65535
```

Ambiguity: `hr/40001` could mean "register 40001 in short form" or "register 1 in 5-digit form". **We choose the 5-digit form**, which is what almost every PLC and device manual means. Short form is accepted only below 10000, so no number can be read two ways. Register 40 001 is written `hr/440001`. `ValidateAddress` returns the normalised form in `AddressInfo.NativeType`, for example `holding_register:float32@offset=0` (offset 0-based), so the HMI can show exactly what will go on the wire.

| Address | Table | Register (1-based) | PDU offset (0-based) | FC read | Registers read |
|---|---|---|---|---|---|
| `modbus://1/hr/40001?type=uint16` | hr | 1 | 0 | 03 | 1 |
| `modbus://1/hr/1?type=uint16` | hr | 1 | 0 | 03 | 1 (same point as above) |
| `modbus://1/hr/40001?type=float32&order=cdab` | hr | 1 to 2 | 0 | 03 | 2 |
| `modbus://1/hr/400100?type=int32` | hr | 100 to 101 | 99 | 03 | 2 |
| `modbus://1/ir/30010?type=int16` | ir | 10 | 9 | 04 | 1 |
| `modbus://3/co/17?type=bool` | co | 17 | 16 | 01 | 1 bit |
| `modbus://3/di/10001?type=bool` | di | 1 | 0 | 02 | 1 bit |
| `modbus://1/ir/40001?type=int16` | | | | | **invalid**: prefix 4 vs table ir |
| `modbus://1/hr/40001?type=bool` | | | | | **invalid** in v1: bit-in-register not supported (see Open questions) |
| `modbus://1/hr/12000?type=uint16` | | | | | **invalid**: short form must be below 10000; write `hr/412000` |
| `modbus://1/hr/412000?type=uint16` | hr | 12000 | 11999 | 03 | 1 |

Test fixture rule: every row of this table is a unit test, and a device-manual example from each first customer gets added to it.

## 3. Types and byte order

Modbus registers are 16-bit big-endian on the wire. Multi-register values follow `order`, written as the byte order of the 4 bytes of a 32-bit value. Register 1 holds bytes `A B` and register 2 holds bytes `C D` when `order=abcd`.

| `order` | Register 1 | Register 2 | Common name | Typical devices |
|---|---|---|---|---|
| `abcd` (default) | A B | C D | big-endian, "high word first" | Schneider (many), most spec-following devices |
| `cdab` | C D | A B | word-swapped, "low word first" | many energy meters, some Siemens configs (UNVERIFIED per device) |
| `badc` | B A | D C | byte-swapped | rare |
| `dcba` | D C | B A | little-endian | rare |

For `float64` (4 registers, bytes `A..H`), `order` describes the swap pattern generalised: `abcd` is `AB CD EF GH`, `cdab` reverses the register order (`GH EF CD AB`), `badc` swaps bytes inside each register, `dcba` does both. UNVERIFIED that `cdab` for 64-bit means full word reversal on real devices. Some devices only swap within 32-bit halves. A fifth order may be needed, which would be an additive change to the contract grammar.

| `type` | Registers | Natural type |
|---|---|---|
| `bool` (co, di only) | 1 bit | `bool` |
| `int16` / `uint16` | 1 | `int64` / `uint64` |
| `int32` / `uint32` | 2 | `int64` / `uint64` |
| `float32` | 2 | `float64` |
| `float64` | 4 | `float64` |

Byte order is the number-one commissioning problem. The HMI connection page should offer a "read raw registers" test (`Read` with `?type=uint16` across the range) so the engineer can compare all four interpretations next to each other. The driver needs no special API for this, because the HMI can issue four `Read` calls.

## 4. Protocol implementation

```go
package modbus

// MBAP header + PDU, one request in flight unless max_concurrent_requests > 1.
type client struct {
    conn    net.Conn
    tid     uint16                    // transaction id, incremented per request, wraps
    pending map[uint16]chan response  // only used when concurrency > 1
    mu      sync.Mutex
}

func (c *client) readRegisters(ctx context.Context, unit uint8, fc uint8, offset, count uint16) ([]byte, error)
func (c *client) readBits(ctx context.Context, unit uint8, fc uint8, offset, count uint16) ([]byte, error)
func (c *client) writeSingleRegister(ctx context.Context, unit uint8, offset, value uint16) error // FC06
func (c *client) writeMultipleRegisters(ctx context.Context, unit uint8, offset uint16, regs []byte) error // FC16
func (c *client) writeSingleCoil(ctx context.Context, unit uint8, offset uint16, on bool) error // FC05
func (c *client) writeMultipleCoils(ctx context.Context, unit uint8, offset uint16, bits []bool) error // FC15
```

Rules:
- Response validation: transaction id, protocol id `0`, unit id, function code and byte count must match the request. Any mismatch is a protocol error, and the driver reconnects (the stream may be out of sync). A late response for an earlier, timed-out transaction is discarded when concurrency is above 1. At concurrency 1, a late response after a timeout means the stream is out of sync, so the driver reconnects.
- Limits from the Modbus spec (from memory, verify: FC03/04 max 125 registers, FC01/02 max 2000 bits, FC16 max 123 registers, FC15 max 1968 coils). Defaults are slightly below these (120) because some devices fail at the maximum.
- Block planning uses the shared planner (scanning.md section 5), grouped by `(unit_id, table)`.
- A 32-bit value is never split across two requests. The planner treats an item's whole register span as one unit.
- Writes: `uint16`/`int16` use FC06 (or FC16 if configured), 32/64-bit values use FC16 so all words are written in one request. Coils use FC05.

## 5. Exception and error mapping

| Modbus exception | Meaning | `Code` | Item quality | Planner action |
|---|---|---|---|---|
| 01 Illegal Function | device does not support this FC | `unsupported` | `bad_config_error` | none |
| 02 Illegal Data Address | register does not exist | `not_found` | `bad_config_error` | split block to isolate (scanning.md) |
| 03 Illegal Data Value | count or value invalid | `device_error` | `bad_config_error` (on read: block too large; halve `max_registers_per_read` for this unit) | split block |
| 04 Server Device Failure | device fault | `device_error` | `bad_device_failure` | none |
| 05 Acknowledge | accepted, long-running | (write only) `ok` | | |
| 06 Server Device Busy | busy | `device_error`, retryable | `bad_comm_failure` | next cycle |
| 0A Gateway Path Unavailable | gateway cannot reach the unit | `not_connected`, retryable | `bad_comm_failure` | none |
| 0B Gateway Target Failed to Respond | RTU slave behind gateway silent | `timeout`, retryable | `bad_comm_failure` | does not count toward link-loss timeouts (the TCP link is fine) |
| request timeout | | `timeout` | `bad_comm_failure` | counts toward `consecutive_timeouts` |

`Error.Details` carries `{"modbus_exception": "02", "unit_id": "1", "function": "03", "offset": "99", "count": "2"}`, so a commissioning engineer can match it against the device manual. Exception codes are from memory and must be checked against the Modbus Application Protocol spec v1.1b3 (UNVERIFIED for 0A/0B naming).

A gateway in front of several RTU units: one silent unit gives 0B (or timeouts) only for its items. The connection goes `degraded`, not `disconnected`. Timeouts are counted **per unit** for link-loss decisions only when the device is a direct TCP device (`settings.unit_timeouts_are_link_timeouts: true`, default `true`. Set it to `false` for gateways).

## 6. Browse

Modbus cannot enumerate points. `Browse` returns `unsupported`. Instead, discovery for Modbus is **template-driven**: an engineer picks a device template (for example `eastron_sdm630`) whose tags carry the addresses, and approves it against a connection. The register map belongs in a template, not in the driver. An optional later "register scan" tool (read ranges, report which respond) should be an HMI/API tool that uses `Read`, not a `Browse` implementation, because scanning can upset some devices.

## Decisions made
1. Own Modbus TCP implementation, with no third-party library in the shipped binary.
2. 5-digit and 6-digit prefixed forms are interpreted as Modicon references. A prefix that conflicts with the table is an error. The normalised wire offset is shown to the user.
3. Default `order=abcd`. Four orders, as in the contract. 64-bit swap semantics are documented, with a risk noted.
4. Read limits default below the spec maximum (120 registers). On exception 03, the per-unit limit adapts down.
5. Concurrency 1 by default. A late response at concurrency 1 forces a reconnect.
6. Gateway exceptions 0A/0B degrade only the affected unit's items.
7. No Browse. Device register maps live in templates.

## Alternatives rejected and why
- **`goburrow/modbus`**: widely used and simple, but (as far as I know) has had little maintenance in recent years and has limited per-request control (UNVERIFIED current state). It is fine as a reference and test client.
- **`simonvetter/modbus`**: client and server, MIT, appears maintained (UNVERIFIED). It is a good candidate for the **test server** in CI, and a fallback client if writing our own slips.
- **0-based addressing in the URI**: the contract fixes 1-based. Most device manuals list 4xxxx numbers.
- **Treating `hr/40001` as short-form register 40001**: almost never what the user meant.

## Open questions
- Bit-in-register (`hr/40001.3` style) for read-only status words is common in water PLCs. Reading is safe and could be added as `?type=bool&bit=3` (additive to the contract grammar). Writing would need FC22 or read-modify-write (see reliability.md).
- Strings in registers (`type=string&len=10`) for device names and serial numbers. Additive later.
- Modbus RTU over serial and RTU-over-TCP: likely needed by customers. Same PDU code; a `transport: rtu|rtu_over_tcp` setting. Serial access from inside a container needs device passthrough (deployment doc).
- 64-bit word order variants beyond the four contract orders (see section 3).
