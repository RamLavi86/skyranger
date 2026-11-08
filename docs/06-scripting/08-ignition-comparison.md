# 08 Comparison with Ignition's scripting and module model

Status: design draft. Ignition is the reference the future platform audience knows. Everything below about Ignition is **from memory of the product as of Ignition 8.1** and is marked UNVERIFIED where not certain; check against Inductive Automation's documentation before using any claim externally (in marketing, migration guides, or comparisons). No Ignition code or documentation is copied.

## 1. Scripting model side by side

| Topic | Ignition (8.1, from memory) | Sky Ranger design |
|---|---|---|
| Language | Jython 2.7 (Python 2 semantics on the JVM). Newer Python 3 syntax and CPython libraries unavailable. Whether 8.3 changes this: UNVERIFIED | CPython 3 |
| Where it runs | Inside the gateway JVM (gateway scope), in Vision/Perspective clients, in the Designer. Full access to Java classes | Separate worker processes in a separate container; no Java-equivalent access path to the core |
| Isolation | None between scripts and gateway: a script can import Java classes, open files, sockets, call `System.exit`-class APIs (UNVERIFIED specifics) | OS-level sandbox, no network or filesystem by default |
| Triggers | Gateway event scripts: startup, shutdown, timer (fixed delay/rate, own thread options), tag change, message handlers; tag event scripts (value, quality, alarm events); client scripts (screens); Perspective session/component scripts | Declarative config: `tag_change`, `timer`, `event`, `startup`, `manual`. No client-side scripting; no `shutdown`; no per-tag script attachment |
| Where scripts are stored | In project resources, project export/import, project inheritance | In config revisions: diff, restore, audit, lossless export, optional `.py` split layout |
| Libraries | Project script library (shared modules; inherited across projects) | `ScriptLibrary` kind, `libs.*` imports, resolved per revision |
| Tag API | `system.tag.readBlocking`, `writeBlocking`, `browse`, `configure`; async variants; `QualifiedValue` objects with quality code and timestamp | `tags.read`, `tags.write`, `tags.browse`; canonical value object; no tag creation from scripts (config only) |
| History | `system.tag.queryTagHistory` (datasets/aggregates) | `history.read` with the REST aggregation rules |
| Database | `system.db.*` against configured DB connections | **No arbitrary DB access in v1.** History and events through APIs; external DBs through a plugin or an allowlisted HTTP service |
| HTTP | `system.net.httpGet/httpPost` and Java HTTP classes | `http.*` through host with allowlist, SSRF defense, secret injection |
| Alarms | alarm pipelines, notification via pipelines | alarms are tag config; scripts react to events; notification channels are a plugin extension point (09) |
| Permissions | Role/security-zone controls on who edits projects; runtime scripts run as the gateway | Per-script deny-by-default permissions, pool ceiling set by the deployer, run-scoped identity |
| Resource limits | Shared gateway thread pools; a runaway script can degrade the gateway (UNVERIFIED how limits work) | Wall-clock, memory, call budgets, queue bounds, quarantine |
| Testing | Script console in the Designer, interactive; no standard unit-test harness (UNVERIFIED) | `skyranger.testing` offline fakes, dry run, simulator integration |
| Debugging | Script console, logging | Dry run, run records with trace, structured logs |

## 2. Module system side by side

| Topic | Ignition modules (from memory) | Sky Ranger plan (09) |
|---|---|---|
| What a module is | A Java package built with the Module SDK, packaged as `.modl`, loaded into the gateway JVM; can add drivers, scripting functions, Vision/Perspective components, reporting features, etc. | A **plugin**: signed package with a manifest; always **out of process** behind a versioned protocol |
| Extension points | Very broad (anything in the gateway; the Module SDK exposes many hooks) | Narrow and explicit: drivers, notification channels, services, script libraries, UI components |
| Isolation | Runs in the gateway JVM, same fate | Separate process/container, crash isolation, declared permissions |
| Language | Java (and Kotlin etc., JVM) | Any language that speaks the protocol (Go SDK first, Python SDK for scripts; gRPC for the rest) |
| Signing and trust | Modules are signed; the gateway checks the signature chain; unsigned modules require an explicit allow setting (UNVERIFIED details of the certificate chain and the setting name) | Ed25519-signed manifest, offline admin-managed trust store; unsigned refused unless an explicit, audited, visible override |
| Distribution | Inductive Automation's module marketplace ("Ignition Exchange" for resources, third-party module vendors) | Files first (`.srp` bundles installed offline); a registry is a v3 question |
| Compatibility | Modules target a minimum Ignition version and the SDK version; major releases (7.9 to 8.0) required module rebuilds (UNVERIFIED specifics) | Protocol majors (`skyranger.driver.v1`...) supported side by side for a defined window (10) |
| Licensing | Module licensing integrated with Ignition's licensing for commercial third-party modules (UNVERIFIED) | Open-core considerations in 10; no license server in v1 or v2 |

## 3. What we copy

1. **Script libraries as a first-class idea.** Shared code reused across scripts is how integrators scale themselves. Kept as `ScriptLibrary`.
2. **Declarative event triggers** (tag change, timer, startup) rather than "write a thread". Same mental model, more explicit config.
3. **Scripts stored with the project config**, so a customer backup contains everything. Ours add diff and per-revision traceability.
4. **Blocking, synchronous call style** for the common case; engineers write `read`, then `if`, then `write`.
5. **Modules as the growth path to a platform**: an ecosystem is how a SCADA product outgrows its authors.
6. **Signed extensions.**

## 4. What we deliberately avoid

1. **Scripts running inside the core process with unrestricted host access.** Ignition's power here is also its largest operational and security risk (UNVERIFIED how much Inductive mitigates). We pay latency per call to get containment.
2. **A legacy interpreter.** Jython 2.7 is frozen; engineers moving to Python 3 hit a wall. We use CPython 3 and hold an API-level compatibility policy so we never end up stranded the same way.
3. **Client-side scripting** (code running in each operator station). Our HMI is a browser app with no user code; computation belongs in derived tags or gateway scripts, visible to every client (05-hmi-api/02-screen-format.md).
4. **Per-tag event scripts as a hidden place for logic.** Logic that fires "when this tag changes" is a `Script` with a `tag_change` trigger: one place to review, one list of what runs on the gateway.
5. **In-process, same-fate modules.** A third-party driver should not be able to crash the tag engine; this is already the stance in driver-interface.md.
6. **A single wide extension API.** Narrow extension points are cheaper to keep compatible.
7. **Tying scripts to a vendor DB or licensing mechanism.**

## 5. Migration cheat sheet (Ignition engineers learning Sky Ranger scripting)

| Ignition idiom (from memory) | Sky Ranger |
|---|---|
| `system.tag.readBlocking(["[default]Site/Meter/Flow"])[0].value` | `tags.read("site/meter/flow").value` (paths are lowercase, no provider prefix) |
| `qv.quality.isGood()` | `tv.quality.is_good` or `tv.good` |
| `system.tag.writeBlocking([path], [value])` | `tags.write(path, value)`, raises on failure |
| Gateway timer script (every 60 s) | `triggers: [{timer: {every: 1m}}]` |
| Tag value-change event script | `triggers: [{tag_change: {paths: [...]}}]`; use `ctx.trigger.new/old` |
| `system.util.getLogger("x").info("...")` | `log.info("...", key=value)` |
| `system.net.httpPost(url, postData=...)` | `http.post(url, json=...)` with the host allowlisted in `permissions.http` |
| `system.db.runQuery(...)` | not in v1: `history.read` or an HTTP service |
| `system.tag.queryTagHistory` | `history.read(paths, start, end, agg=..., interval=...)` |
| Project script `shared.utils.f()` | `from libs import utils` |

Ignition names above are from memory (UNVERIFIED exact signatures).

## 6. Positioning, honestly

- Where Ignition is stronger and will stay stronger for a long time: breadth of drivers and modules, vendor support, client scripting and Vision/Perspective ecosystems, maturity. Sky Ranger does not compete on breadth.
- Where we can credibly be better for our target customers: isolation of user code from the control path, auditability (everything is config in revisions with a trace per run), no vendor lock-in (exportable text), edge-first, a small attack surface, readable evidence.
- Interoperability matters more than rivalry: Sparkplug B northbound lets an Ignition host consume Sky Ranger tags (messages.md section 8). Customers with Ignition upstairs and Sky Ranger on the plant floor are a real scenario.

## Decisions made
1. Scripts are isolated from the core, unlike Ignition's in-process model; we accept per-call IPC cost for containment.
2. We copy the ideas (script libraries, declarative triggers, project-stored scripts, signed extensions) and not the surface (`system.*` names, client scripting, per-tag scripts, JVM modules).
3. The extension story is narrow and out-of-process by construction, not an SDK into the core.
4. Interoperate with Ignition (Sparkplug northbound) instead of imitating it.
5. All Ignition claims in these docs stay UNVERIFIED until checked against its documentation.

## Alternatives rejected and why
- **A compatibility layer (`system.*` shim) to ease migration**: invites copying Ignition's API surface and expectations (async variants, Java types); also legally and technically a trap. A cheat sheet is enough.
- **Jython-compatible Python 2 syntax support**: no.
- **A client-script equivalent** (browser code attached to screens): no sandbox story; contract says screens are data, not code.
- **JVM or any in-process plugin API**: contradicts the process-isolation principle used for drivers.
- **Copying Ignition's licensing/module activation model**: premature before the first project (decision 5).

## Open questions
1. Verify every UNVERIFIED Ignition statement above against its current documentation (including whether Ignition 8.3 changed its scripting runtime or module signing).
2. Is a documented "porting guide" for Ignition projects (script by script) worth writing, or only tag/UDT import? Customer demand will tell.
3. Should we publish a read-only compatibility statement about what Sky Ranger scripts cannot do compared with Ignition (database access, client scripting), so integrators do not discover it late?
