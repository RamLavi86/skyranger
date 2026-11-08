# 04 Sandbox and permissions

Status: design draft. Conforms to [../00-contracts/](../00-contracts/README.md). Feeds the planned STRIDE pass in [../07-security/01-threat-model.md](../07-security/01-threat-model.md) ("When Python scripting lands, scripts become a new boundary"). Runtime in [01-runtime.md](01-runtime.md).

## 1. Threat model

**Who writes scripts:** users with role `engineer` or above (the role that can edit config, api.md section 2). Script authors are *trusted people making mistakes*, and in the platform tier possibly *less-trusted third parties* (plugin script libraries, copied snippets).

**What we defend against, in order:**
1. Bugs: runaway loops, memory blowups, write storms, crashes. (Mostly solved by process isolation, limits, quarantine.)
2. Confused deputy: a script (or a library it imports) doing more than its author intended. (Permissions, deny by default.)
3. A stolen engineer credential or a malicious pasted snippet using scripting as a pivot into the plant LAN or the host. (No network, no filesystem, no subprocess in the worker; egress only via allowlist.)
4. Exfiltration of secrets. (Scripts never hold secrets.)

**What we do not claim:** a Python-level sandbox is **not** a security boundary. Python introspection can defeat import allowlists and audit hooks. The real boundary is the OS: container, UID, seccomp, no network, no tokens in the worker. The Python-level guards exist to produce clear errors and to stop accidents. An `admin` or anyone with root on the box is out of scope (01-threat-model.md already states this).

**Trust domain = pool.** Scripts in one pool share workers and can, if hostile, observe each other through shared worker memory between runs. v1 has one pool. A script that needs stronger separation (e.g. holds powerful permissions) goes to its own pool in v2 (`ScriptPool`, section 2).

## 2. Permission model

Deny by default. A script's effective rights are the **intersection** of what the script declares and what the pool ceiling allows. Declaring something beyond the ceiling is a draft validation error.

```yaml
permissions:
  tags:
    read:  [site1/pumphouse]              # segment-wise prefixes (README naming); "" means all
    write: [site1/pumphouse/meter01/yesterday_l, site1/pumphouse/meter01/low_flow]
  history: { read: [site1/pumphouse] }
  events:
    emit: [daily_report_done, daily_report_incomplete]     # user event names (skyranger.user.{name}.v1)
  alarms: { read: [site1/pumphouse], ack: [] }
  state: true                              # the per-script KV (default true)
  http:
    - host: hooks.example.com              # exact host, no wildcards in v1
      port: 443                            # default 443
      methods: [POST]
      secret_headers: { Authorization: secret://webhooks/leak }
      allow_private: false                 # default false (section 5)
```

Rules:
- Absent key means no permission. `tags.read: []` and a missing `tags` are the same.
- **Matching is segment-wise prefix**, exactly like the tag path rules (`site1/pump` does not grant `site1/pumphouse`). Permissions are on **paths resolved at call time**; renaming a tag moves it out of or into a grant, which is why draft validation warns about permissions whose prefixes match no tag (06).
- Reads of tags carry the same data the `viewer` role can read; writes still require the tag to be `writable`, honor `write_limits`, and pass inverse scaling like any API write.
- `_system/...` tags are readable only if explicitly granted, never writable.
- Permissions are checked **in Go on every call** (host), not in Python. The Python layer has no permission logic to bypass.
- Library code uses the caller's permissions (02 section 3.9).
- The effective permission set is stored with the run record (hash and revision) so "why could it write this?" is answerable.

### Pool ceiling (v1: file in the scripts container; v2: `ScriptPool` kind)

```yaml
# /etc/skyranger/scripts/pool.yaml  (mounted read-only into skyranger-scripts; also read by the host via the Attach handshake)
apiVersion: skyranger/v1
kind: ScriptPool                       # file form in v1; becomes a config kind in v2
metadata: { name: default }
spec:
  workers: 2
  limits: { timeout_max: 60s, memory_max: 512MiB, writes_per_second_max: 100 }
  ceiling:
    tags: { read: [""], write: [site1] }
    http: [ { host: hooks.example.com }, { host: mes.plant.lan, allow_private: true } ]
    events: { emit: ["*"] }
```

Why a file in v1 and not config: the ceiling is a **security control owned by the deployer**, so an `engineer` editing config revisions must not be able to widen it. It is a deployment file edited by someone with box access (admin). v2 may move it to config with a role requirement above `engineer` (open question 2).

### Role vs permission

A manual run records the human requester, but the script runs with **its own** permissions, never the requester's (03). This lets an `operator` trigger "start flush sequence" without receiving general write rights, and means the audit trail has both identities: `requested_by: user:bob`, `performed_by: script:default/flush_sequence`.

## 3. Sandbox layers

Defense in depth; the first three are the boundary, the rest are guardrails.

| # | Layer | Mechanism | What it stops |
|---|---|---|---|
| 1 | Container | `network_mode: none`, read-only rootfs, `cap_drop: [ALL]`, `no-new-privileges`, non-root UID, `pids_limit`, `mem_limit`, `cpus`, tmpfs `/tmp` with `noexec,nosuid` (size-limited) | Network, host filesystem, privilege gain, fork bombs, host memory exhaustion |
| 2 | Process shim | `skyranger-scriptd --sandbox-exec` child: `setrlimit` (`RLIMIT_NOFILE`, `RLIMIT_NPROC`, `RLIMIT_CORE=0`), `prctl(PR_SET_NO_NEW_PRIVS)`, then **seccomp** filter, then `execve(python3)` | Syscalls beyond what Python needs: no `socket`/`connect`/`bind` of any family (including `AF_UNIX` to `host.sock`), no `ptrace`, `mount`, `bpf`, `keyctl`, `unshare`, `setns`, `clone` with namespace flags. Blocking further `execve` after the final one cannot be expressed in a plain filter (UNVERIFIED workaround); subprocesses are stopped by the audit hook, `pids_limit`, and the absence of any useful binary or network in the image |
| 3 | No tokens in the worker | Run tokens exist only in scriptd and the host. Worker pipe has no credentials. `host.sock` is mode `0600` owner scriptd | A compromised worker cannot talk to the host even if it could reach the socket |
| 4 | Import guard | Bootstrap preloads the allowed stdlib, then replaces `builtins.__import__` to refuse any module outside the allowlist **when the importing frame is user code** (module name `sr_user.*`) | Accidental `import os`, `import socket`, `import subprocess`; clear error message |
| 5 | Audit hook | `sys.addaudithook` installed by the bootstrap denies events: `socket.*`, `subprocess.Popen`, `os.system`, `os.exec*`, `os.fork*`, `ctypes.*`, `import` of native extensions outside the stdlib tree, `open` of any path outside the stdlib and the worker directory, `os.remove/rename/mkdir/...` | Same, from stdlib-internal paths; produces `PermissionError`-style `Forbidden` with the audit event name |
| 6 | Clean environment | `python3 -I -S`, empty env except `PYTHONHASHSEED=random`, `PYTHONDONTWRITEBYTECODE=1`, no `site-packages`, `sys.path` fixed | Env-var secrets, user site code, stray packages |
| 7 | Limits and kill | wall-clock, RSS watch, run budgets (03) | Resource exhaustion |

Seccomp tooling: the shim needs a way to install a BPF filter from Go. Candidate libraries: `github.com/elastic/go-seccomp-bpf`, or libseccomp bindings (cgo). Maintenance state and fit are UNVERIFIED; choose in v1 phase and document. The filter must be tested to prove `socket()` returns `EPERM` from inside a worker (a test every release, see 07). Docker's default seccomp profile applies to the container as an outer layer; our filter is stricter inside it.

### Import allowlist (v1)

Allowed for user code: `datetime`, `zoneinfo`, `calendar`, `time` (but `time.sleep` is capped by the run deadline), `math`, `cmath`, `statistics`, `decimal`, `fractions`, `random`, `json`, `re`, `string`, `textwrap`, `uuid`, `hashlib`, `hmac`, `base64`, `binascii`, `struct`, `collections`, `itertools`, `functools`, `operator`, `dataclasses`, `typing`, `enum`, `copy`, `bisect`, `heapq`, `csv` (with `io.StringIO`), `io` (`StringIO`, `BytesIO` only), `skyranger`, `libs.*`.
Denied explicitly: `os`, `sys`, `subprocess`, `socket`, `ssl`, `select`, `selectors`, `asyncio`, `threading`, `multiprocessing`, `concurrent`, `ctypes`, `cffi`, `importlib`, `pkgutil`, `pickle`, `marshal`, `shelve`, `shutil`, `pathlib`, `tempfile`, `glob`, `urllib`, `http`, `ftplib`, `smtplib`, `signal`, `gc`, `inspect`, `builtins` (as import), `code`, `pdb`.
Third-party pip packages are **not available** in v1. Vetted pure-Python extras (and, in v2, plugin script libraries) are vendored read-only into the image or the plugin mount, never `pip install`ed at runtime.

`eval`, `exec`, `compile` and `open` are removed from user-visible `builtins`; dunder-based escapes are not blocked by this layer (it is not a security boundary, section 1).

## 4. Filesystem

- Default: none. `open()` is denied by the audit hook; rootfs is read-only; `/tmp` is a small tmpfs, `noexec`, not exposed through any API.
- Reading reference data (a lookup CSV) is done by putting it in a `ScriptLibrary` as a Python literal or in `state`. No file permission exists in v1.
- A future `permissions.files` (read-only mounts of named directories) is deliberately excluded until a concrete need appears (open questions).

## 5. Outbound HTTP (host-executed)

The worker has no network. `http.request` is a `Call` that the **host** executes with Go's `net/http`:

1. **Allowlist match** on scheme (https only unless the entry says `allow_http: true`), host (exact), port, method. No match: `HttpDenied`.
2. **SSRF defense:** after DNS resolution, the destination IP must be public, unless the entry sets `allow_private: true` (plant-LAN endpoints like an MES, an on-prem SMTP relay). Loopback, link-local (incl. `169.254.169.254`) and the host's own container network are always denied, even with `allow_private`. DNS rebinding is addressed by dialing the checked IP (custom `DialContext`).
3. **TLS verification always on.** CA: the site CA bundle plus system roots; per-entry custom CA is a v2 option. No `verify=False`.
4. **Redirects not followed** by default; if followed, each hop is re-checked against the allowlist.
5. **Secrets:** `secret_headers` values are `secret://` references (contracts: secrets are referenced, never stored in config). The host resolves them at request time and adds the headers; if the script also sets that header, the host's wins. Responses are returned to the script as-is (a server that echoes the secret back would expose it; documented limitation).
6. **Caps:** request body 1 MiB, response body 1 MiB (ceiling 10 MiB), timeout default 10s (max the run's remaining time), at most 10 requests per run (03).
7. **Logging:** URL host and path, method, status, bytes, duration, run_id. Query strings are logged with values redacted. Bodies are never logged.
8. Everything is audited as `script:{pool}/{name}` (08-audit-logging.md owns the record schema).

Egress is therefore reviewable as one list per script plus one list for the pool. The security doc owns the network segmentation story for the host's egress (05-network-segmentation.md).

## 6. Secrets

- Scripts cannot read secrets. There is no `secrets.get()`.
- Secrets reach outbound requests only via `secret_headers` (host-side injection). Basic auth and query-string secrets are not supported in v1 (they would need the secret in the URL, where it ends up in logs).
- `state`, events, logs, tags and run records are all places a script *could* write a value it obtained elsewhere. Since scripts never get secrets, the surface is limited to data from tags and responses.
- Export of config includes `secret://` references, never values (tag-model.md section 9).

## 7. STRIDE pass on the scripting boundary

Numbering continues the table style of 07-security/01-threat-model.md; boundary label **BS** (scripting). IDs are local (S1...), to be merged by the security area.

| ID | STRIDE | Threat | Mitigation | Residual |
|---|---|---|---|---|
| S1 | S | Worker forges calls as another script or run | Run token held by scriptd/host only; per-run, short-lived, bound to `run_id`; no sockets in worker (seccomp) | Compromise of scriptd itself |
| S2 | S | Script spoofs system events (`skyranger.alarm.*`) | `events.emit` can only produce `skyranger.user.*` | none |
| S3 | T | Script alters tags beyond its purpose | Write prefixes in permissions, `writable`, `write_limits`, audit with script principal, rate budgets | Over-broad grants by the engineer: draft warns on `write: [""]` |
| S4 | T | Script source changed outside review | Scripts are config revisions: diff, author, comment, audit; run record stores revision and hash | An engineer can approve their own change (single-person integrator reality) |
| S5 | R | "The script did it" without proof | Run record: run_id, revision, source hash, trigger, calls summary; writes carry `requested_by: script:...` and `comment` | Run records of successful tag_change runs are sampled (05), but every **write** is audited |
| S6 | I | Secret exfiltration | Scripts never see secrets; header injection host-side; secrets not loggable by scripts | Echoing server |
| S7 | I | Reading data beyond scope | `tags.read` prefixes; `_system` explicit | Reads are broad by default for convenience if an engineer grants `""`: warned |
| S8 | I | Exfiltration over HTTP to attacker host | Allowlist is per script and ceilinged by deployer file; no DNS-only exfil because the worker has no resolver | Allowlisted host abuse |
| S9 | D | CPU/memory loop, fork bomb, write storm, log flood | Timeouts, RSS kill, pids limit, budgets, queue bounds, log caps, quarantine | Pool shares CPU with itself; a bad script can slow other scripts, never the core |
| S10 | D | Trigger flood from a hot tag | Per-script rate cap, coalescing, draft warning for broad prefixes | |
| S11 | E | Worker escapes to container root / host | Layers 1 to 3: non-root, no caps, no new privs, seccomp, read-only rootfs, no network, no host mounts except `host.sock` dir and read-only libs | Kernel or CPython-level exploit: keep images patched (07-secure-update.md) |
| S12 | E | Script uses a library's rights | Libraries inherit caller's permissions only; no library-level grants in v1 | |
| S13 | E | `host.sock` abused by another container | Socket directory only mounted into the scripts container, mode `0600`, peer UID checked via `SO_PEERCRED` (as drivers, 03-tls-and-pki.md) | |
| S14 | T | Malicious third-party script library | Libraries are reviewed config; v2 plugin libraries are signed (09) and still run with the caller's permissions | |

## Decisions made
1. The OS (container, seccomp, no network, no tokens in the worker) is the sandbox boundary. Python-level guards (import guard, audit hook) are defense in depth and error clarity, explicitly not a security claim.
2. Deny-by-default permissions: `tags.read/write`, `history.read`, `events.emit`, `alarms`, `http`, `state`, intersected with a pool ceiling the deployer controls through a file, not through config revisions.
3. Permissions are enforced in Go on every host call, using the same prefix semantics as tag paths.
4. A script never acts with its requester's rights; both identities are audited.
5. No filesystem, no subprocess, no sockets, no pip in v1. Outbound HTTP only via the host with exact-host allowlist, SSRF defense, TLS verification, redirect checks, size caps.
6. Scripts never hold secrets; the host injects them into allowlisted request headers.
7. Seccomp is applied by a Go shim before `execve(python3)`; tooling choice is verified during v1.
8. One pool is one trust domain; stronger separation is a second pool in v2.

## Alternatives rejected and why
- **RestrictedPython or `exec` with stripped builtins as the sandbox**: widely known to be bypassable; would create a false sense of security. Used at most as guardrails.
- **gVisor or Firecracker for each worker**: stronger, but heavy for edge boxes and operationally unfamiliar. A possible v3 option for multi-tenant hosts (open questions).
- **Let scripts do arbitrary network I/O within the container's network**: lateral movement into the plant LAN from user code. The allowlisted host proxy keeps egress reviewable.
- **Giving scripts a long-lived API key** (security doc's current `svc-python-runtime`): one leak equals full operator rights, no per-script scoping.
- **Capabilities model tied to the requester's role**: makes script behavior vary by who clicked.
- **Pip-installable packages**: supply-chain and offline-install problems; vendoring vetted pure-Python modules is auditable.
- **Wildcard host allowlists** (`*.example.com`) in v1: invites takeover and typosquat issues; exact hosts first.
- **`secret://` values readable via API but masked in logs**: masking is leaky; not readable at all is simpler.

## Open questions
1. Seccomp library choice and the exact syscall denylist vs allowlist approach: UNVERIFIED, decided in v1 after trying `go-seccomp-bpf` against CPython on both arch targets.
2. Move the pool ceiling into config with a role above `engineer`, or keep it a deployer file? Interacts with security 02 (roles are fixed in v1).
3. Run scriptd and workers under different UIDs without `CAP_SETUID` (user namespaces) so a worker cannot read scriptd memory: needs a Docker/rootless-capable design with deployment area.
4. Does 08-deployment ship the scripts container in the default compose file or as an opt-in profile? Security preference: opt-in.
5. gVisor/Firecracker as an isolation upgrade for hosted or multi-customer deployments.
6. Per-entry custom CA for HTTP egress (private internal PKI) and mTLS client certificates for outbound calls.
7. Whether `time.sleep` should be allowed at all (it burns a worker); today it is allowed and counts against the deadline.
8. A `permissions.files` read-only mount: only if a real need appears.
