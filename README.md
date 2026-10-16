# MPLS L3VPN & Dual-Homed BGP Traffic Engineering Lab

A service-provider network engineering lab built around an enterprise ISP backbone (**AS 65000**). This project demonstrates two core architectures:

1. **Multi-Tenant MPLS L3VPN (RFC 4364)** operating over a **BGP-Free Core**[cite: 2].
2. **Dual-Homed eBGP Edge Traffic Engineering** across two external upstream providers, leveraging deterministic `LOCAL_PREF` manipulation for outbound traffic and `AS_PATH` prepending for inbound return traffic.

> **Architectural Scope:** This repository documents a proof-of-concept portfolio topology (4 provider routers, 2 customer edge routers, and 2 simulated Tier-1 upstreams) running on Cisco IOL in EVE-NG. For production readiness gaps and planned enhancements, refer to [Known Limitations & Future Work](#7-known-limitations--future-work).

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

| Architectural Domain | Implementation Details |
| :--- | :--- |
| **IGP Underlay** | Single-area OSPFv2 (`Area 0`) across provider transit links; `/32` Loopback0 interfaces serve as Router IDs and transport endpoints |
| **MPLS Transport (LSP)** | Label Distribution Protocol (**LDP**, RFC 5036) dynamically binds transport labels to `/32` PE Loopback FECs |
| **Multi-Tenant Isolation** | `VRF_CORP_A` and `VRF_CORP_B` instantiated on edge routers (`PE1_HN`, `PE2_HCM`) with dedicated RDs and RTs[cite: 1, 2] |
| **Overlay VPN Signaling** | Multiprotocol iBGP (**MP-iBGP VPNv4**, RFC 4364) peering between `PE1_HN` (`10.0.0.1`) and `PE2_HCM` (`10.0.0.3`)[cite: 1] |
| **PE-CE Routing** | External BGP (**eBGP**) over 802.1Q subinterfaces (`CE1_Hanoi` = AS 65001, `CE2_HCM` = AS 65002)[cite: 1, 2] |
| **BGP-Free Core** | Transit router `P1_Core` runs OSPF and LDP exclusively, maintaining zero BGP sessions and zero tenant/external prefixes in its RIB/FIB |
| **Upstream Edge Peering** | Border router `ASBR1_Core` maintains dual-homed eBGP sessions to `ISP1_AS100` (AS 100) and `ISP2_AS200` (AS 200) |
| **Outbound Traffic Engineering** | Inbound route-map assigns `LOCAL_PREF 200` to prefixes learned from `ISP1_AS100` (Primary Egress) |
| **Inbound Traffic Engineering** | Outbound route-map applies `AS_PATH` prepending (`3x AS 65000`) toward `ISP2_AS200` (Standby Ingress) |

---

## 2. Topology & Addressing

![Network Topology](images/topology.png)

### 2.1 Device Inventory & Control-Plane Roles

| Node | Role | Loopback0 | Interfaces & Subinterfaces | Active Protocols |
| :--- | :--- | :--- | :--- | :--- |
| **PE1_HN** | Provider Edge (Hanoi) | `10.0.0.1/32` | `e0/0` (to P1); `e0/1` 802.1Q trunk to CE1 (`e0/1.10` in `VRF_CORP_A`, `e0/1.20` in `VRF_CORP_B`)[cite: 1, 2] | OSPFv2, LDP, MP-iBGP VPNv4, VRF, eBGP[cite: 1, 2] |
| **P1_Core** | Provider Transit (P-Router) | `10.0.0.2/32` | `e0/0` (to PE1), `e0/1` (to PE2), `e0/2` (to ASBR1) | OSPFv2, LDP (BGP-Free Core) |
| **PE2_HCM** | Provider Edge (Ho Chi Minh) | `10.0.0.3/32` | `e0/0` (to P1); `e0/1` 802.1Q trunk to CE2 (`e0/1.10` in `VRF_CORP_A`, `e0/1.20` in `VRF_CORP_B`) | OSPFv2, LDP, MP-iBGP VPNv4, VRF, eBGP |
| **ASBR1_Core** | Autonomous System Border Router | `10.0.0.4/32` | `e0/0` (to P1), `e0/1` (to ISP1), `e0/2` (to ISP2) | OSPFv2, LDP, eBGP |
| **CE1_Hanoi** | Customer Edge (Hanoi) | - | `e0/0` 802.1Q trunk (`e0/0.10` for `CORP_A`, `e0/0.20` for `CORP_B`) | eBGP (AS 65001), 802.1Q |
| **CE2_HCM** | Customer Edge (Ho Chi Minh) | - | `e0/0` 802.1Q trunk (`e0/0.10` for `CORP_A`, `e0/0.20` for `CORP_B`) | eBGP (AS 65002), 802.1Q |
| **ISP1_AS100** | Primary Upstream Provider | `8.8.8.8/32` | `e0/0` (to ASBR1) | eBGP (AS 100) |
| **ISP2_AS200** | Secondary Upstream Provider | `8.8.8.8/32` | `e0/0` (to ASBR1) | eBGP (AS 200) |

> **Simulation Notes:**
> - Both upstream routers originate `8.8.8.8/32` to emulate multi-path reachability to an identical global destination across distinct Autonomous Systems.
> - Private ASN ranges (`65000–65002`, RFC 6996) and documentation CIDR blocks (`203.0.113.0/24`, RFC 5737) are utilized in accordance with lab standards.

### 2.2 Multi-Tenant VRF Provisioning

| Tenant | VRF Instance | Route Distinguisher (RD) | Export / Import RT | Access VLAN | Subnet Allocation |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **Customer A** | `VRF_CORP_A`[cite: 2] | `65000:100`[cite: 2] | `65000:100` | VLAN 10 (`e0/1.10`)[cite: 2] | LANs: `192.168.10.0/24` (HN), `192.168.20.0/24` (HCM)[cite: 1] |
| **Customer B** | `VRF_CORP_B`[cite: 1, 2] | `65000:200`[cite: 2] | `65000:200` | VLAN 20 (`e0/1.20`)[cite: 1, 2] | WAN: `10.10.20.0/30` (HN), `10.10.30.0/30` (HCM); LANs: `192.168.100.0/24` (HN), `192.168.200.0/24` (HCM)[cite: 1] |

> **Resource Optimization Note:** To minimize memory footprint within the EVE-NG IOL environment, both tenants terminate on shared physical CE routers and PE-CE access links, logically separated at Layer 2 via **IEEE 802.1Q VLAN tagging**[cite: 1, 2]. In a production deployment, each enterprise customer deploys dedicated CPE hardware or VRF-Lite instances.

---

## 3. Design & Architecture

### 3.1 VRF, Route Distinguisher (RD), and Route Target (RT)

- **Virtual Routing and Forwarding (VRF):** Partitions a physical PE router into multiple independent virtual routers, each maintaining an isolated Routing Information Base (RIB) and Cisco Express Forwarding (CEF) FIB table[cite: 1, 2].
- **Route Distinguisher (RD):** An 8-byte (64-bit) identifier prepended to a 4-byte customer IPv4 prefix to construct a globally unique 12-byte (96-bit) **VPNv4 NLRI** (e.g., `65000:100:192.168.10.0/24`). The RD resolves prefix overlapping within the single MP-iBGP control plane.
- **Route Target (RT):** An 8-byte **MP-BGP Extended Community** attribute attached to VPNv4 updates that governs VRF import and export policies. When an egress PE receives a VPNv4 route whose attached RT matches a local VRF's `route-target import` list, it strips the RD and installs the native IPv4 route into that VRF's RIB[cite: 1].

> **Architectural Summary:** **RD enforces control-plane uniqueness; RT enforces VRF routing policy.**

### 3.2 Hierarchical Two-Label MPLS Stack

| Label Layer | Stack Position | Signaling Protocol | Forwarding Purpose |
| :--- | :--- | :--- | :--- |
| **Transport Label (IGP/LDP)** | Outer (Top, `S=0`) | LDP (RFC 5036) | Switches the frame across transit `P1_Core` routers toward the egress PE's `/32` Loopback address |
| **VPN Service Label** | Inner (Bottom, `S=1`) | MP-iBGP (RFC 4364) | Identifies the destination VRF instance (`VRF_CORP_A` or `VRF_CORP_B`) on the egress PE for payload demultiplexing |

### 3.3 Penultimate Hop Popping (PHP) & BGP-Free Core

- **Penultimate Hop Popping (PHP):** Because each PE router originates its own `/32` Loopback0 prefix, it signals an **`implicit-null` (Label `3`)** binding to adjacent upstream neighbors via LDP. Acting as the penultimate hop, `P1_Core` pops the outer transport label prior to transmitting the frame to the egress PE. Consequently, the egress PE performs only a single LFIB lookup on the exposed inner VPN Service Label.
- **BGP-Free Core:** `P1_Core` operates purely as a Label Switch Router (LSR) inspecting fixed-length 20-bit outer labels in its Label Forwarding Information Base (LFIB). It maintains zero BGP peering sessions and zero knowledge of external or tenant IPv4 prefixes, insulating core hardware resources (TCAM/DRAM) from customer route churn.

---

## 4. Packet Walk: `CE1_Hanoi` → `CE2_HCM`

### 4.1 Control-Plane Signaling (Bottom-Up Label Binding)

1. **Underlay Convergence & Transport Label Binding:** OSPFv2 establishes full reachability across all `/32` Loopback0 addresses. LDP binds local transport labels to each `/32` FEC in the Label Information Base (LIB) and programs the hardware LFIB.
2. **CE-to-PE Route Advertisement:** `CE2_HCM` advertises its local LAN prefix (`192.168.20.0/24`) to `PE2_HCM` over the VRF-bound eBGP session.
3. **VPNv4 Export & Service Label Allocation:** `PE2_HCM` converts the prefix into a 96-bit VPNv4 NLRI, allocates a local VPN Service Label (e.g., `Label 19`), sets the BGP `Next-Hop` to its own Loopback0 (`10.0.0.3`), attaches the Export RT, and advertises the update to `PE1_HN` via MP-iBGP[cite: 1].
4. **Recursive Next-Hop & Label Resolution:** `PE1_HN` imports the route into the target VRF and performs a recursive CEF lookup for BGP Next-Hop `10.0.0.3`, associating it with the outer LDP Transport Label learned from `P1_Core` (e.g., `Label 16`)[cite: 1].

### 4.2 Data-Plane Forwarding (Top-Down Label Switching)

```text
1. CE1 -> PE1 : [ IPv4 Payload ]                        Native IPv4 ingress into VRF subinterface
2. PE1 -> P1  : [ LDP 16 (S=0) ][ VPN 19 (S=1) ][ IPv4 ] Ingress PE imposes 2-label stack
3. P1  -> PE2 : [ VPN 19 (S=1) ][ IPv4 Payload ]        Transit P router pops outer label (PHP)
4. PE2 -> CE2 : [ IPv4 Payload ]                        Egress PE pops VPN label & forwards in VRF
