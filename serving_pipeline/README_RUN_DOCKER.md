# Hướng Dẫn Chạy MLOps Serving (API & UI) Trên Máy Cá Nhân Bằng Docker

Tài liệu này hướng dẫn chi tiết từng bước để bạn có thể khởi động bản build mới nhất của hệ thống phục vụ mô hình (Model Serving) bao gồm **Backend API (FastAPI)** và **Frontend UI (Gradio/Streamlit)** từ Docker ngay trên máy tính của bạn sau khi pipeline CI/CD trên Github Actions đã chạy thành công.

---

## 🚀 Bước 1: Yêu Cầu Cơ Bản Phải Có

Trước khi bắt đầu, hãy đảm bảo hệ máy của bạn đã cài đặt các công cụ sau:
1. Đã cài **Docker** và **Docker Compose**. Bạn có thể tải [Docker Desktop tại đây](https://www.docker.com/products/docker-desktop).
2. Tới thư mục dự án trên Terminal/Command Prompt (bằng Github Bash, Windows PowerShell hoặc VSCode Terminal).

---

## 🛠 Bước 2: Bật Môi Trường Chứa Model (MLFlow & MinIO)

Mô hình của chúng ta được lưu trữ từ xa trên MLflow (ứng dụng quản lý model) và MinIO (kho chứa file mô hình S3). Do đó, **bạn phải chạy hai ứng dụng nền tảng này trước tiên** để `serving_pipeline` có thể kéo mô hình về được.

Từ thư mục gốc dự án (mlops_project), di chuyển vào thư mục cài đặt MLflow và chạy compose như sau:

```bash
docker network create aio-network
cd infra/docker/mlflow
```

Tại đây hãy chắc chắn có file `.env` chứa mật khẩu mặc định (tạo nếu chưa có):
```env
MYSQL_DATABASE=mlflow
MYSQL_USER=mlflow
MYSQL_PASSWORD=mlflow
MYSQL_ROOT_PASSWORD=root
AWS_ACCESS_KEY_ID=minio
AWS_SECRET_ACCESS_KEY=minio123
```

Kích hoạt MLFlow và kho MinIO:
```bash
docker compose up -d
```

Bây giờ bạn có thể thử xem các ứng dụng nền tảng này đã lên hay chưa:
- **Trang Quản Lý Model (MLflow UI):** [http://localhost:5000](http://localhost:5000)
- **Trang Quản Lý File Model (MinIO UI):** [http://localhost:9001](http://localhost:9001) *(Đăng nhập bằng: `minio` / `minio123`)*

---

## 🛠 Bước 3: Tạo File Môi Trường Cho Serving (.env)

Hệ thống serving của chúng ta cần một file `.env` chuyên biệt (nằm CÙNG CẤP với thư mục đang chứa file `docker-compose.yml` của API).

Từ thư mục gốc của dự án, di chuyển sang thư mục code của phần serving:
```bash
cd serving_pipeline/
```

Và tạo một file **CHÍNH XÁC CÓ TÊN LÀ `.env`** đặt TRỰC TIẾP bên trong thư mục `serving_pipeline/` này:

```env
# URI truy cập model qua MLflow (Sử dụng model Champion Logistic Regression bạn đã train)
MODEL_URI="models:/logistic_regression_churn@champion"

# Địa chỉ trỏ tới MLflow và MinIO S3 (Cần mở dịch vụ này lên trước cùng network nếu chạy Model từ server cục bộ)
MLFLOW_TRACKING_URI="http://localhost:5000"
MLFLOW_S3_ENDPOINT_URL="http://localhost:9000"

# Cấu hình MinIO Object Storage
AWS_ACCESS_KEY_ID="minio"
AWS_SECRET_ACCESS_KEY="minio123"
AWS_DEFAULT_REGION="us-east-1"

# URL kết nối giữa UI và Backend
API_BASE_URL="http://api:8000"
```
*(Lưu ý: `API_BASE_URL` mình đổi thành `http://api:8000` thay vì localhost để trong nội bộ mạng Docker thì giao diện UI tự động liên kết thẳng vào container `api`).*

Bên trong `serving_pipeline/docker-compose.yml`, các biến `${REGISTRY}` và `${REF_REPO}` được sử dụng để tải image trực tiếp về nội bộ máy. Vì Github là repos public (hoặc nếu là private bạn sẽ cần `docker login ghcr.io`), chúng ta sẽ gán tường minh 2 biến này vào bằng `.env`:

```env
# Khai báo Repository trên Github của bạn để tải về đúng Image
REGISTRY="ghcr.io"
REF_REPO="gialam77k1/mlops_project"
```

---

## 🔥 Bước 4: Pull Image và Chạy Dịch Vụ Cùng Lúc

Tại chính thư mục `serving_pipeline`, nơi đang chứa file `docker-compose.yml` (và có `.env` do bạn vừa tạo xong), lệnh này sẽ đọc chung biến của Docker:

```bash
docker compose up -d
```

**Những gì Docker sẽ làm:**
1. Nó đọc cấu hình biến `ghcr.io/gialam77k1/mlops_project:main` và tải Container API mới nhất từ Github Workspace về máy.
2. Nó tiếp tục kéo Image của `.../mlops_project-ui:main` về máy.
3. Liên kết chung 2 Image này vào trong `aio-network`.
4. Mở cổng `8000` cho Backend và `7860` cho Frontend phục vụ người dùng.

*Lưu ý: Quá trình này có thể tốn vài phút trong lần chạy đầu tiên do máy phải tải image (~1GB tuỳ dung lượng) từ hệ thống máy chủ mạng Github về local PC của bạn.*

---

## 🎉 Bước 5: Đăng Nhập Và Tận Hưởng Tất Cả UI

Sau khi bạn nhận được thông báo `Container churn-prediction-api Started` và `Container churn-prediction-ui Started`, cùng với MLflow và MinIO, toàn bộ cụm hệ thống MLOps của bạn đã on-air hoàn toàn!

Bạn hãy mở trình duyệt ưa thích của bạn (Chrome, Edge, Safari...) lên và truy cập vào các giao diện Web sau để trải nghiệm:

### 🌟 Hệ Thống Phục Vụ Người Dùng:
1. **👉 Giao Diện Dự Đoán (Gradio/Streamlit UI):** [http://localhost:7860](http://localhost:7860) 
   *(Điền các thông tin tính năng của khách hàng vào Form -> Bấm Dự đoán -> Xem kết quả rủi ro Churn (Khách hàng rời bỏ)!)*
2. **👉 Tài Liệu API Dành Cho DEV (Backend Swagger UI):** [http://localhost:8000/docs](http://localhost:8000/docs)
   *(Xem danh sách toàn cầu của các API như /predict, /health, ... Sử dụng giao diện này để test Post lệnh hoặc tích hợp API vào ứng dụng Mobile/Web Khác)*

### 🌟 Hệ Thống Nền Tảng (Admin):
3. **👉 Giao Diện Quản Quản Trị Mô Hình (MLflow UI):** [http://localhost:5000](http://localhost:5000)
   *(Xem lịch sử tất cả các model bạn đã train, thông số Accuracy, F1-score, version nào đang là "Champion", version nào là "Staging")*
4. **👉 Kho Cấu Cấu Trúc File S3 (MinIO Console UI):** [http://localhost:9001](http://localhost:9001)
   *(Kho lưu trữ vật lý của các file `.pkl`, `.bin`, `.parquet`,... Tài khoản đăng nhập: Tên: `minio` | Mật Khẩu: `minio123`)*
2. **👉 Tài Liệu API Developer (Backend Swagger):** [http://localhost:8000/docs](http://localhost:8000/docs)
   *(Xem danh sách toàn bộ Endpoint /predict, /health, ... để dùng cho việc nhúng model vào App Mobile hoặc trang web khác)*

---

## 🧹 Câu Lệnh Quản Lý Chung

1. Chạy ngầm (bạn có thể tắt Terminal mà tiến trình vẫn ở đó):
   ```bash
   docker compose up -d
   ```
2. Gỡ bỏ/Tắt dịch vụ ngay lập tức:
   ```bash
   docker compose down
   ```
3. Xem log chạy báo lỗi của Container:
   ```bash
   docker compose logs -f
   ```
4. Nếu vừa đẩy bản cập nhật mới trên Github và muốn đồng bộ chạy lại trên máy cá nhân:
   ```bash
   docker compose pull
   docker compose up -d
   ```
