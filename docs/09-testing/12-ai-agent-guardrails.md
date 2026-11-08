# 12 Keeping AI Coding Agents Honest

Much of Sky Ranger will be written by Claude Code with sub-agents. Agents are fast, tireless and, under pressure to "make the tests pass", capable of taking the cheapest path: deleting a failing test, loosening a tolerance, re-recording a golden file, adding `t.Skip`, mocking the thing under test, or writing a test that asserts what the code already does. They also drift: two agents working in parallel quietly diverge from a contract. The tests are the specification agents cannot argue with, but only if the pipeline makes weakening them visible and hard.

This file defines the controls. Principle: **trust the pipeline, not the agent's report.** An agent saying "all tests pass" is a claim; CI output is evidence.

## 1. Threat model of agent failure modes

| ID | Failure mode | Example | Primary control |
|---|---|---|---|
| AG-1 | Delete or skip a failing test | removes `TestRollover`, adds `t.Skip("flaky")` | testguard diff check, protected paths, skip needs an issue |
| AG-2 | Weaken a test | loosens `within`, removes an assertion, widens a tolerance, `cmp.Diff` replaced with `len(x) > 0` | testguard assertion-count and tolerance checks, mutation testing |
| AG-3 | Re-record a golden file or snapshot to match new (wrong) output | `go test -update` in the PR | golden update only via explicit flag, protected path, contract-doc coupling |
| AG-4 | Test that cannot fail | empty body, `assert.True(true)`, asserts on a mock it just set | mutation testing, red-green proof, lint |
| AG-5 | Oracle copied from the implementation | expected value computed by calling the code under test | oracle-independence import rule |
| AG-6 | Contract drift | field renamed, quality code changed, topic altered | golden corpus, `buf breaking`, OpenAPI diff, contract-traceability report |
| AG-7 | Test passes by sleeping or ordering luck | `time.Sleep(2*time.Second)` then assert | lint ban, virtual clock rule, `-shuffle`, `-race` |
| AG-8 | Fix without regression test | bug fixed, nothing proves it | red-green proof |
| AG-9 | Over-broad change hiding a weakening | 3,000-line PR with one removed assertion | testguard counts per file; PR size limit |
| AG-10 | New dependency or insecure pattern added casually | adds a library, `InsecureSkipVerify` | dependency justification, security lint (file 09 section 6) |
| AG-11 | Claims success without running tests | "should work now" | CI is the only authority; PR template requires command outputs; agent self-report is not trusted |
| AG-12 | Edits shared fixtures/simulator to hide a product bug | changes `pkg/sim` so the device "never reboots" | `pkg/sim` oracle code is protected; scenario library protected |
| AG-13 | Sub-agents contradict each other | one changes the quality enum, another builds on the old | single contract source, contract change process, merge-queue re-run |

## 2. Protected paths

Changes under these paths need **human code-owner approval**, always, including from an agent-authored PR. Enforced with branch protection plus `CODEOWNERS` (GitHub semantics; host UNVERIFIED, equivalent controls exist elsewhere).

```
# CODEOWNERS (protected paths; owner = a human maintainer)
/testdata/contracts/**               @maintainer
/test/scenarios/**                   @maintainer
/test/faults/faults.yaml             @maintainer
/test/evidence/claims.yaml           @maintainer
/test/coverage-floors.yaml           @maintainer
/test/perf/thresholds.yaml           @maintainer
/test/mutation-allow.yaml            @maintainer
/test/security/waivers.yaml          @maintainer
/test/soak/judge.yaml                @maintainer
/test/soak/expected-errors.yaml      @maintainer
/test/integration/tolerances.go      @maintainer
/test/replay/**/*.expected.yaml      @maintainer
/hmi/e2e/allowed-console.txt         @maintainer
/pkg/driver/conformance/**           @maintainer
/pkg/sim/**                          @maintainer      # oracle and generators
/tools/testguard/**                  @maintainer
/tools/soakjudge/**                  @maintainer
/tools/evidencepack/**               @maintainer
/.github/workflows/**                @maintainer      # CI definitions (host UNVERIFIED)
/Makefile                            @maintainer
/docs/00-contracts/**                @maintainer
/docs/decisions.md                   @maintainer
/CODEOWNERS                          @maintainer
```

Not protected: ordinary unit tests next to code, new tests (adding is encouraged and unrestricted), new scenarios in a `test/scenarios/proposed/` folder (promoted by a human).

An agent that needs a protected change **stops and asks**: it writes the proposed change and the reason in the PR description under "Protected path change request" and does not merge. This is the control that prevents silent weakening; a human reviews these diffs line by line, and they are few.

## 3. `tools/testguard`: the test-integrity checker

Runs in every PR (`make testguard`), comparing the PR head against the merge base. Deterministic, no AI involved (a checker that is itself an agent can be talked out of a finding).

| Check | Rule | Fail when |
|---|---|---|
| TG-01 Removed tests | Go AST: set of `Test*`, `Benchmark*`, `Fuzz*` functions and subtest names in `_test.go` before and after | A test function disappeared (rename is detected by body similarity and must be declared with `//testguard:renamed OldName` in the PR) |
| TG-02 Assertion count | Per test function: count of `t.Error*`, `t.Fatal*`, `cmp.Diff` uses, helper assertions (`h.Assert*`, `c.Tag*`) | Count dropped for a pre-existing test without declaration |
| TG-03 Skips | New `t.Skip*`, `testing.Short()` guards, `//go:build ignore`, `.skip`/`.only` in Vitest/Playwright, `test.fixme` | Any new one without an issue reference in the message (`t.Skip("SR-123: reason")`) |
| TG-04 Tolerances | Numeric literals in `tolerances.go`, scenario `within`, `expect` blocks, soak and perf thresholds | A tolerance loosened (value increased or window widened) |
| TG-05 Golden changes | Any change under `testdata/contracts/` or `*.golden`/`*.expected.yaml` | Not accompanied in the same PR by a change under `docs/00-contracts/` (or a new `decisions.md` entry) that explains it. **The doc leads, the golden follows.** Plus protected-path approval |
| TG-06 Snapshot update mode | CI environment runs tests; scans test code and Makefile for `-update` flag use in non-golden-update targets | `-update` or `UPDATE_SNAPSHOTS` in a CI path |
| TG-07 Coverage floors | `test/coverage-floors.yaml` diff | Any floor lowered |
| TG-08 Scenario expectations | `expect` entries in scenario YAML | An `expect` removed, or `equals_oracle` replaced by a literal value |
| TG-09 Oracle independence | Import graph: `_test.go` and `pkg/sim/oracle`, `pkg/replay/audit`, `pkg/driver/conformance` must not import the production packages that compute the thing they check (`pkg/tag/derived` for conservation oracles) | Forbidden import (a `depguard`-style rule with a project allowlist) |
| TG-10 Fuzz crashers | New file under `testdata/fuzz/` | n/a (adding is fine); fix PRs for a fuzz-found bug must add one (file 09) |
| TG-11 Forbidden patterns in tests | grep/AST: `time.Sleep` in tests outside `internal/clock` and L4 helpers; `assert.True(t, true)`; empty test bodies; tests with no assertion and no call into the code under test; `t.Run` loops ignoring results | any |
| TG-12 Directives | `//fault:F-...` and `//contract:file.md#section` directives reference IDs that exist; each catalogue fault and each contract section has coverage listed | unknown ID (typo), or a *decrease* in coverage count for any fault or contract section |
| TG-13 PR size | Lines changed excluding generated and test data | Over limit (proposal: 800 lines) without a `large-change-approved` label by a human. Large PRs hide weakening |
| TG-14 Dependencies | `go.mod`, `package.json`, `requirements*.txt` diff | A new dependency without a `Dependency justification:` section in the PR description (what, why not stdlib, license, maintenance) |
| TG-15 Insecure patterns | File 09 section 6 lint list | any |

```go
// tools/testguard/removedtests.go
func CheckRemovedTests(base, head *token.FileSet, baseFiles, headFiles []*ast.File, declared map[string]string) []Finding {
    before := collectTests(baseFiles) // name -> {assertions int, bodyHash string, file string}
    after := collectTests(headFiles)
    var out []Finding
    for name, b := range before {
        a, still := after[name]
        switch {
        case !still:
            if newName, ok := declared[name]; ok && after[newName].bodyHash != "" && similarity(b.body, after[newName].body) > 0.8 {
                continue // declared rename with similar body
            }
            out = append(out, Finding{ID: "TG-01", Test: name, File: b.file, Msg: "test removed"})
        case a.assertions < b.assertions:
            out = append(out, Finding{ID: "TG-02", Test: name, File: a.file,
                Msg: fmt.Sprintf("assertions dropped %d -> %d", b.assertions, a.assertions)})
        }
    }
    return out
}
```

`testguard` has its own test suite: each rule has a planted-violation fixture that must be caught (it lives in a protected path; a guard whose tests can be edited is not a guard).

Waivers: a rule can be waived for one PR by a human adding a `testguard-waiver: TG-02 <reason>` line in the PR **by a code owner**, recorded in the PR, and counted in the release test-integrity report (file 11 section 5). Agents cannot self-waive; the waiver check requires the commit author of the waiver text to be a human code owner.

## 4. Required tests per kind of change

The PR template asks the author to select the change type; `testguard` and reviewers check the evidence. "N/A" needs one line of justification.

| Change | Required tests |
|---|---|
| **Bug fix** | A regression test that **fails on the base commit and passes on the head** (red-green proof, section 5). For field bugs, a replay capture (file 05) where evidence exists |
| New driver | Passes conformance suite in both hostings (file 02); `ValidateAddress` fuzz target; protocol fuzz target; fault tests for F-NET-*, F-VAL-* applicable to it; address grammar golden file |
| New or changed derived function (`pkg/tag/derived`) | Table tests incl. every branch of the contract algorithm; property test with oracle independent of the implementation; golden for any serialized output; scenario in `test/scenarios/proposed/`; mutation score above threshold on the package |
| New or changed field in config, API, events, WS | Docs/contract change first (`docs/00-contracts`), then golden file, schema, `buf breaking`/OpenAPI diff clean or major bump, unknown-field tolerance test, export/import round trip |
| New REST endpoint | OpenAPI entry, permission entry (SEC-A1 matrix), audit behavior test (SEC-A8), request-body fuzz coverage (FZ-10), error format test (RFC 9457), pagination test if a list |
| New alarm behavior | State machine table test incl. shelving and delays on the fake clock; scenario with timeline; HMI test if visible; alarm sheet entry |
| New fault handling (a new recovery path in code) | A catalogue entry in `faults.yaml` (human-approved) and a test tagged `//fault:`; if no entry exists, one is proposed in the PR |
| New parser or decoder | Fuzz target with golden seeds, round-trip property test |
| Concurrency change (locks, goroutines, channels) | Race-enabled test that exercises it; goroutine leak check (`goleak`) in the package's `TestMain`; soak-impact statement in PR |
| Performance change | Benchmark before/after with `benchstat` output pasted; no correctness test weakened |
| HMI component | Vitest component test; a11y check; `data-testid`/`data-quality` hooks; if it displays quality or alarms, an e2e test (file 06) |
| Dependency change | Justification, license, scan results, lockfile diff |
| Refactor with "no behavior change" | No test changes at all (a refactor that edits tests is not a refactor, TG-01/02 enforce) |
| Docs only | `make lint` (YAML examples validated) |

## 5. Red-green proof

For any PR labeled bug fix or any PR adding a test file, CI executes:

1. Check out the **base** commit's production code (everything except the PR's `_test.go` files, golden files and new test data).
2. Run only the tests added or changed by the PR.
3. **Require failure** for those tests tagged as regression tests (`//regression:SR-123` directive above the test function).
4. Check out head, run the same tests: require pass.

```make
# Makefile (sketch)
red-green:
	@tools/testguard redgreen --base $(BASE_SHA) --head HEAD --directive regression
```

Output states per test: `RED on base, GREEN on head: proof accepted` or `GREEN on base: this test does not detect the bug`. The second outcome fails the PR. A bug fix with no `//regression:` test fails TG (change type bug fix requires at least one). Limits: a test that fails on base for the wrong reason (compile error) is rejected by requiring the failure to be a test failure, not a build failure (the checker reads the `go test -json` events).

## 6. Mutation testing

Coverage says code ran, not that anything checked the result. Mutation testing changes the code (flip `<` to `<=`, swap `+` and `-`, delete a statement, return zero value) and checks that at least one test fails. A surviving mutant is a place where an agent (or human) could break the product and nothing would notice.

| Language | Tool (candidates; maintenance status UNVERIFIED, check before adopting) |
|---|---|
| Go | `gremlins` (go-gremlins) or `go-mutesting` (avito-tech). Spike both on `pkg/tag/derived` and pick one |
| TypeScript (HMI) | Stryker Mutator (a real, widely used tool) |
| Python (scripting runtime) | `mutmut` or Cosmic Ray |

Where and when:

| Scope | Packages | Trigger | Threshold (initial proposal, calibrate on baseline) |
|---|---|---|---|
| Critical packages | `pkg/tag/derived`, `pkg/tag/engine` (deadband, quality propagation, ts selection), `pkg/alarm`, `pkg/driver/sdk` (coalescer, backpressure), `internal/sparkplug` (seq, alias, bdSeq logic), `pkg/config` (apply, diff) | nightly full run; per PR on **changed lines** only | mutation score >= 85% (killed / (killed + survived), excluding timeouts and not-covered), ratchet upward |
| HMI quality and alarm rendering | components that display quality, alarm states | nightly | >= 75% |
| Everything else | all | weekly, report only | trend |

- Surviving mutants in critical packages must be **killed by a new test or listed** in `test/mutation-allow.yaml` (protected, one line each with `file:line:mutator` and a reason such as "equivalent mutant: log message only"). The allowlist is reviewed by a human; the number of entries is on the dashboard.
- Equivalent mutants are a known limitation of mutation testing: the 85% threshold is not 100% on purpose.
- Mutation runs are slow; per-PR runs use diff-mode (changed lines) and test selection by package. If this exceeds budget, per-PR becomes nightly-only (file 10 open question), and the red-green rule carries the PR-time load.
- Mutation results feed the evidence pack (file 11 section 2, item 2).

## 7. Property-based and oracle tests as the anti-gaming layer

Property tests (rapid, file 01 section 3) are hard to game because the agent does not choose the inputs. Rules:

- Headline properties (counter conservation, quality propagation, coalescer behavior, WS `seq` contiguity, export/import identity) are **protected** in the sense that an agent cannot reduce iteration counts (`-rapid.checks`, default and CI values are set in the Makefile, protected) or narrow generators.
- Generators draw from the **contract's** input space: all six datatypes, int64 extremes, rollover, duplicate and out-of-order samples. A generator narrowed to "friendly" values is a TG-02-class finding (reviewers check generator diffs).
- Oracles are written from the contract, in packages that cannot import the implementation (TG-09). This is why the simulator oracle is protected (`pkg/sim`).
- Every property test failure prints the seed and the shrunk counterexample; the counterexample is committed as a regression table row by the fixing PR.

## 8. Contract traceability (anti-drift)

Because parallel sub-agents drift, each contract section has a stable address (`tag-model.md#7`), and tests declare what they cover:

```go
//contract:tag-model.md#7
//fault:F-CNT-01
func TestCounterTotalRollover(t *testing.T) { ... }
```

`tools/testguard contracts` prints a coverage table: for each section of `docs/00-contracts/*.md`, the number of tests referencing it, and `0` is flagged. It is advisory per PR (coverage must not *decrease*), and the release gate requires every normative section of the five contract documents to have at least one test. Plus:

- Generated code staleness: `make generate && git diff --exit-code` for protobuf and TS enums.
- `buf breaking`, OpenAPI diff and the golden corpus are checked in every PR (file 10), so a rename by agent A is red before agent B builds on it.
- The merge queue re-runs the full PR pipeline on the merged result, catching semantic conflicts between simultaneously green PRs.
- **Contract change procedure** (an agent may propose, a human decides): (1) PR changes `docs/00-contracts/*` and adds a `decisions.md` entry (human-owned); (2) goldens and schemas follow in the same or a stacked PR; (3) all consumers updated; (4) `buf breaking`/OpenAPI diff acknowledged with a major bump if breaking. An agent that thinks a contract is wrong writes it under **Open questions** in its own doc and continues to conform (the instruction given to every area agent).

## 9. Agent operating rules

### 9.1 Team structure that reduces gaming

1. **Separate the test author from the implementer.** For anything non-trivial (derived functions, drivers, alarm logic), one sub-agent writes tests from the contract *before* the implementer starts, in a PR of its own; the implementer then makes them pass and may add tests but may not edit the author's. A single agent that writes both tests and code tends to write tests that fit the code.
2. **Reviewer agent** (separate context) reads the testguard report, the mutation survivors and the diff, and reports weakening, but it only advises: a human owns protected-path approval.
3. **The human reviews tests more carefully than code.** Review time goes first to diffs under protected paths, then to new tests, then to production code.
4. Sub-agents get the **same CLAUDE.md rules** (below) and may not modify `CLAUDE.md`, hooks or settings (those are protected files too).

### 9.2 Rules for the project `CLAUDE.md` (draft text)

```markdown
## Testing rules (non-negotiable)
- Never delete, skip, or weaken a test to make it pass. If a test seems wrong, stop and explain why in the PR; a human decides.
- Never edit files under protected paths (see CODEOWNERS). Propose changes in the PR description instead.
- Never run `go test -update` or regenerate golden files unless the task explicitly says so AND docs/00-contracts was changed first.
- Fixes need a regression test that fails before your change (//regression:ID). Show the failing output.
- Do not compute expected values using the code under test. Derive them from the contract or the simulator oracle.
- No time.Sleep in tests; use the injected clock. No global math/rand; seed explicitly.
- Run `make pr` and paste the tail of its output in the PR description. CI is the authority, not your summary.
- Before finishing: list every test you added, changed, or removed, and every protected path you touched (should be none).
- If you believe a contract in docs/00-contracts is wrong: keep conforming, and record the issue under Open questions.
```

### 9.3 Tool-level enforcement (belt and braces)

Claude Code supports permission rules and hooks in its settings. A `PreToolUse` hook (a script run before each tool call that can block it) can refuse edits to protected paths and refuse `-update` commands. The exact configuration schema and exit-code semantics must be checked against current Claude Code documentation, **UNVERIFIED** here:

```jsonc
// .claude/settings.json (sketch; check schema against current docs)
{
  "hooks": {
    "PreToolUse": [
      { "matcher": "Edit|Write|MultiEdit",
        "hooks": [ { "type": "command", "command": ".claude/hooks/block-protected.sh" } ] },
      { "matcher": "Bash",
        "hooks": [ { "type": "command", "command": ".claude/hooks/block-update-flags.sh" } ] }
    ]
  }
}
```

```bash
#!/usr/bin/env bash
# .claude/hooks/block-protected.sh: reads the tool call JSON on stdin, extracts the file path,
# matches it against CODEOWNERS protected globs, and exits non-zero with an explanation.
```

This is a convenience that stops honest mistakes early. It is **not** the control, because a hook can be edited or bypassed by a determined agent: the control is server-side branch protection and CODEOWNERS, which an agent cannot change.

### 9.4 PR template (agent-aware)

```markdown
## Change type
- [ ] bug fix (//regression test attached)  - [ ] feature  - [ ] refactor (no test edits)  - [ ] docs/contract
## Tests
- Added: ...   Changed: ...   Removed: ... (must be empty unless explained)
- Protected paths touched: none | list + reason
## Evidence
- `make pr` output tail: ...
- Red-green proof job link: ...
- Mutation survivors in changed lines: none | list
## Dependency justification (if any)
## Contract impact
- docs/00-contracts changed: yes/no; golden updated: yes/no (must match)
```

CI checks that mandatory sections exist; an empty PR template fails.

## 10. Measuring whether this works

Tracked monthly on the test-health dashboard (file 10 section 6):

| Metric | Reading |
|---|---|
| testguard findings per 100 agent PRs, by rule | Rising TG-01/02/03 rates mean agents are under pressure to weaken tests; look at the task framing |
| Protected-path change requests and their disposition | Many approvals of loosening would mean the tests were wrong too often |
| Red-green proof failures | Agents writing tests that do not detect the bug |
| Mutation survivors on changed lines | Assertion strength of new code |
| Escaped defects with a test that should have caught them | The honest outcome measure |
| Time humans spend on protected-path review | Must stay small; if large, the protected set is too broad |

**Periodic audit**: quarterly, a human seeds three deliberate bugs into critical packages (a known mutation outside the tool's mutator list) and checks that CI goes red. If any stays green, the missing test is added and the lesson recorded. Also quarterly: an adversarial exercise where an agent is asked to "make the build green" on a branch with a planted failing test; the pipeline must refuse any path that weakens a test.

## Decisions made
1. Protected paths (golden corpus, scenarios, conformance, simulator oracle, fault index, thresholds, claims, CI definitions, contracts) require human code-owner approval; enforcement is server-side, not a hook.
2. `tools/testguard` is deterministic, AST-based, and checks removed tests, dropped assertions, new skips, loosened tolerances, golden changes without a contract-doc change, oracle independence, PR size, and new dependencies.
3. Every bug fix needs a regression test proven red on the base commit and green on the head.
4. Mutation testing with ratcheting thresholds guards critical packages; surviving mutants are killed or allow-listed by a human.
5. Test authors and implementers are separate agents for non-trivial components, and the human reviews tests before code.
6. Contract sections are traced by `//contract:` directives; the doc leads and the golden follows, never the reverse.
7. The agent's own report is never evidence; CI output is.

## Alternatives rejected and why
- **Trusting agent self-review or an LLM-based test-weakening detector as the gate**: an LLM check can be persuaded; deterministic AST checks cannot. LLM reviewers are advisory only.
- **Banning agents from writing tests**: tests written by a different agent from the contract are among the best uses of agents.
- **Relying on coverage percentage alone**: easily gamed by tests with no assertions.
- **Hooks as the only enforcement**: bypassable and local; branch protection is the real wall.
- **100% mutation score requirement**: equivalent mutants make it unattainable and invite allowlist abuse; 85% with review is practical.
- **Freezing all tests**: would stop legitimate improvement; the model is "add freely, weaken only with a human".

## Open questions
1. Which Go mutation tool is maintained and fast enough on our packages (UNVERIFIED); spike needed. If neither is acceptable, a small in-house mutator for the five or six critical packages is feasible but costs time.
2. Whether the per-PR diff-mode mutation run fits in 10 minutes; otherwise nightly-only.
3. Claude Code hook and permission configuration details need verification against current documentation before they are relied on, and they remain defense in depth.
4. Whether a solo maintainer reviewing all protected-path diffs is sustainable at agent PR volume. A mitigation is batching protected-path requests into a weekly review, with agents blocked on them meanwhile, at the cost of velocity.
5. "A human code owner" for waivers assumes at least one human reviewer; if the author is also the only owner, the independence of waiver approval is weaker. A second reviewer (a peer SCADA engineer) for protected paths is worth finding.
6. The rename-detection heuristic (body similarity above 0.8) will produce false positives and negatives; tune on real history.
