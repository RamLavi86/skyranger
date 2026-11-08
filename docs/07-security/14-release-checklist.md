# 14 Pre-Shipping Security Checklist

This is the gate. Two levels, both must pass:

- **Part A: release gate**, once per release candidate. Mostly automated in CI; a human signs.
- **Part B: site gate**, once per customer deployment (and re-run after major changes). A human with the box in front of them, plus an external scan.

A box is not shipped, and a release is not signed ([07-secure-update.md](07-secure-update.md)), with any unchecked item unless it carries a written, dated exception signed by the author and attached to the commissioning report. "N/A" requires a reason. Items cite the file that defines the requirement. Tool names are examples of boring tools, not mandates.

## Part A: release gate

### A1. Build and supply chain (07)
- [ ] Release built from a signed tag on `main` (branch protection, 2FA on the code host).
- [ ] Container images referenced by digest, signed with the offline release key (cosign), update bundle manifest signed; signature verified on a clean machine with only the pinned public key.
- [ ] SBOM (CycloneDX) generated per image and attached; no unresolved High/Critical findings from `govulncheck`, `npm audit --omit=dev`, and the image scan (Grype or Trivy), or each has a written "not exploitable because" note.
- [ ] `gitleaks` clean on the release range; no secrets, test tokens or private keys in images (scan image layers).
- [ ] Dependency changes since the last release reviewed (reason recorded per new direct dependency).
- [ ] CI uses pinned action versions, least-privilege tokens, and holds no release key.

### A2. Authentication and authorization (02)
- [ ] Automated test iterates every route in `openapi.json` with a token of each role and asserts `403 forbidden` exactly where the permission table says; router refuses to start with an undeclared permission.
- [ ] No default credentials in any image or config (grep for known defaults; first boot requires the one-time setup code; setup endpoint returns `404` after completion).
- [ ] Login throttling, per-IP limits and WebSocket `Origin` check tested (02 section 4 and 8); no username enumeration (identical responses and timing within tolerance for unknown user and wrong password).
- [ ] Token and API key formats redacted in logs (log scrubber test with `sr_s1_` and `sr_k1_` samples); tokens never in URLs (grep access logs from the end-to-end suite).
- [ ] Revocation tests: disabling a user closes their WebSockets with `4401` within seconds; deleted API key stops working immediately (cache cleared).

### A3. Input handling and robustness (07 section 5, 04)
- [ ] Fuzz targets exist and ran at least 1 hour each in the last week for: Sparkplug payload decode, Modbus response parsing, config YAML parsing, tag path parsing, WebSocket message parsing, HMI document (Screen/Symbol) parsing and SVG sanitizer. No unresolved crashers.
- [ ] Any parser that faces devices has a fuzz target (no exceptions).
- [ ] SQL access uses parameterized queries only (static check or grep gate).
- [ ] Symbol SVG sanitizer test corpus (scripts, `on*`, `href`, `foreignObject`, entity tricks) all stripped (05-hmi-api/02 section 6).
- [ ] HMI: strict CSP header present with no `unsafe-inline` script or `unsafe-eval`; Playwright test asserts no CSP violations on every route; `dangerouslySetInnerHTML` lint rule passes; a tag with `display_name` `<img src=x onerror=alert(1)>` renders as text.

### A4. Transport and crypto (03)
- [ ] TLS scan of the HTTPS listener and the MQTT listener (testssl.sh or `sslyze`): TLS 1.2+ only, no weak ciphers, certificate chain valid against the site CA, HSTS present.
- [ ] No plaintext MQTT listener, no websockets listener in the Mosquitto config shipped.
- [ ] PostgreSQL requires TLS and SCRAM; `pg_hba.conf` has no `trust`.
- [ ] gRPC driver sockets: directory `0750`, peer-UID check test passes (a non-host process cannot call `Write`).
- [ ] No custom cryptography: crypto imports limited to the standard library, `golang.org/x/crypto`, `filippo.io/age`, `go-tpm` (grep gate).

### A5. Audit, secrets, backup (06, 08, 10)
- [ ] Audit chain: after the end-to-end suite, `skyrangerctl audit verify` passes; a test that edits a row directly is detected; checkpoint signatures verify.
- [ ] Audit write failure test: mutating requests fail closed (`500`) and monitoring continues.
- [ ] Secrets: write-only API (no value is ever returned or exported); support bundle excludes secret files and PKI store and redacts `sr_` tokens and PEM blocks (test).
- [ ] Config export (`GET /config`, `GET /hmi`) contains no secret values (test with a known secret string).
- [ ] KEK mode `plain` is rejected by the production profile.
- [ ] Backup create, decrypt and full restore drill passed on the release candidate; manifest signature verified; restore refuses a tampered file (10 section 6).

### A6. Device side (04)
- [ ] ESP32 production firmware build has Secure Boot v2 and flash encryption enabled (check eFuse configuration on a sample device); signing key never in CI.
- [ ] Per-device credential and ACL test: device A cannot publish on device B's topics, cannot subscribe to `#`, cannot publish on `skyranger/v1/...` or `spBv1.0/STATE/...` (Mosquitto ACL test suite).
- [ ] Device clone and flapping detection triggers the security event at more than 10 births per hour.
- [ ] OTA: unsigned or wrong-hash image rejected; rollback to previous slot works; anti-rollback behavior documented.

### A7. Documentation and process
- [ ] Threat model (01) re-reviewed for any new boundary in this release; changes recorded.
- [ ] Safety statement (11) present in the HMI About page and the release notes template.
- [ ] Release notes list security fixes with advisory IDs (13); no open High/Critical advisory against this release.
- [ ] Version, commit, SBOM link and security contact visible in the HMI About page.
- [ ] Penetration test: completed on a staged deployment before the **first** paid customer, and re-done after major architecture changes (Open questions: external vendor).

## Part B: site gate

### B1. Network (05)
- [ ] Network diagram and zone table filled in for this site; every conduit has an owner.
- [ ] Firewall rules match the allow list in 05 section 3 (exported rule set attached); default deny; no inbound from the internet; no port forwards, no UPnP.
- [ ] External port scan from the office VLAN and from the field VLAN (`nmap` full TCP, and the UDP ports expected): only the ports in 05 section 3 are open, nothing else (attach output). Host firewall verified independently of the site firewall, including container published ports (Docker/Podman bypass check).
- [ ] Field devices cannot reach each other (client isolation on the AP verified with a test device) or the internet.
- [ ] Remote access: disabled by default, enable/disable window tested, auto-expiry tested; no TeamViewer, AnyDesk or similar installed on the edge box or the HMI PCs it controls.
- [ ] NTP source reachable and `chronyc tracking` offset small; field VLAN gets time from the gateway.

### B2. Edge box (09, 06)
- [ ] UEFI Secure Boot on; firmware password set; USB/network boot off (photo or screen capture in commissioning report).
- [ ] LUKS enabled with TPM unlock and recovery key recorded; reboot test comes back unattended; the KEK mode is `tpm` or `luks-keyfile`, never `plain`; recovery kit created and stored in two places.
- [ ] Enabled services list matches 09 (`systemctl list-unit-files --state=enabled`); no extra listeners (`ss -tulpn` compared with the expected list).
- [ ] Root locked; no password SSH; SSH key-only from the management subnet or disabled; local console password recorded in the kit.
- [ ] Containers: non-root, read-only root FS, dropped capabilities, no `--privileged`, no host network (inspect output attached).
- [ ] Host patch level and image digests recorded; automatic OS updates disabled (maintenance windows per 07).
- [ ] Physical: cabinet locked, unused USB ports disabled.

### B3. Identity and access (02)
- [ ] First-boot setup completed with the one-time code; code file removed; setup endpoint `404`.
- [ ] Named accounts for each human; no shared accounts, except kiosks which have their own `viewer` identity or key; break-glass procedure documented and tested once.
- [ ] Roles assigned by need; `admin` accounts minimized (at most two); admin MFA enabled if v1.1 is installed; service account keys have expiry or are flagged in the report.
- [ ] Session lifetime and idle timeout set; kiosk API key (viewer) scoped with `allowed_cidrs` to the kiosk host where practical.

### B4. PKI, devices, brokers (03, 04)
- [ ] Site CA created; root key exported to the kit and deleted from the box (or the weaker posture recorded and accepted by the customer); root certificate fingerprint recorded in the commissioning report and installed on the HMI PCs.
- [ ] HMI opens without certificate warnings on every client PC; certificate expiry alarms visible (`cert_expiry_days`).
- [ ] Every ESP32 has its own credential (provisioned over USB or approved SoftAP), serial number recorded against `edge_node_id`, and `fw_secure: true` for production units; unknown devices stay in discovery unapproved (decision 4).
- [ ] Any Connection with `allow_insecure: true` is justified in writing; the insecure-connection warning is acknowledged.
- [ ] Modbus TCP devices: confirmed reachable only from the edge box; PLC-side write protection configured where available; residual risk recorded (01 section 7).

### B5. Audit and logging (08)
- [ ] Audit forwarding to the customer's syslog/SIEM verified (a test event arrives with the expected fields) or the customer declined in writing.
- [ ] `skyrangerctl audit verify` passes; disk alarms at 80/90% configured; retention set and recorded.
- [ ] Time zone and NTP verified so audit timestamps are meaningful.

### B6. Backup and recovery (10)
- [ ] Nightly backup running; at least one copy off the box; destination credentials write-only or object-locked.
- [ ] Recovery kit (age identity, KEK export, root CA export) stored in the customer's safe and with the integrator, separately from backup files; inventory recorded.
- [ ] Restore drill performed on the cold spare or a scratch VM within the last 30 days; date recorded.

### B7. Safety boundary and handover (11)
- [ ] Safety statement in the handover document and HMI About page; customer signed the independence acknowledgement.
- [ ] For each writable tag, the device-side limit that makes a bad write harmless is recorded.
- [ ] No connection to any safety system, or an approved one-way link documented by the site's safety engineer.
- [ ] Customer-facing security summary delivered: residual risks (01 section 7), who to call, how to isolate and preserve evidence (12 section 6), how to report a vulnerability (13), support and EOL dates.
- [ ] Training: operators know how to acknowledge alarms, what "stale" and "offline" mean, and that Sky Ranger alarms are not protection.

## Sign-off record (attach to the commissioning report)

```yaml
release: 0.1.0                 # or site: customer-name / gateway_id: gw01
date: 2026-11-07
part: A | B
checked_by: name
exceptions:
  - item: "B1: DMZ proxy not available"
    reason: "Office VLAN reaches 8443 directly, source restricted to SCADA group"
    expires: 2027-02-01
    accepted_by: customer-name
attachments: [nmap-office.txt, nmap-field.txt, ruleset.txt, testssl.txt, sbom.json, audit-verify.txt]
```

## How the checklist stays honest
- Every incident or vulnerability adds or sharpens an item (13 step 7).
- Items that are never failing for 3 releases are candidates for automation, not deletion.
- The checklist version is recorded in each sign-off.

## Decisions made
1. Two gates: a per-release gate (mostly automated) and a per-site gate (manual plus external scan); both must pass.
2. Exceptions are allowed only as written, dated, accepted exceptions attached to the commissioning report.
3. External verification (port scan, TLS scan, restore drill) is evidence, attached, not asserted.
4. A penetration test on a staged deployment gates the first paid customer.
5. Each item names its defining file so the checklist cannot drift from the design.

## Alternatives rejected and why
- **A generic CIS/NIST checklist**: too broad to run; this one lists what Sky Ranger actually promises.
- **Only automated gates**: cannot check cabinets, kits, firewalls and customer acknowledgements.
- **Only manual gates**: regressions in authorization or CSP appear between releases; automation catches them.
- **Self-attestation without evidence files**: not auditable and no use in a dispute or tender.

## Open questions
- Which external vendor or consultant performs the first penetration test, and with what scope and budget (also an open question in 01)?
- Automate Part B items that can be scripted (`skyrangerctl doctor --site-gate` collecting `ss`, service lists, container inspect, TLS scan) so the evidence bundle is one command; deployment area to own the tool.
- Should customers be allowed to run Part B themselves for self-installed sites, and how is the exception process handled then?
- Thresholds such as "fuzz for at least 1 hour" and "restore drill within 30 days" are initial values; adjust after the first two projects.
- IEC 62443 mapping (12) may require extra evidence items once the standard texts are checked.
