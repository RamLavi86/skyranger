# 04 Device Identity and Onboarding (ESP32-class)

Scope: field devices that connect to the gateway's Mosquitto broker as Sparkplug B edge nodes, starting with the `skyranger.pulse_meter.v1` ESP32 (messages.md section 2). PLCs and OPC UA servers are covered by their protocols (03, 05).

Goals:
1. Every device has its own credential. Cloning one device or reading its flash compromises only that device.
2. A device can publish only its own data and receive only its own commands.
3. Onboarding is a five-minute job for an electrician with a laptop or phone, not a PKI exercise.
4. Revoking a device is one click and takes effect within seconds.
5. Works with no internet.

## 1. Identity

| Item | Value |
|---|---|
| Device identity | `edge_node_id` (segment grammar). Default `esp32-` + lower 6 hex digits of MAC (contract). Provisioning may set another id |
| MQTT username | equal to `edge_node_id` |
| MQTT client id | equal to `edge_node_id` (Mosquitto disconnects the older session on duplicate client id, which makes clones visible as flapping) |
| Credential | v1: random 32-byte password, base64url (43 chars), unique per device. v2 option: X.509 client certificate from the site CA |
| Group | `group_id` default `skyranger` (contract) |
| Physical label | QR code on the device enclosure with `edge_node_id` and a 6-digit **pairing code** (section 3). The label is how the electrician proves "this box is the one in the pit" |

### Certificate vs password: trade-off

| Criterion | Per-device password over TLS (v1 default) | Per-device client certificate (mTLS, v2) |
|---|---|---|
| Secret on device | 32 random bytes in encrypted NVS | private key in encrypted NVS or ESP32 Digital Signature peripheral (key never readable, on chips that have it; UNVERIFIED per chip variant) |
| Phishing / server impersonation | Prevented by CA pinning on the device (device verifies the broker cert against the site CA) | Same |
| Secret crosses the wire | Yes, inside TLS on each connect | No, only a signature |
| Broker-side storage | Salted hash in Mosquitto dynamic security (PBKDF2-SHA512 as remembered, UNVERIFIED) | Nothing secret; CN-to-ACL mapping and CRL |
| Revocation | Delete the client: immediate, and Mosquitto disconnects it (UNVERIFIED that dynsec kicks active sessions; if not, `skyrangerd` forces it via a disconnect) | CRL update and broker reload; active session must be kicked |
| Expiry handling | None needed | Certificates expire; a device in a meter pit for 10 years needs renewal over MQTT or a very long lifetime |
| ESP32 resource cost | TLS server-auth only; ~40 to 50 KB heap during handshake (order of magnitude, UNVERIFIED) | Slightly more: client cert and key operation |
| Tooling complexity | Low | Medium: CSR generation on device, signing flow, renewal |
| Fit for 62443 "unique identification and authentication" of devices | Satisfies uniqueness; weaker than PKI-based auth (62443-4-2 has a requirement for PKI-based component auth at higher SLs, UNVERIFIED IDs) | Stronger fit |

**Decision:** v1 ships per-device passwords over TLS with CA pinning. It delivers the main properties (unique, revocable, never shared, encrypted in transit) with the least that can go wrong in the field. The provisioning protocol below is designed so that switching to CSR-based certificates is a change of one message, not a redesign. mTLS becomes the default once renewal over MQTT is designed and tested on real devices.

## 2. What is stored on the device

ESP32 NVS namespace `sr_prov` (in an encrypted NVS partition):

| Key | Content |
|---|---|
| `edge_node_id` | string |
| `group_id` | string |
| `broker_uri` | `mqtts://10.20.1.10:8883` (IP preferred; mDNS optional) |
| `broker_ca` | PEM of the **site issuing CA** chain (pinned trust anchor; the device trusts nothing else) |
| `mqtt_pass` | 32-byte credential (v1) |
| `wifi_ssid`, `wifi_psk` | field Wi-Fi |
| `ntp_server` | gateway IP |
| `prov_version` | integer |

Production devices MUST have, enabled by the firmware build and the factory script (ESP-IDF features; exact eFuse burn steps per chip are UNVERIFIED and must be checked in the ESP-IDF docs for the target chip):
- **Secure Boot v2**: bootloader and app signed; the chip refuses unsigned images.
- **Flash encryption (release mode)**: flash contents, including NVS, unreadable when the chip is desoldered or read by JTAG.
- **NVS encryption**: keys for the encrypted NVS partition.
- **JTAG disabled** and **UART download mode disabled or secure-download mode** after provisioning.

Development devices may run without these; they are labelled `DEV` and the gateway shows `fw_secure: false` (proposed NBIRTH metric, additive) in discovery so an engineer sees it before approving.

## 3. Provisioning flow (v1)

Principle: the device never needs internet; the credential is generated on the gateway, delivered over a short-lived local channel, and bound to a human approval.

Two delivery options; both end in the same state.

### Option A: USB serial provisioning (recommended, simplest, most secure)
Done on the bench before installation.

```
Integrator laptop (skyrangerctl)        Gateway (REST, engineer role)          ESP32 (USB, provisioning mode)
1. skyrangerctl device provision --port /dev/ttyUSB0 --gateway https://gw01:8443
   ─ reads device MAC, chip id, fw_version, fw_secure over serial  ◄──────────────────────── (prov firmware prompt)
2. POST /api/v1/devices  {edge_node_id, group_id, mac, chip_id, fw_version, fw_secure}
                                      ─► creates device record status=provisioned
                                      ─► generates mqtt_pass (32 random bytes)
                                      ─► creates Mosquitto client + role (section 4), audits
                                      ◄─ returns {mqtt_pass, broker_uri, broker_ca, ntp_server} ONCE
3. writes sr_prov NVS values over serial ─────────────────────────────────────────────────► stores, reboots
4. device connects mqtts://..:8883, verifies broker cert against broker_ca, CONNECT user=edge_node_id
5. NBIRTH ─► sparkplug driver ─► discovery (messages.md section 7): status pending, linked to device record
6. Engineer approves in HMI (template pulse_water_meter); tags exist only after revision applied (decision 4)
```

The credential is in memory on the laptop only during the serial write; `skyrangerctl` never writes it to disk.

### Option B: On-site SoftAP pairing (for devices already installed)
1. Device without valid `sr_prov` (or after a long-press factory reset) starts a Wi-Fi access point `SR-{edge_node_id}` (WPA2, PSK printed on the device label). The AP lives for at most 15 minutes. It is only a transport: a label PSK is not strong enough to protect the credential, so the credential itself is delivered over TLS in step 3 and never travels over the AP.
2. Technician's phone/laptop joins the AP and opens `http://192.168.4.1/` served by the device (plain HTTP on a single-hop, short-lived network; acceptable because the payload is protected in step 3).
3. The technician enters the gateway URL and an **enrollment token** obtained from the HMI (`engineer` creates it under Devices, "Enroll new device": 10 minutes validity, single use, bound to the `edge_node_id` read from the device label). The device then connects to the field Wi-Fi and calls the gateway's enrollment endpoint **over TLS**, verifying the gateway against the site CA fingerprint that is included in the enrollment token, and redeems the token for its `mqtt_pass`.
4. Steps 4 to 6 as in option A.

Enrollment token format: `sre1_{edge_node_id}_{base64url(16 random bytes)}_{first 8 bytes of site CA SHA-256, hex}`. Stored server-side as SHA-256. The pairing code on the label is checked against the device-reported code so a token typed into the wrong device fails.

Proposed endpoints (additive, need api.md entries): `POST /api/v1/devices` (engineer), `GET /api/v1/devices`, `POST /api/v1/devices/{edge_node_id}/enrollment-tokens` (engineer), `POST /api/v1/devices/enroll` (unauthenticated, token in body, rate limited to 5 per minute per IP), `POST /api/v1/devices/{edge_node_id}/rotate`, `POST /api/v1/devices/{edge_node_id}/revoke`.

Why the device record exists separately from discovery: discovery (messages.md section 7) is about *tags from data*; the device record is about *who is allowed to connect*. A device can be provisioned but not yet approved for tags (normal), or approved and later revoked (data stops, tags go `uncertain_last_known` then the `health` alarm fires).

### v2: certificate variant
Step 2 changes: the device generates an ECDSA P-256 key on-chip, sends a CSR over serial (A) or within the enrollment call (B); the gateway signs a client cert with CN = `edge_node_id`, lifetime `5y`; Mosquitto field listener switches to `require_certificate true` and `use_identity_as_username true`. Renewal: the gateway sends a Sparkplug `NCMD` metric `Device Control/Renew Certificate` (proposed, additive to the device profile) at 80% of lifetime; the device generates a new key and CSR and calls the enrollment endpoint with its current cert as client auth.

## 4. Broker ACLs (normative)

`skyrangerd` is the only writer of Mosquitto's dynamic security configuration (via its MQTT control API on `$CONTROL/dynamic-security/#`, which only the `skyrangerd-admin` principal can reach; UNVERIFIED topic name for the pinned Mosquitto version). Manual edits are overwritten.

Roles:

| Principal (username / cert CN) | Publish allowed | Subscribe allowed |
|---|---|---|
| device `{edge_node_id}` | `spBv1.0/{group_id}/NBIRTH/{edge_node_id}`, `spBv1.0/{group_id}/NDATA/{edge_node_id}`, `spBv1.0/{group_id}/NDEATH/{edge_node_id}` (Will), and with child devices `spBv1.0/{group_id}/DBIRTH/{edge_node_id}/+`, `DDATA/{edge_node_id}/+`, `DDEATH/{edge_node_id}/+` | `spBv1.0/{group_id}/NCMD/{edge_node_id}`, `spBv1.0/{group_id}/DCMD/{edge_node_id}/+`, `spBv1.0/STATE/{host_id}` |
| sparkplug driver (`skyranger-spb-{connection}`) | `spBv1.0/{group_id}/NCMD/+`, `spBv1.0/{group_id}/DCMD/+/+`, `spBv1.0/STATE/{host_id}` | `spBv1.0/{group_id}/#` for each configured `group_ids` entry |
| `skyrangerd` (internal bus publisher) | `skyranger/v1/{gateway_id}/#` | none needed |
| internal subscribers (`svc-python-runtime`, integrations; listener 8884) | none | `skyranger/v1/{gateway_id}/#` (or narrower prefixes) |
| `skyrangerd-admin` | `$CONTROL/dynamic-security/#` | `$CONTROL/dynamic-security/#` |
| anonymous | denied (`allow_anonymous false`) | denied |

Default ACL: deny publish, deny subscribe. Retained messages: devices may not set retain on any topic (Sparkplug does not use retain for node messages, UNVERIFIED for all message types except STATE which is retained). The broker cannot express "no retain" per role in all versions; if not enforceable, the Sparkplug driver ignores retained N/D messages.

Sparkplug topic wildcard caveat: Sparkplug topic `spBv1.0/{group_id}/{message_type}/{edge_node_id}` puts `message_type` before `edge_node_id`, so the device ACL must enumerate message types rather than use one `.../{edge_node_id}/#` pattern. This is why the table lists them.

Messages from an authenticated device that are well-formed but claim a different `edge_node_id` cannot occur, because the ACL refuses the publish. Messages that fail Sparkplug decoding are dropped by the driver and counted in `_system/connections/{connection}/...` stats.

## 5. Revocation, rotation, and decommissioning

| Action | Who | Effect | Latency |
|---|---|---|---|
| Revoke | `engineer` (`POST .../revoke`, comment required) | Mosquitto client disabled and deleted; active session kicked; device record `status=revoked`; audit `device.revoked`; tags go `uncertain_last_known`, then `health` alarm (contract behaviour) | seconds |
| Rotate credential | `engineer` | New `mqtt_pass` issued; delivered by USB or by an NCMD-triggered re-enrollment (v2); old credential valid for a grace period of `0s` by default (configurable up to `24h` for remote devices) | |
| Lost / stolen device | `engineer` | Revoke. Tags remain with history. A replacement device gets a new `edge_node_id` or re-uses the old one with a new credential (`counter_total` treats the new device's first sample per the reset rules; use `manual_adjust` to align) | |
| Decommission | `engineer` | Revoke, then remove the template instance in config (new revision). The device should be factory reset (`sr_prov` erased); with flash encryption this also makes the key unrecoverable | |
| Site CA rotation | `admin` | Devices pin the issuing CA. Rotation requires pushing the new CA chain to every device *before* the broker cert changes; plan: dual-chain period where the broker cert is issued by the new CA and devices hold both CAs (`broker_ca` is a bundle). Must be tested before the first rotation is needed (issuing CA lifetime 5 y) | |

Clone detection: if two devices share a credential, Mosquitto's client-id takeover makes them disconnect each other repeatedly. The Sparkplug driver counts `NBIRTH`s per node; more than 10 births per node per hour raises a `_system` security event `skyranger.security.device_flapping.v1` (proposed additive event type).

## 6. Firmware integrity and OTA

- Firmware images are signed with the Secure Boot v2 signing key held offline by the release process (07). The device refuses images not signed by that key.
- OTA delivery: the gateway serves images over HTTPS from `https://{gw}:8443/fw/{sha256}.bin`, and the device verifies (a) the TLS server against `broker_ca`, (b) the SHA-256 given in the OTA command, (c) the Secure Boot signature during boot. OTA is triggered by an `engineer` through the API (audited), never automatically.
- Rollback: ESP-IDF app rollback (two OTA slots, mark-valid after successful MQTT connect and first NBIRTH) and anti-rollback (secure version eFuse) for security fixes. Anti-rollback eFuses are finite; bump the secure version only for security releases.
- `fw_version` in NBIRTH is informational and not trusted for security decisions; the gateway records the SHA-256 it shipped.

## 7. What the gateway does NOT trust from devices

- `ts_source` (skew rule, tag-model.md section 5).
- Count plausibility (`max_delta`).
- `profile` and `fw_version` (they only suggest a template; an engineer approves).
- Any device-supplied string shown in the HMI (escaped by React; length-limited by the driver to 4096 bytes per the string datatype).

## Decisions made
1. Device identity is the `edge_node_id`, used as MQTT username and client id.
2. v1 credential: unique random 32-byte password over TLS, device pins the site issuing CA. mTLS with on-device keys is the v2 default once renewal is proven.
3. Credentials are generated by the gateway and delivered over USB (preferred) or a token-gated enrollment call; never typed by hand, never shared.
4. `skyrangerd` owns Mosquitto users and ACLs via dynamic security; per-device ACL restricted to its own Sparkplug topics; default deny.
5. Production ESP32s require Secure Boot v2, flash encryption, encrypted NVS, JTAG off.
6. Revocation is an audited engineer action that takes effect within seconds.
7. Device provisioning (who may connect) is separate from discovery approval (what becomes tags).

## Alternatives rejected and why
- **One shared MQTT password for all devices**: one stolen meter compromises the fleet and revocation is impossible without touching every device.
- **Anonymous MQTT on an isolated SSID**: Wi-Fi is broadcast; any device that joins could publish NBIRTH as any node.
- **Wi-Fi PSK as the only device authentication**: shared secret, not per device; no binding to `edge_node_id`.
- **ESP-IDF Wi-Fi provisioning over BLE with a cloud service**: adds an internet dependency.
- **Mosquitto `password_file` + static `acl_file`**: requires a broker reload/restart for each device change and is edited outside `skyrangerd`; dynamic security gives an API and immediate effect.
- **Trust-on-first-use of devices**: an attacker connecting first wins.

## Open questions
- Contract additions needed: a device registry resource (`/api/v1/devices...`), event types `skyranger.device.provisioned.v1` / `revoked.v1` / `skyranger.security.device_flapping.v1`, optional NBIRTH metric `fw_secure` in a `skyranger.pulse_meter.v1` minor revision (additive metric; existing consumers ignore unknown metrics).
- Does Mosquitto dynamic security disconnect an active client when it is deleted or disabled? UNVERIFIED; if not, use the admin disconnect mechanism or a broker plugin.
- ESP32 variant choice (ESP32, S3, C3, C6) determines Digital Signature peripheral and secure boot details: UNVERIFIED per variant.
- Should approval of a discovered device be blocked (not just warned) if `fw_secure: false` on customer sites?
- Long-lived devices and certificate renewal policy for v2.
