# Burp Suite

## 1. Concept

**Burp Suite** là một nền tảng (Framework) chuyên dụng để kiểm thử bảo mật ứng dụng Web.

* Burp Suite như một "Trạm kiểm soát cửa khẩu" nằm giữa Trình duyệt (người dân) và Web Server (quốc gia). Mọi gói tin (hộ chiếu/hàng hóa) đi qua đều phải dừng lại để trạm này kiểm tra, soi chiếu, thậm chí là sửa đổi nội dung trước khi cho phép đi tiếp.
* **Ví dụ thực tế:** Khi nhấn nút "Mua hàng" với giá 100$, Burp Suite cho phép bạn chặn gói tin đó lại, sửa số 100 thành 1 và gửi lên Server để xem hệ thống có bị lỗi logic thanh toán hay không.

## 2. Key Mechanism

Burp Suite hoạt động dựa trên cơ chế **Man-in-the-Middle (MITM) Proxy**:

1. **Intercept (Chặn bắt):** Chặn đứng Request từ Browser gửi đi.
2. **Analyze & Modify (Phân tích & Sửa đổi):** Người dùng xem nội dung (Headers, Body) và có thể chỉnh sửa tùy ý.
3. **Forward (Chuyển tiếp):** Gửi gói tin đã sửa tới Server.
4. **Response Handling:** Nhận phản hồi từ Server, có thể sửa tiếp Response trước khi trả về cho Browser hiển thị.

**Các thành phần cốt lõi:**

* **Proxy:** Bộ lọc và chặn bắt chính.
* **Repeater:** "Phòng thí nghiệm" để gửi đi gửi lại một Request và thử nghiệm các Payload khác nhau.
* **Intruder:** "Khẩu pháo" tự động hóa (Brute-force, Fuzzing).
* **Decoder:** "Thông dịch viên" để giải mã các định dạng Base64, URL Encode, Hex.
* **Site Map:** "Bản đồ" tự động vẽ lại cấu trúc toàn bộ Website dựa trên các trang bạn đã đi qua.

## 3. Security Perspective

### 🔴 Attacker View (Red Team)

Hacker sử dụng Burp Suite để phá vỡ các lớp bảo vệ phía Client:

* **Bypass Client-side Validation:** Bỏ qua các kiểm tra bằng JavaScript trên trình duyệt bằng cách sửa dữ liệu trực tiếp tại Proxy.
* **Enumeration & Reconnaissance:** Dùng Site Map để tìm các file ẩn, folder nhạy cảm (/admin, .env, .git).
* **Exploitation:** Dùng Intruder để dò mật khẩu (Brute-force) hoặc dùng Repeater để khai thác sâu các lỗi SQL Injection, IDOR (thay đổi id=user thành id=admin).

### 🔵 Defender View (Blue Team/SOC)

SOC Analyst phát hiện sự hiện diện của Burp Suite thông qua các dấu vết (Artifacts) đặc trưng:

* **Dấu hiệu nhận biết trên Network/Server Log:**
  * **User-Agent đặc trưng:** Mặc dù có thể đổi, nhưng mặc định Burp đôi khi để lộ thông tin trong Header.
  * **Tốc độ Request cực cao:** Khi hacker dùng Intruder (bản Pro), log sẽ ghi nhận hàng nghìn request/giây từ 1 IP tới 1 Endpoint duy nhất.
  * **Mẫu lỗi (Error Patterns):** Hàng loạt mã 401 Unauthorized liên tiếp, sau đó là một mã 200 OK -> Dấu hiệu Brute-force thành công.
  * **Cảnh báo SSL/TLS:** Log của Server/WAF có thể ghi nhận "SSL Handshake Failed" nếu hacker chưa cài đặt CA Certificate của Burp đúng cách.
* **Dấu hiệu trên Endpoint (Máy trạm):**
  * Sự xuất hiện của tiến trình Java chạy cổng **8080** (mặc định của Burp).
  * Trình duyệt cài Extension **FoxyProxy**.
  * Chrome/Firefox được chạy với flag --no-sandbox (Cực kỳ nguy hiểm).
* **Ví dụ dòng Log tấn công IDOR bị phát hiện:**

```
# SOC thấy 1 IP đang duyệt ID người dùng liên tục theo thứ tự
10.0.0.15 - [15/Oct/2023:14:05:01] "GET /api/profile?id=1001 HTTP/1.1" 200
10.0.0.15 - [15/Oct/2023:14:05:02] "GET /api/profile?id=1002 HTTP/1.1" 200
10.0.0.15 - [15/Oct/2023:14:05:03] "GET /api/profile?id=1003 HTTP/1.1" 200
=> Alert: Insecure Direct Object Reference (IDOR) Attempt detected.
```

## 4. Useful Commands

### Cấu hình kết nối (Proxy Settings)

* **Address:** 127.0.0.1 (Localhost)
* **Port:** 8080 (Mặc định)
* **CA Certificate Download:** Truy cập http://burp sau khi đã cấu hình Proxy để tải chứng chỉ.

### Phím tắt&#x20;

| Phím tắt         | Chức năng                              |
| ---------------- | -------------------------------------- |
| Ctrl + R         | Gửi Request hiện tại sang **Repeater** |
| Ctrl + I         | Gửi Request hiện tại sang **Intruder** |
| Ctrl + U         | URL Encode đoạn text được chọn         |
| Ctrl + Shift + P | Chuyển nhanh tới tab **Proxy**         |
| Ctrl + F         | Tìm kiếm trong lịch sử HTTP            |

### Kiểm tra trạng thái Proxy trên máy (Linux/macOS)

```bash
# Kiểm tra xem Burp Suite có đang lắng nghe ở cổng 8080 không
netstat -tuln | grep 8080

# Hoặc dùng lsof
lsof -i :8080
```

### Cấu hình Chrome (dành cho Lab)

```bash
# Chạy Chrome bỏ qua Sandbox (Cảnh báo: Chỉ dùng trong môi trường Lab an toàn)
google-chrome --no-sandbox --proxy-server="http://127.0.0.1:8080"
```

