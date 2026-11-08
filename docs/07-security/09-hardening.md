# 09 Hardening: Edge Box, Containers, Drivers, Services

Baseline target: Debian stable (current release at time of build) on x86_64 or ARM64, 4 GB RAM minimum, TPM 2.0 recommended. The deployment doc chooses the exact OS image and container runtime; this file is the security baseline that image must meet. CIS Benchmarks for Debian and Docker are useful references; we do not claim CIS compliance (UNVERIFIED which items we would pass).

## 1. Host OS

| Area | Requirement | How (example) |
|---|---|---|
| Firmware | UEFI Secure Boot on, BIOS/UEFI password set, boot from USB/network disabled | vendor setup; recorded in commissioning report |
| Disk | Full-disk encryption LUKS2, unlock by TPM2 (+ recovery key in recovery kit) | `systemd-cryptenroll --tpm2-device=auto --tpm2-pcrs=7 /dev/nvme0n1p3` (PCR choice UNVERIFIED per platform) |
| Packages | Minimal install: no desktop, no compilers, no `telnet`, `ftp`, `rsh`, no remote desktop tools | debootstrap/preseed list maintained in repo |
| Users | No password login for any account; root locked; one admin account `sradmin` with SSH key only and sudo | `passwd -l root`; `/etc/ssh/sshd_config.d/skyranger.conf` below |
| SSH | Optional; key-only, no root, limited to mgmt subnet, `AllowUsers sradmin` | see below |
| Firewall | nftables policy from 05 section 4, loaded at boot before network-online | `nftables.service` enabled |
| Kernel | sysctl hardening below; unused modules blacklisted (`usb-storage` optional: needed for USB update bundles, so mounted only by the updater via a udev rule) | `/etc/sysctl.d/90-skyranger.conf` |
| Time | chrony, serving NTP to field VLAN (tag-model.md section 5) | `allow 10.20.3.0/24` |
| Logging | journald persistent, size-capped; forwarded with audit (08) if configured | `SystemMaxUse=1G` |
| Updates | Security updates in maintenance windows, audited (07) | |
| Services | Only: `sshd` (optional), `chronyd`, container runtime, `skyranger-updater`, `skyranger-netd` | `systemctl list-unit-files --state=enabled` checked in 14 |
| Physical | Box in locked cabinet; unused USB ports blocked or disabled in BIOS; serial console requires the `sradmin` password-less key? No: serial console login uses a local password stored only in the recovery kit | |

```conf
# /etc/ssh/sshd_config.d/skyranger.conf
PermitRootLogin no
PasswordAuthentication no
KbdInteractiveAuthentication no
PubkeyAuthentication yes
AllowUsers sradmin
X11Forwarding no
AllowTcpForwarding no
AllowAgentForwarding no
MaxAuthTries 3
LoginGraceTime 30
ClientAliveInterval 300
```

```conf
# /etc/sysctl.d/90-skyranger.conf
kernel.kptr_restrict = 2
kernel.dmesg_restrict = 1
kernel.unprivileged_bpf_disabled = 1
kernel.yama.ptrace_scope = 2
net.ipv4.ip_forward = 0                       # 1 only if the box routes between VLANs (05)
net.ipv4.conf.all.rp_filter = 1
net.ipv4.conf.all.accept_redirects = 0
net.ipv4.conf.all.send_redirects = 0
net.ipv4.conf.all.accept_source_route = 0
net.ipv6.conf.all.accept_redirects = 0
net.ipv4.tcp_syncookies = 1
fs.protected_hardlinks = 1
fs.protected_symlinks = 1
fs.suid_dumpable = 0
```

## 2. Containers

Baseline for every container (Docker Compose syntax; Podman equivalents exist):

```yaml
x-hardening: &hardening
  read_only: true
  user: "10001:10001"                 # per-service fixed non-root UID
  cap_drop: [ALL]
  security_opt:
    - no-new-privileges:true
    - seccomp=default                 # runtime default profile; custom profile for skyrangerd later
    - apparmor=docker-default         # where AppArmor is available
  tmpfs:
    - /tmp:size=64m,mode=1777,noexec,nosuid,nodev
  pids_limit: 512
  restart: unless-stopped
  logging:
    driver: journald

services:
  skyrangerd:
    <<: *hardening
    image: registry.example/skyranger/skyrangerd@sha256:...
    user: "10001:10001"
    cap_add: [SETUID, SETGID]         # ONLY to drop driver children to per-driver UIDs (section 3); see note
    ports: ["10.20.2.10:8443:8443"]   # bind to the specific host IP, never 0.0.0.0
    volumes:
      - sr_data:/var/lib/skyranger            # PKI store, setup code, local WAL
      - sr_run:/run/skyranger                 # driver sockets (tmpfs volume)
    networks: [backend]
    mem_limit: 1g
  mosquitto:
    <<: *hardening
    image: registry.example/skyranger/mosquitto@sha256:...
    user: "10002:10002"
    ports: ["10.20.2.10:8883:8883"]
    volumes: [mqtt_data:/mosquitto/data, mqtt_pki:/mosquitto/pki:ro]
    networks: [backend]
  postgres:
    <<: *hardening
    image: registry.example/skyranger/postgres-timescaledb@sha256:...
    user: "10003:10003"
    read_only: true
    volumes: [pg_data:/var/lib/postgresql/data, pg_run:/var/run/postgresql]
    networks: [backend]               # no published ports at all
networks:
  backend:
    internal: true                    # no default route out; egress only via host-allowed paths
    ipam: { config: [{ subnet: 172.30.0.0/24 }] }
```

Rules:
- Never: `privileged: true`, `network_mode: host`, `pid: host`, Docker socket mount (`/var/run/docker.sock`) into any container, `cap_add: [SYS_ADMIN, NET_ADMIN, SYS_PTRACE]`.
- `skyranger-netd` (nftables helper, 05) and `skyranger-updater` run on the **host** as systemd units with narrow sandboxing (section 5), not as privileged containers.
- Published ports bind to the specific site interface IP. PostgreSQL has no published port.
- Images: distroless or minimal base (`gcr.io/distroless/static` or Debian slim for Python later), no shell in the `skyrangerd` image in production builds (a `-debug` variant exists for development).
- Rootless container runtime (rootless Podman or Docker rootless) is preferred where the deployment doc can support it; then container root maps to an unprivileged host UID as a second barrier.
- Note on `SETUID`/`SETGID`: the driver host needs to start children under distinct UIDs. Alternative without capabilities: run each driver as the same UID inside the container but rely on seccomp and separate network namespaces; weaker. Decision in section 3.

## 3. Driver process sandboxing

Driver-interface.md leaves sandboxing to this doc. A driver parses untrusted bytes from the field (Modbus responses, Sparkplug protobuf, OPC UA binary), so it is the most likely component to have a memory-safety or logic bug exploitable from the field.

| Control | Requirement |
|---|---|
| UID | Each connection's driver runs as its own UID `20000 + n` (allocated by the host, stable per connection name), GID `skyranger-drv`. The host drops privileges in the child after fork/exec using `SysProcAttr.Credential` and then the host itself retains `SETUID`/`SETGID` only; `no-new-privileges` stays on |
| Filesystem | Child sees only its socket (via the shared `0750` dir) and read-only binaries. No access to `/var/lib/skyranger` (PKI, setup code). Implemented with permissions (directory owned by host UID `0700`) |
| Secrets | Receives only its own connection's resolved settings over the UDS (06). Never sees the KEK, DB credentials, or other connections' secrets |
| Database | No DB credentials; drivers cannot reach PostgreSQL (`pg_hba.conf` allows only the `skyrangerd` container IP with specific roles, and drivers share that IP, so additionally: drivers have no password) |
| Network | Drivers need outbound to their device only. Inside the shared container network namespace we cannot restrict per process with nftables easily; mitigation: nftables `meta skuid` match in the container's netns is not available to an unprivileged container. **v1 accepts this gap**; drivers can reach anything the container can. Host output policy (05) still limits the container to configured device IPs |
| seccomp | v1: container default profile. v1.x: driver SDK installs an additional seccomp filter after startup (deny `execve`, `ptrace`, `mount`, `bpf`, `keyctl`, etc.) using a Go seccomp library (`elastic/go-seccomp-bpf` or `libseccomp-golang`, evaluate) |
| Resources | `RLIMIT_AS`/cgroup memory per driver (default 256 MiB), CPU share; the host kills on overrun and restarts with backoff (contract lifecycle) |
| Rate | Contract limits (batch size, deadlines) apply; malformed input increments counters, never crashes the host |

Third-party drivers in their own container (contract option) get the same container baseline as section 2, with only the socket dir mounted.

## 4. PostgreSQL

| Item | Setting |
|---|---|
| Network | Listens on the internal `backend` network only; no published port |
| TLS | `ssl = on`, cert from site CA, `ssl_min_protocol_version = 'TLSv1.2'` |
| Auth | `password_encryption = scram-sha-256`; `pg_hba.conf` below; no `trust` lines; superuser `postgres` password random, stored in recovery kit only, not used by any service |
| Roles | `sr_migrate` (DDL, used only by the migration job during updates), `sr_app` (DML on application tables, `SELECT` on `secrets` only for `skyrangerd`), `sr_audit_writer` (INSERT/SELECT on `audit_log`), `sr_readonly` (reporting, optional, `SELECT` on history views, never on `secrets`, `users`, `sessions`, `api_keys`) |
| Extensions | Only `timescaledb` (and its dependencies); no `plpython`, no `adminpack`, no `dblink`/`postgres_fdw` |
| Logging | `log_connections = on`, `log_disconnections = on`, `log_statement = 'ddl'` |

```conf
# pg_hba.conf
# TYPE   DATABASE   USER              ADDRESS          METHOD
local    all        postgres                           peer
hostssl  skyranger  sr_app            172.30.0.10/32   scram-sha-256
hostssl  skyranger  sr_audit_writer   172.30.0.10/32   scram-sha-256
hostssl  skyranger  sr_migrate        172.30.0.20/32   scram-sha-256
hostssl  skyranger  sr_readonly       172.30.0.30/32   scram-sha-256
hostnossl all       all               0.0.0.0/0        reject
host      all       all               0.0.0.0/0        reject
```

Customer reporting tools that want SQL access ("plain SQL" is a selling point, decision 3) connect through `sr_readonly` via a published port on the supervisory VLAN only if the customer asks, with TLS and a source-restricted firewall rule.

## 5. Host-side helpers (systemd sandboxing)

```ini
# /etc/systemd/system/skyranger-netd.service
[Service]
ExecStart=/usr/lib/skyranger/skyranger-netd --socket /run/skyranger-netd/netd.sock
User=root                         # needs CAP_NET_ADMIN only
CapabilityBoundingSet=CAP_NET_ADMIN
AmbientCapabilities=CAP_NET_ADMIN
NoNewPrivileges=yes
ProtectSystem=strict
ProtectHome=yes
PrivateTmp=yes
PrivateDevices=yes
ProtectKernelModules=yes
ProtectKernelTunables=yes
RestrictAddressFamilies=AF_UNIX AF_NETLINK
SystemCallFilter=@system-service
RuntimeDirectory=skyranger-netd
```

`skyranger-netd` accepts JSON `{"set": "device_egress", "elements": [{"ip": "10.20.1.20", "proto": "tcp", "port": 502}]}` from `skyrangerd` only (peer UID check), validates every field (IP literal, port 1 to 65535, proto in {tcp, udp}), and applies them with netlink (`google/nftables` Go library). It cannot add input rules or touch other tables.

## 6. HMI (browser side)

- Headers from 03 section 3 (CSP, HSTS, nosniff, frame-ancestors none).
- No third-party scripts, fonts, or analytics; all assets served from the gateway (also required for offline operation).
- React default escaping; lint rule forbids `dangerouslySetInnerHTML`; device- and config-supplied strings (`display_name`, alarm `message`, discovery `attributes`) are rendered as text only.
- Write confirmations show the target tag path and value and require an explicit click; the HMI never sends writes on hover, drag, or keyboard repeat.
- Session token in memory and `sessionStorage` (contract); cleared on logout and on `4401`.

## 7. Mosquitto

- Config from 03 section 4: no anonymous, no plaintext listener, no websockets listener.
- `max_inflight_messages`, `max_queued_messages`, `max_packet_size` set (e.g. `max_packet_size 262144`) to bound memory from misbehaving clients (directive names UNVERIFIED for pinned version).
- `persistence true` on its own volume; dynamic-security JSON file mode `0600`.

## Decisions made
1. Debian stable minimal host with Secure Boot, LUKS2 + TPM, key-only SSH from the mgmt subnet, nftables input and output policy.
2. All containers non-root, read-only root FS, all capabilities dropped, `no-new-privileges`, published ports bound to the site IP; PostgreSQL never published by default.
3. No privileged containers and no Docker socket mounts; privileged actions are two small host helpers with systemd sandboxing and typed inputs.
4. Each driver connection runs under its own UID with no access to secrets, DB, or PKI; seccomp tightening in v1.x.
5. PostgreSQL uses separate least-privilege roles, SCRAM, TLS, and `pg_hba` reject-by-default.

## Alternatives rejected and why
- **Running drivers as the same UID as `skyrangerd`**: a field-facing parser bug would give access to the KEK in memory and to DB credentials.
- **Privileged container for nftables management**: one compromise gives host root.
- **Alpine base for Go images**: musl differences are not worth it; distroless static is smaller and has no shell.
- **SELinux-only hardening**: Debian defaults to AppArmor; supporting both via the runtime defaults is enough.
- **Disabling SSH entirely by default**: makes field support hard for a one-person integrator; key-only from the mgmt subnet is a reasonable default, and the customer can disable it.

## Open questions
- Per-driver network restriction inside the container is not achieved in v1 (documented gap). Options: a network namespace per driver (needs `CAP_SYS_ADMIN`, rejected), or drivers in separate containers by default for customer sites.
- Rootless Podman vs Docker: deployment doc decision; affects UID mapping above.
- Custom seccomp profile for `skyrangerd`: generate from tests with a tracing tool, then enforce.
- CIS benchmark gap analysis: UNVERIFIED, not done.
