# 07 Soak and Performance Tests (L6)

Two different questions, two different test families:

- **Soak**: does it keep working, with no slow degradation, for days, while faults keep happening? (Memory leaks, goroutine leaks, file descriptor leaks, unbounded queues, disk growth, DB maintenance jobs, clock-related surprises.)
- **Performance and scale**: how much load can it carry, where does it break, and what limit do we promise customers?

Both run on **reference hardware** (a fixed edge-box SKU and SSD; not chosen yet, see Open questions) and nowhere else counts as evidence. A shared CI runner produces noisy numbers that must not appear in the evidence pack. They may be used for trend alarms only.

Every threshold in this file is a **proposal calibrated after the first three baseline runs** on the reference box. Until then they are UNVERIFIED design targets, and the first runs record, not gate (see section 5 "Calibration").

## 1. Soak tests

### 1.1 Durations and cadence

| Run | Duration | Where | Trigger | Purpose |
|---|---|---|---|---|
| Nightly soak | 6 h | reference box (lab runner) | every night on `main` | catches leaks and regressions the same day they are introduced |
| Weekly soak | 24 h | reference box | weekly | a full day cycle: midnight jobs, daily aggregation buckets, retention and compression policy runs |
| **Release soak** | **72 h** | reference box, **the exact release-candidate image digests** | each release candidate | headline evidence; must pass with zero re-runs |
| Pre-first-customer soak | 14 d | reference box | once, and again after any major storage change | slow leaks (below roughly 1 MiB/h) are invisible in 72 h |
| Customer FAT soak | 24 h | our bench, with the customer's real config and the sim | before shipping each customer | "your configuration, on your hardware class, ran a day clean" |

Why 72 h for release: it spans three midnights (every daily job), three compression/retention cycles in the test profile (below), and gives a leak detector at 1 MiB/h about 72 MiB of signal. It is honest to say what it cannot show: leaks below that rate, and anything that needs weeks (certificate expiry, SSD wear). Those are covered by the 14-day run, by design rules (bounded queues, tested in unit and fault tests), and by monitoring in the field. The evidence pack states this (file 11 "limits").

A **gate job never auto-retries** a failed soak. A failed soak is a bug in the product or the test, with a ticket (README principle 7).

### 1.2 The soak plant (`test/scenarios/soak-plant.yaml`)

A mixed, boring plant that exercises every driver and the usual API traffic.

```yaml
apiVersion: skyranger-test/v1
kind: Scenario
metadata:
  name: soak-plant
  description: "~6,700 tags: 500 pulse meters, 20 Modbus PLCs, 1 OPC UA server, realistic API clients, random self-recovering faults"
  tags: [soak, l6]
  faults_covered: [F-NET-01, F-NET-02, F-NET-05, F-NET-06, F-DEV-01, F-DEV-02, F-DEV-03, F-BRK-01, F-DB-01, F-GW-04]
spec:
  seed: 20261109                 # logged; the chaos schedule is derived from it
  duration: 72h
  time: { mode: real }           # soak never uses scaled time
  devices:
    - id: meters
      model: pulse_meter
      count: 500                 # 9 tags each from the template = 4,500 tags
      protocol: { sparkplug: { broker: local, group_id: skyranger, edge_node_id: "esp32-s{index:03d}" } }
      params:
        heartbeat_interval: 60s
        flow: { generator: random_walk, mean_lpm: 6, sigma: 3, min: 0, max: 40 }
        start_jitter: 120s
        random_reboot: { mean_interval: 6h }        # background reboots, oracle-accounted
    - id: plcs
      model: modbus_plc
      count: 20
      protocol: { modbus_tcp: { listen_base: ":5100", unit_ids: [1] } }
      points_template: { holding_float32: 100, input_int16: 5 }   # 2,100 tags polled at 1 s
    - id: opc
      model: opcua_server
      protocol: { opcua: { listen: ":4840", security: none } }
      points_template: { double_random_walk: 100 }                # 100 tags (when the OPC UA driver exists)
  clients:                        # realistic load beside the field data
    - { kind: ws,   count: 10, subscribe: "site1/#", hold: forever }
    - { kind: rest, count: 2,  poll: { path: "/api/v1/tags/values?prefix=site1/area01", every: 2s } }
    - { kind: rest, count: 1,  history_query: { tags: 20, range: 24h, every: 1m } }
    - { kind: rest, count: 1,  config_apply: { change: noop, every: 30m } }
    - { kind: rest, count: 1,  config_apply: { change: add_remove_one_tag, every: 6h } }
    - { kind: rest, count: 1,  alarm_ack: { every: 1h } }
  chaos:
    scheduler: poisson
    mean_interval: 20m
    cool_down: 3m
    select: { soak_safe: true }   # from faults.yaml (file 04 section 5)
  storage_profile: soak-lifecycle # section 1.3
  expect:
    - invariant: conservation_all_meters              # exact, every meter, vs oracle
    - invariant: no_history_gaps_longer_than
      params: { tag_prefix: site1, max: 16m, excluding: fault_windows }
    - judge: test/soak/judge.yaml                      # section 1.4
```

With a 20-minute mean fault interval, a 72 h run injects about 216 faults, seeded and logged. The same seed reproduces the same schedule (device-side and fault-side; network timing is not deterministic, file 03).

### 1.3 Storage lifecycle in a 72 h run

Default retention is `400d` and chunk compression normally acts on old chunks (TimescaleDB default chunk interval is 7 days, UNVERIFIED for the version we ship), so a 72 h run would never exercise compression or retention, which are exactly where long-run failures hide. The soak runs with a **storage profile** that shortens only storage lifecycle knobs:

```yaml
# test/compose/profiles/soak-lifecycle.yaml (applied before the run)
# Production: chunk interval 7d, compress after 7d, retention 400d  (historian doc owns the real values; these are placeholders)
# Soak:       chunk interval 1h, compress after 2h, retention 36h
```

This is legitimate because it changes how often maintenance jobs run, not what timing the gateway claims. The judge then asserts: retention dropped chunks older than 36 h, compression ran and queries spanning compressed and uncompressed chunks return values equal to the oracle, and disk use plateaus (it must stop growing once retention is in steady state; a model predicts the plateau within 20%).

### 1.4 Measurements

The soak judge reads two sources, both public: the Prometheus metrics endpoint (port `9100`, 08-deployment/02-containers.md) and the public API (`_system` tags, `/api/v1/system/...`). It does not read internals. Standard Go/process metric names below come from the Prometheus Go client defaults; `skyranger_*` names are **proposals** that the observability doc (08-deployment/10-observability) must provide, UNVERIFIED until it exists.

| Measure | Source | Scrape | Why |
|---|---|---|---|
| RSS | `process_resident_memory_bytes` | 15 s | leak |
| Go heap in use after GC | `go_memstats_heap_inuse_bytes`, `go_memstats_heap_idle_bytes` | 15 s | leak versus fragmentation |
| Goroutines | `go_goroutines` | 15 s | leak (driver reconnect paths) |
| Open file descriptors | `process_open_fds` | 15 s | leak (sockets) |
| GC pause p99 | `go_gc_duration_seconds` | 15 s | latency spikes |
| Ingest-to-WS latency, p50/p99/max | measured by the soak's own WS client on the sim host, comparing the sim's emit timestamp (oracle) with receipt time on one clock | 1 min windows | the number customers feel |
| Samples ingested, dropped | `skyranger_samples_total`, `skyranger_samples_dropped_total` (proposed) | 15 s | silent loss |
| Historian write queue depth, WAL backlog | proposed `skyranger_history_queue_depth`, `skyranger_history_wal_bytes` | 15 s | must drain after DB faults |
| DB size, row count per hypertable, chunk count | SQL via the sidecar judge account (read-only), the only place the judge touches the DB, restricted to size metrics | 5 min | growth model |
| History query latency p95 | soak client | per query | degradation over time |
| CPU, per container | cgroup stats via cAdvisor (UNVERIFIED tool choice) or `docker stats` | 15 s | headroom |
| Disk write bytes/day, SSD SMART wear indicators, temperature | `smartctl -a` (smartmontools), node exporter | 5 min | write amplification and thermal limits; **reported**, not gated in v1 |
| CPU temperature, throttling | `/sys` thermal zones, `vcgencmd` not applicable on x86 | 1 min | throttling hides as latency |
| Counts: restarts, reconnects, events by type, alarms by name | public API and events | 1 min | expected-vs-actual |
| Log error lines | container logs, counted by level | continuous | new error classes |

### 1.5 Pass criteria (release soak; nightly/weekly use the same rules over their duration)

Warm-up: the first 2 h (or 10% of the run, whichever is smaller) is excluded from slope calculations but not from correctness checks.

| # | Criterion | Rule | Threshold (proposed) |
|---|---|---|---|
| S-1 | **Conservation** | Every meter: final `total_volume` equals oracle exactly, using the tap-validated oracle (file 05 section 2.2). A run with `records_dropped > 0` in the tap is inconclusive, not pass | exact, zero tolerance |
| S-2 | Memory slope | Theil-Sen slope of hourly median RSS after warm-up | < 1 MiB/h **and** final-hour median within 5% of hour-2 median |
| S-3 | Goroutines and FDs | final-hour max versus hour-2 max | <= 1.1x + 50 |
| S-4 | No unexplained restarts | `skyrangerd` and driver restarts equal injected kills | exact |
| S-5 | Recovery times | Each injected fault returns affected tags to `good` within the bound in its catalogue entry (for example F-BRK-01) | per catalogue |
| S-6 | No steady-state degradation | Fraction of tag-seconds with non-good quality outside fault windows plus recovery bound | 0 |
| S-7 | History completeness | No gap longer than `max_interval` + slack outside fault windows; gaps inside fault windows exactly where the oracle says data could not arrive | per tag |
| S-8 | Alarms | No false activations; activations match the scenario's expected leak count; every alarm raised during a fault clears after it | exact |
| S-9 | Latency stability | ingest-to-WS p99 in the last 6 h versus first 6 h after warm-up | within +20%, and below the absolute bound in section 2.3 |
| S-10 | Historian drains | After each DB or broker fault, queue depth and WAL backlog return below 5% of the run's maximum within 5 min of recovery | per fault |
| S-11 | Storage lifecycle | retention, compression, and plateau checks from 1.3 | as stated |
| S-12 | Log hygiene | No `panic`, no `fatal`, no ERROR lines outside an allow-list keyed by fault ID (`test/soak/expected-errors.yaml`, protected) | zero unexpected |
| S-13 | Data integrity | After the run, `pg_amcheck` or equivalent index/heap check (PostgreSQL 14+ ships `pg_amcheck`; UNVERIFIED for our version) passes; config export/import round trip gives an empty diff | pass |
| S-14 | Dropped samples | `dropped_samples` only inside burst fault windows; never for counter tags (cumulative, coalescing allowed, loss is not) | per rule |

Judge sketch:

```go
// test/soak/judge/slope.go
// TheilSen returns the median of pairwise slopes; robust to the occasional outlier hour.
func TheilSen(xs, ys []float64) float64 {
    var slopes []float64
    for i := 0; i < len(xs); i++ {
        for j := i + 1; j < len(xs); j++ {
            if xs[j] != xs[i] { slopes = append(slopes, (ys[j]-ys[i])/(xs[j]-xs[i])) }
        }
    }
    sort.Float64s(slopes)
    return slopes[len(slopes)/2]
}

func TestJudge_RSSSlope(t *testing.T) {
    series := soak.LoadMetric(t, runDir, "process_resident_memory_bytes")
    hourly := soak.HourlyMedian(series.After(2 * time.Hour))
    slopeBytesPerHour := TheilSen(hours(hourly), values(hourly))
    if slopeBytesPerHour > 1<<20 {
        t.Errorf("RSS slope %.0f B/h exceeds 1 MiB/h", slopeBytesPerHour)
    }
    if growth := hourly.Last()/hourly.First() - 1; growth > 0.05 {
        t.Errorf("RSS grew %.1f%% (limit 5%%)", growth*100)
    }
}
```

The judge is a separate Go program (`tools/soakjudge`) with its own unit tests (synthetic series with a planted leak must fail; flat series must pass). A judge that cannot fail is not a judge.

### 1.6 Soak outputs

`soak-report.json` (every criterion, value, threshold, verdict), `soak-metrics.parquet`-or-JSONL of all series (format choice open), PNG graphs for RSS, goroutines, FDs, latency percentiles, queue depth and DB size with the fault schedule as vertical markers and the pass thresholds as dashed lines, the seed, image digests, hardware identification (SKU, kernel, SSD model and firmware). All feed the evidence pack (file 11). Graphs are generated from data, never hand-drawn.

## 2. Performance and scale tests

### 2.1 Why 10,000 and 100,000, and what they mean

| Target | Meaning | Status |
|---|---|---|
| < 500 tags | Milestone 1 and typical small water sites | must pass everything |
| **10,000 tags** | The "recommended" upper range for one gateway on one edge box (08-deployment/03 sizes hardware "up to ~10,000 tags at 1 s") | **supported claim**, backed by evidence |
| **100,000 tags** | Scale-validation target. Shows the architecture has no design cliff and gives a number for where the limit is | **not a promise** until measured; supported only at stated rates on stated hardware, otherwise "use several gateways" |

Reasoning, from the design rather than hope:

- **One engine goroutine owns all tag state** (01-tag-engine README: single writer). Throughput is bounded by per-sample CPU cost on one core: at 100,000 samples/s the budget is 10 µs per sample for decode, scale, deadband, derived evaluation, alarm check, publish to history queue and WS fan-out. That is plausible for Go but not comfortable once derived tags multiply work (a pulse meter's raw count feeds `counter_total`, then `rate`, `continuous_activity`, `liveness`: roughly 4 evaluations per raw sample). So the 100k stress case is the interesting one: it probes the single-writer design.
- **Typical load is far below stress load.** Tags rarely change every second. The deployment sizing assumes ~1 sample per 10 s per tag. Test both.
- **MQTT broker**: Mosquitto's core is single-threaded to our knowledge (UNVERIFIED for the shipped version); its limit may arrive before the engine's. The 10,000-connection sim (file 03 section 8) matters here.
- **History**: the historian batches into TimescaleDB with a WAL in front (file 04 F-DB-*). Row rate = propagated sample rate (default `on_change`). PostgreSQL ingest of tens of thousands of rows/s with batched `COPY` on an SSD is commonly reported feasible (UNVERIFIED for our schema and box; this is exactly what the test measures).
- **Storage maths** (using 08-deployment/03's numbers: about 50 B/row uncompressed, about 10 to 17x compression):

| Workload | Tags | Change rate | Samples/s into engine | History rows/day | Uncompressed/day | Compressed/day (est.) | 400 d retention (est.) |
|---|---|---|---|---|---|---|---|
| W10-typ | 10,000 | 1 per 10 s per tag | 1,000 | 86.4 M | 4.3 GB | 250 to 430 MB | 100 to 170 GB |
| W10-stress | 10,000 | 1 per 1 s per tag | 10,000 | 864 M | 43 GB | 2.5 to 4.3 GB | 1.0 to 1.7 TB |
| W100-typ | 100,000 | 1 per 20 s per tag | 5,000 | 432 M | 21.6 GB | 1.3 to 2.2 GB | 520 to 860 GB |
| W100-stress | 100,000 | 1 per 1 s per tag | 100,000 | 8.6 B | 432 GB | 25 to 43 GB | not supportable at 400 d |

Rows/day = samples/s x 86,400. These are estimates from stated assumptions, to be replaced by measured values. Two consequences already visible: (1) 100k tags at default retention needs terabytes and different disks than the hardware doc's 256 GB recommendation; (2) deadband and `sampled` history modes are the main lever for scale, so the perf tests run with and without them.

### 2.2 Workloads

```yaml
# test/scenarios/perf-10k.yaml (abridged; perf-100k.yaml scales counts)
apiVersion: skyranger-test/v1
kind: Scenario
metadata: { name: perf-10k, tags: [perf, l6], faults_covered: [F-VAL-06] }
spec:
  seed: 7
  time: { mode: real }
  devices:
    - id: spb
      model: generic
      protocol: { sparkplug: { broker: local, group_id: skyranger, edge_node_id: "node-{index:03d}" } }
      count: 100                           # 100 edge nodes
      devices_per_node: 25                 # Sparkplug devices under nodes: fewer TCP connections (file 03 section 8)
      points_per_device: 3                 # 7,500 Sparkplug tags
      generator: { kind: random_walk, change_rate_per_tag: 0.1 }   # parameter swept by the runner
    - id: plcs
      model: modbus_plc
      count: 10
      points_template: { holding_float32: 250 }                   # 2,500 Modbus tags at 1 s polling
  load_profile:
    - { name: typical, change_rate_per_tag: 0.1, hold: 30m }
    - { name: stress,  change_rate_per_tag: 1.0, hold: 15m }
    - { name: burst,   action: burst, params: { updates_per_s: 10000, for: 10s, every: 5m }, hold: 30m }
    - { name: ramp_to_knee, ramp: { step: 25%, step_hold: 10m, until: knee } }
  clients:
    - { kind: ws,   count: 20, subscribe: "site1/#" }
    - { kind: rest, count: 5,  poll: { path: "/api/v1/tags/values?prefix=site1/area0", every: 1s } }
```

For no-protocol engine load (isolating the engine from the broker), the same scenario runs with the `sim` driver (`sim://` addresses, file 03 section 6). Comparing the two shows how much the protocol layer costs.

### 2.3 Measurements and targets

| Metric | W10-typ | W10-stress | W100-typ | W100-stress |
|---|---|---|---|---|
| Ingest-to-WS latency p99 (includes 100 ms WS batching) | <= 300 ms | <= 1 s | <= 1 s | measure only |
| Counter exactness | exact | exact | exact | exact |
| Quality transitions delivered (none dropped) | 100% | 100% | 100% | 100% |
| `dropped_samples` for non-counter value changes | 0 | allowed, counted | 0 | allowed, counted |
| CPU, whole box, reference hardware | <= 30% | <= 70% | <= 50% | measure only |
| RSS `skyrangerd` | <= tag-engine memory budget (01-tag-engine performance doc, pending) | same | same | same |
| Config apply time (add 100 tags to running system) | <= 5 s | n/a | <= 30 s | n/a |
| Cold start: process start to API ready / to all tags `good` | <= 30 s / <= 2 min | n/a | <= 2 min / <= 10 min | n/a |
| `GET /tags/values` 1,000 ids | p99 <= 100 ms | | p99 <= 200 ms | |
| History: 1 tag, 24 h raw | p95 <= 200 ms | | p95 <= 500 ms | |
| History: 1 tag, 30 d, 1 h buckets | p95 <= 500 ms | | p95 <= 1 s | |
| History: 100 tags, 24 h, 1 min buckets | p95 <= 1.5 s | | p95 <= 3 s | |
| 20 WS clients and 5 REST pollers (above), added on top | within the same latency bounds | | | |

All numbers are **proposed targets**, UNVERIFIED, to be replaced after the first baseline runs (section 5). They are written down now so the first run is a comparison, not a shrug.

**The knee test and the headroom rule.** The `ramp_to_knee` profile raises the sample rate by 25% every 10 minutes. The knee is the last step before any of: ingest-to-WS p99 above 3x the baseline, the historian queue depth growing monotonically over a full step, or non-zero drops of quality transitions. **The declared supported rate is at most 50% of the measured knee** (2x headroom), published in the evidence pack together with the hardware identification. A limit with no headroom is a limit that fails on the first unusual day (a restart storm after a power cut produces bursts far above steady state: F-VAL-06 and the reboot storm scenario).

### 2.4 Microbenchmarks (per commit, trend only)

`go test -run '^$' -bench . -benchmem -count 10` for the hot paths: Sparkplug payload decode, driver `Sink` coalescer, tag pipeline (deadband, scale, derived), `counter_total`, registry lookup by path and by id, JSON encode of the canonical value, WS frame build, history batch encode.

```go
func BenchmarkEngineIngest(b *testing.B) {
    e := engine.NewForBench(tagset.Water(1000)) // 1,000 meters, fake clock, no I/O
    samples := bench.PulseSamples(b.N, 1000)
    b.ReportAllocs()
    b.ResetTimer()
    for i := 0; i < b.N; i++ {
        e.Ingest(samples[i%len(samples)])
    }
}
```

- CI per commit runs them only to check they still compile and run (`-benchtime 1x`).
- Nightly on the reference box: `-count 10`, compared against the last release baseline with `benchstat` (`golang.org/x/perf/cmd/benchstat`). Fail on a geomean slowdown above 10% with statistical significance (benchstat's `p < 0.05`), or any hot-path benchmark allocating more than its recorded `allocs/op` (a ratchet file, protected, may only be lowered by a human).
- Benchmarks run on dedicated hardware only; on shared runners they would flap.

### 2.5 Other scale probes

| Probe | What | Pass |
|---|---|---|
| P-CONN | 10,000 Sparkplug edge-node connections (file 03 requirement) with births spread over 5 min, then a **simultaneous reconnect storm** (broker restart) | all births processed, all tags `good` within 10 min, no process crash, memory bounded |
| P-STORM | After a power cut: 500 meters rebirth within 10 s | rebirth rate limit of 30 s per node respected; conservation exact; engine latency recovers within 2 min |
| P-ALARM | 5,000 alarms activating within 1 s, then clearing | alarm engine keeps up, no lost state changes, `GET /alarms` pagination correct, alarm flood does not stall value flow |
| P-HIST-RET | 30 d of 10k tags pre-loaded by a generator (not live), then query set (2.3) | latency table holds on a populated DB, not only an empty one |
| P-REG | 100,000-tag config import, apply, rename 1,000 tags, delete 1,000 | export/import identical; no tag-id changes where the contract says stable |

## 3. Sim load generator limits

At 10,000 connections the simulator, broker and gateway share one test host unless split. Rules so the load generator is not the bottleneck:

- The sim runs on a **separate machine** from the gateway on the perf rig, joined by a dedicated switch; its own CPU must stay under 50% (reported; a saturated sim invalidates the run).
- The WS latency client runs on the sim host (same clock as the oracle). Gateway and sim clocks are disciplined by chrony from one local source and offset is logged; latency is **never** computed across two hosts' wall clocks.
- Every run records sim CPU, tap `records_dropped` (file 05), broker CPU and connection count.

## 4. CI placement

| Job | Tier | Contents |
|---|---|---|
| bench-compile | per push | benchmarks compile and run once |
| perf-nightly (lab runner `lab-perf`) | nightly | W10-typ 30 min, W10-stress 15 min, microbenchmarks with benchstat |
| perf-weekly | weekly | W100-typ 1 h, knee test at 10k, P-CONN, P-STORM, P-ALARM |
| perf-release | release candidate | everything above plus knee test at 100k and the retention-loaded query set; results become the supported-limits table in the evidence pack |
| soak-nightly / weekly / release | per 1.1 | |

## 5. Calibration and ratchets

1. **First three baseline runs** of each test on the reference box record values without gating. Then thresholds are set to the worse of (observed p99 x 1.5) and the proposed target, and committed in `test/perf/thresholds.yaml` (protected).
2. Thresholds ratchet **tighter** automatically when five consecutive releases beat them by 20% (a human approves the PR); they never loosen without a decisions-log entry explaining why.
3. Any change that moves a headline number by more than 10% needs a note in the release notes.

## Decisions made
1. Soak: 6 h nightly, 24 h weekly, 72 h per release candidate on the exact image digests, 14 days before the first customer; failed runs are never retried.
2. The soak plant is mixed (Sparkplug, Modbus, OPC UA, API clients) with a seeded Poisson schedule of self-recovering faults and a storage-lifecycle profile that exercises compression and retention within 72 h.
3. Pass/fail is decided by a separate judge program using public endpoints only, with exact conservation, robust slope tests for memory, and explicit criteria S-1 to S-14.
4. 10,000 tags is a supported claim; 100,000 is a scale-validation target. A published supported limit is at most 50% of the measured knee.
5. Performance numbers count only from reference hardware with a separate load-generator host; shared-runner numbers are trend-only.
6. All thresholds begin as proposals, are calibrated from three baseline runs, and only ratchet tighter.

## Alternatives rejected and why
- **Soak at compressed time**: hides time-dependent defects and would make the evidence a lie. Real time only (file 03 decision 4); the storage-lifecycle profile changes maintenance frequency, not timing claims.
- **Fixed absolute memory limit as the leak test**: a limit that is generous passes a leak for days. Slope plus growth bound is what finds leaks.
- **Gating on shared CI runner perf numbers**: noisy neighbours; flaky gates train people to ignore the gate.
- **Claiming 100k tags up front**: unmeasured claims are the exact thing this strategy exists to avoid.
- **k6 or Locust for load**: HTTP-oriented; our load is protocol-level device traffic, which the simulator already generates, plus a few API clients. Revisit if API load becomes the issue.
- **Gating on `go test -bench` raw ns/op**: noise; benchstat's statistical comparison is the minimum honest method.

## Open questions
1. **Reference hardware is not chosen**; every absolute threshold depends on it. Until then runs record against a named development box and the evidence pack must not be produced.
2. **Contract observation (time precision).** Timestamps are RFC 3339 with millisecond precision and history is keyed on `(tag_id, ts)` (tag-model.md section 5, F-DB-04 idempotent insert). Two distinct samples of the same tag within one millisecond collapse to one key. Irrelevant at 1 sample/s/tag, but the burst fault and any high-rate source (vibration, sub-ms Modbus gateways) can hit it. Is the key `(tag_id, ts)` only, and is dropping the second sample intended? The burst test records collisions as a counter either way.
3. The metric names in section 1.4 beyond the Prometheus Go defaults belong to the observability doc and are proposals.
4. Whether a single-writer engine goroutine is sufficient at 100k stress is the question the knee test answers; if not, the answer (sharding by tag prefix) is an architecture change that the tag-engine doc owns.
5. Storage format for soak metrics (JSONL versus Parquet) is open; JSONL is the default for diffability.
6. 14-day soak on the reference box ties up the rig; it needs a second box or a scheduled lab window.
