# 03 TLS Everywhere and Site PKI

Rule (principle 4): every TCP socket whose both ends Sky Ranger controls uses TLS. Plaintext exists only on Unix domain sockets (kernel-local, permission and peer-credential checked) and on field protocols that have no security mode (Modbus TCP), which are confined by network design (05).

## 1. Site PKI

A small plant has no PKI. Sky Ranger brings a minimal one, generated on first boot, so TLS is on from day one without the integrator running OpenSSL by hand.

```
Sky Ranger Site Root CA  (ECDSA P-256, 10 y, pathlen:1)   -- private key offline or TPM-sealed
 └── Sky Ranger Site Issuing CA  (ECDSA P-256, 5 y, pathlen:0)   -- on the box, TPM-sealed when present
      ├── server: gateway HTTPS            (SAN: DNS gw01.plant.local, IP 10.20.2.10)    825 d max, default 397 d
      ├── server: mosquitto                (SAN: DNS mqtt.plant.local, mosquitto, IP 10.20.2.10)
      ├── server: postgres                 (SAN: DNS postgres)  internal only
      ├── client: skyrangerd → postgres, mosquitto (CN skyrangerd)
      ├── client: opcua application instance cert (one per opcua Connection; see section 5)
      ├── client: per-site sync cert (multi-site, CN = gateway_id) -- issued by central CA instead, section 7
      └── client: per-device certs (ESP32, v2 option, 04)
```

Implementation: `skyrangerd` contains a small CA module using Go `crypto/x509` (no external CA software). CLI: `skyrangerctl pki init|issue|renew|revoke|export-root`.

| Decision | Value |
|---|---|
| Algorithm | ECDSA P-256 with SHA-256 for all certs. RSA 2048 only where a peer requires it (some OPC UA policies, section 5) |
| Root key handling | Default: root key generated, used to sign the issuing CA, then exported encrypted to the recovery kit (10) and **deleted from the box**. Option for tiny sites: keep root sealed on the box (documented weaker posture) |
| Leaf lifetime | `397d` for server certs (fits browser limits as we understand them, UNVERIFIED for current CA/B Forum rules; site CAs are private so browsers may not enforce them), `90d` for internal client certs with auto-renew at 2/3 lifetime |
| Revocation | CRL published by the gateway at `https://{gw}:8443/pki/crl.pem` and loaded by Mosquitto and `skyrangerd`; OCSP is not provided (overkill for one site) |
| Customer PKI | Admin can upload a customer-issued server cert and chain for HTTPS (`POST /api/v1/system/tls` additive endpoint) and point all verification at a customer CA bundle |
| Trust distribution | `skyrangerctl pki export-root` produces `skyranger-site-root.crt` and its SHA-256 fingerprint; the integrator installs it on HMI PCs via GPO or manually. The fingerprint is printed in the commissioning report so users can verify it |

The contract says the gateway generates a self-signed certificate if none is provided (api.md section 1). We satisfy that with a site-CA-issued certificate; a truly self-signed leaf is used only if PKI init has not run (should never happen after setup). Both are "not publicly trusted" and need the root installed on clients.

## 2. TLS profile (all Go servers and clients)

```go
tlsCfg := &tls.Config{
    MinVersion: tls.VersionTLS12,
    // TLS 1.3 suites are not configurable in Go and are all acceptable.
    CipherSuites: []uint16{ // TLS 1.2 only: AEAD + ECDHE
        tls.TLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256,
        tls.TLS_ECDHE_ECDSA_WITH_AES_256_GCM_SHA384,
        tls.TLS_ECDHE_ECDSA_WITH_CHACHA20_POLY1305,
        tls.TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256,
        tls.TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384,
        tls.TLS_ECDHE_RSA_WITH_CHACHA20_POLY1305,
    },
    CurvePreferences: []tls.CurveID{tls.X25519, tls.CurveP256},
}
```

TLS 1.2 stays enabled because ESP32 mbedTLS builds and older HMI PCs may not negotiate 1.3. A site can set `tls.min_version: "1.3"` once its clients support it. No TLS 1.0/1.1, no CBC, no RSA key exchange, no compression.

## 3. HTTPS (REST, WebSocket, HMI)

- One port `8443` (contract). Port `80` is not opened. Optionally port `443` via a reverse proxy in the DMZ (05).
- Response headers (all responses):

```
Strict-Transport-Security: max-age=31536000
Content-Security-Policy: default-src 'self'; script-src 'self'; style-src 'self'; img-src 'self' data:; connect-src 'self' wss:; frame-ancestors 'none'; base-uri 'none'; form-action 'self'
X-Content-Type-Options: nosniff
Referrer-Policy: no-referrer
Cross-Origin-Opener-Policy: same-origin
Cache-Control: no-store            (API responses only; static HMI assets are hashed and cacheable)
```

- `includeSubDomains` is omitted from HSTS by default because plant `.local` domains often host other devices on plain HTTP; the customer can enable it.
- `connect-src 'self' wss:` is tightened to the gateway origin in the build (`wss://{host}:8443`) when the hostname is known.
- Certificate hot reload: `tls.Config.GetCertificate` reads the current cert from the PKI store, so renewal needs no restart.

## 4. MQTT (Mosquitto)

Listeners:

```conf
# /etc/mosquitto/mosquitto.conf (in the mosquitto container)
per_listener_settings false
allow_anonymous false

# Field listener: devices and the sparkplug driver. TLS only.
listener 8883 0.0.0.0
protocol mqtt
cafile    /mosquitto/pki/site-ca-chain.pem
certfile  /mosquitto/pki/mqtt.crt
keyfile   /mosquitto/pki/mqtt.key
tls_version tlsv1.2                  # minimum; 1.3 negotiated when the client supports it
crlfile   /mosquitto/pki/crl.pem
require_certificate false            # v1: username/password over TLS (04). v2 sites set true for mTLS
use_identity_as_username false

# Internal bus listener for local subscribers (Python runtime, integrations)
# on the internal container network only.
listener 8884 172.30.0.2
cafile    /mosquitto/pki/site-ca-chain.pem
certfile  /mosquitto/pki/mqtt.crt
keyfile   /mosquitto/pki/mqtt.key
require_certificate true
use_identity_as_username true        # CN becomes the username for ACLs

# Users and ACLs managed by skyrangerd through the dynamic security plugin (04)
plugin /usr/lib/mosquitto_dynamic_security.so
plugin_opt_config_file /mosquitto/data/dynamic-security.json

# No plaintext 1883 listener at all. No websockets listener.
```

The `plugin` and `plugin_opt_config_file` directive names follow Mosquitto 2.x documentation as remembered (UNVERIFIED; check against the Mosquitto version pinned in the image). `require_certificate`, `use_identity_as_username`, `cafile`, `certfile`, `keyfile`, `tls_version`, `crlfile` are long-standing Mosquitto options; still confirm against the pinned version.

Contract alignment: tag-model.md shows `broker_url: mqtt://mosquitto:1883` in the `Connection` example. Under this design the Sparkplug driver connects with `mqtts://mosquitto:8883` and the site CA. The example in the contract is illustrative; we propose updating it (Open questions). No contract field changes.

## 5. OPC UA

Sky Ranger is an OPC UA **client** in v1 (opcua driver). OPC UA has its own security layer (Secure Channel with an application instance certificate, plus user identity tokens) rather than TLS.

Facts below are from memory and must be checked against OPC UA Part 2, 4, 6 and 7 before the driver is finalized (UNVERIFIED in detail):

| Security policy URI suffix | Our stance |
|---|---|
| `None` | Refused by default. Allowed only with an explicit `allow_insecure: true` and a reason string in the Connection, which raises a persistent warning in the HMI and in the release checklist |
| `Basic128Rsa15` | Refused (deprecated, SHA-1) |
| `Basic256` | Refused (deprecated, SHA-1) |
| `Basic256Sha256` | Allowed (widest support in installed base) |
| `Aes128_Sha256_RsaOaep` | Allowed, preferred over Basic256Sha256 |
| `Aes256_Sha256_RsaPss` | Allowed, most preferred |
| ECC policies (`ECC_nistP256` etc.) | Use when both sides support it; library support uncertain (UNVERIFIED) |

Message security mode: `SignAndEncrypt` required; `Sign` allowed only with a configured reason. User identity: username/password token (password from `secret://`) or X.509 user token; anonymous only with an explicit flag.

Connection settings (opcua driver JSON Schema owns the exact names; proposed):

```yaml
apiVersion: skyranger/v1
kind: Connection
metadata: { name: plc-line1 }
spec:
  driver: opcua
  settings:
    endpoint_url: opc.tcp://10.20.1.20:4840
    security_policy: Aes256_Sha256_RsaPss     # or "auto": strongest offered by server from the allowed list
    security_mode: sign_and_encrypt
    client_certificate: secret://opcua/plc-line1/client   # cert + key, issued by site CA, RSA 2048 (policy requirement)
    server_certificate_pin: "sha256:7f3a..."              # trust-on-first-use, then pinned; or trust via server's CA
    user: secret://opcua/plc-line1/user
```

Server trust: on first connect the driver reports the server certificate thumbprint through discovery/health; an `engineer` approves it in the HMI (pin), recorded in config and audited. Subsequent mismatch: connection `faulted` with `unauthorized` (driver-interface.md section 3/4), all items `bad_config_error`, no auto-retry until config changes. This mirrors how commercial OPC UA clients use a "trusted/rejected" certificate store, but kept in our config so it is exportable.

Application instance certificate: RSA 2048 issued by the site CA because the RSA-based policies require RSA keys. SAN must include the application URI (`urn:skyranger:{gateway_id}:{connection}`) (UNVERIFIED: exact URI requirements and whether the SAN URI must match the ApplicationDescription).

## 6. gRPC on the driver boundary

Drivers talk gRPC over a Unix domain socket at `/run/skyranger/drivers/{connection}.sock` (driver-interface.md section 1). TLS over UDS adds cost and key management without protecting against anything the kernel boundary does not already handle. Instead:

| Control | Implementation |
|---|---|
| Socket directory | `/run/skyranger/drivers` owned by `skyranger-host:skyranger-drv`, mode `0750` |
| Socket file | Created by the driver with umask `0117` (mode `0660`), group `skyranger-drv` |
| Peer verification (driver side) | On accept, driver reads `SO_PEERCRED` and rejects any peer UID other than the host UID (passed as `--host-uid` or inherited env, SDK enforces) |
| Peer verification (host side) | After dial, host reads `SO_PEERCRED` and checks the peer PID is the child it spawned and the UID is the driver's assigned UID |
| Separate UIDs | Each connection's driver runs under its own UID from a range (`skyranger-drv-{n}`), so drivers cannot `ptrace` or read each other (09) |
| Out-of-container drivers | Third-party drivers in their own container (contract allows bind-mounting the socket dir) must use the same UID scheme; if the socket must cross a network (not supported in v1), mTLS with site-CA client certs is required |

The SDK (`skyranger/pkg/driver/sdk`) implements the peer check so driver authors cannot forget it. Implementation note: grpc-go supports a custom `credentials.TransportCredentials` whose `ServerHandshake` can read peer credentials from the underlying `*net.UnixConn`; this is the hook.

## 7. Other links

| Link | Transport | Auth |
|---|---|---|
| `skyrangerd` → PostgreSQL | TLS (`sslmode=verify-full`), site CA | SCRAM-SHA-256 per role, or client cert (09) |
| `skyrangerd` → Mosquitto | TLS 8883 (Sparkplug driver) and 8884 (internal bus) | client cert CN `skyrangerd` / per-driver cert |
| Site → central (multi-site) | TLS 1.3 HTTPS or MQTT over TLS, outbound | mTLS: site client cert issued by **central's** CA, CN = `gateway_id`; site pins central's CA. Central's CA is distinct from every site CA |
| Remote access | WireGuard (05) | per-user keys + MFA at the jump host |
| NTP (chrony on gateway) | UDP 123, unauthenticated by default | NTS (RFC 8915) for upstream sources when internet exists; field devices use plain SNTP to the gateway (ESP32 has no NTS client we trust). Spoofed time is bounded by the `max_clock_skew` rule (tag-model.md section 5) |
| Syslog forwarding | RFC 5425 syslog over TLS (TCP 6514) | server cert verification (08) |
| Modbus TCP | none possible (Modbus/TCP Security, sometimes called Modbus over TLS, exists but device support is rare; UNVERIFIED spec details) | network segmentation only |

## Decisions made
1. One two-tier site CA per site (ECDSA P-256), generated by `skyrangerd`; root key exported to the recovery kit and removed from the box by default.
2. TLS 1.2 minimum, AEAD-only suites; TLS 1.3 preferred.
3. Mosquitto has no plaintext listener; field listener 8883, internal-bus listener 8884 with mandatory client certs.
4. OPC UA: `SignAndEncrypt` with SHA-256-or-better policies; `None` and SHA-1 policies refused unless explicitly overridden with a reason.
5. gRPC driver boundary uses UDS plus `SO_PEERCRED` checks and per-driver UIDs, not TLS.
6. Multi-site uses a central CA distinct from all site CAs; mTLS with CN = `gateway_id`.

## Alternatives rejected and why
- **Let's Encrypt / ACME on the edge**: needs public DNS and internet; breaks edge-first. Customer can still upload a publicly trusted cert.
- **Self-signed leaf per service**: every client must pin each cert separately; renewal is painful. A site CA is barely more work and scales.
- **step-ca or Vault PKI on the box**: proven, but another always-on service with its own state; the CA operations we need are a few hundred lines of Go.
- **TLS over UDS for gRPC**: adds key management, no protection beyond file permissions and peer credentials.
- **Plaintext MQTT on the isolated field LAN**: Wi-Fi is broadcast media and "isolated" networks are routinely bridged; TLS on ESP32 is well supported.

## Open questions
- Contract example `broker_url: mqtt://mosquitto:1883` (tag-model.md section 9) should become `mqtts://mosquitto:8883` with a `ca: secret://pki/site-ca` (or file) setting in the Sparkplug driver schema. Request for the contract owner.
- Additive endpoint `POST /api/v1/system/tls` (upload customer cert) and `GET /pki/crl.pem` need adding to api.md.
- OPC UA security policy names, certificate SAN rules, and ECC policy support in `gopcua/opcua` are UNVERIFIED.
- Browser certificate lifetime rules for private CAs are UNVERIFIED; 397 d is a conservative default.
- Mosquitto dynamic security plugin directive names are UNVERIFIED for the pinned version.
