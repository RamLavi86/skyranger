# Libraries, Licences and Build-vs-Buy

Sky Ranger may be sold. Integrator first, product later ([decisions.md](../decisions.md) entry 5), with the licensing model deferred, but **dependency licences cannot be fixed afterwards without rewrites**. This file sets the licence policy for everything a driver links or ships, lists candidate libraries per protocol, and decides build vs buy.

Honesty rule: every licence, maturity and API statement below comes from memory unless it says "checked". All are **UNVERIFIED**. Before a library enters `go.mod` or the image, someone opens its `LICENSE` file at the pinned version, records the SPDX id, the date and the commit in the table in section 6, and the CI gate (section 4) enforces it. Nothing here is legal advice. A one-hour review by a lawyer who knows open-source licensing is part of the pre-first-sale checklist.

## 1. Licence policy

Why Go makes this stricter: Go binaries are statically linked, so "link" almost always means "combined into one work". Dynamic-linking exceptions that rescue LGPL in C do not help.

| Class | Licences (SPDX) | Policy |
|---|---|---|
| Green: allowed | MIT, BSD-2-Clause, BSD-3-Clause, ISC, Apache-2.0, 0BSD, Unlicense/CC0, MPL-2.0 (see below), EPL-2.0 / EDL-1.0 as used by Eclipse projects (see below) | Allowed in shipped binaries. Keep notices in `LICENSES/` and the SBOM (the install bundle already lists `LICENSES/` and an SBOM, see 08-deployment) |
| Amber: needs a written exception | LGPL-2.1/3.0, EPL-1.0, CDDL, anything with an attribution-advertising clause, anything custom | Not in shipped Go binaries. May be used as a separate unmodified dynamic library or a separate program only after review, recorded in section 6 |
| Red: forbidden in anything we ship | GPL-2.0/3.0, AGPL-3.0, SSPL, BUSL/BSL, Commons Clause, "non-commercial" or "evaluation" licences, source-available with field-of-use limits, unlicensed code (no LICENSE file) | Never linked, never bundled in the image. Allowed **only** in test tooling that is not shipped (section 5) |

Specific rules:

1. **MPL-2.0 is allowed** because its copyleft is per file. We use such libraries unmodified (vendored, pinned). If we must patch one, the patched files are published under MPL-2.0 and recorded. Patches go upstream where possible.
2. **Eclipse EPL-2.0** components (Paho, Tahu, Mosquitto) are acceptable when used unmodified. EPL-2.0 is weak copyleft at module level, and the Eclipse Go/C clients are commonly offered under a dual EPL-2.0 / EDL-1.0 (BSD-3) option, which we elect (UNVERIFIED per project).
3. **Dual licences**: we record which option we elect. libplctag is believed to offer LGPL-2.0+ or MPL-2.0; we elect MPL-2.0 (UNVERIFIED).
4. **Process separation is not a licence shield.** Putting GPL code in its own process behind gRPC may be "mere aggregation" when shipped as a separate program, but a driver designed around a GPL library is a grey area, and shipping GPL binaries in our image triggers source-offer obligations. Policy: no GPL anywhere in the shipped image apart from the base OS packages (Debian or distroless userland), which carry their own notices and source offers. The base image choice is in the deployment doc.
5. **Transitive dependencies count.** The CI gate scans the whole module graph, not only direct imports.
6. **Unmaintained or unlicensed repos** are treated as red regardless of how handy they are. A repo with no LICENSE file grants no rights.
7. **Protocol specifications are not code but have their own terms.** ODVA (EtherNet/IP, CIP), OPC Foundation (OPC UA; specs are free to read, use of the OPC trademark and compliance logo is controlled), Sparkplug (Eclipse, open spec), Modbus (open spec, Modbus Organization), S7comm (no public spec, reverse-engineered). We implement or use libraries for the wire protocol, avoid trademark use implying certification, and say "compatible with" in marketing only after review. All UNVERIFIED.
8. **Trademark hygiene**: product pages say "reads Siemens S7 PLCs" and not "Siemens-certified". No vendor logos.

## 2. Candidate libraries by protocol

Columns: licence (as believed), maturity (as believed), what we use it for. All UNVERIFIED unless the "checked" column says otherwise (none are checked yet).

### 2.1 Shared infrastructure

| Library | Licence | Use | Notes |
|---|---|---|---|
| Go standard library | BSD-3-Clause | everything | `net`, `log/slog`, `crypto/tls` |
| `google.golang.org/grpc`, `google.golang.org/protobuf` | Apache-2.0, BSD-3-Clause | driver gRPC over UDS | the contract fixes gRPC |
| `github.com/bufbuild/buf` | Apache-2.0 | proto lint/breaking | dev tool only |
| `github.com/google/uuid` | BSD-3-Clause | UUIDs | already in the contract |
| `github.com/jackc/pgx` | MIT | not a driver dependency | listed for the licence gate completeness |

### 2.2 Sparkplug B / MQTT

| Library | Licence | Maturity | Use |
|---|---|---|---|
| `github.com/eclipse/paho.mqtt.golang` | EPL-2.0 / EDL-1.0 dual (UNVERIFIED) | most used Go MQTT 3.1.1 client; had reconnect and ordering caveats historically (UNVERIFIED) | `sparkplug` driver (sparkplug.md decision) |
| `github.com/eclipse/paho.golang` (MQTT 5) | EPL-2.0 / EDL-1.0 (UNVERIFIED) | newer | not needed in v1 |
| `github.com/eclipse/tahu` `sparkplug_b.proto` | EPL-2.0 (UNVERIFIED) | spec reference artifact | generate Go code from the `.proto` used verbatim (contract) |
| Eclipse Mosquitto (broker, C) | EPL-2.0 / EDL-1.0 (UNVERIFIED) | very mature | separate process in the image |
| `github.com/mochi-mqtt/server` (embedded Go broker) | MIT (UNVERIFIED) | not evaluated | rejected for now, see section 7 |

### 2.3 Modbus

| Library | Licence | Maturity | Verdict |
|---|---|---|---|
| Own implementation | ours | n/a | **shipped client** (modbus-tcp.md) |
| `github.com/simonvetter/modbus` | MIT (UNVERIFIED) | client and server, seemed maintained | **test server in CI**; fallback client |
| `github.com/goburrow/modbus` | BSD-3-Clause (UNVERIFIED) | very widely used, little recent maintenance | reference only |
| `github.com/grid-x/modbus` | BSD-3-Clause (UNVERIFIED) | goburrow fork with additions | reference only |
| `pymodbus` (Python) | BSD-style (UNVERIFIED) | mature | test server in a container |
| `diagslave`/`modpoll` (Proconx) | proprietary freeware | | **not used**: not open source, licence terms for redistribution unclear |

### 2.4 OPC UA client and server

| Library | Licence | Maturity | Verdict |
|---|---|---|---|
| `github.com/gopcua/opcua` | MIT (UNVERIFIED) | the only serious pure-Go option known; subscription/reconnect stability unknown | client: spike first (opcua.md) |
| open62541 (C) | MPL-2.0 (UNVERIFIED; older releases were LGPL with a static-linking exception) | mature, widely deployed | fallback client and server via cgo, or a small separate C process. **Pin a version whose LICENSE says MPL-2.0** |
| Eclipse Milo (Java) | EPL-2.0 (UNVERIFIED) | mature | separate JVM process possible for the northbound server; heavy for edge boxes. Not preferred |
| node-opcua (Node.js) | MIT core, paid membership for some support (UNVERIFIED) | mature | rejected: Node runtime in the image for one component |
| `asyncua` (Python) | LGPL-3.0 (UNVERIFIED) | mature | **test server only**, never shipped |
| Unified Automation, Softing, Prosys SDKs | commercial | certified | only if a customer requires OPC UA certification; cost per unit |

### 2.5 Siemens S7

| Library | Licence | Verdict |
|---|---|---|
| `github.com/robinson/gos7` | BSD-style (UNVERIFIED) | spike; else own thin client (siemens-s7.md) |
| Snap7 (C++) | LGPL-3.0 + GPL options (UNVERIFIED) | **test server only** |
| `libnodave` | GPL | red |
| `go-snap7` style cgo wrappers | depends on Snap7 | rejected (LGPL static) |

### 2.6 EtherNet/IP

| Library | Licence | Verdict |
|---|---|---|
| libplctag (C) | LGPL-2.0+ or MPL-2.0 dual (UNVERIFIED) | **shipped engine**, MPL-2.0 election, pinned and unmodified (ethernet-ip.md) |
| `github.com/danomagnum/gologix` | MIT (UNVERIFIED) | spike; replaces cgo if good |
| libplctag `ab_server` | same as libplctag (UNVERIFIED) | test server |
| OpENer (open EtherNet/IP adapter stack) | BSD-style (UNVERIFIED) | adapter side only; not needed |

### 2.7 Test tools (not shipped)

| Tool | Licence | Use |
|---|---|---|
| OpenPLC Runtime v3 | GPL-3.0 (UNVERIFIED) | Modbus (and possibly S7 and EtherNet/IP) test PLC on a Pi or in a container |
| Toxiproxy | MIT (UNVERIFIED) | TCP fault injection (09-testing/04) |
| Wireshark / tshark | GPL-2.0 | captures and dissectors |
| Snap7 server, asyncua, open62541 examples | see above | simulators |

## 3. Build vs buy per protocol

| Protocol | Decision | Why | Effort (guess, solo, includes tests) | Biggest risk |
|---|---|---|---|---|
| Sparkplug B / MQTT | **Build the Sparkplug layer, buy the MQTT client** (Paho) and the `.proto` | Sparkplug state machine (births, seq, rebirth, STATE) is the product's core behavior. MQTT itself is easy to get mostly right and hard to get fully right | 2 to 3 weeks | Paho reconnect/ordering behavior |
| Modbus TCP | **Build** | ~1000 lines; we need exact control of transaction ids, one-in-flight, exception detail, planner | 1.5 weeks | none technical; byte order and addressing on real devices |
| OPC UA client | **Buy** (`gopcua`), fallback open62541 | stack is huge (security, sessions, subscriptions, encodings) | 2 weeks incl. 3 to 5 day spike | library stability, security policies |
| OPC UA server | **Buy** (library), separate process | same; plus certificate management | 2 to 3 weeks | server-side maturity of the Go option |
| EtherNet/IP Logix | **Buy** (libplctag) | CIP is large, spec needs ODVA access | 2 weeks incl. cgo build | cgo build, licence election, connection sharing |
| Siemens S7 | **Spike buy (`gos7`), else build thin** | protocol is small, spec is community-documented; libraries are small and uneven | 1 to 2 weeks | no public spec, per-CPU quirks |
| BACnet, DNP3, IEC 104, PROFINET, Modbus RTU | **Not built**, reach via third-party gateway speaking OPC UA or Modbus TCP | narrow demand now | n/a | customers who need them directly |

Effort numbers are guesses to order the work, not commitments.

Rule of thumb used: **build** when the protocol is small, stable and we need exact behavior, or when a library would be our only source of a bug we cannot fix. **Buy** when the protocol is large, security-heavy, or has a long tail of vendor quirks a library already absorbed. Buying never removes our testing duty: every bought library gets the conformance suite and the fault tests.

## 4. The licence gate (CI)

```yaml
# licenses/policy.yaml  (checked in; edited only by a human, protected path)
allow:
  - MIT
  - BSD-2-Clause
  - BSD-3-Clause
  - ISC
  - Apache-2.0
  - MPL-2.0
  - EPL-2.0        # Eclipse components, unmodified; see policy rule 2
  - EDL-1.0
review_required:   # CI fails, and names this file section
  - LGPL-2.1
  - LGPL-3.0
  - EPL-1.0
deny:
  - GPL-2.0
  - GPL-3.0
  - AGPL-3.0
  - SSPL-1.0
  - BUSL-1.1
exceptions:        # each with date, reviewer, reason; empty at the start
  []
```

```go
// tools/licensecheck: runs in CI on every PR that touches go.mod, go.sum, vendor/ or a Dockerfile.
//  1. go-licenses (github.com/google/go-licenses, Apache-2.0, UNVERIFIED) report over ./... per driver binary
//  2. syft/Trivy SBOM for the image (SPDX), compared with the same policy
//  3. a vendored C source (libplctag, open62541) has its LICENSE file hash pinned in licenses/pins.txt
//  4. any unknown or missing licence is a failure, never a warning
```

The same gate produces the `LICENSES/` directory and SBOM already listed in the install bundle (08-deployment) and feeds the evidence pack in 09-testing.

## 5. Test-only exceptions

Red-class tools (OpenPLC, Wireshark) and amber tools (Snap7 server, asyncua) may be used in the lab and in CI **as separate programs** in test containers that are never part of a shipped image or SBOM. The compose files for these live under the testing area, and CI checks that no shipped image layer contains them.

## 6. Dependency register (to be filled during implementation)

| Dependency | Version/commit | SPDX | LICENSE file checked on | Reviewer | Shipped? |
|---|---|---|---|---|---|
| `paho.mqtt.golang` | tbd | tbd | | | yes |
| `gopcua/opcua` | tbd | tbd | | | yes (after spike) |
| libplctag | tbd | tbd | | | yes |
| `gos7` | tbd | tbd | | | decided by spike |
| `simonvetter/modbus` | tbd | tbd | | | test only |

An entry is complete only when SPDX and the check date are filled. Until then the dependency stays in a spike branch.

## 7. Embedded broker?

Running an embedded Go MQTT broker inside the core would simplify the box, but the architecture fixes Mosquitto as a separate process (contracts README). We keep it. Revisit only if packaging a second process proves painful.

## Decisions made
1. Green/amber/red licence policy enforced by a CI gate over the full module graph and the image SBOM. No GPL/AGPL/SSPL/BUSL in anything shipped.
2. MPL-2.0 and Eclipse EPL-2.0/EDL-1.0 libraries are allowed unmodified and pinned. LGPL is not allowed in Go binaries. Dual-licence elections are recorded.
3. Build Modbus and the Sparkplug layer. Buy OPC UA (gopcua, fallback open62541), EtherNet/IP (libplctag), and spike S7 (`gos7`, else thin own client).
4. GPL/LGPL tools (OpenPLC, Snap7, asyncua, Wireshark) are test-only, in separate containers, and CI checks they never reach a shipped image.
5. Protocols we do not implement are reached through third-party gateways.
6. Marketing says "compatible" or "reads" only after a trademark review.

## Alternatives rejected and why
- **"GPL is fine because it is a separate process"**: a gamble for a sold product, and the image would carry GPL source-offer duties.
- **Wait and check licences before release**: by then the code depends on them. Check at the first import.
- **Commercial SDKs from the start** (OPC UA, CIP): cost and lock-in before revenue, contradicting the open-standards goal. Reserved for a customer who demands certification.
- **Vendoring everything blindly**: pinning is good, but vendoring without a licence record hides risk. Pins and records go together.
- **A multi-protocol commercial gateway as our own southbound layer**: defeats the product.

## Open questions
- Every licence, maintenance state and API detail in this file: UNVERIFIED. First task of each driver phase is to check and fill section 6.
- Whether MPL-2.0 file-level obligations are acceptable for open62541 if we must patch it (probably yes, publishing the patch).
- Whether the project itself will be open source (and under which licence). That affects which of our own code is exposed to LGPL-style rules and is a decisions.md item, deferred with the licensing model.
- Legal review budget and timing (suggest: before the first paid project).
- Whether distroless images satisfy source-offer duties for base OS packages, or a notices bundle plus source mirror is needed (deployment doc).
