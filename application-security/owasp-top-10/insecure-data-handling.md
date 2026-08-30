# Insecure Data Handling

## 1. Concept

Nhóm lỗ hổng này xảy ra khi ứng dụng xử lý dữ liệu người dùng một cách "ngây thơ", dẫn đến việc lộ lọt thông tin bí mật hoặc vô tình thực thi mã độc của kẻ tấn công.

* **ELI5 - Cryptographic Failures:** Giống như việc bạn viết mật khẩu vào một tờ giấy rồi giấu dưới thảm thay vì cất vào két sắt. Hoặc bạn dùng một loại mật mã quá đơn giản mà trẻ con cũng bẻ khóa được.
* **ELI5 - Injection:** Là khi ứng dụng không phân biệt được đâu là "Dữ liệu để đọc" và đâu là "Câu lệnh để chạy". Hacker gửi một đoạn dữ liệu trông giống như câu lệnh, và ứng dụng "vô tư" mang nó đi thực thi trên hệ điều hành hoặc cơ sở dữ liệu.

## 2. Key Mechanism&#x20;

### A. Cryptographic Failures (Lỗi mật mã)

Lỗ hổng phát sinh khi các tiêu chuẩn mật mã học bị phớt lờ:

1. **Dữ liệu thô (Plaintext):** Lưu trữ thông tin nhạy cảm (mật khẩu, số thẻ) mà không mã hóa/hashing.
2. **Thuật toán yếu:** Sử dụng MD5, SHA-1 cho hashing mật khẩu. Các hàm này quá nhanh, giúp hacker thử hàng tỷ tổ hợp mỗi giây.
3. **Tự chế mật mã (Rolling your own crypto):** Tự viết logic xáo trộn chuỗi thay vì dùng thư viện chuẩn (như OpenSSL, AES).
4. **Thiếu Muối (Salt):** Không dùng chuỗi ngẫu nhiên (Salt) khi hash, dẫn đến việc hai người dùng cùng mật khẩu sẽ có cùng một chuỗi hash trong Database.

### B. Injection (Chèn mã)

Xảy ra do việc **Concatenation** (Ghép chuỗi) dữ liệu thô vào các câu lệnh:

1. **SQLi:** Chèn cú pháp vào câu lệnh Database.
2. **Command Injection:** Chèn ký tự ngắt lệnh (;, &&) vào lệnh hệ điều hành.
3. **SSTI (Server-Side Template Injection):** Chèn cú pháp biểu mẫu (\{{...\}}) vào các công cụ hiển thị giao diện (Jinja2, Twig).
4. **Prompt Injection:** Thao túng tâm lý AI/LLM bằng các câu lệnh gợi ý để vượt qua rào cản bảo mật.

## 3. Security Perspective&#x20;

### 🔴 Attacker View (Red Team)

Hacker sử dụng các kỹ thuật trích xuất dữ liệu và thực thi mã từ xa (**RCE**):

* **Exploiting Crypto:** Tải Database về và dùng **Hashcat** để bẻ khóa các hash yếu. Tìm kiếm các khóa bí mật bị lộ trong file cấu hình hoặc mã nguồn trên GitHub.
* **Exploiting Injection:** Dùng kỹ thuật **Fuzzing** (gửi hàng loạt ký tự đặc biệt) để xem phản ứng của hệ thống. Sử dụng các Payload như \{{7\*7\}} để xác nhận SSTI hoặc ; whoami để xác nhận Command Injection.

### 🔵 Defender View (Blue Team/SOC)

SOC Analyst giám sát hành vi bất thường ở cả tầng ứng dụng và tiến trình hệ điều hành.

* **Dấu hiệu nhận biết (SOC Alert):**
  * **Inbound Signatures:** WAF kích hoạt cảnh báo khi thấy các chuỗi đặc trưng: \{{, proc/, UNION SELECT, etc/passwd.
  * **Process Monitoring (EDR):** Phát hiện tiến trình Web (nginx, php, python) sinh ra các tiến trình con là Shell (sh, bash, cmd). Đây là bằng chứng **RCE** thành công qua Injection.
  * **Audit Database:** Phát hiện mật khẩu lưu trong DB có độ dài 32 ký tự (MD5) hoặc 40 ký tự (SHA-1) - dấu hiệu của thuật toán yếu.
  * **Credential Leakage:** Phát hiện thông tin đăng nhập/mật khẩu xuất hiện dưới dạng Plaintext trong **Application Logs** (thường do chế độ Debug chưa tắt).
* **Giả lập dòng Log điển hình (WAF & EDR):**

```
# Phát hiện SSTI Payload qua tham số URL (WAF Log)
192.168.1.50 - - [27/May/2024:14:00:05] "GET /profile?name={{config.items()}} HTTP/1.1" 200 5000

# Phát hiện thực thi lệnh sau Injection (EDR/Sysmon Event ID 1)
Process: /usr/bin/python3
Parent Process: /usr/sbin/nginx
Command Line: /usr/bin/python3 -c "import os; os.system('cat /etc/passwd')"
=> ALERT: Critical RCE detected via Web Service.
```

## 4. Useful Commands

### 🛠️ Phân tích Mật mã (Cryptography)

```bash
# Kiểm tra loại mã băm (Hash Identification)
hashid "202cb962ac59075b964b07152d234b70"

# Bẻ khóa mật khẩu bằng Hashcat (Dùng wordlist rockyou.txt)
hashcat -m 0 cracked_hashes.txt /usr/share/wordlists/rockyou.txt

# Kiểm tra chứng chỉ SSL/TLS của trang web
openssl s_client -connect google.com:443
```

### 🔓 Payload chèn mã (Injection Payloads)

```
# SSTI (Thử nghiệm tính toán)
{{7*7}}
${7*7}
<%= 7*7 %>

# SQL Injection (Bypass Login)
' OR 1=1 --

# Command Injection (Dò tìm file)
; ls -la /etc/
| cat /etc/passwd
```

### 🔒 Giải pháp phòng thủ (Remediation)

1. **Strong Hashing:** Luôn dùng **bcrypt**, **scrypt** hoặc **Argon2** cho mật khẩu.
2. **Parameterized Queries:** Dùng Prepared Statements để chống SQLi.
3. **Sanitization:** Làm sạch dữ liệu đầu vào, loại bỏ các ký tự điều khiển ({, }, ;, |).
4. **Secrets Management:** Lưu trữ API Key/Mật khẩu trong **Key Vault** (AWS KMS, Azure Key Vault), không viết vào code.
5. **Disable Debug Mode:** Tắt chế độ gỡ lỗi trên môi trường Production để tránh lộ thông tin nhạy cảm qua log.

