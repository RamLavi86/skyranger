# CLAUDE.md: Sky Ranger

Instructions for every Claude Code session and sub-agent in this repository. Keep this file short; details live in `docs/`.

## Project
Sky Ranger is an open, modular, edge-first SCADA platform (runs fully on one on-prem box, no internet needed). Author: an embedded Linux engineer and ex-PLC programmer, learning Go. Integrator first, platform later.
First milestone: ESP32 pulse-counter water meter (1 pulse = 1 liter, device publishes the raw cumulative count) plus simulated devices -> Sparkplug B -> tag engine -> history (PostgreSQL + TimescaleDB) -> live web dashboard.
Status: design phase. No code exists yet.

## Languages
- Go: gateway core (`skyrangerd`), drivers, tools. Prefer plain, idiomatic, boring Go; the reviewer is new to Go. Add a `doc.go` to every package (purpose, owner, seams, non-goals).
- Python: user scripting runtime only (milestone M4).
- TypeScript + React: web HMI under `web/`.
- MQTT with Sparkplug B is the device backbone; everything ships as containers.

## Hard rules
1. **Contracts are binding.** `docs/00-contracts/**` (plus `proto/`, `schemas/`, `testdata/contracts/`) define names, types, quality codes, topics and APIs. Never fork a name. Change a contract only through a `docs/decisions.md` entry decided by the human; the doc leads, goldens and code follow. If you think a contract is wrong, keep conforming and record it under Open questions or in `work/questions/`.
2. **Never edit** `docs/decisions.md`, `docs/00-contracts/**`, `CLAUDE.md`, `.claude/**`, `CODEOWNERS`, CI definitions or any protected path in `docs/09-testing/12-ai-agent-guardrails.md` section 2. Write a "Protected path change request" in your PR and stop.
3. **Tests are required with every change.** Bug fixes need a regression test that fails first (`//regression:ID`). Never delete, skip or weaken a test, loosen a tolerance, re-record a golden file, or run `-update` targets. Never compute an expected value with the code under test.
4. **No `time.Sleep` in tests; use `internal/clock`. Seed all randomness.**
5. **Stay in your lane.** Edit only the paths owned by your role (`OWNERS.yaml`, `docs/10-agent-team-and-roadmap/01-agent-team.md`). Need another module changed? File a work package for its owner. Do not change `seam.go` interfaces without a `[seam]` PR.
6. **SCADA is not a safety system.** Never write code, UI text, docs or marketing that claims safety function, SIL, or protection of people or equipment (`docs/07-security/11-safety.md`). Alarms inform operators; they do not protect.
7. **No secrets in the repo**: no keys, tokens, passwords, certificates, customer data or real site configs. Use `secret://` references and `secrets.manifest.yaml` names only. Run gitleaks before PRs.
8. **Edge-first**: nothing may require internet, cloud or a vendor service to run, alarm, record or log in.
9. **Writes are never retried automatically**; a write timeout means "outcome unknown".
10. New dependencies need a justification (what, why not stdlib, licence, maintenance). No GPL/AGPL in shipped binaries.
11. Mark anything about an external standard or tool you have not verified as **UNVERIFIED**.
12. CI output is the evidence, not your summary. Do not claim success without running the commands below.

## Workflow
- One work package (`work/wp/WP-NNN-*.md`) per branch: `agent/<role>/<wp-id>-<slug>`. Read the contract sections it names, the area README, and nothing more than you need.
- Rebase on `main` before opening a PR. Keep PRs small (limit proposed: 800 lines).
- In every PR description: tests added, changed, removed; protected paths touched (should be none); tail of `make pr` output; dependency justification if any.
- Never merge, never force-push. Humans merge.

## Where to look
- Index: `docs/README.md`. Decisions: `docs/decisions.md`.
- Contracts: `docs/00-contracts/` (tag model, messages, driver interface, API).
- Areas: `docs/01-tag-engine` .. `docs/09-testing`; team, roadmap, layout: `docs/10-agent-team-and-roadmap/`.
- Test strategy and agent guardrails: `docs/09-testing/README.md`, `docs/09-testing/12-ai-agent-guardrails.md`.
- Water meter acceptance scripts: `docs/01-tag-engine/water-meter-e2e.md`, `docs/04-alarms/11-water-meter-e2e.md`, `docs/05-hmi-api/09-water-meter-slice.md`.

## Build and test commands (TODO: nothing below exists yet)
All commands are planned names from `docs/09-testing/10-ci-and-quality-gates.md` section 8. Replace each TODO when the target exists.
- Go module path: TODO (placeholder `skyranger/...`)
- Go version: TODO
- Generate code: `make generate`  # TODO not implemented
- Lint: `make lint`  # TODO not implemented
- Unit tests: `make test-unit`  # TODO not implemented
- Contract and golden tests: `make test-contract`  # TODO not implemented
- Driver conformance: `make test-conformance`  # TODO not implemented
- Component harness (L3): `make test-harness`  # TODO not implemented
- Test integrity: `make testguard`  # TODO not implemented
- Everything required before a PR: `make pr`  # TODO not implemented
- HMI: `npm ci && npm test --workspaces` in `web/`  # TODO not implemented
- Demo stack: `make demo`  # TODO not implemented
