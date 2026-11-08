# 01 Unit and Contract Tests (L1, L2)

Covers L1 unit tests and L2 contract tests. The goal of L2 is simple: **every name, field, topic, enum value and quality code in `docs/00-contracts` exists exactly once as machine-checked data, and every implementation (Go core, drivers, HMI TypeScript, Python scripting) is tested against that same data.** An agent that renames `bad_not_connected` to `bad_disconnected` breaks a golden test on the first commit.

## 1. Unit test rules (Go)

| Rule | Why | Enforced by |
|---|---|---|
| Table-driven tests with `t.Run(name, ...)` | Readable, easy for agents to extend | review |
| No real clock. All time-dependent code takes a `clock.Clock` (or is tested inside `testing/synctest`) | Stale detection, rates, alarm delays, backoff, checkpoint intervals are all timing | `forbidigo` lint rule banning `time.Now`, `time.Sleep`, `time.After`, `time.NewTimer`, `time.Tick` outside `internal/clock` and `main` packages |
| No network, no filesystem outside `t.TempDir()`, no database | Speed and determinism | build tag `integration` required for anything else |
| Seeded randomness only: `rand.New(rand.NewPCG(seed, seq))`, seed logged on failure | Reproducible failures | lint bans global `math/rand` functions in non-test and test code |
| `-race` on every CI run | Concurrency bugs in the tag engine and driver SDK coalescer are likely | CI |
| `-shuffle=on` on every CI run | Catches test-order dependencies | CI |
| `t.Parallel()` by default in pure unit tests | Speed; surfaces shared state | review, `paralleltest` lint (optional) |
| No `t.Skip` without an issue reference in the message | Skips are where coverage quietly dies | `testguard` (file 12) |
| Assertions: standard library plus `github.com/google/go-cmp/cmp` for diffs | Boring, no assertion DSL to learn | review |

### Clock

```go
// internal/clock/clock.go
package clock

type Clock interface {
    Now() time.Time                         // wall clock, UTC
    Since(t time.Time) time.Duration        // uses monotonic reading when available
    NewTimer(d time.Duration) Timer
    NewTicker(d time.Duration) Ticker
    AfterFunc(d time.Duration, f func()) Timer
}

// Fake is deterministic: time only moves when the test calls Advance or Set.
// Set can move wall time backwards or forwards WITHOUT moving monotonic time,
// which is how clock-jump faults are injected (04-fault-injection.md F-CLK-*).
type Fake struct { /* ... */ }
func NewFake(start time.Time) *Fake
func (f *Fake) Advance(d time.Duration)          // fires due timers in order
func (f *Fake) SetWall(t time.Time)              // wall jump, monotonic unchanged
func (f *Fake) BlockUntilWaiters(n int)          // sync helper for goroutine tests
```

`testing/synctest` (bubble with a fake clock for real `time` calls) is an alternative for code that is hard to thread a `Clock` through. My understanding is that it became generally available in Go 1.25 (UNVERIFIED: confirm version and API, `synctest.Test`, before relying on it). Rule: new code uses `clock.Clock`; `synctest` is allowed for testing third-party code paths (for example an MQTT client's keepalive) where injection is impossible. `synctest` cannot simulate a wall-clock jump separate from monotonic time, which is one reason `clock.Fake` exists.

### What must have unit tests (minimum list)

| Package (proposed) | Must-test behaviors |
|---|---|
| `pkg/tag` | path grammar (accept/reject table from golden file), prefix matching (`site1/pump` vs `site1/pumphouse`), Variant conversions and widening, NaN/Inf handling, quality class, worst-quality propagation and tie rule |
| `pkg/tag/derived` | each built-in: `counter_total` (every branch of the normative algorithm, each discontinuity kind, `max_delta`, write), `rate` (warm-up, decay with `eval_interval`, window edges), `liveness` (all three outputs, bad online input), `continuous_activity` (gap reset, `active_since`), `select`, `scale` |
| `pkg/tag/engine` | `ts` selection (`gateway`, `device`, `auto` with skew and pre-2020 rejection), deadband (absolute, percent, quality change always propagates), `stale_after`, late samples are history-only, hot reload keeps state |
| `pkg/config` | schema validation, template substitution (whole-scalar typed, embedded text, `$${` escape, unknown `${x}` fails), reference resolution, dangling reference fails, path collisions, export then import gives empty diff |
| `pkg/alarm` | state machine transitions, `on_delay`/`off_delay`, deadband clear, quality alarms, shelving orthogonal to state |
| `pkg/driver/sdk` | coalescer: latest-wins, quality transitions never dropped, memory bounded by item count, `dropped_samples` counting |
| `internal/sparkplug` | topic parse/format, seq tracking with wrap at 255, alias table, birth/death pairing by `bdSeq`, rebirth rate limit (one per node per 30 s) |
| `pkg/api` | problem details shape, cursor encoding, int64-as-string encoding, write value decoding (exact integer within ±2^53 accepted as number) |
| `hmi/src` (Vitest) | value object decoding (int64 strings, `null` values), quality to visual state mapping, WS client `seq` gap detection and resync handling, unknown enum tolerance |

## 2. Golden-file corpus

All contract artifacts live in `testdata/contracts/`, one directory per contract surface. Files are **plain, human-reviewable JSON/YAML/hex**, never generated binary blobs only.

```
testdata/contracts/
  quality_codes.json            # every code: {code, name, class}
  identifiers.yaml              # accept/reject lists for segment grammar, tag paths, system paths
  datatypes.json                # six types, JSON wire examples, proto field names
  values/
    good_float.json             # canonical value object, all seven fields
    int64_as_string.json
    uint64_max.json             # "18446744073709551615"
    null_waiting_for_initial.json
    bad_with_last_value.json
    no_ts_source.json
  events/
    counter_discontinuity_reset.json
    counter_discontinuity_rollover.json
    counter_discontinuity_implausible.json
    counter_discontinuity_manual_adjust.json
    write_completed_ok.json   write_completed_timeout.json
    alarm_state_changed.json  connection_state_changed.json
    config_revision_applied.json
    discovery_device_announced.json  discovery_device_changed.json  discovery_status_changed.json
  alarms/   alarm_state_active_unacked.json ...
  topics.yaml                   # tag path -> internal bus topic, Sparkplug topics parse table
  addresses.yaml                # spb://, modbus://, opcua:, sim:// accept/reject + parsed fields
  sparkplug/
    pulse_meter_nbirth.hex      # protobuf bytes, hex, one line per 32 bytes
    pulse_meter_nbirth.json     # same message decoded (protojson), human-readable
    pulse_meter_ndata_heartbeat.{hex,json}
    pulse_meter_ndeath.{hex,json}
    rebirth_ncmd.{hex,json}
    state_online.json           # host STATE payload (UNVERIFIED field names, see messages.md)
  driver/
    sample_batch.{binpb.hex,json}
    health_response.json
  ws/
    subscribe_snapshot_update.jsonl    # transcript: one message per line, direction prefix
    overflow_resync.jsonl
    hello_bad_token.jsonl
  api/
    problem_validation_failed.json
    tag_definition_total_volume.json
    history_delta_1h.json
  config/
    pulse_water_meter_template.yaml    # verbatim from tag-model.md section 11
    pumphouse_taggroup.yaml
    expanded_tags_meter01.json         # expected resolved tags after template expansion
  scenarios/
    water_meter_trace.yaml             # the trace table in tag-model.md section 11 as data
```

### How golden tests work

1. A Go test marshals a fixture built in code and compares to the file byte for byte (after canonical formatting: 2-space JSON indent, sorted keys only where the contract has no field order).
2. The same test unmarshals the file and compares to the fixture (round trip both ways).
3. **Unknown-field tolerance**: each JSON golden has a `*.plus_unknown.json` sibling generated by the test harness (adds `"zz_future_field": 1` at every object level). Every consumer must decode it identically. This tests the "MUST ignore unknown fields" rule in the contracts README.
4. Update only via `go test ./... -run Golden -update` (`make golden-update`). CI never runs with `-update`. Any diff under `testdata/contracts/` is a protected change (file 12).

```go
// pkg/tag/golden_test.go
var update = flag.Bool("update", false, "rewrite golden files")

func TestGoldenQualityCodes(t *testing.T) {
    type row struct {
        Code  uint16 `json:"code"`
        Name  string `json:"name"`
        Class uint16 `json:"class"`
    }
    var got []row
    for _, q := range tag.AllQualities() { // single source list in Go
        got = append(got, row{uint16(q), q.String(), q.Class()})
    }
    golden.JSON(t, "testdata/contracts/quality_codes.json", got, *update)
}

func TestGoldenTagPaths(t *testing.T) {
    var cases struct {
        Accept []string `yaml:"accept"`
        Reject []struct{ Path, Reason string } `yaml:"reject"`
    }
    golden.LoadYAML(t, "testdata/contracts/identifiers.yaml", &cases)
    for _, p := range cases.Accept {
        if err := tag.ValidatePath(p); err != nil {
            t.Errorf("ValidatePath(%q) = %v, want ok", p, err)
        }
    }
    for _, c := range cases.Reject {
        if tag.ValidatePath(c.Path) == nil {
            t.Errorf("ValidatePath(%q) = ok, want reject (%s)", c.Path, c.Reason)
        }
    }
}
```

Example `identifiers.yaml` excerpt (values from tag-model.md section 1):

```yaml
accept:
  - site1/pumphouse/meter01/flow_rate
  - a
  - 0abc/x-y_z
  - _system/connections/local-sparkplug/state   # system paths: accepted by ValidateSystemPath only
reject:
  - { path: "Site1/pump",          reason: uppercase }
  - { path: "/site1/pump",         reason: leading slash }
  - { path: "site1/pump/",         reason: trailing slash }
  - { path: "site1/pump.speed",    reason: dot }
  - { path: "site1/+/x",           reason: mqtt wildcard }
  - { path: "site1/#",             reason: mqtt wildcard }
  - { path: "_private/x",          reason: leading underscore outside _system }
  - { path: "a/b/c/d/e/f/g/h/i/j/k/l/m/n/o/p/q", reason: 17 segments }
  - { path: "-abc",                reason: segment starts with hyphen }
```

### Cross-language consumption

The same corpus is read by:
- **HMI (Vitest)**: `hmi/src/contract.test.ts` loads `testdata/contracts/values/*.json`, `ws/*.jsonl`, `quality_codes.json` and asserts the TypeScript decoders and quality-class mapping agree. The TS quality enum is **generated** from `quality_codes.json` (`make gen`), and a test fails if the generated file is stale.
- **Python scripting runtime (pytest)**: value objects, events, quality codes.
- **ESP32 firmware (host-side unit tests, Unity or plain C on Linux)**: encodes its NBIRTH and heartbeat with the real nanopb encoder and compares the decoded result to `sparkplug/pulse_meter_nbirth.json` (field-level compare, because protobuf byte output may legitimately differ in field order; UNVERIFIED whether nanopb and Go protobuf produce byte-identical output for this message, so byte compare is not required).

### Schema-level breaking-change checks

| Surface | Tool | Check |
|---|---|---|
| gRPC driver proto, Sparkplug proto (vendored Tahu file) | `buf lint`, `buf breaking --against '.git#branch=main'` | as required by driver-interface.md section 7 |
| REST | OpenAPI 3.1 at `api/openapi.yaml`, diffed with `oasdiff breaking` (UNVERIFIED: oasdiff 3.1 support level; fallback is a golden of the normalized spec plus human review) | no removed paths/fields/enum values within `/api/v1` |
| Config | JSON Schema per kind in `schemas/skyranger-v1/*.json`; every YAML example in `docs/` and `testdata/` is validated against it in CI | doc examples cannot drift from the schema |
| Database enums | quality `smallint` codes in migration seed vs `quality_codes.json` | test compares |
| Internal bus topics | `topics.yaml` | parse/format table |

**Docs-as-tests**: a CI job extracts fenced ```yaml and ```json blocks from `docs/00-contracts/*.md` that carry a `apiVersion: skyranger/v1` or are value objects, and validates them against schemas. Because agents read the contract docs, broken examples there are as dangerous as broken code.

## 3. Property-based tests

Library: `pgregory.net/rapid` (pure Go, shrinking, works with `go test`). Properties are the strongest guard against an agent writing code that passes the example tests and nothing else.

| Property | Generator | Invariant |
|---|---|---|
| **Counter conservation** | random sequence of: pulses (0..1000), reboots, MQTT reconnects without reboot, rollovers (start near 2^32), duplicated samples, dropped (QoS 0 lost) samples | `total_after - initial_total == (true_pulses - pulses_lost_to_reboots) * multiplier`, where the oracle computes lost pulses by the contract rule; and `total` never decreases except on `manual_adjust` |
| Counter: never over-counts | same plus out-of-order delivery | `total <= initial_total + true_pulses * multiplier` (see Open question 1: this currently fails on the contract algorithm for out-of-order input) |
| Quality propagation | random input qualities for a derived tag | output class >= worst input class; never better |
| Path round trip | random valid paths | `topic(path)` parses back to `path`; prefix match agrees with a naive segment-split implementation |
| JSON value round trip | random Variants of all six types, incl. int64 extremes | marshal then unmarshal is identity; int64/uint64 are JSON strings |
| Deadband | random value walk | every propagated value differs from the previous propagated value by more than the deadband, or has a different quality |
| Coalescer | random interleaving of samples and sink blocking | per handle: last delivered sample equals last produced; every quality transition is delivered in order; memory bounded |
| WS `seq` | random update stream with random overflow | between snapshots, `seq` is contiguous starting at 1 |
| Config export/import | random valid config sets | export, import, export gives identical bytes and an empty diff |
| Template substitution | random param values of each type | whole-scalar substitution preserves type; `$${` survives as literal `${` |

Sketch:

```go
func TestCounterTotalConservation(t *testing.T) {
    rapid.Check(t, func(t *rapid.T) {
        dev := simmeter.New(rapid.Uint32().Draw(t, "start_raw")) // oracle device model from pkg/sim
        ct := derived.NewCounterTotal(derived.CounterParams{
            Modulus: 1 << 32, Multiplier: 1, MaxDelta: 100000, ResetBaseline: derived.Zero,
        })
        steps := rapid.SliceOfN(stepGen(), 1, 500).Draw(t, "steps")
        for _, s := range steps {
            for _, sample := range dev.Apply(s) {       // device emits what the gateway would receive
                ct.OnSample(sample.Raw, sample.Epoch, tag.QualityGood)
            }
        }
        want := float64(dev.TruePulses() - dev.PulsesLostToReboots())
        if got := ct.Total(); got != want {
            t.Fatalf("total=%v want=%v (lost=%d)", got, want, dev.PulsesLostToReboots())
        }
    })
}
```

`simmeter` is the same device model the simulator uses (file 03), so the oracle is written once.

## 4. Fuzz seed corpus

Every parser has a `FuzzXxx` with the golden files as seeds (see file 09 for the fuzzing campaign). The seed corpus runs as normal unit tests on every commit (Go runs `testdata/fuzz/...` seeds in plain `go test`).

## 5. Coverage

- Collected per package with `go test -coverprofile`, plus `-coverpkg` for the L3 harness so integration-style coverage of the engine counts.
- Floors (enforced, per package): `pkg/tag/derived` 95% statements, `pkg/tag/engine` 90%, `pkg/alarm` 90%, `pkg/driver/sdk` 90%, `internal/sparkplug` 90%, `pkg/config` 85%, everything else 70%. Floors may only rise (ratchet file `test/coverage-floors.yaml`, protected).
- Diff coverage for PRs: new and changed lines at least 80% covered.
- Coverage is necessary but not trusted on its own. Mutation testing (file 12) checks the assertions are real.

## Decisions made
1. One golden corpus in `testdata/contracts/`, consumed by Go, TypeScript, Python and firmware host tests; generated enums (TS quality codes) are checked for staleness.
2. All time-dependent code uses an injected `clock.Clock`; a lint rule bans direct `time.Now`/`time.Sleep` outside `internal/clock`.
3. Property-based tests with `pgregory.net/rapid`, with counter conservation as the headline property and the simulator's device model as the oracle.
4. Breaking-change detection: `buf breaking` for protos, OpenAPI diff for REST, JSON Schema validation of every YAML example in the docs.
5. Per-package coverage floors that only ratchet upward, plus 80% diff coverage.

## Alternatives rejected and why
- **Snapshot testing libraries that auto-accept changes** (for example ones that write snapshots on first run silently): makes it trivial for an agent to "fix" a contract break by re-snapshotting. Our golden update is an explicit flag and a protected path.
- **testify/assert**: fine library, but `cmp.Diff` plus `t.Errorf` is enough and keeps one less dependency; not a strong objection.
- **gopter** for property tests: older, more verbose; rapid has better shrinking ergonomics.
- **Only `testing/synctest`, no Clock interface**: cannot express wall-clock jumps independent of monotonic time, and its availability/API needs verification.
- **A single global coverage threshold**: lets well-tested trivial code hide untested critical code.

## Open questions
1. **Contract issue, counter_total and out-of-order samples.** With `modulus > 0`, a sample whose raw value is lower than `last_raw` (same epoch) is treated as a rollover; the huge delta trips `max_delta` (`implausible`, delta 0) and then `last_raw` is set to the lower value, so the next in-order sample re-counts the difference. Example: raw 90, then 110, 100 (late), 120 gives +20, +0, +20 = 40 instead of 30. tag-model.md section 5 says older-`ts` samples are history-only, but does not say whether they trigger derived evaluation. Proposal for the contract owner: samples that do not advance an input's current value never trigger `counter_total`. The "never over-counts" property test is written to that proposal and will fail until the contract decides.
2. **Contract issue, atomic multi-input evaluation.** An NBIRTH after reboot changes both `boot_count` and `pulse_count`. If the engine evaluates `counter_total` after applying `boot_count` but before `pulse_count`, it computes `delta = 60` from the stale count (over-count), and the following `pulse_count = 3` then looks like a wrap and is flagged `implausible`. In the other order the total ends right but a spurious `rollover` plus `implausible` event pair is stored, which pollutes the audit trail. The contract should state that all samples in one `SampleBatch` with the same `ts_source` (or one Sparkplug message) update inputs atomically before derived evaluation. A dedicated test (`TestNBirthAtomicEpochAndCount`, both orders within a batch) is specified regardless.
3. tag-model.md section 3 says NaN gives `value: null`, while the same section says bad samples carry the last known value. Golden file `values/` needs one answer for "NaN arrives while a last value is held".
4. Whether nanopb and Go protobuf produce identical bytes for the pulse meter messages (affects whether firmware golden tests can be byte-exact).
