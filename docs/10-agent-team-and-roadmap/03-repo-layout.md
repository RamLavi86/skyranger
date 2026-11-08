# 03 Repository Layout

Binding inputs: [../00-contracts/](../00-contracts/README.md); path conventions already used by the area docs ([../01-tag-engine/go-design.md](../01-tag-engine/go-design.md), [../02-drivers/framework.md](../02-drivers/framework.md), [../04-alarms/10-go-design-and-performance.md](../04-alarms/10-go-design-and-performance.md), [../05-hmi-api/01-architecture.md](../05-hmi-api/01-architecture.md), [../09-testing/README.md](../09-testing/README.md)). This file merges them into one tree and records the conflicts it resolved (section 4).

The Go module path is not chosen (contracts README open question). Docs use `skyranger/...` as a placeholder; so does this file.

## 1. Principles

1. **One repository, one Go module, one `web/` workspace, one Python package.** A monorepo lets one PR change a contract, its golden files and every consumer, which is exactly what the contract-change procedure needs ([01-agent-team.md](01-agent-team.md) section 8). Customer site repos are separate ([../08-deployment/05-config-as-code.md](../08-deployment/05-config-as-code.md)).
2. **Every directory has one owner role** ([01-agent-team.md](01-agent-team.md) section 2.1), recorded in `OWNERS.yaml`.
3. **`pkg/` is the public, versioned surface** (contract types, the driver SDK, config types) that third-party drivers and tools may import. Everything else is `internal/`, which the Go toolchain protects from outside imports.
4. **Dependencies point inward and downward**, enforced by a lint rule (depguard-style, tool choice UNVERIFIED): `pkg/tag` imports nothing from the repo; modules do not import each other's internals, only seams.
5. **Generated code is committed**, produced only by `make generate`, checked in CI.
6. **Contracts are machine-readable where possible** (`proto/`, `schemas/`) and always have a prose twin in `docs/00-contracts`.

## 2. The tree

```
skyranger/
├── CLAUDE.md                     top-level agent instructions (protected)
├── README.md
├── CODEOWNERS  OWNERS.yaml       protected paths; role -> path globs for ownercheck
├── contracts.lock                hashes + contracts_version (protected)
├── SECURITY.md                   disclosure policy (proposed in 07; human decides)
├── go.mod  go.sum  Makefile  buf.yaml  buf.gen.yaml
├── .claude/                      agents/*.md, hooks/, settings.json (protected)
├── .github/workflows/            CI (protected; host UNVERIFIED)
│
├── cmd/                          one main package per binary, thin
│   ├── skyrangerd/               gateway daemon (wiring only)
│   ├── skyrangerctl/             host tool: init, backup, apply, support bundle (M3)
│   ├── skyranger-driver-sim/     }
│   ├── skyranger-driver-sparkplug/ }  three-line mains: sdk.Main(pkg.New)
│   ├── skyranger-driver-modbus/  }   (M3)
│   ├── skyranger-sim/            multi-protocol device simulator
│   └── skyranger-scriptd/        script worker supervisor (M4)
│
├── pkg/                          public, semver-stable once v1
│   ├── tag/                      DataType, Variant, Value, Quality (contract types)
│   ├── event/                    CloudEvents envelope and event type structs
│   ├── driver/                   Driver, Sink, Item, Sample, errors (contract)
│   │   ├── sdk/                  Main, gRPC server, coalesce/, scan/, backoff/, connstate/, convert/, addr/
│   │   └── conformance/          conformance suite (protected)
│   ├── config/                   kinds (Gateway, Connection, Template, TagGroup, later Script), parse, canonicalize, validate
│   └── sim/                      generators, seeded RNG, scenario engine
│       └── oracle/               ground truth (protected; independent of production code)
│
├── internal/                     not importable from outside the module
│   ├── clock/                    Clock interface, Real, Fake
│   ├── logging/
│   ├── daemon/                   wiring and lifecycle of skyrangerd
│   ├── tagengine/                registry/ derived/ subs/ checkpoint/ core/ engine/ enginetest/
│   ├── bus/                      internal MQTT mirror (skyranger/v1/...)
│   ├── driverhost/               spawn, supervise, gRPC client, InProcess()
│   ├── sparkplug/                pure protocol logic: seq, alias, bdSeq, birth/death
│   ├── store/                    repository interfaces (HistoryStore, ConfigStore, ...) and pg/ backend
│   ├── historian/                writer: ring, batcher, idempotent insert, replay
│   ├── spool/                    CRC-framed disk spool, shared by historian and alarms
│   ├── alarm/                    model/ state/ cond/ engine/ persist/ notify/ journal/ kpi/ apihook/   (M2)
│   ├── api/                      REST handlers, WS hub adapters, OpenAPI, HMI static hosting
│   ├── security/                 authn/ authz/ audit/ secrets/ pki/
│   ├── scripthost/               script host (M4)
│   └── ctl/                      skyrangerctl internals (M3)
│
├── drivers/                      one directory per driver implementation
│   ├── sim/  sparkplug/          (M1)
│   ├── modbus/                   (M3)
│   └── opcua/ enip/ s7/          (later; see implementation-order)
│
├── proto/                        protobuf sources (contract, Tier 1)
│   ├── skyranger/driver/v1/driver.proto
│   ├── skyranger/script/v1/      (M4)
│   └── third_party/sparkplug/    Sparkplug B .proto (Eclipse Tahu; licence check UNVERIFIED)
├── schemas/                      JSON Schema and OpenAPI (contract, Tier 1)
│   ├── config/                   gateway, connection, template, tag-group (+ script later) v1
│   ├── hmi/                      screen, navigation, symbol v1
│   ├── events/                   one schema per event type v1
│   └── openapi/v1.yaml           REST contract; served at /api/v1/openapi.json
├── gen/                          generated Go from proto and schemas (never hand-edited)
│
├── db/
│   └── migrations/               all PostgreSQL DDL (storage owns); expand/contract
│
├── web/                          TypeScript + React HMI (npm workspaces)
│   ├── apps/hmi/                 SPA served by skyrangerd
│   ├── packages/                 api-client/ live/ screen-model/ components/ tokens/
│   ├── screens/                  Screen/Navigation YAML for slice and demos
│   └── e2e/                      Playwright tests (owned by e2e role)
│
├── scripting/                    Python (M4): `skyranger` runtime package, `skyranger.testing`, worker bootstrap
│
├── firmware/
│   └── esp32-pulse-meter/        ESP32 pulse counter, pulse_meter.v1 profile (C, ESP-IDF assumed, UNVERIFIED)
│
├── simulator/                    non-Go simulator assets: Dockerfile, README, demo scenarios, `make demo` data
│                                 (the Go code is pkg/sim and cmd/skyranger-sim)
│
├── deploy/
│   ├── compose/                  compose.yaml, profiles: sim, dev, perf
│   ├── images/                   Dockerfiles for skyrangerd, drivers, scripts, hmi build stage
│   ├── mosquitto/  postgres/     broker config, init scripts, server settings
│   ├── security/                 PKI scripts, ACL templates (M5)
│   ├── systemd/                  units for skyrangerctl agent (M3)
│   └── appliance/                Yocto + RAUC (explicitly later; placeholder only)
│
├── examples/
│   └── site-pumphouse/           a config-as-code site repo layout: config/, host/, secrets.manifest.yaml
│
├── test/                         (protected parts marked P)
│   ├── harness/                  L3 in-process gateway on a fake clock
│   ├── scenarios/                YAML scenarios (P); proposed/ for agent proposals
│   ├── faults/faults.yaml        fault catalogue (P)
│   ├── integration/              L4 Go tests, build tag `integration`; tolerances.go (P)
│   ├── compose/                  L4 stack and overrides
│   ├── soak/  perf/              runners; judge.yaml and thresholds.yaml (P)
│   ├── hil/                      hardware rig controller (Python)
│   ├── replay/                   recorded captures (synthetic or anonymized only)
│   ├── evidence/claims.yaml      claims register (P)
│   ├── coverage-floors.yaml (P)  mutation-allow.yaml (P)
│   └── security/waivers.yaml (P)
├── testdata/
│   ├── contracts/                golden corpus shared by Go, TS and Python (P)
│   └── fuzz/                     fuzz crashers and seeds
│
├── tools/                        repo tooling (Go)
│   ├── testguard/ (P)  ownercheck/ (P)  soakjudge/ (P)  evidencepack/ (P)  shipcheck/
│
├── work/                         agent coordination, not product
│   ├── templates/                wp.md, question.md, protected-change-request.md (P)
│   ├── wp/                       WP-NNN-slug.md
│   └── questions/                Q-NNN.md
│
└── docs/                         design docs 00 to 10, decisions.md (P)
```

## 3. Notes on specific choices

- **`cmd/` stays thin.** Each `main` parses flags and calls a constructor. Logic lives in `internal/` or `pkg/` so it can be tested without a process. This also helps a Go newcomer: reading `cmd/skyrangerd/main.go` shows the whole system in one screen.
- **`pkg/config` is shared by `skyrangerd` and `skyrangerctl`.** The canonicalizer must live beside the schema so exporter and validator cannot drift ([../08-deployment/05-config-as-code.md](../08-deployment/05-config-as-code.md) open questions proposed exactly this).
- **`internal/sparkplug` versus the simulator.** The simulator must not import `internal/sparkplug` or the driver's MQTT library choice, so a protocol bug cannot hide on both sides ([../09-testing/03-simulator.md](../09-testing/03-simulator.md)). Enforced by the same import rule that protects oracle independence (testguard TG-09).
- **`pkg/sim` and `simulator/`.** The docs place the Go library at `pkg/sim` and the binary at `cmd/skyranger-sim` because the `sim` driver and the standalone simulator share the library. `simulator/` holds what is not Go: container build, demo scenarios, README. Moving all Go code under `simulator/` would be cleaner to browse but would break many doc references; listed as an open question.
- **Storage owns all DDL** (`db/migrations/`), as [../03-storage/README.md](../03-storage/README.md) decision 1 says. Other areas own row semantics. Alarm and audit tables are written by `internal/alarm` and `internal/security/audit` through `internal/store`.
- **`internal/spool`** exists because the storage area asks alarms to adopt the shared spool library rather than a private JSONL spool ([../03-storage/README.md](../03-storage/README.md) cross-area requests).
- **Firmware is in the monorepo** so the device profile, golden Sparkplug payloads and the firmware change in one PR when `messages.md` changes. It builds with ESP-IDF, not `go build`; `make` targets for it are optional so Go-only CI does not need the toolchain.
- **`work/`** keeps coordination files out of `docs/`, which stays design material.
- **Docs ownership**: every area keeps its directory; this area owns `docs/10-agent-team-and-roadmap/`. `docs/README.md` is the index.

## 4. Conflicts between area docs and how this file resolved them

| Topic | Doc A | Doc B | Resolution here | Who must update |
|---|---|---|---|---|
| Shared tag types package | tag engine: `pkg/tag` | alarms (10-go-design): "assumes `skyranger/internal/tag`" | `pkg/tag`, because contract types are public and the driver SDK needs them | alarms area doc |
| Clock package | tag engine: `internal/tagengine/clock` | testing: `internal/clock` used by "all time-dependent code" | `internal/clock`, shared | tag engine go-design |
| HMI e2e tests | HMI: `web/e2e/` | testing: `hmi/e2e/` (also in CODEOWNERS sketch, `/hmi/e2e/allowed-console.txt`) | `web/e2e/` (inside the workspace that builds the HMI) | testing README and guardrails CODEOWNERS list |
| Screen JSON Schema location | HMI: `web/packages/screen-model/schema/` | contracts treat schemas as Tier 1 artifacts; Go validates with the same files | canonical in `schemas/hmi/`, copied into the web package by `make generate` | HMI 01-architecture |
| Storage Go interfaces | storage README: `11-go-interfaces.md` (not yet written) | none | `internal/store` with `pg/` backend (assumed) | storage area when 11 is written |
| Sparkplug logic | drivers: `drivers/sparkplug` | testing mutation list: `internal/sparkplug` | both: protocol logic in `internal/sparkplug`, driver glue in `drivers/sparkplug` | none (consistent) |
| Config kinds location | tag-model / deployment propose `skyranger/config` | none | `pkg/config` | none |
| OpenAPI file | api.md serves `/api/v1/openapi.json`; HMI generates `types.gen.ts` from it | none | source in `schemas/openapi/v1.yaml`, JSON served | api contract owner |

These are recommendations until the human accepts them with `decisions.md` entries (README Open questions). Until then agents follow this table, because a single answer is better than two.

## 5. Build and test entry points

All through `make`, with CI running the same targets ([../09-testing/10-ci-and-quality-gates.md](../09-testing/10-ci-and-quality-gates.md) section 8): `generate`, `lint`, `test-unit`, `test-contract`, `test-conformance`, `test-harness`, `cover-check`, `testguard`, `sec-fast`, `images`, `smoke`, `pr` (the local chain), `demo`. None exists yet; `CLAUDE.md` marks them TODO. Go side: `go build ./...`, `go test ./...`; web side: `npm ci`, `npm test --workspaces`; Python side: pytest (tool choice UNVERIFIED).

## Decisions made
1. One monorepo with a single Go module; `pkg/` for the public contract-facing surface, `internal/` for everything else.
2. Contracts are split into Tier 1 machine-readable sources (`proto/`, `schemas/`) with committed `gen/` output, plus `testdata/contracts` goldens.
3. The conflicts in section 4 are resolved as listed; `pkg/tag`, `internal/clock`, `web/e2e` and `schemas/hmi` are the chosen answers.
4. `simulator/` is for non-Go simulator assets; Go simulator code stays at `pkg/sim` and `cmd/skyranger-sim`.
5. Firmware, scripting runtime (Python), HMI (TypeScript) and deployment live in the same repository in their own top-level directories.
6. `work/` holds work packages and questions; `OWNERS.yaml` and `CODEOWNERS` are the ownership source of truth.
7. A thin `cmd/` layer; wiring only in `internal/daemon`.

## Alternatives rejected and why
- **Multi-repo (core, HMI, firmware, drivers, docs)**: contract changes would span repositories, and agents could not see one coherent diff. Third-party driver authors need only `pkg/`.
- **Go workspaces with several modules**: more build and versioning machinery than a single-person project needs; revisit when the SDK has external users (scripting area v3).
- **Everything under `internal/`**: would hide the driver SDK from third-party driver authors, contradicting the plugin model in [../02-drivers/framework.md](../02-drivers/framework.md).
- **`simulator/` as the home of all simulator Go code**: breaks existing references and splits `pkg/sim` from the `sim` driver that shares it.
- **Generated code not committed**: makes consumers depend on tool versions and hides contract drift from diffs.
- **A separate `interfaces/` package for seams**: see [01-agent-team.md](01-agent-team.md) alternatives.

## Open questions
1. Module path (`github.com/<org>/skyranger`) and the licence of the repository (decisions entry 5 defers licensing; affects `pkg/` stability promises).
2. Whether to move all simulator Go code under `simulator/` and update the docs, or keep the split chosen here.
3. Whether firmware should stay in this repository once it has its own release cadence and secure-boot keys (keys never in the repo, see `CLAUDE.md`).
4. Tool for the dependency-direction lint (`depguard`, `go-arch-lint` or a custom check), UNVERIFIED.
5. Whether `web/screens/` demo screens should instead live in `examples/` to keep `web/` purely code.
6. Python packaging for `scripting/` (single package versus one per sandbox component) is decided with the scripting area in M4.
7. Whether `OWNERS.yaml` globs should be derived from `CODEOWNERS` to keep one source of truth.
