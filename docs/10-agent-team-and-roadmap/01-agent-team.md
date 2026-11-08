# 01 The Agent Team

Binding inputs: [../00-contracts/](../00-contracts/README.md), [../09-testing/12-ai-agent-guardrails.md](../09-testing/12-ai-agent-guardrails.md) (protected paths, testguard, red-green proof), [03-repo-layout.md](03-repo-layout.md) (paths). Anything about Claude Code features that was not checked against current documentation is marked **UNVERIFIED**.

The principle from the guardrails file applies throughout: **trust the pipeline, not the agent's report.**

## 1. What an "agent" is here

A sub-agent is a role definition (a markdown file with a prompt and a tool allowlist, kept in `.claude/agents/<name>.md`, format UNVERIFIED against current Claude Code docs) plus a path allowlist (section 3). The human, or a lead Claude Code session acting for the human, dispatches a **work package** (section 4.3) to one agent. An agent that finishes produces a branch and a PR; it never merges.

Every agent loads the same root `CLAUDE.md`. Role files add: owned paths, read-only paths, the contract sections it must conform to, the `make` targets it must run, and the "stop and ask" triggers (section 2.3).

## 2. The roster

### 2.1 Agents and ownership

"Owns" means the agent may create, edit and delete files there. Test files next to code (`*_test.go`) belong to the same agent unless the table says otherwise, except for protected paths (section 2.2).

| Agent | Milestone | Owns (directories and packages) | Reads (does not edit) |
|---|---|---|---|
| `contract-steward` | M1 on | `proto/`, `schemas/`, `gen/`, `web/packages/api-client/src/types.gen.ts`; drafts changes to `docs/00-contracts/**` and `testdata/contracts/**` as proposals for human approval | all docs |
| `foundation` | M1 W0 on | `go.mod`, `go.sum`, `Makefile`, `.github/workflows/`, `internal/clock`, `internal/logging`, `pkg/tag`, `pkg/event`, `pkg/driver` (types only, generated or hand-written to match the contract), `cmd/skyrangerd`, `internal/daemon` (wiring only), `tools/ownercheck` | contracts |
| `test-author` | M1 on | `test/harness`, `test/integration`, `test/compose`, `test/soak`, `test/perf`, `test/hil` (controller code), `test/scenarios/proposed/`, `testdata/fuzz/`; `*_test.go` written in "red" PRs for other agents' packages | contracts, area docs |
| `tag-engine` | M1 | `internal/tagengine/**`, `pkg/config` (types, parse, canonicalize, validate for config kinds), `internal/bus` (internal MQTT mirror) | `pkg/tag`, `pkg/driver`, `pkg/event` |
| `driver-sdk` | M1 | `pkg/driver/sdk/**`, `internal/driverhost`, `drivers/sim`, `cmd/skyranger-driver-sim` | `pkg/driver`, `pkg/driver/conformance` (protected) |
| `sparkplug-driver` | M1 | `drivers/sparkplug`, `internal/sparkplug` (seq, alias, bdSeq, birth/death logic), `cmd/skyranger-driver-sparkplug` | `pkg/driver/sdk`, `proto/third_party/sparkplug` |
| `simulator` | M1 | `simulator/` (Dockerfiles, demo scenarios), `cmd/skyranger-sim`, `pkg/sim` except `pkg/sim/oracle` and generators (those are protected) | contracts, `pkg/tag` |
| `storage` | M1 | `db/migrations/`, `internal/store/**`, `internal/historian/**`, `internal/spool/**` | contracts, `pkg/tag` |
| `api-server` | M1 | `internal/api/**` (REST, WS hub adapters, OpenAPI serving, static HMI hosting) | `schemas/openapi`, `internal/tagengine` seams |
| `security` | M1 minimal, M5 main | `internal/security/**` (authn, authz, audit, secrets, pki), Mosquitto ACL and PKI scripts under `deploy/security/` | threat model docs |
| `hmi` | M1 | `web/apps/**`, `web/packages/**` (except `types.gen.ts`), screen YAML under `web/screens/` | `schemas/`, golden WS transcripts |
| `e2e` | M1 | `web/e2e/**` (Playwright), HMI acceptance tests from [../05-hmi-api/09-water-meter-slice.md](../05-hmi-api/09-water-meter-slice.md) section 7 | HMI source (black-box only) |
| `firmware` | M1 | `firmware/esp32-pulse-meter/**` (human-led; the agent assists) | `messages.md` device profile |
| `alarms` | M2 | `internal/alarm/**` | contracts, tag-engine seams |
| `deploy` | M1 compose, M3 `skyrangerctl` | `deploy/**`, `cmd/skyrangerctl`, `internal/ctl/**` | area 08 docs |
| `scripting` | M4 | `internal/scripthost`, `cmd/skyranger-scriptd`, `scripting/**` (Python) | `proto/skyranger/script/v1` |
| `driver-modbus` | M3 | `drivers/modbus`, `cmd/skyranger-driver-modbus` | `pkg/driver/sdk` |
| `reviewer` | always | **nothing** (read-only tools; writes only PR review comments) | everything |

Notes:
- `security` has a thin M1 role: local users, tokens, `audit_log` writes and per-device MQTT credentials. Hardening, TLS and PKI wait for M5 ([02-roadmap.md](02-roadmap.md)).
- `e2e` and `hmi` are separate on purpose: the agent that writes the screens must not write the tests that judge them. The same holds for `test-author` and every implementer.
- `foundation` owns wiring (`cmd/skyrangerd`, `internal/daemon`) because wiring is the one place every module touches. Modules expose a constructor; `foundation` adds one line per module after the module PR merges. This turns the worst merge-conflict hotspot into a serial, tiny change.
- `firmware` is human-led: the author is an embedded engineer, ESP32 code touches real hardware, and the language and framework (ESP-IDF in C is assumed) are not fixed by any doc (UNVERIFIED assumption).

### 2.2 Protected paths and who may propose

[../09-testing/12-ai-agent-guardrails.md](../09-testing/12-ai-agent-guardrails.md) section 2 lists the protected paths (golden corpus, scenarios, conformance suite, simulator oracle, thresholds, CI definitions, `docs/00-contracts`, `docs/decisions.md`, `CODEOWNERS`). Rule for every agent: **a protected-path change is a request, not a commit.** The agent writes the proposed diff and reason under "Protected path change request" in its PR and stops. Initial creation of a protected file (for example the first golden corpus, the conformance suite, `tools/testguard`) is also human-approved; `contract-steward`, `test-author` and `foundation` draft them.

Additional protected files proposed by this area (human to add to `CODEOWNERS`): `.claude/**`, `CLAUDE.md`, `work/templates/**`, `contracts.lock` (section 5.2), `tools/ownercheck/**`.

### 2.3 "Stop and ask" triggers (in every role file)

An agent stops, records the issue in its PR or in `work/questions/`, and does not continue past the blocked point when:
1. a contract section is ambiguous, contradictory with another area doc, or marked UNVERIFIED and the code depends on it;
2. the work needs a file outside its owned paths;
3. a test seems wrong, or would pass only by weakening an assertion or tolerance;
4. a new third-party dependency is wanted (needs "Dependency justification", testguard TG-14, and a licence check: no GPL/AGPL in shipped binaries, [../02-drivers/libraries-and-licensing.md](../02-drivers/libraries-and-licensing.md));
5. a seam interface (section 4.2) would change.

### 2.4 May and may not, summarized

| Every agent may | Every agent may not |
|---|---|
| Add new tests anywhere in its own paths, freely | Delete, skip or weaken a test, loosen a tolerance, or run golden update targets |
| Add Open questions to the doc of its own area | Edit `docs/decisions.md`, `docs/00-contracts/**`, `CLAUDE.md`, `.claude/**`, `CODEOWNERS` |
| Run any `make` target locally | Merge, force-push, or alter CI definitions |
| Propose a contract or protected-path change in a PR | Touch another agent's owned paths, even for a one-line fix (file a work package for the owner) |
| Read any file | Commit secrets, keys or customer data (see `CLAUDE.md`) |
| Add a dependency with a justification | Add a dependency that adds GPL/AGPL code to a shipped binary |

## 3. Path enforcement

Ownership that is only written in a document drifts. Proposed controls, cheapest first:

1. **Branch naming**: `agent/<role>/<wp-id>-<slug>`. The role in the branch name is the identity.
2. **`tools/ownercheck`** (new, small Go program; protected): reads `OWNERS.yaml` at the repo root (role to path globs), takes the branch's role, and fails CI if the diff touches a path outside that role's globs, except `docs/<own area>/` Open questions. This is the same style of deterministic check as testguard and cannot be argued with. Humans may touch anything.
3. **`.claude/hooks`** blocking edits outside the role's globs while a session runs. Convenience only; the control is ownercheck plus CODEOWNERS (same reasoning as guardrails section 9.3). Hook schema UNVERIFIED.
4. **Git worktrees**, one per running agent (section 7), so an agent physically edits only its own checkout.

`OWNERS.yaml` is generated from the table in section 2.1 once, then maintained by the human.

## 4. Handover through contracts

### 4.1 Tier 1: contracts (human decides)

`docs/00-contracts/*.md`, `proto/**`, `schemas/**`, `testdata/contracts/**`. These are normative. `proto/` and `schemas/` are the machine-readable form of what the contract docs say in prose; `gen/` (Go) and `types.gen.ts` are generated from them and committed (`make generate && git diff --exit-code` in CI, [../09-testing/12-ai-agent-guardrails.md](../09-testing/12-ai-agent-guardrails.md) section 8). Only `contract-steward` regenerates, so two agents never produce conflicting generated output.

### 4.2 Tier 2: seams (consumer owns, both sides agree)

Go packages talk through small interfaces defined by the **consumer** (Go idiom, and already how [../01-tag-engine/go-design.md](../01-tag-engine/go-design.md) works). Each seam interface lives in a file named `seam.go` in the consumer's package, with a header comment naming the producer and the work package that last changed it. Seams in the first milestone:

| Seam (interface) | Consumer defines | Producer implements | Verified by |
|---|---|---|---|
| `driver.Driver`, `driver.Sink` | `pkg/driver` (contract) | `drivers/*` | conformance suite, in-process and over gRPC |
| `core.HistorySink` | `internal/tagengine/core` | `internal/historian` | seam test in `test/harness` with fake sink and real writer |
| `core.StateStore` (derived state, memory values) | `internal/tagengine/core` | `internal/tagengine/checkpoint` over `internal/store` | restart scenarios in the harness |
| alarm mailbox (value changes) | `internal/alarm` (M2) | `internal/tagengine/subs` | water meter alarm scenarios |
| `api.Engine` (read values, write, subscribe) | `internal/api` | `internal/tagengine/engine` | API golden transcripts against a fake engine, then harness |
| `store.*` repositories | consumers (`historian`, `config`, `alarm`, `security`) | `internal/store/pg` | store conformance suite (storage doc 11, not yet written; UNVERIFIED shape) |
| `ConfigApplier` | `internal/tagengine` | `pkg/config` + `internal/store` | export, wipe, import diff empty (smoke S8) |

A seam change is not a Tier 1 change but is more than a Tier 3 edit. It requires: a note in the PR title (`[seam]`), an updated seam test, the other side's agent (or its work package) updated in the same wave, and reviewer-agent sign-off. Seam files are listed by `make seams` (a grep for the header marker) so the human can see them all at once. This is a proposal, not an existing tool.

### 4.3 Work package (WP): the unit of handover

A WP is a markdown file `work/wp/WP-NNN-slug.md` created by the human or lead session from `work/templates/wp.md`. An agent works on exactly one WP per branch.

```markdown
# WP-014 Counter total derived function
Owner agent: tag-engine          Wave: M1-W1          Depends on: WP-003 (clock), WP-007 (golden corpus)
## Goal (one paragraph, in contract terms)
## Contract sections that bind this work
- tag-model.md#7 (counter_total), messages.md#6 (rollover/reset representation)
## Owned paths (must match OWNERS.yaml)
## Interfaces consumed / produced (seams, with file names)
## Tests that must exist and fail first (written by test-author in WP-013)
## Out of scope (explicit)
## Done means
- make pr green; testguard clean; mutation score >= threshold for the package
- PR lists tests added/changed/removed and protected paths touched (none)
```

"Tests that must exist and fail first" is the interface between agents. The usual sequence is `test-author` PR (red, tests only, merged behind a build tag or marked pending in a way testguard accepts: the proposal is `//go:build pending_WP014`, removed by the implementer's PR; UNVERIFIED that this interacts cleanly with testguard TG-03, to be checked when testguard is built) then implementer PR (green).

### 4.4 How work flows between two agents (example)

`sparkplug-driver` needs `pkg/driver` types, the conformance suite and the birth/death contract. It reads `messages.md` and `driver-interface.md`, not `drivers/sim`. Its tests come from: the conformance suite (protected, human-approved), the Sparkplug add-on suite and golden Sparkplug payloads in `testdata/contracts/sparkplug/` (corpus drafted by `contract-steward` from `messages.md`, approved by the human), and a simulator that uses a different MQTT client library. It never needs to know how `tag-engine` consumes samples; the Sink interface says everything.

## 5. Drift prevention

Drift is two agents holding different pictures of the same contract. Controls, in the order they fire:

### 5.1 Before the work
1. The WP names the exact contract sections; role files tell agents to read only those plus the area README.
2. Every agent checks `contracts.lock` (5.2) at start; if it changed since the WP was written, the agent re-reads the diff.

### 5.2 `contracts.lock` (proposed)
A file at the repo root with the SHA-256 of each `docs/00-contracts/*.md`, `proto/**`, `schemas/**` tree and `testdata/contracts/**` tree, plus a monotonic `contracts_version` integer. It is updated only in a contract-change PR (section 8). CI fails if a contract file changed and the lock did not, or the reverse. This extends testguard TG-05 (doc leads, golden follows) and is protected. Each package's `doc.go` records the `contracts_version` it was last verified against, so stale packages are findable by grep.

### 5.3 During the work
3. Tests carry `//contract:file.md#section` and `//fault:F-...` directives; `tools/testguard contracts` prints sections with zero tests ([../09-testing/12-ai-agent-guardrails.md](../09-testing/12-ai-agent-guardrails.md) section 8).
4. Golden corpus tests run in Go, TypeScript and Python from the same files, so a rename made in Go only is red in TS.
5. `buf breaking`, OpenAPI diff and generated-code staleness run on every PR.
6. `ownercheck` rejects edits outside the role.

### 5.4 After the work
7. The merge queue re-runs the full PR pipeline on the merged result.
8. Each wave ends with an **integration pass** by `test-author` on merged `main`: L3 harness scenarios and L4 smoke ([../09-testing/10-ci-and-quality-gates.md](../09-testing/10-ci-and-quality-gates.md)). Two green PRs that are semantically incompatible show up here at the latest.
9. Each milestone ends with a **contract audit** by the `reviewer` agent: every normative sentence in the contracts is mapped to a test or an Open question. Output is a table, reviewed by the human.

### 5.5 When an agent thinks a contract is wrong
Keep conforming. Record the issue under Open questions in the agent's own area doc or in `work/questions/`. This is the standing instruction from the guardrails file and it prevents a locally "fixed" contract that nobody else knows about.

## 6. Review gates

| Gate | Who or what | Checks | Blocks |
|---|---|---|---|
| G0 Self | Implementing agent | `make pr` locally; list of tests added, changed, removed; protected paths touched | PR creation |
| G1 CI | Pipeline | lint, unit, contract, conformance, harness, coverage floors, testguard, ownercheck, sec-fast, images, smoke, mutation on changed lines | merge |
| G2 Reviewer agent | `reviewer` (fresh context, read-only) | testguard and mutation report, seam changes, contract traceability, Go idiom and simplicity, dependency justification, scope creep versus WP | human review (advisory) |
| G3 Human | Maintainer | protected-path diffs line by line first; then new tests; then production code; seam changes; every contract change | merge of those PRs |
| G4 Wave integration | `test-author` + human | merged `main` passes L3 scenarios and L4 smoke; contract audit table for the wave | start of next wave |
| G5 Milestone exit | Human | every exit criterion in [02-roadmap.md](02-roadmap.md) has linked evidence | milestone declared done |

The reviewer is advisory; only G3 and G5 are authority ([../09-testing/12-ai-agent-guardrails.md](../09-testing/12-ai-agent-guardrails.md) decision 2 and 7).

**Reviewing as someone new to Go.** Two measures keep the human effective: (1) every package has a `doc.go` stating purpose, owner agent, seams and non-goals, so review starts from intent; (2) the PR template gets a section "Go constructs a newcomer might not know" where the agent explains channels, generics, `context`, embedding or similar if it used them, and prefers the plain alternative when equal. Reviewer-agent prompt includes "flag cleverness".

**Review load budget**: a wave with more than four concurrent agents produces more protected-path requests than one person can read carefully (guardrails Open question 4). Hence the cap in section 7.

## 7. Running agents in parallel without collisions

1. **One worktree per running agent** (`git worktree add ../sr-<role>-<wp> -b agent/<role>/<wp>`); Claude Code can isolate sub-agents in worktrees (feature details UNVERIFIED), otherwise the lead session creates them.
2. **Disjoint path sets** by construction (section 2.1). Where two agents need the same file the answer is a different file or a seam, never "take turns on it". Shared hot files and their single owner:

| Hot file | Single owner | How others get changes in |
|---|---|---|
| `go.mod`, `go.sum` | `foundation` | dependency request in WP; batched per wave |
| `Makefile`, CI workflows | `foundation`, human approves | request |
| `cmd/skyrangerd`, `internal/daemon` | `foundation` | modules expose `New(...)`; one-line wiring PR after each module merges |
| `gen/**`, `types.gen.ts` | `contract-steward` | contract-change procedure |
| `docs/00-contracts/**`, `decisions.md` | human | proposals |
| `web/package.json` and lockfile | `hmi` | `e2e` requests via WP |

3. **Wave plan** ([02-roadmap.md](02-roadmap.md)): agents run concurrently only when their WPs depend on frozen contracts and not on each other's unmerged code. Dependencies between agents are expressed as "uses seam X from the contract tests", never "uses agent Y's branch".
4. **Concurrency cap**: at most four implementer agents plus `reviewer` and `test-author` at once during M1; two or three after that. Raise only if G3 review time stays small (metric in guardrails section 10).
5. **Rebase discipline**: agents rebase on `main` before opening the PR, and again if `contracts.lock` changed. The merge queue serializes the rest.
6. **Small PRs**: guardrails TG-13 sets the PR size limit (proposal 800 lines). One WP is sized to fit.
7. **No cross-agent chat.** Agents talk through WPs, PR descriptions and `work/questions/`. A lead session may relay; agents do not read each other's scratch files.

## 8. When a contract changes

Contracts will change; M1 will surface gaps (the area READMEs already list dozens of "CONTRACT" open questions). The procedure is the guardrails one, made concrete:

1. **Stop the line for that section.** The finder (any agent) writes `work/questions/Q-NNN.md`: what is wrong, which sections, proposed wording, additive or breaking, who is affected (consumers by agent role). The agent keeps conforming.
2. **Human decides** and adds the `docs/decisions.md` entry (agents never write it). Rejected: closed with reason. Accepted: continue.
3. **`contract-steward` PR**: edit the contract doc, then `proto/`/`schemas/`, then regenerate `gen/`, then update `contracts.lock` and `contracts_version`. The doc leads, the golden follows (TG-05). `test-author` updates the golden corpus in the same or a stacked PR (human approves, protected).
4. **Impact scan**: `contract-steward` lists affected packages by grepping `//contract:` directives and `contracts_version` in `doc.go`. One WP per affected agent is created, labelled `contract-sync`.
5. **Additive change** (new optional field, new enum value consumers must tolerate): consumers update at their own pace within the wave; unknown-field tolerance tests already guarantee nothing breaks.
6. **Breaking change** (rename, removal, semantic change): new major on the affected surface (`v2`), `buf breaking` acknowledged, all consumers updated in one wave, no merges to the affected packages between step 3 and the last consumer PR (the line stays stopped), integration pass (G4) before reopening.
7. **In-flight WPs** that cite the changed section are paused or re-based by their agents; the human decides which to drop.
8. The next milestone retrospective counts contract changes per area; many changes in one area mean its design doc was wrong, and the area README gets corrected.

## Decisions made
1. Eighteen role definitions (fifteen active in M1), but at most four implementers run at once. Ownership is by directory and package; every path in the repo has exactly one owner role (or the human).
2. `test-author`, `e2e` and `reviewer` are separate from implementers. `reviewer` has no write tools.
3. Three contract tiers; seams are consumer-owned `seam.go` files; `contract-steward` is the only agent that regenerates code from `proto/` and `schemas/`.
4. Deterministic enforcement beats instruction: `ownercheck`, testguard, `contracts.lock`, CODEOWNERS and branch protection. Hooks are convenience.
5. Wiring (`cmd/skyrangerd`) is owned by one agent and changed serially, one line per module.
6. WPs are markdown files in `work/wp/`, with the failing tests named up front.
7. Contract changes follow the stop, decide, steward, impact-scan, update-consumers sequence; additive and breaking changes are handled differently.

## Alternatives rejected and why
- **Agents negotiating directly with each other**: unrecorded, unbounded and impossible to review. WPs and questions files leave a trail.
- **Letting each agent regenerate protobuf and OpenAPI outputs**: parallel regeneration yields conflicting generated files and hides drift. One regenerating owner.
- **Trusting the role prompt alone to keep agents in their lane**: prompts are instructions, not controls. Path checks run in CI.
- **Seam interfaces in a central `interfaces/` package**: it becomes a dependency magnet and every change touches everyone. Consumer-owned interfaces keep changes local.
- **Letting `reviewer` approve merges**: an LLM reviewer can be persuaded ([../09-testing/12-ai-agent-guardrails.md](../09-testing/12-ai-agent-guardrails.md) alternatives).
- **Maximum parallelism (one agent per package, all at once)**: more PRs than the one human can review honestly.

## Open questions
1. **Pending-test mechanism** (section 4.3): whether `//go:build pending_WP...` tags interact acceptably with testguard TG-01/TG-03, or whether red tests should live on the test-author's branch and merge together with the implementation PR.
2. **`pkg/sim` protection granularity**: guardrails protect all of `pkg/sim/**`. That makes every simulator change a human review. Proposal: protect `pkg/sim/oracle/**` and generators only. Needs a CODEOWNERS edit by the human.
3. **Claude Code mechanics** (UNVERIFIED): sub-agent definition format and tool allowlists, worktree isolation per sub-agent, whether sub-agents can be restricted to path globs natively, and hook exit-code semantics. Verify against current documentation before relying on any of them.
4. **Single human reviewer**: the author is also the sole code owner. A second reviewer (peer SCADA engineer) for protected paths is recommended by the guardrails file; not arranged.
5. **`firmware` agent**: whether an agent should touch firmware at all before the bench rig ([../09-testing/08-hardware-and-power-cut.md](../09-testing/08-hardware-and-power-cut.md)) can validate its output. Default here: human-led, agent drafts only.
6. **Model choice per role** (cheaper model for mechanical work, stronger for tag engine and alarms) is a cost question not decided here.
7. **`ownercheck` versus testguard**: whether ownercheck is a testguard rule (TG-16) or a separate tool. Either is fine; testguard's own test discipline should apply to it.
