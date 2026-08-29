# SSRF: Server-Side Request Forgery

## 1. Concept

**SSRF** là lỗ hổng bảo mật cho phép kẻ tấn công ép máy chủ (Server) thực hiện các yêu cầu HTTP thay cho mình tới các mục tiêu mà bình thường kẻ tấn công không thể tiếp cận trực tiếp.

* **ELI5:** Hãy tưởng tượng Server giống như một nhân viên chuyển phát nhanh. Bạn (Hacker) không thể vào được căn mật thất (Mạng nội bộ/Cloud Metadata), nhưng bạn có thể đưa một bức thư cho nhân viên đó và nói: "Hãy vào mật thất lấy tài liệu này cho tôi". Vì nhân viên đó có thẻ ra vào hợp lệ, căn phòng sẽ mở cửa cho họ mà không hề nghi ngờ.
* **Phân loại:**
  * **Regular SSRF:** Server trả về dữ liệu của mục tiêu ngay trên trình duyệt của hacker.
  * **Blind SSRF:** Server thực hiện yêu cầu nhưng không trả về nội dung. Hacker phải dùng kỹ thuật "gọi ngược" (OOB - Out of Band) để xác nhận.

## 2. Key Mechanism

Cơ chế SSRF dựa trên **Mối quan hệ tin cậy (Trust Relationship)** trong hạ tầng mạng:

1. **Input:** Người dùng cung cấp một URL cho ứng dụng (vd: chức năng dán link ảnh để web tự tải về).
2. **Request:** Backend khởi tạo một kết nối mạng (HTTP Request) từ chính địa chỉ IP của máy chủ Web tới URL đó.
3. **Vulnerability:** Nếu URL là địa chỉ nội bộ (`127.0.0.1` hoặc IP Cloud Metadata `169.254.169.254`), các dịch vụ phía sau sẽ tin tưởng đây là yêu cầu từ "người nhà" và trả về dữ liệu nhạy cảm.
4. **OOB (Out-of-Band):** Với Blind SSRF, hacker dùng các tên miền như `burpcollaborator.net` để kiểm tra xem Server có thực hiện phân giải DNS hoặc kết nối HTTP ra ngoài hay không.

## 3. Security Perspective

### 🔴 Attacker View (Red Team)

Hacker luôn nhắm tới các mục tiêu có giá trị cao nhất qua SSRF:

* **Cloud Metadata Theft:** Truy cập `http://169.254.169.254/` trên AWS/GCP để lấy Access Keys của tài khoản quản trị Cloud.
* **Internal Port Scanning:** Dò quét các cổng mạng nội bộ (Port 3306 - MySQL, 6379 - Redis) để tìm dịch vụ không có mật khẩu.
* **Bypass Filtering:**
  * Dùng **Decimal IP** (`2130706433` thay cho `127.0.0.1`).
  * Dùng **DNS Rebinding** hoặc dịch vụ như `nip.io`.
  * Dùng ký tự @ để lách Allow List: `https://tin-cay.com@hacker.com`.

### 🔵 Defender View (Blue Team/SOC)

SOC Analyst phát hiện SSRF qua việc đối chiếu log **Inbound** và **Outbound**.

* **Dấu hiệu nhận biết (SOC Alert):**
  * **Suspicious Parameters:** Tham số URL chứa IP nội bộ, `localhost`, hoặc `169.254.169.254`.
  * **OOB Interaction:** Log DNS/HTTP ghi nhận máy chủ Web truy cập các tên miền lạ như `*.oastify.com`, `*.interact.sh`.
  * **Abnormal Outbound:** Máy chủ Web tự kết nối đến chính nó trên các cổng lạ hoặc IP Cloud Metadata qua cổng 80.
  * **Open Redirect Chaining:** Một chuỗi log ghi nhận mã `302 Redirect` dẫn máy chủ từ một link "sạch" sang một IP nội bộ.
* **Giả lập dòng Log điển hình (Web Access Log):**

```
# Dấu hiệu lấy trộm Cloud Metadata trên AWS
192.168.1.50 - - [21/May/2024:10:00:01] "GET /proxy?url=http://169.254.169.254/latest/meta-data/iam/security-credentials/ HTTP/1.1" 200 150

# Dấu hiệu quét cổng nội bộ bằng IP Decimal (2130706433 = 127.0.0.1)
192.168.1.50 - - [21/May/2024:10:05:10] "GET /api/fetch?link=http://2130706433:6379 HTTP/1.1" 500 40

# Dấu hiệu dùng kỹ thuật lách dấu @ (Allow list bypass)
192.168.1.50 - - [21/May/2024:10:10:05] "GET /preview?u=https://trusted-domain.com@127.0.0.1/admin HTTP/1.1" 200 2000
```

## 4. Useful Commands

### 🛠️ Các địa chỉ Cloud Metadata nhạy cảm

```bash
# AWS & OpenStack
http://169.254.169.254/latest/meta-data/

# Google Cloud (GCP) - Yêu cầu thêm Header đặc biệt nhưng đôi khi SSRF vẫn lách được
http://metadata.google.internal/computeMetadata/v1/

# Azure
http://169.254.169.254/metadata/instance?api-version=2017-08-01
```

### 🔓 Kỹ thuật Bypass Payload

```
# IP Shorthand
http://0/
http://127.1/

# IP dạng Decimal (Dùng để lách các bộ lọc chữ thô)
http://2130706433/

# Sử dụng dịch vụ DNS động (nip.io, sslip.io)
http://127.0.0.1.nip.io

# URL Encoding (Double Encoding)
%25%31%32%37%25%32%65%25%30%25%32%65%25%30%25%32%65%25%31
```

### 🔒 Remediation (Giải pháp phòng thủ)

1. **Deny List (Không khuyến khích):** Cấm `127.0.0.1`, `localhost`, `169.254.169.254` (Dễ bị lách).
2. **Allow List (Khuyến khích):** Chỉ cho phép kết nối tới các tên miền/IP cụ thể đã được định nghĩa trước.
3. **Disable HTTP Redirections:** Không cho phép thư viện HTTP tự động đi theo mã `302`.
4. **Network Isolation:** Cấu hình tường lửa (Firewall) chặn mọi kết nối **Outbound** từ Web Server tới mạng nội bộ hoặc Cloud Metadata (chỉ mở các cổng thực sự cần thiết).
