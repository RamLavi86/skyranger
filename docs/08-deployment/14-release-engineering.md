# 14 Release Engineering and Versioning Policy

Binding inputs: [README.md](../00-contracts/README.md) (versioned surfaces, additive rule), [api.md](../00-contracts/api.md) section 8 (it delegates the API lifecycle policy to this document), [driver-interface.md](../00-contracts/driver-interface.md) section 8 (driver conformance). Consumers: [04-install-and-first-boot.md](04-install-and-first-boot.md), [06-updates-and-rollback.md](06-updates-and-rollback.md).

## 1. What is versioned

| Surface | Version marker | Changes when |
|---|---|---|
| **Product release** | semver `MAJOR.MINOR.PATCH` (`0.4.0`) | every release; the number in `GET /api/v1/system/info`, the bundle name, `release.lock` |
| REST API | `/api/v1` | breaking only: new path major |
| WebSocket | subprotocol `skyranger.v1` | breaking only |
| Config schema | `apiVersion: skyranger/v1` | breaking only |
| Internal bus, events | `skyranger/v1/...`, `skyranger.<area>.<name>.v1` | breaking only |
| Driver gRPC | package `skyranger.driver.v1` | breaking only |
| Database schema | numbered migrations, each with `compat` | every migration ([07-data-and-migrations.md](07-data-and-migrations.md)) |
| Release manifest | `manifest_version` | when this document's manifest schema changes |
| `skyrangerctl` | same as the product release | shipped inside the bundle; host and bundle must match, see 06 |

The contract versions (`v1`) and the product semver are independent. Product `3.2.0` may still speak `/api/v1`. The product major bumps for **operator-visible breaks** (below), not only for contract majors.

## 2. Semver rules for the product

- **PATCH** (`0.4.1`): bug and security fixes only. No schema migration unless it is `compat: backward` and required by the fix. No new config fields. Always safe to apply and to roll back.
- **MINOR** (`0.5.0`): new features, additive API/config fields, new migrations (`backward` preferred; `breaking` allowed but flagged in the changelog and the manifest). Upgrade path guaranteed from the **two previous minors** and from the current LTS (section 5).
- **MAJOR** (`1.0.0` , `2.0.0`): any of: removing a deprecated API/config/driver surface, a new contract major served alone, a data migration that cannot be rolled back and is not reversible by restore alone, dropping a supported host OS, a change to the host directory layout. Upgrade only from the last minor of the previous major.
- **0.x**: any minor may contain breaking changes, but the *process* is identical: migrations declare `compat`, manifests declare `min_from`, changelog lists breaks. We do not use 0.x as an excuse to skip the upgrade tests. The first release we recommend to a customer without us on site is 1.0.

## 3. Deprecation and removal policy (this fulfills api.md section 8)

| Thing | Announced | Earliest removal | Mechanism |
|---|---|---|---|
| REST endpoint, WS message, config field, event type, bus topic | minor N changelog + `Deprecation`/`Sunset` response headers (api.md) + a `skyranger.system.deprecated_use.v1` event suggestion (see Open questions) | minor N+2 **and** at least 6 months later | still works, returns headers |
| Whole contract major (`/api/v2` replaces `/api/v1`) | at v2 general availability | v1 served alongside v2 for **at least 12 months and two minors**; removed only in a product major | both served by one `skyrangerd` |
| Host OS version (Debian N) | when the next Debian stable is supported | when Debian security support ends (UNVERIFIED dates; check at the time) | `preflight` warns, then refuses |
| First-party driver | N | N+2 and 6 months | `Connection` validation warns |
| `skyrangerctl` flags | N | N+1 | prints warning |

Deprecation never removes data export (exit plan): export formats are stable surfaces, see [12-exit-plan.md](12-exit-plan.md).

## 4. Release channels

| Channel | Source | Who | Promise |
|---|---|---|---|
| `nightly` | `main`, every night | us, simulator bench | none; may not install over an existing box |
| `candidate` | a tagged `X.Y.Z-rc.N` | pilot sites ([08-fleet.md](08-fleet.md) ring 1), our own test boxes | passed every automated gate, awaiting soak |
| `stable` | tagged `X.Y.Z` | customers | passed all gates plus the 72 h soak on at least one real box |
| `lts` | a `stable` minor we designate | customers who want fewer changes | patch releases only, see below |

A box follows exactly one channel (`site.yaml`/fleet desired state). Without a fleet, the channel is just "which bundle the integrator downloads". Channel names appear in the manifest, so a box refuses a bundle from a lower channel than configured (a `stable` box does not install `nightly`).

## 5. Support windows (honest, small-team numbers)

We are an integrator first; promises must be ones one or two engineers can keep.

- **Current stable minor**: full fixes (bugs, security).
- **Previous stable minor**: security and data-loss fixes only, for **6 months** after the next minor ships.
- **LTS** (first designation at 1.0, then roughly every 12 months, only if a customer needs it): security and data-loss fixes for **24 months**.
- Customers under a support agreement get what the agreement says; the table above is the default for everyone else.
- Upgrade path guaranteed from: the previous two minors and the current LTS (manifest `min_from` encodes this). Skipping further back requires stepping through an intermediate release; `skyrangerctl update` computes the steps and refuses a direct jump the manifest does not allow.
- Security response targets (to confirm we can staff them): acknowledge in 3 business days, fix for critical issues released within 14 days, advisory published with the fix. UNVERIFIED as a commitment; these are goals.

## 6. The release bundle and manifest

One bundle per architecture ([04-install-and-first-boot.md](04-install-and-first-boot.md) section 1). `manifest.json` is the only thing the installer, updater, fleet and `preflight` trust:

```json
{
  "manifest_version": 1,
  "product": "skyranger",
  "version": "0.5.0",
  "channel": "stable",
  "arch": "amd64",
  "built_at": "2026-11-07T14:03:22.123Z",
  "git_commit": "9f2c1d4",
  "signing_key_id": "release-2026-a",
  "images": {
    "skyrangerd":  "ghcr.io/<org>/skyrangerd@sha256:<digest>",
    "postgres":    "timescale/timescaledb@sha256:<digest>",
    "mosquitto":   "eclipse-mosquitto@sha256:<digest>",
    "simulator":   "ghcr.io/<org>/skyranger-sim@sha256:<digest>"
  },
  "versions": { "postgresql": "17.x", "timescaledb": "2.x", "mosquitto": "2.0.x" },
  "contracts": { "api": ["v1"], "config": ["skyranger/v1"], "driver": ["v1"] },
  "migrations": [
    { "id": "0041_add_alarm_shelve_reason", "compat": "backward", "est_duration": "2s" },
    { "id": "0042_split_events_hypertable", "compat": "breaking", "est_duration": "5m", "needs_backup": true }
  ],
  "db_major": { "required": 17, "upgrade_from": [16] },
  "upgrade": { "min_from": "0.3.0", "rollback_to_min": "0.4.0" },
  "host": { "debian": ["13", "12"], "min_docker": "<UNVERIFIED>", "min_compose": "<UNVERIFIED>" },
  "sbom": "SBOM.spdx.json",
  "files": { "compose.yaml": "sha256:<h>", "images.oci.tar": "sha256:<h>", "skyrangerctl": "sha256:<h>" }
}
```

Field notes:
- `images` are digests; tags are not accepted. The same digests are what [02-containers.md](02-containers.md) pins in `compose.yaml`.
- `migrations` lists every migration **new in this release relative to `min_from`**, with `compat`. Update logic (06) reads it to decide whether automatic rollback keeps data.
- `upgrade.rollback_to_min`: the oldest release that can run against the schema this release leaves behind if every listed migration is `backward`.
- Numbers like `17.x` are placeholders until the DB images are chosen.

## 7. Build pipeline

Principles: a build is a `make release` on any machine with Docker; CI is just a runner for the same make targets (no pipeline logic that cannot be run locally; the author is a one-person shop and CI vendors change).

```
make test           go test ./..., driver conformance (driver-interface.md section 8), HMI unit tests
make images         docker buildx, linux/amd64 + linux/arm64, -trimpath, pinned Go toolchain and base image digests
make sbom           SPDX per image (tool choice UNVERIFIED: syft is a common one)
make scan           vulnerability scan of images and the Go module graph (tool choice UNVERIFIED: trivy, grype, govulncheck)
make bundle         assemble bundle dir, write manifest, tar
make sign           minisign the manifest and the tar (offline key, section 8)
make verify-release the whole gate list below, against the signed bundle
make publish        push images to the registry by digest, upload bundle + signature
```

Reproducibility is a **goal, checked, not promised**: two builds of the same commit should produce identical `skyrangerd` binaries (Go with `-trimpath`, fixed `SOURCE_DATE_EPOCH`, vendored modules). Image layers may differ (timestamps in base images); we pin by digest and compare binary hashes.

## 8. Signing and keys

Two separate trust chains, matching [03-packaging-and-hardware.md](03-packaging-and-hardware.md):

| Chain | Signs | Algorithm and tool | Verified by |
|---|---|---|---|
| **Release signing key** | `manifest.json`, the bundle tar, and the fleet's signed desired-state documents (08) use a different key (see 08) | Ed25519, `minisign` (simple, one small file, verifiable offline with a public key embedded in `skyrangerctl` and `install.sh`) | `install.sh`, `skyrangerctl install/update/preflight` |
| **RAUC bundle CA** (appliance only) | RAUC OS bundles | X.509 per RAUC | RAUC on the device |

Decision between minisign and cosign: 04 left it open. We choose **minisign for bundles and manifests**: no transparency log or network dependency to verify at an air-gapped site, a signature is one text file. cosign is better for per-image signatures in registries; we may also sign images with cosign for customers who run their own admission policy, but the install path never depends on it.

Key handling:
- The release key is generated offline, stored on a hardware token or an encrypted offline medium (UNVERIFIED that minisign works with a hardware token; if not, wrap the key file in an encrypted store and sign on an air-gapped machine). Never in CI. CI produces the unsigned bundle; a human signs after the gates pass.
- `signing_key_id` in the manifest selects a public key from `/usr/share/skyranger/keys.d/*.pub` on the host. **Rotation**: release N is signed by the old key and ships the new public key in its `keys.d`; release N+1 may be signed by the new key. Boxes that miss a rotation by more than one release need a manual `skyrangerctl keys import` with an out-of-band fingerprint check.
- **Compromise procedure**: publish a signed (by a pre-generated, offline backup key whose public half is embedded from the first release) revocation list `keys.d/revoked`; the next release carries it; `install.sh` documentation lists the fingerprint check by phone/in person for the first install.
- Every release is also tagged and signed in git (`git tag -s`).

## 9. Release gates (all must pass before `candidate`; the soak before `stable`)

| Gate | What | Automated |
|---|---|---|
| Unit + conformance | Go tests, driver conformance suite, contract tests that check JSON against the examples in 00-contracts | yes |
| Simulator integration | full compose with `sim`, scenario scripts, fault injection (dropped connections, stale values, bad quality, bursts: decision 7) | yes |
| Fresh offline install | clean Debian VM, no network, bundle only, `install` + `init --from` + commission the sim meter | yes |
| **Upgrade matrix** | from each supported `min_from` release (previous two minors, LTS) with a **golden database** (realistic size) to this release; health gate passes; counts of tags, history rows, alarms unchanged | yes |
| **Rollback matrix** | for every `backward` migration set: update, then roll back to `rollback_to_min`, health gate passes; for `breaking`: update, roll back via restore, verify restored data | yes |
| Config round trip | export, import, export gives an empty diff; golden repos from `docs` apply cleanly | yes |
| **Exit test** | `skyrangerctl export` from the sim site, load history into **plain PostgreSQL without TimescaleDB** and the config into a fresh gateway; row counts match ([12-exit-plan.md](12-exit-plan.md)) | yes |
| Power-cut soak | cycle power (relay rig or VM hard-reset loop) 500 times during sim load; PostgreSQL recovers, no corruption, history gaps within the documented bound ([11-time-and-power.md](11-time-and-power.md)) | VM loop yes; real box rig before each hardware qualification |
| 72 h soak | one real box (or VM) with sim at production rate: memory flat, disk growth predicted, no restarts | yes, unattended, results reviewed by a human |
| Update on A/B | appliance only: RAUC install, forced bad boot, confirm fallback | yes, on a real appliance |
| Security | image and Go module scan has no unwaived critical findings; SBOM present; licenses allowed | yes |
| Docs | changelog entry, migration notes, UNVERIFIED items touched by the release are resolved or still marked | human checklist |

A gate failure blocks promotion. Waivers are written in the release PR, with a reason.

## 10. Hotfixes and withdrawing a release

- A hotfix is a PATCH on the release branch `release/X.Y`, cherry-picks only, running the **same gates** (the soak may be shortened to 24 h for a security fix with sign-off in the PR).
- A bad release is **withdrawn** by publishing a signed `advisories.json` entry (`{"version": "0.5.0", "action": "do_not_install", "reason": "..."}`) that `skyrangerctl update` and the fleet service check, and by moving the channel pointer back. Already installed boxes are not force-rolled back; the advisory tells them to update to the fix.
- The changelog follows keep-a-changelog headings (Added, Changed, Deprecated, Removed, Fixed, Security) plus mandatory sections **Migrations** (id, compat, duration) and **Operator actions**.

## 11. Distribution and no-lock-in

- Registry: a public OCI registry for convenience (`ghcr.io/<org>`; placeholder, org not chosen). It is **never required**: bundles contain all images (`images.oci.tar`).
- Bundles, signatures and `advisories.json` are plain files on a static HTTPS site and may be mirrored by the customer. `skyrangerctl update --source <url|dir>` accepts any mirror; trust comes from the signature, not the host.
- Source is published under an open license (not yet chosen; decision 5 defers licensing). The customer's right to keep running and rebuilding the release they have is part of the exit plan, so the license choice is a prerequisite for 1.0.

## Decisions made
1. Product semver is independent of contract versions; the product major bumps for operator-visible breaks. 0.x follows the same process as 1.x.
2. Deprecation: removal no earlier than two minors and six months; a contract major is served alongside its predecessor for at least 12 months and two minors. This closes the lifecycle question in api.md section 8.
3. Four channels (`nightly`, `candidate`, `stable`, `lts`); a box follows one.
4. Support windows: current minor full, previous minor security for 6 months, LTS for 24 months; upgrade path from the previous two minors and LTS.
5. One signed manifest of digests, migrations with `compat`, and `min_from`/`rollback_to_min`; minisign (Ed25519) for bundles and manifests, X.509 only for RAUC.
6. Builds are `make` targets runnable anywhere; signing is a human step on an offline key.
7. Upgrade matrix, rollback matrix, exit test and power-cut soak are release gates, not nice-to-haves.
8. Withdrawal by signed advisory, not by silent removal.

## Alternatives rejected and why
- **Calendar versioning**: nice for cadence, hides which releases change data or break contracts. Semver communicates compatibility, which is what operators need.
- **Rolling `latest`**: no reproducibility, no rollback target, no support boundary.
- **cosign keyless (Sigstore) for the install path**: depends on online transparency infrastructure and OIDC identity at verification time; air-gapped sites cannot use it.
- **GPG**: works, but key handling and UX are worse than minisign for a single signature on a manifest.
- **Signing in CI**: a CI compromise would sign malware that every box trusts. A human on an offline key is slower and safer at this scale.
- **Long support windows now**: promises we cannot staff. They can be extended by agreement per customer.
- **Auto-promote candidate to stable by time**: stable means a human looked at soak results.

## Open questions
- Contract addition (events catalog): `skyranger.system.deprecated_use.v1` (an event when a deprecated surface is called) is proposed, not agreed.
- Open source license and contributor terms (decision 5 deferral) must be chosen before 1.0; it decides what "customers can rebuild it" means.
- Registry and org name (`<org>` placeholder), and whether we host a public mirror at all.
- Tooling picks marked UNVERIFIED: SBOM and scanner, minisign with a hardware token, Debian support end dates, minimum Docker/Compose versions (needs testing of `up --wait`, `required: false` in 02).
- Whether `candidate` boxes at customers (pilot ring) are acceptable to any customer; if not, ring 1 is our own boxes only.
- Whether PATCH releases may carry `compat: backward` migrations (section 2 allows it; stricter is simpler to reason about).
