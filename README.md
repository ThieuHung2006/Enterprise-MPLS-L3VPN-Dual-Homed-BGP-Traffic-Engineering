# MPLS L3VPN & Dual-Homed BGP Traffic Engineering Lab

A service-provider lab built around a small ISP backbone (**AS 65000**). It covers two topics:

1. **Multi-tenant MPLS L3VPN** (RFC 4364) with a BGP-free core.
2. **Dual-homed eBGP to two upstream providers**, using Local Preference for outbound traffic and AS-Path prepending for inbound traffic.

> **Scope:** this is a learning/portfolio lab (4 provider routers, 2 customer routers, 2 simulated upstreams). It is not a production design. See [Known Limitations](#7-known-limitations--future-work).

---

## Table of Contents

1. [Project Overview](#1-project-overview)
2. [Topology & Addressing](#2-topology--addressing)
3. [Design & Architecture](#3-design--architecture)
4. [Packet Walk](#4-packet-walk-ce1_hanoi--ce2_hcm)
5. [BGP Traffic Engineering](#5-bgp-traffic-engineering)
6. [Verification](#6-verification)
7. [Known Limitations & Future Work](#7-known-limitations--future-work)

---

## 1. Project Overview

| Area | What is implemented |
| :--- | :--- |
| Underlay | OSPFv2 single area (Area 0) between provider routers; `/32` loopbacks used as router IDs and BGP/LDP endpoints |
| Transport | LDP (RFC 5036) builds LSPs between PE loopbacks |
| Customer VPNs | `VRF_CORP_A` and `VRF_CORP_B` on the PEs, each with its own RD and RT |
| VPN signaling | MP-iBGP VPNv4 between `PE1_HN` and `PE2_HCM` |
| PE-CE routing | eBGP (CE1 = AS 65001, CE2 = AS 65002) |
| BGP-free core | `P1_Core` runs only OSPF + LDP, with no BGP and no customer routes |
| Internet edge | `ASBR1_Core` has eBGP to ISP1 (AS 100) and ISP2 (AS 200) |
| Outbound policy | Local Preference 200 for routes learned from ISP1 |
| Inbound policy | AS-Path prepending on advertisements to ISP2 |

---

## 2. Topology & Addressing

![Network Topology](images/topology.png)

### 2.1 Devices

| Node | Role | Loopback0 | Interfaces | Protocols |
| :--- | :--- | :--- | :--- | :--- |
| **PE1_HN** | Provider Edge (Hanoi) | `10.0.0.1/32` | `e0/0` to P1; `e0/1` trunk to CE1 with subinterfaces `e0/1.10` (VRF_CORP_A) and `e0/1.20` (VRF_CORP_B) | OSPF, LDP, MP-iBGP VPNv4, VRF, eBGP |
| **P1_Core** | Provider (core) | `10.0.0.2/32` | `e0/0`, `e0/1`, `e0/2` | OSPF, LDP (no BGP) |
| **PE2_HCM** | Provider Edge (Ho Chi Minh) | `10.0.0.3/32` | `e0/0` to P1; `e0/1` trunk to CE2 with subinterfaces `e0/1.10` (VRF_CORP_A) and `e0/1.20` (VRF_CORP_B) | OSPF, LDP, MP-iBGP VPNv4, VRF, eBGP |
| **ASBR1_Core** | AS border router | `10.0.0.4/32` | `e0/0` to P1; `e0/1` to ISP1; `e0/2` to ISP2 | OSPF, LDP, eBGP |
| **CE1_Hanoi** | Customer edge (Hanoi) | - | `e0/0` trunk with subinterfaces `.10` (CORP_A) and `.20` (CORP_B) | eBGP, AS 65001 |
| **CE2_HCM** | Customer edge (Ho Chi Minh) | - | `e0/0` trunk with subinterfaces `.10` (CORP_A) and `.20` (CORP_B) | eBGP, AS 65002 |
| **ISP1_AS100** | Upstream (primary) | `8.8.8.8/32` | `e0/0` to ASBR1 | eBGP, AS 100 |
| **ISP2_AS200** | Upstream (backup) | `8.8.8.8/32` | `e0/0` to ASBR1 | eBGP, AS 200 |

> Both upstreams advertise `8.8.8.8/32` so the same destination is reachable through two different paths. This is a simulation shortcut.
> Private AS numbers (65000-65002) and documentation ranges (`203.0.113.0/24`) are used because this is a lab.

### 2.2 Customers

| Customer | VRF | RD | Export / Import RT | PE-CE VLAN | Notes |
| :--- | :--- | :--- | :--- | :--- | :--- |
| A | `VRF_CORP_A` | `65000:100` | `65000:100` | 10 | LANs `192.168.10.0/24` (Hanoi), `192.168.20.0/24` (HCM) |
| B | `VRF_CORP_B` | `65000:200` | `65000:200` | 20 | PE1-CE1 link `10.10.20.0/30`; LANs `192.168.100.0/24` (Hanoi), `192.168.200.0/24` (HCM) |

> **Lab simplification:** both customers share the same physical CE and PE-CE link, separated by 802.1Q VLANs. In a real deployment each customer would normally have its own CE.

---

## 3. Design & Architecture

### 3.1 VRF, RD and RT

- **VRF** gives each customer a separate routing table and CEF table on the PE.
- **Route Distinguisher (RD)** is an 8-byte value prepended to an IPv4 prefix to make a 12-byte **VPNv4 prefix** (for example `65000:100:192.168.10.0/24`). It makes prefixes unique inside BGP, so overlapping customer prefixes do not collide.
- **Route Target (RT)** is an extended community that controls which VRFs import a VPNv4 route. When `PE2_HCM` receives a VPNv4 route whose RT matches the import RT of a local VRF, it installs the IPv4 route into that VRF.

In short: **RD provides uniqueness, RT provides policy.**

### 3.2 Two-label stack

| Label | Position | Distributed by | Purpose |
| :--- | :--- | :--- | :--- |
| Transport (LDP) | Top | LDP | Gets the packet to the egress PE loopback across the core |
| VPN service | Bottom | MP-BGP | Tells the egress PE which VRF to use |

### 3.3 PHP and the BGP-free core

- `P1_Core` is the penultimate hop toward each PE loopback. The PE advertises **implicit-null (label 3)** via LDP for its own `/32`, so `P1_Core` **pops the transport label** before forwarding. The egress PE then only looks up the VPN label.
- `P1_Core` forwards by label only. It has no BGP configuration and holds no customer or Internet prefixes, so tenant route changes never reach the core.

---

## 4. Packet Walk: `CE1_Hanoi` → `CE2_HCM`

### 4.1 Control plane

1. OSPF gives reachability between all `/32` loopbacks. LDP binds a label to each one.
2. `CE2_HCM` advertises its prefix to `PE2_HCM` over eBGP.
3. `PE2_HCM` allocates a VPN label (example: `19`), sets the next hop to its loopback `10.0.0.3`, and sends the VPNv4 route to `PE1_HN` over MP-iBGP.
4. `PE1_HN` resolves next hop `10.0.0.3` recursively through the LDP label learned from `P1_Core` (example: `16`).

### 4.2 Data plane

```text
1. CE1 -> PE1 : [ IPv4 ]                          native IP into the VRF interface
2. PE1 -> P1  : [ LDP 16 ][ VPN 19 ][ IPv4 ]      PE1 pushes two labels
3. P1  -> PE2 : [ VPN 19 ][ IPv4 ]                P1 pops the top label (PHP)
4. PE2 -> CE2 : [ IPv4 ]                          PE2 pops the VPN label, forwards in the VRF
```

> Label values `16` and `19` are examples. The real values are in the verification screenshots.

- **PE1_HN (ingress):** does a VRF lookup, pushes the VPN label (bottom of stack, S=1) and then the LDP label (top).
- **P1_Core (transit):** swaps or pops using only the top label. Here it pops because of PHP.
- **PE2_HCM (egress):** uses the VPN label to select the VRF, pops it, and sends native IPv4 to the CE.

---

## 5. BGP Traffic Engineering

`ASBR1_Core` has eBGP sessions with both upstreams.

### 5.1 Outbound: Local Preference

Route-map `RM_FROM_ISP1_IN` sets **LOCAL_PREF 200** on routes learned from ISP1. Routes from ISP2 keep the default of 100. LOCAL_PREF is carried across iBGP inside AS 65000, so the highest value wins and outbound traffic prefers ISP1. ISP2 becomes the backup path.

### 5.2 Inbound: AS-Path prepending

Route-map `RM_TO_ISP2_OUT` prepends AS 65000 three times when advertising `203.0.113.128/25` to ISP2. Remote networks then see:

- via ISP1: `65000`
- via ISP2: `65000 65000 65000 65000` (the original plus 3 prepends)

All else being equal, remote networks prefer the shorter path through ISP1.

> Prepending only **influences** inbound traffic. Each remote AS applies its own policy, such as its own Local Preference, so it is not a guarantee.

---

## 6. Verification

### 6.1 Underlay: OSPF and LDP

![P1 OSPF Neighbor](images/p1-core-ospf-neighbor.png)
![P1 LDP Neighbor](images/p1-core-mpls-ldp-neighbor.png)

- `show ip ospf neighbor` on `P1_Core` shows all neighbors in `FULL` state.
- `show mpls ldp neighbor` shows all peers in `Oper` state over TCP 646, with peer IDs on loopback addresses (`mpls ldp router-id Loopback0 force`). The `:0` suffix means a per-platform label space.

### 6.2 BGP-free core

![P1 BGP Free Core](images/p1-core-bgp-free-verification.png)
![P1 LFIB Table](images/p1-core-mpls-forwarding-table.png)

- `show ip route 192.168.20.0` on `P1_Core` returns `% Network not in table`.
- `show mpls forwarding-table` shows `Pop Label` toward the PE loopbacks, which confirms PHP.

### 6.3 End-to-end L3VPN (Customer A)

![PE1 VPNv4 and VRF Route](images/pe1-vpnv4-vrf-routing.png)
![CE1 to CE2 Ping and Traceroute](images/ce1-to-ce2-ping-traceroute.png)

- The MP-iBGP VPNv4 session between `10.0.0.1` and `10.0.0.3` is established.
- In `VRF_CORP_A`, `192.168.20.0/24` is learned via BGP with next hop `10.0.0.3`.
- Traceroute from CE1 to CE2 shows the MPLS label stack at the core hop, and the ping succeeds.

### 6.4 BGP traffic engineering

![ASBR1 Outbound LocalPref](images/asbr1-bgp-local-preference.png)
![ISP2 Inbound Prepending](images/isp2-bgp-as-path-prepending.png)

- `show ip bgp 8.8.8.8` on `ASBR1_Core`: the path via ISP1 (`203.0.113.2`) has LocPrf 200 and is marked best (`*>`). The path via ISP2 (`203.0.113.6`) has LocPrf 100.
- `show ip bgp 203.0.113.128` on `ISP2_AS200` shows the prepended path `65000 65000 65000 65000`.

### 6.5 Second customer (`VRF_CORP_B`) over the same PE-CE link

![PE1 VRF list](images/pe1-show-ip-vrf.png)
![PE1 VRF_CORP_B routing table](images/pe1-vrf-corp-b-route.png)

**What is verified:**

- `show ip vrf` on `PE1_HN`: `VRF_CORP_A` (RD `65000:100`) is bound to `Et0/1` and `Et0/1.10`; `VRF_CORP_B` (RD `65000:200`) is bound to `Et0/1.20`.
- `show ip route vrf VRF_CORP_B` on `PE1_HN`:
  - `10.10.20.0/30` is connected on `Ethernet0/1.20` (with local `/32`).
  - `192.168.100.0/24` is learned from `CE1` via eBGP (AD 20, next hop `10.10.20.2`).
  - `192.168.200.0/24` and `10.10.30.0/30` are learned from `PE2_HCM` via MP-iBGP (AD 200, next hop `10.0.0.3`).
  - `192.168.10.0/24` is also present via eBGP from `10.10.20.2` and `192.168.20.0/24` via iBGP from `10.0.0.3`. These are Customer A's LANs (see the note below).

**Still to verify before claiming full isolation:**

- [ ] `show ip vrf detail VRF_CORP_B` shows export and import RT `65000:200`
- [ ] `show ip route vrf VRF_CORP_A` does **not** contain `192.168.100.0/24` or `192.168.200.0/24`
- [ ] `ping vrf VRF_CORP_B 192.168.200.1` succeeds from `PE1_HN`
- [ ] `ping vrf VRF_CORP_A 192.168.200.1` **fails** (no route)
- [ ] `show ip bgp vpnv4 vrf VRF_CORP_B summary` shows the CE neighbor `10.10.20.2` as Established

> **Note:** Customer A's LANs (`192.168.10.0/24`, `192.168.20.0/24`) also appear in `VRF_CORP_B`. Because both VRFs connect to the same CE, the CE is most likely advertising the same LANs on both eBGP sessions. This is not true overlapping-IP separation. To fix it, apply an outbound prefix-list on the CE so each session only advertises that customer's LANs, or use separate CEs/VRF-lite. Overlapping-address testing (the same prefix in both VRFs but in different sites) is still to be done.

---

## 7. Known Limitations & Future Work

**Redundancy**
- Single P router and single PE per site, so there is no core redundancy and no link-failure test.
- ISP1-to-ISP2 failover is configured but a shutdown/recovery test with loss and timing measurements is still to be done.

**Security and filtering on the ASBR**
- No prefix-lists (bogons, RFC 1918, own-prefix only), no `maximum-prefix`, no BGP/LDP authentication.
- No explicit policy preventing AS 65000 from becoming a transit between ISP1 and ISP2 (outbound filtering to allow only own prefixes).

**Convergence and operations**
- No BFD or timer tuning, so no sub-second convergence claim is made. Measured convergence times are still to be added.
- No monitoring (syslog, SNMP, NetFlow) and no automated config management.

**Scalability and features**
- No Route Reflector (full-mesh iBGP is fine for two PEs only).
- Not covered: MPLS TE, QoS, IPv6/6PE, Inter-AS VPN, hub-and-spoke or extranet RT designs.

**Suggested next steps**
1. Complete the Customer B verification checklist in section 6.5.
2. Add a prefix-list on the CE (or separate CEs) and a true overlapping-IP test.
3. Add an ISP1 failure and recovery test with ping loss and timing results.
4. Add ASBR prefix filters and BGP authentication.
5. Add a second P router for core redundancy.
6. Publish device configs (`configs/` folder) so the lab can be reproduced.
