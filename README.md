# Enterprise Network Design — Cisco Packet Tracer

> A multi-site enterprise network simulating a real-world corporate environment, with 12 IOS devices across edge, core, distribution, and access layers, plus a centralized server farm and a hardened DMZ behind a Cisco ASA 5506-X firewall.

![Topology](screenshots/01-topology.png)

This is the second iteration in my Cisco lab series. The first focused on OSPFv2 multi-area routing; this one is deliberately built around **EIGRP**, **HSRP**, and **three-zone firewall security** to expand hands-on coverage of enterprise network technologies. Every design choice in this repo is rationalized below, every configuration is committed, and every verification step is reproducible from the saved `.pkt` topology file.

---

## Table of Contents

- [Overview](#overview)
- [Architecture](#architecture)
- [IP Addressing & VLAN Design](#ip-addressing--vlan-design)
- [Layer 2 Infrastructure](#layer-2-infrastructure)
- [Layer 3 Routing](#layer-3-routing)
- [Security Architecture](#security-architecture)
- [Centralized Services](#centralized-services)
- [Management Plane](#management-plane)
- [Device Inventory](#device-inventory)
- [Configuration Highlights](#configuration-highlights)
- [Verification & Testing](#verification--testing)
- [Repository Structure](#repository-structure)
- [Lessons Learned](#lessons-learned)
- [Next Phase](#next-phase)

---

## Overview

This project simulates a real-world corporate network with multiple offices, centralized services, and a publicly accessible DMZ — all behind a stateful firewall. The objective was twofold:

1. **Practical depth in EIGRP and HSRP** — protocols I'd studied but not yet implemented end-to-end. Building a fully working topology around them forced understanding beyond textbook level.
2. **Documentation discipline** — every design decision is rationalized in this README, every configuration is committed to `configs/`, and every verification command is reproducible from the `.pkt` file in `topology/`.

The result is a clearly tiered architecture spanning 14 devices (12 IOS + ASA + OUT) that I can hand to a reviewer for line-by-line inspection.

---

## Architecture

The network follows a hierarchical three-layer design with a collapsed core at the distribution layer:



Internet
                                │
                        ┌───────┴───────┐
                        │      OUT      │   WAN edge router
                        └───────┬───────┘
                                │
                        ┌───────┴───────┐
                        │  ASA 5506-X   │   Firewall (inside / outside / DMZ)
                        └───┬───────┬───┘
                            │       │
                        ┌───┴──┐    └────────┐
                        │  R0  │             │
                        └─┬──┬─┘          DMZSW
                          │  │           │     │
                    ┌─────┘  └─────┐    WEB  EMAIL
                    │              │
                  ┌─┴──┐         ┌─┴──┐
                  │MLS1├─────────┤MLS2│   Collapsed core / L3 boundary
                  └─┬──┘         └─┬──┘
                    │              │
              ┌─────┘              └─────┐
              │                          │
            ┌─┴──┐                    ┌──┴─┐
            │DIST1│                   │DIST2│
            └┬─┬─┘                    └┬─┬─┘
             │ │                       │ │
          ACC1 ACC2                  ACC3 ACC4






**Design rationale:**

- **WAN edge separation.** The OUT router terminates the simulated WAN before the firewall, mirroring real-world ISP-handoff designs and keeping public routing concerns separate from internal policy enforcement.
- **Collapsed core.** Rather than a dedicated core/distribution split (overkill for four offices), MLS1 and MLS2 serve as both L3 boundary and aggregation. They're connected via a 2-port LACP EtherChannel so either can fail without partitioning the network.
- **Symmetric branches.** Office 1/2 are anchored to MLS1, Office 3/4 to MLS2, with HSRP active/standby roles balanced across the two switches so neither sits idle as pure backup.
- **Hardened service zones.** The server farm (VLAN 60) and the DMZ (VLAN 80) live behind separate firewall interfaces, with explicit ACL policy controlling traffic between them and the inside network.

---

## IP Addressing & VLAN Design

All internal addressing uses **10.0.0.0/8** private space, subdivided for clarity:

| VLAN | Subnet | Purpose | HSRP Gateway | Active Switch |
|------|--------|---------|--------------|---------------|
| 10 | 10.0.10.0/24 | Office 1 (east) | 10.0.10.1 | MLS1 |
| 20 | 10.0.20.0/24 | Office 2 (east) | 10.0.20.1 | MLS2 |
| 30 | 10.0.30.0/24 | Office 3 (west) | 10.0.30.1 | MLS2 |
| 40 | 10.0.40.0/24 | Office 4 (west) | 10.0.40.1 | MLS2 |
| 60 | 10.0.60.0/24 | Server farm | 10.0.60.1 | MLS1 |
| 80 | 10.0.80.0/24 | DMZ | ASA 10.0.80.1 | — |
| 90 | 10.0.90.0/24 | Out-of-band management | 10.0.90.1 | MLS1 |
| 99 | — | Native VLAN (no IP, no users) | — | — |

**External addressing:**

| Subnet | Purpose |
|--------|---------|
| 200.1.1.0/30 | ASA outside ↔ OUT transit |
| 200.1.2.0/24 | DMZ public address pool (NAT mapped from VLAN 80) |
| 198.51.100.0/24 | Outside zone (remote IT, external workstations) |
| 203.0.113.0/24 | VPN endpoint addressing |

**Transit subnets (point-to-point /30):**

| Subnet | Endpoints |
|--------|-----------|
| 10.0.0.0/30 | R0 ↔ ASA inside |
| 10.0.255.0/30 | R0 ↔ MLS1 |
| 10.0.255.4/30 | R0 ↔ MLS2 |

**Loopback router-IDs:**

| Device | Loopback IP |
|--------|-------------|
| R0 | 10.0.255.100/32 |
| MLS1 | 10.0.255.101/32 |
| MLS2 | 10.0.255.102/32 |

---

## Layer 2 Infrastructure

### Trunking

All inter-switch links carry 802.1Q trunks with **VLAN 99 as the native VLAN**. VLAN 99 has no SVI, no users, and no DHCP — it's deliberately unused. Setting it as native on every trunk mitigates VLAN-hopping attacks where an attacker on a default-VLAN access port could send double-tagged frames into a sensitive VLAN.

### Spanning Tree

The network runs **Rapid PVST+** (Cisco's per-VLAN RSTP). To avoid relying on automatic root election (which can yield suboptimal topologies), root bridge placement is deterministic:

| Device | Priority | Role |
|--------|----------|------|
| MLS1 | 0 | Primary root for all VLANs |
| MLS2 | 4096 | Secondary root |
| DIST1, DIST2 | 16384 | Never root |
| ACC1–ACC4, SFSW, DMZSW, RITSW | 32768 (default) | Edge |

All access ports have **PortFast + BPDU Guard** enabled, so end-user devices skip listening/learning states and any rogue switch plugging into a user port immediately puts the port into `err-disabled`.

### EtherChannel

Three LACP EtherChannels carry inter-switch traffic:

| Port-Channel | Endpoints | Members | Mode | Purpose |
|--------------|-----------|---------|------|---------|
| PO1 | MLS1 ↔ MLS2 | Fa0/21–22 ↔ Fa0/20–21 | LACP active | Inter-MLS spine |
| PO10 / PO3 | MLS1 ↔ DIST1 | Fa0/20, 23, 24 ↔ Fa0/22, 23, 24 | LACP active | East distribution uplink |
| PO20 / PO3 | MLS2 ↔ DIST2 | Fa0/22, 23, 24 ↔ Fa0/22, 23, 24 | LACP active | West distribution uplink |

PO10 and PO1 use **asymmetric port mapping** — local port numbers on each side don't match. LACP doesn't require symmetric port numbering; what matters is that each side's `channel-group` membership is internally consistent. Useful in production where physical port allocation is constrained.

---

## Layer 3 Routing

### EIGRP

EIGRP **AS 100** runs across R0, MLS1, and MLS2:








**Key design choices:**

- `passive-interface default` followed by selective `no passive-interface` on transit links only — EIGRP hellos never leak onto user-facing SVIs or trunks. Less unnecessary traffic, smaller attack surface.
- Loopback router-IDs for stability across reboots.
- `network 10.0.0.0 0.255.255.255` — broadest possible match; specific subnets are advertised based on the passive/active logic above.
- Default route originated on R0 (pointing to ASA inside) is redistributed into EIGRP with an explicit metric so MLS1 and MLS2 learn it as an external EIGRP route.

### HSRP v2

Every user-facing VLAN has HSRP v2 with two physical gateways and an optional third backup on R0. Load is balanced across MLS1 and MLS2:

| VLAN | Active | Standby | Backup (R0) |
|------|--------|---------|-------------|
| 10 | MLS1 (110, preempt) | MLS2 (90) | 90 |
| 20 | MLS2 (110, preempt) | MLS1 (90) | 110 (preempt) |
| 30 | MLS2 (110, preempt) | — | 90 |
| 40 | MLS2 (110, preempt) | — | 110 (preempt) |
| 60 | MLS1 (110, preempt) | MLS2 (90) | — |
| 90 | MLS1 (110, preempt) | MLS2 (90) | — |

**Preempt** is enabled on the designated active so that after a failure and recovery, the role returns to the intended switch rather than getting stuck on the backup.

### DHCP Relay

Centralized DHCP runs on `10.0.60.10`. Every SVI on MLS1 and MLS2 has `ip helper-address 10.0.60.10`, which converts broadcast DHCP DISCOVER messages from clients into unicast packets to the server. This pattern means:

- No DHCP daemons on distribution switches
- One place to manage scopes, reservations, and option codes
- Adding a new VLAN is one new `ip helper-address` line, not a new DHCP server

---

## Security Architecture

### ASA 5506-X Zones

Three security levels enforce a strict perimeter:

| Interface | Zone | Security Level | Subnet |
|-----------|------|----------------|--------|
| Gi1/1 | inside | 100 | 10.0.0.0/30 (to R0) |
| Gi1/2 | outside | 0 | 200.1.1.0/30 (to OUT) |
| Gi1/4 | dmz | 50 | 10.0.80.0/24 |

ASA defaults permit traffic from a higher security level to a lower one and deny the reverse. Explicit ACLs override these defaults for required inbound services.

### NAT

This is a textbook hardened-DMZ posture: if a DMZ server is ever compromised, the attacker cannot pivot freely into the internal network.

### IPsec VPN

The ASA is configured for remote-access IPsec VPN with a dedicated tunnel group, group policy, address pool, and crypto map. ISAKMP policy, IKE Phase 2 transform set, and crypto map ipsec-isakmp are applied per Cisco standards. Full tunnel-encryption validation is scheduled for the Phase 2 migration where a complete crypto stack is available.

---

## Centralized Services

The server farm (VLAN 60) hosts six dedicated server roles. Centralization means each service has a single configuration point and a clear ownership boundary:

| Service | IP | Role |
|---------|-----|------|
| NTP / DHCP | 10.0.60.10 | Time sync + DHCP scopes for VLANs 10/20/30/40 |
| DNS | 10.0.60.11 | Forward zone `giorgi.mtch` with A records for every device |
| Syslog | 10.0.60.13 | Centralized log destination (`logging 10.0.60.13` on every device) |
| FTP | 10.0.60.14 | Automated config backups; user: `backup` / `ftp123` |
| Internal Web | 10.0.60.15 | Intranet service (VLAN 60 only) |
| DMZ Web | 10.0.80.10 → 200.1.2.10 | Public website |
| DMZ Email | 10.0.80.11 → 200.1.2.11 | Public SMTP/POP3 |

### Automated FTP Config Backups

Each IOS device has FTP credentials stored locally:

All 12 device backups live on the FTP server and are committed to `configs/` in this repo. This is a production-grade operational practice: if a device fails or a configuration change introduces an outage, the last known-good config is one `copy ftp: running-config` away.

---

## Management Plane

Out-of-band management uses VLAN 90 (`10.0.90.0/24`). Every L2 switch has a `Vlan90` SVI; every L3 device has a `Loopback0` used both for routing identity and stable management.

| Practice | Implementation |
|----------|----------------|
| Authentication | Local: `username admin secret <type-5 hash>` |
| Privileged mode | Separate `enable secret class123` (type-5 hash) |
| Line passwords | `service password-encryption` → type-7 obfuscation |
| Remote access | SSH v2 only, RSA 1024-bit keys, no Telnet anywhere |
| Inactivity | `exec-timeout 15 0` on console and VTY |
| Banners | Authorized-access-only banner on every device |
| Reachability | Mgmt VLAN reachable from every switch, loopbacks from every router |

Intent: a compromised single credential exposes only that device, not the fleet. A natural stepping-stone to centralized AAA (Phase 2).

---

## Device Inventory

| # | Hostname | Model | Role | Mgmt IP |
|---|----------|-------|------|---------|
| 1 | OUT | Cisco 2911 | WAN edge router | — |
| 2 | ASA | Cisco ASA 5506-X | Perimeter firewall | — |
| 3 | R0 | Cisco 7200 | Core router | 10.0.255.100 (Lo0) |
| 4 | MLS1 | Catalyst 3560-24PS | Distribution + L3 | 10.0.90.2 |
| 5 | MLS2 | Catalyst 3560-24PS | Distribution + L3 | 10.0.90.3 |
| 6 | DIST1 | Catalyst 2960-24TT | East access aggregation | 10.0.90.11 |
| 7 | DIST2 | Catalyst 2960-24TT | West access aggregation | 10.0.90.12 |
| 8 | ACC1 | Catalyst 2960-24TT | Office 1 access | 10.0.90.21 |
| 9 | ACC2 | Catalyst 2960-24TT | Office 2 access | 10.0.90.22 |
| 10 | ACC3 | Catalyst 2960-24TT | Office 3 access | 10.0.90.23 |
| 11 | ACC4 | Catalyst 2960-24TT | Office 4 access | 10.0.90.24 |
| 12 | SFSW | Catalyst 2960-24TT | Server farm access | 10.0.90.31 |
| 13 | DMZSW | Catalyst 2960-24TT | DMZ access | 10.0.80.5 |
| 14 | RITSW | Catalyst 2960-24TT | Outside-zone access | 198.51.100.5 |

---

## Configuration Highlights

Full per-device configurations live in [`configs/`](configs/). A few representative snippets:

### MLS1 — Active HSRP Gateway for VLAN 10


---

## Lessons Learned

**1. Hierarchical design pays off in troubleshooting.** When EIGRP didn't converge at one point during this build, the strict layer separation meant I knew exactly which two interfaces to investigate. Compare this to a flat L2/L3 mix where the same symptom could be any of a dozen devices.

**2. Passive-interface discipline matters.** Initially I had EIGRP hellos leaking onto every SVI. Forcing the protocol to be active only on transit links reduces unnecessary traffic and tightens the attack surface — an attacker on a user VLAN cannot inject EIGRP updates if the gateway is passive there.

**3. Symmetric load balancing requires explicit priority assignment.** HSRP defaults to whoever booted first as active, which means after a power cycle you can end up with both active roles on the same switch. Setting `priority 110 preempt` on one switch and `priority 90` on the other per VLAN, alternated, gives reliable load distribution.

**4. EtherChannel port mapping doesn't need symmetry.** When physical port allocation forced asymmetric mappings (PO10 uses Fa0/20-23-24 on MLS1 but Fa0/22-23-24 on DIST1), LACP came up without complaint. Worth remembering when cabling reality doesn't match the textbook.

**5. AAA fallback design needs the same care as the primary path.** Early in the project I deployed TACACS+ centrally, but a fallback mismatch caused multiple device lockouts. The final design uses local-auth only; TACACS deployment is reserved for the Phase 2 migration after verifying local-auth break-glass on every device.

**6. Centralized services compound returns.** One DHCP server, one DNS server, one syslog destination. Every router and switch points back to them. Adding a new office is plug-and-go because the services already exist. This is what operational simplicity looks like at small scale.

---

## Next Phase

Phase 2 moves the design into GNS3 / EVE-NG with real IOS-XE and ASA images and extends the scope:

- **GNS3 migration** with full crypto-stack IPsec VPN validation
- **802.1X port-based authentication** on user access ports with a RADIUS backend
- **SNMPv3** monitoring via a centralized NMS (LibreNMS or PRTG)
- **TACACS+ centralized authentication** with verified local-auth fallback
- **HSRP → VRRP migration** for multi-vendor neutrality
- **Redundant ASA failover pair** for security-layer resilience
- **MPLS-TE** between core devices, introducing service-provider concepts

---

**Built by Giorgi** — Junior Network Engineer, Tbilisi
**Reach out:** [LinkedIn](https://linkedin.com/in/your-handle) | [Email](mailto:you@example.com)
**Status:** Phase 1 complete · Phase 2 in planning

If you found this useful or have feedback on any design choice, open an issue or a PR — happy to discuss.

