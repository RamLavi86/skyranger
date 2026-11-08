# 09 Security Testing

Design of the security controls lives in 07-security. This file defines how we **test** them continuously and what we can honestly claim afterwards. It covers: fuzzing every parser that touches untrusted input, API and protocol security tests, dependency/image/secret scanning, container and deployment checks, and external review.

Honesty rule: we do not claim a certification (for example IEC 62443 or any penetration-test "pass") that we have not obtained. The evidence pack lists scans run, fuzz-hours accumulated and findings fixed, not adjectives.

## 1. Fuzzing

### 1.1 Targets

Every function that parses bytes or text from outside the process gets a Go native fuzz target (`func FuzzXxx(f *testing.F)`, run with `go test -fuzz`). The golden files from file 01 are the seed corpus.

| ID | Target | Package (proposed) | Input | Invariants beyond "does not panic" |
|---|---|---|---|---|
| FZ-01 | Sparkplug B payload decode (`NBIRTH`, `NDATA`, `DBIRTH`, `DDATA`, `NCMD`) | `internal/sparkplug` | arbitrary bytes | returns error or valid payload; decoded metric count bounded by input size; no allocation over 10x input size |
| FZ-02 | Sparkplug topic parse | `internal/sparkplug` | arbitrary string | parse then format round-trips; never accepts `+`/`#` in names |
| FZ-03 | Sparkplug state machine (seq, bdSeq, alias table) | `internal/sparkplug` | sequence of structured events from the fuzzer | never marks `good` with stale data; rebirth rate limit of one per 30 s per node holds; state memory bounded |
| FZ-04 | Modbus TCP response parsing (MBAP + PDU) | `pkg/driver/modbus` | arbitrary bytes as a server reply | length fields honored; no out-of-range slice; exception codes map to contract qualities |
| FZ-05 | Modbus address strings `modbus://...` and Sparkplug `spb://...`, `sim://...` | each driver | arbitrary string | `ValidateAddress` never panics; accepted addresses round-trip |
| FZ-06 | OPC UA mapping layer (StatusCode and Variant to our quality and types) | `pkg/driver/opcua` | arbitrary variant values | mapping total and never improves quality; the OPC UA library itself is out of scope except for a dependency scan |
| FZ-07 | Config YAML parse + validate (all four kinds) | `pkg/config` | arbitrary YAML | no panic, bounded memory (YAML alias bombs: the parser limits expansion, a known class of YAML attack); valid input round-trips export/import to an empty diff |
| FZ-08 | Template substitution engine | `pkg/config/template` | template + params | `$${` literal survives; no recursion; output size bounded |
| FZ-09 | Tag path and reference parse | `pkg/tag` | arbitrary string | accept implies the ABNF; reject never panics; prefix matching agrees with the naive model |
| FZ-10 | REST request bodies (every endpoint's JSON decode) | `pkg/api` | arbitrary body per route, generated from OpenAPI schemas | server responds with RFC 9457 error for invalid, never 500; unknown fields ignored where the contract says so |
| FZ-11 | WebSocket client messages (`hello`, subscribe, ping) | `pkg/api/ws` | arbitrary frames | connection closed with a documented code or message ignored; per-connection memory bounded (1 MiB per connection, api.md) |
| FZ-12 | Query parameters: `limit`, `cursor` (opaque), `prefix`, time ranges, durations | `pkg/api` | arbitrary strings | cursor tampering never returns another user's data or an error 500; huge `limit` capped |
| FZ-13 | Event envelope and CloudEvents parse | `pkg/event` | arbitrary JSON | decode then encode stable; unknown fields ignored |
| FZ-14 | Duration and RFC 3339 timestamp parsing | `pkg/tag` | arbitrary strings | only contract formats accepted (Go duration strings, UTC `Z`) |
| FZ-15 | Driver gRPC service request handlers (`skyranger.driver.v1`) | `pkg/driverhost` | arbitrary proto messages | rejected cleanly; host never crashes from a malformed driver message (isolation boundary) |
| FZ-16 | ESP32 firmware parsers (NCMD decode via nanopb) built for host | firmware repo/dir | arbitrary bytes | no memory errors under AddressSanitizer. libFuzzer or AFL++ (real tools; build details UNVERIFIED) |

### 1.2 Sketch

```go
// internal/sparkplug/fuzz_test.go
func FuzzDecodePayload(f *testing.F) {
    for _, p := range goldenpayloads.All(f) { // testdata/contracts/sparkplug/*.bin
        f.Add(p)
    }
    f.Fuzz(func(t *testing.T, data []byte) {
        pl, err := sparkplug.DecodePayload(data)
        if err != nil {
            return
        }
        if len(pl.Metrics) > len(data) { // each metric costs at least one byte on the wire
            t.Fatalf("decoded %d metrics from %d bytes", len(pl.Metrics), len(data))
        }
        re, err := sparkplug.EncodePayload(pl)
        if err != nil {
            t.Fatalf("decoded payload does not re-encode: %v", err)
        }
        pl2, err := sparkplug.DecodePayload(re)
        if err != nil || !sparkplug.Equal(pl, pl2) {
            t.Fatalf("round trip changed payload: %v", err)
        }
    })
}
```

### 1.3 Campaign

| Run | Time per target | Where | Notes |
|---|---|---|---|
| Per commit | seeds only (a plain `go test` runs `testdata/fuzz` seeds) | every push | cheap regression of every past crasher |
| Nightly | 5 min per target | CI runner | parallel; new corpus entries uploaded as artifacts |
| Weekly | 1 h per target | lab runner | accumulated corpus restored from the corpus cache |
| Release | cumulative fuzz-hours per target reported (target: >= 24 h each before the first customer, **UNVERIFIED as a meaningful bar**, it is a minimum effort statement, not a proof) | | |

Rules:
- **A crasher becomes a committed regression file** under `testdata/fuzz/FuzzXxx/` in the fixing PR, so it runs on every commit forever. A fuzz fix without the crasher file is rejected by `testguard` (file 12).
- The accumulated corpus is stored in a cache/bucket (not in git beyond minimized seeds) and **minimized** periodically.
- Fuzz targets need **timeouts and memory bounds** (`-fuzztime`, per-input time limit through context) so a slow-input bug (algorithmic complexity) is reported as a finding.
- A **differential** oracle where one exists: Sparkplug decode compared with the Eclipse Tahu reference implementation in a CI-only Java or Python job (UNVERIFIED availability and licensing of a convenient harness).

## 2. API and authorization testing

| ID | Test | Method | When |
|---|---|---|---|
| SEC-A1 | **Route-by-role matrix.** Enumerate every route in `openapi.json`, call it with no token, an expired token, a revoked token, and a token for each role (`viewer`, `operator`, `engineer`, `admin`); assert `401`/`403` exactly as the permission table says. A route absent from the permission table fails the build (deny by default). This is the test promised in 07-security/02 | Go table test in the L4 stack, generated from OpenAPI | every PR |
| SEC-A2 | Schema-based API fuzzing | **Schemathesis** (property-based OpenAPI fuzzer, a real tool; exact flags UNVERIFIED) against the compose stack with an `engineer` key, plus a no-auth pass; checks: no 5xx, response conforms to schema, no stack trace in body | nightly |
| SEC-A3 | Token handling | token never in URL, logs, WS URL, or error bodies; WS `hello` required before any data; tokens opaque; API key stored hashed (DB row inspection in the judge account: the DB holds no plaintext key) | every PR |
| SEC-A4 | Login abuse | brute-force protection per 07-security (throttling or lockout); timing equality between unknown user and wrong password within a statistical tolerance | nightly |
| SEC-A5 | Object-level access | `viewer` cannot read audit endpoints beyond its role; cursor tampering; tag id guessing returns identical `404` for missing and forbidden where the security doc says so | nightly |
| SEC-A6 | WebSocket limits | oversized frame, 21st connection (`4429`), revoked token (`4401`), idle timeout, unsubscribed topic flood | every PR (L3/L4) |
| SEC-A7 | Request limits | very large JSON bodies, deeply nested JSON, large `limit`, 1000+ ids on `/tags/values` | nightly |
| SEC-A8 | Audit completeness | every mutating request in OpenAPI produces an `audit_log` row with before/after values; the test discovers mutating routes from the document, so a new route without audit fails | every PR |
| SEC-A9 | Config import safety | YAML with secrets in plain text rejected or flagged; export never contains secret values; `secret://` references resolved only server-side. Canary secret test (section 5) | every PR |
| SEC-A10 | TLS | gateway generates a self-signed certificate on first boot (api.md); scan with `testssl.sh` or `sslyze` (real tools): protocol versions, ciphers, certificate fields, no HTTP fallback, HSTS if specified. Expected profile is defined by 07-security/03 | nightly |
| SEC-A11 | CORS and CSRF | HMI is same-origin; cross-origin requests with credentials rejected; bearer token in header means CSRF is not a risk by design, tested anyway | nightly |

```go
//go:build integration

func TestAuthzMatrix(t *testing.T) {
    s := stack.Shared(t)
    spec := openapi.Load(t, s.URL("/api/v1/openapi.json"))
    perms := authz.LoadTable(t, "testdata/contracts/permissions.yaml") // route -> minimum role; reviewed, protected
    for _, op := range spec.Operations() {
        want, ok := perms.MinRole(op.Method, op.Path)
        if !ok { t.Errorf("%s %s has no permission entry (deny by default)", op.Method, op.Path); continue }
        for _, role := range []string{"", "viewer", "operator", "engineer", "admin"} {
            got := s.CallAs(t, role, op.SampleRequest())
            allowed := authz.RoleAtLeast(role, want)
            if allowed && (got.Status == 401 || got.Status == 403) {
                t.Errorf("%s %s as %q: got %d, want allowed", op.Method, op.Path, role, got.Status)
            }
            if !allowed && got.Status != 401 && got.Status != 403 {
                t.Errorf("%s %s as %q: got %d, want 401/403", op.Method, op.Path, role, got.Status)
            }
        }
    }
}
```

## 3. Protocol and network security tests

| ID | Test | Expected |
|---|---|---|
| SEC-P1 | MQTT without credentials | connection refused |
| SEC-P2 | MQTT as a device user publishes to `skyranger/v1/...` | denied by ACL (only `skyrangerd` may publish there, messages.md section 3). Test attempts publish and subscribes from another client to confirm nothing arrived |
| SEC-P3 | Device user publishes as another device's `edge_node_id` or sets another's Will | denied by ACL pattern (07-security/04-device-identity) or flagged; relates to F-DEV-09 |
| SEC-P4 | Device user subscribes to `#` | denied |
| SEC-P5 | TLS to the broker: wrong CA, expired cert, no TLS | refused; gateway reports `faulted` or `disconnected` with a clear `last_error`, no downgrade |
| SEC-P6 | Modbus and OPC UA ports exposed beyond intended segments | port scan of the compose stack from an "enterprise LAN" container: only the documented ports answer (list owned by 08-deployment/02-containers: 8443 HMI/API, 9100 metrics on localhost or LAN, broker ports) |
| SEC-P7 | OPC UA security modes: `None` rejected when policy requires signing (once the driver exists); untrusted server certificate rejected | per driver doc |
| SEC-P8 | Replay of an old Sparkplug birth/data message by a rogue publisher | detected or harmless: cumulative counters and `seq` rules mean the gateway ignores or flags it; asserted via oracle, not by assumption |
| SEC-P9 | Plant LAN and enterprise LAN isolation on the two-NIC box (HIL-NET-08) | no forwarding between interfaces |
| SEC-P10 | Test-only tools are unreachable: sim control API refuses `0.0.0.0` without the flag (file 03), is never in a release image | image inspection (section 4) |

## 4. Supply chain and static analysis

| Tool | What | Gate |
|---|---|---|
| `govulncheck` | Go module vulnerabilities with call-graph reachability (reduces noise) | per PR: fail on reachable vulnerabilities; nightly on `main` |
| `osv-scanner` or `npm audit` (HMI) and `pip-audit` (Python scripting runtime, test tools) | Non-Go dependencies. Tool choice among the three is open (they overlap) | per PR and nightly |
| **Trivy** (image, filesystem, config, secret scanners) | OS packages in every container image, misconfigurations in Dockerfiles and compose, secrets | per PR on built images; nightly; **release gate on the exact digest** |
| **Syft** CycloneDX SBOM | SBOM for each image and for Go modules (07-security/07-secure-update) | release |
| **Re-scan of shipped SBOMs** | Nightly job scans the SBOM of every release still deployed against the current vulnerability database, so a CVE published after release creates a ticket without waiting for a new build | nightly |
| `gosec` and `golangci-lint` security linters | Common Go code weaknesses | per PR |
| `gitleaks` | Secrets in commits | per PR, plus full-history scan weekly |
| `hadolint` | Dockerfile lint | per PR |
| License check (`go-licenses` or equivalent) | Prohibits licenses incompatible with distribution (to be listed by the project) | per PR on dependency change |
| `go mod verify`, lock files, pinned image digests | Integrity of fetched dependencies | per PR |
| Reproducible build check | Build twice, compare digests (where achievable; at minimum compare Go binary hashes with `-trimpath` and fixed build info) | release, **UNVERIFIED feasibility for container layers** |

**Severity policy.** Release is blocked by: any reachable critical or high vulnerability with an available fix; any secret found; any image with a root user where the hardening doc forbids it. Unfixed findings need an entry in `test/security/waivers.yaml` (protected path) with a reason, an owner, a compensating control and an **expiry date (maximum 90 days)**; an expired waiver fails CI. New direct dependencies need an explicit justification line in the PR (file 12) because agents add libraries casually, and each is attack surface.

### 4.1 Deployment-artifact assertions (run against every release image set)

```go
func TestReleaseImageHardening(t *testing.T) {
    for _, img := range release.Images(t) {
        c := docker.Inspect(t, img)
        assert(t, c.Config.User != "" && c.Config.User != "root", "%s runs as root", img)
        for _, forbidden := range []string{"/usr/bin/skyranger-sim", "/usr/bin/skyranger-capture", "/test/", "/scenarios/"} {
            assert(t, !docker.FileExists(t, img, forbidden), "%s contains test artifact %s", img, forbidden)
        }
        assert(t, !docker.HasAccount(t, img, "svc-test-harness"), "test account present in %s", img)
    }
    // compose-level: read-only root fs, dropped capabilities, no privileged, memory limits (08-deployment/02, 07-security/09)
    compose := composecfg.Load(t, "deploy/compose.yaml")
    for name, svc := range compose.Services {
        assert(t, !svc.Privileged, "%s is privileged", name)
        assert(t, svc.ReadOnly || allowlisted(name), "%s writable root fs", name)
        assert(t, len(svc.CapAdd) == 0, "%s adds capabilities", name)
    }
}
```

Exact expected values come from 07-security/09-hardening and 08-deployment/02-containers; this test encodes them so a drift is a red build.

## 5. Secret and data leakage tests

A **canary secret** approach avoids "looks fine":

1. The test stack is configured with unique, random canary strings as the MQTT gateway password, a Modbus-like credential, the DB password, an API key and a user password (for example `CANARY-7f3a...`).
2. After a scenario that exercises login, config export, backup, log-heavy faults (F-NET-*, F-DB-*), the audit log, support-bundle generation (if specified), and crashes (a deliberately panicking test driver), the test greps: every container log, config export, API response bodies captured by the contract tap, backup archives, error bodies, the `events` and `audit_log` tables, core/crash dumps if enabled, and the compose environment.
3. **Zero occurrences of any canary** outside the secret store. Any occurrence fails the build.

This directly tests the contract rule "secrets never appear in config; export never includes their values" (tag-model.md section 9).

## 6. Security regression for AI-written code

AI agents tend to introduce a few specific classes of defect. Targeted checks:

- `testguard` blocks insecure patterns by lint: `InsecureSkipVerify: true`, `math/rand` for tokens, string-built SQL (use parameters), `exec.Command` with shell strings, new HTTP routes without a permission entry, logging of whole config structs.
- Any new endpoint requires: OpenAPI entry, permission entry, audit behavior, matrix row (SEC-A1 enforces).
- New dependencies: justification in PR, vulnerability scan, license check (section 4).

## 7. External review

| Activity | When | Notes |
|---|---|---|
| **Independent penetration test** of the shipped appliance or VM image and its network exposure | Before the first customer whose gateway is reachable beyond an isolated OT LAN, and annually afterward | We do not claim "pen-tested" until a report exists. The report's summary and the fix status go in the evidence pack with the tester's consent |
| Threat model review against 07-security/01 | Each minor release | Each threat row has at least one test ID in this file or file 04, and `testguard` lists threats without one |
| Peer review of crypto/TLS/PKI choices | Once, when 07-security/03 is implemented | An outside reader catches what the author cannot |
| Vulnerability disclosure channel | Before first customer | `security.txt`-style contact; response targets belong to the security doc |

## 8. Reporting

CI emits `security-report.json`: fuzz-hours per target, crashers found/fixed, scanner versions and database dates, findings by severity, waivers in force with expiry, SBOM digests, matrix results. This feeds the evidence pack section "Security testing" (file 11).

## Decisions made
1. Every untrusted-input parser has a native Go fuzz target seeded from the golden corpus; crashers are committed as permanent regression inputs; fuzz-hours are reported.
2. Authorization is tested as a generated route-by-role matrix, with deny-by-default, directly from the OpenAPI document, so new endpoints cannot silently skip it.
3. Canary secrets, not eyeballing, prove secrets never leak into logs, exports, backups or errors.
4. The nightly job re-scans the SBOMs of *shipped* releases against new vulnerability data.
5. Waivers are explicit, owned, compensated and expire within 90 days.
6. External penetration testing is required before exposure beyond an isolated OT network, and we claim nothing until a report exists.

## Alternatives rejected and why
- **Only SAST and dependency scanning**: finds known classes, not protocol-handling bugs. Fuzzing is what catches parser faults.
- **A commercial DAST suite**: Schemathesis plus the route matrix already targets our specific API; a general scanner adds noise. Re-evaluate for the pen test.
- **OSS-Fuzz style continuous fuzzing**: OSS-Fuzz is for public projects and requires acceptance; we start with CI and lab fuzzing and can apply later if the project goes open source.
- **Claiming a standard's conformance early (IEC 62443)**: certification is a formal process; the evidence pack describes controls tested, not a certificate.
- **Ignoring unreachable vulnerabilities entirely**: govulncheck reachability reduces noise, but Trivy findings in OS packages are still tracked, because reachability analysis does not cover those.

## Open questions
1. Which of `osv-scanner`, `npm audit`, `pip-audit` to standardize on: tool overlap, UNVERIFIED which has the best signal for the HMI toolchain.
2. The permission table `testdata/contracts/permissions.yaml` is a new protected artifact; 07-security/02 must agree it is the single source for route permissions (the router and the test both read it).
3. Differential testing of Sparkplug decode against a second implementation needs a chosen reference (Eclipse Tahu language binding) and licensing review.
4. Fuzzing `skyrangerd` against a malicious *driver* (FZ-15) assumes drivers are less trusted than the host. 07-security/01 decides whether third-party drivers are in scope for v1.
5. Brute-force and lockout policy (SEC-A4) depends on a decision in 07-security/02 (not yet fixed here).
6. A dedicated vulnerability response SLA (time from CVE publication to patch) belongs to the release engineering doc, which does not exist yet.
