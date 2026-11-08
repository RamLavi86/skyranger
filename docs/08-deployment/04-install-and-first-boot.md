# 04 Install and First-Boot Experience

Goal: an integrator with a fresh Debian box and the ESP32 meter on the bench gets live flow rate on the HMI in **under 30 minutes, without internet at the site** (target to measure in milestone 1).

## 1. Install media

A release is distributed as one file plus a detached signature:

```
skyranger-0.4.0-amd64.tar          # release bundle (see 14-release-engineering)
skyranger-0.4.0-amd64.tar.sig      # signature (cosign blob or minisign; see 14)
```

Bundle contents:

```
skyranger-0.4.0-amd64/
  manifest.json            version, image digests, migration list + compat flags, min/max supported previous version
  manifest.json.sig
  compose.yaml             digest-pinned (02-containers)
  images.oci.tar           all images as an OCI archive (docker load), so install works offline
  skyrangerctl             static Go binary for the host
  install.sh               thin wrapper: verifies signature, then runs skyrangerctl install
  host/                    systemd units, chrony.conf, journald drop-in, nftables template, mosquitto templates
  debs/ (optional)         pinned docker-ce, containerd, compose plugin .debs for fully offline installs
  LICENSES/, SBOM.spdx.json
```

Because `images.oci.tar` and optional `debs/` are included, a USB stick is enough. Online installs can pull images from the registry by digest instead (`--online`).

## 2. Install sequence (P1, generic Linux PC)

```
$ sudo ./install.sh
```

`install.sh` does only this: verify `manifest.json.sig` and the bundle hash with the public key embedded in the script and in `skyrangerctl`, then exec `skyrangerctl install --bundle .`. Everything else is in Go so it is testable.

`skyrangerctl install` steps (each is idempotent, so a failed install can be rerun):

1. **Preflight**: Debian version supported, x86-64/arm64, at least 4 GB RAM, at least 20 GB free on the target for `/var/lib/skyranger`, ports 8443/8883/123 free, clock not before the bundle build date (if it is, stop and ask for the correct time). Print a table of pass/warn/fail.
2. **Host packages**: install Docker Engine + Compose plugin (from `debs/` or the pinned apt repo), chrony, nftables. Refuse to proceed if an unsupported Docker version is already installed, unless `--accept-existing-docker`.
3. **Users and directories**: create system group `skyranger`, create the directory tree from [02-containers.md](02-containers.md) section 3 with permissions.
4. **Host config**: chrony (client + LAN server), journald persistent with `SystemMaxUse=2G`, systemd watchdog if `/dev/watchdog`, sshd hardening drop-in, nftables baseline.
5. **Load images**: `docker load -i images.oci.tar`, then verify that each loaded image digest matches `manifest.json`.
6. **Place release**: copy bundle to `/var/lib/skyranger/releases/0.4.0/`, point `current` at it.
7. **Install units**: `skyranger.service`, `skyranger-agent.service`, backup timer. Enable but **do not start** yet.
8. Print: "Run `sudo skyrangerctl init` to configure this gateway."

## 3. `skyrangerctl init` (site identity)

Interactive by default, fully scriptable with flags or `--from site.yaml` for repeatable installs.

```
$ sudo skyrangerctl init
Gateway id (segment grammar, e.g. gw01): gw01
Site display name: Example Water Co. - Pumphouse
Timezone (IANA) [Europe/London]:
LAN interface for HMI/MQTT/NTP [enp1s0 10.0.0.10/24]:
Allowed client networks [10.0.0.0/24]:
Upstream NTP servers (blank = this box is the reference, see warning): 10.0.0.1
Topology [single]: single
Enable simulator profile? [y/N]: y
Admin username [admin]:
Admin password: (or blank to generate a one-time setup code)
```

What it does:
1. Validates `gateway_id` against the segment grammar `^[a-z0-9][a-z0-9_-]{0,62}$` ([00-contracts/README.md](../00-contracts/README.md)).
2. Writes `/etc/skyranger/site.yaml` and `/etc/skyranger/skyranger.env`.
3. Generates secrets into `/etc/skyranger/secrets/`: DB password, MQTT credentials (`secret://mqtt/gateway`, plus a per-device credential slot for the ESP32), and a self-signed TLS certificate for 8443 and 8883 (api.md section 1 says the gateway generates one on first boot if none is provided; we generate it here so MQTT and HTTPS share it). It prints the certificate SHA-256 fingerprint for the user to compare in the browser.
4. Renders Mosquitto config and ACLs (internal bus `skyranger/v1/{gateway_id}/#` writable only by the gateway user, per messages.md section 3).
5. Writes the **initial config** as YAML: one `Gateway` document (`metadata.name = gateway_id`, `display_name`, `timezone`, `sparkplug.host_id: skyranger-{gateway_id}`) and one `Connection` named `local-sparkplug` (driver `sparkplug`, `broker_url: mqtt://mosquitto:1883`, `credentials: secret://mqtt/gateway`, `group_ids: [skyranger]`, discovery enabled). If the simulator is enabled, a `Connection` named `sim1` (driver `sim`). These are exactly the shapes in [tag-model.md](../00-contracts/tag-model.md) section 9.
6. Initializes the local config git repo with that YAML ([05-config-as-code.md](05-config-as-code.md)).
7. Starts `skyranger.service` and waits for health (see sequence below).
8. Prints the URL, fingerprint, and the next step.

## 4. First boot sequence

```
systemd
 ├─ chrony starts; time-sync.target is reached when chronyd reports sync, or after the
 │  skyranger-timewait timeout (11-time-and-power) with a recorded warning
 ├─ docker.service
 └─ skyranger.service
     ├─ ExecStartPre: skyrangerctl preflight (disk, clock plausibility, secrets, release signature)
     └─ docker compose up -d --wait
         ├─ postgres: initdb on empty PGDATA, pg_isready
         ├─ mosquitto: healthy
         └─ skyrangerd:
             1. connect DB, take advisory lock "skyranger-migrate"
             2. run migrations from /usr/share/skyranger/migrations (07-data-and-migrations)
             3. if config_revisions is empty: import /etc/skyranger/bootstrap/*.yaml as revision 1,
                comment "bootstrap by skyrangerctl init", applied_by "system:init"
             4. if users is empty: create admin from /etc/skyranger/secrets/bootstrap-admin, then delete that file
             5. start driver host, tag engine, API on 8443; publish skyranger/v1/gw01/status online
             6. /api/v1/system/health -> ok
 skyranger-agent.service: sees healthy stack, records boot-health.json, starts timers
```

Step 3 and 4 are requirements on `skyrangerd` (config service and auth owners). The bootstrap files are written by `init` and are only read when the database is empty, so they cannot silently overwrite a live config.

If no admin password was given, `init` generates a **one-time setup code** (8 digits plus a check digit), printed on the console and valid for 24 h. The first browser visit to the HMI then shows a "Set admin password" page that requires the code. This needs an unauthenticated setup endpoint, which api.md does not have (see Open questions). Until that exists, `init` requires a password.

## 5. First login and commissioning the water meter

1. Browser to `https://10.0.0.10:8443`, accept the self-signed certificate after comparing the fingerprint printed by `init`.
2. Log in as admin. The HMI landing page shows **System**: gateway id, version, config revision 1, connection `local-sparkplug` state `connected`, and (if enabled) the simulator connection.
3. Provision the ESP32 (firmware doc owns the method; likely serial or captive portal): Wi-Fi, `mqtt_host = 10.0.0.10`, port, credentials, NTP server `10.0.0.10`.
4. The ESP32 sends NBIRTH. The device appears under **Discovered** with profile `skyranger.pulse_meter.v1` and suggested template `pulse_water_meter` (messages.md section 7). The template itself is in the shipped **starter library** (see below).
5. Click Approve, fill `base_path site1/pumphouse`, `instance_name meter01`, `initial_total_l` from the mechanical register. Review the diff. Apply. Revision 2.
6. Flow rate appears; health goes to `ok`. Pour water through the meter (or use the simulator's `pulse_meter` generator) and watch total volume increase.
7. `skyrangerctl status` on the console now shows revision 2 and that the config git repo has a new commit from the HMI apply ([05-config-as-code.md](05-config-as-code.md) section 4).

### Starter library
The release ships `/usr/share/skyranger/library/*.yaml` with `Template` documents (initially `pulse_water_meter`, exactly as in tag-model.md section 11). `init` copies the selected templates into the bootstrap config so revision 1 contains them. They are ordinary config from then on: owned by the site, exportable, editable.

## 6. Re-running and reinstalling

- `skyrangerctl init` refuses to run if `/etc/skyranger/site.yaml` exists, unless `--reconfigure` (which edits env and site identity but never touches the DB).
- Changing `gateway_id` after commissioning is a migration, not a setting: it changes MQTT topics, Sparkplug host id and the northbound edge node id. `skyrangerctl rename-gateway` is out of scope for v1; document it as "export, reinstall, import".
- **Restore onto a new box** (cold spare): `install.sh`, then `skyrangerctl restore --from backup.tar` instead of `init`. Restore writes site identity, secrets and DB, then starts the stack ([07-data-and-migrations.md](07-data-and-migrations.md)).

## 7. Appliance image (P2) first boot

Same sequence, but `install` is already done in the image. On first boot of the data partition, a console and a local web page on a fixed setup address (e.g. `https://<dhcp-ip>:8443/setup`, also shown on HDMI console and as a QR code) run the `init` questions. Until `init` completes, the box serves only the setup page. This needs the setup endpoint mentioned above.

## Decisions made
1. One signed, offline-capable release bundle per architecture (images, compose, host tool, optional host debs, SBOM).
2. `install.sh` only verifies and hands off to `skyrangerctl install`, which is idempotent.
3. `skyrangerctl init` captures site identity, generates secrets and TLS, and writes the initial config as contract-conformant YAML (`Gateway`, `Connection`, starter `Template`s) that `skyrangerd` imports as revision 1 only into an empty database.
4. The ESP32 is commissioned through the standard discovery and approve flow; there is no special-case installer path for devices.
5. Changing `gateway_id` after commissioning is not supported in v1.

## Alternatives rejected and why
- **`curl | bash` online installer**: needs internet at the site and makes signature checking optional in practice.
- **Debian packages (`.deb`) for the whole platform**: good for the host tool, poor for shipping pinned container images and coordinated migrations. We may still package `skyrangerctl` as a `.deb` later.
- **Ansible as the installer**: good for us running many installs, not something a customer engineer should need to reinstall a box. `skyrangerctl` can be driven by Ansible instead.
- **Default admin password (admin/admin)**: a classic SCADA weakness; we generate a secret or a one-time setup code.

## Open questions
- Contract addition request (api.md): an unauthenticated, setup-code-protected `POST /api/v1/setup/admin` that works only while the `users` table is empty. Until agreed, `init` requires a password on the console.
- Contract/tag-engine request: the bootstrap import of `/etc/skyranger/bootstrap/*.yaml` into an empty `config_revisions` table. Alternatively `init` could call `POST /api/v1/config/drafts` + apply after startup, which needs no new behavior but needs an admin token first. Leaning toward the API path to avoid a second import code path; decide with the config owner.
- ESP32 provisioning method (serial, captive portal, BLE) belongs to the firmware doc.
- How the self-signed certificate is replaced by a customer CA certificate: security doc.
