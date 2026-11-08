# 05 Network Segmentation and Reference Network

## 1. Purdue model, as we use it

The Purdue Enterprise Reference Architecture (PERA), as adopted in ICS security practice, describes levels from physical process (Level 0) up to enterprise (Level 4/5), with an industrial DMZ between OT and IT. Exact level definitions vary between sources (ISA-95, ISA/IEC 62443 materials, vendor reference architectures); we use the common practitioner version below and do not claim normative alignment to any particular document. IEC 62443-3-2 uses **zones and conduits** rather than Purdue levels; section 5 maps our design to zones and conduits.

| Level | Common meaning | Sky Ranger components |
|---|---|---|
| L5/L4 | Enterprise / business network, internet edge | Office PCs, customer IdP, central/cloud Sky Ranger (multi-site) |
| L3.5 | Industrial DMZ (IDMZ) | Optional: reverse proxy for HMI from office, remote-access jump point, sync relay. Small sites may have none |
| L3 | Site operations (historian, site-wide servers) | For a small site, merged with L2 on the edge box |
| L2 | Supervisory control (SCADA, HMI) | **Edge box** (`skyrangerd`, Mosquitto, PostgreSQL), HMI workstations |
| L1 | Basic control (PLCs, RTUs, controllers) | PLCs, OPC UA servers on PLCs, ESP32 meters (they act as field I/O plus simple controller) |
| L0 | Physical process (sensors, actuators) | Pulse output of the water meter, pumps, valves |
| — | Safety instrumented systems | **Not Sky Ranger.** Separate, independent (11) |

Our placement: the edge box is an L2/L3 device. It reaches **down** to L1 and is reached **from** L2 HMI clients and, through a controlled conduit, from L4. It never accepts connections from the internet.

## 2. Reference network: small site

Target: one site, one edge box, a few ESP32 meters, one PLC, office network present, internet present but untrusted. Uses one small industrial firewall/router with VLANs (e.g. OPNsense/pfSense appliance, or a managed industrial switch plus firewall). If the customer has no capability for this, the edge box itself can be the router between field and supervisory networks (two NICs, nftables, section 4) as a minimum.

```
                   ┌───────────────┐
  Internet ────────┤ ISP router    │
                   └──────┬────────┘
                          │ WAN
                   ┌──────┴─────────────────────────────────────────┐
                   │ Site firewall (OT/IT boundary)                 │
                   │  VLAN 10 Office      10.10.0.0/24  (L4)        │
                   │  VLAN 30 DMZ         10.30.0.0/24  (L3.5, opt) │
                   │  VLAN 20 Supervisory 10.20.2.0/24  (L2)        │
                   │  VLAN 21 Field wired 10.20.1.0/24  (L1)        │
                   │  VLAN 22 Field Wi-Fi 10.20.3.0/24  (L1)        │
                   │  VLAN 99 Mgmt        10.99.0.0/24  (switch/AP/BMC mgmt) │
                   └──┬──────────┬──────────┬─────────┬─────────────┘
                      │          │          │         │
                 Office PCs   Edge box   PLC/OPC UA  Field AP (SSID "sr-field", WPA2/3-PSK or WPA2-Enterprise)
                 HMI browser  10.20.2.10 10.20.1.20      └── ESP32 meters 10.20.3.x (DHCP reservations)
                              (+ HMI panel PCs in VLAN 20)
```

Edge box addressing: one interface in VLAN 20 (`10.20.2.10`). With two NICs, a second interface in VLAN 21 is allowed only if the firewall still enforces VLAN 22 to VLAN 21 isolation; simplest is one NIC and route through the firewall.

Field Wi-Fi: dedicated SSID, client isolation enabled on the AP (meters cannot talk to each other), no internet route, DHCP from the firewall with reservations per MAC (helps spot unknown devices). Wireless is never bridged into VLAN 21.

## 3. Firewall rules (site firewall)

Default policy: **deny all**, both directions, between every pair of zones. Log denied packets from field zones (they are rare and interesting). Rules below are the complete allow list for milestone scope.

| # | From | To | Proto/Port | Purpose | Notes |
|---|---|---|---|---|---|
| 1 | VLAN 22 Field Wi-Fi | Edge 10.20.2.10 | TCP 8883 | ESP32 MQTT over TLS | |
| 2 | VLAN 22 Field Wi-Fi | Edge 10.20.2.10 | UDP 123 | SNTP from gateway | |
| 3 | VLAN 22 Field Wi-Fi | Edge 10.20.2.10 | TCP 8443 | Device enrollment and OTA download (04) | Restrict to `/api/v1/devices/enroll` and `/fw/` at a reverse proxy if available; otherwise rely on API auth |
| 4 | VLAN 22 | firewall | UDP 67/68, UDP 53 (only if devices use DNS) | DHCP/DNS | Prefer IP literals; no external DNS forwarding for VLAN 22 |
| 5 | Edge 10.20.2.10 | PLC 10.20.1.20 | TCP 502 | Modbus TCP polling | Source = edge only. No other host may reach 502 |
| 6 | Edge 10.20.2.10 | PLC/OPC UA 10.20.1.20 | TCP 4840 | OPC UA client | |
| 7 | VLAN 20 HMI clients | Edge 10.20.2.10 | TCP 8443 | HMI, REST, WSS | Intra-VLAN; enforce on host firewall too (section 4) |
| 8 | VLAN 10 Office (named group) | Edge 10.20.2.10 *or* DMZ proxy 10.30.0.5 | TCP 8443 / 443 | Office HMI access | Prefer via DMZ proxy. Restrict source to an "SCADA users" address group, not the whole office |
| 9 | DMZ proxy 10.30.0.5 | Edge 10.20.2.10 | TCP 8443 | Proxy to gateway | Only when rule 8 uses the proxy |
| 10 | Edge 10.20.2.10 | Internet (specific FQDNs/IPs) | TCP 443 | Optional cloud sync (mTLS), update registry pull | Outbound only. Destination allow list. Disabled when no cloud |
| 11 | Edge 10.20.2.10 | Internet NTP or customer NTP | UDP 123 (NTS: TCP 4460) | Upstream time | |
| 12 | Edge 10.20.2.10 | Jump host (internet) | UDP 51820 | WireGuard remote access, outbound | Only while a remote session is enabled (section 6) |
| 13 | Edge 10.20.2.10 | Customer syslog/SIEM | TCP 6514 | Audit forwarding over TLS (08) | Optional |
| 14 | VLAN 99 Mgmt admin PC | Edge 10.20.2.10 | TCP 22 | SSH (key-only) for maintenance | Optional; can be disabled entirely in favour of console |
| — | Internet | anything | any | **Denied. No port forwards, no UPnP, no inbound NAT.** | Non-negotiable |
| — | VLAN 21/22 | Internet | any | **Denied.** Field devices never reach the internet | |
| — | VLAN 21 ↔ VLAN 22 | | any | Denied | |
| — | VLAN 10 Office | VLAN 21/22 | any | Denied: office never reaches field directly | |

OPNsense/pfSense, nftables, or vendor syntax will differ; the table is the specification. Example in nftables syntax for the case where the edge box itself is the router between VLANs 20, 21, and 22 (interfaces `sup0`, `fld0`, `wifi0`):

```nft
table inet sr_router {
  chain forward {
    type filter hook forward priority 0; policy drop;
    ct state established,related accept
    ct state invalid drop
    # rule 5/6: edge (local) to PLC is output, not forward; see host rules
    # Field to field: drop
    iifname "wifi0" oifname "fld0" drop
    iifname "fld0"  oifname "wifi0" drop
    # HMI clients on supervisory LAN never reach field
    iifname "sup0" oifname { "fld0", "wifi0" } log prefix "sr-drop-sup2field " drop
  }
}
```

## 4. Host firewall on the edge box (always on, defence in depth)

Even with a site firewall, the edge box runs its own nftables policy. Containers publish ports to the host; Docker/Podman insert their own rules, so the policy hooks at a priority that runs before container NAT, or containers bind published ports only to the specific host IP. The deployment doc chooses Docker vs Podman; both must be validated with an external port scan (14).

```nft
#!/usr/sbin/nft -f
# /etc/nftables.d/skyranger.nft
table inet sr_host {
  set hmi_clients   { type ipv4_addr; flags interval; elements = { 10.20.2.0/24, 10.30.0.5 } }
  set field_devices { type ipv4_addr; flags interval; elements = { 10.20.3.0/24 } }
  set mgmt_admins   { type ipv4_addr; elements = { 10.99.0.20 } }
  set egress_allow  { type ipv4_addr; flags interval; }   # filled by skyranger-netd when cloud/registry/remote enabled

  chain input {
    type filter hook input priority filter; policy drop;
    iif lo accept
    ct state established,related accept
    ct state invalid drop
    ip protocol icmp icmp type { echo-request, destination-unreachable, time-exceeded } limit rate 10/second accept
    ip saddr @hmi_clients   tcp dport 8443 accept
    ip saddr @field_devices tcp dport { 8883, 8443 } accept
    ip saddr @field_devices udp dport 123 accept
    ip saddr @mgmt_admins   tcp dport 22 accept
    log prefix "sr-host-drop " limit rate 5/second drop
  }

  chain output {
    type filter hook output priority filter; policy drop;
    oif lo accept
    ct state established,related accept
    ip daddr 10.20.1.20 tcp dport { 502, 4840 } accept            # field devices per Connection
    ip daddr @field_devices tcp sport 8883 accept                  # (covered by established; explicit for clarity)
    udp dport 123 accept                                           # upstream NTP
    tcp dport 4460 accept                                          # NTS-KE
    ip daddr { 10.20.2.1 } udp dport 53 accept                     # site DNS resolver only
    # cloud sync / registry / WireGuard: managed by skyrangerctl, added to set when enabled
    ip daddr @egress_allow tcp dport 443 accept
    ip daddr @egress_allow udp dport 51820 accept
    log prefix "sr-host-out-drop " limit rate 5/second drop
  }
}
```

Output filtering matters: if the box is compromised, it should not be able to reach arbitrary internet hosts or scan the field LAN. `skyrangerd` generates the field-device output rules from `Connection` configs (each `opcua`/`modbus` endpoint), applied by a small privileged helper (`skyranger-netd`) at config apply time and audited. This is the only privileged action triggered by config, and the helper accepts only a typed list of `{ip, port, proto}`, not raw nft text.

## 5. Zones and conduits view (IEC 62443-3-2 style)

| Zone | Contents | Target SL (proposed) | Conduits |
|---|---|---|---|
| Z1 Enterprise | Office, IdP | n/a (customer IT) | C1 to Z3 (HTTPS via DMZ) |
| Z2 IDMZ | Reverse proxy, jump point | SL 2 | C1, C2 |
| Z3 Supervisory | Edge box, HMI stations | SL 2 | C2 (HTTPS), C3 (field), C4 (cloud egress) |
| Z4 Field wired | PLCs, OPC UA servers | SL 1 to 2 (depends on PLC capability) | C3 |
| Z5 Field wireless | ESP32 meters | SL 2 for confidentiality/integrity of the link; availability is not defendable against jamming | C3 |
| Z6 Safety | SIS | Out of scope; no conduit to Sky Ranger except one-way status read if the safety engineer approves (11) | none by default |

SL targets are our proposal for the "small water utility / light industry" class; a real 62443-3-2 risk assessment with the asset owner decides them (12). Security level definitions (SL 1 to 4 against increasingly capable attackers) are described at a high level; exact wording UNVERIFIED.

## 6. Outbound-only cloud sync and remote access

### Cloud sync (multi-site)
- A `cloud-sync` component (separate container, service account `svc-cloud-sync`, role `viewer`) reads from the local API or the internal bus and pushes to central over **outbound** TLS (HTTPS batch upload or MQTT over TLS to central's broker), authenticated with the site's client certificate (03 section 7).
- No listening sockets. No reverse tunnels that expose site services. No inbound configuration path. Central cannot request data except by what the site chooses to push; "pull" requests from central are not supported in v1.
- Local buffering on outage; replay with CloudEvents `id` de-duplication.
- Data minimisation: `sync.include_prefixes: [site1/pumphouse]`, `sync.events: [skyranger.alarm., skyranger.tag.counter_discontinuity.]`. Audit records are not synced by default (they may contain usernames and comments).

### Remote access (integrator support)
- Disabled by default. A site `admin` enables it for a bounded window (`skyrangerctl remote enable --for 4h` or HMI toggle), which audits `remote_access.enabled` and starts a WireGuard tunnel **outbound** to the integrator's jump host.
- Over the tunnel only `8443` (HMI/API) and optionally `22` are reachable. The tunnel peer is the jump host, not the internet.
- Remote humans authenticate to the jump host with MFA, then to Sky Ranger with their own account; Sky Ranger requires TOTP for sessions arriving from the tunnel interface (02 section 5).
- Window expiry tears the tunnel down; the egress rule is removed from `egress_allow`.
- TeamViewer, AnyDesk, and similar always-on tools are not installed on the edge box and the release checklist (14) checks for them.

## 7. Multi-site network

Each site uses the small-site pattern. Central sits in a cloud VPC or HQ data centre, exposes one TLS endpoint for sync (and optionally a broker), and holds the central CA. There is no site-to-site connectivity. A site's egress allow list contains only central's sync endpoint and, if used, the update registry and the jump host.

## Decisions made
1. The edge box sits at Purdue L2 (merged L2/L3 for small sites); field devices at L1 in separate wired and Wi-Fi VLANs with no internet route.
2. Default-deny firewall between all zones; nothing inbound from the internet, ever; no port forwards.
3. The edge box runs its own nftables input **and output** policy regardless of the site firewall.
4. Field-device egress rules are generated from `Connection` config through a narrow privileged helper.
5. Cloud sync and remote access are outbound-initiated only; remote access is time-boxed, admin-enabled, MFA-protected, audited.
6. Office access to the HMI goes through a source-restricted rule, preferably via a DMZ reverse proxy.

## Alternatives rejected and why
- **Flat network with "the gateway's auth is enough"**: Modbus and many PLCs have no auth; segmentation is the only control for them.
- **Port-forwarding 8443 for remote support**: exposes the HMI to every internet scanner (T1); historically a top cause of ICS incidents.
- **Always-on VPN or commercial remote desktop tools**: standing access nobody remembers to revoke.
- **Data diode hardware for cloud sync**: strongest one-way guarantee but costly and overkill for the target customers; software one-way plus outbound-only is proportionate. Offer as an option for critical sites.
- **Edge box as the only firewall**: acceptable as a minimum, but a compromised box then also controls the network boundary.

## Open questions
- Docker vs Podman interaction with host nftables must be settled by the deployment doc and verified with an external scan.
- Should rule 3 (field Wi-Fi to 8443) be replaced by a separate enrollment/OTA listener on its own port so the full API is not reachable from the field VLAN? Recommended; needs a deployment decision.
- Purdue level definitions and 62443-3-2 zone/conduit terminology and SL definitions are described from memory: UNVERIFIED against the published texts.
