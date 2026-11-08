# 10 CI Pipeline, Quality Gates and the "Ships to a Customer" Checklist

This file turns the test layers (README), faults (04), soak and performance (07), hardware (08) and security (09) into a pipeline with explicit gates. It also holds the single checklist that decides whether a build goes to a customer, and the FAT and SAT checklists.

Design constraint: the pipeline is expressed as `make` targets. The CI host is assumed to be GitHub Actions plus self-hosted lab runners, which is **UNVERIFIED** (README open question); swapping CI systems means rewriting thin YAML, not the pipeline.

## 1. Pipeline overview

```
 pre-commit ─► PR (per push) ─► merge queue ─► main (post-merge) ─► nightly ─► weekly ─► release candidate ─► ship
   L0, fast L1    L0..L3 + smoke    re-run on     image build,        L4 full,   soak 24h,   72h soak, 1,000 cuts,   promote
                  + guards          merged result  publish by digest   L5, 6h     perf, HIL   full security, FAT       by digest
                                                                       soak, HIL
                                                                       subset
```

Two rules shape everything:

1. **Build once, test that artifact, ship that artifact.** Images are built on `main` post-merge, identified by digest, and every later stage (nightly, weekly, release, FAT, customer install) tests and ships those same digests. Rebuilding "for release" invalidates the evidence. Release promotion is retagging a digest that already passed.
2. **Gates never auto-retry.** A red job is a bug in the product or the test, with a ticket (README principle 7). A manual re-run is allowed only to distinguish infrastructure failure (runner died) from test failure, is logged, and a test that fails and then passes without a code change is filed as flaky (section 6).

## 2. Stage tables

### 2.1 Pre-commit and per-PR (target: a PR gets a verdict in 15 minutes)

| Job | Make target | Contents | Budget | Required to merge |
|---|---|---|---|---|
| lint-static (L0) | `make lint` | gofmt, go vet, golangci-lint (incl. clock/rand lint rules, file 01), buf lint, `tsc`, eslint, JSON Schema validation of every YAML example in the docs, hadolint, gitleaks | 2 min | yes |
| unit (L1) | `make test-unit` | `go test -race -shuffle=on ./...` excluding `integration`; Vitest for HMI; fuzz seed corpus runs here | 5 min | yes |
| contract (L2) | `make test-contract` | golden tests (no `-update`), `buf breaking` vs `main`, OpenAPI diff vs `main`, cross-language golden checks (TS, Python), generated-code staleness | 2 min | yes |
| conformance | `make test-conformance` | driver conformance suite, in-process hosting, every driver in repo, with the simulator as peer; device-profile PM-01..07 against sim | 3 min | yes |
| component (L3) | `make test-harness` | harness scenarios (all `test/scenarios/` at virtual time), replay corpus subset (file 05) | 5 min | yes |
| coverage | `make cover-check` | per-package floors (`test/coverage-floors.yaml`) and 80% diff coverage | 1 min | yes |
| **testguard** | `make testguard` | test-integrity checks (file 12): deleted/weakened tests, skips, protected paths, red-green proof, directive coverage (`//fault:`, `//contract:`) | 2 min | yes |
| security-fast | `make sec-fast` | govulncheck, gosec, osv/npm/pip audit, Trivy fs scan, license check on dependency change | 3 min | yes |
| build | `make images` | Build images (for smoke only on PR; not published), SBOM | 4 min | yes |
| L4 smoke + L5 smoke | `make smoke` | compose smoke set S1..S8 and Playwright smoke (file 06) on the images just built | 8 min | yes |
| mutation (changed lines) | `make mutate-diff` | Mutation testing restricted to changed lines in critical packages (file 12) | 10 min | yes for critical packages |

Jobs run in parallel where independent (lint, unit, contract, conformance, harness, security-fast); smoke follows build. Wall time is bounded by the longest chain (about 15 min). Fork PRs run only the jobs that need no secrets or lab access (everything above except where noted); the lab runners never execute code from forks.

### 2.2 Merge queue and post-merge (`main`)

| Job | Contents | Notes |
|---|---|---|
| merge-queue re-run | Same as PR jobs against the merged result | prevents the semantic-conflict merge (two green PRs that break together) |
| image build and publish | Build, sign (cosign, 07-security/07), push by digest; SBOM attestation | digest list written to `release-candidates/main-<sha>.json` |
| post-merge smoke on published digests | L4 smoke + L5 smoke against the published images | proves the published artifact, not just the PR build |

### 2.3 Nightly (on the latest `main` digests)

| Job | Contents | Runner | Budget |
|---|---|---|---|
| L4 full | catalogue entries with L4 tier, fleet-500, mixed-plant, replay corpus all captures, PostgreSQL variants of L3 scenarios, production-compose check | CI | 60 min |
| L5 full | all HMI tests except weekly, Chromium and Firefox | CI | 30 min |
| soak-nightly | 6 h soak-plant | `lab-perf` | 6 h |
| perf-nightly | W10-typ, W10-stress, microbenchmarks with benchstat | `lab-perf` | 1 h |
| HIL nightly | 30 power cuts, PM-08 short, a subset of HIL-NET | `lab-hil` | 1.5 h |
| fuzz-nightly | 5 min per target | CI | 1 h |
| security-nightly | Trivy on images, SBOM re-scan of **shipped** releases, schemathesis, authz matrix, testssl | CI | 30 min |
| mutation-nightly | full mutation run on critical packages | CI | 2 h |
| fault coverage report | `tools/testguard faults` | CI | 1 min |

Failure of a nightly job opens an issue automatically, assigns the area owner, and blocks the merge queue if the failure is in a gate category (section 3, G-MAIN). Nightly results are posted to a dashboard; the previous 30 days are the trend that matters.

### 2.4 Weekly

| Job | Contents |
|---|---|
| soak-weekly | 24 h soak-plant on `lab-perf` |
| perf-weekly | W100-typ 1 h, knee test at 10k, P-CONN, P-STORM, P-ALARM |
| HIL weekly | 200 power cuts, network chaos HIL-NET-01..07, HIL-ESP-03/04/06 |
| fuzz-weekly | 1 h per target on the lab runner with cached corpus |
| full-history secret scan | gitleaks over all history |
| dependency drift | list of dependencies with available updates; automated PRs (Renovate or Dependabot, UNVERIFIED which) that must pass the full PR pipeline |
| flaky-test review | list from section 6 |

### 2.5 Release candidate (RC)

Triggered by tagging a digest set from `main` as `rc`. All of the following run on **that digest set**, and all must be green with no manual re-runs, to reach G-RC:

| # | Job | Content | Duration |
|---|---|---|---|
| R1 | Full PR pipeline re-run | everything in 2.1 | 15 min |
| R2 | L4 full and L5 full | including the upgrade/rollback and backup/restore sets (file 06) | 2 h |
| R3 | Release soak | 72 h soak-plant on the reference box | 72 h |
| R4 | Performance | perf-release (file 07): supported-limit table regenerated | 6 h |
| R5 | HIL | 1,000 cuts per supported SKU+SSD, PM-08, HIL-ESP-01..10, HIL-NET-01..08, HIL-HW-01..07 | about 36 h per SKU |
| R6 | Fault catalogue coverage | every `critical` and `high` fault covered at all `tiers_required` **and passed in R2..R5** | 1 min |
| R7 | Security | full fuzz campaign summary, Trivy on digests, SBOM, waivers valid, authz matrix, canary secret test, image hardening assertions | 2 h |
| R8 | Mutation | full critical-package run meets threshold | 2 h |
| R9 | Docs and contracts | contract-doc to golden consistency, `buf breaking` against the previous release tag, OpenAPI diff vs previous release (breaking changes need a major bump) | 5 min |
| R10 | Evidence pack generation | `tools/evidencepack` builds and signs the pack from R1..R9 outputs (file 11) | 10 min |
| R11 | FAT on the candidate | section 5 | 1 day |

R3 and R5 can run concurrently on different rigs. Calendar time for an RC is about 4 days; the release cadence is planned around it (an RC every one to two weeks is realistic for a solo integrator, UNVERIFIED).

### 2.6 Hotfix path

A hotfix may use a shortened release gate, never a skipped one:

| Job | Hotfix requirement |
|---|---|
| R1, R2, R6, R7, R9 | full |
| Soak | 24 h instead of 72 h |
| HIL | 200 cuts instead of 1,000, plus targeted cuts for the changed area |
| Fault coverage | unchanged |
| Red-green regression test | mandatory (a hotfix without a test that reproduced the bug is rejected) |
| Evidence pack | regenerated, marked "hotfix" with the reduced counts printed, never silently reusing 72 h or 1,000 numbers |

## 3. Quality gates

| Gate | Where | Pass criteria |
|---|---|---|
| **G-PR** | before merge | all jobs in 2.1 green; diff coverage >= 80%; no testguard violation; no protected-path change without code-owner approval; no new `t.Skip` without issue; PR checklist complete (file 12) |
| **G-MAIN** | `main` health | nightly jobs green over the last 3 nights for gate categories (L4 full, L5 full, soak-nightly, HIL nightly, security-nightly, fuzz-nightly); red gate category freezes merges except fixes for it |
| **G-RC** | candidate selected | R1 to R10 green, **zero** waived failures, fault coverage table complete, evidence pack generated and signed |
| **G-SHIP** | per customer | G-RC for the digests being installed **plus** FAT passed on the customer's configuration (section 5), the customer-specific checklist (section 4) complete, known-issues list reviewed with the customer |

Gate numbers, not feelings: a gate has a script exit code. A human can *decline* to ship a build that passes a gate, but cannot ship one that fails it. An override of a failed gate requires a written entry in the release record naming the failed criterion, the risk accepted, and the approver; the evidence pack prints it.

## 4. "Ships to a customer" acceptance checklist

This is the one list that must be true before any customer receives a Sky Ranger system. Items are machine-checked where possible (`tools/shipcheck` evaluates what it can from CI results and prints the rest as a manual checklist). Every item links to evidence (a CI run, a file hash, a signed record).

### 4.1 Build and release integrity

- [ ] SHIP-01 Installed images are exactly the digests tested in G-RC (`shipcheck` compares installed digests with the release record).
- [ ] SHIP-02 Images and the RAUC bundle (if P2) are signed; signature verified on the target box.
- [ ] SHIP-03 SBOM for every image attached and archived with the release record.
- [ ] SHIP-04 No test artifacts, test accounts or debug interfaces in the shipped images (image hardening assertions, file 09 section 4.1).
- [ ] SHIP-05 Release notes list every change, breaking changes, and migration steps; known issues list is complete and sent to the customer.

### 4.2 Evidence for this build

- [ ] SHIP-10 All G-RC gates green: R1 to R10, no waived failure.
- [ ] SHIP-11 72 h release soak passed on the reference box with these digests (S-1 to S-14).
- [ ] SHIP-12 1,000 power cuts on **the customer's hardware SKU and SSD** passed; if the customer's hardware is not a tested SKU, **a SKU qualification run** (1,000 cuts, 24 h soak, watchdog test) is completed first or the hardware is rejected.
- [ ] SHIP-13 Fault catalogue: zero `critical` or `high` faults uncovered or failing.
- [ ] SHIP-14 Security: no open critical/high findings without a valid waiver; pen-test status as required by file 09 section 7.
- [ ] SHIP-15 Supported limits (tags, samples/s, retention) cover the customer's sizing with 2x headroom (file 07); the customer's actual tag list is within the limit.
- [ ] SHIP-16 Evidence pack signed and delivered (file 11).

### 4.3 Customer configuration (FAT)

- [ ] SHIP-20 Customer configuration (YAML) validates with zero errors; export then import gives an empty diff.
- [ ] SHIP-21 FAT passed (section 5.1) on the exact configuration revision to be deployed.
- [ ] SHIP-22 Every alarm on the customer's tag list was triggered and acknowledged in FAT (alarm test sheet).
- [ ] SHIP-23 Every I/O point mapped to the field was checked point-to-point against the source device or a documented simulated stand-in (point list reviewed).
- [ ] SHIP-24 Roles and users reviewed with the customer; default and test credentials removed; passwords or keys generated per site.
- [ ] SHIP-25 Backup configured and a restore drill completed on the FAT system (file 06 backup/restore).
- [ ] SHIP-26 Upgrade and rollback tested from the previous shipped version, if the customer already has one.

### 4.4 Operations readiness

- [ ] SHIP-30 Time sync: chrony configured as documented; NTP source reachable or local source defined; clock monitored.
- [ ] SHIP-31 Alerting on silence is configured (the box tells someone when it stops reporting; deployment/observability doc).
- [ ] SHIP-32 Customer has the runbook: restart, backup restore, replace-a-box, who to call, response times.
- [ ] SHIP-33 Customer understands **scope limits** in writing: Sky Ranger is a monitoring and supervisory system; **it is not a safety system**, and safety interlocks stay in PLCs/hardwired logic. This sentence is in the contract and the evidence pack.
- [ ] SHIP-34 Customer power: UPS present with power-fail signal wired, or the customer has accepted the documented consequences of uncontrolled power loss (the evidence pack's power-cut results apply to a box on the tested supply).

### 4.5 Site Acceptance Test (SAT, after install)

- [ ] SAT-01 Installed digests equal the release record (`shipcheck --site`).
- [ ] SAT-02 Every connection is `connected`; every enabled tag is `good` or has a documented reason.
- [ ] SAT-03 Live value of at least one tag per device compared with a local reading (handheld meter, local display) and recorded with tolerance.
- [ ] SAT-04 Pulled-cable test: unplug one device link; HMI shows the correct state (offline versus stale versus no flow) and recovers on reconnect.
- [ ] SAT-05 Gateway restart (planned): everything recovers without operator action within the documented time; totals continuous.
- [ ] SAT-06 If agreed with the customer: controlled power cut of the gateway; recovery verified.
- [ ] SAT-07 Each alarm test repeated on the real plant where it is safe to do so.
- [ ] SAT-08 Remote access path (if any) tested and its security reviewed with the customer's IT.
- [ ] SAT-09 Operators trained; HMI role behavior demonstrated.
- [ ] SAT-10 Customer sign-off signed; SAT record stored.

## 5. FAT and customer-specific tests

### 5.1 FAT (our bench, before shipping)

Runs on a box of the same SKU as the customer will receive, with the customer's configuration and a simulator standing in for their field devices (or their real devices if reachable on the bench).

| Step | Test | Evidence |
|---|---|---|
| F1 | Import customer config via the API as an engineer; apply; view diff | revision record |
| F2 | Scenario generated from the customer's tag list: each source tag gets a simulated device with plausible values (`tools/fatgen` generates `test/fat/<customer>.yaml`, a `skyranger-test/v1` Scenario). **Scenarios generated for FAT are not in the protected scenario library** but are validated by the same schema | scenario file hash |
| F3 | Run 24 h with `demo-pumphouse`-style faults at low intensity: reboots, comms loss, restarts, a power cut each 6 h | soak judge report (file 07) |
| F4 | Alarm sheet: a script drives each alarm to active, verifies state and message, acks, clears | alarm sheet (HMI screenshots as artifacts) |
| F5 | Operator walk-through with the customer present (video call acceptable) on the real HMI screens | notes |
| F6 | Backup, wipe, restore | before/after hashes |
| F7 | Security checklist for this site: certificates, users, ports, segmentation plan | checklist |

### 5.2 Customer-config static checks (`tools/fatcheck`)

Automatic checks on the customer's YAML beyond schema validation: every alarm has a message and severity; every writable tag has `write_limits`; every `source` tag has `stale_after` or a documented waiver; engineering ranges set for analog tags; units are valid UCUM; history `max_interval` set (gaps distinguishable from flat lines); no tag path collides; no template parameter left default where site-specific (`edge_node_id`); `heartbeat_stale` consistent with the device heartbeat (the template's 150 s for a 60 s heartbeat). Output is a list of findings; unresolved findings need a note in the FAT record.

## 6. Flaky tests and test health

- **Definition**: a test that fails and then passes on the same commit without a code change, or one that fails in the nightly run without a corresponding code cause.
- **Handling**: auto-detect from CI history (a job that failed then passed on the same SHA is recorded). A test failing twice in 14 days becomes a priority-1 ticket. The fix is in the product or the test; **removing or skipping** the test is not a fix (file 12). A temporary quarantine (excluded from gates, still run and reported) requires a human, an issue link and an expiry of at most 14 days; at expiry it either returns to the gate or the issue escalates. The quarantine list is protected and shown on the dashboard.
- **Virtual time first**: most flakiness in this domain is real-time sleeping. The cure is moving the test to L3, not adding retries.

### Test-health metrics tracked on the dashboard

| Metric | Why |
|---|---|
| Time to green on PRs (p50, p90) | developer and agent friction |
| Flaky test count and age | trust in the gates |
| Fault catalogue coverage (required tiers filled) | the headline completeness number |
| Mutation score per critical package | assertion strength (file 12) |
| Coverage floors and their ratchet history | no silent decline |
| Escaped defects (field bugs) and whether a test existed that should have caught them | the honest measure of the strategy |
| Soak slope trends (RSS, latency) across releases | slow regressions |
| Time since last full release soak on current `main` | staleness of evidence |

## 7. Workflow YAML sketch (GitHub Actions; the host is UNVERIFIED, shown to make the structure concrete)

```yaml
name: pr
on: { pull_request: {} }
permissions: { contents: read }
concurrency: { group: pr-${{ github.ref }}, cancel-in-progress: true }
jobs:
  lint:        { runs-on: ubuntu-latest, steps: [ {uses: actions/checkout@v4}, {run: make lint} ] }
  unit:        { runs-on: ubuntu-latest, steps: [ {uses: actions/checkout@v4}, {run: make test-unit} ] }
  contract:    { runs-on: ubuntu-latest, steps: [ {uses: actions/checkout@v4, with: {fetch-depth: 0}}, {run: make test-contract} ] }
  conformance: { runs-on: ubuntu-latest, steps: [ {uses: actions/checkout@v4}, {run: make test-conformance} ] }
  harness:     { runs-on: ubuntu-latest, steps: [ {uses: actions/checkout@v4}, {run: make test-harness} ] }
  testguard:   { runs-on: ubuntu-latest, steps: [ {uses: actions/checkout@v4, with: {fetch-depth: 0}}, {run: make testguard} ] }
  sec-fast:    { runs-on: ubuntu-latest, steps: [ {uses: actions/checkout@v4}, {run: make sec-fast} ] }
  smoke:
    needs: [lint, unit]
    runs-on: ubuntu-latest
    steps: [ {uses: actions/checkout@v4}, {run: make images}, {run: make smoke} ]
    # no retries; artifacts uploaded on failure by the final step
```

Action versions and runner labels are illustrative. Real pipelines pin actions by commit SHA. Lab runners (`lab-perf`, `lab-hil`) are used only by workflows triggered from protected branches, schedules and tags, never from `pull_request` events on forks.

## 8. Make targets (the contract between CI and developers)

`make lint | test-unit | test-contract | test-conformance | test-harness | cover-check | testguard | sec-fast | images | smoke | mutate-diff | golden-update | nightly-l4 | nightly-l5 | soak SOAK_HOURS=6 | perf | hil | fuzz FUZZTIME=5m | release-check | shipcheck`. A developer or an AI agent runs `make pr` locally (the whole 2.1 chain minus lab jobs) before opening a PR. CI runs the same targets, so "works locally" and "passes CI" mean the same thing.

## Decisions made
1. Build once by digest; test and ship exactly those digests. No rebuilds between test and release.
2. Gates are scripts with exit codes; humans may decline but not override a failed gate without a recorded exception that prints in the evidence pack.
3. Four gates: G-PR (every change), G-MAIN (nightly health), G-RC (72 h soak, 1,000 cuts, full security, fault coverage), G-SHIP (G-RC plus customer FAT and checklist).
4. No automatic retries in gating jobs; flaky tests become tickets and quarantine is time-boxed and human-approved.
5. Hotfixes take a reduced but explicit gate, and the evidence pack prints the reduced counts.
6. The pipeline is `make` targets; the CI vendor is replaceable.
7. The "ships to a customer" checklist includes an explicit statement that Sky Ranger is not a safety system.

## Alternatives rejected and why
- **Continuous deployment to customer sites**: the soak and hardware gates take days; SCADA changes at a plant need a maintenance window. Releases are deliberate.
- **Run everything on every PR**: 72 h soak and 1,000 cuts cannot be; the tiering gives feedback in 15 minutes and evidence in 4 days.
- **Retry-until-green for flaky tests**: hides real intermittent defects.
- **Rebuilding artifacts per environment**: breaks the evidence chain.
- **A manual release checklist only**: unrepeatable and unprovable; `shipcheck` automates what can be automated and prints the remainder.
- **Gating on shared-runner performance numbers**: see file 07.

## Open questions
1. CI host and runner topology are assumptions (GitHub Actions plus two self-hosted lab runners). A solo integrator with one lab box means R3 and R5 run serially, about a week of calendar time per RC; a second reference box is probably the single best investment for release speed.
2. `tools/shipcheck`, `tools/fatgen` and `tools/fatcheck` do not exist yet; this file specifies them. FAT scenario generation from an arbitrary customer tag list needs the simulator to model each device type; unknown device types fall back to `generic` generators and that limitation must be recorded in the FAT record.
3. The "known-issues list sent to the customer" assumes a customer-facing channel and process the business side has not defined.
4. Release cadence versus customer maintenance windows is a business decision.
5. Whether a mutation-testing PR gate (2.1) is affordable in 10 minutes on the CI runners is **UNVERIFIED**; the fallback is nightly-only plus the red-green rule.
6. Software-supply-chain provenance levels (SLSA-style) are in 07-security/07; the release job here assumes they exist.
