# Subdomain Enumeration: Mapping the Attack Surface

## 1. Concept

**Subdomain Enumeration** là quá trình tìm kiếm tất cả các tên miền phụ (subdomain) thuộc về một tên miền chính (root domain).

* **ELI5:** Hãy tưởng tượng tên miền chính là một tòa nhà văn phòng lớn. Tên miền phụ là các phòng ban, kho hàng, hoặc lối đi phụ không được ghi trên bản đồ chính thức ở sảnh. Việc tìm ra các "phòng ẩn" này giúp bạn tìm thấy những cửa vào lỏng lẻo hơn (như máy chủ thử nghiệm, trang quản trị cũ).
* **Mục tiêu:** Mở rộng **Attack Surface** (Bề mặt tấn công). Càng tìm được nhiều subdomain, cơ hội tìm thấy lỗ hổng càng cao.

## 2. Key mechanism

Kỹ thuật này được chia làm hai loại chính:

1. **Passive Reconnaissance (Thụ động):** Thu thập thông tin từ bên thứ ba, không tương tác trực tiếp với máy chủ mục tiêu.
   * **CT Logs (Certificate Transparency):** Tra cứu "sổ hộ khẩu" chứng chỉ SSL công khai (ví dụ: crt.sh).
   * **Search Engines:** Dùng Google Dorks để lọc các subdomain đã được lập chỉ mục (index).
   * **Aggregation Tools:** Dùng công cụ gom tin từ nhiều nguồn API (Sublist3r).
2. **Active Reconnaissance (Chủ động):** Tương tác trực tiếp với hạ tầng của mục tiêu.
   * **DNS Bruteforce:** Thử hàng ngàn tên miền phụ phổ biến bằng cách hỏi trực tiếp DNS Server (dnsrecon).
   * **Vhost Fuzzing:** Thử thay đổi **Host Header** trong gói tin HTTP để tìm các trang web chạy chung một địa chỉ IP nhưng không có trên DNS (ffuf).

## 3. Security Perspective

### 🔴 Attacker View (Red Team)

Hacker săn tìm các **Shadow IT** (Thiết bị/Dịch vụ nằm ngoài tầm kiểm soát của đội IT):

* **Staging/Dev environments:** Các subdomain như dev.example.com thường chứa mã nguồn mới, lỗi debug và bảo mật kém hơn trang chính thức.
* **Subdomain Takeover:** Tìm các subdomain trỏ tới các dịch vụ bên thứ ba (như S3, Zendesk) đã bị gỡ bỏ nhưng bản ghi DNS vẫn còn tồn tại để chiếm quyền sở hữu.
* **Hidden Admin Panels:** Các cổng quản trị không công khai (internal-admin.example.com).

### 🔵 Defender View (Blue Team/SOC)

SOC Analyst cần phân biệt rõ giữa các loại dấu vết để xác định mức độ đe dọa:

* **Dấu hiệu nhận biết (SOC Alert):**
  * **DNS NXDOMAIN Storm:** Một IP gửi hàng ngàn yêu cầu DNS liên tục và nhận về mã lỗi NXDOMAIN (tên miền không tồn tại). Đây là dấu hiệu của **DNS Bruteforce**.
  * **Host Header Anomaly:** Web Server nhận hàng loạt Request vào trang chủ (/) nhưng trường Host thay đổi liên tục theo thứ tự từ điển. Đây là dấu hiệu của **Virtual Host Fuzzing**.
  * **Shadow IT Discovery:** Khi quét CT Logs của chính mình, nếu thấy một subdomain lạ vừa được cấp chứng chỉ SSL (như Let's Encrypt), đó có thể là nhân viên đang dựng máy chủ trái phép.
* **Giả lập dòng Log điển hình cho SOC:**

```
# Dấu vết DNS Bruteforce (Phát hiện từ DNS Server Log)
May 17 14:00:01 dns-server named[123]: client @0x7f 10.10.50.15#53: query: mail.example.com IN A -E(0)
May 17 14:00:01 dns-server named[123]: client @0x7f 10.10.50.15#53: query: vpn.example.com IN A -NXDOMAIN
May 17 14:00:02 dns-server named[123]: client @0x7f 10.10.50.15#53: query: test.example.com IN A -NXDOMAIN
=> ALERT: NXDOMAIN Storm detected from IP 10.10.50.15. Potential DNS Bruteforce.

# Dấu vết Virtual Host Fuzzing (Phát hiện từ Web Server Access Log)
10.10.50.15 - - [17/May/2024:14:10:01] "GET / HTTP/1.1" 200 1500 "Host: dev.example.com"
10.10.50.15 - - [17/May/2024:14:10:02] "GET / HTTP/1.1" 404 200 "Host: admin-test.example.com"
10.10.50.15 - - [17/May/2024:14:10:03] "GET / HTTP/1.1" 404 200 "Host: staging.example.com"
=> ALERT: Unusual Host Header variation from IP 10.10.50.15.
```

## 4. Useful Commands

### 🕵️ OSINT & Passive Discovery

```bash
# Sử dụng Sublist3r để gom tin thụ động từ nhiều nguồn (Google, Bing, crt.sh...)
sublist3r -d example.com

# Google Dorking để tìm subdomain (Trừ đi trang www chính thức)
site:*.example.com -site:www.example.com
```

### 🚀 Active Discovery (Bruteforce & Fuzzing)

```bash
# DNS Bruteforce bằng dnsrecon
dnsrecon -t brt -d example.com

# Virtual Host Fuzzing bằng ffuf
# -w: wordlist, -u: URL mục tiêu, -H: Header tùy chỉnh, -fs: lọc bỏ kích thước rác
ffuf -w /usr/share/wordlists/secLists/Discovery/Web-Content/vhosts.txt \
     -u http://10.10.x.x -H "Host: FUZZ.example.com" -fs 612
```

### 🛠️ Linux Networking Tools (SOC)

```bash
# Kiểm tra DNS của một subdomain cụ thể
dig a dev.example.com

# Kiểm tra Verbose kết nối HTTP để xem Host Header
curl -v -H "Host: secret.example.com" http://10.10.x.x

# Kiểm tra các cổng đang lắng nghe (Phát hiện Proxy/Burp)
netstat -tuln | grep 8080
```
