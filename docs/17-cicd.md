# 📘 Phần 17: CI/CD – Tự Động Hóa Tích Hợp & Triển Khai Với GitHub Actions

> **Motto cốt lõi:**  
> *Không bao giờ deploy bằng cách gõ lệnh thủ công trên Production | Mỗi lần `git push` là một lần kiểm thử và triển khai tự động | Bảo mật tuyệt đối với GitHub Repository Secrets!*  
> Chu trình chuẩn: **Bản chất CI/CD → Kiến trúc GitHub Actions → Quản lý Secrets an toàn → Viết Workflow YAML → Triển khai Tự Động → Healthcheck & Lab.**

---

## 🎯 1. Mục Tiêu Của Phần Học

Đây là phần học đỉnh cao khép lại toàn bộ lộ trình 17 phần của chuỗi Quản trị Ubuntu Server, kết nối toàn bộ kiến thức từ **SSH, User/Permission, Nginx, Systemd, Database, Docker** thành một hệ thống tự động hóa hoàn chỉnh.

Sau khi hoàn thành bài học này, bạn sẽ:
1. **Hiểu bản chất CI/CD & GitOps:** Phân biệt rõ ràng giữa **Continuous Integration** (Tích hợp liên tục: Tự động Lint/Test/Build) và **Continuous Deployment** (Triển khai liên tục: Tự động cập nhật code lên Server).
2. **Làm chủ cấu trúc GitHub Actions:** Nắm vững `workflow`, `event triggers`, `jobs`, `runners`, `steps` và `actions`.
3. **Bảo mật bí mật hạ tầng (Secrets Management):** Sử dụng GitHub Repository Secrets để bảo vệ SSH Private Key, IP Server và các biến môi trường nhạy cảm.
4. **Triển khai 2 chiến lược CD thực chiến chuẩn doanh nghiệp:**
   - **Chiến lược 1 (Systemd Native):** Tự động SSH vào server $\rightarrow$ `git pull` $\rightarrow$ `npm ci` $\rightarrow$ `npm run build` $\rightarrow$ `systemctl restart`.
   - **Chiến lược 2 (Docker Containerized):** Tự động Build Docker Image $\rightarrow$ Đẩy lên Registry $\rightarrow$ Server tự động kéo (`docker compose pull`) và khởi chạy bản mới.
5. **Thiết lập kiểm tra an toàn sau triển khai (Post-deployment Healthcheck):** Tự động kiểm tra Endpoint `/api/health` và tự động báo lỗi nếu ứng dụng không khởi động được.
6. **Xử lý các lỗi kinh điển:** Lỗi `Host key verification failed`, lỗi phân quyền SSH Key trong Runner, và lỗi xung đột file khi `git pull`.

---

## 🔄 2. Bản Chất CI/CD: Từ Lập Trình Viên Tới Máy Chủ Production

```text
┌─────────────────────────────────────────────────────────────────────────┐
│                    VÒNG ĐỜI TỰ ĐỘNG HÓA CỦA PIPELINE CI/CD              │
└────────────────────────────────────┬────────────────────────────────────┘
                                     │
    1. DEVELOPER (Máy Mac) ──────────┴──► git push origin main
                                     │
                                     ▼
    ┌─────────────────────────────────────────────────────────────────┐
    │ 2. CONTINUOUS INTEGRATION (CI) - Chạy trên GitHub Cloud Runner   │
    │    ├── Step 1: Tải mã nguồn (Checkout Repository)               │
    │    ├── Step 2: Cài đặt Node.js & Dependencies (npm ci)          │
    │    ├── Step 3: Kiểm tra định dạng mã nguồn (Linter)             │
    │    ├── Step 4: Chạy Unit Tests tự động (npm test)               │
    │    └── Step 5: Thử nghiệm Build xem có lỗi TypeScript không     │
    └────────────────────────────────┬────────────────────────────────┘
                                     │ (NẾU TẤT CẢ TEST ĐỀU PASS)
                                     ▼
    ┌─────────────────────────────────────────────────────────────────┐
    │ 3. CONTINUOUS DEPLOYMENT (CD) - Kết nối tới Ubuntu Server       │
    │    ├── Step 1: Lấy SSH Private Key từ GitHub Secrets            │
    │    ├── Step 2: SSH an toàn vào Ubuntu Server VM (Port 22)       │
    │    ├── Step 3: Cập nhật mã nguồn mới nhất                       │
    │    ├── Step 4: Build và Khởi động lại Service (Systemd / Docker)│
    │    └── Step 5: Gọi Healthcheck xác nhận hệ thống sống 100%!     │
    └─────────────────────────────────────────────────────────────────┘
```

---

## 🔑 3. Thiết Lập Khóa SSH Bảo Mật Dành Riêng Cho CI/CD

Để GitHub Actions Runner có thể đăng nhập vào Ubuntu Server VM của bạn mà không cần gõ mật khẩu, chúng ta sẽ tạo một cặp SSH Key chuyên dụng cho CI/CD (Deploy Key).

---

### Bước 3.1: Tạo cặp SSH Key riêng cho CI/CD (Trên máy Mac)
```bash
ssh-keygen -t ed25519 -C "github-actions-deploy-key" -f ~/.ssh/github_deploy_key
```
*Lệnh này sinh ra 2 file:*
* `~/.ssh/github_deploy_key` $\rightarrow$ **Private Key** (Dùng để đưa vào GitHub Secrets).
* `~/.ssh/github_deploy_key.pub` $\rightarrow$ **Public Key** (Dùng để copy vào Server).

---

### Bước 3.2: Đưa Public Key lên Ubuntu Server VM

Copy nội dung của `github_deploy_key.pub` vào file `~/.ssh/authorized_keys` của user `ubuntu` trên server:

```bash
# Cách 1: Dùng lệnh ssh-copy-id từ máy Mac
ssh-copy-id -i ~/.ssh/github_deploy_key.pub ubuntu@192.168.64.2

# Hoặc Cách 2: Dán thủ công vào ~/.ssh/authorized_keys trên Server
```

Kiểm tra kết nối thử nghiệm từ máy Mac:
```bash
ssh -i ~/.ssh/github_deploy_key ubuntu@192.168.64.2
```
*Nếu đăng nhập thành công vào server mà không hỏi mật khẩu $\rightarrow$ Đã cấu hình chuẩn xác!*

---

### Bước 3.3: Khai báo Secrets trên GitHub Repository

Vào trang GitHub của dự án $\rightarrow$ **Settings** $\rightarrow$ **Secrets and variables** $\rightarrow$ **Actions** $\rightarrow$ Bấm **New repository secret**:

```text
┌───────────────────────┬─────────────────────────────────────────────────┐
│ Secret Name           │ Giá trị (Value)                                 │
├───────────────────────┼─────────────────────────────────────────────────┤
│ SSH_HOST              │ Địa chỉ IP của Ubuntu Server (VD: 192.168.64.2) │
│ SSH_USER              │ ubuntu                                          │
│ SSH_PRIVATE_KEY       │ Toàn bộ nội dung file ~/.ssh/github_deploy_key   │
│ SSH_PORT              │ 22                                              │
└───────────────────────┴─────────────────────────────────────────────────┘
```

> [!CAUTION]
> Khi copy `SSH_PRIVATE_KEY`, bạn phải copy **toàn bộ file bao gồm cả dòng đầu `-----BEGIN OPENSSH PRIVATE KEY-----` và dòng cuối `-----END OPENSSH PRIVATE KEY-----`**.

---

## 🚀 4. Chiến Lược 1: Triển Khai Systemd Native (SSH Deployment)

Đây là chiến lược phù hợp với các ứng dụng chạy trực tiếp trên máy chủ quản lý bởi Systemd.

Tạo file cấu hình Workflow tại: `.github/workflows/deploy-systemd.yml`:

```yaml
name: CI/CD Pipeline - NestJS Systemd Deploy

# Kích hoạt khi có commit đẩy lên nhánh 'main'
on:
  push:
    branches: [ "main" ]

jobs:
  # ============================================================================
  # GIAI ĐOẠN 1: TÍCH HỢP LIÊN TỤC (CI - TEST & BUILD TRÊN RUNNER)
  # ============================================================================
  build-and-test:
    runs-on: ubuntu-latest

    steps:
      - name: 📥 Tải mã nguồn từ GitHub
        uses: actions/checkout@v4

      - name: 🟢 Thiết lập môi trường Node.js 20
        uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: 'npm'

      - name: 📦 Cài đặt Dependencies
        run: npm ci

      - name: 🧪 Chạy Kiểm thử (Unit Tests)
        run: npm run test --if-present

      - name: 🔨 Biên dịch thử nghiệm TypeScript
        run: npm run build

  # ============================================================================
  # GIAI ĐOẠN 2: TRIỂN KHAI LIÊN TỤC (CD - DEPLOY LÊN UBUNTU SERVER)
  # ============================================================================
  deploy:
    needs: build-and-test # Chỉ chạy khi giai đoạn CI ở trên thành công 100%
    runs-on: ubuntu-latest

    steps:
      - name: 🚀 SSH vào Server và Cập nhật Ứng dụng
        uses: appleboy/ssh-action@v1.0.3
        with:
          host: ${{ secrets.SSH_HOST }}
          username: ${{ secrets.SSH_USER }}
          key: ${{ secrets.SSH_PRIVATE_KEY }}
          port: ${{ secrets.SSH_PORT }}
          script: |
            echo "🚀 [1/5] Đi vào thư mục ứng dụng..."
            cd /home/ubuntu/apps/nestjs-api

            echo "📥 [2/5] Kéo mã nguồn mới nhất từ GitHub..."
            git reset --hard origin/main
            git pull origin main

            echo "📦 [3/5] Cập nhật dependencies..."
            npm ci

            echo "🔨 [4/5] Biên dịch ứng dụng sang JavaScript..."
            npm run build

            echo "🔄 [5/5] Khởi động lại dịch vụ Systemd..."
            sudo systemctl restart nestjs-api

            echo "🩺 [Healthcheck] Kiểm tra tình trạng hoạt động..."
            sleep 3
            curl -f http://127.0.0.1:3000/api/health || exit 1
            echo "✅ TRIỂN KHAI THÀNH CÔNG VÀ AN TOÀN 100%!"
```

---

## 🐳 5. Chiến Lược 2: Triển Khai Bằng Docker Compose (Hiện Đại)

Chiến lược chuẩn Cloud-Native: Runner build Docker Image $\rightarrow$ Đẩy lên Docker Hub $\rightarrow$ Báo Server tải bản Image mới về chạy.

Tạo file `.github/workflows/deploy-docker.yml`:

```yaml
name: CI/CD Pipeline - Docker Compose Deploy

on:
  push:
    branches: [ "main" ]

jobs:
  build-and-push-docker:
    runs-on: ubuntu-latest

    steps:
      - name: 📥 Checkout Code
        uses: actions/checkout@v4

      - name: 🔐 Đăng nhập vào Docker Hub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}

      - name: 🔨 Build và Push Docker Image
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ${{ secrets.DOCKERHUB_USERNAME }}/nestjs-api:latest

  deploy-to-server:
    needs: build-and-push-docker
    runs-on: ubuntu-latest

    steps:
      - name: 🚀 Kéo Docker Image mới và Khởi chạy lại
        uses: appleboy/ssh-action@v1.0.3
        with:
          host: ${{ secrets.SSH_HOST }}
          username: ${{ secrets.SSH_USER }}
          key: ${{ secrets.SSH_PRIVATE_KEY }}
          port: ${{ secrets.SSH_PORT }}
          script: |
            cd /home/ubuntu/apps/nestjs-api
            echo "📥 Kéo image mới nhất từ Docker Hub..."
            docker compose pull

            echo "🔄 Khởi động lại container mà không gián đoạn dịch vụ..."
            docker compose up -d --remove-orphans

            echo "🧹 Dọn dẹp image cũ không dùng..."
            docker system prune -f

            echo "🩺 Kiểm tra phản hồi API..."
            sleep 5
            curl -f http://127.0.0.1:3000/api/health || exit 1
            echo "✅ DOCKER CONTAINER ĐÃ ĐƯỢC TRIỂN KHAI THÀNH CÔNG!"
```

---

## 🛡️ 6. Thiết Lập Sudoers An Toàn Cho CI/CD (Không Cần Gõ Mật Khẩu Sudo)

Trong kịch bản Deploy bằng Systemd, tài khoản `ubuntu` cần chạy lệnh `sudo systemctl restart nestjs-api`. Để lệnh này không bị treo hỏi mật khẩu sudo khi chạy ngầm trong CI/CD, chúng ta cấu hình **Quy tắc đặc quyền tối thiểu (Least Privilege)** trong sudoers:

```bash
# Mở trình soạn thảo sudoers an toàn
sudo visudo -f /etc/sudoers.d/cicd-deploy
```

Thêm duy nhất 1 dòng cho phép user `ubuntu` restart dịch vụ cụ thể này:

```text
ubuntu ALL=(ALL) NOPASSWD: /usr/bin/systemctl restart nestjs-api, /usr/bin/systemctl reload nginx
```

*Lưu lại:* Bấm `Ctrl + O` $\rightarrow$ `Enter`, sau đó `Ctrl + X`.

---

## 🚨 7. Xử Lý 3 Sự Cố Phổ Biến Trong CI/CD

### 🔴 Sự cố 1: `Host key verification failed`
* **Nguyên nhân:** SSH Client của GitHub Runner không biết Fingerprint của máy chủ bạn.
* **Cách khắc phục:** Action `appleboy/ssh-action` đã tự động xử lý việc này. Nếu dùng script thuần, hãy thêm lệnh:
  ```bash
  ssh-keyscan -H -p ${{ secrets.SSH_PORT }} ${{ secrets.SSH_HOST }} >> ~/.ssh/known_hosts
  ```

---

### 🔴 Sự cố 2: `git pull` bị xung đột (Merge Conflict / Local changes)
* **Nguyên nhân:** Có ai đó đã đăng nhập vào server và tự ý sửa file code trực tiếp trên production.
* **Cách khắc phục:** Luôn đưa lệnh `git reset --hard origin/main` vào trước `git pull` để ép production phải khớp chính xác 100% với Git repository.

---

### 🔴 Sự cố 3: Pipeline báo Pass nhưng ứng dụng thực tế bị sập
* **Nguyên nhân:** Code build thành công nhưng lúc khởi động bị lỗi kết nối Database hoặc thiếu biến môi trường `.env`.
* **Cách khắc phục:** Bắt buộc phải có bước **Post-deployment Healthcheck** ở cuối (`curl -f http://127.0.0.1:3000/api/health || exit 1`). Nếu API không trả về status 200, pipeline sẽ lập tức báo đỏ và cảnh báo cho lập trình viên.

---

## 🧪 8. Bài Thực Hành Lab (10 Bước Hoàn Chỉnh)

*Hãy thực hiện kịch bản xây dựng hệ thống CI/CD tự động hóa 100%:*

1. Tạo cặp SSH Deploy Key riêng cho CI/CD: `ssh-keygen -t ed25519 -f ~/.ssh/github_deploy_key`.
2. Đưa Public Key lên server: `ssh-copy-id -i ~/.ssh/github_deploy_key.pub ubuntu@<IP_VM>`.
3. Kiểm tra kết nối SSH bằng Private Key từ máy Mac.
4. Cấu hình 4 thông số Secrets trên GitHub Repository (`SSH_HOST`, `SSH_USER`, `SSH_PRIVATE_KEY`, `SSH_PORT`).
5. Cấu hình file `/etc/sudoers.d/cicd-deploy` để cho phép restart service không cần gõ mật khẩu sudo.
6. Tạo thư mục `.github/workflows/` trong mã nguồn dự án.
7. Viết file `deploy.yml` với 2 giai đoạn: CI (Test & Build) và CD (SSH & Deploy).
8. Thực hiện một thay đổi nhỏ trong code (ví dụ đổi message API trong `app.service.ts`).
9. Đẩy code lên GitHub: `git add . && git commit -m "feat: test auto cicd" && git push origin main`.
10. Mở tab **Actions** trên GitHub, theo dõi Pipeline chạy thời gian thực và mở trình duyệt máy Mac kiểm chứng kết quả cập nhật ngay lập tức!

---

## 📌 9. Bảng Tra Cứu Lệnh CI/CD & Cú Pháp Workflow

| Cú pháp / Lệnh | Ý nghĩa & Chức năng |
| :--- | :--- |
| `on: push: branches: [main]` | Kích hoạt pipeline khi có commit đẩy lên nhánh main |
| `runs-on: ubuntu-latest` | Chỉ định môi trường máy ảo chạy Worker của GitHub |
| `needs: <job_name>` | Thiết lập sự phụ thuộc: Job sau chỉ chạy khi Job trước thành công |
| `${{ secrets.NAME }}` | Lấy giá trị biến bí mật an toàn từ GitHub Secrets |
| `appleboy/ssh-action` | Action chính thức dùng để SSH và thực thi lệnh trên remote server |
| `curl -f <url> \|\| exit 1` | Kiểm tra HTTP status: Báo lỗi và dừng pipeline nếu không phải 200 OK |
| `git reset --hard origin/main`| Đồng bộ hóa cưỡng bức mã nguồn trên server khớp với Git repo |
