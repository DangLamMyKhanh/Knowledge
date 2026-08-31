---
description: Deep Learning, Adversarial ML & LLM in Cyber Security
---

# Emerging AI Threats

## 1. Concept

Sự trỗi dậy của **Học sâu (Deep Learning)** và **Mô hình ngôn ngữ lớn (LLMs)** đang định hình lại chiến trường an ninh mạng.

* **ELI5:**
  * **NLP & Deep Learning:** Giống như dạy một đứa trẻ đọc hiểu ý nghĩa thực sự của một email thay vì chỉ tìm các từ khóa cấm. Nó biết từ "bank" trong "bank account" (tài khoản ngân hàng) khác với "bank" trong "river bank" (bờ sông).
  * **Adversarial ML (Học máy đối kháng):** Là việc tạo ra các "ảo ảnh quang học/toán học" để đánh lừa AI. Giống như bạn dán một mẩu băng dính nhỏ lên biển báo "Stop", con người vẫn đọc được nhưng xe tự lái (AI) lại nhìn thành biển "Tốc độ 100km/h" và đâm xe.
  * **Agentic AI:** AI không chỉ là Chatbot thụ động nữa, nó trở thành một "nhân viên SOC ảo" (Tác tử). Nó tự suy nghĩ (Reasoning) -> Tự mở công cụ (Tools) tra cứu Log -> Tự động cô lập máy bị nhiễm mã độc.

## 2. Key Mechanism

### A. Xử lý Ngôn ngữ Tự nhiên (NLP) trong Security

Để AI hiểu văn bản (Email, URL, Log), dữ liệu phải đi qua Pipeline:

1. **Tokenization & Normalization:** Cắt câu thành các từ nhỏ. Đưa mọi email/link lạ về một token chung (vd: EMAIL\_TOKEN, URL\_TOKEN) để giảm nhiễu.
2. **Vectorization (Biểu diễn số học):**
   * TF-IDF: Gắn trọng số cao cho các từ hiếm (vd: paypal1, urgent). Phù hợp cho mô hình Machine Learning truyền thống.
   * Word Embeddings / BERT: Ánh xạ từ vào không gian đa chiều để hiểu **Ngữ cảnh**. Từ Phishing và Scam sẽ nằm gần nhau về mặt không gian hình học.

### B. Học máy Đối kháng (Adversarial ML)

Tấn công vào ranh giới quyết định toán học của AI thay vì lỗ hổng phần mềm.

* **Evasion Attacks (Lách né):** Hacker thêm các **Nhiễu loạn siêu nhỏ (Perturbations)** vào file mã độc. File vẫn chạy mã độc bình thường, nhưng AI lại phân loại nó là file an toàn 100%. (Kỹ thuật phổ biến: FGSM, PGD).
* **Data Poisoning (Đầu độc dữ liệu):** Bơm dữ liệu rác vào quá trình huấn luyện để làm biến dạng khả năng phân loại của AI.

### C. Mối đe dọa từ LLM & Agentic AI

* **Prompt Injection:** Trộn lẫn "Câu lệnh hệ thống" và "Dữ liệu người dùng". Hacker ép AI bỏ qua luật an toàn (Jailbreak) và thực thi lệnh xấu.
* **RAG Poisoning:** Đầu độc các tài liệu lưu trữ nội bộ. Khi AI truy xuất (Retrieval) tài liệu này để trả lời, nó sẽ đọc phải lệnh độc hại ẩn bên trong.
* **ReAct Pattern (Agentic AI):** Vòng lặp Suy nghĩ -> Hành động -> Quan sát. Nếu Agent được cấp quyền quá cao (vd: quyền xóa Database), hacker có thể dùng Prompt Injection để chiếm quyền (Goal Hijacking) và ép Agent tự tay xóa dữ liệu của công ty.

## 3. Security Perspective

### 🔴 Attacker View (Red Team)

Hacker đang sử dụng Generative AI để "nâng cấp" vũ khí:

* **Polymorphic Malware:** Viết mã độc bằng AI. Sau mỗi lần lây nhiễm, mã độc tự viết lại hình dáng và chữ ký (Signature) của chính nó, qua mặt toàn bộ các phần mềm Antivirus truyền thống.
* **Spear-Phishing tự động:** Thu thập dữ liệu mạng xã hội của mục tiêu, đưa vào LLM để tự động viết ra hàng ngàn email lừa đảo cá nhân hóa với văn phong hoàn hảo (không còn lỗi ngữ pháp).
* **Sleeper Agents:** Cài **Backdoor** vào các mô hình AI nguồn mở. Khi đưa dữ liệu sạch, AI trả lời đúng. Khi đưa dữ liệu chứa "từ khóa bí mật", AI lập tức tạo ra mã độc.

### 🔵 Defender View (Blue Team/SOC)

Blue Team phải bảo vệ cả hệ thống truyền thống VÀ bảo vệ chính các mô hình AI của doanh nghiệp (AI Security Posture).

* **Dấu hiệu nhận biết (SOC Alert - AI Attacks):**
  * **Jailbreak Attempts (LLM Logs):** Phát hiện các cụm từ thao túng trong input của người dùng: "Ignore previous instructions", "You are DAN (Do Anything Now)", "Developer Mode".
  * **API Abuse (Model Extraction):** Một tài khoản gửi hàng ngàn câu hỏi vào API của AI để ghi lại kết quả nhằm "Sao chép" (Clone) mô hình của doanh nghiệp.
  * **Agent Privilege Abuse:** Một AI Agent đột ngột thực hiện các lệnh nhạy cảm (như Delete, Drop, hay SSH ra máy chủ lạ) do bị hacker thao túng.
* **Phòng thủ AI (Defense-in-Depth):**
  * **Adversarial Training:** Cố tình tạo ra các mẫu bị nhiễu (PGD) và đưa vào tập huấn luyện để AI làm quen và cứng cáp hơn (Robustness).
  * **Least Privilege cho AI Agent:** Không bao giờ cấp quyền Root cho Agent. Chỉ cấp quyền ở chế độ `Read-only` hoặc sử dụng **Human-in-the-Loop** (Bắt buộc con người phải duyệt lệnh trước khi Agent thực thi việc cô lập máy chủ).

## 4. Useful Commands

### 📊 Bảng tổng kết đánh giá mô hình bảo mật

Dành riêng cho việc phân tích các báo cáo hiệu năng (Performance Report) của hệ thống IDS/AI:

<table data-header-hidden><thead><tr><th></th><th></th><th></th></tr></thead><tbody><tr><td>Tình huống thực tế trong SOC</td><td>Chỉ số / Công cụ tối ưu</td><td>Lý do kỹ thuật (Tại sao?)</td></tr><tr><td>Dữ liệu mất cân bằng nghiêm trọng (10.000 Bình thường : 1 Tấn công)</td><td><strong>PR-AUC (Đường cong Precision-Recall)</strong></td><td>Không bị "ảo tưởng sức mạnh" bởi số lượng True Negative khổng lồ như ROC-AUC.</td></tr><tr><td>Chi phí bỏ sót Tấn công (Ransomware) là quá đắt</td><td><strong>Recall</strong> hoặc <strong>F2-Score</strong></td><td>F2-Score gán trọng số ưu tiên bắt mã độc cao gấp 2 lần độ chính xác.</td></tr><tr><td>Analyst bị quá tải vì Cảnh báo giả (Alert Fatigue)</td><td><strong>Precision</strong></td><td>Tối ưu độ tin cậy. 10 cảnh báo thì phải đúng 9 cảnh báo.</td></tr><tr><td>Chứng minh Mô hình IDS mới thực sự tốt hơn mô hình cũ</td><td><strong>McNemar's Test</strong></td><td><p>Kiểm định ý nghĩa thống kê (</p><pre><code>p−value&#x3C;0.05p−value&#x3C;0.05
</code></pre><p>). Chứng minh sự cải thiện không phải do ăn may.</p></td></tr><tr><td>Log mạng chứa quá nhiều giá trị đột biến (DDoS Outliers)</td><td><strong>RobustScaler</strong></td><td>Sử dụng Median (Trung vị) thay vì Mean (Trung bình), không bị bóp méo bởi các spike.</td></tr></tbody></table>

### 🛠️ Kỹ thuật Adversarial Training (Phòng thủ AI) bằng Python

Sử dụng thư viện Adversarial Robustness Toolbox (ART)

```python
from art.attacks.evasion import ProjectedGradientDescent
from art.estimators.classification import KerasClassifier

# Tạo ra các mẫu đối kháng (Adversarial Examples) bằng thuật toán PGD
attack = ProjectedGradientDescent(estimator=classifier, eps=0.1, eps_step=0.01, max_iter=20)
x_test_adv = attack.generate(x=x_test)

# Huấn luyện lại mô hình (Adversarial Training) bằng cả dữ liệu sạch và dữ liệu bị nhiễu
x_train_robust = np.vstack((x_train, x_train_adv))
y_train_robust = np.vstack((y_train, y_train))

classifier.fit(x_train_robust, y_train_robust)
```
