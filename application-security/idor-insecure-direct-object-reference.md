# IDOR (Insecure Direct Object Reference)

## 1. Concept

**IDOR** xảy ra khi một ứng dụng cho phép người dùng truy cập trực tiếp vào các đối tượng dữ liệu dựa trên đầu vào do người dùng cung cấp (ID), nhưng lại thiếu bước kiểm tra quyền sở hữu (Authorization) ở phía máy chủ.

* **ELI5:** Hãy tưởng tượng bạn thuê phòng 101 trong một khách sạn. Bạn nhận ra rằng nếu bạn đi tới cửa phòng 102 và vặn tay nắm cửa, nó sẽ mở ra ngay lập tức mà không cần chìa khóa. Khách sạn này bị lỗi IDOR vì họ tin rằng: "Ai biết số phòng thì người đó là chủ nhân của phòng đó".
* **Ví dụ thực tế:** Thay đổi URL từ https://bank.com/statement?id=1001 thành id=1002 để xem trộm sao kê ngân hàng của người khác.

## 2. Key Mechanism

Cơ chế cốt lõi của IDOR là sự sai lệch giữa **Danh tính (Authentication)** và **Quyền hạn (Authorization)**:

1. **Request:** Trình duyệt gửi yêu cầu chứa mã định danh (ID) của tài nguyên (Hóa đơn, File, User Profile).
2. **Processing:** Server nhận ID, truy vấn trực tiếp vào Database để lấy dữ liệu.
3. **Vulnerability:** Server trả dữ liệu về ngay lập tức nếu ID tồn tại, mà **quên không đối chiếu** xem tài khoản đang đăng nhập có phải là chủ sở hữu của ID đó hay không.

**Các biến thể của ID:**

* **Numeric IDs:** Số thứ tự tăng dần (1, 2, 3...). Dễ bị quét (Enumeration) nhất.
* **Encoded IDs:** ID được chuyển sang Base64 để che mắt (vd: id=MQ==).
* **Hashed IDs:** ID được băm (MD5, SHA1) để tăng độ khó (vd: id=c4ca42...).
* **Unpredictable IDs (UUID):** Chuỗi định danh duy nhất dài và khó đoán. Chỉ có thể khai thác nếu bị rò rỉ (Leak).

## 3. Security Perspective

### 🔴 Attacker View (Red Team)

Hacker sử dụng các kỹ thuật tìm IDOR từ đơn giản đến phức tạp:

* **Kỹ thuật 2 tài khoản (Two-Account Technique):** Sử dụng tài khoản A để lấy ID mẫu, sau đó dùng tài khoản B để thử truy cập ID đó. Đây là cách chính xác nhất để xác nhận lỗi.
* **Parameter Mining:** Nhồi thêm các tham số ẩn như ?user\_id=1 hoặc \&admin=true vào các API để tìm kiếm các IDOR ngầm.
* **Encoding/Hashing Analysis:** Nếu thấy chuỗi lạ, hacker sẽ dùng hashid hoặc base64 -d để tìm ra ID gốc bên dưới.
* **JS/Source Mining:** Tìm kiếm các API endpoint ẩn trong file JavaScript có thể chứa lỗ hổng IDOR.

### 🔵 Defender View (Blue Team/SOC)

&#x20;Gói tin IDOR trông rất sạch sẽ (Mã 200 OK). Tuy nhiên, có thể phát hiện qua hành vi (Behavioral Analysis):

* **Dấu hiệu nhận biết (SOC Alert):**
  * **Sequential Access Patterns:** Một IP truy cập liên tiếp hàng trăm tài nguyên (ID 101, 102, 103...) trong thời gian cực ngắn.
  * **Cross-Tenant Access:** (Dành cho Log cao cấp) Phát hiện sự không trùng khớp giữa User\_ID trong Session/Token và Object\_ID trong URL Path hoặc Request Body.
  * **High 404/403 Rate during Fuzzing:** Khi hacker chạy script dò quét qua một dải ID (đặc biệt là Hashed ID), sẽ phát sinh nhiều lỗi do ID không tồn tại hoặc hệ thống có phân quyền một phần.
* **Giả lập dòng Log điển hình (Web Access Log):**

```
# Dấu hiệu Enumeration (Dò quét ID tuần tự)
192.168.50.10 - [19/May/2024:23:45:01] "GET /api/v1/invoice/1001 HTTP/1.1" 200 1500
192.168.50.10 - [19/May/2024:23:45:02] "GET /api/v1/invoice/1002 HTTP/1.1" 200 1500
192.168.50.10 - [19/May/2024:23:45:03] "GET /api/v1/invoice/1003 HTTP/1.1" 200 1500
=> ALERT: Sequential Resource Access from single IP. Potential IDOR Harvesting.

# Dấu hiệu Parameter Mining (Thử nghiệm tham số ẩn)
10.0.0.5 - [19/May/2024:23:50:10] "GET /api/me?user_id=1 HTTP/1.1" 200 450
10.0.0.5 - [19/May/2024:23:50:12] "GET /api/me?account_id=99 HTTP/1.1" 403 220
=> ALERT: Unusual Query Parameters detected on standard endpoint.
```

## 4. Useful Commands

### 🛠️ Dò quét IDOR tự động (FFUF)

```bash
// # Quét ID số tuần tự (1-1000)
# -u: URL, -w: wordlist tạo nhanh bằng lệnh seq
seq 1 1000 > numbers.txt
ffuf -w numbers.txt -u http://example.com/api/v1/invoice/FUZZ -mc 200

# Quét ID đã được Base64 Encode
# Dùng bộ lọc size (-fs) để loại bỏ các trang lỗi có kích thước giống nhau
ffuf -w encoded_ids.txt -u http://example.com/view?id=FUZZ -fs 1240
```

### 🔓 Giải mã & Phân tích ID

```bash
# Giải mã Base64 (Linux)
echo "eyJpZCI6MTB9" | base64 -d

# Nhận diện loại hàm băm (MD5, SHA1...)
hash-identifier "c4ca4238a0b923820dcc509a6f75849b"
```

### 🔒 Giải pháp phòng vệ (Best Practices)

1. **Object-Level Authorization:** Luôn kiểm tra if (invoice.owner\_id == current\_user.id) trước khi trả về dữ liệu.
2. **Use GUID/UUID:** Thay thế ID số bằng chuỗi ngẫu nhiên khó đoán (tăng độ khó cho hacker).
3. **Indirect Reference Maps:** Sử dụng các ID tạm thời chỉ có giá trị trong một phiên làm việc (Session-specific IDs).
