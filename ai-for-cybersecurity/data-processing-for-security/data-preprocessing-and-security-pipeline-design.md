# Data Preprocessing & Security Pipeline Design

## 1. Concept (Khái niệm cơ bản)

Trong AI cho an ninh mạng, có một định luật tối thượng: **GIGO (Garbage In, Garbage Out - Rác vào, Rác ra)**. Dù thuật toán AI của bạn có siêu việt đến đâu, nếu dữ liệu đầu vào là "rác", kết quả dự đoán cũng sẽ là "rác".

* **ELI5:** Hãy tưởng tượng bạn là một siêu đầu bếp (AI Model). Bạn không thể nhét nguyên một con cá chưa đánh vảy và mớ rau dính bùn (Raw Logs) vào chảo để nấu. Bạn cần một đội ngũ phụ bếp (Data Pipeline) để làm sạch, thái nhỏ, tẩm ướp (Preprocessing & Feature Engineering) và chia tỉ lệ nguyên liệu hợp lý (Handling Imbalance).
* **Thực tế trong SOC:** Lưu lượng mạng bình thường chiếm 99.9%, còn các cuộc tấn công chỉ chiếm 0.1%. Nếu không biết cách xử lý sự "lệch pha" (Imbalance) này, AI sẽ mặc định đoán mọi thứ là "Bình thường" để đạt độ chính xác 99.9%, nhưng lại hoàn toàn vô dụng trong việc bắt mã độc.

## 2. Key Mechanism

Quy trình biến Logs thô thành "nhiên liệu" cho AI đi qua một **End-to-End Pipeline** khép kín:

### **Bước 1: Thu thập & Chuẩn hóa (Ingestion & Parsing)**

* Thu thập **PCAP** (nhìn sâu vào Payload nhưng nặng), **NetFlow** (nhẹ, chỉ chứa Metadata), và **Host Logs** (Windows Event, Syslog).
* Dùng **Regex** bóc tách các trường thô (Raw text) thành các cột dữ liệu có cấu trúc (IP, Port, Username, Timestamp).

### **Bước 2: Xử lý dữ liệu mạng đặc thù (Cyber-specific Handling)**

* **Không dùng One-Hot Encoding cho IP:** Sẽ làm tràn bộ nhớ (4 tỷ cột). Thay vào đó, dùng hàm phân loại IP (Private/Public) hoặc chuyển sang thập phân.
* **Scale bằng RobustScaler:** Trong an ninh mạng, các đợt DDoS tạo ra các Outliers (ngoại lai) khổng lồ làm hỏng giá trị Trung bình (Mean). Bắt buộc phải dùng RobustScaler (sử dụng Trung vị - Median) để cô lập nhiễu.

### **Bước 3: Feature Engineering (Triết xuất đặc trưng)**

* **Payload Entropy:** Đo độ "hỗn loạn" của gói tin. Text thường có Entropy thấp (3-5), mã độc bị nén/mã hóa có Entropy cao (\~8.0).
* **Inter-Arrival Time (IAT):** Tính thời gian giữa 2 gói tin liên tiếp. Nếu IAT rất đều đặn (độ lệch chuẩn ≈ 0 ), đó là máy móc (Mã độc gọi về C2 Server).

### **Bước 4: Giải quyết mất cân bằng & Giảm chiều (Balancing & PCA)**

* Dùng **PCA (Principal Component Analysis)** để nén hàng ngàn đặc trưng dư thừa nhằm tránh Overfitting.
* Dùng **SMOTE + ENN:** SMOTE sinh ra các mẫu tấn công giả lập để cân bằng dữ liệu, sau đó ENN (Edited Nearest Neighbours) đi dọn sạch các vùng ranh giới bị chồng lấn, giúp AI ra quyết định sắc nét, giảm False Positive.

## 3. Security Perspective (Góc nhìn Bảo mật)

### 🔴 Attacker View (Red Team)

Hacker biết Blue Team đang dùng AI, nên chúng sử dụng các kỹ thuật Anti-ML (Chống lại Học máy):

* **Jittering (Làm nhiễu IAT):** Thay vì để mã độc gọi về máy chủ C2 cứ đúng 60 giây 1 lần, hacker lập trình thêm tính năng Jitter. Mã độc sẽ gọi ngẫu nhiên từ 45s - 90s, phá vỡ hoàn toàn đặc trưng **Inter-Arrival Time (IAT)**, khiến AI lầm tưởng đây là con người đang lướt web.
* **Payload Encryption:** Hơn 80% lưu lượng hiện nay là HTTPS. Hacker bọc mã độc trong TLS/SSL. Vì AI không thể đọc được Payload (DPI bị mù), mô hình phát hiện bị tước đi một lượng lớn đặc trưng (Features) quan trọng.
* **DGA (Domain Generation Algorithms):** Hacker dùng thuật toán sinh ra hàng ngàn tên miền ngẫu nhiên (ví dụ: xk7q9m2p.xyz). Blue Team phải dùng kỹ thuật **Character N-grams** để đo tỷ lệ nguyên âm/phụ âm nhằm nhận diện các tên miền "phi tự nhiên" này.

### 🔵 Defender View (Blue Team/SOC)

Khi xây dựng AI cho SOC, điều đáng sợ nhất không phải là mô hình độ chính xác thấp, mà là các "Lỗi vận hành" thầm lặng:

* **Train-Serving Skew (Lệch pha Huấn luyện - Vận hành):** Mô hình ở phòng Lab chạy chính xác 99%, nhưng đưa ra hệ thống thực tế lại báo động giả liên tục. Nguyên nhân: Mã code tiền xử lý (Scaling/Encoding) lúc train offline khác với mã code chạy realtime (online). Giải pháp: Đóng gói Scaler và Model vào chung một Pipeline object.
* **Temporal Leakage (Rò rỉ tương lai):** Dùng dữ liệu của tháng 10 để huấn luyện AI bắt mã độc của tháng 8. Điều này là gian lận. Trong SOC, bắt buộc phải chia tập Train/Test theo trục thời gian thực (ví dụ: Train tháng 1-8, Test tháng 9-10).
* **Alert Fatigue Prevention:** Nhờ áp dụng kỹ thuật dọn ranh giới rác **ENN** sau khi dùng **SMOTE**, SOC Analyst không bị ngập lụt trong các cảnh báo giả (False Positives) do ranh giới quyết định của AI mờ nhạt.

## 4. Useful Commands

Dưới đây là các kỹ thuật xử lý dữ liệu Cyber bằng Python (`pandas`, `scikit-learn`):

### 🛠️ 1. Regex bóc tách Log SSH và Firewall

```python
import pandas as pd
import re

raw_logs = [
    "Jan 10 14:32:11 server sshd[11]: Failed password for root from 192.168.1.50 port 22",
    "Jan 10 14:40:22 firewall DROP TCP 10.0.0.5 -> 192.168.1.100:80"
]
df = pd.DataFrame({'raw_log': raw_logs})

# Hàm trích xuất IPv4 bằng Regex
def extract_ip(log):
    match = re.search(r'\b(?:[0-9]{1,3}\.){3}[0-9]{1,3}\b', log)
    return match.group(0) if match else None

df['src_ip'] = df['raw_log'].apply(extract_ip)
print(df['src_ip']) 
# Output: 192.168.1.50, 10.0.0.5
```

### 🛠️ 2. Xử lý IP an toàn (Private vs Public)

Không dùng One-Hot Encoding cho IP, hãy phân loại bản chất của IP.

```python
import ipaddress

def is_private_ip(ip):
    try:
        # Trả về 1 nếu là IP LAN, 0 nếu là Public Internet
        return int(ipaddress.ip_address(ip).is_private)
    except:
        return -1 # Bắt lỗi nếu log bị hỏng

df['is_private'] = df['src_ip'].apply(is_private_ip)
```

### 🧠 3. Giải quyết Mất cân bằng bằng SMOTE + ENN

Tạo mẫu tấn công nhân tạo (SMOTE) và dọn sạch nhiễu ở ranh giới (ENN).

```python
from imblearn.combine import SMOTEENN
from sklearn.ensemble import RandomForestClassifier
from imblearn.pipeline import Pipeline # Dùng pipeline của imblearn để tránh Train-Serving Skew

# Khởi tạo mô hình tối ưu SMOTE + ENN
smote_enn = SMOTEENN(random_state=42)
rf_model = RandomForestClassifier(random_state=42)

# Gom chung vào 1 Pipeline chuẩn sản xuất
soc_pipeline = Pipeline([
    ('balancing', smote_enn),
    ('classification', rf_model)
])

# Huấn luyện mô hình
# X_train, y_train là dữ liệu đã được số hóa
soc_pipeline.fit(X_train, y_train)
```

### ⏱️ 4. Tính toán đặc trưng IAT (Inter-Arrival Time) cho C2 Beaconing

```python
# Giả sử df đã được sắp xếp theo thời gian (Timestamp)
df['timestamp'] = pd.to_datetime(df['timestamp'])

# Tính độ trễ giữa gói tin hiện tại và gói tin trước đó (tính bằng giây)
df['iat_seconds'] = df['timestamp'].diff().dt.total_seconds()

# Nếu Độ lệch chuẩn của IAT gần bằng 0 -> Khả năng cao là Botnet/C2 Beaconing
```

