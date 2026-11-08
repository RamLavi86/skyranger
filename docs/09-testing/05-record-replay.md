# 05 Record and Replay

Real devices do things simulators do not think of: odd retransmit patterns, a Wi-Fi stack that sends two births in a row, a Modbus PLC that answers slowly after 03:00. Recording real traffic and replaying it later turns every field surprise into a permanent regression test. This file defines what is captured, in what format, how it is anonymized, and how a replay becomes a pass/fail test when there is no simulator oracle.

Two capture levels, because they answer different questions:

| Level | What is recorded | Where tapped | Replays into | Answers |
|---|---|---|---|---|
| **Wire** | Raw protocol messages with arrival timing: MQTT packets (topic, payload bytes, QoS, retain) for Sparkplug; Modbus TCP ADUs (MBAP header + PDU) request/response pairs | Broker-side subscriber tap (MQTT); logging TCP proxy in front of the PLC (Modbus) | `skyranger-sim` model `replay` (real protocol on real sockets) | Does the driver and everything above it still handle exactly what the field sent? |
| **Sample** | The `Sample` and `ConnectionState` stream a driver delivered to the driver host (driver-interface.md `Sink`), after decoding | Tap inside the driver host (a `Sink` decorator, enabled by config flag `capture`) | `replay` into the L3 harness (no protocol) or the `sim` driver path | Does the tag engine, alarm engine and historian still behave the same on this input? Works for OPC UA, where wire capture is impractical (encrypted sessions, large library surface) |

Wire replay covers the driver; sample replay covers everything above the driver. A bug found in the field is first captured at whichever level reproduces it.

## 1. Capture file format `skyranger-capture/v1`

JSON Lines, one record per line, UTF-8, optionally gzip (`.jsonl.gz`). Chosen over pcap because it is greppable, diffable, trivially anonymized at field level, and needs no libpcap. It deliberately differs from `apiVersion: skyranger/v1` so a capture can never be mistaken for config.

```jsonl
{"format":"skyranger-capture/v1","level":"wire","protocol":"mqtt","captured_at":"2026-11-07T10:00:00.000Z","gateway_version":"0.1.0","anonymized":false,"tap":{"name":"broker-tap","version":"0.1.0"},"notes":"Pumphouse, 3 meters, 6 h"}
{"t":"0.000000","dir":"in","proto":"mqtt","topic":"spBv1.0/skyranger/NBIRTH/esp32-a1b2c3","qos":0,"retain":false,"payload_b64":"CgQ..."}
{"t":"0.412381","dir":"in","proto":"mqtt","topic":"spBv1.0/skyranger/NDATA/esp32-a1b2c3","qos":0,"retain":false,"payload_b64":"CjI..."}
{"t":"61.002114","dir":"event","kind":"tap_gap","detail":"tap reconnected; records may be missing","missing_hint":"unknown"}
```

Rules:
- `t`: seconds since capture start as a decimal string with microsecond precision, from the tap's **monotonic** clock. Absolute time appears only in the header. This lets replay rebase onto any start time (virtual or real).
- `dir`: `in` (device to broker/gateway), `out` (gateway to device, e.g. NCMD rebirth, Modbus request), or `event` (tap-level markers: `tap_gap`, `tap_start`, `tap_stop`). A tap that cannot guarantee completeness MUST write `tap_gap` rather than silently losing records.
- Payloads are base64 of the exact bytes. Decoded views for humans are produced by `skyranger-capture show`, never stored (stored decodes drift from the bytes).
- Sample-level capture has the same envelope with `"level":"sample"` and records `{"t":..., "handle": "...", "address": "spb://...#pulse_count", "value": {"type":"uint64","v":"60"}, "quality":"good", "ts_source":"...", "ts_gateway":"..."}` using the canonical encodings (64-bit ints as strings, RFC 3339 UTC ms). `ts_gateway` is kept for audit but the replayer re-stamps it (section 3).
- A companion `*.meta.yaml` states: what real hardware and firmware produced it, the config revision (the exported YAML of the gateway at capture time, with secrets removed), anonymization status, consent reference, and the **expected-output file** (section 4).

## 2. Capturing

### 2.1 Taps

| Tap | Implementation | Notes |
|---|---|---|
| MQTT broker tap | `skyranger-capture tap mqtt --broker ... --topics 'spBv1.0/#'` Go program using an MQTT client library different from the gateway's where practical (same reasoning as file 03 section 8). Subscribes QoS 1 and appends records | Sees what the broker forwarded, which is what the gateway's subscription also sees. Records also the gateway's `NCMD` publishes (`out`). Never writes under `skyranger/v1/...` unless asked (the internal bus is derived data) |
| Modbus TCP logging proxy | `skyranger-capture tap modbus --listen :5502 --upstream plc:502` Records each ADU pair with both timestamps; understands MBAP framing so records are whole frames | Placed in front of a real PLC for a short capture window. Never left in production |
| Driver-host `Sink` decorator | Config flag on the driver host: `capture: {level: sample, path: /var/lib/skyranger/capture, max_size: 256MiB, rotate: 1h}` | Off by default; **UNVERIFIED** whether this belongs in the production binary or a debug build. Proposal: production binary, off by default, enabled only by an `engineer` through config, audited like any config change |
| HIL rig tap | The L7 rig (file 08) runs the MQTT tap permanently during every run and archives it with the results | This builds the real-device corpus (section 5) at no extra effort |

### 2.2 Tap validation (must pass before a capture counts as evidence)

The L4 conservation oracle (file 03 section 5) trusts the MQTT tap for "what was delivered". A tap that drops records under load produces false failures or, worse, false passes. Tap validation:

1. `TestTapCompleteness`: simulator publishes N numbered messages at the perf-target rate (file 07); tap must record exactly N, in order per topic, with `tap_gap` count zero. Run on the tap itself nightly.
2. The simulator logs what it sent (oracle JSONL); a compare tool checks tap records against it 1:1 by (topic, payload hash).
3. The tap exposes `records_written`, `records_dropped`, `reconnects`. Any non-zero `records_dropped` in a run marks the run's conservation result **inconclusive**, not pass.

### 2.3 Capture hygiene and consent

- Capturing customer traffic requires written customer consent that names what is captured, retention, and who can access it (template in the evidence-pack process, file 11). The consent reference goes in `meta.yaml`.
- Customer captures **never enter the main repository** (README decision: tests live in the main repo except customer captures). They live in a private object store (bucket or encrypted volume) accessible to the lab runner only; CI jobs on public/fork PRs cannot reach it.
- A capture of a customer is anonymized (section 2.4) before leaving the customer-controlled machine whenever feasible.

### 2.4 Anonymization

Goal: nothing that identifies the customer, site, or people survives; everything that affects behavior does survive.

| Item | Treatment |
|---|---|
| `group_id`, `edge_node_id`, `device_id` | Replaced by a keyed deterministic mapping (`anon-0001`, ...) per capture; mapping stored only in an encrypted side file kept by us, never in the capture. Must still satisfy the segment grammar |
| Metric names | Mapped only if they look site-specific; the pulse-meter profile names (`pulse_count`, `boot_count`, `uptime_s`, ...) are kept verbatim because behavior depends on them. A configurable allowlist of safe names |
| MQTT username / client id / credentials | Dropped (the tap never records the CONNECT packet payload credentials) |
| Payload strings (`fw_version`, free text, String metrics) | Kept if on the allowlist (`profile`, `fw_version`), else replaced by same-length placeholder |
| Timestamps | Shifted by a random constant per capture; relative spacing preserved exactly. Header `captured_at` coarsened to the date |
| IP addresses, MACs (e.g. MAC embedded in `esp32-a1b2c3`) | Mapped; the MAC-derived `edge_node_id` is a node id, so it is covered by the first row |
| Numeric values | **Kept**. Counter values and flow behavior are the test. If a customer considers volumes confidential, apply a single secret multiplier to `pulse_count` only after confirming with the owner that rollover behavior is not under test (a multiplier changes where wraps happen). Default: no value changes |

Sparkplug payloads are protobuf, so anonymization means decode, rewrite fields, re-encode with the Tahu `.proto` (the contract artifact in messages.md section 1), recompute nothing else. `seq` and `bdSeq` are preserved bit-exactly. The tool's own tests:

```go
// cmd/skyranger-capture/anonymize_test.go
func TestAnonymizeRemovesIdentifiersKeepsBehavior(t *testing.T) {
    in := loadCapture(t, "testdata/capture/synthetic-pumphouse.jsonl") // synthetic, contains planted canaries
    canaries := []string{"Acme-Water", "pumphouse-north", "esp32-deadbe", "10.20.30.40"}
    out, mapping := anonymize.Run(in, anonymize.Options{Seed: 1, KeepMetrics: anonymize.PulseMeterProfileNames})

    raw := marshalAll(out)
    for _, c := range canaries {
        if bytes.Contains(raw, []byte(c)) { t.Errorf("canary %q survived anonymization", c) }
        if bytes.Contains(raw, []byte(base64.StdEncoding.EncodeToString([]byte(c)))) { t.Errorf("canary %q survived inside payload", c) }
    }
    // behavior preserved: replaying both through the L3 harness gives the same tag outputs modulo the id mapping
    a := replayL3(t, in)
    b := replayL3(t, out)
    if diff := cmp.Diff(a.Normalize(mapping), b.Normalize(identity)); diff != "" {
        t.Fatalf("anonymization changed behavior:\n%s", diff)
    }
}
```

Planted canaries in a synthetic capture are the only trustworthy test; "looks anonymous" by eye is not accepted. A human still spot-checks the first capture of each new customer before it is stored.

## 3. Replaying

### 3.1 Replayer

`skyranger-sim` gets a device model `replay` (extends file 03 section 2 models). It reads a capture and re-emits the `in` records on real sockets (MQTT publish to the broker; Modbus server answering from recorded responses keyed by request bytes, in recorded order per request key).

```yaml
apiVersion: skyranger-test/v1
kind: Scenario
metadata:
  name: replay-field-meter-reboot-0042
  description: "Captured from a real ESP32 on the HIL rig: reboot during rapid flow change"
  tags: [replay, water_meter, counter]
  faults_covered: [F-DEV-01]
spec:
  seed: 1
  time: { mode: scaled, speed: 20 }          # logic regression only; timing claims never from scaled runs (file 03)
  brokers:
    - { name: local, url: "mqtt://mosquitto:1883" }
  devices:
    - id: field
      model: replay
      protocol: { sparkplug: { broker: local } }
      params:
        capture: test/replay/hil/reboot-0042.jsonl.gz
        on_gap: preserve                     # preserve | compress (cap any gap to max_gap) 
        rebase_start: 2026-11-07T10:00:00Z   # rewrites payload timestamps to this start
        respond_to_ncmd: false               # recorded `out` NCMD are not honored; see 3.3
  expect_from: test/replay/hil/reboot-0042.expected.yaml
```

Modes:

| Mode | How | Used for |
|---|---|---|
| **L3 virtual** | Capture fed through in-memory transport on `clock.Fake`; gaps cost nothing | every commit, for the small curated set |
| **L4 scaled/real** | `skyranger-sim replay` through Mosquitto; `scaled` allowed for logic checks | nightly, full set |
| **Real time** | speed 1 | weekly / pre-release for the captures that include timing-sensitive behavior (stale detection) |

### 3.2 What is rewritten on replay

A replay must not feed the gateway stale absolute times, or the clock-skew rule (`max_clock_skew`, default 10 s, tag-model.md section 5) would reject every sample.

- Payload and metric timestamps are rebased: `ts_new = rebase_start + (ts_old - first_ts_old)`. Devices that were unsynced in the capture (1970 timestamps) keep the **relative** weirdness: a flag `preserve_unsynced: true` keeps timestamps below 2020-01-01 as recorded, so F-CLK-01 style captures still exercise the rejection path.
- `ts_gateway` always comes from the gateway's own clock, never from the capture.
- `seq`, `bdSeq`, aliases, `boot_count`, `pulse_count`: **never rewritten**. These are the behavior under test.
- MQTT session details (client id, keepalive, Will) come from the replay model parameters, not the capture.

### 3.3 Limits of replay (stated, so nobody over-trusts it)

- Replay is **open loop**: the gateway's responses (rebirth requests) do not influence the recorded stream. If the capture had a rebirth exchange, that exchange is replayed as it happened; if the new gateway would have behaved differently (e.g. sends a rebirth where the old one did not), the device does not react. The replay model supports `respond_to_ncmd: true` where the capture contains a complete NBIRTH to answer with, a best-effort approximation, flagged in the report.
- Modbus replay answers by request key. If the new driver sends different requests (different batching), the replay peer cannot answer and reports `unmatched_request`. That is a finding about driver behavior change, and the test fails with that list rather than guessing.
- Replay proves "same input, same output", not "correct output". Correctness comes from the oracle in section 4.

## 4. Pass/fail without a simulator oracle

A real capture has no ground truth for volume, but three kinds of checks are available.

1. **Independent recomputation** (the strong one). The contract guarantees any total can be recomputed from raw history (messages.md section 6). `pkg/replay/audit` reads the capture, decodes `pulse_count` and `boot_count` itself, and computes the expected total with its own small implementation of the conservation rule, which deliberately does not import `pkg/tag/derived` (same independence rule as file 03 section 5). It then compares with the gateway's `total_volume` from `/api/v1/history`. Exact equality, as with the simulator.
2. **Golden outputs**. After a human has reviewed a replay run once and agreed it is correct, the run's observable outputs are frozen in `*.expected.yaml`: event counts by type and kind, ordered quality transitions per tag (value-free), alarm state sequence, final totals. Replays compare against it; a diff fails.
3. **Invariants** applied to every replay without per-capture authoring: no `implausible` events unless the metadata says the capture contains one; every `reset` event has a `boot_count` change in the capture at that point; `ts` is monotonic per tag in live values; no history rows with `ts` before 2020; no NaN in any output; total non-decreasing except through `manual_adjust`.

```yaml
# test/replay/hil/reboot-0042.expected.yaml (protected)
source_review: { reviewed_by: "human", date: "2026-11-09", commit: "abc123" }
events:
  - { type: skyranger.tag.counter_discontinuity.v1, kind: reset, count: 1 }
  - { type: skyranger.connection.state_changed.v1, count_min: 0 }   # not asserted exactly: depends on broker timing
tags:
  site1/pumphouse/meter01/health:
    quality_sequence: [good]
    value_sequence: [2, 0, 2]
  site1/pumphouse/meter01/total_volume:
    equals_audit: true            # independent recomputation, section 4 item 1
invariants: [default]
```

A frozen golden that was generated by the code under test proves nothing about correctness, only stability. Every `expected.yaml` therefore requires `source_review` and `equals_audit` wherever a number is involved. The file is a protected path (file 12).

## 5. Corpus and lifecycle

| Corpus | Location | Content | Runs in |
|---|---|---|---|
| `synthetic/` | `test/replay/synthetic/` (in repo) | Captures generated by `skyranger-sim` itself (not independent evidence; used to test the replayer and the anonymizer) | every commit (L3) |
| `hil/` | `test/replay/hil/` (in repo, small, compressed) | Captures from our own ESP32 on the HIL rig: reboots, power cuts, Wi-Fi loss. Real firmware behavior | every commit L3 (curated subset under 5 MB total, UNVERIFIED limit), nightly L4 (all) |
| `field/` | private object store | Anonymized customer captures | nightly on the lab runner; never in public CI |
| `bugs/` | in repo if non-sensitive, else private | One capture per production bug: the minimal slice that reproduces it | with the regression test for that bug |

Lifecycle rules:
- **Every field bug gets a capture** where the evidence exists. The bug-fix PR's red-green proof (file 12 section 3) can be this replay: fails on the base commit, passes on the fix.
- Captures are trimmed with `skyranger-capture slice --from 1h02m --to 1h09m` to the smallest reproducing window, keeping the NBIRTH preceding it so the slice is self-contained.
- Corpus size budget and compression: `.jsonl.gz`, one scenario per file. Large captures (hours) stay in the object store; the repo holds slices.
- A capture is **re-recorded, not edited**, when the contract changes (e.g. a v2 device profile). Editing recorded bytes defeats the point.
- Replay report (`replay-report.json`: capture id, mode, speed, pass/fail by check type, audit total vs gateway total) goes into the evidence pack's "real-device behavior" table (file 11).

## 6. Go test sketch (L3)

```go
//go:build !integration

func TestReplayCorpus(t *testing.T) {
    files, _ := filepath.Glob("test/replay/*/*.jsonl.gz")
    for _, f := range files {
        f := f
        t.Run(filepath.Base(f), func(t *testing.T) {
            t.Parallel()
            meta := replay.LoadMeta(t, strings.TrimSuffix(f, ".jsonl.gz")+".meta.yaml")
            h := harness.New(t, harness.Config(meta.ConfigYAML), harness.Seed(1))
            h.Replay(f, replay.Rebase(meta.RebaseStart), replay.Transport(h.MemTransport()))
            h.RunUntilReplayEnd()

            got := h.Observe()   // events, quality sequences, alarms, totals via the public API
            if meta.Expected.EqualsAudit {
                want := audit.Recompute(t, f)        // independent implementation
                for path, total := range want.Totals {
                    h.AssertTagValue(path, total)    // exact
                }
            }
            if diff := cmp.Diff(meta.Expected.Normalize(), got.Normalize()); diff != "" {
                t.Fatalf("replay output changed vs reviewed golden:\n%s", diff)
            }
            replay.CheckInvariants(t, got)
        })
    }
}
```

## Decisions made
1. Two capture levels, wire and sample, because wire replay tests the driver and sample replay tests everything above it, and OPC UA is only practical at sample level.
2. A plain JSON Lines capture format with exact payload bytes, monotonic relative time, and explicit `tap_gap` markers; no pcap dependency.
3. Replay results are judged by independent recomputation from raw counts (not by the production derived-tag code), a human-reviewed golden file, and a default invariant set.
4. Customer captures never enter the repository; synthetic and own-hardware (HIL) captures do. Anonymization is tested with planted canaries.
5. A capture tap that drops records marks the run inconclusive instead of passing.
6. Every field bug gets a minimal capture and a red-green regression test where evidence exists.

## Alternatives rejected and why
- **pcap/tcpdump captures**: need TLS key handling once TLS is mandatory (security docs), are opaque to review, and anonymizing inside packets is error-prone. JSONL of decoded-at-the-edge messages is simpler. pcap remains fine for ad-hoc debugging.
- **Replaying by re-driving the simulator with a fitted model** (infer flow profile from the capture): loses exactly the oddities that justified capturing.
- **Storing customer captures in the repo, anonymized**: one anonymization bug becomes a permanent public leak in git history. Separate private store.
- **Golden files auto-generated on first run**: stable but unreviewed output is not evidence. Review plus audit recomputation required.
- **Replaying Modbus by regenerating requests from the capture**: hides driver request-pattern changes. Request-keyed replay surfaces them.

## Open questions
1. Whether the driver-host `Sink` capture decorator ships in production builds (needed for field debugging, but adds a data-exfiltration surface). Needs a decision with the security doc; it is off by default and engineer-gated in this proposal.
2. Repo size limits for `test/replay/hil/` (5 MB is a proposal) and whether Git LFS is acceptable. UNVERIFIED hosting limits.
3. Whether OPC UA sample captures can be taken at the `Sink` level without driver changes; depends on the driver framework in 02-drivers.
4. The rebase rule rewrites payload timestamps; Sparkplug `metric.timestamp` and `payload.timestamp` semantics for historical/unsynced values (messages.md Open question on permitted omission) are UNVERIFIED and affect what the replayer may legally emit.
5. Consent template and retention periods are legal matters; the text here is a placeholder, not legal advice.
