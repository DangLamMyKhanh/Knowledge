# Content Discovery

## 1. Concept

**Content Discovery** là quá trình tìm kiếm các tài nguyên (file, thư mục, tính năng) hiện đang tồn tại trên Web Server nhưng không được hiển thị công khai hoặc không có liên kết (link) dẫn tới từ trang chủ.

* **ELI5:** Giống như việc bạn đi thám hiểm một ngôi nhà. Bạn không chỉ nhìn những gì bày sẵn ở phòng khách, mà còn đi gõ tường tìm hầm bí mật, tìm chìa khóa dưới thảm, hoặc leo lên mái nhà để tìm lối vào khác.
* **Mục tiêu:** Tìm thấy các "kho báu" bị bỏ quên như file sao lưu (backup.zip), file cấu hình (.env), trang quản trị ẩn (/admin\_v2), hoặc mã nguồn cũ.

## 2. Key mechanismm

Có 3 phương pháp chính để khám phá nội dung:

1. **Manual Discovery (Thủ công):**
   * Kiểm tra các file tiêu chuẩn: robots.txt, sitemap.xml, favicon.ico.
   * Đọc Page Source và sử dụng Developer Tools.
2. **OSINT (Tình báo nguồn tin công khai):**
   * **Google Dorking:** Sử dụng các toán tử tìm kiếm nâng cao (Search Operators) để lọc dữ liệu từ Google.
   * **Wayback Machine:** Xem lại các phiên bản cũ của website để tìm các đường dẫn đã bị xóa.
   * **GitHub/S3 Buckets:** Tìm kiếm mã nguồn hoặc dữ liệu bị lộ trên các nền tảng lưu trữ đám mây.
3. **Automated Discovery (Tự động hóa):**
   * Sử dụng **Wordlists** (danh sách từ điển) và các công cụ "vét cạn" (Brute-force) để thử hàng ngàn đường dẫn trong thời gian ngắn.

## 3. Security Perspective

### 🔴 Attacker View (Red Team)

Hacker cố gắng mở rộng **Attack Surface** (Bề mặt tấn công) bằng cách:

* Tìm file .git để tải toàn bộ mã nguồn website.
* Tìm file .env hoặc config.php để lấy mật khẩu Database hoặc API Keys.
* Dò tìm Subdomains (dev.example.com, test.example.com) - nơi thường có bảo mật lỏng lẻo hơn trang chính.

### 🔵 Defender View (Blue Team/SOC)

Content Discovery là một trong những hành động dễ bị phát hiện nhất nếu hacker không cẩn thận (đặc biệt là khi dùng công cụ tự động).

* **Dấu hiệu nhận biết (SOC Alert):**
  * **404 Error Spike:** Một địa chỉ IP phát sinh hàng nghìn lỗi 404 Not Found trong vài phút. Đây là dấu hiệu của việc chạy công cụ dò thư mục (Directory Brute-force).
  * **Suspicious User-Agents:** Sự xuất hiện của các chuỗi định danh như gobuster, ffuf, python-requests trong Web Log.
  * **Sensitive File Access:** Bất kỳ Request nào tới các file như .env, web.config, backup.sql đều phải được đánh dấu là **High Severity Alert**.
* **Giả lập dòng Log điển hình (Web Access Log):**

```
# IP lạ đang chạy Gobuster với User-Agent mặc định
10.10.20.5 - - [17/May/2024:10:00:01] "GET /admin HTTP/1.1" 403 220 "-" "gobuster/3.1.0"
10.10.20.5 - - [17/May/2024:10:00:02] "GET /backup HTTP/1.1" 404 150 "-" "gobuster/3.1.0"
10.10.20.5 - - [17/May/2024:10:00:03] "GET /config HTTP/1.1" 404 150 "-" "gobuster/3.1.0"
=> SOC ACTION: Block IP 10.10.20.5 do hành vi Directory Scanning.
```

## 4. Useful Commands

### 🛠️ Công cụ dò tìm tự động (Automated Discovery)

```bash
# Sử dụng Gobuster để dò thư mục (Cơ bản)
gobuster dir -u http://10.10.x.x -w /usr/share/wordlists/dirb/common.txt

# Sử dụng FFUF để dò thư mục với tốc độ cao
ffuf -u http://10.10.x.x/FUZZ -w /usr/share/wordlists/dirb/common.txt

# Dò tìm file có đuôi cụ thể (vd: .php, .txt, .bak)
gobuster dir -u http://10.10.x.x -w wordlist.txt -x php,txt,bak
```

### 🌍 Google Dorks (OSINT)

<table data-header-hidden><thead><tr><th width="198.4000244140625"></th><th></th></tr></thead><tbody><tr><td><strong>Dork</strong></td><td><strong>Ý nghĩa</strong></td></tr><tr><td>site:example.com</td><td>Giới hạn tìm kiếm trong tên miền mục tiêu</td></tr><tr><td>filetype:log</td><td>Tìm các file nhật ký hệ thống bị lộ</td></tr><tr><td>intitle:"index of"</td><td>Tìm các thư mục bị lỗi Directory Listing</td></tr><tr><td>inurl:admin</td><td>Tìm các trang có chứa chữ "admin" trên URL</td></tr></tbody></table>

### 🔍 Lệnh CURL cho SOC & Pentest

```bash
# Xem thông tin Header chi tiết (Verbose mode)
curl -v http://10.10.x.x

# Lấy mã băm MD5 của Favicon để tìm các trang web dùng chung công nghệ
curl https://example.com/favicon.ico | md5sum

# Tải file và giữ nguyên tên trên Server
curl -O http://10.10.x.x/secret_file.txt
```

### 💻 PowerShell (Dành cho Windows SOC)

```powershell
# Tải file bằng PowerShell (Bypass một số bộ lọc cơ bản)
Invoke-WebRequest -Uri "http://10.10.x.x/file.txt" -OutFile "file.txt"

# Tính mã băm file để kiểm tra tính toàn vẹn
Get-FileHash .\file.txt -Algorithm MD5
```
