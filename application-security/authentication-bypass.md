# Authentication Bypass

## 1. Concept

**Authentication Bypass** là kỹ thuật vượt qua cơ chế xác thực của ứng dụng web mà không cần biết thông tin đăng nhập hợp lệ, hoặc sử dụng thông tin đăng nhập của người dùng khác để chiếm quyền điều khiển.

* **ELI5:** Hãy tưởng tượng một tòa nhà có bảo vệ ở cửa. Thay vì trình thẻ nhân viên (mật khẩu), hacker có thể:
  * Hỏi bảo vệ: "Có ai tên là Admin ở đây không?" cho đến khi bảo vệ gật đầu (**Enumeration**).
  * Thử hàng vạn chiếc chìa khóa giả cho đến khi mở được cửa (**Brute Force**).
  * Lừa bảo vệ bằng cách gửi thư xác nhận tới địa chỉ của hacker thay vì địa chỉ của chủ nhà (**Logic Flaw**).
  * Tự tay sửa thông tin trên thẻ nhân viên từ "Khách" thành "Giám đốc" (**Cookie Tampering**).

## 2. Key Mechanism

### A. Username Enumeration (Dò quét tài khoản)

Dựa vào sự khác biệt trong thông báo lỗi của hệ thống (Verbose Errors).

* **Cơ chế:** Thử đăng ký/đăng nhập với một danh sách tên người dùng. Nếu Server phản hồi "User already exists" hoặc "Invalid password" (thay vì "Invalid user"), hacker biết tên người dùng đó có tồn tại.

### B. Brute Force (Tấn công vét cạn)

Sử dụng danh sách tài khoản đã tìm được ở bước trên kết hợp với danh sách mật khẩu (Wordlist) để thử mọi tổ hợp khả thi.

* **Dấu hiệu thành công:** Trong các công cụ như ffuf, đăng nhập sai thường trả về mã 200 OK (kèm thông báo lỗi), trong khi đăng nhập đúng thường trả về mã 302 Found (Redirect vào Dashboard).

### C. Logic Flaw (Lỗi Logic xử lý)

Khai thác cách mã nguồn xử lý dữ liệu không đồng nhất.

* **Ví dụ tiêu biểu:** Tham số email xuất hiện cả ở URL (GET) và Body (POST). Nếu lập trình viên dùng biến $\_REQUEST trong PHP, giá trị trong POST sẽ được ưu tiên, dẫn đến việc link reset mật khẩu được gửi về Email của hacker thay vì của nạn nhân.

### D. Cookie Tampering (Thao túng Cookie)

Sửa đổi các giá trị trong Cookie được lưu tại trình duyệt.

* **Dạng Plaintext/Encoded:** Sửa user\_id=100 thành user\_id=1 hoặc Decode Base64 chuỗi {"role": "user"} thành {"role": "admin"} rồi Encode ngược lại.
* **Dạng Hashing:** Sử dụng các trang như crackstation.net để bẻ khóa các chuỗi MD5/SHA1 đơn giản và thay thế bằng chuỗi hash của quyền cao hơn.

## 3. Security Perspective

### 🔴 Attacker View (Red Team)

Hacker kết hợp nhiều công cụ để tối ưu hóa cuộc tấn công:

* **Tool:** ffuf (để dò quét nhanh), Burp Suite (để sửa đổi Cookie/Params thủ công), Crackstation (bẻ khóa hash).
* **Chiến thuật:** Luôn tìm kiếm **Username Enumeration** trước để thu hẹp phạm vi tấn công **Brute Force**. Tìm kiếm các tham số trùng tên để thử **Parameter Pollution**.

### 🔵 Defender View (Blue Team/SOC)

Đây là loại tấn công tạo ra các mẫu hành vi (Behavioral Patterns) cực kỳ rõ nét trong Log:

* **Dấu hiệu nhận biết (SOC Alert):**
  * **Login Failure Spike:** Một địa chỉ IP phát sinh hàng trăm/ngàn yêu cầu đăng nhập lỗi (200 OK với nội dung "Failed") trong thời gian ngắn.
  * **Credential Stuffing:** Nhiều IP khác nhau cùng thử đăng nhập vào một tài khoản duy nhất (dấu hiệu của việc dùng Proxy/Botnet).
  * **Single Success after Multiple Failures:** Sau một chuỗi dài mã 200 (đăng nhập sai), xuất hiện duy nhất một mã 302 (thành công) từ cùng một IP.
  * **Parameter Mismatch:** Log của WAF ghi nhận tham số trùng tên nhưng giá trị khác nhau (GET vs POST).
* **Giả lập dòng Log điển hình (Web Access Log):**

```
# SOC thấy IP 192.168.1.50 đang Brute Force tài khoản 'admin'
192.168.1.50 - - [18/May/2024:22:00:01] "POST /login HTTP/1.1" 200 512 "user=admin&pass=123456"
192.168.1.50 - - [18/May/2024:22:00:02] "POST /login HTTP/1.1" 200 512 "user=admin&pass=password"
...
192.168.1.50 - - [18/May/2024:22:05:10] "POST /login HTTP/1.1" 302 0 "user=admin&pass=admin@123"
=> ALERT: Successful Brute Force Attack detected from 192.168.1.50 on user 'admin'.
```

## 4. Useful Commands

### 🛠️ Sử dụng FFUF để dò quét (Enumeration & Brute Force)

```bash
# Username Enumeration: Lọc các response chứa thông báo "tài khoản đã tồn tại"
ffuf -w usernames.txt:FUZZ -u http://10.10.x.x/signup -X POST \
     -d "username=FUZZ&email=x&password=x" -mr "already exists"

# Brute Force: Thử cặp User:Pass, lọc bỏ mã 200 (thất bại) để tìm mã 302 (thành công)
ffuf -w valid_users.txt:W1,passwords.txt:W2 -u http://10.10.x.x/login -X POST \
     -d "username=W1&password=W2" -fc 200
```

### 🍪 Cookie & Hashing

* **Base64 Decode (Linux):** echo "dXNlcjp1c2Vy" | base64 -d
* **MD5 Hashing (Linux):** echo -n "admin" | md5sum
* **Bẻ khóa Hash trực tuyến:** [CrackStation.net](https://www.google.com/url?sa=E\&q=https%3A%2F%2Fcrackstation.net%2F)

### 📝 Cấu hình bảo mật (Best Practices cho Developer)

* **Generic Error Messages:** Luôn báo "Invalid Username or Password" cho cả hai trường hợp.
* **Rate Limiting:** Giới hạn số lần thử đăng nhập từ 1 IP (vd: 5 lần/phút).
* **Account Lockout:** Khóa tài khoản tạm thời sau nhiều lần nhập sai.
* **HttpOnly & Secure Flags:** Ngăn chặn truy cập Cookie qua JavaScript và chỉ gửi qua HTTPS.

***

<br>
