# 📘 Phần 09: Node.js & NestJS – Thiết Lập Môi Trường Thực Thi Backend Trên Ubuntu Server

> **Motto cốt lõi:**  
> *Hiểu vòng đời ứng dụng: TypeScript `src/` (Dev) $\rightarrow$ JavaScript `dist/` (Prod) | Production dùng `npm ci` và `node dist/main.js`, tuyệt đối không chạy `npm run start:dev` trên Server!*  
> Chu trình chuẩn: **Khái niệm → Bản chất Runtime → Cài đặt chuẩn Production → Build & Triển khai → Xử lý sự cố → Hands-on Lab.**

---

## 🎯 1. Mục Tiêu Của Phần Học

Sau khi hoàn thành phần này, bạn sẽ:
1. **Hiểu bản chất Node.js Runtime trên Linux:** V8 Engine, Libuv, Event Loop và cơ chế Non-blocking I/O vận hành như thế nào trong môi trường máy chủ.
2. **Lựa chọn đúng phương pháp cài đặt Node.js:** So sánh `apt` mặc định, **NodeSource Repository** (Chuẩn Production) và **NVM/FNM** (Môi trường Dev đa phiên bản).
3. **Làm chủ quy trình đóng gói & triển khai NestJS:** Phân biệt rõ ranh giới giữa môi trường **Development** (`npm run start:dev`) và **Production** (`npm run build` $\rightarrow$ `node dist/main.js`).
4. **Tối ưu hóa Dependencies:** Hiểu sự khác biệt sống còn giữa `npm install` và `npm ci --omit=dev`.
5. **Quản lý biến môi trường an toàn:** Thiết lập file `.env` chuẩn trên server và nạp cấu hình bảo mật vào ứng dụng.
6. **Nhận thức vấn đề Process Lifecycle:** Hiểu vì sao khi tắt kết nối SSH thì ứng dụng Node.js bị tắt theo, chuẩn bị nền tảng chuyển giao sang **[Phần 10: Systemd]**.
7. **Xử lý 4 lỗi kinh điển:** `EADDRINUSE: 3000`, `Cannot find module`, Lỗi thiếu RAM khi build TypeScript (Out of Memory), và Lỗi quên nạp biến môi trường.

---

## 🧠 2. Bản Chất Node.js Runtime & Vòng Đời Ứng Dụng NestJS

### 2.1. Node.js hoạt động như thế nào trên Linux Server?

```text
┌─────────────────────────────────────────────────────────────────────────┐
│                    KIẾN TRÚC NODE.JS TRÊN LINUX SERVER                  │
├─────────────────────────────────────────────────────────────────────────┤
│  Mã nguồn JavaScript (Code của bạn: dist/main.js)                       │
│                                  │                                      │
│                                  ▼                                      │
│  V8 JavaScript Engine (Google) ──► Biên dịch JS sang mã máy nhị phân    │
│                                  │                                      │
│                                  ▼                                      │
│  Libuv (C++) ───────────────────► Quản lý Event Loop, Thread Pool,      │
│                                    Non-blocking Socket Network, File I/O│
│                                  │                                      │
│                                  ▼                                      │
│  Linux Kernel (Ubuntu) ─────────► epoll / System Calls / Network Socket │
└─────────────────────────────────────────────────────────────────────────┘
```

* **Single-Threaded Event Loop:** Node.js xử lý tất cả các request đến trên một luồng chính duy nhất (Main Thread). Khi có tác vụ I/O nặng (đọc ghi đĩa, query Database), Libuv sẽ đẩy xuống các luồng ngầm (Thread Pool) hoặc giao cho Linux Kernel (`epoll`), giúp server phục vụ hàng ngàn request đồng thời với lượng RAM cực thấp.

---

### 2.2. Vòng đời ứng dụng NestJS: Dev vs Production

NestJS được viết bằng **TypeScript** (ngôn ngữ có kiểu dữ liệu tĩnh). Tuy nhiên, **Node.js chỉ có thể đọc và thực thi JavaScript thuần**.

```text
┌──────────────────────────────────┐            ┌──────────────────────────────────┐
│   DEVELOPMENT (Trên máy Mac)     │            │    PRODUCTION (Trên Ubuntu VM)   │
├──────────────────────────────────┤            ├──────────────────────────────────┤
│ - Mã nguồn: src/*.ts             │            │ - Mã nguồn: dist/*.js            │
│ - Chạy qua: ts-node (JIT Compile)│            │ - Chạy qua: node dist/main.js    │
│ - Lệnh: npm run start:dev        │            │ - Lệnh: node dist/main.js        │
│ - Dung lượng RAM: ~300MB - 600MB │            │ - Dung lượng RAM: ~60MB - 120MB  │
│ - Mục đích: Hot-reload khi code  │            │ - Mục đích: Tối đa hóa tốc độ,   │
│   (KHÔNG BAO GIỜ DÙNG TRÊN PROD) │            │   tiết kiệm RAM, ổn định 100%    │
└──────────────────────────────────┘            └──────────────────────────────────┘
```

```text
                           QUY TRÌNH BUILD NESTJS
                           
      src/main.ts (TypeScript)
      src/app.module.ts
      src/users/*.ts
                 │
                 │ npm run build (TypeScript Compiler: tsc)
                 ▼
      dist/main.js (JavaScript thuần tối ưu)
      dist/app.module.js
      dist/users/*.js
```

---

## 🛠️ 3. Cài Đặt Node.js Trên Ubuntu Server Bằng NVM (Node Version Manager)

**NVM (Node Version Manager)** là công cụ quản lý phiên bản Node.js chuẩn mực và linh hoạt nhất hiện nay. NVM cho phép bạn dễ dàng cài đặt, chuyển đổi và quản lý nhiều phiên bản Node.js khác nhau trên cùng một máy chủ mà không bị lỗi quyền hạn `sudo/root`.

```text
┌───────────────────────────┬───────────────────┬─────────────────────────────┐
│ Phương pháp               │ Ưu điểm           │ Đánh giá                    │
├───────────────────────────┼───────────────────┼─────────────────────────────┤
│ 1. NVM (Node Version Mgr) │ Đổi version linh  │ KHUYẾN NGHỊ: Cài đặt dễ     │
│                           │ hoạt, không cần   │ dàng, quản lý phiên bản độc │
│                           │ sudo khi npm -g   │ lập cho từng dự án          │
│ 2. NodeSource (APT Repo)  │ Chuẩn hệ thống APT│ Thích hợp cho môi trường OS │
│                           │ toàn cục          │ chỉ chạy 1 phiên bản cố định│
│ 3. `apt install nodejs`   │ Có sẵn trong OS   │ Không nên dùng (bản rất cũ) │
└───────────────────────────┴───────────────────┴─────────────────────────────┘
```

---

### Các bước tải và cài đặt Node.js 24 qua NVM (Từng bước chi tiết)

Đăng nhập vào Ubuntu Server VM và thực hiện chuỗi lệnh sau:

```bash
# Bước 1: Tải và cài đặt NVM (Phiên bản v0.40.7 mới nhất)
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.7/install.sh | bash

# Bước 2: Nạp môi trường NVM vào shell hiện tại (không cần khởi động lại Terminal)
\. "$HOME/.nvm/nvm.sh"

# Bước 3: Tải và cài đặt phiên bản Node.js 24
nvm install 24

# Bước 4: Thiết lập phiên bản 24 làm phiên bản mặc định của hệ thống
nvm alias default 24
nvm use 24

# Bước 5: Kiểm tra xác nhận phiên bản Node.js và NPM
node -v # In ra phiên bản Node.js (ví dụ: v24.x.x)
npm -v  # In ra phiên bản NPM (ví dụ: 11.x.x)
```

> [!TIP]
> **Ưu điểm lớn khi dùng NVM:**  
> Khi bạn cài đặt các package toàn cục (ví dụ: `npm install -g @nestjs/cli`, `npm install -g pnpm`), bạn **KHÔNG CẦN gõ `sudo`**, vì toàn bộ binary được lưu trong thư mục người dùng `~/.nvm/versions/node/` thay vì `/usr/local/bin/`.

```bash
# (Tùy chọn) Cài đặt thêm các Package Manager hiện đại
npm install -g pnpm
npm install -g yarn
```

---

## 🚀 4. Khởi Tạo & Đóng Gói Ứng Dụng NestJS Mẫu

Chúng ta sẽ tạo một ứng dụng NestJS hoàn chỉnh để thực hành triển khai thực tế trên server.

### Bước 4.1: Cài đặt NestJS CLI và tạo dự án mới

```bash
# 1. Cài đặt NestJS CLI toàn cục
sudo npm install -g @nestjs/cli

# 2. Đi vào thư mục apps của user ubuntu
mkdir -p ~/apps && cd ~/apps

# 3. Tạo một dự án NestJS mới mang tên "nestjs-api" (chọn package manager là npm)
nest new nestjs-api --skip-git --package-manager npm
```

---

### Bước 4.2: Tạo một Endpoint API kiểm tra sức khỏe hệ thống (Health Check)

Chỉnh sửa file `src/app.service.ts`:
```bash
nano ~/apps/nestjs-api/src/app.service.ts
```

Thay đổi nội dung để API trả về thông tin chi tiết của hệ thống:
```typescript
import { Injectable } from '@nestjs/common';
import * as os from 'os';

@Injectable()
export class AppService {
  getSystemStatus() {
    return {
      status: 'online',
      service: 'NestJS Production API',
      node_version: process.version,
      platform: process.platform,
      arch: process.arch,
      server_uptime: Math.floor(process.uptime()) + ' seconds',
      free_memory_mb: Math.floor(os.freemem() / 1024 / 1024) + ' MB',
      timestamp: new Date().toISOString(),
    };
  }
}
```

Chỉnh sửa file `src/app.controller.ts`:
```bash
nano ~/apps/nestjs-api/src/app.controller.ts
```

```typescript
import { Controller, Get } from '@nestjs/common';
import { AppService } from './app.service';

@Controller()
export class AppController {
  constructor(private readonly appService: AppService) {}

  @Get()
  getHello() {
    return this.appService.getSystemStatus();
  }

  @Get('api/health')
  getHealth() {
    return { status: 'healthy', database: 'connected' };
  }
}
```

---

## 📦 5. Quy Trình Build & Chạy Production Chuẩn Mực

Đây là quy trình bắt buộc phải nắm vững khi deploy bất kỳ ứng dụng NestJS nào lên server:

```text
Thư mục dự án (~/apps/nestjs-api)
   │
   ├── 1. Cài đặt dependencies: npm ci
   │
   ├── 2. Biên dịch mã nguồn: npm run build ──► Sinh ra thư mục dist/
   │
   ├── 3. Cấu hình biến môi trường: .env
   │
   └── 4. Chạy Production: node dist/main.js
```

---

### Bước 5.1: Cài đặt Dependencies với `npm ci`

> [!IMPORTANT]
> **Tại sao trên Server dùng `npm ci` thay vì `npm install`?**
> * **`npm install`:** Có thể tự ý cập nhật các package lên bản mới hơn nếu file `package.json` dùng dấu `^` hoặc `~`, dễ gây lỗi không tương thích code.
> * **`npm ci` (Clean Install):** Đọc chính xác 100% file `package-lock.json`, xóa sạch `node_modules` cũ và cài đặt đúng từng phiên bản một cách hoàn hảo, tốc độ nhanh hơn gấp 2 lần.

```bash
cd ~/apps/nestjs-api
npm ci
```

---

### Bước 5.2: Biên dịch ứng dụng sang JavaScript thuần (`npm run build`)

```bash
npm run build
```

Kiểm tra thư mục `dist/` vừa được sinh ra:
```bash
ls -la dist/
```
*Bạn sẽ thấy các file JavaScript đã được biên dịch:* `main.js`, `app.module.js`, `app.service.js`...

---

### Bước 5.3: Cấu hình Biến Môi Trường (`.env`)

Tạo file biến môi trường cho môi trường Production:
```bash
nano ~/apps/nestjs-api/.env
```

Dán nội dung cấu hình sau:
```env
NODE_ENV=production
PORT=3000
APP_NAME=NestJS-Production-Server
API_SECRET_KEY=MyUltraSecureSecretKey2026!
```

*Phân quyền an toàn chỉ cho chủ sở hữu đọc file `.env`:*
```bash
chmod 600 ~/apps/nestjs-api/.env
```

---

### Bước 5.4: Chạy thử nghiệm Production Build

```bash
node dist/main.js
```

*Quan sát terminal xuất hiện log của NestJS:*
```text
[Nest] 15204  - 08/26/2026, 8:15:00 AM     LOG [NestFactory] Starting Nest application...
[Nest] 15204  - 08/26/2026, 8:15:01 AM     LOG [InstanceLoader] AppModule dependencies initialized
[Nest] 15204  - 08/26/2026, 8:15:01 AM     LOG [RoutesResolver] AppController {/}:
[Nest] 15204  - 08/26/2026, 8:15:01 AM     LOG [RouterExplorer] Mapped {/, GET} route
[Nest] 15204  - 08/26/2026, 8:15:01 AM     LOG [NestApplication] Nest application successfully started
```

Mở một tab Terminal khác trên máy Mac hoặc trên Server để gửi request kiểm tra:
```bash
curl http://127.0.0.1:3000/
```

*Kết quả trả về JSON cực kỳ ấn tượng:*
```json
{
  "status": "online",
  "service": "NestJS Production API",
  "node_version": "v20.x.x",
  "platform": "linux",
  "arch": "arm64",
  "server_uptime": "15 seconds",
  "free_memory_mb": "1420 MB",
  "timestamp": "2026-08-26T08:15:15.000Z"
}
```

Bấm `Ctrl + C` tại cửa sổ đang chạy NestJS để dừng ứng dụng.

---

## 🔌 6. Tích Hợp Hoàn Chỉnh: Nginx Reverse Proxy $\rightarrow$ NestJS

Bây giờ chúng ta sẽ kết nối Nginx đã học ở **Phần 08** để làm Reverse Proxy đứng trước ứng dụng NestJS vừa tạo.

```text
Browser trên Mac ──(HTTP/HTTPS)──► Nginx (Port 80/443) ──(proxy_pass)──► NestJS (127.0.0.1:3000)
```

### Bước 6.1: Cập nhật cấu hình Nginx Server Block

Mở file cấu hình Nginx:
```bash
sudo nano /etc/nginx/sites-available/nestjs-api.conf
```

Dán cấu hình chuyển tiếp:
```nginx
server {
    listen 80;
    server_name _;

    location / {
        proxy_pass http://127.0.0.1:3000;

        # Bộ 4 Headers chuẩn mực
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # Hỗ trợ WebSocket nếu NestJS dùng Gateway/Socket.io
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";

        proxy_connect_timeout 60s;
        proxy_read_timeout 60s;
    }
}
```

### Bước 6.2: Kích hoạt website và Reload Nginx

```bash
# 1. Kích hoạt Server Block mới
sudo ln -sf /etc/nginx/sites-available/nestjs-api.conf /etc/nginx/sites-enabled/

# 2. Xóa các site cũ không dùng
sudo rm -f /etc/nginx/sites-enabled/default /etc/nginx/sites-enabled/static-site.conf

# 3. Kiểm tra cú pháp
sudo nginx -t

# 4. Nạp lại cấu hình
sudo systemctl reload nginx
```

### Bước 6.3: Chạy NestJS ở chế độ ngầm tạm thời và kiểm tra từ máy Mac
```bash
# Chạy NestJS ở chế độ background (&)
cd ~/apps/nestjs-api && node dist/main.js &

# Từ máy Mac, mở trình duyệt truy cập thẳng vào IP của VM:
# http://192.168.64.2/
```
$\rightarrow$ Giao diện JSON của NestJS API xuất hiện ngay trên trình duyệt máy Mac thông qua Nginx!

---

## ❓ 7. Vấn Đề Lớn: Tại Sao Cần Systemd / PM2? (Giới Thiệu Phần 10)

Hãy thử làm một thí nghiệm:
1. Đăng xuất khỏi SSH (`exit`).
2. Mở trình duyệt máy Mac và tải lại trang: `http://192.168.64.2/`.
3. **Kết quả:** Trình duyệt lập tức báo lỗi **`502 Bad Gateway`**!

```text
┌─────────────────────────────────────────────────────────────────────────┐
│              TẠI SAO ỨNG DỤNG NODE.JS BỊ CHẾT KHI THOÁT SSH?             │
└────────────────────────────────────┬────────────────────────────────────┘
                                     │
    Bạn đăng nhập SSH ──────────────► Khởi tạo phiên Terminal Shell
                                     │
    Chạy `node dist/main.js` ────────► Tiến trình Node là TIẾN TRÌNH CON (Child Process)
                                       của phiên Shell đó
                                     │
    Bạn gõ `exit` / Đóng cửa sổ ────► Shell gửi tín hiệu `SIGHUP` (Hangup Signal)
                                       tiêu diệt toàn bộ tiến trình con!
```

> [!WARNING]
> **Hạn chế của việc chạy bằng lệnh thường hoặc `nohup` / `&`:**
> 1. Không tự khởi động lại khi Server bị Restart/Reboot.
> 2. Nếu code bị lỗi Unhandled Exception làm sập app (Crash), tiến trình sẽ chết vĩnh viễn và không tự hồi sinh.
> 3. Không có cơ chế quản lý tài nguyên và xoay vòng log hệ thống.
> 
> $\rightarrow$ **Giải pháp chuẩn doanh nghiệp:** Chuyển giao toàn bộ việc quản lý tiến trình cho **[Systemd Service]** (sẽ học chi tiết ở **Phần 10**).

---

## 🚨 8. Xử Lý 4 Lỗi Kinh Điển Khi Triển Khai Node.js / NestJS

### 🔴 Lỗi 1: `Error: listen EADDRINUSE: address already in use :::3000`
* **Nguyên nhân:** Cổng 3000 đã bị chiếm giữ bởi một tiến trình Node.js cũ chưa được tắt.
* **Cách xử lý:**
  ```bash
  # 1. Tìm PID của tiến trình đang chiếm cổng 3000
  sudo ss -tulpn | grep :3000
  # (Hoặc: sudo lsof -i :3000)

  # 2. Tiêu diệt tiến trình đó bằng PID
  sudo kill -9 <PID>
  ```

---

### 🔴 Lỗi 2: `Error: Cannot find module '/home/ubuntu/apps/.../dist/main.js'`
* **Nguyên nhân:** Bạn chưa chạy lệnh biên dịch `npm run build` hoặc đang đứng sai thư mục làm việc khi chạy `node`.
* **Cách xử lý:**
  ```bash
  cd ~/apps/nestjs-api
  npm run build
  ls -la dist/main.js
  ```

---

### 🔴 Lỗi 3: `JavaScript heap out of memory` khi chạy `npm run build` trên VM ít RAM
* **Nguyên nhân:** Trình biên dịch TypeScript `tsc` ngốn rất nhiều RAM khi phân tích cây cú pháp (AST). Nếu máy ảo chỉ có 1GB hoặc 2GB RAM, tiến trình sẽ bị Linux OOM Killer tiêu diệt.
* **Cách xử lý:** Tăng giới hạn bộ nhớ Heap cho Node.js khi build:
  ```bash
  NODE_OPTIONS="--max-old-space-size=1536" npm run build
  ```

---

### 🔴 Lỗi 4: Không đọc được biến môi trường trong file `.env`
* **Nguyên nhân:** Chưa cài đặt package nạp môi trường `@nestjs/config` (hoặc `dotenv`) trong NestJS.
* **Cách xử lý:**
  ```bash
  npm install @nestjs/config
  ```
  Thêm `ConfigModule.forRoot({ isGlobal: true })` vào mảng `imports` trong `src/app.module.ts`.

---

## 🧪 9. Bài Thực Hành Lab (10 Bước Triển Khai NestJS)

*Hãy thực hiện toàn bộ quy trình thiết lập môi trường Backend trên Ubuntu Server VM:*

1. Cài đặt Node.js 20 LTS từ NodeSource Repository.
2. Kiểm tra phiên bản `node -v` và `npm -v`.
3. Cài đặt `@nestjs/cli` toàn cục.
4. Tạo dự án mới `~/apps/nestjs-api`.
5. Tạo endpoint `/api/health` trả về JSON trạng thái hệ thống.
6. Chạy `npm ci` và biên dịch dự án với `npm run build`.
7. Tạo file `.env` chứa `PORT=3000` và phân quyền `chmod 600 .env`.
8. Chạy thử nghiệm `node dist/main.js` và test nội bộ bằng `curl http://127.0.0.1:3000/api/health`.
9. Cấu hình Nginx Reverse Proxy chuyển tiếp cổng 80 vào `http://127.0.0.1:3000`.
10. Mở trình duyệt trên máy Mac, truy cập `http://<IP_VM>/api/health` và xác nhận kết quả thành công!

---

## 📌 10. Bảng Tra Cứu Lệnh Node.js / NestJS Cốt Lõi

| Lệnh | Ý nghĩa & Khi nào dùng |
| :--- | :--- |
| `node -v` / `npm -v` | Kiểm tra phiên bản Node.js và NPM |
| `npm ci` | Cài đặt chính xác dependencies theo `package-lock.json` (Dùng trên Server) |
| `npm ci --omit=dev` | Chỉ cài dependencies chạy production, bỏ qua devDependencies (Tiết kiệm đĩa) |
| `npm run build` | Biên dịch TypeScript `src/` sang JavaScript `dist/` |
| `node dist/main.js` | Lệnh khởi chạy ứng dụng Production chuẩn mực |
| `sudo ss -tulpn \| grep :3000` | Kiểm tra tiến trình đang chiếm giữ cổng 3000 |
| `pkill -f "node"` | Tắt nhanh toàn bộ các tiến trình Node đang chạy ngầm |
| `NODE_OPTIONS="--max-old-space-size=..."` | Cấp thêm RAM tối đa cho Node.js khi build dự án lớn |
