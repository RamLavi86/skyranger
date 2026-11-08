# 09 Plugins (platform tier, v2)

Status: design outline for v2, with enough detail that v1 choices do not block it. Conforms to [../00-contracts/](../00-contracts/README.md), in particular [driver-interface.md](../00-contracts/driver-interface.md) (the driver protocol is the first plugin protocol and already exists). New contract proposals P8 and P9 are listed in [README.md](README.md). Security alignment: [../07-security/07-secure-update.md](../07-security/07-secure-update.md) (cosign, offline verification).

## 1. Goals and non-goals

Goal: other SCADA engineers can add capabilities without forking Sky Ranger and without being able to destabilize the core.

Rules that follow from decisions already made:
1. **Out of process, always.** The driver contract rejected in-process Go plugins (driver-interface.md "Alternatives rejected"). The same logic holds for every extension point. A plugin crash must not take down `skyrangerd`.
2. **Narrow extension points**, each with a versioned protocol (`skyranger.<point>.v1`) and its own conformance tests.
3. **Declared permissions, granted by an admin at install**, enforced by the host and the container, never by trusting the plugin.
4. **Signed, verified offline** (edge-first: no internet at the plant).
5. **The core never depends on a plugin to run** (a gateway with no plugins is complete).

Non-goals in v2: a marketplace or registry, paid-license enforcement, plugins that modify the core's behavior in-process, plugins that add tag datatypes, quality codes, or API endpoints in `/api/v1`.

## 2. Extension points

| Extension point | What it adds | Protocol | Runs as | Status |
|---|---|---|---|---|
| **Driver** | A new southbound protocol (`Connection.spec.driver: acme-dnp3`) | `skyranger.driver.v1` (existing contract, unchanged) | child process in the `skyrangerd` image **or** its own container with the socket dir bind-mounted (driver-interface.md section 1) | v2, first one to ship |
| **Notification channel** | Delivery of alarm and system notifications (SMS gateway, Teams, PagerDuty, SMTP) | `skyranger.notify.v1` | container | v2 |
| **Script library** | Pure-Python modules importable as `libs.<plugin>.<module>` | none (files) | loaded by the scripts pool, read-only mount | v2 |
| **Service** | Long-running background logic (state machines, protocol bridges, report generators) that uses the host API like a script but without run-to-completion limits | `skyranger.service.v1` | container | v2 (open question) |
| **UI component** | Custom HMI widgets | `skyranger.ui.v1` | JS in a sandboxed iframe in the browser | v3 |
| **Northbound integration** | e.g. OPC UA server, cloud sync targets | via `skyranger.service.v1` | container | v3 |

Not extension points: the tag engine, alarm state machine, historian, auth. Those are the product.

### 2.1 Drivers

No new protocol. A third-party driver implements `driver.Driver` via `skyranger/pkg/driver/sdk` (driver-interface.md section 2) and passes `skyranger/pkg/driver/conformance` (section 8). Packaging adds only a manifest and signature. Naming:
- Driver `name` follows the segment grammar (README). Built-in names (`sparkplug`, `modbus`, `opcua`, `sim`) are reserved. Third-party names must be prefixed with the publisher segment: `acme-dnp3`. The binary is `skyranger-driver-acme-dnp3` (existing rule `skyranger-driver-{name}`).
- Driver config schema and address syntax come from `Info` (GET `/drivers` already lists them), so the HMI and validator need no plugin-specific code.

### 2.2 Notification channels (`skyranger.notify.v1`)

```protobuf
syntax = "proto3";
package skyranger.notify.v1;
import "google/protobuf/timestamp.proto";

service Channel {
  rpc GetInfo(GetInfoRequest) returns (Info);                  // name, version, config_schema (JSON Schema), capabilities
  rpc Configure(ConfigureRequest) returns (ConfigureResponse); // settings JSON; secrets as secret:// refs resolved by host into `secrets` map
  rpc Send(SendRequest) returns (SendResponse);                // one notification to one recipient
  rpc Health(HealthRequest) returns (HealthResponse);
}

message SendRequest {
  string notification_id = 1;        // UUIDv7, idempotency key
  string recipient = 2;              // channel-specific address (phone, webhook name, e-mail)
  string severity = 3;               // low|medium|high|critical (tag-model.md section 8)
  string title = 4;
  string body = 5;
  google.protobuf.Timestamp time = 6;
  bytes event_json = 7;              // the CloudEvent that caused this (messages.md section 4)
}
message SendResponse { bool delivered = 1; bool retryable = 2; string error = 3; }
```

Routing (which alarm goes to whom, escalation, quiet hours) belongs to the alarm area, not to the plugin; the plugin only delivers. A `NotificationChannel` config kind or an alarm-area structure references channels by name (P8, open question 4). Delivery is **at-least-once with idempotency key**; channels de-duplicate on `notification_id`.

### 2.3 Script libraries

A plugin ships `libs/` Python files. They are mounted read-only into the scripts container at `/opt/skyranger/plugin-libs/{publisher}/{plugin}/` and import as `libs.{plugin}`. Rules: pure Python, allowed stdlib imports only (04), no native extensions in v2, run with the caller's permissions, hashes recorded in run records. A script library plugin cannot add permissions; it makes code reusable, not privileged. (Vendoring third-party pure-Python packages inside a plugin is allowed and covered by its signature.)

### 2.4 Services (`skyranger.service.v1`)

A service plugin is a container that connects to the host through a gRPC `HostAPI` with the **same method set and permission model as scripts** (02, 04): `tags.read/write`, `history.read`, `events.emit`, subscriptions as a server stream, with the plugin's own principal `plugin:{publisher}/{name}` and manifest-declared permissions. It is what Ignition engineers might call a module: persistent logic that is not request/response. It is deliberately **not** in v1 or the first v2 milestone (open question 2): it is the point most likely to turn into an unbounded API.

### 2.5 UI components (`skyranger.ui.v1`, v3)

The HMI contract says screens are data and **no arbitrary JavaScript** (05-hmi-api/02-screen-format.md). A UI plugin is the one place where code runs in the browser, so it gets the strictest isolation:
- Rendered inside `<iframe sandbox="allow-scripts">` (no `allow-same-origin`, so no access to the HMI's token, storage, or DOM) with its own CSP; served from a distinct origin path by the gateway.
- Talks to the host page only through `postMessage` with a small versioned protocol: `init {props, theme, locale}`, `value {bindings}`, `resize`, `emit {action}` (one of a **declared** list). The host page performs tag subscription and permission checks; the iframe never sees a bearer token or a WebSocket.
- Bindings are declared in the manifest (names, datatypes), so the screen resolver can still compute the complete path list of a screen before rendering.
- No writes from components in v3.0 except through the existing operator confirmation flow (05-hmi-api/05-roles-and-control.md): a component can only request a named, declared action that the screen definition wires.

## 3. Plugin package

### 3.1 Manifest (`plugin.yaml`)

Same envelope as all Sky Ranger documents. Not part of tag config; stored beside it in the install state (P8).

```yaml
apiVersion: skyranger/v1
kind: Plugin
metadata:
  name: dnp3                       # segment grammar
  description: "DNP3 outstation master driver"
spec:
  publisher: acme                  # segment grammar; must match the signing key scope (section 4)
  version: 1.4.2                   # semver
  license: Apache-2.0              # SPDX identifier
  homepage: https://example.com/acme/skyranger-dnp3
  requires:
    skyranger: ">=1.2.0 <2.0.0"    # gateway version range (semver, 10)
    protocols: { driver: v1 }      # extension protocols used; majors only
  provides:
    - type: driver
      name: acme-dnp3
      protocol: skyranger.driver.v1
      entrypoint: bin/skyranger-driver-acme-dnp3        # or image: for container form
      hosting: process             # process (inside skyrangerd image volume) | container
  runtime:
    image: registry.example/acme/skyranger-dnp3@sha256:3f1c...   # container form: digest-pinned only
    resources: { memory: 128MiB, cpus: 0.5, pids: 64 }
    user: 20100                    # assigned by the host if absent
  permissions:                     # what the admin is asked to grant at install
    network:
      egress: [ { cidr: 10.20.0.0/24, ports: [20000] } ]    # plant LAN subnets the driver may reach
    devices: []                    # e.g. ["/dev/ttyUSB0"] for serial drivers
    host_api: {}                   # driver: none; service/notify: tag/history/event grants
  files:                           # integrity list for the bundle
    - { path: bin/skyranger-driver-acme-dnp3, sha256: "9f2c..." }
    - { path: LICENSE, sha256: "..." }
```

Driver plugins need no `host_api` (a driver cannot read tags; the host feeds it). Notification channels need none either. Services and script libraries state theirs. The manifest declares **maximum** permissions; the install records what the admin actually granted (subset).

### 3.2 Bundle format `.srp` (Sky Ranger plugin)

Same structure idea as the update bundle `.srb` (07-secure-update.md): a tar containing the manifest, an OCI image layout (container form) or the files tree (process/library/UI form), and signatures.

```
dnp3-1.4.2.srp (tar)
  plugin.yaml
  plugin.yaml.sig            # Ed25519 over the canonical bytes of plugin.yaml (which lists every file hash)
  oci/ ...                   # container form: OCI layout, image referenced by digest
  files/bin/skyranger-driver-acme-dnp3
  files/LICENSE
  SBOM.cdx.json              # CycloneDX, required for the signed publisher tier (section 4)
```

Because `plugin.yaml` lists the SHA-256 of every file (and the image digest), one signature over the manifest covers everything. Verification is `crypto/ed25519` from the Go standard library: no extra dependency.

### 3.3 Install, enable, upgrade

```
POST /api/v1/plugins            (admin)  multipart: the .srp           -> install record, status "pending_approval"
GET  /api/v1/plugins                      installed plugins, versions, status, granted permissions
POST /api/v1/plugins/{publisher}/{name}/approve   {"grant": {...}, "comment": ""}   -> "installed"
POST /api/v1/plugins/{publisher}/{name}/enable|disable
DELETE /api/v1/plugins/{publisher}/{name}
```

Flow: upload, verify signature against the trust store (section 4), verify file hashes, check `requires` against the gateway version and protocol majors, **show the manifest permissions and publisher to the admin**, admin approves a grant, host launches (process, or container with exactly the granted network/device/resource settings through the deployment area's container runtime). Everything is audited (`plugin.install`, `plugin.approve`, ...). Uninstall removes files and grants but keeps config documents that reference the plugin's driver; those connections become `faulted` with `bad_config_error` until the driver is back (consistent with driver-interface.md states).

Upgrade: install a higher version; the previous version stays until the new one passes `Health`; rollback is "re-enable previous". Downgrade below a recorded minimum is refused unless forced (rollback protection, section 4).

Plugin instances are enabled through config where config exists: a driver plugin is used by creating a `Connection` with its `driver:`; no new kind. (`Plugin` enablement and grants are install state, not tag configuration; see P8 and open question 1 on whether they should also appear in the config export for lossless restore.)

## 4. Signing and trust

Decision: **Ed25519 signatures over the manifest, verified against an admin-managed offline trust store.** Aligns with the security area's offline-first principle; differs from its container-image mechanism (cosign, key-based) deliberately and visibly (open question 3).

Trust store (files, root-owned, read-only to the container):

```
/etc/skyranger/trust/plugins/
  skyranger-release.pub               # first-party key (also signs official plugins)
  acme-2026.pub                       # a vendor key an admin imported
  trust.yaml
```

```yaml
# trust.yaml
apiVersion: skyranger/v1
kind: PluginTrust
keys:
  - key_id: acme-2026                 # segment grammar
    file: acme-2026.pub
    publisher: acme                   # this key may sign only plugins whose spec.publisher == acme
    not_after: 2028-01-01T00:00:00Z
    added_by: admin
    note: "Acme Controls, imported 2026-11-08 after phone verification"
revoked:
  - key_id: acme-2025
    reason: "key compromise"
policy:
  allow_unsigned: false               # true = loud override (below)
```

Rules:
- **Scope binding.** A key is bound to a publisher segment; a valid signature from `acme` cannot install a plugin claiming publisher `skyranger`. The `skyranger` publisher is reserved for the project's key.
- **Unsigned plugins are refused** unless `allow_unsigned: true`. When set: a permanent HMI banner, an event `skyranger.plugin.unsigned_allowed.v1` (naming TBD with P3-style additions), an audit record, and a `_system/plugins/unsigned_allowed` bool tag so alarms can watch it. Intended for development gateways and a plugin author's own box.
- **Key distribution is out of band.** The plant admin imports a publisher key after verifying it with the publisher (phone, in person). There is no TOFU and no online key server in v2.
- **Revocation** is the `revoked` list inside `trust.yaml`, updated with the signed offline update bundles (07-secure-update.md). Revoked key: plugin refused on next start and flagged immediately.
- **Rollback protection:** the host records the highest installed version per plugin; installing a lower version needs admin force.
- **Expiry:** `not_after` on keys. An expired key does not invalidate already-installed plugins (offline plants have no safe way to renew), but blocks new installs, and the HMI shows the expiry.
- **What signing does not give:** it proves who published and that the bytes are unmodified, not that the code is safe. That is why the permission grant and the out-of-process boundary exist.
- **Container form:** image referenced by digest in the manifest; the manifest signature covers the digest. The host does not pull by tag.
- **Key custody guidance for publishers:** hardware token if possible; never in CI without protection (same guidance as the release key in 07-secure-update.md).

Signed-publisher tier extras (not v2.0): SBOM required, reproducible-build attestation, vulnerability scan, and a project "verified" listing. Left to v3 and the community process.

## 5. Isolation and enforcement for plugins

| Surface | Mechanism |
|---|---|
| Process | Own container or child process under a distinct UID; `cap_drop ALL`, `no-new-privileges`, read-only rootfs, writable data dir only `/var/lib/skyranger/plugins/{publisher}/{name}` |
| Network | Driver and service containers get only the egress their grant lists, implemented by the deployment area (network namespace plus nftables per container; 05-network-segmentation.md). No access to the internal MQTT broker or Postgres unless a grant says so (none do in v2) |
| Resources | Manifest `resources`, capped by an admin-set ceiling |
| Host API | Per-plugin identity; same permission and budget machinery as scripts (04 section 2). `plugin:{publisher}/{name}` principal in audit |
| Secrets | Same rule as scripts: host resolves `secret://` and hands the plugin only what its `settings` reference, over the local socket |
| Host crash handling | Same supervision as drivers: backoff restarts, `Health` checks, state in `_system/plugins/{publisher}/{name}/...` |
| Logging | Plugin stdout/stderr captured as JSON lines, tagged with plugin id, same as drivers |

## 6. What v1 must do now so v2 is possible

1. Keep driver loading generic: the driver host launches `skyranger-driver-{name}` by name from a directory, with built-ins from the image and room for plugin-provided ones.
2. Use `principal` strings of the form `script:...` now; reserve `plugin:...` (P4).
3. Keep permission checks centralized in the host (04) so services can reuse them.
4. Keep the config export lossless for unknown kinds (`Plugin` later): additive kinds already tolerated ("consumers MUST ignore unknown fields").
5. Reserve the `libs.<plugin>` import namespace (done: `libs.*`).

## Decisions made
1. Plugins are always out of process, behind versioned gRPC protocols named `skyranger.<point>.v1`; there is no in-process plugin API. The driver protocol is the first and unchanged.
2. Five extension points (drivers, notification channels, script libraries, services, UI components) shipping in that order; UI components in a sandboxed iframe with a postMessage protocol (v3).
3. Packages are `.srp` tarballs: YAML manifest listing every file hash and the image digest, Ed25519 signature over the manifest, OCI image layout for container form, SBOM for the signed-publisher tier.
4. Trust is an admin-managed offline trust store: keys bound to publisher names, no TOFU, no online lookups; revocation via signed offline updates; unsigned only through an explicit loud override.
5. Install requires an admin approving the declared permissions; the grant is a subset of the manifest maximum and is audited.
6. Third-party driver names carry the publisher prefix; built-in names are reserved.
7. Notification routing stays in the alarm area; the plugin only delivers, idempotently.
8. Services and UI components are later than drivers, channels and libraries because they widen the host API.

## Alternatives rejected and why
- **Go `plugin` package or shared libraries in-process**: fragile (exact toolchain match), Linux-only, no crash isolation (already rejected in driver-interface.md).
- **HashiCorp go-plugin as the universal mechanism**: proven, but plain gRPC over a Unix socket is already the project's pattern and is language-neutral (driver-interface.md). Revisit only if its features are needed.
- **WASM plugins for everything**: attractive isolation and portability but driver-class plugins need sockets and serial ports; ecosystem maturity UNVERIFIED. Possible for pure-logic extension points later.
- **Reusing cosign for plugin signatures**: boring and already adopted for images, but plugin publishers would need cosign tooling and key formats (support for Ed25519 keys in cosign key-based mode is UNVERIFIED), and the trust model here is per-publisher keys rather than a single release key. Alignment is an open question.
- **TOFU or online key lookup (keyless Sigstore)**: needs internet; edge-first plants have none.
- **A central plugin registry in v2**: operational burden and a trust position the project cannot yet staff. Files first.
- **Plugin-defined REST endpoints**: widens the public API contract every client depends on. Plugins integrate through protocols, not by extending `/api/v1`.
- **Letting UI plugins run in the HMI origin**: any plugin could steal the bearer token from `sessionStorage` (api.md section 2). Iframe sandbox with no same-origin is the minimum.

## Open questions
1. Whether installed plugin state and grants appear in the config export (needed for "restore a gateway from export alone") or live as separate install state restored from a bundle. Leaning to include a `Plugin` kind (grants and settings, no code) in config, with code restored from `.srp` files.
2. Is the `service` extension point worth building, or should "long-running logic" stay out of scope forever (use an external process talking REST/WS)? Strong argument for "out of scope": the REST/WS API already supports external clients.
3. Single signing mechanism with the security area: cosign for everything, or Ed25519 manifests for plugins as here? Needs a joint decision with 07-security (07-secure-update.md owns the update path).
4. Notification channels as a config kind (`NotificationChannel`) vs part of the alarm area's routing config: with 04-alarms.
5. Container egress enforcement mechanism per plugin (deployment area), and how to deal with serial-port passthrough for serial drivers (device mapping in the container runtime).
6. Naming of the new events (`skyranger.plugin.*.v1`) and the `plugin-host` component for `source`: additive to messages.md, record with P3-style proposals.
7. Third-party driver testing: the conformance suite ships as a Go package and a container so non-Go drivers can run it over gRPC (the protocol is language-neutral).
8. Plugin data (`/var/lib/...`) backup and the licensing story for commercial plugins (10).
