# 09 Testing and QA

Status: design, v1. Binding inputs: [docs/00-contracts](../00-contracts/README.md) (tag model, messages, driver interface, API) and [docs/decisions.md](../decisions.md). Where this area found a gap or a likely bug in a contract, it is listed under Open questions in the relevant file. Nothing here silently deviates from a contract.

## Why this area matters more than usual

Sky Ranger's business case is "a cheaper, open system that is as trustworthy as Siemens or Rockwell". A customer cannot audit our code, but they can read evidence. The test strategy therefore has three jobs:

1. **Find bugs before customers do** (normal QA).
2. **Produce evidence** a skeptical plant manager or water-utility engineer can read: fault-by-fault behavior, soak graphs, power-cut counts, vulnerability scans. See [11-evidence-pack.md](11-evidence-pack.md).
3. **Fence in AI coding agents.** Much of the code is written by Claude Code sub-agents. The tests, golden files and conformance suites are the specification they cannot argue with, and CI enforces that they are not weakened. See [12-ai-agent-guardrails.md](12-ai-agent-guardrails.md).

## Files

| File | Contents |
|---|---|
| README.md (this) | Principles, test pyramid, layer definitions, repo layout, glossary |
| [01-unit-and-contract-tests.md](01-unit-and-contract-tests.md) | Unit test rules, clock injection, golden-file corpus for every contract, cross-language golden checks, property-based tests |
| [02-driver-conformance.md](02-driver-conformance.md) | The conformance suite every driver must pass (`skyranger/pkg/driver/conformance`), Sparkplug add-on suite, device-profile conformance |
| [03-simulator.md](03-simulator.md) | `skyranger-sim` design: many virtual devices, Sparkplug B, Modbus TCP, OPC UA, YAML scenarios, seeded determinism, time modes; the `sim` driver's `sim://` address details |
| [04-fault-injection.md](04-fault-injection.md) | The fault catalogue: every fault, how it is injected, the expected behavior in contract terms, and the tier it runs in |
| [05-record-replay.md](05-record-replay.md) | Capturing real traffic (wire and sample level), anonymizing, replaying as regression tests |
| [06-integration-and-e2e.md](06-integration-and-e2e.md) | Docker Compose integration stack, the in-process harness, HMI browser tests with Playwright |
| [07-soak-and-performance.md](07-soak-and-performance.md) | Soak tests (duration, measurements, pass criteria), 10k / 100k tag targets with reasoning, benchmarks |
| [08-hardware-and-power-cut.md](08-hardware-and-power-cut.md) | Hardware-in-the-loop rig, ESP32 with a ground-truth pulse generator, relay-driven power cuts, network chaos on real hardware |
| [09-security-testing.md](09-security-testing.md) | Fuzzing parsers, API fuzzing, authorization matrix, dependency and image scanning, SBOM |
| [10-ci-and-quality-gates.md](10-ci-and-quality-gates.md) | CI pipeline (commit / nightly / weekly / release), quality gate, "ships to a customer" acceptance checklist, SAT checklist |
| [11-evidence-pack.md](11-evidence-pack.md) | The customer-facing evidence pack: contents, format, how it is generated |
| [12-ai-agent-guardrails.md](12-ai-agent-guardrails.md) | How AI agents are kept honest: protected tests, red-green proof, test-integrity check, mutation testing, forbidden patterns |

## Scope coverage map

| Scope item | Where |
|---|---|
| Test pyramid, layers, repo layout | this file |
| Contract tests, golden files, property tests | 01 |
| Driver conformance suite | 02 |
| Device simulator | 03 |
| Fault-injection catalogue | 04 |
| Recorded capture and replay | 05 |
| Docker Compose integration, in-process harness, HMI browser tests | 06 |
| Soak, 10k and 100k tag performance and scale | 07 |
| Chaos and power-cut testing on real edge hardware, ESP32 HIL | 08 |
| Security testing (fuzzing, API, dependency and image scanning) | 09 |
| CI pipeline, quality gates, "ships to a customer" checklist, FAT and SAT | 10 |
| Customer-facing evidence pack and claims register | 11 |
| AI coding agent guardrails (protected tests, red-green, mutation testing) | 12 |

## Contract issues found by this area (consolidated; each is detailed in the file named)

These are reported, not fixed here. The contracts in docs/00-contracts are binding until a human changes them with a decisions.md entry.

1. `counter_total` and out-of-order samples with `modulus > 0` (01 Open question 1).
2. Non-atomic evaluation of `boot_count` and `pulse_count` after a rebirth (01 Open question 2).
3. NaN with a held last value: `value: null` or last value? (01 Open question 3).
4. Gateway clock stepping backward freezes live values (04 F-CLK-04).
5. After an `implausible` sample `last_raw` is set to the implausible value (04 F-CNT-03).
6. A `boot_count` NVS commit lost to a power cut makes a reboot look like a rollover (08 Open question 1).
7. Millisecond timestamp precision with `(tag_id, ts)` history key can collapse sub-millisecond samples (07 Open question 2).

## Principles

1. **Ground truth beats eyeballing.** Every scenario has an oracle that knows the true answer: the simulator knows how many pulses it emitted, the hardware rig's pulse generator knows how many pulses it sent. Tests compare the gateway's `total_volume` against that number, exactly.
2. **Faults are first-class test inputs.** Every fault in [04-fault-injection.md](04-fault-injection.md) has an ID, an injection method, an expected outcome written in contract terms (quality codes, events, connection states), and at least one automated test. A fault without a test is a gap on the dashboard.
3. **Determinism where possible, tolerance where not.** In-process tests run on a virtual clock with a seeded random source and must give the same result every run. Black-box tests (Docker, hardware) run in real time and assert with explicit, documented tolerances.
4. **Tests are the spec agents cannot edit.** Golden files, the conformance suite and the scenario library are protected paths. Changing them needs a human (see file 12).
5. **Boring tools.** Go `testing`, `testing/synctest` where it fits, Docker Compose, Toxiproxy, Playwright, `go test -fuzz`, govulncheck, Trivy. No custom test framework beyond thin helpers.
6. **The same tests produce the evidence.** Reports in the evidence pack are generated from CI artifacts, never written by hand.
7. **A flaky test is a bug in the product or the test, with a ticket.** Gate jobs never auto-retry. In SCADA the intermittent bugs are the ones that matter.

## Test pyramid

```
                         ┌────────────────────┐
  L7  Hardware / HIL     │ power cuts, ESP32, │   release + nightly subset, real hardware
                         │ real network chaos │
                       ┌─┴────────────────────┴─┐
  L6  Soak / perf      │ 6h nightly, 72h release│   dedicated runner / reference hardware
                       │ 10k / 100k tags        │
                     ┌─┴────────────────────────┴─┐
  L5  HMI e2e        │ Playwright vs compose stack │   smoke per PR, full nightly
                   ┌─┴────────────────────────────┴─┐
  L4  Integration  │ docker compose: skyrangerd,    │   smoke per PR, fault catalogue nightly
                   │ mosquitto, timescaledb, sim,   │
                   │ toxiproxy                      │
                 ┌─┴────────────────────────────────┴─┐
  L3  Component  │ in-process harness: real tag engine,│   every commit, virtual time,
                 │ drivers in-process, fake clock,     │   deterministic, seconds
                 │ scenario YAML, embedded test DB opt.│
               ┌─┴────────────────────────────────────┴─┐
  L2  Contract │ golden files (JSON, protobuf, YAML, WS │   every commit
               │ transcripts), buf breaking, OpenAPI    │
               │ diff, driver conformance (in-process)  │
             ┌─┴────────────────────────────────────────┴─┐
  L1  Unit   │ Go table tests, property tests (rapid),     │   every commit, < 2 min total
             │ fuzz seed corpus, Vitest for HMI            │
           ┌─┴────────────────────────────────────────────┴─┐
  L0  Static│ gofmt, go vet, golangci-lint, buf lint, tsc,   │   every commit, pre-commit hook
           │ eslint, JSON Schema validation of examples     │
           └────────────────────────────────────────────────┘
```

Target proportions and budgets (guidance, not law):

| Layer | Approx share of test count | Wall-time budget per run | Runs |
|---|---|---|---|
| L0 static | n/a | 1 min | pre-commit, every push |
| L1 unit | 70% | 2 min (with `-race`: 5 min) | pre-commit, every push |
| L2 contract | 10% | 2 min | every push |
| L3 component (harness) | 12% | 5 min | every push |
| L4 integration | 5% | smoke 5 min, full 60 min | smoke per PR, full nightly |
| L5 HMI e2e | 2% | smoke 5 min, full 30 min | smoke per PR, full nightly |
| L6 soak/perf | few, long | 6h nightly, 72h release | nightly, release |
| L7 hardware | few, long | 1h nightly subset, ~24h release | nightly, release |

The L3 harness is deliberately fat. Most SCADA bugs are timing bugs (stale detection, reconnect, rollover, alarm delays). On a virtual clock they are fast and deterministic, so we can afford hundreds of them. The water meter's 2 h leak alarm is tested in milliseconds there.

## Layer definitions

- **L1 unit**: one package, no network, no real clock, no database. Uses `clock.Fake` or `testing/synctest`.
- **L2 contract**: asserts that a serialized form (JSON, protobuf, YAML, topic string, SQL enum) matches the committed golden corpus, or that a schema has no breaking change versus `main`.
- **L3 component (harness)**: `test/harness` boots the real tag engine, alarm engine, historian writer (against an in-memory history sink, or a real PostgreSQL when `SKYRANGER_TEST_PG` is set), the API server on a loopback port, and drivers in-process via `driverhost.InProcess(d)`, all sharing one `clock.Fake`. The simulator feeds them through in-memory transports. Scenarios from `test/scenarios/` run here.
- **L4 integration**: real containers, real binaries, real broker and database, real time. Faults injected with Toxiproxy, `docker compose` commands, and `tc`/`iptables` in a privileged helper.
- **L5 e2e**: Playwright drives the real HMI in Chromium (and Firefox nightly) against the L4 stack.
- **L6 soak/perf**: L4 stack on fixed reference hardware for hours to days, metrics scraped and judged automatically.
- **L7 hardware**: real edge box, real ESP32, relay power cuts, real switch, ground-truth pulse generator.

## Proposed repository layout (testing parts only)

```
testdata/contracts/            golden corpus shared by Go, TypeScript and Python tests
  values/  events/  alarms/  sparkplug/  ws/  config/  topics/  quality_codes.json
internal/clock/                Clock interface, Real, Fake (used by all time-dependent code)
pkg/sim/                       generators, seeded RNG, scenario engine (shared by sim driver and skyranger-sim)
cmd/skyranger-sim/             multi-protocol device simulator binary
pkg/driver/conformance/        driver conformance suite (contract section 8)
test/harness/                  L3 in-process gateway harness
test/scenarios/                YAML scenarios (protected path)
test/faults/                   fault catalogue machine-readable index (faults.yaml)
test/integration/              L4 Go tests, build tag `integration`
test/compose/                  compose files and overrides
test/soak/  test/perf/         soak and perf runners and judges
test/hil/                      hardware rig controller (Python, runs on the rig controller Pi)
test/replay/                   recorded captures (public, synthetic or anonymized only)
hmi/e2e/                       Playwright tests
tools/testguard/               test-integrity checker for CI (agent guardrails)
```

## Glossary

- **Oracle**: the component that knows the correct answer for a test (simulator ground truth, pulse generator count, golden file).
- **Conservation invariant**: for a pulse meter, `total_volume(end) - total_volume(start) = (pulses_emitted - pulses_lost_to_reboots) * liters_per_pulse`, where `pulses_lost_to_reboots` is computed by the simulator from the contract rule in tag-model.md section 7 (counts between the last delivered sample of an epoch and the reboot). Asserted exactly, not approximately.
- **FAT / SAT**: Factory Acceptance Test (before shipping, on our bench) and Site Acceptance Test (at the customer). Terms SCADA customers already use; the checklists in file 10 are named that way on purpose.
- **Reference hardware**: the specific edge-box SKUs and storage devices the evidence pack is valid for (file 07, file 08).

## Decisions made
1. Eight layers L0 to L7, with a deliberately large L3 in-process harness on a virtual clock so timing-heavy behavior is tested deterministically and fast.
2. Every test of measured values uses an oracle with exact ground truth; the conservation invariant is the headline correctness check for the water meter.
3. Test artifacts (golden corpus, conformance suite, scenarios, fault catalogue) are protected paths; changing them needs human review.
4. Evidence-pack reports are generated from CI output only.
5. Gate jobs do not auto-retry failed tests.

## Alternatives rejected and why
- **Mostly end-to-end testing through the HMI**: slow, flaky, and poor at timing faults. Kept small at the top of the pyramid.
- **Testing timing in real time with sleeps**: makes a 2 h leak alarm untestable per commit and creates flakes. Replaced by virtual time in L1 and L3.
- **A custom BDD framework (Gherkin etc.)**: one more language for agents and humans to get wrong. YAML scenarios plus Go tests cover the need.
- **Separate test repos**: tests drift from code. Everything lives in the main repo except customer captures (file 05).

## Open questions
- Reference hardware SKUs are not chosen yet (deployment doc). Performance and power-cut numbers are only meaningful once they are.
- CI host is assumed to be GitHub Actions plus one self-hosted lab runner; UNVERIFIED that the repo will live on GitHub. The pipeline is written as `make` targets so the CI system is swappable.
- Whether L3 harness runs against real PostgreSQL by default (more realistic, needs Docker in the unit-test job) or an in-memory sink (faster). Proposal: in-memory per commit, PostgreSQL variant of the same scenarios nightly.
