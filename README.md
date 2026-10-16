# 🌐 Enterprise MPLS L3VPN & Dual-Homed BGP Traffic Engineering

## 1. Project Overview

This project simulates a fully functional Internet Service Provider (ISP) core network architecture. It is designed to deliver two critical telecommunications services:

1.  **MPLS L3VPN (Virtual Private Network):** Providing secure, isolated, and routable private connections between enterprise branches (Hanoi and Ho Chi Minh City) over a shared ISP infrastructure. This is implemented using a **BGP-Free Core** design to minimize the routing table size on transit routers.
2.  **BGP Traffic Engineering (TE):** Managing inbound and outbound Internet traffic via a Dual-Homed upstream connection (ISP1 and ISP2). BGP path attributes are manipulated to enforce primary and backup routing paths for optimal performance and redundancy.

## 2. Network Topology & IP Addressing

*(Insert your refined network topology image here: e.g., `![Network Topology](images/topology.png)`)*

### Device Roles & Protocol Mapping

| Device Node | Role | Loopback 0 (Router-ID) | Physical Interfaces (WAN/Link) | Routing Protocols & Features |
| :--- | :--- | :--- | :--- | :--- |
| **PE1_HN** | Provider Edge (Site HN) | `10.0.0.1/32` | `e0/0` (To P1), `e0/1` (To CE1) | OSPF, LDP, MP-iBGP (VPNv4), VRF, eBGP |
| **P1_Core** | Transit Core (P-Router) | `10.0.0.2/32` | `e0/0`, `e0/1`, `e0/2` | OSPF Area 0, MPLS LDP *(BGP-Free)* |
| **PE2_HCM** | Provider Edge (Site HCM) | `10.0.0.3/32` | `e0/0` (To P1), `e0/1` (To CE2) | OSPF, LDP, MP-iBGP (VPNv4), VRF, eBGP |
| **ASBR1_Core** | Autonomous System BR | `10.0.0.4/32` | `e0/0` (To P1), `e0/1`, `e0/2` (To ISPs) | OSPF, LDP, eBGP (Upstream) |
| **CE1_Hanoi** | Customer Edge (HN) | `192.168.10.1/24` | `e0/0` (To PE1) | eBGP (AS 65001) |
| **CE2_HCM** | Customer Edge (HCM)| `192.168.20.1/24` | `e0/0` (To PE2) | eBGP (AS 65002) |
| **ISP1_AS100**| Upstream Provider 1 | `8.8.8.8/32` | `e0/0` (To ASBR1) | eBGP (AS 100) |
| **ISP2_AS200**| Upstream Provider 2 | `8.8.8.8/32` | `e0/0` (To ASBR1) | eBGP (AS 200) |

## 3. Core Technologies & Architecture (Deep Dive)

### 3.1. Customer Isolation via VRF
To prevent IP overlapping and ensure data privacy, the enterprise traffic is placed into a dedicated routing instance (`VRF_CORP_A`). 
*   **Route Distinguisher (RD):** `65000:100` (Makes the IPv4 prefix globally unique across the ISP).
*   **Route Target (RT):** `65000:100` (Controls the import and export of VPNv4 routes between PE routers).

### 3.2. Two-Label Stacking (MPLS) & BGP-Free Core
The architecture utilizes a double-label stack mechanism for data plane forwarding:
*   **Transport Label (Outer):** Distributed by **LDP**, used by the transit core (`P1_Core`) to switch packets toward the egress PE.
*   **VPN Service Label (Inner):** Distributed by **MP-BGP**, used by the egress PE to identify which specific VRF the packet belongs to.
*   **Result:** The `P1_Core` router operates purely on label switching and maintains absolutely no knowledge of customer IP prefixes (BGP-Free Core), heavily optimizing hardware resources.

### 3.3. BGP Traffic Engineering
To enforce ISP1 as the primary link and ISP2 as the backup link for Internet traffic:
*   **Outbound TE:** Applied a `Route-Map` on `ASBR1_Core` to set **`Local-Preference 200`** for prefixes learned from ISP1 (default is 100).
*   **Inbound TE:** Applied **`AS-Path Prepending`** (adding AS 65000 three times) for the public prefix `203.0.113.128/25` advertised to ISP2. This artificially inflates the path length, forcing the global Internet to return traffic via ISP1.

## 4. Verification & Proof of Concept

### 4.1. Underlay Network (OSPF & MPLS LDP)
*Verified that the core network has established OSPF adjacencies and LDP peerings successfully.*
*(Insert Screenshot: `show ip ospf neighbor` and `show mpls ldp neighbor` from P1_Core)*

### 4.2. BGP-Free Core & MPLS Forwarding
*Demonstrating that the P-router has no customer routes in its RIB and forwards traffic purely based on LFIB.*
*(Insert Screenshot: `show ip route 192.168.20.0` showing "% Network not in table" on P1_Core)*

### 4.3. End-to-End L3VPN Connectivity
*Verification of data plane connectivity from CE1 to CE2. The `traceroute` clearly shows the double-label stack (LDP & VPN labels) being pushed and swapped across the ISP core.*
*(Insert Screenshot: `ping` and `traceroute` from CE1_Hanoi to CE2_HCM)*

### 4.4. BGP Traffic Engineering Selection
*Verification of the BGP Best Path selection based on manipulated attributes.*
*   **(Outbound):** Traffic to `8.8.8.8` prefers ISP1 due to `LocPrf 200`.
*   **(Inbound):** Advertising route to ISP2 shows an elongated AS-Path (`65000 65000 65000 65000 i`).
*(Insert Screenshot: `show ip bgp 8.8.8.8` and `show ip bgp 203.0.113.128` from ASBR1 and ISP2)*

## 5. Next Steps & Enhancements
*   Wireshark packet capture analysis to inspect the structure of MP-BGP Update messages and MPLS headers.
*   Implement `as-override` or `allowas-in` to simulate scenarios where branches use identical Autonomous System Numbers.
*   Implement Route Leaking to provide Internet access directly to the VRF customers.
