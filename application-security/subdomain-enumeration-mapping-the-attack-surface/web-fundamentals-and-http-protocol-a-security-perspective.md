# Web Fundamentals & HTTP Protocol: A Security Perspective

## 1. Concept

* **Front End (Client-Side):** Là "gương mặt" của web (HTML/CSS/JS). Mọi thứ ở đây đều nằm trên máy của người dùng, nghĩa là **Hacker có toàn quyền kiểm soát và thay đổi nó**.
* **Back End (Server-Side):** Là "bộ não" xử lý logic và lưu trữ dữ liệu. Đây là mục tiêu cuối cùng mà Hacker muốn xâm nhập.

**Ví dụ thực tế:** Hãy tưởng tượng Website là một nhà hàng.

* **HTML** là khung nhà và thực đơn.
* **JS** là nhân viên phục vụ (tương tác với khách).
* **HTTP** là ngôn ngữ để khách gọi món và nhà hàng trả lời.
* **Cookies** là thẻ thành viên để nhà hàng biết bạn là ai trong lần tới.

## 2. Key Mechanism

Quá trình một yêu cầu Web diễn ra theo mô hình **Request - Response**:

1. **User Request:** Người dùng nhập URL, Browser tạo gói tin HTTP Request (chứa Method, Headers, và Data).
2. **Server Processing:** Server nhận gói tin, phân tích Headers, kiểm tra Cookies/Session để xác định danh tính.
3. **Data Exchange:** Server truy vấn Database (nếu cần) và tạo gói tin HTTP Response.
4. **Browser Rendering:** Trình duyệt nhận Response, đọc HTML/CSS để vẽ giao diện và thực thi JavaScript để tạo tính năng tương tác.

## 3. Security Perspective

### 🔴 Attacker View (Red Team)

Hacker lợi dụng sự tin tưởng của trình duyệt và sự bất cẩn của lập trình viên:

* **Information Gathering:** Sử dụng F12 (View Source) hoặc GoBuster để tìm **Sensitive Data Exposure** (Comments chứa pass, API Keys, hoặc file .bak, .config ẩn qua mã 200 OK).
* **Manipulation:** Dùng **Burp Suite** để bắt gói tin HTTP và thay đổi các giá trị:
  * Sửa Cookie: user=member thành user=admin (Privilege Escalation).
  * Chèn mã HTML vào các ô nhập liệu (HTML Injection) để thay đổi giao diện hoặc lừa đảo.
* **Exploitation:** Thử nghiệm các ký tự đặc biệt như ' hoặc \<script> vào **Query String** hoặc **POST Body** để tìm lỗi SQLi/XSS nếu Server trả về mã 500 Internal Server Error.

### 🔵 Defender View (Blue Team/SOC)

SOC Analyst giám sát lưu lượng qua Web Server Log hoặc WAF (Web Application Firewall).

* **Dấu hiệu nhận biết tấn công:**
  * **Brute-force/Directory Scanning:** Một IP gửi hàng ngàn Request trong thời gian ngắn, phần lớn trả về mã 404 Not Found hoặc 403 Forbidden.
  * **Insecure Header:** Phát hiện Response thiếu các Header bảo mật như HttpOnly, Secure trong Set-Cookie (nguy cơ Session Hijacking).
  * **Payload Injection:** Log ghi lại các ký tự lạ trong URL/Body.
* **Giả lập dòng Log điển hình (Nginx/Apache):**

```apache
# Phát hiện dấu hiệu HTML Injection/XSS trong GET request
192.168.1.50 - - [15/Oct/2023:10:30:05] "GET /search?q=<h1>Hacked</h1><script>alert(1)</script> HTTP/1.1" 200 1543 "-" "Mozilla/5.0..."

# Phát hiện dấu hiệu dò tìm file nhạy cảm (Directory Traversal)
192.168.1.50 - - [15/Oct/2023:10:31:10] "GET /../../etc/passwd HTTP/1.1" 403 221 "-" "BurpSuite/2023.1"
```

## 4. Useful Commands

### Kiểm tra Header và Banner Grabbing

Dùng để xem thông tin Server và các cấu hình Header nhạy cảm.

```bash
# Kiểm tra HTTP Header của một trang web
curl -I https://example.com

# Kiểm tra đầy đủ cả Request và Response (Verbose mode)
curl -v https://example.com

# Banner Grabbing bằng Nmap để xác định phiên bản Server (Fingerprinting)
nmap -sV -p 80,443 --script=http-headers <target_ip>
```

### Dò tìm thư mục ẩn (Directory Brute-force)

```bash
# Sử dụng GoBuster để tìm các file nhạy cảm như .env, config.php, admin
gobuster dir -u http://example.com -w /usr/share/wordlists/dirb/common.txt
```

### JavaScript Debugging (Dùng tại Console F12)

```javascript
// Kiểm tra tất cả cookie hiện có của trang web (Nếu không có HttpOnly)
console.log(document.cookie);

// Kiểm tra các biến toàn cục để tìm thông tin nhạy cảm
console.log(window);
```

### Bảng mã HTTP Status Code cho SOC:

<table data-header-hidden><thead><tr><th width="124"></th><th></th></tr></thead><tbody><tr><td><strong>Status Code</strong></td><td><strong>Ý nghĩa bảo mật</strong></td></tr><tr><td><strong>200 OK</strong></td><td>Truy cập thành công. Nếu xuất hiện sau một chuỗi 404 -> Đã tìm thấy file ẩn.</td></tr><tr><td><strong>301/302</strong></td><td>Redirect. Kiểm tra Header Location để tránh Open Redirect (lừa đảo).</td></tr><tr><td><strong>401/403</strong></td><td>Unauthorized/Forbidden. Dấu hiệu của việc truy cập trái phép.</td></tr><tr><td><strong>404</strong></td><td>Not Found. Quá nhiều mã này từ 1 IP = Đang bị Scan thư mục.</td></tr><tr><td><strong>500</strong></td><td>Server Error. Payload tấn công (SQLi) có thể đã làm lỗi Logic code.</td></tr></tbody></table>
