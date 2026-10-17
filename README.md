# MPLS L3VPN & Dual-Homed BGP Traffic Engineering Lab

A service-provider lab built around a small ISP backbone (**AS 65000**), running on Cisco IOL in EVE-NG. It demonstrates two advanced architectures:

1. **Multi-tenant MPLS L3VPN (RFC 4364)** over a **BGP-free core**, featuring end-to-end tenant isolation using VRF-Lite on Customer Edge (CE) routers.
2. **Dual-homed eBGP edge traffic engineering** toward two upstream providers, using `LOCAL_PREF` for outbound traffic and `AS_PATH` prepending to influence inbound traffic.

---

## Table of Contents

1. [Project Overview](#1-project-overview)
2. [Topology & Addressing](#2-topology--addressing)
3. [Design & Architecture](#3-design--architecture)
4. [Packet Walk](#4-packet-walk-ce1_hanoi--ce2_hcm)
5. [BGP Traffic Engineering](#5-bgp-traffic-engineering)
6. [Verification & Proof of Concept](#6-verification--proof-of-concept)
7. [Future Enhancements](#7-future-enhancements)

---

## 1. Project Overview

| Domain | Implementation |
| :--- | :--- |
| **IGP underlay** | Single-area OSPFv2 (Area 0) on provider links; `/32` Loopback0 interfaces are router IDs and transport endpoints |
| **MPLS transport** | LDP (RFC 5036) binds labels to the `/32` PE loopbacks with Session Protection and IGP Synchronization enabled |
| **Tenant isolation** | `VRF_CORP_A` and `VRF_CORP_B` configured on both PEs and CEs (VRF-Lite) ensuring strict control-plane separation |
| **VPN signaling** | MP-iBGP VPNv4 between `PE1_HN` (`10.0.0.1`) and `PE2_HCM` (`10.0.0.3`) |
| **PE-CE routing** | eBGP over 802.1Q subinterfaces with dedicated Address Families per VRF |
| **BGP-free core** | `P1_Core` runs only OSPF and LDP, with no BGP sessions and no tenant or Internet prefixes |
| **Upstream peering** | `ASBR1_Core` has eBGP to `ISP1_AS100` (AS 100) and `ISP2_AS200` (AS 200) |
| **Traffic Engineering**| Inbound route-map sets `LOCAL_PREF 200` from ISP1 (Primary); Outbound route-map prepends AS 65000 x3 toward ISP2 (Backup) |

---

## 2. Topology & Addressing

![Network Topology](images/topology.png)

### 2.1 Device Inventory

| Node | Role | Loopback0 | Interfaces | Protocols |
| :--- | :--- | :--- | :--- | :--- |
| **PE1_HN** | Provider Edge (Hanoi) | `10.0.0.1/32` | `e0/0` to P1; `e0/1` trunk | OSPF, LDP, MP-iBGP VPNv4, VRF, eBGP |
| **P1_Core** | Provider (P router) | `10.0.0.2/32` | `e0/0` to PE1, `e0/1` to PE2, `e0/2` to ASBR1 | OSPF, LDP (no BGP) |
| **PE2_HCM** | Provider Edge (HCM) | `10.0.0.3/32` | `e0/0` to P1; `e0/1` trunk | OSPF, LDP, MP-iBGP VPNv4, VRF, eBGP |
| **ASBR1_Core** | AS border router | `10.0.0.4/32` | `e0/0` to P1, `e0/1` to ISP1, `e0/2` to ISP2 | OSPF, LDP, eBGP |
| **CE1_Hanoi** | Customer Edge (Hanoi) | - | `e0/0` trunk (`.10` = CORP_A, `.20` = CORP_B) | eBGP, VRF-Lite, 802.1Q |
| **CE2_HCM** | Customer Edge (HCM) | - | `e0/0` trunk (`.10` = CORP_A, `.20` = CORP_B) | eBGP, VRF-Lite, 802.1Q |

### 2.2 Tenants

| Tenant | VRF | RD | Export / Import RT | VLAN | Addressing |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **Customer A** | `CORP_A` | `65000:100` | `65000:100` | 10 | LANs `192.168.10.0/24` (HN), `192.168.20.0/24` (HCM) |
| **Customer B** | `CORP_B` | `65000:200` | `65000:200` | 20 | LANs `192.168.100.0/24` (HN), `192.168.200.0/24` (HCM) |

---

## 3. Design & Architecture

### 3.1 End-to-End VRF Isolation
- **PE Routers:** Utilize standard VRFs with Route Distinguishers (RD) to keep overlapping prefixes unique in BGP, and Route Targets (RT) to control VPNv4 route import/export policies.
- **CE Routers (VRF-Lite):** To prevent route leaking across a shared global routing table at the customer edge, **VRF-Lite** is implemented on `CE1_Hanoi` and `CE2_HCM`. Each tenant's interfaces and eBGP peering sessions are strictly bound to their respective VRFs.

### 3.2 Two-Label Stack & PHP
| Label | Position | Distributed by | Purpose |
| :--- | :--- | :--- | :--- |
| **Transport** | Outer (`S=0`) | LDP | Carries the frame across the core to the egress PE |
| **VPN service** | Inner (`S=1`) | MP-iBGP | Selects the destination VRF on the egress PE |

**PHP (Penultimate Hop Popping):** `P1_Core` pops the outer transport label before delivering the packet to the egress PE, saving the PE from performing a double label lookup.

---

## 4. Packet Walk: `CE1_Hanoi` → `CE2_HCM`

1. **CE1 -> PE1:** Native IPv4 packet enters `PE1_HN` via VRF `CORP_A`.
2. **PE1 -> P1:** `PE1_HN` imposes two labels: VPN Label (Inner) and LDP Transport Label (Outer).
3. **P1 -> PE2:** `P1_Core` swaps the transport label and eventually pops it (PHP) before forwarding to `PE2_HCM`.
4. **PE2 -> CE2:** `PE2_HCM` reads the VPN label, identifies VRF `CORP_A`, pops the label, and forwards the native IPv4 packet to `CE2_HCM`.

---

## 5. BGP Traffic Engineering

`ASBR1_Core` handles eBGP sessions with `ISP1_AS100` (primary) and `ISP2_AS200` (backup).
- **Outbound (LOCAL_PREF):** Route-map `RM_FROM_ISP1_IN` sets `LOCAL_PREF 200` for routes received from ISP1, making it the preferred exit path for the entire AS 65000.
- **Inbound (AS_PATH Prepending):** Route-map `RM_TO_ISP2_OUT` prepends AS 65000 three times (`65000 65000 65000 65000 i`) when advertising prefixes to ISP2, influencing external networks to route return traffic through ISP1.

---

## 6. Verification & Proof of Concept

### 6.1 BGP-Free Core & Label Switching
<p align="center">
  <img src="images/p1-core-no-customer-routes.png" alt="P1 BGP Free Core">
  <br>
  <i><b>Figure 1:</b> <code>show ip route 192.168.20.0</code> on P1_Core returns <code>% Network not in table</code>: tenant prefixes are completely absent from the core.</i>
</p>

### 6.2 Data Plane: Two-Label Stack and PHP
<p align="center">
  <img src="images/ce1-l3vpn-traceroute-label-stack.png" alt="CE1 to CE2 Ping and Traceroute">
  <br>
  <i><b>Figure 2:</b> Traceroute from CE1 to CE2. Hop 2 (P1_Core) shows the 2-label stack <code>[Labels 20/16]</code>. Hop 3 shows a single label <code>[Label 16]</code>, proving PHP action.</i>
</p>

### 6.3 End-to-End Strict Tenant Isolation (VRF-Lite Verification)
Implementing VRF-Lite on the CE routers eliminated all route leaking.

**Control Plane Validation (PE1_HN):**
* `show ip route vrf VRF_CORP_A` only contains `.10.0/24` and `.20.0/24` subnets.
* `show ip route vrf VRF_CORP_B` only contains `.100.0/24` and `.200.0/24` subnets.

**Data Plane Validation (Cross-Tenant Ping):**
```text
CE1_Hanoi# ping vrf CORP_A 192.168.200.1
% Network not in table
