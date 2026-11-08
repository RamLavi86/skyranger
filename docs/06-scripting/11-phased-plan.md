# 11 Phased plan

Status: design draft. Conforms to [../00-contracts/](../00-contracts/README.md) and decisions.md (first milestone is the vertical slice; integrator first; platform later). Phases are ordered by need, with exit criteria that can be tested. Effort numbers are deliberately omitted: no data yet, and invented estimates are worse than none.

## Phase 0: before scripting (the vertical slice needs none)

The first milestone (ESP32 meter, simulated devices, tag engine, history, live dashboard) is fully served by derived-tag built-ins (tag-model.md section 7): `counter_total`, `rate`, `liveness`, `continuous_activity`. **No scripting work starts before the milestone is demonstrably running.** Prerequisites that scripting needs from other areas:

- In-process tag-change and event streams in the tag engine (01-tag-engine) that a second consumer can subscribe to (needed for P1).
- The write service layer callable from Go, shared by REST handlers and the script host.
- The config service able to host additional kinds (P2) with a validation hook.
- `_system` tags generation and the audit log in place.

## Phase v1: minimal, safe scripting for the integrator

Why it exists: the first customer jobs will want a daily report, a webhook on alarm, and a few glue rules that derived tags cannot express. Customers have no engineers who will write Python; the author does.

Scope (all of it, nothing more):
1. `skyranger-scriptd` and the Python bootstrap: one pool `default`, pooled workers, soft timeout plus `SIGKILL`, RSS kill (01).
2. Script host in `skyrangerd`: `skyranger.script.v1`, run tokens, permission and budget enforcement on every call (01, 04).
3. Config kinds `Script`, `ScriptLibrary` with draft validation (03, 06). Pool ceiling file (04).
4. Triggers: `tag_change`, `timer`, `event`, `startup`, `manual` (03).
5. API: `tags`, `history`, `events`, `alarms` (read, plus `ack` behind a permission), `http` (allowlist, secrets via header injection), `state`, `log`, `now` (02).
6. Run records, `_system/scripts/...` tags, three events, `/api/v1/scripts` endpoints (05).
7. Sandbox: container profile, seccomp shim, import guard, audit hook, hostile-script regression suite (04, 07).
8. Testing: `skyranger.testing` wheel, dry run endpoint (07). Simulator T3 tests for the three reference scripts (daily report, leak webhook, reset audit).
9. Docs: script API reference (generated), a "your first script" tutorial using the water meter.

Explicitly **not** in v1: plugins, `ScriptPool` kind, services, durable event delivery, `shutdown` trigger, `kind: script` tags, third-party libraries, `notify.send`, file access, DB access, UI hooks, scripts container in the default compose profile (opt-in).

Contract changes needed before coding: P1 to P7 (README), plus the additive items listed in the files' Open questions (draft `warnings`, `metadata.id`, Gateway retention block).

Exit criteria (all testable):
- The hostile-script suite passes on both amd64 and arm64 images; `socket()` is `EPERM` inside a worker (04).
- With the scripts container killed repeatedly during a soak run, tag latency, history completeness, and alarm timing are unchanged (measured against the L6 soak baseline, 09-testing/07).
- The three reference scripts pass T1, T2, T3.
- Measured numbers recorded for worker cold start, per-call latency, idle RSS per worker on the target edge box (replacing the UNVERIFIED guesses in 01).
- A revision rollback restores a previous script version and the run record of the next run names the rollback revision.
- Run-to-revision trace demonstrated: pick a failed run, retrieve its exact source from the revision.
- STRIDE table (04 section 7) merged into 07-security/01-threat-model.md and reviewed.
- One real customer-style deployment ran the daily report for four consecutive weeks without intervention (field evidence, in line with the integrator-first plan).

## Phase v2: plugins and pools

Trigger to start: at least two real projects completed (decisions.md 5 licensing gate) and a concrete plugin need (a driver the project does not ship, a customer notification channel, or an integrator asking to reuse scripts across sites).

Scope:
1. **Plugin host**: `Plugin` install flow, `.srp` format, manifest, trust store, approval and audit, `/api/v1/plugins` (P8, P9) (09).
2. **First extension point: third-party drivers** (protocol exists), including the conformance container for non-Go drivers. The project's own drivers continue to use the same SDK.
3. **Notification channels** (`skyranger.notify.v1`) with one reference channel (webhook or SMTP) plus alarm-area routing integration.
4. **Script library plugins** (`libs.<plugin>`).
5. **`ScriptPool` as a config kind** (multiple pools, per-pool ceilings, role requirement above `engineer` for ceiling edits), `isolation: run` (fresh process per run) option.
6. **Durable event delivery** (bounded) for scripts that must not miss events during restarts.
7. `notify.send` in the script API (permissioned).
8. Possible: `kind: script` tags (tag-model.md section 2) if integrators ask for historized, alarmable computed values beyond built-ins.
9. Signing tools for publishers (`plugin sign`, `plugin lint`), first-party plugin signed with the project key.

Exit criteria:
- A driver written by someone other than the author (or at least outside the repo, in a separate module using only the public SDK) installs from a signed `.srp`, passes conformance, runs in a container with granted egress only, and survives kill and restart.
- Unsigned, tampered (one byte changed), wrong-publisher, revoked-key, and expired-key bundles are each refused with a distinct audited reason.
- Upgrade and rollback of a plugin under load do not disturb unrelated connections.
- A plugin crash loop produces `faulted` connections and `_system` state, never core instability (soak run).
- Compatibility check blocks a gateway upgrade that would break an installed plugin before cutover.
- Licensing decision recorded in decisions.md (core license, contributor terms) before any external plugin is published.

## Phase v3: public SDK and ecosystem

Trigger to start: external engineers actually want to build (inbound requests, not speculation).

Scope:
1. **SDK as a product**: Go SDK module with the stability promise (10), Python `skyranger` and `skyranger.testing` on PyPI (or vendored), generated reference docs, plugin templates, tutorials.
2. **UI component extension point** with the iframe sandbox protocol `skyranger.ui.v1` and a designer integration (05-hmi-api/04-designer.md).
3. **Service plugins** (`skyranger.service.v1`) if question 2 in 09 is answered yes.
4. Compatibility machinery at scale: support-window enforcement, `config migrate`, CI that runs the previous release's reference plugins against each release.
5. **Verified-publisher tier**: SBOM and build attestations, a project listing (a static signed index file first; a registry service only if needed).
6. Optional isolation upgrade for multi-tenant hosts (gVisor/Firecracker, or CPython-on-WASM if matured) (04, 01).
7. Contribution process: DCO/CLA in force, governance, security disclosure policy, trademark policy.

Exit criteria:
- A new engineer, using only the public docs and templates, builds and installs a driver plugin and a script library plugin in a measured time budget (usability test with at least two people outside the project; set the budget after the first trial).
- The compatibility policy has been exercised once for real: a deliberate v1 to v2 protocol or api_level change with both supported side by side for the stated window.
- External plugin published under the project's listing policy with a signed manifest and SBOM.
- Security disclosure process exercised with at least one test report.

## Dependencies on other areas

| Need | From | Needed by |
|---|---|---|
| In-process subscription API for tag changes and events | 01-tag-engine | v1 |
| Config kinds with validation hook, `warnings`, `metadata.id` | 01/00-contracts | v1 |
| Scripts container in compose, network-none, volumes, opt-in profile | 08-deployment | v1 |
| Seccomp, STRIDE merge, secret resolver for `secret_headers`, audit fields | 07-security | v1 |
| Simulator time control and control API usable by script tests | 09-testing | v1 (T3) |
| Alarm routing and notification config | 04-alarms | v2 |
| Update tool pre-cutover script/plugin validation | 07-security/07-secure-update.md, 08-deployment | v2 |
| Designer integration for UI components | 05-hmi-api | v3 |

## Risks

| Risk | Phase | Mitigation |
|---|---|---|
| Scripting becomes a way to avoid designing derived-tag built-ins or alarm features | v1 | Rule: logic that every site needs becomes a built-in; scripts are for site-specific glue |
| Sandbox holes (seccomp mistakes, library behavior) | v1 | Syscall-level regression tests, patch cadence, opt-in container, pool is the trust domain |
| API surface creep (the "just add one more function" problem) | v1 to v3 | api_level policy, every new function needs a permission story and a test in the fake |
| Support-window costs exceed a solo maintainer | v2, v3 | Short window numbers decided on data; fewer extension points; refuse `service` kind if needed |
| Plugin ecosystem never materializes | v2, v3 | Phase triggers are demand-based; v2 is useful on its own (third-party drivers for the author's own customers) |
| Licensing decision made under pressure | v2 | Gate: no external plugin before the decision (exit criterion) |
| Learning-curve load on the author (Go, gRPC, seccomp) | v1 | Small Go surface: two binaries, one proto; Python bootstrap stdlib-only; seccomp tooling spike first with a go/no-go on the library choice |

## Decisions made
1. Scripting starts after the vertical-slice milestone; v1 is the smallest set that serves the integrator (reports, webhooks, glue) with the full safety envelope, rather than a thin unsafe prototype.
2. v1 excludes everything that widens the platform surface: plugins, pools, services, durable delivery, file/DB access.
3. v2 starts only after two real projects and a concrete plugin need, with drivers first because their protocol already exists; the licensing decision gates the first external plugin.
4. v3 is demand-driven and adds the SDK as a product, UI components, and ecosystem machinery.
5. Every phase has measurable exit criteria, including field evidence for v1 and an outside-author test for v2/v3.

## Alternatives rejected and why
- **Plugins before scripting**: the integrator phase needs scripting now; plugins serve a market that does not exist yet.
- **Ship v1 as an in-process embedded interpreter "to get going" and isolate later**: the isolation boundary is the hard part and cannot be retrofitted without changing the API semantics (latency, no shared memory).
- **A prototype without permissions "just for me"**: permissions shape the whole API (principal, budgets); adding them later changes every call path.
- **Building the SDK first**: a stable SDK needs at least two real consumers (our own drivers and scripts) to know what is stable.
- **Time-boxed estimates in this document**: no velocity data; exit criteria are the honest planning tool.

## Open questions
1. Does a customer actually need scripting before the second customer project? If derived tags and webhooks-in-alarm-config suffice, v1 can wait further.
2. Should v1 include `notify.send` for SMTP/webhook so that alarm notification does not depend on scripts at all (alarm area decides notification design)?
3. Opt-in vs default scripts container: decided with 08-deployment.
4. Which real project provides the four-week field evidence.
5. Whether v2's first third-party driver is built by the author himself as the dogfood case (suggested: yes).
6. Contract proposals P1 to P9: the owner decision session should happen before v1 coding starts, as a single decisions.md pass.
