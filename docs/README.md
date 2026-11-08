# Sky Ranger Documentation Index

Sky Ranger is an open, modular, edge-first SCADA platform. The first milestone is an ESP32 pulse-counter water meter (1 pulse = 1 liter) plus simulated devices flowing through a tag engine into history and a live web dashboard.

Status: design phase, no code yet. Documents marked draft may still change; the contracts in `00-contracts` are binding and change only through a [decisions.md](decisions.md) entry. Claims about external standards or tools that were not checked are marked **UNVERIFIED** in place.

## Areas

| Area | One line |
|---|---|
| [decisions.md](decisions.md) | The decision log: edge-first, the stack, open standards, hybrid tag discovery, integrator first, first milestone, testing. Human-owned. |
| [00-contracts/](00-contracts/README.md) | Binding interface contracts: tag model, messages and device profile, driver interface, REST/WebSocket API, naming conventions. |
| [01-tag-engine/](01-tag-engine/README.md) | The heart of `skyrangerd`: registry, update pipeline, derived tags (`counter_total`, `rate`, ...), subscriptions, writes, Go design, performance, water meter walkthrough. |
| [02-drivers/](02-drivers/README.md) | Driver framework and protocols: `sim`, Sparkplug B, Modbus TCP, OPC UA, EtherNet/IP, S7; reliability, libraries and licensing, implementation order, test bench. |
| [03-storage/](03-storage/README.md) | PostgreSQL and TimescaleDB schema, historian write path, store-and-forward, config storage, events and audit storage, backup, disk and power failure (draft, still in progress; files 09 to 11 not yet written). |
| [04-alarms/](04-alarms/README.md) | Alarm engine: state machine, alarm types, nuisance management, acknowledgement, notifications, journal, KPIs, restart and sync, leak alarm walkthrough. |
| [05-hmi-api/](05-hmi-api/README.md) | Web HMI (TypeScript and React): architecture, screen format, components, designer, roles and control, ergonomics, i18n, API extensions, the water meter dashboard spec. |
| [06-scripting/](06-scripting/README.md) | Python scripting and extensibility: sandboxed workers, script API, triggers, permissions, testing, plugins and SDK tiers, phased plan. |
| [07-security/](07-security/README.md) | Threat model, authn/authz, TLS and PKI, device identity, segmentation, secrets, secure update, audit, hardening, safety statement, IEC 62443, release checklist. |
| [08-deployment/](08-deployment/README.md) | Topologies, containers, packaging and hardware, install and first boot, config as code, updates, migrations, fleet, cloud sync, observability, time and power, exit plan, release engineering. |
| [09-testing/](09-testing/README.md) | Test pyramid L0 to L7, golden contracts, driver conformance, simulator, fault injection, replay, integration, soak, hardware and power cut, security testing, CI gates, evidence pack, AI agent guardrails. |
| [10-agent-team-and-roadmap/](10-agent-team-and-roadmap/README.md) | The Claude Code agent team and who owns what, the milestone roadmap with exit criteria, and the proposed monorepo layout. |

## Recommended reading order

For the human owner (first read, about the whole system):
1. [../README.md](../README.md) and [decisions.md](decisions.md)
2. [00-contracts/README.md](00-contracts/README.md), then its four documents
3. [01-tag-engine/README.md](01-tag-engine/README.md), then [water-meter-e2e.md](01-tag-engine/water-meter-e2e.md) (best single place to check the pieces fit)
4. [09-testing/README.md](09-testing/README.md) and [12-ai-agent-guardrails.md](09-testing/12-ai-agent-guardrails.md)
5. [10-agent-team-and-roadmap/](10-agent-team-and-roadmap/README.md): layout, team, roadmap
6. [02-drivers/README.md](02-drivers/README.md) and [implementation-order.md](02-drivers/implementation-order.md)
7. [03-storage/README.md](03-storage/README.md), [04-alarms/README.md](04-alarms/README.md), [05-hmi-api/README.md](05-hmi-api/README.md)
8. [07-security/README.md](07-security/README.md) and [08-deployment/README.md](08-deployment/README.md)
9. [06-scripting/README.md](06-scripting/README.md) (needed from milestone M4)

For an agent working on one module: the root [../CLAUDE.md](../CLAUDE.md), the contracts it is assigned, its area README, [10-agent-team-and-roadmap/01-agent-team.md](10-agent-team-and-roadmap/01-agent-team.md) for its ownership rules, and the testing files that apply to it ([01](09-testing/01-unit-and-contract-tests.md) and [12](09-testing/12-ai-agent-guardrails.md) always).

## Conventions across all docs
- Every design file ends with Decisions made, Alternatives rejected and why, Open questions (area READMEs also list contract proposals).
- Paths such as `skyranger/...` use a placeholder Go module path until one is chosen.
- A proposal that needs a contract change is never assumed by code; it is listed, decided by the human and recorded in `decisions.md` first.
