# 10 SDK surfaces, compatibility policy, licensing considerations

Status: design draft. Conforms to [../00-contracts/](../00-contracts/README.md), whose versioning rules are the base ("Breaking change = new major everywhere that changed. Additive fields are not breaking; all consumers MUST ignore unknown fields", contracts README; api.md section 8). Licensing is **not legal advice**; decisions.md 5 defers the licensing model until after two or three real projects. This file lists considerations and a default so that nothing built before then forecloses the options.

## 1. Versioned surfaces

Everything a third party can depend on is an explicit, separately versioned surface. Anything not listed here is internal and may change in any release.

| # | Surface | Identifier | Versioning | Authority file |
|---|---|---|---|---|
| 1 | REST API | `/api/v1` | major in path, additive within | `openapi.json`, api.md |
| 2 | WebSocket | subprotocol `skyranger.v1` | major in name | api.md |
| 3 | Event envelope and catalog | `skyranger.<area>.<name>.v1` | per event type; additive data fields | messages.md |
| 4 | Internal MQTT bus | `skyranger/v1/{gateway_id}/...` | major in topic | messages.md |
| 5 | Driver protocol | gRPC `skyranger.driver.v1` | package major; proto rules (never renumber) | driver-interface.md, `driver.proto` |
| 6 | Script host protocol | gRPC `skyranger.script.v1` | package major | `script.proto` (internal between our two binaries; not a third-party surface) |
| 7 | Notify, service protocols | `skyranger.notify.v1`, `skyranger.service.v1` | package major | `.proto` files |
| 8 | UI component protocol | `skyranger.ui.v1` (postMessage) | major in `init` message | v3 doc |
| 9 | Config kinds | `apiVersion: skyranger/v1` | additive within; new kinds additive | tag-model.md section 9 |
| 10 | **Python script API** | `api_level` (integer) + `skyranger` package semver | see section 2 | 02-script-api.md |
| 11 | **Go SDK** | Go module `.../skyranger/sdk` (placeholder, contracts README: module path not chosen) | Go module semver; v2+ uses `/v2` path | package docs |
| 12 | Plugin manifest | `kind: Plugin`, `apiVersion: skyranger/v1` | additive within | 09-plugins.md |

The Go SDK is a **small, separate module** from the core (`sdk/driver`, `sdk/notify`, later `sdk/service`), not a window into core packages. The core's `internal/` packages stay unimportable by Go rules. Internal shared types needed by SDK users (`tag.Variant`, `tag.Quality`) are re-exported from `sdk` (today they live at `skyranger/pkg/tag` and `skyranger/pkg/driver` in the contracts; whether `pkg/` is the SDK or becomes it is open question 1).

## 2. Python script API compatibility

- **`api_level`:** an integer, starting at 1, incremented only for **breaking** changes to the script-visible API (removed or changed function, changed exception behavior, changed default). Additive changes (new function, new optional parameter, new exception subclass, new trigger attribute) do not change it.
- A `Script` declares `spec.api_level: N` (03). The host's worker runtime supports a **range** of levels and routes through a thin shim per level; scripts written for level N keep running while level N is supported.
- **Support window:** each api_level is supported for at least **24 months after the release that introduces the next level**, and never less than 2 minor gateway releases (UNVERIFIED as a number; proposed, open question 3). Deprecation is announced in release notes and surfaced at draft validation as a **warning** (`api_level_deprecated`), then an error once unsupported.
- The `skyranger` Python package semver mirrors this: major = api_level. The testing package pins the same major.
- **Within a level**, the Python side tolerates unknown fields in host replies (forward compatible, like all consumers) and the host tolerates unknown optional args from older runtimes.
- **Python language version:** the pinned CPython minor changes in gateway releases; this is a platform change, not a script API change, but it can break scripts (removed stdlib modules, syntax). Policy: the CPython minor changes at most once per gateway **minor release train** (not in patches), is called out in release notes, and the draft validator runs a `compile()` check against the new interpreter on upgrade (`script invalid` is visible before cutover; update tool dry-runs scripts first, 07-secure-update.md owns update flow).
- **Import allowlist** growth is additive. Shrinking it is breaking and requires an api_level bump.

## 3. Protocol and surface compatibility

| Rule | Detail |
|---|---|
| Additive within a major | New fields, enum values, endpoints, events are not breaking. Consumers ignore unknown fields and tolerate unknown enum values (contracts). |
| Breaking = new major alongside | `skyranger.driver.v2` is served by the host **in addition to** v1 for the support window. Plugins declare the majors they speak in `requires.protocols`. |
| Support window | Gateway supports protocol majors N and N-1 for **at least 12 months** after N is introduced (UNVERIFIED as a number; the contract says only "at least one minor release cycle" for REST, api.md section 8; plugin authors need a longer window than HMI clients: proposed, open question 3). |
| Deprecation signalling | REST: `Deprecation`/`Sunset` headers (api.md; the Deprecation RFC number is UNVERIFIED there). Protocols: `GetInfo`-time warnings in the host log and `_system/plugins/.../deprecated` tag. Scripts: draft validation warnings. |
| Plugin compat check | At install and at every gateway upgrade: `requires.skyranger` semver range and protocol majors checked; incompatible plugins are disabled (not deleted), connections that depend on them become `faulted` (driver-interface.md), and the update tool reports it before cutover. |
| Gateway semver | `MAJOR.MINOR.PATCH`. Gateway major bumps only with removal of a surface major from the support window. |
| Config | Existing `apiVersion: skyranger/v1` documents continue to import for the support window; new fields are optional. A migration tool (`config migrate`) ships with any config major. |
| Never reuse | Field numbers, event type strings, error codes, quality codes (tag-model.md: codes 192+ reserved). |

**Conformance as the compatibility test.** Every surface a third party implements ships a conformance suite (the driver one exists: driver-interface.md section 8). A plugin that passes the suite for protocol vN is considered compatible with every gateway that supports vN. CI runs the previous release's conformance-passing reference plugin against the new gateway.

## 4. Tooling for plugin and script authors (v3)

- **Templates** for a Go driver, a notification channel, a script library, and a script project (pytest + split layout).
- **`skyranger-sdk` docs generated from the protos and OpenAPI**; golden-file examples (the contracts' golden corpus, 09-testing/01).
- **A `plugin lint`** (validates manifest, hashes, permissions are minimal, protocol majors, SBOM present) and **`plugin sign`** (Ed25519 over manifest) tools. The signing key stays with the publisher.
- **Reference implementations** maintained by the project: the Sparkplug and Modbus drivers use the public SDK too (dogfooding: no private driver API).

## 5. Licensing considerations for open core

Not legal advice. Facts about licenses below are general and from memory; verify with a lawyer before publishing.

### 5.1 What the project already promised

- README and decisions.md 3: open standards, **customers can always walk away with data and config**; vendor-neutral; "a selling point against Siemens/Rockwell lock-in".
- Decisions.md 5: licensing deferred until after real projects.
- The author is an integrator first: customers receive a deployed system and may have it maintained by someone else.

Any license choice that makes a customer's deployment legally awkward (network-copyleft surprises, source-available restrictions on commercial use) undermines the pitch.

### 5.2 Options

| Option | Core | Extensions/SDK | Pros | Cons / risks |
|---|---|---|---|---|
| A. Permissive core + proprietary commercial add-ons | Apache-2.0 (or MIT) | SDK Apache-2.0; commercial plugins closed | Maximum adoption, no objections from integrators or customers' legal, patent grant (Apache-2.0), plugin authors choose any license | A cloud vendor or competitor can host it without contributing; revenue only from add-ons, support, hosting |
| B. Network-copyleft core + commercial license (dual) | AGPL-3.0 | SDK permissive | Strong protection vs hosted forks; dual licensing sells exceptions | Customers' legal teams dislike AGPL; integrators worry about "derivative work" for scripts and plugins; requires copyright ownership (CLA) to dual-license |
| C. Source-available (BSL/FSL style) core | non-OSI license with time-delayed open conversion | SDK permissive | Protects commercial position | Not "open source" by OSI definition: conflicts with "open" positioning and some customers' policies; community trust cost |
| D. Open core split by feature | Apache-2.0 core, closed enterprise features (SSO, multi-site, HA) | SDK Apache-2.0 | Common and understood business model | Constant tension about what is "core"; features customers consider essential (audit, backup) must not be paywalled given the trust pitch |

### 5.3 Default direction (until decisions.md 5 is revisited)

1. **Contracts, protocols (`.proto`), SDKs, Python `skyranger` and `skyranger.testing` packages, schemas: Apache-2.0.** These are what third parties build against; a permissive license with a patent grant removes friction and keeps plugin authors free to choose their own license, including commercial. This is the most important decision to keep stable.
2. **Core gateway and drivers: Apache-2.0 leaning** (option A/D), final choice deferred. Options B and C stay possible only if copyright ownership is kept clean now (below); relicensing an Apache-2.0 codebase later for *new* versions is legal, but changes trust and community.
3. **Do not paywall safety, audit, backup/restore, export, or security features.** They are the trust pitch.
4. Commercial revenue paths compatible with all of the above: integration services (the current plan), support and maintenance contracts, signed/certified plugin listings, hosted cloud sync, training. (UNVERIFIED which will prove viable; decisions.md 5.)

### 5.4 Housekeeping that is cheap now and expensive later

| Item | Why | Action |
|---|---|---|
| **Copyright ownership / contributor terms** | Dual licensing or relicensing needs rights from every contributor | Decide **DCO vs CLA** before the first external contribution; until then single-author, keep a `CONTRIBUTING.md` saying so |
| **AI-agent-written code** | Parts of the code are produced by AI agents (09-testing/12-ai-agent-guardrails.md); copyright status of generated code differs by jurisdiction and is unsettled (UNVERIFIED); license compatibility of any training-derived snippets is unknowable | Human review and authorship record per commit; do not rely on exclusive-ownership claims for dual licensing without legal advice |
| **Dependency licenses** | Copyleft deps (GPL, AGPL) in the core would constrain the choice | Allowlist in CI (SPDX scan via the SBOM, 07-secure-update.md): Apache-2.0, MIT, BSD, ISC, MPL-2.0 (file-level copyleft: review), no GPL/AGPL in linked code. Go dependencies and CPython (PSF license, permissive) are fine in principle; **each dependency's license must be checked, not assumed (UNVERIFIED for the full tree)**. Mosquitto's license and TimescaleDB's split licensing (some features under a non-OSI license) need a specific review since the stack depends on them (UNVERIFIED details; check before shipping a product that bundles TimescaleDB features) |
| **Plugin license boundary** | Out-of-process plugins talking gRPC are separate programs, which reduces (does not eliminate) copyleft entanglement questions | Document the intended boundary in the SDK README; get legal review before relying on it |
| **Trademark** | "Sky Ranger" name use by forks/plugins | Register or at least publish a trademark policy before a plugin ecosystem exists; check name availability (UNVERIFIED) |
| **Plugin submission terms** | Third-party plugin authors' licenses are their own | The listing policy requires a declared SPDX license in the manifest (09) |

### 5.5 Customer-facing statements to be able to make

- "All your configuration, tags, history and scripts export as plain text and standard formats." (contracts)
- "Scripts and plugins you or we write for you belong under the terms you contract; the platform's SDK license permits proprietary plugins." (requires the SDK license choice above)
- "No license server or phone-home is required for the gateway to run." (edge-first)

## Decisions made
1. Every third-party-visible surface is an explicit, separately versioned contract; the Go SDK is a small separate module, not core internals; the project's own drivers use it.
2. Python script API has an integer `api_level` declared per script, supported as a range with a stated window; breaking only by bumping it; draft validation warns before removal.
3. Protocol majors are served side by side; plugin compatibility is checked at install and at gateway upgrade; conformance suites define "compatible".
4. Plugin and script-library compatibility with the pinned CPython minor is managed by release-train rules and pre-cutover validation.
5. Default license direction: Apache-2.0 for contracts, protocols, SDKs and Python packages now; core license leaning Apache-2.0 but formally deferred (decisions.md 5); no paywalling of safety, audit, backup, export, security.
6. Decide contributor terms (DCO vs CLA) before the first external contribution.
7. Dependency license allowlist enforced in CI from the SBOM.

## Alternatives rejected and why
- **Semver only, no `api_level`**: script authors read the gateway version, not the Python package version; a single integer per script is explicit and checkable at validation.
- **Breaking script API changes with a one-release grace**: integrators have deployed systems with tested scripts; they need a longer window.
- **Exposing core Go packages as the SDK**: couples plugin authors to core internals and makes every refactor a breaking change.
- **Dynamic Python API negotiation at runtime** (feature flags): untestable matrix; one integer level is enough.
- **AGPL core by default**: the customer-legal-department friction and the CLA requirement outweigh the protection for a small project at this stage (revisit if hosted-fork risk materializes).
- **BSL-style license**: conflicts with "open" positioning.
- **Deciding the license now**: decisions.md 5 explicitly defers it; only the cheap, reversible steps are taken now.

## Open questions
1. Is `skyranger/pkg/...` (contracts) the public SDK, or a larger package set from which a smaller `sdk` module is carved? The contracts say the module path is unchosen.
2. DCO or CLA, and when (before any external PR).
3. Support windows (24 months for script api_level, 12 months for protocol majors) are proposals with no data behind them; they cost real maintenance. Pick numbers after the first customer deployment cycle (typical plant update cadence is slow, UNVERIFIED).
4. License of TimescaleDB-dependent features and bundled components (verify before distributing images; consider plain PostgreSQL fallbacks).
5. Whether to offer a "verified plugin" program and who staffs it.
6. Trademark and name clearance for "Sky Ranger".
7. Whether the Python `skyranger` package should be published on PyPI under a name guaranteed free (name availability UNVERIFIED).
