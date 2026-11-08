# 07 Secure Update and Supply Chain

Ownership: the deployment doc owns *how* updates are rolled out (compose layout, migration order, version lifecycle). This file owns the *trust* requirements every update path must meet.

## 1. What gets updated

| Artifact | Format | Signed with | Verified by |
|---|---|---|---|
| Container images (`skyrangerd` incl. drivers, HMI assets, `mosquitto`, `postgres+timescaledb`, `cloud-sync`) | OCI image | cosign (Sigstore) key pair, offline-held release key | `skyranger-updater` on the box, before `compose up` |
| Offline update bundle | `skyranger-{version}.srb` = tar of OCI image layout + manifest + SBOMs + signature | cosign `sign-blob` over the manifest | `skyranger-updater` |
| Host OS | Distro packages (Debian stable assumed) | distro keys | apt |
| ESP32 firmware | `.bin` | ESP-IDF Secure Boot v2 key (RSA-3072 or ECDSA per chip support, UNVERIFIED) | the chip's ROM bootloader; gateway also checks SHA-256 |
| Python script packages (future) | wheels | out of scope here | |

Third-party base images (Mosquitto, PostgreSQL/TimescaleDB) are **rebuilt or re-tagged under our registry and signed by us** after review, so the box trusts exactly one signing identity and does not pull from Docker Hub at runtime.

## 2. Signing

Decision: cosign with a long-lived key pair rather than keyless (Fulcio/Rekor) signing.

- Why: the edge box verifies offline; keyless verification depends on the Sigstore transparency log and certificate chains that the box cannot reach. Key-based cosign verification works fully offline with the public key.
- Release key: ECDSA P-256 generated on a hardware token (YubiKey PIV via cosign's PKCS#11/PIV support, UNVERIFIED feature details per cosign version) held by the author; backup token in a safe. CI does not hold the release key; CI builds, a human signs. This is the "two-step" release: CI produces artifacts and an attestation; the release manager reviews and signs.
- Public key is baked into the OS image at `/etc/skyranger/keys/release-2026.pub`, plus the next key (`release-2027.pub`) for planned rotation. Key rotation: new releases are signed with both keys during overlap.
- Optionally also sign keyless in CI and publish to Rekor for public transparency. The box ignores it; it helps customers who want to verify independently.

Example (release machine):

```sh
cosign sign --key pkcs11:... registry.example/skyranger/skyrangerd@sha256:3f1c...
cosign attest --key pkcs11:... --type cyclonedx --predicate sbom.cdx.json registry.example/skyranger/skyrangerd@sha256:3f1c...
cosign sign-blob --key pkcs11:... --output-signature manifest.sig manifest.json
```

Verification on the box:

```sh
cosign verify --key /etc/skyranger/keys/release-2026.pub --insecure-ignore-tlog=true \
  registry.example/skyranger/skyrangerd@sha256:3f1c...
```

(`--insecure-ignore-tlog` is the cosign flag for offline verification without Rekor as remembered; UNVERIFIED for the pinned cosign version. The updater uses the cosign Go library directly so the flag name does not matter in production.)

## 3. Update bundle manifest

```json
{
  "bundle_version": 1,
  "product": "skyranger",
  "version": "0.4.0",
  "min_from_version": "0.3.0",
  "released_at": "2026-11-07T12:00:00.000Z",
  "images": [
    { "name": "skyrangerd", "digest": "sha256:3f1c...", "sbom": "sbom/skyrangerd.cdx.json" },
    { "name": "mosquitto",  "digest": "sha256:91ab...", "sbom": "sbom/mosquitto.cdx.json" },
    { "name": "postgres",   "digest": "sha256:c0de...", "sbom": "sbom/postgres.cdx.json" }
  ],
  "compose": { "file": "compose.yaml", "sha256": "..." },
  "db_migrations": { "from_schema": 12, "to_schema": 14, "reversible": false },
  "security_fixes": ["SR-2026-003"],
  "anti_rollback_floor": "0.3.2"
}
```

Rules enforced by `skyranger-updater`:
1. Signature over `manifest.json` verifies against a trusted release key, else abort.
2. Every image digest in the bundle matches the manifest; compose file references images **only by digest** (`image: .../skyrangerd@sha256:...`), never by tag.
3. `version` > current, and `version` >= `anti_rollback_floor` of the currently installed release (prevents downgrading to a version with a known vulnerability). Overriding the floor requires `admin` on the console with an audited reason.
4. Bundle is staged, images loaded into the local store, nothing restarted until all checks pass.
5. The update is an `admin` action (HMI or `skyrangerctl update apply bundle.srb`), audited with the manifest hash. Never automatic on customer sites in v1.

## 4. Rollback

A/B at the **application level** (containers), not at the OS partition level, for v1:

1. Before applying: `skyrangerctl backup create --pre-update` (encrypted, 10) and record current compose + digests as `slot A`.
2. Stop `skyrangerd`, apply DB migrations, start new images (`slot B`).
3. Health gate: `/api/v1/system/health` `ok` within `120s`, all enabled connections reach `connected` or their previous state within `300s`, no crash loop. Otherwise **automatic rollback**: stop B, restore the pre-update DB backup if migrations ran and were not reversible, start A.
4. Manual rollback within 7 days: `skyrangerctl update rollback` (same mechanism). Data written after the update is lost if a non-reversible migration is rolled back; the tool states how many hours of history that is and requires confirmation.

Host OS: Debian stable with unattended security updates **disabled by default** on customer sites (an unplanned reboot of a SCADA box is an incident); security updates applied in maintenance windows via the same audited process. An image-based OS (A/B root partitions with RAUC, Mender, or similar) is the long-term target for appliance deployments (Open questions).

ESP32: two OTA slots with ESP-IDF rollback (04 section 6).

## 5. Supply chain controls (build side)

| Control | Implementation | Gate |
|---|---|---|
| Source integrity | Signed commits or signed tags for releases; branch protection on `main`; 2FA on the code host | release |
| Reproducible-ish builds | Go `-trimpath`, `CGO_ENABLED=0` for `skyrangerd` and drivers where possible, pinned Go toolchain (`toolchain` directive in `go.mod`), `-buildvcs` metadata | release |
| Dependency pinning | `go.sum` (Go checksum DB verification on), `package-lock.json` with `npm ci`, base images by digest, Dockerfile `FROM ...@sha256:` | CI |
| Dependency hygiene | Minimal deps policy: any new direct dependency needs a reason in the PR | review |
| Vulnerability scanning | `govulncheck ./...` (call-graph aware), `npm audit --omit=dev`, image scan with Grype or Trivy against the SBOM | CI blocks on known exploitable High/Critical |
| SBOM | CycloneDX JSON per image, generated with Syft (or `cyclonedx-gomod` for Go modules) and attached as a cosign attestation and in the bundle | release |
| Provenance | SLSA-style provenance attestation from CI (GitHub Actions `actions/attest-build-provenance` or `slsa-github-generator`; level achieved is to be assessed, UNVERIFIED claim of any SLSA level) | release |
| Secrets in repo | `gitleaks` in pre-commit and CI | CI |
| Static analysis | `go vet`, `staticcheck`, `gosec` (triaged), ESLint security rules for the HMI | CI |
| Fuzzing | `go test -fuzz` targets for Sparkplug payload decoding, Modbus response parsing, config YAML parsing, tag path parsing, WS message parsing | nightly; parsers that face devices must have a fuzz target before release (14) |
| CI hardening | Pinned action SHAs, least-privilege tokens, no release key in CI | review |

SBOM format: CycloneDX chosen over SPDX for tooling breadth in vulnerability scanners; SPDX can be generated on request from the same Syft run. Customers increasingly ask for SBOMs (and EU Cyber Resilience Act obligations are coming for products with digital elements; scope and dates for Sky Ranger are UNVERIFIED and need legal review before selling in the EU).

## Decisions made
1. cosign key-based signing with an offline hardware-held release key; offline verification on the box; images referenced only by digest.
2. Third-party images are re-published and signed under our registry; the box trusts one signer.
3. Signed offline update bundles (`.srb`) are the primary update path; registry pull is optional.
4. Anti-rollback floor in the manifest; application-level A/B with health-gated automatic rollback and a pre-update encrypted backup.
5. Updates are always a human `admin` action on customer sites; no automatic updates in v1, including OS security updates.
6. CycloneDX SBOM per image, attached as attestation and shipped in the bundle; `govulncheck` and image scanning gate CI.

## Alternatives rejected and why
- **Keyless Sigstore only**: verification needs online services; breaks edge-first.
- **Docker Content Trust / Notary v1**: effectively legacy; cosign is the current boring choice.
- **Watchtower-style auto-update**: uncontrolled restarts of a SCADA system.
- **Pulling third-party images from Docker Hub on the box**: unpinned trust, rate limits, and internet dependency.
- **Full OS A/B (RAUC/Mender) in v1**: right long term for appliances, but a lot of integration work before the first customer; containers already give application rollback.

## Open questions
- Choose Grype vs Trivy; both acceptable.
- Image-based OS (RAUC, Mender, or a Debian-based immutable image) for appliance deployments: decision for the deployment doc.
- cosign PKCS#11/PIV support and offline flags are UNVERIFIED for the version to be pinned.
- EU CRA applicability and timelines: UNVERIFIED, needs legal input.
- Can CI-built Go binaries be made bit-for-bit reproducible so customers can verify independently?
