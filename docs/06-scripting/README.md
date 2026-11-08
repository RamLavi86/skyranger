# 06 Scripting and Extensibility

Status: design draft for v1 scripting, with the v2 plugin and v3 SDK tiers outlined. Binding contracts are in [../00-contracts/](../00-contracts/). Nothing here overrides them. Where this area needs a contract change, it is listed below as a **contract proposal** and repeated in the Open questions of the file that needs it.

## What this area covers

How user Python runs alongside the Go core, what it may touch, how it is triggered, limited, logged, stored, versioned and tested. It also covers how Sky Ranger later becomes a platform that other engineers extend with plugins and a versioned SDK.

| File | Contents |
|---|---|
| [01-runtime.md](01-runtime.md) | Embedded interpreter vs sandboxed worker process (recommendation: worker), process layout (`skyrangerd` script host, `skyranger-scriptd` supervisor, CPython workers), the `skyranger.script.v1` gRPC protocol, the worker pipe protocol, Go and Python sketches |
| [02-script-api.md](02-script-api.md) | The Python API: `tags`, `history`, `events`, `alarms`, `http`, `log`, `ctx`, type mapping, worked water-meter examples |
| [03-triggers-and-execution.md](03-triggers-and-execution.md) | `Script` and `ScriptLibrary` config kinds, triggers (tag change, timer, event, startup, manual), queueing, concurrency, timeouts, resource limits, write-loop protection, quarantine |
| [04-sandbox-and-permissions.md](04-sandbox-and-permissions.md) | Trust boundary, the permission model (deny by default), sandbox layers (container, rlimits, pipes, audit hooks, seccomp), secrets, outbound HTTP, STRIDE pass |
| [05-errors-and-logging.md](05-errors-and-logging.md) | Exceptions, run records, `_system/scripts/...` tags, events, logs, the `/api/v1/scripts` endpoints |
| [06-storage-and-versioning.md](06-storage-and-versioning.md) | Scripts inside config revisions, export layout for git, validation at draft time, rename safety, run-to-revision traceability |
| [07-testing.md](07-testing.md) | Offline unit tests with `skyranger.testing`, dry runs on a live gateway, integration tests with the simulator and fault injection |
| [08-ignition-comparison.md](08-ignition-comparison.md) | Side-by-side with Ignition's scripting and module model, and what we copy and avoid |
| [09-plugins.md](09-plugins.md) | Platform tier: extension points (drivers, notification channels, services, script libraries, UI components), package format, manifest, signing, trust store |
| [10-sdk-compat-licensing.md](10-sdk-compat-licensing.md) | Versioned SDK surfaces, compatibility and deprecation policy, open-core licensing considerations |
| [11-phased-plan.md](11-phased-plan.md) | v1 minimal, v2 plugins, v3 SDK, with exit criteria |

Read in order: README, 01, 03, 02, 04. The rest can be read as needed. Phase mapping: files 01 to 07 are v1, 09 is v2 (and v3 for UI components), 10 and 11 span all phases, 08 is background. Unverified facts (Ignition details, seccomp library, performance figures, license facts) are marked UNVERIFIED in place.

## Architecture in one picture

```
 ┌──────────────────────── skyrangerd container ────────────────────────┐
 │ skyrangerd (Go)                                                      │
 │   tag engine ──in-process subscriptions──► script host               │
 │   alarm engine, events ───────────────────► (triggers, scheduler,    │
 │   writes service  ◄── same code path as REST ─ permission checks,    │
 │   history service ◄────────────────────────── run records, budgets,  │
 │   http egress (allowlist, secrets) ◄───────── audit "script:...")    │
 └──────────────────────────────┬───────────────────────────────────────┘
                                │ gRPC skyranger.script.v1 over UDS
                                │ /run/skyranger/scripts/default/host.sock (shared volume)
 ┌──────────────── skyranger-scripts container ─────────────────────────┐
 │  network_mode: none, read-only rootfs, non-root, cap_drop ALL,       │
 │  mem/pids/cpu limits                                                 │
 │  skyranger-scriptd (Go supervisor)                                   │
 │     ├── pipes (fd 3/4, JSON lines) ── python3 worker #0 (CPython)    │
 │     └── pipes (fd 3/4, JSON lines) ── python3 worker #1              │
 └──────────────────────────────────────────────────────────────────────┘
```

- **Triggers are decided in Go**, in `skyrangerd`, from in-process tag engine subscriptions and the event stream. Python only executes runs it is handed.
- **Workers have no network and no sockets.** Every effect (write, history query, event, HTTP) is a request back to the script host, which enforces permissions and budgets and uses the same service layer as the REST API, so validation and audit are identical.
- **The scripts container is the trust boundary.** If it hangs, crashes or is absent, the core keeps running: tags, alarms, history and HMI are unaffected.

## Key decisions (area summary)

1. CPython 3 in **separate worker processes** inside a **separate, network-less container**. No embedded interpreter in `skyrangerd`.
2. **Declarative triggers in config** (`kind: Script`), scheduled by Go. Run-to-completion handlers with hard timeouts. No long-running user loops in v1.
3. **Deny-by-default permissions** per script with a pool-wide ceiling, enforced in Go on every call. No filesystem, network or subprocess access by default; outbound HTTP only through the host's allowlisted egress, with secrets injected host-side.
4. **Scripts are config.** They live in config revisions, are diffed, applied atomically, restored, exported (with optional split-out `.py` files for git), and every run records the revision and source hash it ran.
5. **Plugins are always out-of-process** behind versioned protocols (`skyranger.driver.v1`, `skyranger.notify.v1`, `skyranger.service.v1`, `skyranger.ui.v1`), signed with Ed25519, trusted via an admin-managed offline trust store.

## Contract proposals (need a decisions.md entry before implementation)

| # | Contract | Proposal | Why |
|---|---|---|---|
| P1 | messages.md section 3: "The Python scripting runtime will use the REST API for writes" (and the internal bus being for the scripting runtime) | Scripts reach the core through the **script host** over `skyranger.script.v1`; the host calls the same Go service layer as REST handlers (same validation, `write_limits`, audit, `skyranger.tag.write_completed.v1`). External Python clients still use REST and the MQTT mirror. | Lets the worker have no sockets at all, gives unforgeable per-run identity, and avoids losing tag-change triggers on QoS 0 retained topics or 100 ms WS batching |
| P2 | tag-model.md section 9: "Kinds in v1: Gateway, Connection, Template, TagGroup" | Add `Script` and `ScriptLibrary` (v1), `ScriptPool` (v2). Additive | Scripts must be in config revisions |
| P3 | messages.md section 4 event catalog and `source` component list | Add types `skyranger.script.run_failed.v1`, `skyranger.script.state_changed.v1`, `skyranger.script.run_requested.v1`; add user-event family `skyranger.user.{name}.v1`; add component `script-host` | Observability and script-raised events |
| P4 | api.md section 3 | Add `/api/v1/scripts...` endpoints (05-errors-and-logging.md) and `requested_by` principal form `script:{pool}/{name}` | Manual triggers, run records, logs |
| P5 | tag-model.md section 1 `_system` tags | Add `_system/scripts/{name}/...` and `_system/script_pools/{pool}/...` | Health visibility |
| P6 | tag-model.md section 8 / section 1 ("`_system` never in user config") | Allow alarm definitions that target `_system` tags (e.g. an `AlarmBinding` or a `system_alarms` block in `Gateway`) | Otherwise nobody can alarm on a failing script or a down driver |
| P7 | tag-model.md section 4 `good_manual` | A memory-tag write by a script gets quality `good`, not `good_manual` (which means operator-set) | Reports must distinguish operator overrides from automation |
| P8 | tag-model.md section 9 kinds; messages.md section 4 | v2: add `Plugin` (grants and settings, no code), `PluginTrust` file form; events `skyranger.plugin.*.v1`; `source` component `plugin-host` ([09-plugins.md](09-plugins.md)) | Plugin install state must be restorable and auditable |
| P9 | api.md section 3 | v2: `/api/v1/plugins...` endpoints (admin) and `requested_by` form `plugin:{publisher}/{name}` ([09-plugins.md](09-plugins.md)) | Install, approve, enable, remove |

Smaller additive items the files rely on (record with P2/P4): `warnings` array in the draft response (06); optional `metadata.id` (UUIDv7) on `Script` (06); script retention setting in `Gateway.spec` (05); `GET /config?layout=split` (06).

**Where this area disagrees with a binding contract or another area's draft:** messages.md section 3 says the Python runtime uses REST for writes, and [../07-security/02-authn-authz.md](../07-security/02-authn-authz.md) (`svc-python-runtime`, role `operator`) builds on that. P1 replaces it; if P1 is rejected the fallback in [01-runtime.md](01-runtime.md) section 7 applies. The alarm area is asked to accept that `memory` tags written by scripts are the way to raise script-detected alarms ([02-script-api.md](02-script-api.md) section 3.4).

## Decisions made
1. Area split into eleven files plus this index; runtime and triggers are the load-bearing ones.
2. Nine contract proposals (P1 to P9) are listed here and nowhere silently assumed elsewhere; every file that depends on one says so.
3. The scripts container is optional: a gateway without it is a supported, fully functional configuration with scripting unavailable.

## Alternatives rejected and why
- **One big scripting file**: too long to review; triggers, sandboxing and plugins have different reviewers and lifecycles.
- **Designing plugins first**: the integrator phase needs scripting now; plugins are v2. Designing the plugin boundary as "always out of process" now is enough to avoid painting ourselves into a corner.

## Open questions
- All nine contract proposals (P1 to P9) need an owner decision and a decisions.md entry.
- Security area (07-security) should run its planned STRIDE pass on the scripting boundary against [04-sandbox-and-permissions.md](04-sandbox-and-permissions.md).
- Whether the scripts container is part of the default compose file or an opt-in profile (deployment area, 08-deployment).
