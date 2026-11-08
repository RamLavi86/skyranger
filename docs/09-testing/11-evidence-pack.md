# 11 The Customer-Facing Evidence Pack

The business case is "as reliable as the incumbents, cheaper, and you can leave". A customer will not read our source code, but a plant manager, a utility engineer or their IT auditor can read evidence. The evidence pack is that evidence: a signed, versioned document set generated from CI results, tied to the exact build and hardware the customer receives.

Two ground rules:

1. **Every number in the pack and in sales material traces to a CI run.** No hand-written figures. A *claims register* (section 3) fails the build when a claim is no longer supported.
2. **We state what we did not test.** An evidence pack that only shows green is less credible than one that says "not tested: temperature extremes, SIL certification, networks above 100k tags". Honest limits are a selling point to people who have been burned by spec sheets.

We make **no comparative claims about Siemens, Rockwell or any other vendor's reliability**, because we have no verified data on their products. The pack makes absolute, reproducible claims about Sky Ranger; the reader draws the comparison. (A comparison table of *features* and *lock-in* is a separate marketing document, outside this area.)

## 1. Audiences and formats

| Artifact | Audience | Length | Format |
|---|---|---|---|
| **One-page summary** | Executive, plant manager | 1 page | PDF; generated section "What was tested, what passed, what is not covered" with five headline numbers |
| **Technical evidence report** | Engineer, utility technical reviewer | about 20 to 30 pages | PDF (generated from HTML) with graphs |
| **Evidence bundle** | Auditor, customer IT, future us | files | A directory/zip with raw JSON reports, metric data, SBOMs, graphs as SVG/PNG, scenario files, the manifest and signature |
| **Reproduction kit** | Skeptical customers who want to see it themselves | docker compose + README | `make demo` runs the simulator and the pumphouse scenario on a laptop or on the customer's own hardware; FAT can be witnessed |
| **Customer appendix** | The specific customer | few pages | Their FAT and SAT records (file 10 section 5), customer-config checks, the hardware SKU qualification (if it was needed) |

Generation: `tools/evidencepack` (Go) reads CI artifacts (JSON reports from files 02, 04, 05, 07, 08, 09, 10), renders Markdown from Go templates, converts to HTML with plain CSS, and prints PDF with headless Chromium (already in the toolchain through Playwright, file 06). Graphs are drawn from the metric files with a fixed script, so the same data always gives the same picture. The manifest lists every included file with its SHA-256; the manifest is signed with the release key (cosign, 07-security/07-secure-update). The pack states the image digests it covers; a verifier compares them with what is installed (`shipcheck`, file 10).

## 2. Contents of the technical report

Each section lists its source, so the generator knows what to read and a reviewer knows what to challenge.

| # | Section | What it shows | Source |
|---|---|---|---|
| 1 | **Scope and identity** | Release version, image digests, config schema version (`skyranger/v1`), device profile (`skyranger.pulse_meter.v1`), reference hardware (SKU, SSD model and firmware, kernel), date, CI run IDs | release record |
| 2 | **What was tested, by layer** | Test counts and pass rates for L0 to L7 (pyramid with real numbers), coverage by package, mutation score for critical packages | CI reports |
| 3 | **Interface contract conformance** | Golden-file corpus passed; breaking-change checks versus the previous release; driver conformance table: every case ID per driver per hosting mode (in-process and gRPC), device-profile PM-01..PM-10 | files 01, 02 |
| 4 | **Fault tolerance** | The fault catalogue as a table: ID, what we did to the system, what the system did, which tiers executed it, pass/fail. Sorted by severity. This is the heart of the pack | file 04 (`faults.yaml` + results) |
| 5 | **Data integrity: counted volume** | Conservation results: simulator runs, soak runs and HIL runs, each with oracle total versus gateway total and the difference (**0 expected**). Counter rollover, reboot and reconnect cases listed separately. The exact bound of loss at a device reboot stated plainly | files 03, 07, 08 |
| 6 | **Soak** | 72 h graphs (memory, goroutines, file descriptors, latency percentiles, historian queue, DB size) with the fault schedule marked and criteria S-1..S-14 verdicts | file 07 |
| 7 | **Power loss and hardware** | N power cuts, cut modes and targeted phases, failures (zero or listed), boot-to-healthy histogram, history loss distribution, A/B update interruption results, watchdog result. The **rule-of-three** bound stated: "0 failures in 1,000 cuts: failure probability per cut below about 0.3% at 95% confidence" | file 08 |
| 8 | **Device accuracy** | ESP32 pulse accuracy (PM-08) vs the two-counter ground truth; accuracy table across rates; 7-day run | file 08 |
| 9 | **Performance and supported limits** | Supported tags, samples per second, retention, number of clients on the tested hardware; latency percentiles; the knee measurement and the 2x headroom rule | file 07 |
| 10 | **Security testing** | Fuzz-hours per parser, crashers found and fixed, authorization matrix results, scanner versions and dates, SBOM reference, open findings with waivers and expiry, penetration-test status (report summary if one exists, "not yet performed" if not) | file 09 |
| 11 | **Recovery and operations** | Upgrade and rollback, backup and restore, factory restore, time-to-recover per fault | files 06, 10 |
| 12 | **Real-world behavior** | Replay of captured real-device traffic: count, results, independent audit match | file 05 |
| 13 | **Reproducibility** | How the customer can rerun the demo and witness FAT; seeds and scenario files included | files 03, 10 |
| 14 | **Limits and non-claims** | The honest list (section 4) | claims register |
| 15 | **Change history** | Differences from the previous release's evidence: numbers that got better or worse | previous pack |

### 2.1 Example of the generated fault table (illustrative layout, not real results)

```
Fault        Severity  What we did                              Expected (summary)                                  L3   L4   L6   L7   Result
F-DEV-01     critical  Power-cycled the ESP32 (1,000 times on   Meter shown offline, then recovered; one reset      pass pass pass pass PASS
                       real hardware, 5,000 in simulation)      event per reboot; counted volume exact vs ground
                                                                truth within the documented loss bound
F-BRK-01     critical  Restarted the MQTT broker                All values marked "no connection", then restored    pass pass pass -    PASS
F-DB-01      critical  Stopped the database for 10 minutes     Live values and alarms unaffected; history          pass pass pass -    PASS
                                                                back-filled without gaps or duplicates
F-CLK-04     high      Moved the gateway clock back 10 minutes  (known limitation, see section 14)                  pass -    -    pass KNOWN-ISSUE
```

The result column has exactly three values: `PASS`, `FAIL` (never ships), `KNOWN-ISSUE` (a documented limitation with a ticket, a described workaround, and customer notification). A `KNOWN-ISSUE` on a `critical` fault blocks release unless the exception procedure in file 10 section 3 is used.

### 2.2 The five headline numbers on the one-page summary

Chosen because each is a plain-language, checkable statement. All come from the claims register and update automatically.

1. Counted volume error across all ground-truth tests: **exactly 0 liters** (n runs, N pulses), except the contract-defined loss at device reboot which is shown separately.
2. Unattended continuous run: **72 hours** with N injected faults, zero unexplained restarts.
3. Power cuts survived: **n cuts, 0 failures, 0 configuration corruptions** on the stated hardware.
4. Tested capacity: **x tags / y samples per second on the stated hardware**, with 2x headroom.
5. Security: **N fuzz-hours**, zero known unpatched critical or high vulnerabilities at release, independent penetration test **status**.

## 3. The claims register (stops us overselling)

Every sentence in the evidence pack, the website and sales decks that contains a number or a guarantee must exist in `test/evidence/claims.yaml`. CI evaluates each claim against the data and fails when evidence is missing or below threshold, so a regression that breaks a promise breaks the build instead of surprising a customer.

```yaml
# test/evidence/claims.yaml (protected path)
- id: CLAIM-VOL-001
  text: "Counted volume matches an independent ground-truth count exactly in all ground-truth tests."
  evidence: [soak.conservation, hil.conservation, sim.conservation]
  rule: all_equal_zero
  applies_to: release
  limits_text: "Excludes pulses lost between the last delivered message and a device reboot (documented bound)."

- id: CLAIM-SOAK-001
  text: "Ran {soak.hours} hours continuously with {soak.faults} injected faults and {soak.unexplained_restarts} unexplained restarts."
  evidence: [soak.release]
  rule: { soak.hours: ">= 72", soak.unexplained_restarts: "== 0" }

- id: CLAIM-PWR-001
  text: "{hil.cuts} power cuts on {hil.sku} with {hil.failures} failures."
  evidence: [hil.power_cut_release]
  rule: { hil.cuts: ">= 1000", hil.failures: "== 0" }
  statement_template: "failure probability per cut below {rule_of_three(hil.cuts)} at 95% confidence"

- id: CLAIM-CAP-001
  text: "Supported up to {perf.supported_tags} tags at {perf.supported_samples_per_s} samples per second on {hil.sku}, with at least 2x headroom."
  evidence: [perf.knee, perf.supported]
  rule: { perf.supported_samples_per_s: "<= 0.5 * perf.knee_samples_per_s" }

- id: CLAIM-SEC-001
  text: "No known unpatched critical or high vulnerabilities in shipped images at release."
  evidence: [sec.trivy, sec.govulncheck, sec.waivers]
  rule: no_unwaived_critical_high

- id: CLAIM-PT-001
  text: "Independently penetration tested ({pt.vendor}, {pt.date})."
  evidence: [sec.pentest]
  rule: exists
  applies_to: [conditional]   # printed ONLY if a pentest report is on file; otherwise the pack prints "not yet performed"
```

Rules:
- The generator renders claim text from the data (`{soak.hours}` etc.). Nobody types a number.
- A claim whose evidence is missing is **omitted and listed under "not claimed"**, never silently dropped. `CLAIM-PT-001` is the model: absent evidence changes the sentence, not the truth of it.
- Marketing text outside the pack that quotes a number must reference a `CLAIM-` id (checked by a link-checker script run over the website repo or deck source, **UNVERIFIED** workflow). Reviewers can then audit that every public number traces to a run.
- Claims are scoped by `applies_to` (hardware SKU, release). Evidence for SKU A is never printed for SKU B.
- The register is a protected path (file 12): an agent cannot loosen a threshold to turn the build green.

## 4. Limits and non-claims (section 14 of the report, written in plain language)

Always printed, regenerated from facts where possible:

- **Not a safety system.** Sky Ranger supervises and records; safety interlocks stay in PLC or hardwired logic. No SIL or functional-safety claim is made.
- **No certification claimed** unless obtained (IEC 62443, cybersecurity or metrology approvals). Water metering used for **billing** may need approved meters and calibration regimes: the platform's totalizer is not a legal-for-trade measurement (**UNVERIFIED** regulatory position; local rules apply, to be confirmed with the customer's regulator).
- **Loss at device reboot** is bounded and recorded (the pulses between the device's last delivered message and its reboot cannot be recovered). The report states the bound and the typical magnitude from the soak data (pulses lost per reboot, distribution), and shows the `reset` events that flag affected reporting periods.
- **Hardware**: results are valid for the listed SKU(s) and SSD(s) only. Other hardware needs qualification (file 10 SHIP-12).
- **Environment**: tested at lab ambient (state it). No temperature, vibration or EMC testing unless done (file 08).
- **Duration limits**: 72 h soak plus 14 days pre-first-customer; leaks smaller than about 1 MiB/h cannot be excluded by these runs (file 07).
- **Scale**: tested to the stated tag counts; behavior above them is not claimed.
- **Known issues** with workarounds (the `KNOWN-ISSUE` rows), including, as of this design, the clock-step issue F-CLK-04 until the tag-model contract is amended.
- **Test method limits**: the simulator is validated against the real device profile (PM-01..PM-07) but is not the real plant. Real-device behavior is covered by the HIL rig and by replay, and the pack says which results are from which.
- **Independence**: tests were written and run by the same organization that wrote the product. The pack says so, and the independent penetration test (when it exists) is the exception.

## 5. Making the evidence trustworthy

A skeptical reader's questions, answered by design:

| Question | Answer in the pack |
|---|---|
| "Did you test the thing you're selling me?" | Digests in the pack equal digests on the box (`shipcheck`); signed manifest |
| "Were the tests weakened to pass?" | Test-integrity report (file 12): protected paths changed in this release, with reviewer names; coverage floors and mutation scores with history; count of tests added/removed since the previous release |
| "Is this one lucky run?" | Seeds listed; same scenarios run nightly for months: the pack includes the **last 30 nightly pass rates** (a chart, with every red night shown and its cause) |
| "Does it work on my hardware?" | SKU table; qualification run if not listed |
| "What happened when it failed?" | The KNOWN-ISSUE list and the post-release escaped-defect list since the previous release |
| "Can I see it fail?" | The reproduction kit: customer can run faults themselves with the simulator's control API (file 03) and watch the HMI |

## 6. Process

1. RC build selected (file 10); R1 to R9 complete.
2. `tools/evidencepack build --release <rc-id>` collects reports, evaluates the claims register, renders the pack, writes the manifest.
3. A human (the author) reads the generated pack, specifically sections 4, 7, 14 and the diff in section 15, and signs the release record. **A human reads it every release**: the generator can be wrong too.
4. Pack signed and archived (immutable storage, retention at least the support life of the release plus the customer contract period).
5. Customer receives the one-pager, the report, the customer appendix; the bundle on request; the reproduction kit on request.
6. Later vulnerability or defect discoveries do not edit a delivered pack: they produce a **pack addendum** (new signed document referencing the original), so the history stays honest.

## 7. Evidence for the pre-sale stage (before a build exists for the customer)

For prospects, a **sample pack** from the latest release with the customer-specific appendix omitted and the reference hardware named. It is clearly watermarked "sample, not applicable to any installation until a customer appendix is issued". The reproduction kit lets a prospect watch a 20-minute demo (`demo-pumphouse` scenario, no test-only actions, file 03) on their own laptop: meter shown, reboot injected, leak alarm, broker restart, DB down and restored, counted volume compared with the simulator's oracle on screen.

## 8. Sketch of the generator contract

```go
// tools/evidencepack/claims.go
type Claim struct {
    ID       string            `yaml:"id"`
    Text     string            `yaml:"text"`
    Evidence []string          `yaml:"evidence"`
    Rule     RuleSpec          `yaml:"rule"`
    AppliesTo []string         `yaml:"applies_to"`
    LimitsText string          `yaml:"limits_text"`
}

type Verdict struct {
    ClaimID   string
    Printed   bool   // false => listed under "not claimed" with the reason
    Reason    string // "evidence missing: hil.power_cut_release", "rule failed: hil.failures == 0 (got 1)"
    Rendered  string // text with values substituted
}

func Evaluate(claims []Claim, data EvidenceStore) []Verdict { /* pure function, unit tested */ }

func TestEvaluate_FailingEvidenceFailsBuild(t *testing.T) {
    data := fixture.EvidenceWith(map[string]any{"hil.cuts": 1000, "hil.failures": 1})
    vs := Evaluate(claims.Load(t, "testdata/claims.yaml"), data)
    if !slices.ContainsFunc(vs, func(v Verdict) bool { return v.ClaimID == "CLAIM-PWR-001" && !v.Printed }) {
        t.Fatal("a failing power-cut result must not be printed as a claim")
    }
    if err := Gate(vs); err == nil {
        t.Fatal("release gate must fail when a required claim fails")
    }
}
```

The generator itself is tested with planted failing evidence (like the soak judge, file 07): a generator that cannot refuse to print a failing claim is not safe to ship.

## Decisions made
1. The pack is generated from CI artifacts and signed, with a manifest of hashes and the image digests; nothing in it is hand-typed.
2. A claims register binds every public number or guarantee to a machine-checked rule; failed or missing evidence changes what is claimed and fails the release gate.
3. Honest limits and non-claims are a mandatory section; we make absolute claims about Sky Ranger only and no comparative claims about other vendors.
4. Power-cut and similar statistical results are stated with their confidence bounds (rule of three), not as "never fails".
5. A human reads the generated pack at every release; later findings go in signed addenda, not edits.
6. A prospect gets a sample pack and a reproduction kit, so they can watch the faults themselves.

## Alternatives rejected and why
- **A hand-written white paper**: goes stale at the next commit and cannot be audited. Generated and regenerated.
- **A generic "quality" web dashboard as the deliverable**: changes under the customer's feet and has no signature. A frozen, signed document for the release plus the live dashboard for us.
- **Marketing claims with round numbers ("99.99% uptime")**: not measurable by these tests; availability in the field depends on the customer's power, network and hardware. We claim what we measured, and we describe measurement conditions.
- **Comparison tests against Siemens or Rockwell products**: requires access to their systems and licenses we do not have, and unverified comparisons are the kind of claim this document forbids.
- **PDF-only**: customers' auditors want raw data too; the bundle includes machine-readable reports.

## Open questions
1. Reference hardware and SSD models are unchosen; sections 1, 7 and 9 of the pack cannot be filled until they are.
2. Legal review of the limits language, warranty implications and any regulatory statement on billing-grade measurement is needed. Nothing here is legal advice.
3. PDF generation through headless Chromium: **UNVERIFIED** that its output is stable enough byte-for-byte; the manifest signs the PDF produced, and the HTML is also kept.
4. Whether to publish a public version of the sample pack on a website (marketing benefit versus exposing known issues). Recommendation: yes, because hiding the KNOWN-ISSUE list undermines the premise.
5. Retention period for archived packs and who holds the signing key if the author is unavailable (key custody belongs to 07-security/07).
6. The link-checker for marketing text that must cite `CLAIM-` ids needs an owner and a place to run, since marketing material lives outside this repo.
