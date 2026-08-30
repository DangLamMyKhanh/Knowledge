# Web Attack Signatures

**Attack Signature (Chữ ký tấn công)** là những chuỗi ký tự, mẫu dữ liệu (patterns), hoặc hành vi đặc trưng bị hệ thống giám sát (WAF, Web Server, SIEM) ghi lại khi hacker thực hiện một cuộc tấn công.

* **ELI5:** Giống như hiện trường một vụ án. Nếu bạn thấy kính vỡ và một chiếc xà beng, bạn biết đó là vụ đột nhập (Command Injection). Nếu bạn thấy một tờ giấy giả mạo chữ ký, bạn biết đó là lừa đảo (XSS/CSRF).
* **Phân loại:**
  * String-based Signatures: Dấu hiệu tĩnh dựa trên ký tự (vd: \<script>, UNION SELECT).
  * Behavioral Signatures: Dấu hiệu động dựa trên hành vi (vd: 1000 request/giây, quét ID tuần tự).

1. Hacker gửi gói tin HTTP (GET/POST) lên máy chủ.
2. WAF quét qua URL (Query String), Headers (User-Agent, Cookie) và Body (Data).
3. Nếu dữ liệu khớp với một **Signature** có sẵn trong cơ sở dữ liệu của WAF (ví dụ: phát hiện cụm 1=1), WAF sẽ chặn gói tin (Block) và ghi Log cảnh báo (Alert).

Các hệ thống phòng thủ như **WAF (Web Application Firewall)** hoặc **IDS (Intrusion Detection System)** hoạt động dựa trên cơ chế **Regex (Regular Expression - Biểu thức chính quy)** để so khớp dữ liệu đầu vào.

## 1. Dấu hiệu SQL Injection (SQLi)

Mục tiêu: Đánh cắp dữ liệu, vượt quyền đăng nhập.

<table><thead><tr><th width="184.800048828125">Signature (Dạng thô)</th><th>Dạng mã hóa URL (URL Encoded)</th><th>Ý nghĩa / Mục đích</th></tr></thead><tbody><tr><td>' OR 1=1 --</td><td>%27%20OR%201%3D1%20--</td><td>Authentication Bypass (Bypass Login).</td></tr><tr><td>UNION SELECT</td><td>UNION%20SELECT</td><td>Hút dữ liệu từ bảng khác (Union-based).</td></tr><tr><td>information_schema</td><td>information_schema</td><td>Dò tìm tên Database, Table, Column.</td></tr><tr><td>WAITFOR DELAY / SLEEP()</td><td>SLEEP%2810%29</td><td>Blind SQLi (Time-based) - Ép DB ngủ.</td></tr><tr><td>; hoặc -- hoặc #</td><td>%3B / %2D%2D / %23</td><td>Ngắt câu lệnh SQL gốc, chèn bình luận.</td></tr></tbody></table>

## 2. Dấu hiệu Cross-Site Scripting (XSS)

Mục tiêu: Thực thi mã độc trên trình duyệt nạn nhân, trộm Cookie.

<table><thead><tr><th width="177.5999755859375">Signature (Dạng thô)</th><th width="178.20001220703125">Dạng mã hóa URL (URL Encoded)</th><th>Ý nghĩa / Mục đích</th></tr></thead><tbody><tr><td>&#x3C;script></td><td>%3Cscript%3E</td><td>Chèn thẻ kịch bản HTML trực tiếp.</td></tr><tr><td>javascript:</td><td>javascript%3A</td><td>Kích hoạt script qua đường link (href).</td></tr><tr><td>onerror= / onload=</td><td>onerror%3D</td><td>Bypass bộ lọc không dùng thẻ script (dùng thẻ IMG).</td></tr><tr><td>document.cookie</td><td>document.cookie</td><td>Lệnh đánh cắp Session Cookie.</td></tr><tr><td>alert(1) / prompt(1)</td><td>alert%281%29</td><td>Lệnh kiểm thử (PoC) xem web có dính XSS không.</td></tr></tbody></table>

## 3. Dấu hiệu Path Traversal & File Inclusion (LFI/RFI)

Mục tiêu: Đọc file hệ thống, thực thi mã độc từ xa.

<table><thead><tr><th width="194.39996337890625">Signature (Dạng thô)</th><th>Dạng mã hóa URL (URL Encoded)</th><th>Ý nghĩa / Mục đích</th></tr></thead><tbody><tr><td>../ hoặc ..\</td><td>%2E%2E%2F / %2E%2E%5C</td><td>Đi lùi thư mục (Path Traversal).</td></tr><tr><td>....//</td><td>%2E%2E%2E%2E%2F%2F</td><td>Bypass bộ lọc xóa chuỗi ../ một lần.</td></tr><tr><td>/etc/passwd (Linux)</td><td>%2Fetc%2Fpasswd</td><td>File chứa danh sách người dùng hệ thống.</td></tr><tr><td>win.ini (Windows)</td><td>win.ini</td><td>File cấu hình hệ thống Windows.</td></tr><tr><td>http:// (trong tham số)</td><td>http%3A%2F%2F</td><td>RFI: Ép máy chủ tải file từ URL bên ngoài.</td></tr><tr><td>php://filter/</td><td>php%3A%2F%2Ffilter%2F</td><td>Đọc mã nguồn PHP dưới dạng Base64 (LFI).</td></tr></tbody></table>

## 4. Dấu hiệu Command Injection (RCE)

Mục tiêu: Thực thi lệnh hệ điều hành trực tiếp trên máy chủ.

<table><thead><tr><th width="148">Signature (Dạng thô)</th><th width="236.20001220703125">Dạng mã hóa URL (URL Encoded)</th><th>Ý nghĩa / Mục đích</th></tr></thead><tbody><tr><td>; / &#x26;&#x26; / |</td><td>%3B / %26%26 / %7C</td><td>Ký tự ngắt/nối lệnh hệ điều hành.</td></tr><tr><td><code>whoami</code> / $(id)</td><td>%60whoami%60</td><td>Khai thác Command Injection ẩn.</td></tr><tr><td>cat /etc/shadow</td><td>cat%20%2Fetc%2Fshadow</td><td>Lệnh đọc file chứa mật khẩu Linux.</td></tr><tr><td>wget / curl</td><td>wget / curl</td><td>Máy chủ Web tự tải công cụ/mã độc từ ngoài về.</td></tr><tr><td>nc -e /bin/sh</td><td>nc%20-e%20%2Fbin%2Fsh</td><td>Lệnh tạo Reverse Shell về máy hacker.</td></tr></tbody></table>

## 5. Dấu hiệu Server-Side Request Forgery (SSRF)

Mục tiêu: Ép máy chủ Web kết nối tới mạng nội bộ hoặc Cloud Metadata.

<table><thead><tr><th width="184.79998779296875">Signature (Dạng thô)</th><th width="185.39996337890625">Dạng mã hóa URL (URL Encoded)</th><th>Ý nghĩa / Mục đích</th></tr></thead><tbody><tr><td>127.0.0.1 / localhost</td><td>127.0.0.1</td><td>Ép Web Server tự kết nối tới chính nó (Cổng nội bộ).</td></tr><tr><td>169.254.169.254</td><td>169.254.169.254</td><td>Cực kỳ nguy hiểm! Đánh cắp AWS/GCP Cloud Metadata.</td></tr><tr><td>2130706433</td><td>2130706433</td><td>Bypass IP bằng dạng Decimal (Tương đương 127.0.0.1).</td></tr><tr><td>.burpcollaborator.net</td><td>.burpcollaborator.net</td><td>Dấu vết hacker dùng công cụ OOB để dò Blind SSRF.</td></tr></tbody></table>

## 6. Behavioral Signatures (Dấu hiệu Hành vi)

Không có từ khóa cụ thể, phát hiện qua thống kê lưu lượng mạng (Rate/Frequency).

* **Brute-Force / Credential Stuffing:** Hàng trăm mã HTTP 200 OK (hoặc 401) từ 1 IP nhắm vào /login trong 1 phút, sau đó xuất hiện 1 mã HTTP 302 Found.
* **Directory Scanning (Gobuster/FFUF):** Hàng ngàn mã HTTP 404 Not Found từ 1 IP nhắm vào các thư mục khác nhau.
* **IDOR / BOLA:** Một Session ID gọi liên tiếp các tham số tăng dần: ?id=101, ?id=102, ?id=103 (Thường trả về mã 200 OK).
* **Race Condition (TOCTOU):** 10-50 request giống hệt nhau (cùng Cookie, cùng Data) gửi đến cùng một endpoint (như /withdraw) với Timestamp lệch nhau < 50 mili-giây.

## Useful Commands

### 🔎 Threat Hunting trên SIEM (Splunk SPL)

```splunk
# 1. Săn lùng SQL Injection (Tìm các ký tự đặc trưng trong URL)
index=web_logs sourcetype=access_combined 
| search uri_query="*UNION*" OR uri_query="*SELECT*" OR uri_query="*1=1*" OR uri_query="*%27*"

# 2. Phát hiện Path Traversal & LFI
index=web_logs 
| search uri_query="*../*" OR uri_query="*..%2F*" OR uri_query="*/etc/passwd*" 
| stats count by client_ip, uri_path 
| where count > 5

# 3. Cảnh báo ĐỎ: Cloud Metadata SSRF Attempt
index=web_logs 
| search uri_query="*169.254.169.254*"
| eval severity="CRITICAL"

# 4. Săn lùng Command Injection (RCE) - Tìm lệnh Linux bị tuồn vào tham số
index=web_logs
| regex uri_query="(?i)(whoami|cat\s+/etc|wget|curl|nc\s+-e|/bin/sh)"

# 5. Phát hiện Tool Scanning (Gobuster/Nmap qua User-Agent)
index=web_logs
| search http_user_agent="*gobuster*" OR http_user_agent="*sqlmap*" OR http_user_agent="*nmap*"
```
