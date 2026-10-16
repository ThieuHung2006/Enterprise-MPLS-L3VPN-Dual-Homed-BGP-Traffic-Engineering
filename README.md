# 🌐 Enterprise MPLS L3VPN & Dual-Homed BGP Traffic Engineering

## 1. Project Overview

### Executive Summary
This enterprise-grade infrastructure project emulates an autonomous **Tier-2 Internet Service Provider (ISP) backbone (AS 65000)**, demonstrating production-ready implementations of **Multi-Protocol Label Switching Layer 3 Virtual Private Networks (MPLS L3VPN)** and **Dual-Homed Multi-AS BGP Traffic Engineering**. 

Designed strictly in compliance with service provider architectural standards (**RFC 4364 / RFC 2547**), the design addresses two primary enterprise networking challenges:
* Providing secure, multi-tenant enterprise WAN transport over a shared core while completely decoupling tenant routes from core transit routers (**BGP-Free Core**).
* Engineering end-to-end multi-homed Internet routing redundancy and deterministic path selection using advanced BGP attribute tuning.

---

### Key Architectural Pillars

#### 1. Multi-Tenant Enterprise MPLS L3VPN (Core WAN Transport)
* **Customer Isolation via VRF & Multiprotocol BGP**: Implements Virtual Routing and Forwarding (`VRF_CORP_A`) on edge nodes, isolating enterprise routing instances locally. Inter-site prefix uniqueness and distribution across the ISP core are managed using unique **Route Distinguishers (RD)** and granular **Route Target (RT)** extended communities transported via **MP-iBGP (VPNv4 Address Family)**.
* **Two-Label MPLS Switching & PHP**: Leverages a dual-label stack architecture comprising an outer **Transport Label** (distributed via LDP for core transit) and an inner **VPN Service Label** (distributed via MP-BGP for VRF demultiplexing). Core switching incorporates **Penultimate Hop Popping (PHP)** on the P router to optimize packet forwarding efficiency and eliminate double-label lookups at egress nodes.
* **BGP-Free Core Scalability**: Offloads all customer routing complexity to Provider Edge (PE) routers, ensuring the transit core (`P1_Core`) remains 100% agnostic to tenant routes (zero customer prefixes in core RIB/FIB), drastically minimizing control plane overhead and hardware resource utilization.

#### 2. Dual-Homed BGP Traffic Engineering (Upstream Edge Transit)
* **Deterministic Outbound Optimization (Local Preference)**: Establishes an Active/Standby egress model by overriding default BGP path selection using a higher `Local-Preference` (200) for upstream routes received from **ISP1 (AS 100)** over **ISP2 (AS 200)**, deterministic routing all internal outbound Internet traffic via the primary transit link.
* **Inbound Traffic Manipulation (AS-Path Prepending)**: Solves the asymmetrical routing dilemma inherent in multi-homed BGP designs by applying `AS-Path Prepending` (triple ASN injection) on advertised public enterprise subnets (`203.0.113.128/25`) towards ISP2. This artificially degrades the metric seen by remote autonomous systems, ensuring optimal and deterministic inbound convergence through ISP1.

---

### Core Technical Highlights
* **Underlay Routing**: OSPFv2 Single-Area Backbone (Area 0) providing sub-second loopback reachability for LDP and MP-BGP transport peering.
* **Label Distribution**: Native LDP (RFC 5036) for seamless Label Switched Path (LSP) establishment across the transit core.
* **Overlay Multiprotocol Signaling**: MP-iBGP peering between PE routers for VPNv4 exchange; eBGP peering for tenant CE-to-PE and external Upstream multi-homing.
* **High Availability & Fault Tolerance**: End-to-end failover architecture capable of instant path redirection upon primary link degradation without route blackholing.

---

## 2. Network Topology & Addressing Table

### Topology Diagram
![Network Topology](images/topology.png)

### Device Roles & Protocol Mapping
| Node | Role | Router ID / Loopback 0 | Physical Interfaces | Protocols & Features |
| :--- | :--- | :--- | :--- | :--- |
| **PE1_HN** | Provider Edge (Hanoi) | `10.0.0.1/32` | `e0/0` (To P1), `e0/1` (To CE1) | OSPF Area 0, LDP, MP-iBGP (VPNv4), VRF, eBGP |
| **P1_Core** | Transit Core (P-Router) | `10.0.0.2/32` | `e0/0`, `e0/1`, `e0/2` | OSPF Area 0, MPLS LDP (BGP-Free Core) |
| **PE2_HCM** | Provider Edge (HCM) | `10.0.0.3/32` | `e0/0` (To P1), `e0/1` (To CE2) | OSPF Area 0, LDP, MP-iBGP (VPNv4), VRF, eBGP |
| **ASBR1_Core** | AS Border Router | `10.0.0.4/32` | `e0/0` (To P1), `e0/1` (ISP1), `e0/2` (ISP2) | OSPF Area 0, LDP, eBGP (Upstream Multi-homing) |
| **CE1_Hanoi** | Customer Edge (Hanoi) | `192.168.10.1/24` | `e0/0` (To PE1) | eBGP (AS 65001) |
| **CE2_HCM** | Customer Edge (HCM) | `192.168.20.1/24` | `e0/0` (To PE2) | eBGP (AS 65002) |
| **ISP1_AS100** | Upstream Tier-1 (Primary) | `8.8.8.8/32` | `e0/0` (To ASBR1) | eBGP (AS 100) |
| **ISP2_AS200** | Upstream Tier-1 (Backup) | `8.8.8.8/32` | `e0/0` (To ASBR1) | eBGP (AS 200) |

---

## 3. Core Architecture & Technical Deep-Dive

### 3.1. Multi-Tenant Isolation & VPNv4 Control Plane (RFC 4364)
To support overlapping customer IPv4 address spaces across a shared MPLS backbone, the architecture enforces strict routing separation at the Provider Edge (`PE1_HN` and `PE2_HCM`) using Virtual Routing and Forwarding (`VRF_CORP_A`).

* **Route Distinguisher (RD `65000:100`) — Uniqueness in BGP:** Standard BGP-4 cannot distinguish between identical IPv4 prefixes belonging to different customers. By prepending an 8-byte RD (`ASN:NN`) to the 4-byte IPv4 prefix, the PE router constructs a globally unique 12-byte (96-bit) **VPNv4 NLRI** (e.g., `65000:100:192.168.10.0/24`). This ensures that route advertisements within the MP-iBGP session never overwrite one another, even when multiple tenants utilize identical RFC 1918 subnets.
* **Route Target (RT `65000:100`) — Policy-Driven Topology Control:** Unlike the RD, which merely provides uniqueness, the Route Target is an 8-byte **MP-BGP Extended Community** attribute (`rt 65000:100`) that dictates VRF import/export policies. When `PE1_HN` redistributes customer routes from `CE1_Hanoi` (AS 65001) into MP-BGP, it attaches the export RT `65000:100`. Upon receiving the VPNv4 update, `PE2_HCM` evaluates its local VRF import lists, matches `65000:100`, strips the RD, and installs the native IPv4 route into the `VRF_CORP_A` Routing Information Base (RIB).
* **CEF & LFIB Integration:** Each VRF maintains an isolated Cisco Express Forwarding (CEF) table. When MP-BGP allocates a local **VPN Service Label** (Inner Label) for a customer prefix, it binds that label directly to the VRF's specific adjacency entry in the Label Forwarding Information Base (LFIB).

---

### 3.2. End-to-End Packet Walk: Control Plane vs. Data Plane
The MPLS L3VPN architecture relies on a hierarchical **Two-Label Stack** (`Label EtherType 0x8847`) that separates core transport routing from edge service demultiplexing.

#### A. Control Plane Signaling (Bottom-Up Label Binding)
1. **IGP & Transport Label Distribution:** OSPF Area 0 establishes full reachability between the `/32` Loopback0 interfaces of `PE1_HN` (`10.0.0.1`), `P1_Core` (`10.0.0.2`), and `PE2_HCM` (`10.0.0.3`). Label Distribution Protocol (LDP) operates alongside OSPF, binding local transport labels to each `/32` FEC (Forwarding Equivalence Class) in the LIB (Label Information Base) and programming the hardware LFIB.
2. **Service Label Signaling:** `CE2_HCM` advertises `192.168.20.0/24` to `PE2_HCM` via eBGP. `PE2_HCM` allocates a local VPN label (e.g., `Label 19`) for this prefix, sets the BGP `Next-Hop` to its own Loopback0 (`10.0.0.3`), and signals the update to `PE1_HN` over the MP-iBGP VPNv4 session.
3. **Recursive Next-Hop Resolution:** `PE1_HN` receives the VPNv4 route with Next-Hop `10.0.0.3`. To forward traffic toward `10.0.0.3`, `PE1_HN` performs a recursive lookup in its global CEF table, matching the LDP-assigned Transport Label (e.g., `Label 16`) learned from `P1_Core`.

#### B. Data Plane Forwarding Walk-Through (`CE1_Hanoi` $\rightarrow$ `CE2_HCM`)
+-----------------------------------------------------------------------------------+
| 1. CE1 -> PE1 : [ IPv4 Payload (Src: 192.168.10.1 -> Dst: 192.168.20.1) ]         |
| 2. PE1 -> P1  : [ Outer LDP Label (16) ] [ Inner VPN Label (19) ] [ IPv4 Payload ]|
| 3. P1  -> PE2 : [ Inner VPN Label (19) ] [ IPv4 Payload ]  <-- PHP Pops Top Label |
| 4. PE2 -> CE2 : [ IPv4 Payload (Src: 192.168.10.1 -> Dst: 192.168.20.1) ]         |
+-----------------------------------------------------------------------------------+

---

## 4. Verification & Proof of Concept (PoC)

### 4.1. Core Underlay Infrastructure (OSPF & MPLS LDP)
Xác nhận trạng thái hình thành láng giềng OSPF (`FULL`) và phiên LDP (`Oper`) trên hạ tầng mạng lõi `P1_Core`:
![P1 OSPF Neighbor](images/p1-core-ospf-neighbor.png)
![P1 LDP Neighbor](images/p1-core-mpls-ldp-neighbor.png)

### 4.2. BGP-Free Core & LFIB Switching
Kiểm chứng `P1_Core` hoàn toàn không có định tuyến khách hàng trong RIB (báo `% Network not in table`), chuyển tiếp hoàn toàn qua bảng LFIB với cơ chế gỡ nhãn PHP (`Pop Label`):
![P1 BGP-Free RIB](images/p1-core-no-customer-routes.png)
![P1 LFIB Forwarding Table](images/p1-core-mpls-forwarding-table.png)

### 4.3. MP-BGP Control Plane & VRF Routing Table
Kiểm tra phiên MP-iBGP VPNv4 giữa các PE và bảng định tuyến độc lập của khách hàng trên `PE1_HN`:
![PE1 BGP VPNv4 Summary](images/pe1-bgp-vpnv4-summary.png)
![PE1 VRF Routing Table](images/pe1-vrf-corp-a-routing-table.png)

### 4.4. End-to-End L3VPN Data Plane (Two-Label Stacking)
Kiểm tra kết nối CE-to-CE. Lệnh `traceroute` hiển thị rõ cơ chế ngăn xếp 2 nhãn MPLS (Transport Label 18 và VPN Service Label 16):
![CE1 to CE2 Traceroute Label Stack](images/ce1-l3vpn-traceroute-label-stack.png)

### 4.5. BGP Traffic Engineering Selection
Kiểm chứng thuật toán chọn đường BGP Best Path cho cả 2 chiều:
- **Outbound**: Tuyến đường đi `8.8.8.8` ưu tiên chọn ISP1 nhờ `Local-Preference 200`[cite: 2].
- **Inbound**: Tại ISP2, prefix `203.0.113.128/25` nhận được mang chuỗi AS-Path bị kéo dài (`65000 65000 65000 65000`)[cite: 4].

![ASBR1 Outbound Local Preference Selection](images/asbr1-bgp-outbound-local-pref.png)
![ISP2 Inbound AS-Path Prepending Verification](images/isp2-bgp-inbound-as-path-prepend.png)

---

## 5. Directory Structure
```text
.
├── images/
│   ├── asbr1-bgp-outbound-local-pref.png
│   ├── ce1-l3vpn-traceroute-label-stack.png
│   ├── isp2-bgp-inbound-as-path-prepend.png
│   ├── p1-core-mpls-forwarding-table.png
│   ├── p1-core-mpls-ldp-neighbor.png
│   ├── p1-core-no-customer-routes.png
│   ├── p1-core-ospf-neighbor.png
│   ├── pe1-bgp-vpnv4-summary.png
│   ├── pe1-vrf-corp-a-routing-table.png
│   └── topology.png
└── README.md
