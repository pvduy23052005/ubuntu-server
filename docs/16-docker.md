# 📘 Phần 16: Docker & Containerization – Đóng Gói & Triển Khai Ứng Dụng Backend Chuẩn Doanh Nghiệp

> **Motto cốt lõi:**  
> *"Nó chạy được trên máy tôi!" $\rightarrow$ Đóng gói cả môi trường vào Docker Container để chạy được ở MỌI NƠI!*  
> Chu trình chuẩn: **Bản chất VM vs Container → Kiến trúc Dockerfile Multi-Stage → Quản lý Image & Container → Điều phối Docker Compose Full-Stack → Giới hạn tài nguyên & Lab.**

---

## 🎯 1. Mục Tiêu Của Phần Học

Sau khi hoàn thành phần này, bạn sẽ:
1. **Hiểu bản chất Containerization vs Virtual Machine (VM):** Biết cơ chế nhân Linux (**Namespaces** và **Cgroups**) tạo ra sự cô lập siêu nhẹ của Container mà không cần ảo hóa phần cứng như VM.
2. **Làm chủ quy trình 3 bước của Docker:** `Dockerfile` (Công thức đóng gói) $\rightarrow$ `Docker Image` (Bản mẫu bất biến) $\rightarrow$ `Docker Container` (Tiến trình sống đang chạy).
3. **Thành thạo kỹ thuật Multi-Stage Build:** Đóng gói ứng dụng NestJS/TypeScript từ kích thước 1GB xuống chỉ còn **~120MB**, loại bỏ sạch rác và devDependencies.
4. **Tối ưu hóa Dockerfile với `.dockerignore` & Non-root User:** Tăng tốc độ build gấp 5 lần và gia cố bảo mật (chạy bằng `USER node` thay vì root).
5. **Làm chủ Docker CLI:** Tạo, chạy ngầm, gán cổng (Port Mapping), truyền biến môi trường, xem log và debug bên trong container.
6. **Điều phối đa dịch vụ với Docker Compose:** Kết nối đồng bộ Nginx Gateway + Frontend Next.js + Backend NestJS + PostgreSQL Database trên cùng một **Docker Bridge Network**.
7. **Thiết lập giới hạn tài nguyên (Resource Limits):** Khống chế CPU và RAM để container không bao giờ chiếm dụng làm sập máy chủ.

---

## 🧠 2. Bản Chất: Virtual Machine (VM) vs Docker Container

Tại sao chúng ta đã dựng Ubuntu VM bằng Multipass ở **Phần 01**, nhưng khi triển khai ứng dụng thực tế lại cần **Docker**?

```text
┌─────────────────────────────────────────────────────────────────────────┐
│                    VIRTUAL MACHINE (VM) vs DOCKER CONTAINER             │
├────────────────────────────────────┬────────────────────────────────────┤
│  VIRTUAL MACHINE (Multipass/VMware)│  DOCKER CONTAINER                  │
│                                    │                                    │
│   ┌────────────────────────────┐   │   ┌────────────────────────────┐   │
│   │ App A  │ App B  │ App C    │   │   │ App A  │ App B  │ App C    │   │
│   ├────────────────────────────┤   │   ├────────────────────────────┤   │
│   │ Bins/Libs (Mỗi VM một bộ)  │   │   │ Bins/Libs (Cô lập riêng)   │   │
│   ├────────────────────────────┤   │   ├────────────────────────────┤   │
│   │ GUEST OS (Kernel riêng)    │   │   │ DOCKER ENGINE (Daemon)     │   │
│   ├────────────────────────────┤   │   ├────────────────────────────┤   │
│   │ HYPERVISOR (Ảo hóa P.Cứng) │   │   │ LINUX KERNEL CHUNG         │   │
│   ├────────────────────────────┤   │   │ (Namespaces + Cgroups)     │   │
│   │ HOST OS (macOS)            │   │   ├────────────────────────────┤   │
│   └────────────────────────────┘   │   │ MÁY CHỦ VẬT LÝ / VM        │   │
│                                    │   └────────────────────────────┘   │
│  - Nặng (Chiếm 2GB - 20GB Disk)    │  - Siêu nhẹ (Chỉ vài chục MB)      │
│  - Khởi động mất 20 - 60 giây      │  - Khởi động trong 50 mili-giây    │
│  - Tiêu tốn nhiều RAM của Host     │  - Dùng chung tài nguyên CPU/RAM   │
└────────────────────────────────────┴────────────────────────────────────┘
```

### Hai trụ cột của Linux tạo nên Docker:
1. **Linux Namespaces:** Tạo sự cô lập hoàn toàn về: Mã tiến trình (PID), Hệ thống file (Mount), Mạng (Network) và Người dùng (User). Container tưởng mình là một hệ điều hành riêng biệt.
2. **Control Groups (Cgroups):** Đo lường và giới hạn lượng RAM, CPU và I/O mà một container được phép sử dụng.

---

## 🏗️ 3. Kỹ Thuật Đóng Gói NestJS: Multi-Stage Dockerfile Chuẩn Production

Khi đóng gói một ứng dụng NestJS/TypeScript, chúng ta gặp một bài toán lớn:
* Trình biên dịch TypeScript (`tsc`) và các gói `@types/...` chỉ cần lúc **Build**.
* Khi chạy **Production**, chúng ta chỉ cần thư mục `dist/` (JavaScript thuần) và các thư viện `dependencies` tối thiểu.

$\rightarrow$ **Multi-Stage Build** giải quyết triệt để vấn đề này bằng cách tạo 2 phòng thí nghiệm riêng biệt trong cùng 1 Dockerfile:

```text
┌─────────────────────────────────────────────────────────────────────────┐
│                     KỸ THUẬT MULTI-STAGE BUILD NESTJS                   │
├─────────────────────────────────────────────────────────────────────────┤
│  STAGE 1: BUILDER (Phòng chế tạo - Nặng ~1GB)                           │
│  - Cài đặt toàn bộ dependencies (kể cả devDependencies)                 │
│  - Chạy `npm run build` để sinh ra thư mục `dist/`                      │
│                                  │                                      │
│                                  │ (CHỈ COPY THÀNH PHẨM SANG STAGE 2)   │
│                                  ▼                                      │
│  STAGE 2: PRODUCTION RUNTIME (Thành phẩm siêu nhẹ - Chỉ ~120MB)         │
│  - Dùng Base Image siêu nhẹ: `node:20-alpine`                           │
│  - Chỉ cài đặt `dependencies` (bỏ sạch devDependencies)                │
│  - Copy thư mục `dist/` từ Stage 1 sang                                 │
│  - Chuyển sang user không có quyền root: `USER node`                   │
│  - Khởi chạy bằng: `node dist/main.js`                                  │
└─────────────────────────────────────────────────────────────────────────┘
```

---

### Bước 3.1: Tạo file `.dockerignore` (Bắt buộc)
Đứng tại thư mục dự án `~/apps/nestjs-api`:
```bash
nano ~/apps/nestjs-api/.dockerignore
```

Dán nội dung loại bỏ các file rác không gửi vào Docker Build Context:
```text
node_modules
dist
.git
.gitignore
.env
*.log
npm-debug.log
```

---

### Bước 3.2: Viết file `Dockerfile` Multi-Stage

```bash
nano ~/apps/nestjs-api/Dockerfile
```

Dán nội dung Dockerfile chuẩn Production:

```dockerfile
# ==============================================================================
# STAGE 1: XÂY DỰNG VÀ BIÊN DỊCH MÃ NGUỒN (BUILD STAGE)
# ==============================================================================
FROM node:20-alpine AS builder

WORKDIR /usr/src/app

# Copy các file khai báo dependencies trước để tận dụng Docker Layer Caching
COPY package*.json ./

# Cài đặt đầy đủ dependencies để build TypeScript
RUN npm ci

# Copy toàn bộ mã nguồn vào
COPY . .

# Biên dịch TypeScript sang JavaScript thuần trong thư mục dist/
RUN npm run build

# ==============================================================================
# STAGE 2: MÔI TRƯỜNG THỰC THI PRODUCTION (PRODUCTION RUNTIME)
# ==============================================================================
FROM node:20-alpine AS production

WORKDIR /usr/src/app

# Thiết lập biến môi trường production
ENV NODE_ENV=production

# Copy file định nghĩa package
COPY package*.json ./

# Chỉ cài đặt các package chạy production (bỏ qua devDependencies)
RUN npm ci --omit=dev && npm cache clean --force

# Copy thư mục dist đã biên dịch từ Stage 1 sang
COPY --from=builder /usr/src/app/dist ./dist

# Đổi sang user thường 'node' có sẵn trong Alpine Linux để tăng bảo mật
USER node

# Khai báo cổng ứng dụng lắng nghe
EXPOSE 3000

# Lệnh khởi chạy ứng dụng
CMD ["node", "dist/main.js"]
```

---

## 🛠️ 4. Xây Dựng Image & Quản Lý Container Bằng Docker CLI

### Bước 4.1: Xây dựng Docker Image (Build Image)
```bash
cd ~/apps/nestjs-api

# Build image với tên 'nestjs-api' và tag 'v1.0'
docker build -t nestjs-api:v1.0 .
```

Kiểm tra danh sách và dung lượng image vừa tạo:
```bash
docker images
```
*Quan sát thấy:* `nestjs-api:v1.0` chỉ có dung lượng khoảng **~130MB**!

---

### Bước 4.2: Khởi chạy Container từ Image

```bash
# -d: Chạy ngầm (Detached mode)
# -p 3000:3000: Ánh xạ Port 3000 của Host vào Port 3000 của Container
# --name: Đặt tên cho Container dễ quản lý
# --restart always: Tự khởi động lại khi crash hoặc reboot máy chủ
# --memory="512m": Giới hạn RAM tối đa 512MB
docker run -d \
  -p 3000:3000 \
  --name nestjs_backend \
  --restart always \
  --memory="512m" \
  nestjs-api:v1.0
```

---

### Bước 4.3: Các câu lệnh kiểm tra và điều tra Container

```bash
# 1. Xem danh sách các Container đang chạy
docker ps

# 2. Xem mức tiêu thụ RAM & CPU thời gian thực của Container
docker stats nestjs_backend --no-stream

# 3. Xem nhật ký log của ứng dụng bên trong Container
docker logs -f nestjs_backend

# 4. Chui vào bên trong Container để kiểm tra file (Debugging)
docker exec -it nestjs_backend sh
# (Gõ 'exit' để thoát ra)
```

Gửi request kiểm tra từ máy chủ:
```bash
curl http://127.0.0.1:3000/
```
*Kết quả:* Trả về JSON trạng thái từ ứng dụng đang chạy bên trong Docker Container!

---

## 🎼 5. Điều Phối Toàn Diện Với Docker Compose (Full-Stack: Next.js + NestJS + Postgres + Nginx)

Trong môi trường Production doanh nghiệp, một hệ thống hoàn chỉnh thường bao gồm 4 thành phần: **Nginx Gateway + Frontend Next.js (SSR) + Backend NestJS (API) + PostgreSQL Database**. Toàn bộ được cô lập an toàn trên cùng một mạng **Docker Bridge Network**:

```text
                                MẠNG INTERNET
                                      │
                                      │ (HTTP Port 80 / HTTPS Port 443)
                                      ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                           UBUNTU SERVER VM                              │
│                                                                         │
│ ┌ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ┐ │
│   DOCKER BRIDGE NETWORK (app_network)                                   │
│ │                                                                     │ │
│   ┌───────────────────────────────────────────────────────────────┐     │
│ │ │ CONTAINER: nginx_gateway (CỔNG VÀO DUY NHẤT MỞ RA NGOÀI HOST) │   │ │
│   │ ──► Mở Port 80 & 443 từ Internet                              │     │
│ │ │ ──► Điều hướng /api/* ──► http://api_backend:4000             │   │ │
│   │ ──► Điều hướng /*     ──► http://frontend_web:3000           │     │
│ │ └───────────────────────────────┬───────────────────────────────┘   │ │
│                                   │                                     │
│ │                 ┌───────────────┴───────────────┐                   │ │
│                   ▼                               ▼                     │
│ │ ┌───────────────────────────────┐ ┌───────────────────────────────┐ │ │
│   │ CONTAINER: frontend_web       │ │ CONTAINER: api_backend        │   │
│ │ │ (Next.js SSR - Port 3000)     │ │ (NestJS API - Port 4000)      │ │ │
│   └───────────────────────────────┘ └───────────────┬───────────────┘   │
│ │                                                   │                 │ │
│                                                     ▼ (Nội bộ)          │
│ │                                   ┌───────────────────────────────┐ │ │
│                                     │ CONTAINER: postgres_db        │   │
│ │                                   │ (Port 5432 - Không mở ra host)│ │ │
│                                     │ ──► Named Volume: postgres_data│  │
│ │                                   └───────────────────────────────┘ │ │
│ └ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ┘ │
└─────────────────────────────────────────────────────────────────────────┘
```

> [!TIP]
> **Quy tắc điều phối 4 dịch vụ:**  
> 1. **Zero Host Exposure:** Cả `frontend_web`, `api_backend` và `postgres_db` đều chạy ngầm bên trong mạng ảo `app_network`, không mở cổng trực tiếp ra ngoài máy chủ.
> 2. **Nginx làm Traffic Controller:** Nginx nhận request từ người dùng, nếu URL bắt đầu bằng `/api/` thì chuyển tới NestJS, còn lại toàn bộ giao diện (SSR, trang HTML, static assets) chuyển tới Next.js.

---

### Bước 5.1: Cấu trúc thư mục dự án Full-Stack

Chúng ta tổ chức mã nguồn theo mô hình Monorepo / Multi-Service rõ ràng:

```text
~/apps/my-project/
├── docker-compose.yml
├── nginx/
│   └── nginx.conf                 ◄── File điều hướng Nginx Reverse Proxy
├── backend/                       ◄── Mã nguồn NestJS API
│   ├── Dockerfile                 ◄── Multi-Stage Build NestJS (từ Mục 3)
│   ├── .dockerignore
│   └── src/
└── frontend/                      ◄── Mã nguồn Next.js
    ├── Dockerfile                 ◄── Multi-Stage Standalone Next.js
    ├── .dockerignore
    ├── next.config.mjs (hoặc js)
    └── src/ (hoặc app/)
```

---

### Bước 5.2: Dockerfile Multi-Stage Cho Next.js (Standalone Mode)

> [!IMPORTANT]
> Để kích hoạt chế độ đóng gói siêu nhẹ của Next.js, hãy thêm `output: "standalone"` vào file `next.config.mjs`:
> ```javascript
> /** @type {import('next').NextConfig} */
> const nextConfig = {
>   output: "standalone",
> };
> export default nextConfig;
> ```

Tạo file `~/apps/my-project/frontend/Dockerfile`:
```dockerfile
# STAGE 1: Cài đặt dependencies
FROM node:20-alpine AS deps
RUN apk add --no-cache libc6-compat
WORKDIR /app
COPY package.json package-lock.json* ./
RUN npm ci

# STAGE 2: Build source code với Standalone Output
FROM node:20-alpine AS builder
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .
ENV NEXT_TELEMETRY_DISABLED=1
RUN npm run build

# STAGE 3: Production Runner siêu nhẹ (~90MB)
FROM node:20-alpine AS runner
WORKDIR /app
ENV NODE_ENV=production
ENV NEXT_TELEMETRY_DISABLED=1
ENV PORT=3000
ENV HOSTNAME="0.0.0.0"

RUN addgroup --system --gid 1001 nodejs && \
    adduser --system --uid 1001 nextjs

# Copy static assets và standalone build thành phẩm
COPY --from=builder /app/public ./public
COPY --from=builder --chown=nextjs:nodejs /app/.next/standalone ./
COPY --from=builder --chown=nextjs:nodejs /app/.next/static ./.next/static

USER nextjs
EXPOSE 3000
CMD ["node", "server.js"]
```

---

### Bước 5.3: Cấu hình Nginx Gateway Điều Hướng Kép (`nginx/nginx.conf`)

Tạo file `~/apps/my-project/nginx/nginx.conf`:

```nginx
events { worker_connections 1024; }

http {
    # 1. Chuyển hướng toàn bộ HTTP sang HTTPS
    server {
        listen 80;
        server_name _;
        return 301 https://$host$request_uri;
    }

    # 2. Cấu hình HTTPS bảo mật
    server {
        listen 443 ssl;
        server_name _;

        # Đường dẫn chứng chỉ mount từ Host (/etc/letsencrypt -> /etc/nginx/ssl)
        ssl_certificate /etc/nginx/ssl/live/your_domain.com/fullchain.pem;
        ssl_certificate_key /etc/nginx/ssl/live/your_domain.com/privkey.pem;

        ssl_protocols TLSv1.2 TLSv1.3;
        ssl_ciphers HIGH:!aNULL:!MD5;
        client_max_body_size 20M;

        # ──► A. ĐIỀU HƯỚNG API BACKEND (NestJS)
        location /api/ {
            proxy_pass http://api_backend:4000;
            proxy_http_version 1.1;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection 'upgrade';
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
            proxy_cache_bypass $http_upgrade;
        }

        # ──► B. ĐIỀU HƯỚNG FRONTEND (Next.js SSR & Static Assets)
        location / {
            proxy_pass http://frontend_web:3000;
            proxy_http_version 1.1;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection 'upgrade';
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
            proxy_cache_bypass $http_upgrade;
        }
    }
}
```

> [!IMPORTANT]
> **Tại sao mount cả thư mục `/etc/letsencrypt` vào container?**  
> Các file trong `/etc/letsencrypt/live/your_domain.com/` thực chất là các **Symbolic Link** trỏ tương đối tới `../../archive/your_domain.com/`. Mount toàn bộ `/etc/letsencrypt` vào `/etc/nginx/ssl:ro` giúp bảo toàn symlink và không bị gãy liên kết khi chứng chỉ được tự động gia hạn.

---

### Bước 5.4: File `docker-compose.yml` Hoàn Chỉnh 4 Dịch Vụ

Tạo file `~/apps/my-project/docker-compose.yml`:

```yaml
version: '3.8'

services:
  # 1. CƠ SỞ DỮ LIỆU POSTGRESQL
  postgres_db:
    image: postgres:16-alpine
    container_name: postgres_db
    restart: always
    environment:
      POSTGRES_USER: devuser
      POSTGRES_PASSWORD: MySecurePassword2026!
      POSTGRES_DB: nestjs_db
    volumes:
      - postgres_data:/var/lib/postgresql/data
    networks:
      - app_network
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U devuser -d nestjs_db"]
      interval: 5s
      timeout: 5s
      retries: 5

  # 2. BACKEND API (NestJS - Lắng nghe Port 4000 nội bộ)
  api_backend:
    build:
      context: ./backend
      dockerfile: Dockerfile
    container_name: nestjs_api_service
    restart: always
    environment:
      NODE_ENV: production
      PORT: 4000
      DATABASE_HOST: postgres_db
      DATABASE_PORT: 5432
      DATABASE_USER: devuser
      DATABASE_PASSWORD: MySecurePassword2026!
      DATABASE_NAME: nestjs_db
    depends_on:
      postgres_db:
        condition: service_healthy
    deploy:
      resources:
        limits:
          cpus: '1.0'
          memory: 512M
    networks:
      - app_network

  # 3. FRONTEND WEB (Next.js - Lắng nghe Port 3000 nội bộ)
  frontend_web:
    build:
      context: ./frontend
      dockerfile: Dockerfile
    container_name: nextjs_web_service
    restart: always
    environment:
      NODE_ENV: production
      PORT: 3000
    deploy:
      resources:
        limits:
          cpus: '1.0'
          memory: 512M
    networks:
      - app_network

  # 4. NGINX GATEWAY (Cửa ngõ duy nhất mở Port 80/443 ra Host)
  nginx_gateway:
    image: nginx:alpine
    container_name: nginx_gateway
    restart: always
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf:ro
      - /etc/letsencrypt:/etc/nginx/ssl:ro
    depends_on:
      - api_backend
      - frontend_web
    networks:
      - app_network

networks:
  app_network:
    driver: bridge

volumes:
  postgres_data:
```

---

### Bước 5.5: Khởi chạy và kiểm tra hệ thống Full-Stack

> [!WARNING]
> **Xử lý xung đột cổng 80:** Hãy đảm bảo đã tắt Nginx trên host trước khi chạy cụm container:  
> `sudo systemctl stop nginx && sudo systemctl disable nginx`

```bash
# 1. Build và khởi chạy toàn bộ 4 dịch vụ
docker compose up -d --build

# 2. Kiểm tra cả 4 container đều đang chạy ổn định
docker compose ps

# 3. Xem log thời gian thực của cả hệ thống
docker compose logs -f

# 4. Kiểm tra phản hồi:
# - Truy cập Frontend Next.js:
curl -i http://localhost/

# - Truy cập Backend NestJS API:
curl -i http://localhost/api/
```

---

## 🧹 6. Dọn Dẹp Tài Nguyên Rác (Docker Housekeeping)

Trong quá trình build và cập nhật code, Docker sẽ tích lũy các Image cũ (Dangling Images) làm đầy ổ cứng:

```bash
# 1. Dọn dẹp các container đã tắt, network thừa và image không dùng
docker system prune -f

# 2. Xóa triệt để toàn bộ image rác cũ (Không xóa Volume dữ liệu)
docker system prune -a --volumes=false

# 3. Kiểm tra dung lượng ổ đĩa mà Docker đang chiếm dụng
docker system df
```

---

## 🚨 7. Xử Lý 3 Sự Cố Thường Gặp Khi Dùng Docker

### 🔴 Sự cố 1: Lỗi `permission denied while trying to connect to the Docker daemon socket`
* **Nguyên nhân:** User `ubuntu` chưa được thêm vào group `docker`.
* **Cách sửa:** Chạy `sudo usermod -aG docker $USER`, sau đó đăng xuất SSH và đăng nhập lại.

---

### 🔴 Sự cố 2: Container thoát ngay lập tức (`Exited (1)`)
* **Nguyên nhân:** Mã nguồn bị lỗi Crash lúc khởi động hoặc lệnh `CMD` trong Dockerfile viết sai.
* **Cách sửa:** Soi log lỗi bằng lệnh: `docker logs <tên_container>`.

---

### 🔴 Sự cố 3: Lỗi đụng cổng (`Bind for 0.0.0.0:3000 failed: port is already allocated`)
* **Nguyên nhân:** Cổng 3000 trên máy chủ đang bị Systemd Service cũ hoặc tiến trình khác chiếm giữ.
* **Cách sửa:** Dừng service cũ: `sudo systemctl stop nestjs-api` hoặc tìm PID chiếm cổng: `sudo ss -tulpn | grep :3000`.

---

## 🧪 8. Bài Thực Hành Lab (10 Bước Triển Khai)

*Hãy thực hiện toàn bộ kịch bản đóng gói và triển khai ứng dụng NestJS qua Docker trên Ubuntu Server VM:*

1. Cài đặt Docker và Docker Compose trên Ubuntu Server VM.
2. Tạo file `.dockerignore` tại thư mục dự án `~/apps/nestjs-api`.
3. Viết `Dockerfile` Multi-Stage tối ưu hóa với `node:20-alpine`.
4. Tiến hành build Docker Image: `docker build -t nestjs-api:v1.0 .`.
5. Kiểm tra kích thước Image với `docker images` (xác nhận dung lượng < 150MB).
6. Viết `Dockerfile` cho Frontend Next.js, tạo cấu hình `nginx/nginx.conf` và viết file `docker-compose.yml` kết nối Full-Stack 4 dịch vụ (Nginx Gateway + Next.js + NestJS + PostgreSQL).
7. Khởi chạy toàn bộ hệ thống bằng `docker compose up -d`.
8. Kiểm tra trạng thái container và healthcheck với `docker compose ps`.
9. Kiểm tra kết nối từ máy Mac qua Nginx Reverse Proxy: `http://<IP_VM>/`.
10. Thử nghiệm dọn dẹp hệ thống bằng `docker system prune` và kiểm tra `docker system df`.

---

## 📌 9. Bảng Tra Cứu Lệnh Docker Cốt Lõi

| Lệnh | Ý nghĩa & Khi nào dùng |
| :--- | :--- |
| `docker build -t <tag> .` | Đóng gói mã nguồn thành Docker Image |
| `docker run -d -p 3000:3000 <image>` | Khởi chạy Image thành một Container chạy ngầm |
| `docker ps` | Xem danh sách các Container đang hoạt động |
| `docker ps -a` | Xem toàn bộ Container (kể cả các container đã dừng) |
| `docker logs -f <container>` | Theo dõi Live Stream log của Container |
| `docker exec -it <container> sh` | Truy cập vào Terminal bên trong Container |
| `docker stop <container>` / `docker rm <container>` | Dừng và xóa một Container |
| `docker compose up -d` | Khởi chạy toàn bộ cụm dịch vụ được định nghĩa trong YAML |
| `docker compose down` | Dừng và hủy cụm container (Vẫn bảo toàn dữ liệu volume) |
| `docker system df` / `prune` | Kiểm tra và dọn dẹp các tài nguyên thừa của Docker |
