# Walking an Application: Web Reconnaissance & DevTools

## 1. Concept

**Walking an Application** là giai đoạn trinh sát (Reconnaissance) chủ động. Thay vì sử dụng các công cụ tự động ngay lập tức, người kiểm thử sẽ đóng vai một người dùng cực kỳ "tò mò" để hiểu rõ mọi ngóc ngách của ứng dụng.

* **Mục tiêu:** Xây dựng một "bản đồ tâm trí" (Mental Map) về cách dữ liệu được xử lý và nơi có khả năng xảy ra lỗi.

## 2. key Mechanism

Quy trình này gồm 3 lớp phân tích chính:

1. **Lớp bề mặt (Surface Discovery):** Tìm các điểm tương tác (Form, Button, Upload) và các tham số trên URL (?id=1).
2. **Lớp tĩnh (Static Analysis - View Source):** Đọc mã HTML/JS gốc để tìm bình luận (Comments), liên kết ẩn, và phiên bản các thư viện (Frameworks).
3. **Lớp động (Dynamic Analysis - DevTools):**
   * **Inspector:** Sửa đổi giao diện và cấu trúc DOM để bypass các rào cản hiển thị (Paywall).
   * **Debugger:** Dừng thực thi JavaScript (Breakpoints) để bắt các dữ liệu nhạy cảm xuất hiện tức thời.
   * **Network:** Theo dõi các luồng dữ liệu ngầm (AJAX/XHR) gửi về Server.

## 3. Security Perspective

### 🔴 Attacker View (Red Team)

Hacker tìm kiếm các "điểm yếu chết người" trong logic ứng dụng:

* **Information Disclosure:** Tìm mật khẩu, tên nhân viên, hoặc lộ trình cập nhật trong thẻ \<!-- comment -->.
* **Parameter Enumeration:** Thay đổi id=1 thành id=2, 3, 4... để tìm lỗi **IDOR** (truy cập trái phép dữ liệu người khác).
* **Logic Bypass:** Sử dụng Inspector để chuyển display: none thành display: block nhằm bỏ qua các form đăng ký hoặc thanh toán.
* **Malicious Script Analysis:** Giải mã (De-obfuscate) các đoạn code JS bị làm rối để tìm cách thức ứng dụng mã hóa dữ liệu, từ đó tạo ra các Payload tấn công tương ứng.

### 🔵 Defender View (Blue Team/SOC)

Dưới góc nhìn giám sát, các hành động Reconnaissance này tạo ra những biến động đáng ngờ:

* **Dấu hiệu nhận biết (Indicator of Attack):**
  * **Anomalous Browsing:** Một IP truy cập vào toàn bộ các Endpoint của website (từ Home đến Admin) trong thời gian quá nhanh.
  * **Directory/File Brute-force:** Sự xuất hiện liên tục của mã 404 Not Found hoặc 403 Forbidden khi hacker cố truy cập vào các file nhạy cảm tìm thấy trong Source code (như .env, config.php.bak).
  * **API Abuse:** Các Request AJAX gửi tới Server với tần suất dày đặc hoặc chứa các ký tự lạ (' OR 1=1).
  * **Unauthorized Resource Access:** Truy cập trực tiếp vào các file trong thư mục /assets/ hoặc /uploads/ mà không thông qua giao diện người dùng.
* **Giả lập dòng Log cảnh báo (Web Server Access Log):**

```
# SOC thấy IP lạ đang cố truy cập các file cấu hình nhạy cảm (thường tìm từ Source/Comment)
192.168.10.25 - [16/May/2024:20:15:01] "GET /.git/config HTTP/1.1" 403 220
192.168.10.25 - [16/May/2024:20:15:05] "GET /v1/api/debug_users HTTP/1.1" 200 5120
=> ALERT: Sensitive File Discovery Attempt from IP 192.168.10.25
```

## 4. Useful Commands

### Phím tắt DevTools (Chrome/Firefox/Edge)

* `F12` hoặc `Ctrl + Shift + I`: Mở Developer Tools.
* `Ctrl + Shift + C`: Chọn nhanh một phần tử trên trang để Inspect.
* `Ctrl + U`: Xem mã nguồn trang (View Source).
* Trong tab **Network**: Chuột phải vào một Request -> **Copy** -> **Copy as cURL** (Dùng để đưa vào Burp Suite hoặc Terminal để tái hiện lại gói tin).

### Kiểm tra nhanh thông tin Web bằng CLI

```bash
# Lấy mã nguồn của trang web để tìm kiếm nhanh từ khóa nhạy cảm
curl -s http://example.com | grep -iE "password|admin|config|api_key"

# Kiểm tra robots.txt - nơi chứa các đường dẫn mà chủ web muốn giấu (nhưng hacker lại rất thích)
curl -s http://example.com/robots.txt

# Kiểm tra sitemap.xml để lấy toàn bộ danh sách các URL hiện có
curl -s http://example.com/sitemap.xml
```

### JavaScript Debugging Tips

* Dùng **Pretty Print** (nút { } ở góc dưới tab Debugger) để định dạng lại code đã bị nén (Minified).
* Đặt **XHR Breakpoints**: Trong tab Source/Debugger, bạn có thể đặt lệnh dừng mỗi khi ứng dụng gửi một yêu cầu AJAX đi. Đây là cách tốt nhất để bắt các Flag hoặc Data nhạy cảm.
