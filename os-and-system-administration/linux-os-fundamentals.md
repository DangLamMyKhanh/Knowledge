# Linux OS Fundamentals

## 1. Concept

Linux là hệ điều hành thống trị thế giới Server, Cloud, và các thiết bị IoT. Đối với một chuyên gia bảo mật, Linux tuân theo một triết lý tối thượng: **"Everything is a file" (Mọi thứ đều là tệp tin)**.

* **ELI5:** Từ một file văn bản bình thường, một cổng mạng đang mở, ổ cứng cắm ngoài, cho đến các tiến trình (Process) đang chạy... tất cả đều được Linux quản lý dưới dạng một "File".
* **Ý nghĩa Bảo mật:** Vì mọi thứ là file, nên nếu hacker muốn thay đổi hệ thống (đổi pass, mở cổng mạng, chạy mã độc), BẮT BUỘC phải chạm vào các file. Nhiệm vụ của SOC là biết chính xác file nào cần theo dõi.

## 2. Key Mechanism&#x20;

### A. Kiến trúc thư mục (Filesystem Hierarchy)

Không giống Windows có ổ `C:\` hay `D:\`, Linux gộp chung tất cả vào một cây thư mục duy nhất bắt đầu bằng Root (`/`).

* `/etc`: Chứa toàn bộ file cấu hình của hệ thống (Ví dụ: file cấu hình password, firewall).
* `/var/log`: "Mỏ vàng" của SOC, nơi chứa toàn bộ nhật ký hệ thống.
* `/tmp` và `/dev/shm`: Thư mục lưu trữ tạm thời. Bất cứ ai cũng có quyền ghi vào đây, và dữ liệu sẽ mất khi reset máy.
* `/home/<user>` và `/root`: Thư mục cá nhân của người dùng thường và quản trị viên tối cao (Root).

### B. Quyền hạn (File Permissions)

Linux chia quyền theo 3 nhóm: **Owner** (Chủ sở hữu), **Group** (Nhóm), và **Others** (Những người còn lại). Mọi file đều có 3 quyền cơ bản:

* **Read (r - 4):** Đọc nội dung.
* **Write (w - 2):** Sửa nội dung.
* **Execute (x - 1):** Chạy file như một chương trình.\
  (Ví dụ: Quyền `chmod 777` nghĩa là tất cả mọi người đều có full quyền Đọc/Sửa/Chạy - một lỗ hổng bảo mật chết người).

## 3. Security Perspective

### 🔴 Attacker View (Red Team)

Môi trường Linux là sân chơi lý tưởng của hacker. Khi xâm nhập thành công, chúng sẽ thực hiện:

1. **Hiding in Volatile Memory:** Rất nhiều mã độc được hacker tải thẳng vào thư mục `/tmp` hoặc `/dev/shm` (lưu trên RAM). Vì thư mục này cho phép chạy file tự do và tự xóa dấu vết khi khởi động lại máy.
2. **Persistence (Nằm vùng):** Để duy trì quyền truy cập, hacker sẽ ghi thêm mã độc vào **Cronjobs** (Hệ thống đặt lịch tự động của Linux tại `/etc/crontab`), hoặc thêm Public Key của hacker vào file `~/.ssh/authorized_keys` để đăng nhập không cần pass.
3. **Privilege Escalation (Leo thang đặc quyền):** Hacker tìm kiếm các file thực thi có cờ **SUID** (Cho phép một người dùng thường chạy file đó với quyền của Root) để mượn quyền chiếm máy.
4. **Anti-Forensics:** Xóa lịch sử câu lệnh bằng cách xóa file `~/.bash_history` hoặc gõ `history -c`.

### 🔵 Defender View (Blue Team/SOC)

Khi có sự cố (Incident Response), SOC Analyst lập tức rà quét các thư mục cốt lõi để tìm Dấu hiệu xâm nhập (Indicators of Compromise - IoC).

* **Dấu hiệu nhận biết (SOC Alert & Hunting):**
  * **Brute-Force SSH:** File `/var/log/auth.log` (Ubuntu) hoặc `/var/log/secure` (CentOS) xuất hiện hàng ngàn dòng `Failed password for root`.
  * **Suspicious User Creation:** File `/etc/passwd` đột nhiên xuất hiện một user mới có ID = 0 (Tương đương quyền Root).
  * **Malicious Cronjob:** Lệnh `crontab -l` hiển thị một lịch trình tự động gọi lệnh `curl` hoặc `wget` để tải một file từ IP lạ bên ngoài Internet.
  * **RCE Detection:** Ứng dụng Web (chạy bằng user `www-data`) đột ngột thực thi các lệnh hệ điều hành (như `whoami`, `cat /etc/shadow`).

## 4. Useful Commands

Đây là những câu lệnh sinh tồn (Survival Commands) dùng để phân tích log và rà soát mã độc trên máy chủ Linux bị hack:

### 🔎 1. Log Analysis & Data Extraction&#x20;

```bash
# Xem trực tiếp (Real-time) các nỗ lực đăng nhập sai qua SSH
tail -f /var/log/auth.log | grep "Failed password"

# Lọc nhanh các địa chỉ IP đã cố gắng tấn công Brute-force
grep "Failed password" /var/log/auth.log | awk '{print $11}' | sort | uniq -c | sort -nr

# Xem danh sách những ai đã đăng nhập vào máy tính gần đây
last
```

### 🕸️ 2. Network & Process Hunting

```bash
# Kiểm tra các cổng mạng đang mở và tiến trình (PID) nào đang lắng nghe
netstat -tulpn
# Hoặc dùng lsof để xem có process lạ nào đang kết nối Internet không
lsof -i -P -n

# Tìm kiếm Process ngốn CPU (Thường là mã độc đào coin - Cryptominer)
top -c

# Xem chi tiết một Process (Ví dụ PID là 1234) đang chạy từ đường dẫn nào
ls -l /proc/1234/exe
```

### 👤 3. User & Persistence Auditing

```bash
# Đọc file chứa danh sách tài khoản, tìm xem có User lạ nào có quyền Root (UID=0) không
cat /etc/passwd | grep "x:0:"

# Kiểm tra các tác vụ lên lịch tự động (Cronjobs) của mọi user
cat /etc/crontab
ls -la /etc/cron.*

# Tìm tất cả các file mã độc có khả năng bị giấu trong thư mục /tmp
ls -la /tmp /var/tmp /dev/shm
```
