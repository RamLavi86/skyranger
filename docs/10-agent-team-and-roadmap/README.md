# 10 Agent Team and Roadmap

Status: design draft v0.1. Conforms to [../00-contracts/](../00-contracts/README.md) (binding) and [../decisions.md](../decisions.md). This is the integration layer: it takes the nine design areas and turns them into who builds what, in what order, in which directories, with which checks.

Sky Ranger will be implemented mostly by Claude Code sub-agents, supervised by one human who is an embedded Linux engineer and ex-PLC programmer, new to Go, working as an integrator first and a platform builder later. Two consequences shape everything here:

1. **Work splits along module boundaries, contracts first.** An agent never needs to read another agent's code to do its job; it reads the contract and the tests. The contract documents in `docs/00-contracts` and the golden corpus are the handover medium.
2. **The human is the scarce resource.** The plan limits parallelism to what one reviewer can verify (see [01-agent-team.md](01-agent-team.md) section 6) and sends human attention to the few places where it matters: contract changes, protected paths and milestone gates.

## Files

| File | Covers |
|---|---|
| [01-agent-team.md](01-agent-team.md) | The roster of sub-agents, ownership by directory and package, may and may not touch, handover through contracts and seams, drift prevention, review gates, cross-verification, parallel running, the contract-change procedure |
| [02-roadmap.md](02-roadmap.md) | Milestones M1 (vertical slice, in waves), M2 alarms and operator actions, M3 second driver, M4 scripting, M5 security hardening, M6 pilot customer. Each with scope, exit criteria drawn from [../09-testing](../09-testing/README.md), and what is deliberately left out |
| [03-repo-layout.md](03-repo-layout.md) | The proposed Go monorepo: `cmd/`, `pkg/`, `internal/`, `drivers/`, `web/`, `proto/`, `schemas/`, `simulator/`, `deploy/`, `firmware/`, `test/`, `docs/`, and how the area docs' paths were reconciled |

Read in order: this README, 03 (where things live), 01 (who touches what), 02 (when).

## How this area relates to the others

```
 00-contracts ──► 03-repo-layout (paths) ──► 01-agent-team (ownership) ──► 02-roadmap (order)
 09-testing/12 (guardrails: protected paths, testguard) ───────┘
 02-drivers/implementation-order ──► roadmap M1 and M3 (driver phases)
 06-scripting/11-phased-plan ──► roadmap M4
 07-security/14-release-checklist ──► roadmap M5 and M6 gates
```

This area adds no new product behaviour and changes no contract. Where it found a conflict between area docs (package paths, test locations) it picked one and listed the other as an open question for the owning doc; see [03-repo-layout.md](03-repo-layout.md) section 4.

## Summary of key decisions

1. **Eighteen narrowly scoped roles (fifteen active in M1, at most four implementers running at once)**, each owning a list of directories; a reviewer agent that cannot edit; a contract steward that can only propose. Agents own packages, never "features".
2. **Three tiers of contract.** Tier 1: `docs/00-contracts`, `proto/`, `schemas/`, golden corpus (human decides). Tier 2: Go seam interfaces between packages (owned by the consuming package; both sides and the reviewer must agree to change). Tier 3: everything inside a package (owner decides).
3. **Tests first across an agent boundary.** For non-trivial components the test author is a different agent from the implementer ([../09-testing/12-ai-agent-guardrails.md](../09-testing/12-ai-agent-guardrails.md) section 9.1).
4. **Merge is gated by CI, not by agent reports.** `make pr` locally, the same targets in CI, testguard, then reviewer agent, then human for protected paths and milestone gates.
5. **Roadmap in waves with exit criteria taken from the test strategy**, no calendar dates (no velocity data exists yet).
6. **Milestone order: slice, alarms, second driver, scripting, security hardening, pilot.** The release checklist in [../07-security/14-release-checklist.md](../07-security/14-release-checklist.md) gates any customer delivery, so M5 precedes M6. Per-device credentials start in M1 because device identity is expensive to retrofit; TLS everywhere waits for M5.

## Decisions made
1. The area has four files: an index, the team, the roadmap and the repo layout.
2. This area records no decisions in `docs/decisions.md`; it proposes that the human adds entries for the items under Open questions, because agents must not edit that file.

## Alternatives rejected and why
- **One general-purpose "developer" agent**: it would hold the whole codebase in context, drift from contracts silently and write tests that fit its code.
- **One agent per feature slice (water meter, alarms, dashboard)**: slices cut across every package, so parallel agents would collide in the same files. Package ownership gives clean seams.
- **Putting the roadmap in `docs/decisions.md`**: decisions.md is human-owned and short; a roadmap changes every milestone.

## Open questions
- The human should add `decisions.md` entries for: agent roster and tiers (01), milestone order (02), the path reconciliations in 03 section 4.
- Whether the human can review protected-path changes at the pace of M1 wave 1 (four or five agents in parallel). The plan assumes yes with batching; see 01 section 6.
- Calendar dates are intentionally absent. Revisit after M1 when there is velocity data.
