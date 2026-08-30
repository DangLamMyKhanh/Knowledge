# IAAA Failures

## 1. Concept

**IAAA** là mô hình quản lý truy cập 4 lớp, đảm bảo mọi thực thể tương tác với hệ thống đều được kiểm soát chặt chẽ.

* **ELI5:** Hãy tưởng tượng quy trình tại một ngân hàng:
  1. **Identification (Định danh):** Bạn đến quầy và nói "Tôi tên là An" (Username).
  2. **Authentication (Xác thực):** Nhân viên yêu cầu bạn xuất trình CCCD (Mật khẩu/MFA) để chứng minh bạn đúng là An.
  3. **Authorization (Phân quyền):** Sau khi xác nhận bạn là An, ngân hàng kiểm tra xem bạn được làm gì (An chỉ được rút tiền của An, không được xem sổ sách của ngân hàng).
  4. **Accountability (Trách nhiệm giải trình):** Mọi hành động của An đều được camera ghi lại và lưu vào sổ nhật ký (Logging) để đối soát khi có sự cố.
* **Hậu quả khi thất bại:** Hacker có thể mạo danh, chiếm quyền Admin, hoặc thực hiện hành vi xấu mà không để lại dấu vết.

## 2. Key Mechanism

Hệ thống IAAA vận hành theo một chuỗi tuyến tính không được phép ngắt quãng:

1. **Identification & Authentication (A07:2021 - Identification and Authentication Failures):**
   * Hệ thống so khớp thông tin người dùng cung cấp với cơ sở dữ liệu.
   * **Lỗi Case-Insensitivity:** Lập trình viên dùng mã nguồn phân biệt chữ hoa/thường (admin khác Admin), nhưng Database lại dùng **SQL Collation** không phân biệt (như utf8\_general\_ci), dẫn đến việc hacker đăng ký aDmiN để chiếm quyền của admin.
2. **Authorization (A01:2021 - Broken Access Control):**
   * Kiểm tra quyền hạn tại **Server-side** cho mỗi yêu cầu.
   * **Horizontal Escalation (Ngang):** Truy cập dữ liệu của user khác có cùng cấp bậc (IDOR).
   * **Vertical Escalation (Dọc):** User thường thực thi chức năng của Admin (truy cập /admin/panel).
3. **Accountability (A09:2021 - Security Logging and Monitoring Failures):**
   * Ghi lại dấu vết (Audit Trails). Nếu thiếu lớp này, SOC sẽ bị "mù" khi điều tra sự cố (Incident Response).

## 3. Security Perspective

### 🔴 Attacker View (Red Team)

Hacker tấn công vào các "mắt xích" yếu nhất trong chuỗi IAAA:

* **Exploiting Authentication:** Dùng các biến thể chữ hoa/thường để ghi đè tài khoản Admin hoặc Brute-force các tài khoản có mật khẩu yếu/mặc định.
* **Exploiting Authorization:** Thay đổi các ID trên URL hoặc trong Cookie để xem trộm dữ liệu người khác. Thử truy cập các đường dẫn ẩn dành cho quản trị viên.
* **Exploiting Accountability:** Sau khi xâm nhập, hành động đầu tiên của hacker thường là xóa file log (history -c, rm /var/log/auth.log) để ẩn mình.

### 🔵 Defender View (Blue Team/SOC)

SOC Analyst giám sát tính toàn vẹn của mô hình IAAA qua hệ thống SIEM.

* **Dấu hiệu nhận biết (SOC Alert):**
  * **Authorization Violation:** Tần suất mã lỗi HTTP 403 Forbidden tăng vọt từ một User ID (Dấu hiệu đang cố truy cập trái phép).
  * **Authentication Anomaly:** Một tài khoản đăng nhập thành công từ 2 địa chỉ IP cách xa nhau về địa lý trong thời gian ngắn.
  * **Case-Mismatch detection:** Log ghi nhận Username lúc đăng ký là aDmiN nhưng khi truy vấn database lại khớp với admin.
  * **Logging Failures:** Phát hiện dịch vụ ghi log bị tắt đột ngột hoặc file log bị giảm dung lượng bất thường (Dấu hiệu bị xóa log).
* **Giả lập dòng Log điển hình (Web/Audit Log):**

```
# Phát hiện Broken Access Control (403 spike)
10.0.0.5 - [25/May/2024:14:30:05] "GET /admin/delete-user?id=50 HTTP/1.1" 403 220 (User: an_normal)
10.0.0.5 - [25/May/2024:14:30:08] "GET /admin/config HTTP/1.1" 403 220 (User: an_normal)

# Phát hiện Authentication Failure (Case manipulation)
[Auth-Event] Request_User: "aDmiN" | DB_Matched_User: "admin" | Status: Login_Success | UID: 1
=> ALERT: Potential Account Takeover via Case Sensitivity Flaw.
```

## 4. Useful Commands

### 🔎 Kiểm tra Log hệ thống (Linux)

```bash
# Xem các lần đăng nhập thất bại gần nhất
lastb

# Theo dõi log xác thực thời gian thực
tail -f /var/log/auth.log | grep -E "Accepted|Failed"

# Kiểm tra các file log có bị xóa trắng không
ls -lh /var/log/
```

### 🛠️ Kiểm tra cấu hình Database (SQL)

```sql
-- Kiểm tra Collation của bảng users (Nếu có chữ '_ci' là Case-Insensitive - Nguy hiểm)
SHOW FULL COLUMNS FROM users;

-- Truy vấn kiểm tra trùng lặp logic
SELECT username, COUNT(*) FROM users GROUP BY username HAVING COUNT(*) > 1;
```

### 📝 Windows Event IDs

<table data-header-hidden><thead><tr><th width="123.20001220703125"></th><th></th></tr></thead><tbody><tr><td>Event ID</td><td>Ý nghĩa</td></tr><tr><td><strong>4624</strong></td><td>Đăng nhập thành công (Kiểm tra lớp Authentication)</td></tr><tr><td><strong>4625</strong></td><td>Đăng nhập thất bại (Dấu hiệu Brute-force)</td></tr><tr><td><strong>4672</strong></td><td>Đăng nhập với đặc quyền Admin (Kiểm tra lớp Authorization)</td></tr><tr><td><strong>1102</strong></td><td>Nhật ký hệ thống bị xóa (Thất bại ở Accountability - Cảnh báo Đỏ)</td></tr></tbody></table>

