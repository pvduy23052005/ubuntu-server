# 📘 CẨM NANG TRIỂN KHAI PRODUCTION FULL-STACK VỚI DOCKER COMPOSE

> **Motto cốt lõi:**  
> *Cô lập mạng an toàn $\rightarrow$ Zero Port Exposure (chỉ mở 80/443 qua Nginx) $\rightarrow$ Dữ liệu bền vững với Named Volume (Toàn bộ stack: MongoDB + NestJS/Express Backend + Next.js Frontend + Nginx Reverse Proxy).*

---

## 🏛️ 1. Sơ Đồ Kiến Trúc & Luồng Kết Nối Container

```text
INTERNET (Người dùng / Cloudflare Tunnel)
  │
  ▼ (Chỉ mở Port 80 & 443)
┌─────────────────────────────────────────────────────────────────────────┐
│ DOCKER HOST (Ubuntu Server VM)                                          │
│                                                                         │
│  [ nano-nginx ] (Cổng 80:80, 443:443)                                    │
│        │                                                                │
│        │ (Mạng nội bộ: nano-net - Bridge Network)                       │
│        ├────────────────────────────────┬───────────────────────────────┤
│        │                                │                               │
│        ▼ (HTTP :3000)                   ▼ (HTTP/WebSocket :5050)        │
│  [ nano-frontend ]              [ nano-backend ]                        │
│  (Next.js App)                  (NestJS / Express API & Socket.IO)      │
│  Image: ghcr.io/...             Image: ghcr.io/...                      │
│                                         │                               │
│                                         ▼ (mongodb:27017)               │
│                                 [ nano-mongodb ]                        │
│                                 (Database NoSQL - Cổng 27017 cô lập)    │
│                                         │                               │
│                                         ▼ (Mount Volume)                │
│                                 ┌──────────────┐                        │
│                                 │ mongodb_data │ (Lưu trên SSD Host)    │
│                                 └──────────────┘                        │
└─────────────────────────────────────────────────────────────────────────┘
```

### 🌟 Nguyên Tắc An Toàn Sống Còn (Zero Port Exposure):
* **Chỉ mở duy nhất Nginx ra ngoài:** Chỉ có container `nano-nginx` sử dụng chỉ thị `ports: ["80:80", "443:443"]`.
* **Frontend, Backend và Database hoàn toàn ẩn danh:** Các container này chỉ dùng chỉ thị `expose` hoặc lắng nghe cổng nội bộ. Kẻ tấn công từ Internet **tuyệt đối không thể quét thấy hay truy cập trực tiếp vào cổng 27017 (MongoDB), 5050 (Backend) hay 3000 (Frontend)**.
* **DNS Resolution nội bộ:** Trong mạng `nano-net`, các container tự phân giải địa chỉ bằng tên dịch vụ (ví dụ: backend kết nối tới `mongodb:27017`, Nginx chuyển tiếp tới `backend:5050` và `frontend:3000`).

---

## 📁 2. Cấu Trúc Thư Mục Chuẩn Mực Trên Máy Chủ

Khuyến nghị tổ chức thư mục ứng dụng tại `/home/ubuntu/apps/nano-box/` (hoặc `~/apps/nano-box/`):

```text
/home/ubuntu/apps/nano-box/
├── docker-compose.yml          ──► File điều phối toàn bộ cụm container
├── database/
│   └── .env                    ──► Chứa mật khẩu & tài khoản MongoDB (chmod 600)
├── nginx.conf                  ──► Cấu hình Reverse Proxy, HTTPS, GraphQL & Socket.IO
├── ssl/                        ──► Thư mục chứa chứng chỉ SSL (Let's Encrypt / Origin CA)
│   ├── fullchain.pem           ──► Chứng chỉ công khai (cert.pem)
│   └── privkey.pem             ──► Khóa bí mật (key.pem - chmod 600)
├── backend/
│   └── .env                    ──► Biến môi trường Backend (Mongo URI, Secret Key...)
└── frontend/
    └── .env                    ──► Biến môi trường Frontend (API URL, Public Key...)
```

---

## 📄 3. File Cấu Hình `docker-compose.yml` Hoàn Chỉnh

```yaml
version: '3.8'

services:
  # ============================================================================
  # 1. CƠ SỞ DỮ LIỆU MONGODB
  # ============================================================================
  mongodb:
    image: mongo:latest
    container_name: nano-mongodb
    restart: unless-stopped
    # Nạp tài khoản & mật khẩu từ file riêng, TUYỆT ĐỐI KHÔNG ghi lộ mật khẩu vào đây
    env_file:
      - ./database/.env
    volumes:
      # Named Volume đảm bảo dữ liệu không mất khi restart/recreate container
      - mongodb_data:/data/db
    networks:
      - nano-net

  # ============================================================================
  # 2. BACKEND API & REAL-TIME GATEWAY (NESTJS / EXPRESS)
  # ============================================================================
  backend:
    image: ghcr.io/pvduy23052005/nano-box-backend:latest
    container_name: nano-backend
    restart: unless-stopped
    expose:
      - "5050"
    env_file:
      - ./backend/.env
    depends_on:
      - mongodb
    networks:
      - nano-net

  # ============================================================================
  # 3. FRONTEND GIAO DIỆN NGƯỜI DÙNG (NEXT.JS)
  # ============================================================================
  frontend:
    image: ghcr.io/pvduy23052005/nano-box-frontend:latest
    container_name: nano-frontend
    restart: unless-stopped
    expose:
      - "3000"
    env_file:
      - ./frontend/.env
    networks:
      - nano-net

  # ============================================================================
  # 4. REVERSE PROXY & SSL TERMINATION (NGINX)
  # ============================================================================
  nginx:
    image: nginx:alpine
    container_name: nano-nginx
    restart: unless-stopped
    ports:
      # Mở cổng 80 (HTTP) và 443 (HTTPS) đón nhận truy cập ngoài
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
      - ./ssl:/etc/nginx/ssl:ro
    depends_on:
      - frontend
      - backend
    networks:
      - nano-net

# ==============================================================================
# QUẢN LÝ DỮ LIỆU BỀN VỮNG (VOLUMES)
# ==============================================================================
volumes:
  mongodb_data:
    name: nano_mongodb_data
    driver: local

# ==============================================================================
# MẠNG NỘI BỘ RIÊNG BIỆT (NETWORKS)
# ==============================================================================
networks:
  nano-net:
    name: nano_bridge_network
    driver: bridge
```

---

## ⚙️ 4. Thiết Lập Biến Môi Trường Chuẩn Production

> [!CAUTION]
> **Tuyệt đối KHÔNG hardcode mật khẩu trực tiếp trong `docker-compose.yml`!**  
> 1. File `docker-compose.yml` thường được push lên Git repository (GitHub/GitLab). Nếu ghi trực tiếp `MONGO_INITDB_ROOT_PASSWORD=...`, mật khẩu sẽ bị lộ công khai trong commit history vĩnh viễn.  
> 2. **Giải pháp chuẩn:** Tách toàn bộ thông tin đăng nhập ra file `database/.env`, đưa file này vào `.gitignore` và phân quyền `chmod 600` trên máy chủ.

---

### 4.1. File `database/.env` (Bảo mật tài khoản MongoDB)
Tạo file `database/.env`:

```env
MONGO_INITDB_ROOT_USERNAME=admin
MONGO_INITDB_ROOT_PASSWORD=YourUltraSecurePassword2026!
MONGO_INITDB_DATABASE=nano-db
```

---

### 4.2. File `backend/.env`
Tạo file `backend/.env` với các tham số kết nối trực tiếp tới container `mongodb`:

```env
NODE_ENV=production
PORT=5050

# Kết nối MongoDB qua DNS container 'mongodb' trong mạng nano-net
DATABASE_URL=mongodb://admin:YourUltraSecurePassword2026!@mongodb:27017/nano-db?authSource=admin
MONGO_URI=mongodb://admin:YourUltraSecurePassword2026!@mongodb:27017/nano-db?authSource=admin

# Các cấu hình bảo mật khác
JWT_SECRET=MySuperSecretProductionKey2026!
CORS_ORIGIN=https://nano-boxes.id.vn
```

---

### 4.3. File `frontend/.env`
Tạo file `frontend/.env` cho ứng dụng Next.js:

```env
NODE_ENV=production
PORT=3000

# URL gọi từ trình duyệt của client (đi qua Nginx domain thật)
NEXT_PUBLIC_API_URL=https://nano-boxes.id.vn/graphql
NEXT_PUBLIC_SOCKET_URL=https://nano-boxes.id.vn
```

---

### 4.4. Phân quyền bảo mật các file biến môi trường & SSL
```bash
chmod 600 database/.env backend/.env frontend/.env
chmod 600 ssl/privkey.pem
```

---

## 🚀 5. Quy Trình Khởi Chạy & Đăng Nhập GHCR Từng Bước

### Bước 1: Đăng nhập vào GitHub Container Registry (GHCR)
Vì image được lưu trên GHCR (`ghcr.io/pvduy23052005/...`), nếu repository ở chế độ Private, bạn cần đăng nhập bằng **Personal Access Token (Classic)** có quyền `read:packages`:

```bash
# Thay thế <YOUR_GITHUB_TOKEN> bằng mã Token của bạn
echo "<YOUR_GITHUB_TOKEN>" | docker login ghcr.io -u pvduy23052005 --password-stdin
```
*Output:* `Login Succeeded`

---

### Bước 2: Tải Image Mới Nhất Về Máy Chủ
```bash
cd ~/apps/nano-box
docker compose pull
```

---

### Bước 3: Khởi Chạy Toàn Bộ Các Dịch Vụ
```bash
# Chạy ở chế độ nền (Detached mode)
docker compose up -d
```

---

### Bước 4: Kiểm Tra Trạng Thái Hoạt Động (Status Check)
```bash
docker compose ps
```

*Output mong đợi:*
```text
NAME            IMAGE                                                   COMMAND                  SERVICE    STATUS
nano-backend    ghcr.io/pvduy23052005/nano-box-backend:latest          "docker-entrypoint.s…"   backend    Up
nano-frontend   ghcr.io/pvduy23052005/nano-box-frontend:latest         "docker-entrypoint.s…"   frontend   Up
nano-mongodb    mongo:latest                                            "docker-entrypoint.s…"   mongodb    Up
nano-nginx      nginx:alpine                                            "/docker-entrypoint.…"   nginx      Up (0.0.0.0:80->80/tcp, 0.0.0.0:443->443/tcp)
```

---

### Bước 5: Soi Log Trực Tiếp Theo Thời Gian Thực
```bash
# Xem log toàn bộ hệ thống
docker compose logs -f

# Xem riêng log của Backend hoặc Nginx
docker compose logs -f backend
docker compose logs -f nginx
```

---

## 🔄 6. Vận Hành Nâng Cao: Cập Nhật Ứng Dụng (Zero-Downtime Rollout)

Khi bạn đẩy code mới lên GitHub và GitHub Actions đã build xong image mới, hãy chạy các lệnh sau trên server để cập nhật mà **không làm gián đoạn CSDL MongoDB**:

```bash
# 1. Tải bản image mới nhất
docker compose pull backend frontend

# 2. Re-create chỉ riêng backend và frontend (MongoDB và Nginx vẫn chạy bình thường)
docker compose up -d --no-deps backend frontend

# 3. Dọn dẹp các image cũ không còn sử dụng (giải phóng dung lượng đĩa)
docker image prune -f
```

---

## 💾 7. Sao Lưu Dữ Liệu MongoDB Tự Động (Backup & Restore)

### 7.1. Lệnh Sao Lưu (Backup CSDL):
Dữ liệu được dump trực tiếp từ bên trong container ra máy chủ:

```bash
# Nạp thông tin mật khẩu từ file database/.env
source database/.env

# Tạo bản sao lưu định dạng BSON nén gzip
docker exec -t nano-mongodb mongodump \
  --username "$MONGO_INITDB_ROOT_USERNAME" \
  --password "$MONGO_INITDB_ROOT_PASSWORD" \
  --authenticationDatabase admin \
  --db "$MONGO_INITDB_DATABASE" \
  --archive > ~/backups/nano_db_$(date +%Y%m%d_%H%M%S).dump
```

### 7.2. Lệnh Khôi Phục (Restore CSDL):
```bash
# Nạp thông tin mật khẩu từ file database/.env
source database/.env

# Khôi phục dữ liệu từ file dump
docker exec -i nano-mongodb mongorestore \
  --username "$MONGO_INITDB_ROOT_USERNAME" \
  --password "$MONGO_INITDB_ROOT_PASSWORD" \
  --authenticationDatabase admin \
  --nsInclude="${MONGO_INITDB_DATABASE}.*" \
  --archive < ~/backups/nano_db_backup_file.dump
```

---

## 🚨 8. Sổ Tay Khắc Phục Sự Cố Kinh Điển (Troubleshooting)

| Lỗi | Nguyên nhân | Cách xử lý |
| :--- | :--- | :--- |
| **`Error response from daemon: Head "...": unauthorized`** | Chưa đăng nhập GHCR hoặc Token GitHub hết hạn / thiếu quyền `read:packages`. | Tạo lại GitHub PAT (Classic) với quyền `read:packages` và chạy: `echo $TOKEN \| docker login ghcr.io -u <username> --password-stdin`. |
| **Backend báo: `MongoServerSelectionError: connect ECONNREFUSED`** | Backend khởi động quá nhanh trước khi MongoDB kịp mở socket `27017`. | Kiểm tra log mongo: `docker compose logs mongodb`. Đảm bảo trong `backend/.env` dùng `mongodb:27017` chứ không phải `localhost:27017`. |
| **Nginx báo: `502 Bad Gateway`** | Container `nano-backend` hoặc `nano-frontend` bị crash hoặc chưa khởi động xong. | Kiểm tra `docker compose ps` xem có container nào hiển thị `Restarting` không. Đọc log bằng: `docker compose logs backend`. |
| **Mất dữ liệu sau khi chạy `docker compose down`** | Khởi chạy container mà không chỉ định volume lưu trữ. | Khi dừng container, chỉ dùng `docker compose down`. **Tuyệt đối KHÔNG gõ `docker compose down -v`** (tham số `-v` sẽ xóa vĩnh viễn named volume!). |
