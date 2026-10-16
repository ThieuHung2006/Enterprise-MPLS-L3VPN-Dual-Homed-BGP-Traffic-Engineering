# 🌐 Enterprise MPLS L3VPN & Dual-Homed BGP Traffic Engineering

## 1. Project Overview
Dự án mô phỏng kiến trúc mạng lõi của một Nhà cung cấp dịch vụ Internet (ISP - AS 65000), cung cấp hai giải pháp mạng trọng yếu:
1. **MPLS L3VPN**: Thiết lập mạng riêng ảo (VRF) bảo mật, độc lập giữa các chi nhánh doanh nghiệp (Hà Nội và TP.HCM) qua hạ tầng ISP với thiết kế **BGP-Free Core**.
2. **Dual-Homed BGP Traffic Engineering**: Quản lý định tuyến Inbound và Outbound ra Internet qua hai cổng Upstream quốc tế (ISP1 - AS 100 và ISP2 - AS 200) thông qua việc điều hướng các thuộc tính BGP (Local Preference và AS-Path Prepending).

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

## 3. Core Architecture & Mechanisms

### 3.1. Customer Isolation via VRF
- **VRF Name**: `VRF_CORP_A`
- **Route Distinguisher (RD)**: `65000:100` biến prefix IPv4 của doanh nghiệp thành VPNv4 duy nhất trong mạng ISP.
- **Route Target (RT)**: `65000:100` điều khiển việc Import/Export tuyến đường VPNv4 giữa các PE router.

### 3.2. Two-Label Stacking & BGP-Free Core
- **Transport Label (Outer)**: Phân phối bởi giao thức LDP, được router `P1_Core` sử dụng để hoán đổi nhãn (Label Swapping) và chuyển tiếp gói tin tới egress PE kế tiếp. Áp dụng cơ chế **Penultimate Hop Popping (PHP)** tại router P[cite: 5].
- **VPN Service Label (Inner)**: Phân phối bởi MP-BGP (VPNv4), được egress PE bóc tách để đẩy dữ liệu vào đúng bảng định tuyến của `VRF_CORP_A`[cite: 3].
- **BGP-Free Core**: `P1_Core` hoàn toàn không tham gia tiến trình BGP và không chứa bất kỳ routing entry nào của mạng doanh nghiệp, giúp giảm tải tối đa phần cứng[cite: 7].

### 3.3. BGP Traffic Engineering Strategy
- **Outbound Path Selection**: Áp dụng chính sách gán `Local-Preference 200` cho tuyến đường học từ ISP1 (so với mặc định 100 từ ISP2), ép toàn bộ traffic nội bộ đi ra ngoài qua ISP1[cite: 2].
- **Inbound Path Selection**: Áp dụng kỹ thuật **AS-Path Prepending** (lặp thêm 3 lần ASN 65000) khi quảng bá prefix công cộng `203.0.113.128/25` sang ISP2, làm tăng độ dài AS-Path để ép Internet bên ngoài ưu tiên trả traffic về qua ISP1[cite: 4].

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
