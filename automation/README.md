# Event-Driven Network Automation with Zabbix & Netmiko

Tài liệu hướng dẫn tích hợp kịch bản tự động hóa cào log và gửi Telegram khi xảy ra sự cố BGP.

## 1. Môi trường yêu cầu
- Zabbix Server 7.0 (Alpine Docker container)
- Python 3 + Netmiko + Paramiko + Requests

## 2. Triển khai nhanh vào Docker
Chạy script cài đặt:
```bash
chmod +x setup_zabbix_docker.sh
./setup_zabbix_docker.sh
```

## 3. Cấu hình trên Zabbix Web

### Bước 1: Khai báo Script
* Truy cập: **Alerts** $\rightarrow$ **Scripts** $\rightarrow$ **Create script**
* **Name:** `Auto Cisco BGP Forensic Dispatch`
* **Type:** `Script`
* **Execute on:** `Zabbix server`
* **Commands:**
  ```bash
  /usr/bin/python3 /usr/lib/zabbix/externalscripts/bgp_forensic.py {HOST.IP} {HOST.NAME}
  ```

### Bước 2: Thiết lập Action (Trigger Actions)
* Truy cập: **Alerts** $\rightarrow$ **Actions** $\rightarrow$ **Trigger actions**
* Trong tab **Operations**, thêm đồng thời 2 hành động:
  1. **Send message** $\rightarrow$ Gửi cảnh báo tới kênh Telegram NOC.
  2. **Execute script** $\rightarrow$ Chọn script `Auto Cisco BGP Forensic Dispatch`.