# 02 Roadmap

Binding inputs: [../decisions.md](../decisions.md) (entries 5 and 6: integrator first; first milestone is the water meter slice), [../09-testing/](../09-testing/README.md) (exit criteria come from here), [../02-drivers/implementation-order.md](../02-drivers/implementation-order.md), [../06-scripting/11-phased-plan.md](../06-scripting/11-phased-plan.md), [../07-security/14-release-checklist.md](../07-security/14-release-checklist.md).

**No calendar dates and no effort estimates in this file.** There is no velocity data yet, and the scripting plan already argues that invented estimates are worse than none. Milestones are ordered by dependency and risk. Effort figures in the driver docs are guesses; recalibrate them after M1.

## 0. Rules for every milestone

1. **Exit criteria are evidence, not opinion.** Each criterion links to a CI run, a generated report or a signed bench record ([../09-testing/11-evidence-pack.md](../09-testing/11-evidence-pack.md)). The human signs the milestone at gate G5 ([01-agent-team.md](01-agent-team.md) section 6).
2. **Contracts before code.** Each milestone starts with a contract pass: the open "CONTRACT" questions that the milestone depends on are decided by the human and recorded in `decisions.md` before the wave that needs them starts.
3. **Fault behaviour is in scope from the first milestone.** A feature without its fault tests is not done ([../09-testing/README.md](../09-testing/README.md) principle 2).
4. **Each milestone ends with a short retrospective** that updates this file, the effort guesses and the metrics in [../09-testing/12-ai-agent-guardrails.md](../09-testing/12-ai-agent-guardrails.md) section 10.
5. **Left-out lists are commitments.** Items under "Deliberately left out" are not started, even if an agent is idle.

## 1. M1: the vertical slice

**Goal** (decisions.md entry 6): one real ESP32 pulse-counter water meter (1 pulse = 1 liter) plus simulated devices flow through Sparkplug B into the tag engine, into history, and onto a live web dashboard, correctly across reboots, rollover and outages.

### 1.1 Scope

| Area | In M1 |
|---|---|
| Firmware | ESP32 publishes raw cumulative `pulse_count`, `boot_count`, heartbeat per the `skyranger.pulse_meter.v1` profile; NBIRTH/NDEATH; NVS-persisted boot count; per-device MQTT credential |
| Drivers | `sim` driver and driver host (implementation-order phase 0); `sparkplug` driver (phase 1) |
| Simulator | `pkg/sim` with the pulse meter oracle; `skyranger-sim` publishing Sparkplug B for N virtual meters, with fault injection from the 04 catalogue subset below |
| Tag engine | Registry, pipeline, `counter_total`, `rate`, `liveness`, `continuous_activity`, subscriptions, history gate. Tags: `total_volume`, `flow_rate`, device health, `leak` condition tag (the tag, not the alarm) |
| Storage | PostgreSQL + TimescaleDB schema and migrations, historian writer with spool, config revisions (apply, export, import), `history_15m` aggregate, retention at defaults |
| API | REST read endpoints, history, config apply/export, discovery approval, WebSocket subscriptions with snapshot-then-update; local users, four roles, opaque tokens |
| HMI | Shell, login, `plant_overview` and faceplate per the slice spec, trend, connection and quality display, English strings through i18n keys |
| Deploy | Compose stack: mosquitto, postgres+timescale, `skyrangerd` (serves HMI), `sim` profile; `make demo` |
| Security | Per-device MQTT credentials from day one; password hashing; audit log writes for config apply and discovery approval. Plain HTTPS with a dev certificate; MQTT on the trusted bench LAN |
| Testing | Everything under 1.3 |

### 1.2 Waves

Each wave ends at gate G4. Agents are named as in [01-agent-team.md](01-agent-team.md) section 2.1. A wave only starts when the previous wave's G4 is green.

**W0 Foundation** (human with `foundation`, `contract-steward`, `test-author`)
- Repo skeleton from [03-repo-layout.md](03-repo-layout.md); module path chosen (contracts README open question); `Makefile` targets from [../09-testing/10-ci-and-quality-gates.md](../09-testing/10-ci-and-quality-gates.md) section 8; CI; `CODEOWNERS`, `OWNERS.yaml`, `.claude/agents/`.
- `proto/` and `schemas/` written from the contracts; `gen/` generated; `contracts.lock`.
- `internal/clock` (Real, Fake); `pkg/tag`, `pkg/event`, `pkg/driver` types.
- Golden corpus seed (values, quality codes, topics, Sparkplug payloads, config examples, WS transcripts) drafted by `contract-steward`, approved by the human.
- `tools/testguard` v0 with planted-violation fixtures, `tools/ownercheck`, licence gate.
- Contract pass: decide the M1-relevant CONTRACT questions (for example: NaN with held value, out-of-order rule for device-clock timestamps, `counter_total` with `modulus > 0`, `d` in durations, `_system` tag kind, durable `GET /events` cursors).
- Exit: `make pr` green on the skeleton; testguard catches every planted violation; golden corpus validates in Go (TS and Python checks come with their first consumers).

**W1 Independent modules, in parallel** (four implementers at once: `tag-engine`, `driver-sdk`, `storage`, `hmi`; plus `test-author`, `simulator`, `firmware` which are lighter)
- `tag-engine`: `core` with fake clock, registry compile, pipeline, derived built-ins, subscriptions; water meter scenarios from [../01-tag-engine/water-meter-e2e.md](../01-tag-engine/water-meter-e2e.md) as harness tests.
- `driver-sdk`: SDK, host, `sim` driver, conformance suite wiring (suite itself drafted by `test-author`, human-approved).
- `storage`: migrations, historian writer against real PostgreSQL, spool, idempotent insert, query layer for raw and bucketed history.
- `hmi`: `api-client` and `live` packages tested in Node against recorded WS transcripts (slice build order step 1), tokens, component library start.
- `simulator`: pulse meter oracle (human-approved), Sparkplug publisher with a different MQTT client library from the driver.
- `firmware`: profile implementation on the bench, with a host-side test of the counter and boot-count logic where feasible.
- Exit: each module's own tests green; conformance green for `sim` in both modes; storage passes its fault tests (F-DB-01 style: DB stop and restore without loss).

**W2 Connections** (`sparkplug-driver`, `api-server`, `tag-engine` config apply, `storage` config revisions, `security` minimal, `hmi` shell)
- `sparkplug` driver passes the shared suite plus the Sparkplug add-on suite ([../09-testing/02-driver-conformance.md](../09-testing/02-driver-conformance.md)).
- API against a fake engine with golden transcripts, then against the real engine.
- Config revisions apply and export; discovery and approval flow; users and tokens; audit writes.
- HMI shell, login, link banner, status bar.
- Exit: harness test "ESP32-style NBIRTH to `total_volume`" green; API contract tests green.

**W3 Assembly** (`foundation` wiring, `deploy`, `hmi`, `test-author`, `e2e`)
- `skyrangerd` wired; Compose stack; `plant_overview` and faceplate screens as YAML; trend.
- L3 scenarios with the conservation invariant exact; L4 smoke S1 to S8; L5 Playwright for the slice tests below.
- Exit: `make smoke` green on built images; slice screen renders live data from the simulator.

**W4 Real hardware and soak** (human, `test-author`, `firmware`)
- ESP32 on the bench, pulse generator as ground truth, three reboots, one broker restart, 24 h.
- Nightly 6 h soak on reference hardware; one pull-plug test on the target box.
- Evidence pack v0 generated from CI artifacts.
- Contract audit table by `reviewer`.

### 1.3 M1 exit criteria (all required)

| # | Criterion | Source |
|---|---|---|
| E1 | Conservation invariant holds exactly in all water meter scenarios at L3 (rollover, reboot, duplicate, out-of-order, NDEATH, broker loss, gateway restart) | [../09-testing/README.md](../09-testing/README.md) glossary; [../01-tag-engine/water-meter-e2e.md](../01-tag-engine/water-meter-e2e.md) |
| E2 | `sim` passes the conformance suite in-process and over gRPC; kill, hang and blocked-sink behaviours per phase 0 criteria 2 to 4 | [../02-drivers/implementation-order.md](../02-drivers/implementation-order.md) phase 0 |
| E3 | `sparkplug` passes the shared and add-on suites; bench ESP32 oracle count equals `total_volume` after 24 h with 3 device reboots and 1 broker restart; NDEATH gives `uncertain_last_known`, cable pull gives `bad_not_connected`, health tag distinguishes them | implementation-order phase 1 criteria 1 to 3 |
| E4 | Smoke S1 to S8 green on built images (stack boots, wm contract trace exact, reboot vs reconnect, broker restart, DB stop 60 s, `skyrangerd` SIGKILL conserves counts, unapproved device creates no tag, config export-wipe-import diff empty) | [../09-testing/06-integration-and-e2e.md](../09-testing/06-integration-and-e2e.md) section 2.5 |
| E5 | HMI acceptance tests 1 to 5 (login, live latency, quiet device, link down, browser link loss), 8 (rollover and reboot), 12 (gap rendering), 13 (dense trend), 19 (export), 20 (offline gateway) pass in Playwright | [../05-hmi-api/09-water-meter-slice.md](../05-hmi-api/09-water-meter-slice.md) section 7 |
| E6 | Nightly 6 h soak passes judge thresholds: memory flat, no unexplained quality flips, no gaps in history beyond injected ones | [../09-testing/07-soak-and-performance.md](../09-testing/07-soak-and-performance.md) |
| E7 | Pull-plug test on the target box: no corruption, bounded loss window, cumulative counters recover | [../03-storage/08-disk-power-corruption.md](../03-storage/08-disk-power-corruption.md) |
| E8 | Mutation score at or above the initial threshold on `pkg/tag/derived`-equivalent code (`internal/tagengine/derived`) and `internal/sparkplug` | [../09-testing/12-ai-agent-guardrails.md](../09-testing/12-ai-agent-guardrails.md) section 6 (85%, to be calibrated) |
| E9 | testguard, ownercheck, gitleaks and licence gate clean on `main`; zero waivers outstanding; `contracts.lock` consistent | guardrails; [../02-drivers/libraries-and-licensing.md](../02-drivers/libraries-and-licensing.md) |
| E10 | Contract audit table: every normative section used by M1 maps to a test or a recorded Open question | [01-agent-team.md](01-agent-team.md) section 5.4 |
| E11 | Manual column: jug test within one pulse; Wi-Fi pulled shows "Offline" within `heartbeat_stale` plus UI latency | slice spec section 7 |

### 1.4 Deliberately left out of M1
- Alarm engine, alarm banner behaviour beyond showing the `leak` condition tag, notifications (M2). The slice's `alarm_*` widgets render empty or hidden.
- Control writes from the HMI and role-gating tests 9 to 11 (M2). `counter_total` manual alignment stays an API call exercised by tests only.
- TLS on MQTT and PKI, MFA, OIDC, secrets store, signed images (M5). M1 documents the bench network as trusted.
- Modbus, OPC UA, EtherNet/IP, S7 (M3 and later); northbound OPC UA and Sparkplug.
- Scripting, plugins, cloud sync, fleet, `skyrangerctl`, appliance image.
- Visual designer, kiosk, Hebrew translation, accessibility audit (keys and `dir` handling scaffolded only), performance tests 14 to 18.
- 100k-tag targets; M1 verifies the water meter and a 5,000-tag generated load only as a smoke.
- Per-area permissions, customer-facing evidence pack beyond v0.

## 2. M2: alarms and operator actions

**Why second**: the leak and device-offline alarms are in decisions entry 6, and an alarm system is where trust is won or lost. Operator writes belong here because they share the audit and permission machinery.

**Scope**
- Alarm engine per [../04-alarms/](../04-alarms/README.md): four contract states, orthogonal flags, quality-hold rule, on/off-delay, deadband, persist-then-announce transactions, restart recovery, DB-outage spool (reusing `internal/spool`).
- Alarm types in the contract only (`high`, `low`, `equal`, `not_equal`, `quality`) plus the two alarms of the slice: leak (`equal true` on `continuous_activity`) and device offline.
- Notifications: outbox with email (SMTP) and webhook channels; dead-man heartbeat.
- Acknowledge, shelve, comment; alarm list and banner in the HMI.
- Write path ([../01-tag-engine/writes.md](../01-tag-engine/writes.md)): REST write, confirm dialog, `?wait=`, unknown outcome handling; role gating; audit with hash chain (`audit_log`).
- KPIs minimal: alarm rate, standing alarms, chattering list.

**Exit criteria**
1. [../04-alarms/11-water-meter-e2e.md](../04-alarms/11-water-meter-e2e.md) scenarios pass at L3 and L4 (leak, device offline, gateway restart, broker loss, suppression).
2. Slice tests 6, 7, 9, 10, 11, 14 and 16 pass ([../05-hmi-api/09-water-meter-slice.md](../05-hmi-api/09-water-meter-slice.md)): leak alarm and ack with audit, health alarm, control success, unknown outcome with exactly one `/write` request, role gating, performance, axe-core.
3. State machine table tests cover every transition; mutation score at or above threshold on `internal/alarm/state` and `internal/alarm/engine` ([../09-testing/12-ai-agent-guardrails.md](../09-testing/12-ai-agent-guardrails.md) section 6).
4. Fault tests: DB stop during an alarm transition loses nothing (journal replay idempotent); SMTP down never blocks the engine; every transition is journalled before it is announced ([../04-alarms/README.md](../04-alarms/README.md) decision 3).
5. `audit_log` hash chain verifies in a test that tampers one row and detects it.

**Deliberately left out**: SMS modem and push channels, escalation tiers, new alarm types (`deviation`, `rate_of_change`), consoles and per-operator KPIs, ISA-18.2 conformance claims (specifics are UNVERIFIED until the standard is read), signed audit checkpoints (M5), alarm sounds and kiosk behaviour.

## 3. M3: second driver (Modbus TCP) and the first operations tooling

**Why**: Modbus TCP has the widest reach on small water and metering sites and is the first polled driver, proving the scheduler and block planner that `s7` and `enip` reuse ([../02-drivers/implementation-order.md](../02-drivers/implementation-order.md) section 4). The `deploy` agent is otherwise idle here, so a minimal `skyrangerctl` runs as a parallel track, because a customer box needs install, backup and restore.

**Gate before starting**: confirm the first customer's hardware. If a first customer needs Modbus RTU/serial or OPC UA first, reorder using the decision gate in implementation-order phase 2; if the OPC UA spike is wanted, it can run alongside (it is time-boxed with written pass/fail).

**Scope**
- `modbus` driver: own client, address grammar, planner with gap merging, byte orders, exception mapping ([../02-drivers/modbus-tcp.md](../02-drivers/modbus-tcp.md)).
- Modbus TCP server in `skyranger-sim` (so CI never needs hardware); OpenPLC bench on a Raspberry Pi.
- `skyrangerctl` v0: `init`, `backup`, `restore`, `export`, support bundle ([../08-deployment/04-install-and-first-boot.md](../08-deployment/04-install-and-first-boot.md), [../08-deployment/07-data-and-migrations.md](../08-deployment/07-data-and-migrations.md), [../08-deployment/13-supportability.md](../08-deployment/13-supportability.md)).
- Config as code: repo layout and `apply` with validation in CI ([../08-deployment/05-config-as-code.md](../08-deployment/05-config-as-code.md)).

**Exit criteria**
1. Driver phase 2 criteria 1 to 5 ([../02-drivers/implementation-order.md](../02-drivers/implementation-order.md)): every address-table row is a passing unit test; deleting a register degrades only that tag; 24 h against OpenPLC with zero unexplained flips; reconnect after PLC reboot under 35 s; write is never retried.
2. Conformance suite green in both modes; fault catalogue entries F-NET-* and F-VAL-* applicable to Modbus are tested ([../09-testing/04-fault-injection.md](../09-testing/04-fault-injection.md)).
3. Backup then restore on a clean box reproduces config and history; export-wipe-import diff empty ([../03-storage/07-backup-restore.md](../03-storage/07-backup-restore.md)).
4. A mixed site (Sparkplug meter plus Modbus PLC plus sim) runs the 6 h nightly soak.

**Deliberately left out**: OPC UA, EtherNet/IP, S7, northbound OPC UA (own phases per implementation-order), Modbus RTU unless the gate says otherwise, fleet and cloud sync, OS A/B updates.

## 4. M4: scripting v1

**Prerequisite**: the human decides contract proposals P1 to P7 in [../06-scripting/README.md](../06-scripting/README.md) (P8 and P9 are v2) and records them in `decisions.md`. Scripting adds attack surface, so its sandbox tests are part of the milestone, not of M5.

**Scope**: exactly the v1 list in [../06-scripting/11-phased-plan.md](../06-scripting/11-phased-plan.md): `skyranger-scriptd` with pooled CPython workers in a network-less container, script host in `skyrangerd`, declarative triggers (tag change, timer, event, startup, manual), deny-by-default permissions, run records, scripts as config documents, `skyranger.testing` for offline tests. First real scripts: a daily consumption report and an alarm webhook.

**Exit criteria**
1. The phase v1 exit criteria in [../06-scripting/11-phased-plan.md](../06-scripting/11-phased-plan.md) pass (that file is authoritative; not restated here to avoid drift).
2. Sandbox tests from [../09-testing/09-security-testing.md](../09-testing/09-security-testing.md) for the scripting boundary pass: no filesystem, network or subprocess access without grant; runaway script killed by wall-clock and RSS limits; core keeps running with the scripts container stopped.
3. Python mutation score threshold on `scripting/` runtime pieces (mutmut or Cosmic Ray, tool choice UNVERIFIED).
4. A script write goes through the same service layer as REST: audit record with principal `script:{pool}/{name}`.

**Deliberately left out**: plugins (v2), SDK compatibility policy and open-core licensing work (v3), `ScriptPool` (v2), long-running user loops, UI components from scripts.

## 5. M5: security hardening

**Why before the pilot**: every customer delivery must pass the release checklist ([../07-security/14-release-checklist.md](../07-security/14-release-checklist.md)); this milestone is the work to make it passable. The cheap-now items (per-device credentials, hashing, audit writes) already shipped in M1.

**Scope**
- Site CA and TLS on HTTPS, MQTT (8883), PostgreSQL, driver gRPC boundary as specified ([../07-security/03-tls-and-pki.md](../07-security/03-tls-and-pki.md)); ESP32 CA pinning.
- Mosquitto dynamic security plugin managed by `skyrangerd`: per-device ACLs and revocation ([../07-security/04-device-identity.md](../07-security/04-device-identity.md)).
- `secret://` store with master key (TPM when present) ([../07-security/06-secrets.md](../07-security/06-secrets.md)).
- Audit signed checkpoints, optional syslog forward ([../07-security/08-audit-logging.md](../07-security/08-audit-logging.md)).
- TOTP for admin and any remote path; break-glass local admin retained.
- Signed images (cosign), digest-pinned compose, SBOM per release, offline update bundles with rollback ([../07-security/07-secure-update.md](../07-security/07-secure-update.md), [../08-deployment/06-updates-and-rollback.md](../08-deployment/06-updates-and-rollback.md)).
- Hardening baseline for host, containers and drivers ([../07-security/09-hardening.md](../07-security/09-hardening.md)); encrypted backups ([../07-security/10-backup-security.md](../07-security/10-backup-security.md)).
- Vulnerability disclosure file `SECURITY.md` (proposed by 07; human decides) ([../07-security/13-vulnerability-disclosure.md](../07-security/13-vulnerability-disclosure.md)).

**Exit criteria**
1. Security tests in [../09-testing/09-security-testing.md](../09-testing/09-security-testing.md) pass: fuzz targets for every parser (Sparkplug, Modbus, YAML config, API bodies), authorization matrix for every endpoint (SEC-A1) including audit behaviour (SEC-A8), dependency and image scans with no unwaived high findings.
2. Release checklist items SHIP-* and the security checklist in file 14 all pass or are waived by the human with a recorded reason.
3. TLS-off configuration is impossible without an explicit, logged setting; a certificate rotation and a device revocation are each demonstrated in a test.
4. Update and rollback: a deliberately failing release rolls back automatically; a tampered bundle is rejected.
5. Threat-model review: the STRIDE pass on the scripting boundary requested by the scripting area is done ([../06-scripting/README.md](../06-scripting/README.md) open questions).

**Deliberately left out**: IEC 62443 certification (self-assessment against 62443-4-2 at SL 2 is in scope as a document, not certification; [../07-security/12-iec62443.md](../07-security/12-iec62443.md)), OIDC and LDAP, OS A/B appliance image (RAUC), mTLS for devices (v2 option), cloud-side security, third-party penetration test (recommended before the second customer; not in M5).

## 6. M6: pilot customer

**Scope**: one real site, installed by the author as integrator, on reference hardware, with the evidence pack and FAT/SAT records. The pilot chooses what is real: which devices, which alarms, which reports.

**Preconditions**: M5 exit; a written site scope with what is explicitly not covered; the SCADA-is-not-a-safety-system statement in the contract paperwork ([../07-security/11-safety.md](../07-security/11-safety.md)); a customer who accepts pilot terms.

**Scope items**
- Site configuration as a config-as-code repo; `skyrangerctl init` install on the reference box.
- Customer-specific screens and alarm rationalization (cause, consequence, action, time to respond for every alarm).
- FAT on the bench with the customer's configuration, then SAT on site ([../09-testing/10-ci-and-quality-gates.md](../09-testing/10-ci-and-quality-gates.md) section 5).
- Evidence pack generated for the pilot release ([../09-testing/11-evidence-pack.md](../09-testing/11-evidence-pack.md)).
- Support runbooks exercised once ([../08-deployment/13-supportability.md](../08-deployment/13-supportability.md)).
- Exit plan rehearsed: full export and restore on a second box ([../08-deployment/12-exit-plan.md](../08-deployment/12-exit-plan.md)).

**Exit criteria**
1. "Ships to a customer" checklist SHIP-01.. and FAT items pass; `shipcheck` output attached; SAT items pass with the customer witnessing.
2. 72 h soak on the reference hardware and relay power-cut series pass before shipping ([../09-testing/07-soak-and-performance.md](../09-testing/07-soak-and-performance.md), [../09-testing/08-hardware-and-power-cut.md](../09-testing/08-hardware-and-power-cut.md)).
3. A defined production observation period with no unexplained history gaps, no lost alarms and no unplanned restarts that the restart-recovery tests did not predict. The length (proposal: 30 days) is a human decision. UNVERIFIED that 30 days surfaces enough; the first pilot is partly the experiment.
4. At least one real incident or fault (power cut, network loss, device reboot) handled with a post-mortem that feeds a regression test (record and replay, [../09-testing/05-record-replay.md](../09-testing/05-record-replay.md)).
5. Customer sign-off and a retrospective that updates this roadmap, the claims register and the licensing question.

**Deliberately left out**: multi-site, fleet management, cloud sync, appliance image, multi-tenant anything, a licensing model (decisions entry 5: after two or three real projects), the plugin platform and SDK promises, remote access beyond the on-demand outbound tunnel if it exists by then (otherwise on-site support only), IEC 62443 certification.

## 7. Milestone dependency summary

```
 M1 slice ──► M2 alarms+actions ──► M3 Modbus + skyrangerctl ──► M4 scripting ──► M5 hardening ──► M6 pilot
   │                                   ▲                                              │
   └─ per-device creds, audit writes,  └─ hardware gate: first customer's PLCs        └─ release checklist 14
      hashing (cheap now)                 may reorder M3/M4 (OPC UA, RTU)
```

M4 and M5 may swap if the pilot customer does not want scripts. M3's Modbus may be replaced by whichever protocol the pilot's hardware needs; the milestone is "second driver", not "Modbus" ([../02-drivers/implementation-order.md](../02-drivers/implementation-order.md) decision 1).

## Decisions made
1. Order: M1 slice, M2 alarms and operator actions, M3 second driver plus minimal `skyrangerctl`, M4 scripting v1, M5 security hardening, M6 pilot.
2. M1 is built in five waves (W0 to W4) with a gate between each; only W1 runs four implementers in parallel.
3. Alarms are split from M1: the slice needs tags, history and a live dashboard; alarms and writes share audit and permission machinery and form M2.
4. Per-device credentials, password hashing and audit writes are in M1; TLS, PKI, secrets store, signing and MFA are in M5.
5. Exit criteria are taken from the existing test strategy and area docs by reference, not restated, so they cannot drift.
6. No dates and no effort numbers until M1 gives velocity data.
7. M5 precedes M6 because the security release checklist gates any customer delivery.
8. Left-out lists are binding.

## Alternatives rejected and why
- **Alarms inside M1**: the slice acceptance list is already long (hardware, soak, power cut). Alarms would roughly double the risk surface of the first milestone. The leak condition tag exists in M1, so M2 adds the engine, not the physics.
- **Security hardening before scripting**: attractive, but scripting changes the trust boundary and its sandbox tests belong with its design. Hardening after M4 covers the final surface once.
- **Pilot customer before hardening**: contradicts the release checklist, and device identity and TLS are expensive to retrofit on fielded devices.
- **Starting OPC UA as the second driver**: highest library risk and not needed by small sites; spike optional in M3 (implementation-order rejects it the same way).
- **Cloud sync or fleet in the first six milestones**: edge-first (decisions entry 1); the pilot needs one box that works alone.
- **Dated plan**: would be fiction at this stage.

## Open questions
1. M1 contract pass: which CONTRACT questions from the area READMEs block W1? Proposal: all those listed in [../09-testing/README.md](../09-testing/README.md) "Contract issues found" (seven items) plus duration `d` unit, `_system` tag kind and durable event cursors. Human to rule before W1.
2. Reference hardware SKU for E6, E7 and M6 is not chosen ([../09-testing/README.md](../09-testing/README.md) open questions). It must exist before W4.
3. Whether the L3 harness runs against real PostgreSQL by default in M1 (09 proposes in-memory per commit, PostgreSQL nightly).
4. Whether M3 should instead be Modbus RTU or OPC UA, which depends on the pilot's hardware. Talk to the first customer before M2 ends.
5. Length of the M6 observation period and whether a peer review of protected-path changes is available by then.
6. Who writes and maintains Hebrew strings (slice spec flags the placeholders as unreviewed); deferred past M1 but needed if the pilot requires it.
7. Storage docs 09 to 11 (capacity, queries, Go interfaces) are listed in the storage README but not yet written. M1 W1 `storage` work depends on them; they should be finished or explicitly waived before W1.
