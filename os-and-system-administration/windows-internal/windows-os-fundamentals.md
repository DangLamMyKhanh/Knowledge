# Windows OS Fundamentals

## 1. Concept

Để bảo vệ một ngôi nhà, bạn phải hiểu rõ bản vẽ thiết kế của nó. Tương tự, để phát hiện mã độc (Malware) hoặc tin tặc (Hacker) trên môi trường Windows, một SOC Analyst phải nắm vững cấu trúc bên trong của hệ điều hành này.

* **ELI5:** Hệ điều hành Windows giống như một thành phố:
  * **File System (NTFS):** Là quy hoạch đất đai (ai được xây nhà ở đâu, nhà rộng bao nhiêu).
  * **Users & Groups:** Là hộ khẩu (ai là thị trưởng - Admin, ai là dân thường - Standard User).
  * **Registry:** Là cuốn sổ luật pháp của thành phố (mọi quy định, cấu hình đều ghi ở đây).
  * **Services & Scheduled Tasks:** Là các công nhân dọn dẹp và bảo vệ làm việc liên tục cả ngày lẫn đêm.

## 2. Key Mechanism

### A. The File System (NTFS)

Hệ thống quản lý tệp NTFS giải quyết các hạn chế của chuẩn FAT/FAT32 cũ (giới hạn file 4GB). Điểm mạnh nhất của NTFS là tính năng **Permissions (Phân quyền)** ở mức thư mục/tệp và **ADS (Alternate Data Streams)**.

* **System32 (C:\Windows\System32):** Trái tim của Windows, chứa các file thực thi cốt lõi (.exe, .dll). Đường dẫn này được lưu trong biến môi trường %windir%.

### B. User Accounts & UAC

* **Accounts:** Có 2 loại chính là Administrator (Quyền cao nhất, thay đổi hệ thống) và Standard User (Chỉ thay đổi thư mục cá nhân). Hồ sơ (Profile) được lưu tại C:\Users\\\<username>.
* **UAC (User Account Control):** Ngay cả khi bạn là Admin, các ứng dụng bình thường vẫn chạy với quyền thấp. Khi ứng dụng cần thay đổi hệ thống (như cài phần mềm), màn hình sẽ mờ đi (Secure Desktop) và hiển thị hộp thoại cảnh báo (Icon cái khiên). Điều này giúp ngăn chặn mã độc tự động cài đặt ngầm.

### C. Services & Scheduled Tasks

* **Services (services.msc):** Các ứng dụng chạy ngầm không cần giao diện người dùng. Có 3 trạng thái khởi động: Automatic, Manual, Disabled.
* **Task Scheduler:** Công cụ lên lịch tự động chạy một chương trình theo mốc thời gian (ví dụ: 10:00 AM mỗi ngày).

### D. Windows Registry (regedit)

* Đây là cơ sở dữ liệu phân cấp trung tâm lưu trữ toàn bộ cấu hình của hệ điều hành, phần cứng và các ứng dụng. Mọi sự thay đổi trên Windows đều được ghi lại vào đây.

## 3. Security Perspective

Đây là cách Blue Team (SOC) và Red Team (Attacker) nhìn nhận và lợi dụng các cơ chế nội tại của Windows.

### 🔴 Attacker View (Red Team)

Hacker rất thích lạm dụng các tính năng có sẵn của Windows (Kỹ thuật **Living-off-the-Land - LoTL**) để tránh bị Antivirus phát hiện:

1. **Alternate Data Streams (ADS):** Hacker giấu mã độc vào phía sau một file văn bản bình thường (ví dụ: `note.txt:malware.exe`). Nếu nhìn bằng File Explorer thông thường, bạn chỉ thấy file note.txt.
2. **Persistence (Nằm vùng) qua Registry / Task Scheduler / Startup:** Để mã độc tự động chạy lại khi nạn nhân khởi động lại máy, hacker sẽ thêm một Scheduled Task mới, hoặc chèn đường dẫn mã độc vào thư mục `shell:startup`, hoặc ghi vào khóa Registry `Run` / `RunOnce`.
3. **VSS Deletion (Xóa bản sao lưu):** Khi mã độc tống tiền (Ransomware) lây nhiễm, hành động đầu tiên của nó là dùng lệnh vssadmin để xóa toàn bộ **Volume Shadow Copies** (Bản sao lưu hệ thống), khiến nạn nhân không thể khôi phục lại dữ liệu nếu không có backup offline.
4. **System32 Masquerading:** Giấu mã độc vào thư mục `System32` và đặt tên giống hệt các tiến trình hệ thống (như `svchost.exe`, `lsass.exe`) để qua mặt người dùng.

#### 🔵 Defender View (Blue Team/SOC)

SOC Analyst phải giám sát chặt chẽ sự thay đổi của các cấu trúc cốt lõi này bằng **Sysmon** hoặc **Windows Event Viewer**.

* **Dấu hiệu nhận biết (SOC Alerts & Hunting):**
  * **Event ID 4624 / 4625:** Theo dõi log đăng nhập thành công/thất bại (Dấu hiệu Brute-force).
  * **Event ID 4720:** Một tài khoản người dùng mới vừa được tạo. (Hacker thường tạo user mới sau khi xâm nhập để làm Backdoor).
  * **Unusual UAC Bypasses:** WAF/EDR phát hiện tiến trình cố tình vượt qua UAC mà không làm mờ màn hình.
  * **Suspicious AutoStart:** Phát hiện một **Service** mới được tạo ở chế độ `Automatic` hoặc một **Scheduled Task** mới trỏ tới một file .exe / .bat nằm trong thư mục tạm (`C:\Temp` hoặc `C:\Users\...\AppData`).
  * **VSS Deletion:** Hệ thống kích hoạt cảnh báo đỏ (Critical Alert) khi phát hiện tiến trình gọi lệnh `vssadmin.exe Delete Shadows`.

## 4. Useful Commands&#x20;

Dưới đây là các câu lệnh CMD/PowerShell sinh tồn mà một SOC Analyst dùng để **Incident Response (Điều tra sự cố)** trên máy nạn nhân:

### 🔎 Network & Process Hunting

```cmd
:: Kiểm tra các kết nối mạng hiện tại và tiến trình (PID) nào đang mở cổng đó
netstat -ano

:: Xem thông tin chi tiết về mạng (IP, DNS, Gateway)
ipconfig /all

:: Lấy danh sách các tiến trình đang chạy (Giống Task Manager)
tasklist
```

### 👥 User & Privilege Auditing

```cmd
:: Xem danh sách tất cả các user đang có trên máy
net user

:: Kiểm tra xem ai đang nằm trong nhóm Quản trị viên (Phát hiện hacker tự gán quyền)
net localgroup administrators

:: Xem mình đang là ai và có đặc quyền gì
whoami /priv
```

### 📂 System & Persistence Checking

```powershell
# Xem chi tiết các luồng dữ liệu ẩn (ADS) của file để tìm mã độc giấu mình
Get-Item -Path C:\file.txt -Stream *

# Mở thư mục chứa các ứng dụng tự khởi động cùng hệ thống
# (Nhấn Win+R và gõ lệnh sau)
shell:startup
```

### 🛠️ Các công cụ GUI thiết yếu (Run Dialog)

* `lusrmgr.msc`: Quản lý User và Group cục bộ.
* `msconfig`: Cấu hình Boot và kiểm tra hệ thống khởi động.
* `services.msc`: Kiểm tra các ứng dụng chạy ngầm (Services).
* `compmgmt.msc`: Mở Computer Management (Chứa Task Scheduler và Event Viewer).
* `resmon`: Resource Monitor (Phân tích sâu về CPU, RAM, Disk, Network của từng tiến trình).
