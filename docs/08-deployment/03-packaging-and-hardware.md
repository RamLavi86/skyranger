# 03 Edge Box Packaging and Hardware

Three packaging options, one software stack. All three run the same signed release bundle ([06-updates-and-rollback.md](06-updates-and-rollback.md)); they differ in who owns the OS.

| Option | OS owner | Updates | When |
|---|---|---|---|
| P1 Generic Linux PC | Customer IT or us, Debian stable | app release via `skyrangerctl`; OS via `unattended-upgrades` (security only) | milestone 1 and first real projects |
| P2 Sky Ranger appliance image | Us, Yocto-built, read-only root, A/B | RAUC bundle for OS + app release on top | after 2 to 3 real projects, when we support boxes we did not hand-build |
| P3 Raspberry Pi demo kit | Us, Raspberry Pi OS Lite (64-bit) or Debian arm64 | same as P1 | demos, training, bench; never production |

## P1: Generic Linux PC with Docker

Install base: Debian stable (13 "trixie" assumed, UNVERIFIED; 12 "bookworm" as fallback), minimal netinst, no desktop, SSH server only. Then `install.sh` from the release bundle ([04-install-and-first-boot.md](04-install-and-first-boot.md)).

What the install script configures on the host:
- Docker Engine and Compose plugin from a pinned apt source and version.
- `chrony` as NTP client and LAN server, `systemd-journald` persistent with size caps, `unattended-upgrades` for the Debian security suite only.
- Hardware watchdog via systemd (`RuntimeWatchdogSec=`), if `/dev/watchdog` exists.
- `sshd`: key-only auth, `ListenAddress` on the maintenance interface.
- `nftables` baseline: allow 8443, 8883, 123/udp, 22 from configured LAN ranges, drop the rest inbound; Docker's own iptables rules are left on (see Open questions).
- Filesystem: ext4, `/var/lib/skyranger` separate partition, `noatime`, `errors=remount-ro`.

Pros: anyone can source the hardware, customer IT understands it, quickest to field. Cons: OS drift (someone installs things), no atomic OS rollback, Debian major upgrades are a manual, scheduled job.

OS-level rollback on P1: before a planned Debian upgrade, take an LVM snapshot or a full disk image (`skyrangerctl os snapshot`, which uses LVM if the root is on LVM; otherwise it refuses and tells you to image the disk). We do not invent an A/B scheme on top of a mutable distro.

## P2: Signed appliance image (Yocto + RAUC)

Target state when we support several boxes we did not install by hand.

- **Build**: Yocto (an LTS release, e.g. Scarthgap 5.0, UNVERIFIED which LTS is current at build time), `meta-virtualization` for docker-moby, `meta-rauc` for updates, `meta-security` optional. Machine layers: `meta-intel` for x86-64 boxes; `meta-raspberrypi` only if we ever ship Pi-based appliances.
- Why Yocto over Buildroot: Buildroot has no package management and a full rebuild model, which is fine, but Docker and its dependencies plus RAUC plus systemd are better trodden in Yocto layers, and Yocto LTS gives a security update cadence. Buildroot remains a good option for a much smaller, non-container appliance; we are not that.
- **Partition layout** (GPT, single SSD):

```
  p1  EFI system / boot   256 MiB   vfat     (GRUB or systemd-boot + RAUC boot selection, see below)
  p2  rootfs A            2 GiB     ext4 or squashfs, mounted read-only
  p3  rootfs B            2 GiB     same
  p4  factory / recovery  1 GiB     read-only rescue system with skyrangerctl restore
  p5  data                rest      ext4, /var/lib/skyranger, /var/lib/docker, /etc/skyranger (bind), journald
```

- Root filesystem is read-only. Writable host config lives on `p5` and is bind-mounted to `/etc/skyranger` and the few `/etc` files that must be writable (`/etc/machine-id`, `/etc/hostname`, network config) via an overlay or bind. Docker's `/var/lib/docker` is on `p5`, so app images survive OS slot switches.
- **Bootloader and A/B**: RAUC with GRUB on x86 (RAUC supports GRUB, U-Boot, barebox and EFI boot selection; UNVERIFIED exact EFI integration details per RAUC version). Boot counting: the bootloader decrements a try counter for the new slot; if userspace does not mark the slot good (`rauc status mark-good`) after the health gate, the next boot falls back to the old slot.
- **Signing**: RAUC bundles are signed with an X.509 key; the device holds the CA cert in the read-only rootfs and rejects unsigned bundles. Keys are in an offline or hardware-backed store ([14-release-engineering.md](14-release-engineering.md)).
- **Secure Boot**: optional, per customer. Measured/secure boot with signed kernel and dm-verity rootfs is a later hardening step; the security doc owns it.
- **Factory reset**: boots `p4`, offers "restore from backup on USB" or "wipe data and re-run first boot". It never touches the OS slots.

Pros: atomic OS updates with automatic rollback, no OS drift, known kernel and drivers, reproducible. Cons: a real Yocto build pipeline to maintain, longer feature turnaround for host-level changes, we own kernel CVEs.

## P3: Raspberry Pi class (demos only)

- Raspberry Pi 5 (4 or 8 GB) with an **NVMe SSD via the M.2 HAT** or a USB 3 SSD; boot from SSD. A good 27 W supply (the official PSU).
- Do not run PostgreSQL on an SD card. SD cards have poor sustained random-write endurance and unpredictable behavior on power loss; we have seen this fail in the field in general embedded work, and it is common knowledge in the Pi community. This is the main reason the Pi is demo-only.
- No RTC battery by default on older Pis (the Pi 5 has an RTC that needs an external battery, UNVERIFIED battery part). Without it, time is wrong until NTP syncs, which matters ([11-time-and-power.md](11-time-and-power.md)).
- No hardware watchdog wiring guarantees, consumer-grade temperature range, no DIN mount, no wide-range DC input.
- Good for: trade-show kit with the ESP32 meter and the simulator, training labs, developer bench.

## Hardware recommendations

These are characteristics, not endorsements. Specific models change every year; check current availability and get an evaluation unit before quoting.

### Minimum and recommended (x86-64 industrial PC)

| Item | Minimum (milestone, < 500 tags) | Recommended (up to ~10,000 tags at 1 s) | Why |
|---|---|---|---|
| CPU | 4-core x86-64 (Intel N-series / Atom-class) | 4 to 8 core, Core i3/i5 embedded class | Go core is light; PostgreSQL compression jobs and history queries are the heavy part |
| RAM | 4 GB | 16 GB | PostgreSQL shared buffers and page cache matter more than CPU |
| Storage | 128 GB industrial SSD | 256 to 1000 GB industrial SSD with **power-loss protection (PLP)** capacitors, high TBW | History is the bulk; PLP protects flushes on power cut |
| Second storage (optional) | none | 2nd SSD or USB for local backups | survives first disk failure |
| Network | 1 Ethernet | 2 Ethernet (OT and uplink separated) | network separation, see security doc |
| Watchdog | software only | hardware watchdog exposed as `/dev/watchdog` (iTCO, IT87xx, or vendor) | recovery from kernel or hardware hangs |
| RTC | coin cell RTC | coin cell RTC | correct time before NTP |
| Power | wide-range DC (9 to 36 V) | wide-range DC + DIN-rail UPS or supercap module with a "power fail" digital input or USB/serial signal | graceful shutdown on loss of supply |
| Environment | 0 to 50 C, fanless | -20 to 60 C fanless, DIN-rail mount, conformal coating if humid | pumphouses are not server rooms |
| Serial | optional | 1 to 2 RS-485 ports | future Modbus RTU driver |
| Certification | CE / UKCA / FCC as needed | plus industrial EMC (EN 61000-6-2) where specified | customer spec sheets ask for it (UNVERIFIED which standards each customer requires) |

### Sizing history storage (rule of thumb, to validate)

Uncompressed hypertable row for one numeric sample: roughly 40 to 60 bytes including index overhead (UNVERIFIED; depends on the historian schema). TimescaleDB native compression on numeric time series commonly reports 10x to 20x (UNVERIFIED for our data shape).

Example: 1,000 tags, on-change with deadbands, average 1 change per 10 s:
`1000 * 8640 samples/day * 50 B = ~430 MB/day` uncompressed, `~25 to 45 MB/day` compressed, `~10 to 16 GB` for the default 400 day retention (`Gateway.spec.history.default_retention: 400d`). A 256 GB SSD is comfortable; leave 50% free for compression jobs, WAL, backups and SSD wear levelling.

The historian doc owns the real numbers; `skyrangerctl doctor` reports actual bytes per sample and projected days to full.

### Caveats
- Cheap mini-PCs often lack a usable hardware watchdog in Linux, have consumer SSDs without PLP, and throttle under heat in a closed cabinet. Test before you ship.
- USB-attached storage for the main DB: avoid; USB resets under EMI are common.
- If the box will sit on a UPS, the UPS must signal the box. A UPS that runs flat without telling anyone only moves the power cut by twenty minutes.
- Virtual machines: supported as P1 on customer hypervisors (VMware, Proxmox, Hyper-V). No hardware watchdog (use the hypervisor's virtual watchdog device where available, e.g. `i6300esb` on KVM). Time sync must come from chrony, not hypervisor tools alone.

## Decisions made
1. P1 generic Debian + Docker on an industrial x86-64 PC is the first production target.
2. P2 appliance image uses Yocto LTS + `meta-virtualization` + RAUC, with a read-only A/B root, a recovery partition, and a data partition holding Docker images and all state.
3. P3 Raspberry Pi 5 with SSD is demo/bench only; SD-card PostgreSQL is never supported.
4. Recommended hardware characteristics: industrial SSD with PLP, hardware watchdog, RTC, wide-range DC input, power-fail signal, two NICs.
5. VMs are supported as P1 with chrony and a virtual watchdog.

## Alternatives rejected and why
- **Buildroot for P2**: excellent for small fixed-function images, weaker fit for Docker + systemd + RAUC + a security update stream.
- **Ubuntu Core / snaps**: proven A/B and signing, but ties updates to the Snap Store model and Canonical's infrastructure, which conflicts with "customers can leave easily" and outbound-only on-prem update mirrors.
- **balenaOS**: good fleet A/B container OS, but the fleet side is a commercial service (self-hosted openBalena exists with reduced features, UNVERIFIED current state). Rejected as a dependency we do not control; worth a look as a reference design.
- **Mender instead of RAUC**: also proven, with a server component. RAUC is a client-side updater with no required server, which fits outbound-only and "fleet is optional". Mender remains a valid alternative if we later want its server UI.
- **ARM industrial boxes as the default**: viable, but x86-64 gives the widest choice of off-the-shelf industrial PCs and the simplest image builds. arm64 images are still built for P3 and future boxes.

## Open questions
- Docker's iptables rules bypass a simple nftables INPUT policy for published ports (known Docker behavior; exact interaction with nftables on Debian 13 UNVERIFIED). Binding published ports to the LAN address helps; the security doc should decide the final firewall approach.
- Which industrial PC vendors offer a documented Linux hardware watchdog and a power-fail signal: requires evaluation units.
- Whether the P2 image should support arm64 industrial boxes from day one.
- Storage sizing numbers above are UNVERIFIED until the historian schema exists and we measure on the simulator.
