# LAB6 - Xác thực Active Directory và Chính sách truy cập trên pfSense

- Họ và tên: Phạm Hữu Ân
- MSSV: 1150080084
- Mã lớp: CNPM1
- Tên lab: LAB6 - Xác thực Active Directory và Chính sách truy cập trên pfSense

## Phiên bản môi trường

- Hypervisor / Virtualization: Oracle VirtualBox 7.x
- Tường lửa: pfSense CE (Community Edition)
- Domain Controller: Windows Server 2019 / 2022 (AD DS + DNS)
- Máy trạm Client: Windows 11 (Client-A, Client-B)
- Máy chủ DMZ: Ubuntu Server (Nginx với các IP ảo mô phỏng)
- Công cụ quản trị: Máy thật Host (Quy hoạch mạng quản trị MGMT)

## Cách dựng môi trường

1. **Tạo các mạng ảo (VirtualBox Network Manager)**:
   - `LAB6-WAN`: NAT Network (192.168.250.0/24) cho cổng WAN của pfSense.
   - `MGMT`: Host-Only Network (192.168.56.0/24) để quản trị viên truy cập WebGUI.
   - `lab6-lan`: Internal Network (10.6.0.0/24) cho vùng mạng LAN (DC01, Client-A, Client-B).
   - `lab6-dmz`: Internal Network (172.16.6.0/24) cho vùng chứa các website mô phỏng.
2. **Cấu hình 4 Card mạng cho pfSense-LAB6**:
   - Adapter 1: NAT Network (`LAB6-WAN`)
   - Adapter 2: Host-Only Adapter (`MGMT`)
   - Adapter 3: Internal Network (`lab6-lan`)
   - Adapter 4: Internal Network (`lab6-dmz`)
3. **Cài đặt và quy hoạch IP các Interface trên pfSense**:
   - WAN: Nhận IP động qua DHCP.
   - MGMT: `192.168.56.254/24` (Dùng để truy cập WebGUI quản trị).
   - LAN: `10.6.0.1/24` (Gateway cho mạng LAN).
   - DMZ: `172.16.6.1/24` (Gateway cho vùng DMZ).
4. **Chuẩn bị các máy chủ và dịch vụ**:
   - Cấu hình Domain Controller (`DC01` tại `10.6.0.10`), cài đặt AD DS với domain `lab6.test`, tạo OU `LAB6-Users` và các tài khoản `sv_a`, `sv_b`, cùng service account `svc_pfsense_ldap`.
   - Cấu hình máy chủ `DMZ-WEB` (`172.16.6.10`) gắn thêm các IP phụ (`.20`, `.30`, `.40`) để mô phỏng các nhóm website (General, Job, Social, Restricted).

## Các tình huống đã thực hiện

### TH1 - Tích hợp Active Directory (LDAP) vào pfSense
- Cấu hình LDAP Authentication Server trên pfSense kết nối tới DC01 qua cổng 389.
- Kiểm thử xác thực thành công cho các tài khoản miền `sv_a` và `sv_b` tại mục *Diagnostics > Authentication*.
- Kết quả: **PASS**.

### TH2 - Cấu hình Captive Portal xác thực User Domain
- Tạo Captive Portal Zone (`LAB6_USERS`) gán vào giao diện LAN.
- Tích hợp nguồn xác thực thông qua LDAP Server đã kết nối AD.
- Kiểm thử việc ép buộc đăng nhập tài khoản miền khi client truy cập web.
- Kết quả: **PASS**.

### TH3 - Tạo Aliases và Schedules
- Thiết lập các Alias phân loại máy trạm (`CLIENT_A`, `CLIENT_B`, `POLICY_CLIENTS`) và các website đích trong DMZ (`GENERAL SITE`, `JOB SITE`, `SOCIAL_SITE`, `RESTRICTED SITE`).
- Thiết lập các mốc thời gian (`LUNCH_WINDOW`, `WORK_HOURS`) phục vụ cho các chính sách lọc theo khung giờ.
- Kết quả: **PASS**.

### TH4 - Thực hành các Chính sách truy cập theo Kịch bản
- **Kịch bản 1**: Chặn Client-A truy cập vào Job Site (`172.16.6.20`).
- **Kịch bản 2**: Cấm Restricted Site (`172.16.6.40`) cho cả Client-A và Client-B.
- **Kịch bản 3**: Cấu hình Client-B chỉ được truy cập Social Site (`172.16.6.30`) trong giờ nghỉ trưa (`LUNCH_WINDOW`).
- **Kịch bản 4**: Giới hạn dịch vụ Internet chỉ cho phép HTTP/HTTPS trong giờ hành chính (`WORK_HOURS`).
- **Kịch bản 5**: Cô lập phân vùng DMZ không cho khởi tạo kết nối ngược về mạng LAN.
- **Kịch bản 6 & 7**: So sánh hành vi giữa Block và Reject, đọc Firewall Log và xử lý State Table.
- Kết quả: **PASS**.

## Kết quả tóm tắt

| Kịch bản / Tình huống | Dự đoán | Thực tế | Kết quả |
|---|---|---|---|
| Xác thực LDAP (sv_a / sv_b) | Thành công | Thành công | **PASS** |
| Captive Portal Login | Yêu cầu xác thực | Hoạt động đúng | **PASS** |
| Kịch bản 1: Chặn Job Site | Bị chặn | Bị chặn đúng quy tắc | **PASS** |
| Kịch bản 2: Chấm Restricted Site | Bị chặn | Bị chặn cả A và B | **PASS** |
| Kịch bản 3: Social theo Schedule | Theo khung giờ | Đúng giờ mở, ngoài giờ chặn | **PASS** |
| Kịch bản 4: Giờ hành chính | Chỉ cho phép web | Đúng cấu hình schedule | **PASS** |
| Kịch bản 5: Cô lập DMZ | Chặn chiều DMZ->LAN | Hoạt động ổn định | **PASS** |
| Kịch bản 6 & 7: Log & State | Ghi nhận đầy đủ | Khớp ruleset thực tế | **PASS** |

## Lỗi gặp phải và cách khắc phục

### Lỗi kết nối LDAP Authentication
Ban đầu kiểm tra kết nối LDAP bị lỗi do cấu hình sai `Base DN` và `Authentication Container`.
Đã khắc phục bằng cách điều chỉnh lại chính xác chuỗi `DC=lab6,DC=test` và chỉ định đúng thư mục `OU=LAB6-Users`.

### Lỗi lưu lượng không áp dụng do State cũ
Sau khi thay đổi hoặc thêm rule mới, một số kết nối vẫn thông do tường lửa ghi nhớ trạng thái cũ (*stateful*).
Đã khắc phục bằng cách truy cập `Diagnostics > States > Reset States` và mở phiên kết nối mới trên máy client.

## An toàn và phạm vi thực hành

- Mô hình hoàn toàn cô lập trong hệ thống máy ảo VirtualBox nội bộ.
- Không sử dụng thông tin tài khoản hoặc dữ liệu thật của cá nhân/tổ chức.
- Không truy cập các trang web độc hại thực tế trên Internet trong quá trình thực hành mô phỏng.