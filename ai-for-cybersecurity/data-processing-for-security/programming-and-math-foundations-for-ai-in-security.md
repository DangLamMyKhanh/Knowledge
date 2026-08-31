# Programming & Math Foundations for AI in Security

## 1. Concept

Để xây dựng hệ thống Học máy (Machine Learning) ứng dụng trong An toàn thông tin, chúng ta cần hai công cụ cốt lõi: **Toán học** (Đóng vai trò là Bộ não) và **Python** (Đóng vai trò là Đôi tay).

* **ELI5:** Hãy tưởng tượng công việc SOC giống như việc mò kim đáy bể (tìm 1 gói tin mã độc trong 1 triệu gói tin bình thường).
  * **Python (Pandas, NumPy, Scikit-learn):** Là chiếc máy xúc giúp bạn gom toàn bộ lượng dữ liệu khổng lồ (Logs) đó lại, sàng lọc rác, và sắp xếp chúng gọn gàng.
  * **Toán học (Xác suất thống kê, Định lý Bayes):** Là chiếc kính lúp ma thuật giúp bạn tính toán xác suất: "Với hình dáng và độ bóng này, khả năng đây là cây kim (mã độc) là bao nhiêu phần trăm?".
* **Ví dụ thực tế:** Khi hệ thống lọc Email Spam nhìn thấy từ "Free", nó không chặn ngay. Nó dùng **Định lý Bayes (Bayes' Theorem)** để tính toán: "Dựa trên dữ liệu lịch sử, xác suất một email là Spam khi có chứa từ 'Free' là 98%", từ đó đưa ra quyết định tự động.

## 2. Key Mechanism

Quy trình biến dữ liệu thô (Raw Logs) thành một mô hình AI nhận diện mối đe dọa đi qua 4 bước step-by-step:

1. **Data Ingestion & Typing (Thu thập & Định kiểu dữ liệu):** Sử dụng Python để đọc dữ liệu. Ép kiểu chính xác: IP là String, Port là Integer, Kích thước gói tin là Float. Việc sai kiểu dữ liệu sẽ làm hỏng hoàn toàn mô hình toán học phía sau.
2. **Data Cleaning & Preprocessing (Làm sạch dữ liệu):** Dùng thư viện Pandas để xử lý **Missing Values** (Dữ liệu khuyết thiếu) và loại bỏ **Outliers** (Dữ liệu ngoại lai) bằng phương pháp **IQR (Interquartile Range)**.
3. **Statistical Modeling (Mô hình hóa Thống kê):** Phân phối dữ liệu vào các mô hình Toán học. Sử dụng **Phân phối chuẩn (Gaussian Distribution)** để tìm ra trạng thái "bình thường" (Mean/Kỳ vọng) của hệ thống. Bất cứ hành vi nào nằm lệch xa khỏi trạng thái bình thường (Độ lệch chuẩn - Variance lớn) sẽ bị coi là bất thường (Anomaly).
4. **Probabilistic Decision (Ra quyết định dựa trên Xác suất):** Sử dụng **Ước lượng hợp lý cực đại (MLE)** hoặc **Định lý Bayes** để tính toán xác suất một sự kiện là cuộc tấn công.

## 3. Security Perspective&#x20;

Trong kỷ nguyên AI, các cuộc chiến giữa Red Team và Blue Team không chỉ nằm ở mã nguồn, mà nằm ở **Dữ liệu (Data)**.

### 🔴 Attacker View (Red Team)

Hacker khai thác các điểm mù trong mô hình Thống kê và Xác suất của hệ thống AI:

* **Data Poisoning (Đầu độc dữ liệu):** Hacker biết AI học trạng thái "Bình thường" thông qua giá trị trung bình (Mean). Chúng sẽ âm thầm thực hiện các hành vi bất thường ở cường độ cực nhỏ (ví dụ: gửi dữ liệu ra ngoài thêm 1KB mỗi ngày). Điều này dần dần kéo lệch giá trị **Mean** của hệ thống, khiến hành vi ăn cắp dữ liệu sau này bị AI coi là "bình thường".
* **Evasion Attacks (Tấn công lách luật):** Lợi dụng **False Negative (Sai số loại II)**. Hacker tìm cách thay đổi đặc trưng của mã độc (thêm mã rác, đổi tên hàm) để xác suất nhận diện rơi xuống dưới ngưỡng kích hoạt (Threshold) của mô hình.
* **Spurious Correlations (Khai thác tương quan giả):** Đánh lừa AI bằng cách chèn các từ khóa "vô hại" vào email phishing, khiến AI bị nhiễu loạn giữa Tương quan (Correlation) và Nhân quả (Causation).

### 🔵 Defender View (Blue Team/SOC)

Blue Team áp dụng Toán học để tinh chỉnh hệ thống giám sát, giải quyết bài toán lớn nhất của SOC: **Alert Fatigue (Mệt mỏi vì quá nhiều cảnh báo giả)**.

* **Tối ưu hóa Rủi ro có điều kiện (Conditional Risk):** Trong SOC, chi phí của một lỗi **False Negative** (Bỏ lọt mã độc) cao gấp hàng ngàn lần lỗi **False Positive** (Cảnh báo giả). Do đó, SOC áp dụng Lý thuyết quyết định Bayesian để hạ ngưỡng phân loại (Classification Threshold) từ 0.5 xuống 0.2. Nghĩa là: Chỉ cần có 20% khả năng là mã độc, hệ thống vẫn phải báo động.
* **Sử dụng Thống kê cứng cáp (Robust Statistics):** Để chống lại Data Poisoning, thay vì dùng Mean (dễ bị hacker bóp méo), SOC cấu hình AI sử dụng Median (Trung vị) vì nó có sức chống chịu cực tốt trước các giá trị ngoại lai.
* **Anomaly Detection (Phát hiện bất thường bằng Z-Score):** Nếu một User bình thường tải 10MB/ngày (Kỳ vọng μ ), hôm nay tải 10GB. Z-score sẽ vượt ngưỡng 3σ  . AI lập tức khóa tài khoản.

## 4. Useful Commands

Là một SOC Data Scientist, dùng Python liên tục để xử lý dữ liệu Log (Syslog, Zeek, Suricata).

### 🛠️ Xử lý Outliers (Ngoại lai) trong Log Mạng bằng Pandas & IQR

Công dụng: Loại bỏ các gói tin rác có kích thước quá dị biệt trước khi đưa vào AI.

```python
import pandas as pd

# Giả lập load file log mạng (CSV)
df = pd.read_csv('network_traffic_logs.csv')

# Tính toán IQR (Interquartile Range) cho cột 'packet_size'
Q1 = df['packet_size'].quantile(0.25)
Q3 = df['packet_size'].quantile(0.75)
IQR = Q3 - Q1

# Xác định ngưỡng trên và ngưỡng dưới
lower_bound = Q1 - 1.5 * IQR
upper_bound = Q3 + 1.5 * IQR

# Lọc bỏ các giá trị Outlier (Giữ lại dữ liệu sạch)
clean_data = df[(df['packet_size'] >= lower_bound) & (df['packet_size'] <= upper_bound)]

print(f"Đã loại bỏ {len(df) - len(clean_data)} log ngoại lai.")
```

### 🧠 Tính toán Định lý Bayes cho Lọc Phishing Mail

Công dụng: Hiểu bản chất thuật toán Naive Bayes ở cấp độ toán học.

```python
# P(Spam) - Xác suất tiền nghiệm: 40% email nhận được là Spam
p_spam = 0.40  

# P("Urgent" | Spam) - Khả năng chữ "Urgent" xuất hiện trong mail Spam
p_urgent_given_spam = 0.80  

# P("Urgent" | Normal) - Khả năng chữ "Urgent" xuất hiện trong mail bình thường
p_urgent_given_normal = 0.10  

# P(Normal) - Xác suất email bình thường
p_normal = 1 - p_spam  

# P("Urgent") - Xác suất chung gặp chữ "Urgent" (Luật xác suất toàn phần)
p_urgent = (p_urgent_given_spam * p_spam) + (p_urgent_given_normal * p_normal)

# P(Spam | "Urgent") - Định lý Bayes: Xác suất mail là Spam NẾU chứa chữ "Urgent"
p_spam_given_urgent = (p_urgent_given_spam * p_spam) / p_urgent

print(f"Xác suất email này là Phishing: {p_spam_given_urgent:.2%}")
# Output sẽ là ~ 84.21% -> Đưa ra quyết định Block email!
```

