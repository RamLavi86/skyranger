# `enip` Driver: EtherNet/IP (Allen-Bradley Logix)

Reads and writes symbolic tags of Rockwell/Allen-Bradley Logix controllers (ControlLogix, CompactLogix) over EtherNet/IP explicit messaging (CIP over TCP port 44818). Contract inputs: [driver-interface.md](../00-contracts/driver-interface.md), [tag-model.md](../00-contracts/tag-model.md). Scan model: [scanning.md](scanning.md) section 5 (symbolic protocols). Link loss: [reliability.md](reliability.md).

Build decision: **buy** the protocol engine, **libplctag** (C, via a thin cgo wrapper), because CIP is large, partly unpublished outside ODVA membership, and libplctag is the one widely used open implementation. Licence and alternatives: [libraries-and-licensing.md](libraries-and-licensing.md).

All libplctag attribute names, function names and status names below are from memory of its public documentation. Everything marked UNVERIFIED must be checked against the libplctag version we pin before code is written.

## 1. Scope

| In v1 | Out of v1 |
|---|---|
| Logix family (ControlLogix, CompactLogix, "5000" series): symbolic read/write of atomic types, arrays elements, UDT members, bits of integers | PLC-5, SLC 500, MicroLogix (different addressing, `N7:0` style), Micro800 (different tag model), Omron NJ/NX over CIP |
| Explicit messaging, request/response polling | Implicit I/O (Class 1 cyclic, produced/consumed tags), CIP Safety, CIP Motion |
| Tag listing (browse) as a later phase | Message routing to remote chassis via several hops beyond a simple `path` |
| Controller-scoped and `Program:` scoped tags | Data Highway / ControlNet bridging |

The `plc` setting exists in settings so later families do not change the grammar. Only `controllogix` and `compactlogix` are accepted in v1. Micro800 is a likely early customer need (it is the cheap bench PLC, see test-bench.md) and is listed in Open questions.

## 2. Capabilities and settings

```go
driver.Info{
    Name:          "enip",
    AddressSyntax: "enip:{tag_name}?type={bool|int8|uint8|int16|uint16|int32|uint32|int64|uint64|float32|float64|string}[&bit={0-63}][&elems={n}]",
    Capabilities:  driver.Capabilities{Browse: true /* phase 4b */, Discovery: false, Write: true, Push: false, SourceTime: false},
}
```

```yaml
apiVersion: skyranger/v1
kind: Connection
metadata:
  name: clx-line2
spec:
  driver: enip
  enabled: true
  timestamp_source: gateway
  settings:
    plc: controllogix          # controllogix | compactlogix
    host: 192.168.30.5
    path: "1,0"                # CIP route: backplane port 1, slot 0 (CPU slot). CompactLogix: usually "1,0" as well (UNVERIFIED), see section 3
    timeouts: { connect: 3s, request: 2s }
    consecutive_timeouts: 3
    scan: { default_interval: 1s, min_interval: 250ms, max_concurrent_requests: 1 }
    max_tags: 2000             # cap on libplctag tag handles (each holds memory and a PLC-side read)
    write: { enabled: false }
```

## 3. Address grammar

`enip:{tag_name}?type={t}[&bit={n}][&elems={n}]`

`tag_name` is the Logix symbolic name exactly as the PLC knows it (case-insensitive in the PLC, but we compare and store it as written). Only the first `:` after `enip` is a delimiter, so `Program:MainProgram.Pump1.Speed` is fine. Allowed in the name part: letters, digits, `_`, `.`, `:`, `[`, `]` and `,` inside brackets. `ValidateAddress` rejects anything else. `?` and `&` are reserved for the query.

| Address | Meaning | Natural type |
|---|---|---|
| `enip:Pump1_Speed?type=float32` | controller-scoped REAL | `float64` |
| `enip:Program:Main.Count?type=int32` | program-scoped DINT | `int64` |
| `enip:Tank[3].Level?type=float32` | member of an array element of a UDT | `float64` |
| `enip:Flags?type=int32&bit=5` | bit 5 of a DINT, read-only in v1 | `bool` |
| `enip:Pump1.Run?type=bool` | BOOL member | `bool` |
| `enip:Names[0]?type=string` | Logix STRING (UDT: LEN + DATA[82]) | `string` |
| `enip:Recipe?type=float32&elems=10` | **invalid in v1**: arrays are not subscribable as one item (convert doc table). Address each element | |
| `enip:Pump1?type=int32` | where `Pump1` is a UDT: **invalid in v1** (structs are not subscribable, address members) | |

Type handling. libplctag needs an element size for a tag handle and returns raw bytes. The driver therefore takes the type from the address, not from the PLC. A wrong `type` gives garbage rather than an error when the sizes happen to match (a REAL read as DINT). That is a commissioning risk, so:

- The HMI/tag-import flow for Logix uses **Browse** (tag listing, phase 4b) to propose the exact type for each tag, and the engineer approves. Manual typing is allowed but flagged.
- After the first successful read, the driver checks the element size libplctag reports (`plc_tag_get_size`) against the width implied by `type` and reports `type_mismatch` when they differ.
- libplctag can read the CIP data type word via the tag listing (`@tags`) mechanism (UNVERIFIED: exact special tag names and the UDT template calls).

`bit` is read-only in v1: writing a single bit of a DINT would be read-modify-write, which can clobber bits the PLC program changed in between (reliability.md). Writable bits must be BOOL tags.

Signed/unsigned: Logix has only signed integer types (SINT, INT, DINT, LINT) plus newer unsigned variants on some firmware (UNVERIFIED). The natural type follows the `type` in the address.

## 4. How the driver uses libplctag

```go
package enip

/*
#cgo pkg-config: libplctag    // or vendored static build, see libraries-and-licensing.md
#include <libplctag.h>
*/
import "C"

// One handle per item. All handles of one connection share gateway+path, so libplctag
// multiplexes them over one CIP connection (UNVERIFIED: confirm connection sharing and
// request packing in the pinned version).
type item struct {
    handle uint32
    tagID  int32          // libplctag tag id
    typ    tag.DataType
    nat    nativeType     // size, signed, float
    bit    int            // -1 if none
}

func attrs(c Settings, name string) string {
    // protocol=ab-eip & gateway=host & path=1,0 & plc=controllogix & name=Pump1.Speed
    //   & elem_count=1 & timeout=2000 & (auto_sync_read_ms=... NOT used: see below)
    return fmt.Sprintf("protocol=ab-eip&gateway=%s&path=%s&plc=%s&name=%s&elem_count=1&timeout=%d",
        c.Host, c.Path, c.PLC, name, c.Timeouts.Request.Milliseconds())
}

// Poll cycle (called by scan.Scheduler through ReadFunc):
//   for each item in the class: plc_tag_read(tagID, 0)   // 0 timeout = non-blocking, starts the read
//   then wait for all with plc_tag_status(tagID) != PLCTAG_STATUS_PENDING, up to request timeout
//   then plc_tag_get_float32/int32/... and map status -> quality
```

Design choices:

- **Own scheduler, not libplctag's auto-sync.** libplctag can re-read tags by itself on a timer and by callbacks (`auto_sync_read_ms`, UNVERIFIED). We do not use it. The shared scheduler gives uniform overrun handling, scan alignment and one flush per cycle. We use libplctag only as "read this tag now".
- **Issue all reads of a class first, then wait.** Starting every `plc_tag_read` non-blocking and then collecting lets libplctag pack requests into CIP packets (UNVERIFIED how aggressively) and avoids one round trip per tag in sequence.
- **cgo threading.** libplctag creates its own threads. All calls from Go happen on goroutines that block in cgo, so `GOMAXPROCS=2` plus the process limits in framework.md section 5 apply. A hang inside the C library is the exact case the process-per-connection isolation exists for: Health times out, host restarts the process (reliability.md F7).
- **Tag handle lifecycle.** Create on `Subscribe`, destroy (`plc_tag_destroy`) on `Unsubscribe` and `Close`. A failed create (`PLCTAG_ERR_NOT_FOUND`-style status, UNVERIFIED name) is a per-item `not_found`, not a connection failure. Handle creation to a missing tag is asynchronous: creation returns while the first status is pending, so the driver waits for the first non-pending status before reporting the `ItemResult`.
- **Library log.** libplctag logging is routed into `slog` at debug level and is off by default (`plc_tag_set_debug_level`, UNVERIFIED).
- **Static build.** The driver is built in a separate image stage and statically linked against a vendored, pinned libplctag source tree with no local modifications, to satisfy the licence choice (libraries-and-licensing.md).

## 5. Quality and error mapping

libplctag reports an integer status. Symbolic names below are from memory (UNVERIFIED); the mapping is done by one `classify` function with a table-driven unit test fed from recorded statuses.

| libplctag status (UNVERIFIED names) | `Code` | Item quality | Link lost |
|---|---|---|---|
| `PLCTAG_STATUS_OK` | ok | `good` | |
| `PLCTAG_ERR_TIMEOUT` | `timeout` | `bad_comm_failure` | counts toward `consecutive_timeouts` |
| `PLCTAG_ERR_NOT_FOUND` (tag does not exist) | `not_found` | `bad_config_error` | no |
| `PLCTAG_ERR_BAD_GATEWAY` / `BAD_DEVICE` / `BAD_CONNECTION` | `not_connected` | `bad_comm_failure` then host `bad_not_connected` | yes |
| `PLCTAG_ERR_NO_DATA`, `PLCTAG_ERR_NO_MATCH` (size mismatch) | `type_mismatch` | `bad_type_mismatch` | no |
| `PLCTAG_ERR_NOT_ALLOWED` / write-protected (UNVERIFIED) | `not_writable` | `bad_config_error` | no |
| any CIP general status from the PLC (tag type unsupported, privilege violation, path error) | `device_error` with `Details{"cip_status": "0x.."}` | `bad_device_failure` | no |
| other / unknown | `internal` | `bad_comm_failure` | after 3 in a row |

CIP general status codes (from memory: `0x04` path segment error, `0x05` path destination unknown, `0x08` service not supported, `0x13` not enough data, `0x1E` embedded service error, `0xFF` general error with extended status) are placed in `Details` verbatim for the commissioning engineer. UNVERIFIED: check against CIP Volume 1 (ODVA) or libplctag docs.

A wrong `path` (bad slot) is a **configuration** error and looks like a timeout or path error, not an auth error. The connection stays in connect-retry with backoff, and `last_error` states "check path (slot) and that the CPU is in that slot". It does not become `faulted`, because the same symptom occurs when the PLC is simply unreachable behind a bridge.

## 6. Behaviour notes

- **Connection limits.** Each Logix controller has a finite number of CIP connections shared among HMIs, I/O modules, produced/consumed tags and engineers' laptops (the number depends on the Ethernet module, UNVERIFIED). One `enip` connection should use one CIP connection (shared by all tag handles). Do not create several Sky Ranger connections to the same PLC to "parallelise".
- **Atomicity.** Each tag read is atomic per tag. Two tags in the same scan class are not guaranteed to come from the same PLC scan. For coherent sets, read a UDT member group by making the PLC program copy values into one UDT, then read the elements. (Whole-UDT read is out of v1.)
- **Time.** No timestamps in this service. `timestamp_source: gateway`.
- **Controller mode.** Reads work in run and program mode. Writes to tags may be refused or ignored depending on mode and tag external-access settings (Read/Write, Read Only, None) in Studio 5000. `External Access` set to `None` makes a tag invisible: `not_found`. Document in the commissioning checklist.
- **Load.** Heavy polling of thousands of tags can raise controller CPU and communication load. Default scan 1 s, `max_tags: 2000`, and the commissioning guide asks the integrator to check controller task/communication load.
- **Safety.** Explicit read is harmless. Writes follow the generic rules: off by default, no retry, timeout means unknown outcome. Never write to safety tags (the driver cannot tell; the tag model's `writable: false` default is the guard).

## 7. Browse (phase 4b)

Tag listing is a separate step after read/write works. It returns controller-scoped tags and program-scoped tags with their type names, and UDT member layouts, so the HMI can offer a tree and propose the `type`. Implementation depends on libplctag's tag-listing support (UNVERIFIED), so the capability is declared only when the pinned version supports it. Paging uses `PageSize`/`PageToken` from the contract. Large controllers (tens of thousands of tags) need a lazy, per-program browse rather than one dump.

## 8. Library alternatives (UNVERIFIED)

| Option | Licence (UNVERIFIED) | Verdict |
|---|---|---|
| libplctag (C) through cgo | believed dual LGPL-2.0+ or MPL-2.0; we elect MPL-2.0 | primary choice |
| `github.com/danomagnum/gologix` (pure Go) | believed MIT | spike candidate; if reads of 1000+ tags, reconnect and error typing are solid, it replaces cgo and simplifies the build. Unknown maturity |
| `github.com/libplctag/libplctag.go` or other Go wrappers | unknown | not used; we write the 150-line wrapper ourselves so the cgo surface is small and reviewed |
| Own CIP implementation | ours | rejected for v1: spec access via ODVA, large surface (forward open, multiple service packet, symbol/template objects) |
| Kepware / vendor gateway | commercial | the fallback integrators already know: expose Logix via OPC UA and use the `opcua` driver |

ODVA and trademarks: the EtherNet/IP and CIP specifications are controlled by ODVA, and conformance testing and the use of the "EtherNet/IP" trademark are ODVA matters. A **client** library such as libplctag is widely used without ODVA membership, but whether our product may be marketed as "EtherNet/IP compatible" is a legal question. Marketing wording stays "reads Allen-Bradley Logix controllers over Ethernet" until checked. UNVERIFIED; get a legal opinion before the first sale.

## 9. Conformance additions

- Address table in section 3 as unit tests (valid and invalid rows).
- `classify` table test for every status in section 5.
- Against libplctag's own `ab_server` simulator in a container (UNVERIFIED supported types and licence, see test-bench.md): read/write of DINT, REAL, BOOL, arrays elements, wrong tag name, connection cut.
- Tag-handle leak test: subscribe/unsubscribe 10 000 times, goroutines, RSS and open sockets stay flat.
- Fault: kill `ab_server` mid-scan, all items `bad_comm_failure` then `bad_not_connected`, reconnect within backoff, no handle leak.
- A recorded capture from a real CompactLogix or Micro800 bench PLC is kept for replay.

## Decisions made
1. v1 supports the Logix family only, symbolic explicit messaging, polled by our shared scheduler. Implicit I/O and legacy families are out.
2. Engine: libplctag via a small cgo wrapper we write, statically linked from a pinned, unmodified source tree, built in a separate image stage, under the MPL-2.0 election (to be verified).
3. We do not use libplctag's automatic sync timers; the SDK scheduler owns timing.
4. The type comes from the address; Browse proposes it; the driver validates element size on first read.
5. Single-bit writes are not supported; bit reads are.
6. A wrong CIP path is a retrying configuration error, not `faulted`.
7. Protocol-level `Details` carry CIP status codes verbatim for commissioning.

## Alternatives rejected and why
- **Pure-Go own CIP stack now**: large effort for a protocol with a narrow v1 need; libplctag has years of field use.
- **libplctag auto-sync and callbacks**: second scheduler with different overrun and flush behaviour; harder to test with the conformance suite.
- **Whole-UDT and array reads in v1**: types and alignment rules (UDT templates) are a project in themselves; element addressing covers real needs.
- **Reading by CIP instance ID**: brittle across program downloads.
- **Making a wrong path `faulted`**: same symptom as an unreachable PLC; cannot be told apart reliably.

## Open questions
- Micro800 (Micro820/850) and MicroLogix/SLC/PLC-5 support: libplctag supports several (UNVERIFIED), but tag models differ. Micro800 is the cheap bench PLC and a probable small-site customer. Decide after the first Logix customer.
- Whether libplctag (pinned version) packs multiple reads per CIP packet and shares one CIP connection per PLC. Must be measured on the bench with a packet capture.
- gologix spike result (maturity, performance, licence file).
- Exact status code names and numeric values, CIP status table, tag listing API. All UNVERIFIED.
- Static linking with musl vs glibc for the cgo stage (deployment doc).
- ODVA wording and conformance questions for marketing.
- Contract note: the `enip:` scheme is not yet in driver-interface.md section 5. Additive, needs a decisions.md entry.
