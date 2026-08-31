# Machine Learning Algorithms & Model Evaluation in Cyber Security

## 1. Concept

Trong an ninh mạng, các bài toán học máy thường được chia làm 2 trường phái chính:

* **Supervised Learning (Có giám sát):** Dùng để phân loại mã độc đã biết. Mô hình được học trên một bộ từ điển (Dataset) có gán nhãn rõ ràng: "Đây là virus, đây là file sạch".
  * Ví dụ: Cảnh báo Email Phishing, Hệ thống IDS dựa trên Signature.
* **Unsupervised Learning (Không giám sát):** Dùng để phát hiện các mối đe dọa Zero-day chưa từng xuất hiện. Mô hình tự học hành vi "Bình thường" của hệ thống và đánh dấu mọi thứ sai lệch là "Bất thường" (Anomaly).
  * Ví dụ: Hệ thống UEBA (Phân tích hành vi người dùng).

## 2. Key Mechanism&#x20;

### A. Logistic Regression

Mặc dù có tên là "Hồi quy", đây lại là một thuật toán **Phân loại (Classification)**.

*   **Cơ chế:** Nó tính toán một điểm số tuyến tính (Linear Score), sau đó đưa qua hàm **Sigmoid** để ép điểm số này về xác suất từ 0 đến 1. Nếu xác suất > 0.5

    , nó kết luận là Tấn công.
* **Ưu điểm SOC:** Đây là mô hình "Hộp trắng" (White-box). SOC Analyst có thể giải thích chính xác tại sao AI lại chặn IP này dựa vào các trọng số (Weights).

### B. Decision Tree & Random Forest

* **Decision Tree:** Hoạt động như một lưu đồ câu hỏi (Ví dụ: Kích thước gói tin > 1000? Nếu có -> Chuyển sang câu hỏi Port là gì?).
* **Random Forest:** Xây dựng hàng trăm cây quyết định nhỏ. Mỗi cây chỉ được học một phần dữ liệu và một phần đặc trưng (Ngẫu nhiên hóa). Sau đó chúng "bầu chọn" kết quả cuối cùng.
* **Ưu điểm SOC:** Chống Overfitting cực tốt, tự động đánh giá mức độ quan trọng của đặc trưng (Feature Importance).

### C. k-Nearest Neighbors

* **Cơ chế:** Thuật toán lười biếng (Lazy Learner). Nó không học gì cả, chỉ lưu lại toàn bộ dữ liệu. Khi có gói tin mới, nó sẽ đo khoảng cách hình học tới k điểm gần nhất. Nếu xung quanh toàn là mã độc -> Nó là mã độc.
* **Nhược điểm SOC:** Không thể dùng cho IDS theo thời gian thực (Real-time) vì tốc độ dự đoán quá chậm và tốn quá nhiều RAM. Bắt buộc phải **Scale dữ liệu** (RobustScaler) trước khi dùng kNN.

### D. Support Vector Machine (SVM)

* **Cơ chế:** Cố gắng kẻ một đường ranh giới (Siêu phẳng - Hyperplane) phân chia 2 lớp sao cho khoảng cách lề (Margin) là lớn nhất.
* **Kernel Trick:** Nếu không thể kẻ đường thẳng, nó dùng thủ thuật "Kernel" bẻ cong không gian để phân tách.

## 3. Security Perspective

Trong bảo mật, việc chọn sai chỉ số đánh giá (Metric) sẽ dẫn đến những thảm họa thực sự.

### 🔴Nghịch lý độ chính xác

Giả sử có 10.000 gói tin, chỉ có 1 gói là mã độc. Một AI "ngu ngốc" viết code chặn mặc định: Luôn trả về "Bình thường".\
-> Độ chính xác (**Accuracy**) của AI này là 99.99%. Nhưng nó **hoàn toàn vô dụng** vì bỏ lọt mã độc (Recall = 0%).\
**Tuyệt đối không dùng Accuracy để đánh giá mô hình bảo mật.**

### 🔵 Góc nhìn SOC: Chọn Metric nào?

1. **Recall (Độ nhạy - True Positive Rate):**
   * Ý nghĩa: Trong số các cuộc tấn công THỰC SỰ diễn ra, AI bắt được bao nhiêu?
   * Khi nào ưu tiên: Bắt mã độc tống tiền (Ransomware), Tấn công APT. Vì **bỏ sót (False Negative) là chí mạng**. Thà bắt nhầm còn hơn bỏ sót.
2. **Precision (Độ chính xác cảnh báo):**
   * Ý nghĩa: Trong số 100 cảnh báo AI phát ra, có bao nhiêu cảnh báo là thật?
   * Khi nào ưu tiên: Hệ thống Auto-blocking (Tự động chặn IP) hoặc Auto-delete Email. Vì **cảnh báo giả (False Positive)** sẽ làm sập hoạt động kinh doanh và gây mệt mỏi cho SOC (Alert Fatigue).
3. **F-Beta Score (**&#x3B2;=&#x32;**):**
   * Đây là chỉ số vàng cho IDS. Nó gộp cả Precision và Recall, nhưng nhân đôi trọng số cho Recall (Ưu tiên bắt mã độc hơn là sợ cảnh báo giả).
4. **PR Curve (Precision-Recall Curve):**
   * Thay vì dùng ROC Curve (dễ bị ảo tưởng sức mạnh khi dữ liệu quá chênh lệch), hãy dùng đường cong PR. Diện tích dưới đường cong (PR-AUC) càng lớn, mô hình càng xuất sắc trên tập dữ liệu mất cân bằng.

## 4. Useful Commands

Dưới đây là một Pipeline Python chuẩn mực mực nhất để xây dựng IDS (Intrusion Detection System), ngăn chặn tuyệt đối lỗi rò rỉ dữ liệu (Data Leakage) và ưu tiên chỉ số `F2-Score`.

```python
import pandas as pd
from sklearn.model_selection import train_test_split, StratifiedKFold, GridSearchCV
from sklearn.preprocessing import RobustScaler
from sklearn.ensemble import RandomForestClassifier
from sklearn.metrics import classification_report, make_scorer, fbeta_score
from imblearn.combine import SMOTEENN
from imblearn.pipeline import Pipeline as ImbPipeline

# 1. Tải và số hóa dữ liệu
df = pd.read_csv('IDS_Logs.csv')
X = pd.get_dummies(df.drop('label', axis=1)) # One-hot encoding
y = (df['label'] != 'normal').astype(int) # 1: Attack, 0: Normal

# 2. CHIA TÁCH DỮ LIỆU ĐẦU TIÊN (Tránh Data Leakage)
X_train, X_test, y_train, y_test = train_test_split(
    X, y, test_size=0.2, stratify=y, random_state=42
)

# 3. Đóng gói Pipeline chuẩn SOC (Bao gồm làm sạch Outliers và cân bằng dữ liệu)
soc_pipeline = ImbPipeline([
    ('scaler', RobustScaler()), # Dùng RobustScaler để chống nhiễu DDoS
    ('resample', SMOTEENN(random_state=42)), # Tăng mẫu thiểu số + xóa ranh giới rác
    ('model', RandomForestClassifier(random_state=42))
])

# 4. Định nghĩa không gian siêu tham số và Scoring bằng F2
param_grid = {
    'model__n_estimators': [100, 150],
    'model__max_depth': [10, 15]
}

# ƯU TIÊN RECALL (Bắt bằng được mã độc) bằng Beta=2
f2_scorer = make_scorer(fbeta_score, beta=2)

# 5. Huấn luyện với Cross-Validation
grid_search = GridSearchCV(
    estimator=soc_pipeline, param_grid=param_grid, 
    cv=StratifiedKFold(5), scoring=f2_scorer, n_jobs=-1
)
grid_search.fit(X_train, y_train)

# 6. Đánh giá thực tế trên tập Test độc lập
y_pred = grid_search.best_estimator_.predict(X_test)
print(classification_report(y_test, y_pred, target_names=['Normal', 'Attack']))
```
