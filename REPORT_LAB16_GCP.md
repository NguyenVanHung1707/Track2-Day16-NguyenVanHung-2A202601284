# BÁO CÁO THỰC HÀNH LAB 16: CLOUD AI ENVIRONMENT SETUP (GCP)
**Họ và tên / Student:** Nguyễn Văn Hùng  
**Project ID GCP:** `track2-day16-2a202601284`  
**Dịch vụ Cloud:** Google Cloud Platform (GCP)  
**Phương thức triển khai (IaC):** Terraform  

---

## 1. TỔNG QUAN KIẾN TRÚC HẠ TẦNG DỰ ÁN

Hạ tầng Môi trường Cloud AI trên GCP đã được triển khai tự động bằng **Terraform**:
1. **VPC Network & Subnet:** Mạng VPC riêng (`ai-vpc`) với 1 Private Subnet (`10.0.0.0/24`) tại region `us-central1`.
2. **Cloud Router & Cloud NAT:** Cho phép Compute Node kết nối Internet ra ngoài để tải dataset/thư viện an toàn mà không cần cấp Public IP trực tiếp.
3. **Compute Node (CPU VM):** Instance `e2-medium` (2 vCPU, 4 GB RAM) chạy Debian 12.
4. **Bảo mật & Kết nối:**
   - Firewall rules chỉ cho phép cổng 22 (SSH) qua IAP IP range (`35.235.240.0/20`) và cổng 8000 (Health Check).
   - Truy cập kết nối SSH an toàn thông qua Identity-Aware Proxy (`gcloud compute ssh --tunnel-through-iap`).
5. **External HTTP Load Balancer:** Cấu hình trỏ tới cổng 8000 của Compute Node.

---

## 2. KẾT QUẢ HUẤN LUYỆN & BENCHMARK MÔ HÌNH LIGHTGBM TRÊN CPU

Thực hiện huấn luyện mô hình **LightGBM Classifier** trên bộ dữ liệu gian lận thẻ tín dụng thực tế **Credit Card Fraud Detection** (284,807 giao dịch, 30 đặc trưng).

![Benchmark Output Output](images/benchmark_output.png)

### Bảng tổng hợp các chỉ số hiệu năng (Metrics Table):

| Chỉ số / Metric | Kết quả đo lường | Đánh giá & Nhận xét |
|---|---|---|
| **Thời gian nạp dữ liệu (Data Load)** | **2.8815 s** | Đọc 284,807 dòng từ file `creditcard.csv` vào DataFrame |
| **Thời gian huấn luyện (Training Time)** | **1.3869 s** | Tốc độ huấn luyện cực nhanh trên CPU `e2-medium` (2 vCPU) |
| **Số vòng lặp tối ưu (Best Iteration)** | **1** | Mô hình đạt điểm hội tụ sớm |
| **AUC-ROC** | **0.951654 (95.17%)** | Khả năng phân biệt giao dịch gian lận rất cao |
| **Accuracy (Độ chính xác)** | **0.998947 (99.89%)** | Tổng số giao dịch phân loại đúng |
| **F1-Score** | **0.727273** | Chỉ số hài hòa giữa Precision và Recall |
| **Precision** | **0.655738** | Độ chính xác khi cảnh báo gian lận |
| **Recall** | **0.816327** | Bắt được 81.63% các giao dịch gian lận thực tế |
| **Inference Latency (1 dòng)** | **1.3343 ms** | Độ trễ xử lý 1 giao dịch cực thấp |
| **Inference Throughput (1000 dòng)** | **670,805.99 QPS** | Xử lý hơn 670k câu hỏi/dự đoán mỗi giây |

---

## 3. GIÁM SÁT TÀI NGUYÊN VÀ BIỂU ĐỒ (MONITORING)

### 3.1. Màn hình CPU Usage (`top`)
- Tải CPU ổn định, hệ thống hoạt động nhẹ nhàng.
![CPU Usage](images/top_cpu.png)

### 3.2. Màn hình RAM Usage (`free -h`)
- Bộ nhớ RAM tiêu thụ ~521 MB, RAM khả dụng **3.3 GB / 3.8 GB**.
![RAM Usage](images/free_memory.png)

### 3.3. Màn hình Network Traffic (`ip -s link`)
- Tổng dung lượng đã tải (RX): ~253 MB qua card mạng `ens4`.
![Network Traffic](images/network_link.png)

### 3.4. Giao diện GCP Observability Dashboard
- Biểu đồ biến động CPU và Network thực tế trên Google Cloud Console:
![GCP Observability](images/gcp_observability.png)

---

## 4. BÁO CÁO ĐÁNH GIÁ (EVALUATION REPORT)

1. **Hiệu năng vượt trội trên CPU:** Mô hình LightGBM chạy trên instance nhỏ `e2-medium` (2 vCPU / 4GB RAM) chỉ mất **1.38 giây** để hoàn thành training. Điều này chứng minh thuật toán GBDT tối ưu hoá luồng CPU vô cùng hiệu quả, không nhất thiết phải đầu tư chi phí lớn cho GPU đối với các bài toán dữ liệu bảng (tabular data).
2. **Chất lượng mô hình cao:** Mặc dù tập dữ liệu bị mất cân bằng nghiêm trọng (dưới 0.2% giao dịch gian lận), mô hình vẫn đạt chỉ số **AUC-ROC = 0.9516** và **Recall = 81.63%**, đáp ứng tốt bài toán nghiệp vụ ngăn ngừa rủi ro tài chính.
3. **Độ trễ dự đoán cực thấp:** Inference latency chỉ **1.33 ms** và Throughput đạt tới **670,805 QPS**, hoàn toàn sẵn sàng đưa vào các hệ thống chấm điểm gian lận thời gian thực (real-time scoring system).
4. **Kiến trúc an toàn & Tiết kiệm:** Việc đưa Compute Node vào Private VPC kết hợp Cloud NAT giúp ngăn chặn tấn công từ bên ngoài. Tổng chi phí duy trì toàn bộ hạ tầng ước tính chỉ **~$0.09/giờ**.

---

## 5. XÁC NHẬN DỌN DẸP TÀI NGUYÊN
Toàn bộ 16 tài nguyên trên GCP đã được giải phóng hoàn toàn bằng lệnh `terraform destroy -auto-approve`, đảm bảo tài khoản GCP không bị phát sinh chi phí phát sinh sau bài lab.
