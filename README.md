# MPLS L3VPN & Dual-Homed BGP Traffic Engineering Lab

A service-provider lab built around a small ISP backbone (**AS 65000**), running on Cisco IOL in EVE-NG. It demonstrates two architectures:

1. **Multi-tenant MPLS L3VPN (RFC 4364)** over a **BGP-free core**, with tenant separation extended to the customer edge using **VRF-Lite** on the CE routers.
2. **Dual-homed eBGP edge traffic engineering** toward two upstream providers, using `LOCAL_PREF` for outbound traffic and `AS_PATH` prepending to influence inbound traffic.

> **Scope:** a proof-of-concept portfolio lab (4 provider routers, 2 customer edge routers, 2 simulated upstreams). It is not a production design. Gaps and planned work are listed in [section 7](#7-known-limitations--future-work).

---

## Table of Contents

1. [Project Overview](#1-project-overview)
2. [Topology & Addressing](#2-topology--addressing)
3. [Design & Architecture](#3-design--architecture)
4. [Packet Walk](#4-packet-walk-ce1_hanoi--ce2_hcm)
5. [BGP Traffic Engineering](#5-bgp-traffic-engineering)
6. [Verification & Proof of Concept](#6-verification--proof-of-concept)
7. [Known Limitations & Future Work](#7-known-limitations--future-work)

---

## 1. Project Overview

| Domain | Implementation |
| :--- | :--- |
| **IGP underlay** | Single-area OSPFv2 (Area 0) on provider links; `/32` Loopback0 interfaces are router IDs and transport endpoints |
| **MPLS transport** | LDP (RFC 5036) binds labels to the `/32` PE loopbacks; LDP-IGP synchronization and LDP session protection are configured |
| **Tenant isolation (provider)** | `VRF_CORP_A` and `VRF_CORP_B` on `PE1_HN` and `PE2_HCM`, each with its own RD and RT |
| **Tenant isolation (customer)** | VRF-Lite (`CORP_A`, `CORP_B`) on `CE1_Hanoi` and `CE2_HCM`, so each tenant has its own routing table and BGP instance on the CE |
| **VPN signaling** | MP-iBGP VPNv4 between `PE1_HN` (`10.0.0.1`) and `PE2_HCM` (`10.0.0.3`) |
| **PE-CE routing** | eBGP over 802.1Q subinterfaces, one per-VRF address family and session per tenant |
| **BGP-free core** | `P1_Core` runs only OSPF and LDP, with no BGP sessions and no tenant or Internet prefixes |
| **Upstream peering** | `ASBR1_Core` has eBGP to `ISP1_AS100` (AS 100) and `ISP2_AS200` (AS 200) |
| **Outbound TE** | Inbound route-map sets `LOCAL_PREF 200` on routes from ISP1 (primary egress) |
| **Inbound TE** | Outbound route-map prepends AS 65000 three times toward ISP2 (backup ingress) |

---

## 2. Topology & Addressing

![Network Topology](images/topology.png)

### 2.1 Device Inventory

| Node | Role | Loopback0 | Interfaces | Protocols |
| :--- | :--- | :--- | :--- | :--- |
| **PE1_HN** | Provider Edge (Hanoi) | `10.0.0.1/32` | `e0/0` to P1; `e0/1` trunk to CE1 (`.10` = tenant A, `.20` = tenant B) | OSPF, LDP, MP-iBGP VPNv4, VRF, eBGP |
| **P1_Core** | Provider (P router) | `10.0.0.2/32` | `e0/0` to PE1, `e0/1` to PE2, `e0/2` to ASBR1 | OSPF, LDP (no BGP) |
| **PE2_HCM** | Provider Edge (Ho Chi Minh) | `10.0.0.3/32` | `e0/0` to P1; `e0/1` trunk to CE2 (`.10` = tenant A, `.20` = tenant B) | OSPF, LDP, MP-iBGP VPNv4, VRF, eBGP |
| **ASBR1_Core** | AS border router | `10.0.0.4/32` | `e0/0` to P1, `e0/1` to ISP1, `e0/2` to ISP2 | OSPF, LDP, eBGP |
| **CE1_Hanoi** | Customer Edge (Hanoi) | - | `e0/0` trunk (`.10` = `CORP_A`, `.20` = `CORP_B`) | eBGP (AS 65001), VRF-Lite, 802.1Q |
| **CE2_HCM** | Customer Edge (Ho Chi Minh) | - | `e0/0` trunk (`.10` = `CORP_A`, `.20` = `CORP_B`) | eBGP (AS 65002), VRF-Lite, 802.1Q |
| **ISP1_AS100** | Primary upstream | `8.8.8.8/32` | `e0/0` to ASBR1 | eBGP (AS 100) |
| **ISP2_AS200** | Secondary upstream | `8.8.8.8/32` | `e0/0` to ASBR1 | eBGP (AS 200) |

> Both upstreams advertise `8.8.8.8/32` so one destination is reachable over two paths. Private ASNs and documentation ranges (`203.0.113.0/24`) are used because this is a lab.

### 2.2 Tenants

| Tenant | PE VRF | CE VRF | RD | Export / Import RT | VLAN | Addressing |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| **Customer A** | `VRF_CORP_A` | `CORP_A` | `65000:100` | `65000:100` | 10 | LANs `192.168.10.0/24` (HN), `192.168.20.0/24` (HCM) |
| **Customer B** | `VRF_CORP_B` | `CORP_B` | `65000:200` | `65000:200` | 20 | WAN `10.10.20.0/30` (HN), `10.10.30.0/30` (HCM); LANs `192.168.100.0/24` (HN), `192.168.200.0/24` (HCM) |

> Both tenants still share one physical CE and one PE-CE link per site, separated by VLANs and, on the CE, by VRF-Lite. A dedicated CE per tenant would be the more common real-world design.

---

## 3. Design & Architecture

### 3.1 VRF, RD and RT

- **VRF** splits one router into independent virtual routers, each with its own RIB and CEF table.
- **Route Distinguisher (RD)** is an 8-byte value prepended to a customer IPv4 prefix, forming a 12-byte **VPNv4 prefix** (for example `65000:100:192.168.10.0/24`). It keeps prefixes unique inside BGP even if tenants reuse the same addresses.
- **Route Target (RT)** is an extended community that controls which VRFs import a route. A PE installs a received VPNv4 route into a VRF only if its RT matches that VRF's import RT.

**RD provides uniqueness. RT provides policy.**

### 3.2 VRF-Lite on the CE routers

In the first version of this lab, both PE-CE subinterfaces on each CE belonged to the CE's single global routing table. Every `network` statement in its global BGP process was advertised on both eBGP sessions, so Tenant A's LANs (`192.168.10.0/24`, `192.168.20.0/24`) appeared inside `VRF_CORP_B` on the PE.

The PE VRFs were not the cause. The shared CE was the bridge between the tenants. VRF-Lite fixes this at the source: each CE subinterface and its eBGP session is bound to its own VRF, with its own BGP address family and its own LAN.

### 3.3 Two-label stack

| Label | Position | Distributed by | Purpose |
| :--- | :--- | :--- | :--- |
| **Transport** | Outer (`S=0`) | LDP | Carries the frame across the core to the egress PE |
| **VPN service** | Inner (`S=1`) | MP-iBGP | Selects the destination VRF on the egress PE |

### 3.4 PHP and BGP-free core

- **PHP:** `P1_Core` is the penultimate hop toward each PE loopback. The PE advertises **implicit-null (label 3)** for its own `/32`, so `P1_Core` pops the outer label and the egress PE needs only one lookup (on the VPN label).
- **BGP-free core:** `P1_Core` forwards on labels only. It holds no tenant or Internet routes, so edge route churn does not reach it.

### 3.5 Key configuration excerpts

> Reference excerpts built from the values documented in this README. Replace the `<...>` placeholders and adjust names to match the real device configs.

```text
! PE: tenant B VRF with its own RD and RT
vrf definition VRF_CORP_B
 rd 65000:200
 address-family ipv4
  route-target export 65000:200
  route-target import 65000:200

! CE (VRF-Lite): tenant B instance
vrf definition CORP_B
 address-family ipv4
interface Ethernet0/0.20
 encapsulation dot1Q 20
 vrf forwarding CORP_B
 ip address 10.10.20.2 255.255.255.252
router bgp 65001
 address-family ipv4 vrf CORP_B
  neighbor 10.10.20.1 remote-as 65000
  neighbor 10.10.20.1 activate
  network 192.168.100.0 mask 255.255.255.0

! P and PE: LDP bound to loopback, IGP synchronization, session protection
mpls ldp router-id Loopback0 force
mpls ldp session protection
router ospf 1
 mpls ldp sync

! ASBR: outbound preference toward ISP1
route-map RM_FROM_ISP1_IN permit 10
 set local-preference 200

! ASBR: AS-path prepending toward ISP2
route-map RM_TO_ISP2_OUT permit 10
 match ip address prefix-list <PL_OWN_PUBLIC>
 set as-path prepend 65000 65000 65000
```

---

## 4. Packet Walk: `CE1_Hanoi` → `CE2_HCM`

### 4.1 Control plane

1. OSPF provides reachability between loopbacks. LDP binds transport labels to every `/32` loopback FEC.
2. `CE2_HCM` advertises `192.168.20.0/24` from its `CORP_A` VRF to `PE2_HCM` over eBGP.
3. `PE2_HCM` allocates a VPN label, sets next hop `10.0.0.3`, attaches the export RT `65000:100`, and sends the VPNv4 update to `PE1_HN` via MP-iBGP.
4. `PE1_HN` imports the route into `VRF_CORP_A` and resolves next hop `10.0.0.3` through the LDP label learned from `P1_Core`.

### 4.2 Data plane

Label values below were observed in the traceroute capture (Figure 2): LDP transport label `20`, VPN label `16`.

```text
1. CE1 -> PE1 : [ IPv4 ]                                  native IPv4 into VRF_CORP_A
2. PE1 -> P1  : [ LDP 20 (S=0) ][ VPN 16 (S=1) ][ IPv4 ]  PE1 imposes two labels
3. P1  -> PE2 : [ VPN 16 (S=1) ][ IPv4 ]                  P1 pops the outer label (PHP)
4. PE2 -> CE2 : [ IPv4 ]                                  PE2 pops the VPN label, forwards in VRF_CORP_A
```

- **PE1_HN (ingress):** performs a VRF lookup, then pushes the VPN label (bottom) and the LDP label (top).
- **P1_Core (transit):** receives both labels. As the penultimate hop for `10.0.0.3/32`, it **pops** the outer label and forwards the frame with only the VPN label.
- **PE2_HCM (egress):** uses the VPN label to select the VRF, pops it, and forwards native IPv4 to `CE2_HCM`.

---

## 5. BGP Traffic Engineering

`ASBR1_Core` terminates eBGP sessions with `ISP1_AS100` (primary) and `ISP2_AS200` (backup).

### 5.1 Outbound: `LOCAL_PREF`

Route-map `RM_FROM_ISP1_IN` sets `LOCAL_PREF 200` on prefixes received from ISP1. Routes from ISP2 keep the default of 100. `LOCAL_PREF` is carried over iBGP throughout AS 65000 and is evaluated **before** `AS_PATH` length in best-path selection, so internal routers prefer the exit through ISP1 and ISP2 acts as backup.

### 5.2 Inbound: `AS_PATH` prepending

`LOCAL_PREF` is not sent across eBGP, so it cannot steer return traffic. Route-map `RM_TO_ISP2_OUT` instead prepends AS 65000 three times when advertising `203.0.113.128/25` to ISP2. Remote networks see:

| Path | `AS_PATH` |
| :--- | :--- |
| via ISP1 | `100 65000 i` |
| via ISP2 | `200 65000 65000 65000 65000 i` (original + 3 prepends) |

Unless a remote AS overrides this with its own policy, it prefers the shorter path through ISP1.

> **Caveat:** prepending *influences* inbound path selection but cannot guarantee it. Remote networks apply their own `LOCAL_PREF` and other policy before `AS_PATH` length.

---

## 6. Verification & Proof of Concept

### 6.1 Underlay: OSPF and LDP

<p align="center">
  <img src="images/p1-core-ospf-neighbor.png" alt="P1 OSPF neighbors">
  <br>
  <i><b>Figure 1:</b> <code>show ip ospf neighbor</code> on P1_Core. All transit adjacencies are FULL.</i>
</p>

<p align="center">
  <img src="images/p1-core-mpls-ldp-neighbor.png" alt="P1 LDP neighbors">
  <br>
  <i><b>Figure 2:</b> <code>show mpls ldp neighbor</code> on P1_Core. All sessions are Oper over TCP 646, anchored to loopbacks through <code>mpls ldp router-id Loopback0 force</code>. The <code>:0</code> suffix denotes a per-platform label space.</i>
</p>

### 6.2 BGP-free core and PHP

<p align="center">
  <img src="images/p1-core-no-customer-routes.png" alt="P1 has no customer routes">
  <br>
  <i><b>Figure 3:</b> <code>show ip route 192.168.20.0</code> on P1_Core returns <code>% Network not in table</code>: tenant prefixes are absent from the core.</i>
</p>

<p align="center">
  <img src="images/p1-core-mpls-forwarding-table.png" alt="P1 LFIB">
  <br>
  <i><b>Figure 4:</b> <code>show mpls forwarding-table</code> on P1_Core shows <code>Pop Label</code> toward the PE loopbacks, confirming PHP.</i>
</p>

### 6.3 End-to-end L3VPN (`VRF_CORP_A`)

<p align="center">
  <img src="images/pe1-bgp-vpnv4-summary.png" alt="PE1 VPNv4 summary">
  <br>
  <i><b>Figure 5:</b> <code>show ip bgp vpnv4 all summary</code> on PE1_HN. The MP-iBGP session with <code>10.0.0.3</code> is Established.</i>
</p>

<p align="center">
  <img src="images/pe1-vrf-corp-a-routing-table.png" alt="PE1 VRF_CORP_A routing table">
  <br>
  <i><b>Figure 6:</b> <code>show ip route vrf VRF_CORP_A</code> on PE1_HN. The remote LAN <code>192.168.20.0/24</code> is installed via BGP with next hop <code>10.0.0.3</code>.</i>
</p>

<p align="center">
  <img src="images/traceroute vrf CORP_A.png" alt="CE1 to CE2 traceroute Tenant A">
  <br>
  <i><b>Figure 7:</b> Traceroute from CE1_Hanoi to CE2_HCM inside <code>VRF_CORP_A</code>. Hop 2 (P1_Core) shows the two-label stack <code>[Labels 20/16]</code>; hop 3 shows the single VPN   label <code>[Label 16]</code>, confirming PHP at P1.</i>
</p>

### 6.4 Tenant isolation with VRF-Lite

**Before VRF-Lite (shared CE routing table)**

<p align="center">
  <img src="images/before-vrf-lite-corp-b-route-leak.png" alt="VRF_CORP_B routing table before VRF-Lite">
  <br>
  <i><b>Figure 8:</b> <code>show ip route vrf VRF_CORP_B</code> on PE1_HN before VRF-Lite. Tenant A prefixes <code>192.168.10.0/24</code> and <code>192.168.20.0/24</code> appear inside VRF_CORP_B because the shared CE advertised them on both eBGP sessions.</i>
</p>

**After VRF-Lite**

<p align="center">
  <img src="images/after-vrf-lite-corp-b-routing-table.png" alt="VRF_CORP_B routing table after VRF-Lite">
  <br>
  <i><b>Figure 9:</b> <code>show ip route vrf VRF_CORP_B</code> on PE1_HN after VRF-Lite. No Tenant A prefixes remain.</i>
</p>

<p align="center">
  <img src="images/after-vrf-lite-corp-a-routing-table.png" alt="VRF_CORP_A routing table after VRF-Lite">
  <br>
  <i><b>Figure 10:</b> <code>show ip route vrf VRF_CORP_A</code> on PE1_HN after VRF-Lite. No Tenant B prefixes (<code>192.168.100.0/24</code>, <code>192.168.200.0/24</code>).</i>
</p>

<p align="center">
  <img src="images/pe1-show-ip-vrf-detail.png" alt="VRF detail with RD and RT">
  <br>
  <i><b>Figure 11:</b> <code>show ip vrf detail</code> on PE1_HN showing separate RD and import/export RT for each tenant.</i>
</p>

**Data-plane test (Cross-tenant ping and traceroute)**

<p align="center">
  <img src="images/cross-tenant-ping-test.png" alt="Cross-tenant ping test success">
  <br>
  <img src="images/cross-tenant-ping-test_false.png" alt="Cross-tenant ping test failure">
  <br>
  <i><b>Figure 12:</b> Ping within Tenant B succeeds (<code>!!!!!</code>), while ping from Tenant A toward a Tenant B-only prefix fails (<code>% Network not in table</code>).</i>
</p>

<p align="center">
  <img src="images/traceroute vrf CORP_B.png" alt="CE1 to CE2 traceroute Tenant B">
  <br>
  <i><b>Figure 12b:</b> Traceroute from CE1_Hanoi to CE2_HCM inside <code>VRF_CORP_B</code>. Transit hop across P1_Core uses the same outer transport label <code>20</code>, but distinct inner VPN label <code>26</code> (compared to label <code>16</code> in Tenant A), proving deterministic data-plane separation across the shared core.</i>
</p>

**Result:** after VRF-Lite, no prefixes of one tenant were observed in the other tenant's VRF, and the cross-tenant ping failed. This tests one direction; the reverse direction (`ping vrf CORP_B 192.168.20.1`) can be added for completeness.

### 6.5 LDP hardening

- **LDP-IGP synchronization:** while the LDP session on a link is not yet established, OSPF advertises the maximum link metric on that link, so traffic is not sent onto a path that has no labels (a blackhole during convergence).
- **LDP session protection:** keeps the LDP session and its label bindings alive through a targeted LDP session when the direct link fails but an alternate path to the neighbor exists.

<p align="center">
  <img src="images/p1-core-ldp-igp-sync.png" alt="LDP IGP sync status">
  <br>
  <i><b>Figure 13:</b> <code>show mpls ldp igp sync</code> on P1_Core shows synchronization achieved on the transit interfaces.</i>
</p>

<p align="center">
  <img src="images/p1-core-ldp-session-protection.png" alt="LDP session protection">
  <img src="images/p1-core-ldp-session-protection_discover.png" alt="LDP session protection discover">
  <br>
  <i><b>Figure 14:</b> <code>show mpls ldp neighbor</code> / <code>show mpls ldp discovery</code> output showing session protection configured.</i>
</p>

> In the current single-P topology there is no alternate path to the PEs, so session protection is configured but cannot show a benefit yet. It becomes testable after a second P router or redundant link is added.

### 6.6 BGP traffic engineering

<p align="center">
  <img src="images/asbr1-bgp-outbound-local-pref.png" alt="ASBR1 LOCAL_PREF">
  <br>
  <i><b>Figure 15:</b> <code>show ip bgp 8.8.8.8</code> on ASBR1_Core. The path via ISP1 (<code>203.0.113.2</code>) has <code>localpref 200</code> and is best; the path via ISP2 (<code>203.0.113.6</code>) keeps <code>localpref 100</code>.</i>
</p>

<p align="center">
  <img src="images/isp2-bgp-inbound-as-path-prepend.png" alt="ISP2 sees prepended path">
  <br>
  <i><b>Figure 16:</b> <code>show ip bgp 203.0.113.128</code> on ISP2_AS200. The prefix arrives with the elongated path <code>65000 65000 65000 65000 i</code>.</i>
</p>

## 7. Technical Verification & Wireshark Packet Analysis

---

### 7.1 MPLS L3VPN Control Plane & Data Plane Encapsulation

This section provides packet-level verification via Wireshark to validate the MPLS Two-Label Stack architecture, Penultimate Hop Popping (PHP) optimization, and MP-iBGP VPNv4 control-plane signaling across the provider core.

#### 1. Two-Label Stack Encapsulation (PE1_HN to P1_Core)

When customer traffic belonging to `VRF_CORP_A` travels from `CE1_Hanoi` (`192.168.10.1`) to `CE2_HCM` (`192.168.20.1`), the ingress Provider Edge router (`PE1_HN`) imposes a two-level MPLS label stack onto the IPv4 payload before forwarding the frame into the MPLS core.

<p align="center">
  <img src="images/wireshark-pe1-p1-mpls-two-labels.png" alt="Two-Label Stack Encapsulation">
  <br>
  <i><b>Figure 17:</b> Packet capture on the PE1_HN to P1_Core link demonstrating the Two-Label Stack operation for Tenant A traffic.</i>
</p>

* **Outer Label (IGP/LDP Transport Label – `18`):**
  * **Bottom of Label Stack (`S=0`):** Indicates that additional MPLS headers follow underneath.
  * **Functionality:** Dynamically allocated and distributed by LDP over OSPF. Transit core routers (`P1_Core`) inspect exclusively this top label to label-switch the packet toward the BGP Next-Hop loopback address of the egress PE (`10.0.0.3` on `PE2_HCM`), keeping the core completely BGP-free.
* **Inner Label (MP-BGP VPN Label – `21`):**
  * **Bottom of Label Stack (`S=1`):** Marks the end of the MPLS label stack, immediately preceding the Layer 3 IPv4 header.
  * **Functionality:** Allocated by `PE2_HCM` via MP-iBGP and bound to `VRF_CORP_A`. This label remains untouched across the transit path and instructs the egress PE on which specific VRF routing instance to use for final lookup.
* **Encapsulated Payload:** The original customer ICMP Echo packet (`Src: 192.168.10.1`, `Dst: 192.168.20.1`) is transparently tunneled across the shared service provider backbone.

---

#### 2. Penultimate Hop Popping (PHP) Mechanism (P1_Core to PE2_HCM)

To eliminate the performance penalty of a double lookup (LFIB lookup followed by FIB lookup) at the egress Provider Edge router, LDP leverages Penultimate Hop Popping (PHP) triggered by an `implicit-null` (Label `3`) advertisement from `PE2_HCM`.

<p align="center">
  <img src="images/wireshark-p1-pe2-mpls-one-labe.png" alt="Penultimate Hop Popping">
  <br>
  <i><b>Figure 18:</b> Packet capture on the P1_Core to PE2_HCM link verifying Penultimate Hop Popping (PHP) behavior.</i>
</p>

* **Outer Label Removal:** The penultimate router (`P1_Core`) pops the transport label (`Label 18`) prior to transmitting the frame across the final link to `PE2_HCM`.
* **Single Remaining Label (`Label: 21, S=1`):** The packet arrives at `PE2_HCM` carrying only the bottom-of-stack VPN label (`21`). Furthermore, the MPLS TTL decrements from `254` to `253`, confirming a valid transit hop across `P1_Core`.
* **Data-Plane Optimization:** Upon receipt, `PE2_HCM` performs a single LFIB lookup on Label `21`, strips the MPLS header, and forwards the native IPv4 packet directly out the egress sub-interface toward `CE2_HCM`.

---

#### 3. MP-iBGP VPNv4 Routing Update & Route Target Filtering

The provider edge routers establish an MP-iBGP peering session (`AFI: 1`, `SAFI: 128`) over Loopback0 interfaces to exchange customer VPNv4 prefixes, labels, and extended communities.

<p align="center">
  <img src="images/wireshark-bgp-vpnv4-update-rt.png" alt="MP-BGP VPNv4 Update Message">
  <br>
  <i><b>Figure 19:</b> Wireshark capture of an MP-iBGP UPDATE message advertising Tenant A reachability and attributes.</i>
</p>

* **Multiprotocol Reachable NLRI (`MP_REACH_NLRI`):**
  * **Address Family (`AFI: 1 / SAFI: 128`):** Negotiates Labeled VPNv4 Unicast capability.
  * **Route Distinguisher (`RD: 65000:100`):** Prepended to the customer IPv4 subnet (`192.168.10.0/24`) to construct a globally unique 96-bit VPNv4 prefix, allowing overlapping IP spaces across different tenants.
* **Label Binding (`Label Stack: 21`):** Associates MPLS VPN Label `21` with the advertised prefix in the control plane.
* **Extended Communities (`Route Target: 65000:100`):** Attached as a BGP transitive extended community attribute. The receiving PE evaluates this value against its configured `route-target import` policies to selectively import the route into `VRF_CORP_A`.

---

### 7.2 Multi-Tenant Traffic Isolation via VRF-Lite

At the Customer Edge (`CE1_Hanoi`), VRF-Lite combined with IEEE 802.1Q VLAN tagging is deployed on the PE-CE trunk interface to enforce strict Layer 2 and Layer 3 segmentation between tenants, preventing inter-tenant route leaking.

<p align="center">
  <img src="images/wireshark-vlan20-bgp-update-vrf-lite.png" alt="VRF-Lite 802.1Q BGP Update on VLAN 20">
  <br>
  <i><b>Figure 20:</b> BGP UPDATE captured on the PE1-CE1 trunk (VLAN 20) after VRF-Lite implementation, confirming strict route isolation.</i>
</p>

* **802.1Q Data-Link Isolation (`VLAN ID: 20`):** The eBGP session between `CE1_Hanoi` (`10.10.20.2`) and `PE1_HN` (`10.10.20.1`) is encapsulated within VLAN 20 on sub-interface `Ethernet0/0.20`, isolating Tenant B control-plane traffic at Layer 2.
* **Strict VRF RIB Separation:** Inspection of the BGP `NLRI` field confirms that only Tenant B's internal subnet (`192.168.100.0/24`) is advertised over this peering session. Tenant A prefixes (`192.168.10.0/24`) are completely isolated within `vrf CORP_A`, eliminating cross-tenant route propagation.

---

### 7.3 External BGP Traffic Engineering & Automated Failover (ISP1 / ISP2)

The enterprise core (`AS 65000`) is dual-homed to two independent upstream providers: **ISP1 (`AS 100`)** and **ISP2 (`AS 200`)**. Deterministic routing policies are enforced at `ASBR1_Core` to designate ISP1 as the primary transit path and ISP2 as the standby backup path.

#### 1. Inbound Traffic Engineering via AS-Path Prepending

To influence upstream autonomous systems to route return traffic via the primary link (`ISP1_AS100`), `ASBR1_Core` applies an outbound route-map toward `ISP2_AS200` that prepends its own ASN multiple times.

<p align="center">
  <img src="images/wireshark-bgp-as-path-prepend-isp2.png" alt="BGP UPDATE to ISP2 with AS-Path Prepend">
  <br>
  <i><b>Figure 21:</b> Outbound BGP UPDATE messages from ASBR1_Core to ISP2_AS200 demonstrating 3x AS-Path Prepending.</i>
</p>

* **Path Manipulation (`AS_PATH: 65000 65000 65000 65000`):** By prepending `AS 65000` three additional times, the total AS-Path length advertised to ISP2 increases to four hops. Upstream BGP speakers executing the standard BGP Best-Path Selection algorithm prefer the shorter path through `ISP1_AS100` for inbound traffic delivery.
* **BGP Update Packing & Attribute Granularity:** Wireshark captures two separate `UPDATE` messages within the same TCP segment because BGP groups prefixes by shared Path Attributes:
  * The first UPDATE advertises transit prefixes learned from AS 100 (`8.8.8.8/32`, `100.0.0.0/24`) carrying the path `65000 65000 65000 65000 100`.
  * The second UPDATE advertises locally originated enterprise space (`203.0.113.128/25`) carrying the path `65000 65000 65000 65000` along with `MULTI_EXIT_DISC (MED: 0)`.

---

#### 2. Outbound Traffic Enforcement & Automated Failover Verification

Outbound path selection is controlled via the BGP **Local Preference** attribute: routes learned from `ISP1_AS100` are assigned `Local-Pref 200`, whereas routes from `ISP2_AS200` retain the default `Local-Pref 100`. Data-plane behavior is validated by monitoring continuous ICMP probes to `8.8.8.8` before and after a simulated primary link failure.

<p align="center">
  <img src="images/wireshark-failover-primary-isp1.png" alt="Normal Outbound Traffic via ISP1">
  <br>
  <i><b>Figure 22:</b> Normal operating state. Outbound ICMP traffic to 8.8.8.8 exits via the primary ISP1 link (Source IP: 203.0.113.1).</i>
</p>

* **Primary Path Operation (Figure 22):** Under normal conditions, ICMP Echo Requests egress via interface `Ethernet0/1` (`Src: 203.0.113.1`, subnet `203.0.113.0/30`), confirming that `Local-Preference 200` installs `ISP1_AS100` as the active FIB next-hop.

<p align="center">
  <img src="images/wireshark-failover-backup-isp2.png" alt="Automated Failover Outbound Traffic via ISP2">
  <br>
  <i><b>Figure 23:</b> Failover operating state. Following primary link failure, outbound ICMP traffic automatically converges onto the backup ISP2 link (Source IP: 203.0.113.5).</i>
</p>

* **Automated Convergence & Failover (Figure 23):** When interface `Ethernet0/1` is administratively shut down to simulate a fiber cut, `ASBR1_Core` invalidates the primary BGP next-hop and immediately installs the standby path via `ISP2_AS200` into the CEF table. Subsequent ICMP packets automatically egress through `Ethernet0/2` with source address `203.0.113.5` (subnet `203.0.113.4/30`), demonstrating resilient sub-second failover without manual intervention.
2. ISP1 failure and recovery test with ping loss and convergence time.
3. ASBR filtering and BGP authentication.
4. A second P router for core redundancy, which also lets LDP session protection be demonstrated.
5. BFD on eBGP peers, with measured (not assumed) convergence times.
6. Publish device configs in a `configs/` folder so the lab can be reproduced.
