# 📘 CẤU HÌNH NGINX PRODUCTION TOÀN DIỆN: HTTPS, GRAPHQL & SOCKET.IO (WEBSOCKET)

> **Motto cốt lõi:**  
> *Hiệu năng đỉnh cao $\rightarrow$ Bảo mật nghiêm ngặt $\rightarrow$ Thời gian thực ổn định (Full-stack Reverse Proxy: Next.js Frontend + GraphQL API + Real-time Socket.IO).*

---

## 🏛️ 1. Sơ Đồ Kiến Trúc & Luồng Chuyển Tiếp Request

```text
                                CLIENT (Browser / Mobile App)
                                              │
                     ┌────────────────────────┴────────────────────────┐
                     │ (Port 80 HTTP)                  │ (Port 443 HTTPS)
                     ▼                                 ▼
         [ 301 Redirect to HTTPS ]           [ NGINX REVERSE PROXY ]
                                             ├── SSL Termination (TLS 1.2/1.3)
                                             ├── Security Headers & HSTS
                                             ├── Rate Limiting (15 req/s)
                                             └── Gzip Compression
                                                       │
         ┌────────────────────────┬────────────────────┴───────────────────┬────────────────────────┐
         │                        │                                        │                        │
         ▼                        ▼                                        ▼                        ▼
[ /_next/static/ ]          [ / (Frontend) ]                          [ /graphql ]            [ /socket.io/ ]
Cache tĩnh 365 ngày        Next.js SSR / Pages                       GraphQL API Queries     WebSocket Real-time
(Tắt log đĩa)              (Upstream keepalive 32)                   & Mutations             (Timeout 86400s)
         │                        │                                        │                        │
         └───────────┬────────────┘                                        └───────────┬────────────┘
                     ▼                                                                 ▼
         ┌────────────────────────┐                                        ┌────────────────────────┐
         │ CONTAINER: FRONTEND    │                                        │ CONTAINER: BACKEND     │
         │ (Next.js - Port 3000)  │                                        │ (NestJS - Port 5050)   │
         └────────────────────────┘                                        └────────────────────────┘
```

---

## 📄 2. Toàn Bộ File Cấu Hình Hoàn Chỉnh (`nginx.conf`)

```nginx
# Số tiến trình worker chạy song song (tự động theo số core CPU)
worker_processes auto;

events {
    # Số kết nối tối đa trên mỗi worker process (2048 x số core)
    worker_connections 2048;
    # Cho phép worker chấp nhận tất cả các kết nối mới cùng lúc
    multi_accept on;
}

http {
    # --------------------------------------------------------------------------
    # 1. CẤU HÌNH CƠ BẢN & MÃ HÓA MIME
    # --------------------------------------------------------------------------
    include       /etc/nginx/mime.types;
    default_type  application/octet-stream;

    # Ẩn phiên bản Nginx trên header phản hồi để bảo mật (chống dò lỗi phiên bản)
    server_tokens off;

    # Tối ưu hóa gửi file từ Kernel (Zero-Copy)
    sendfile        on;
    tcp_nopush      on; # Gom gói tin TCP gửi đi 1 lần, tiết kiệm băng thông
    tcp_nodelay     on; # Gửi dữ liệu ngay lập tức, cực kỳ quan trọng cho WebSocket

    # Thời gian duy trì kết nối Keep-Alive với Client
    keepalive_timeout  65;

    # Giới hạn kích thước body upload (mặc định của Nginx chỉ có 1MB)
    client_max_body_size 25M;

    # --------------------------------------------------------------------------
    # 2. NÉN DỮ LIỆU GZIP (TĂNG TỐC TẢI TRANG LÊN ĐẾN 70%)
    # --------------------------------------------------------------------------
    gzip on;
    gzip_vary on;
    gzip_proxied any;
    gzip_comp_level 6; # Cân bằng hoàn hảo giữa tỷ lệ nén và tải CPU
    gzip_types 
        text/plain 
        text/css 
        application/json 
        application/javascript 
        text/xml 
        application/xml 
        application/xml+rss 
        image/svg+xml;

    # --------------------------------------------------------------------------
    # 3. GIỚI HẠN TẦN SUẤT TRUY CẬP (RATE LIMITING - CHỐNG DDOS/SPAM)
    # --------------------------------------------------------------------------
    # Vùng nhớ 10MB theo dõi IP nhị phân (chứa được ~160.000 IP), giới hạn 15 req/s
    limit_req_zone $binary_remote_addr zone=api_limit:10m rate=15r/s;

    # --------------------------------------------------------------------------
    # 4. KHAI BÁO CỤM UPSTREAM (KẾT NỐI TỚI DOCKER CONTAINER)
    # --------------------------------------------------------------------------
    upstream frontend_upstream {
        server nano-frontend:3000;
        # Duy trì 32 socket nhàn rỗi để tái sử dụng, giảm độ trễ TCP Handshake
        keepalive 32;
    }

    upstream backend_upstream {
        server nano-backend:5050;
        # Duy trì 32 socket nhàn rỗi kết nối tới Backend
        keepalive 32;
    }

    # --------------------------------------------------------------------------
    # 5. CHUYỂN HƯỚNG HTTP (CỔNG 80) SANG HTTPS (CỔNG 443)
    # --------------------------------------------------------------------------
    server {
        listen 80;
        listen [::]:80;
        server_name nano-boxes.id.vn;

        # Chuyển hướng vĩnh viễn (301) sang giao thức an toàn HTTPS
        return 301 https://$host$request_uri;
    }

    # --------------------------------------------------------------------------
    # 6. MÁY CHỦ CHÍNH: HTTPS & HTTP/2
    # --------------------------------------------------------------------------
    server {
        listen 443 ssl http2;
        listen [::]:443 ssl http2;
        server_name nano-boxes.id.vn;

        # Đường dẫn cặp chứng chỉ SSL (Let's Encrypt hoặc Cloudflare Origin CA)
        ssl_certificate     /etc/nginx/ssl/fullchain.pem;
        ssl_certificate_key /etc/nginx/ssl/privkey.pem;

        # Chuẩn giao thức mã hóa an toàn (Loại bỏ hoàn toàn SSLv3, TLS 1.0, TLS 1.1)
        ssl_protocols TLSv1.2 TLSv1.3;
        ssl_ciphers HIGH:!aNULL:!MD5;
        ssl_prefer_server_ciphers on;

        # Bộ nhớ đệm SSL Session (10MB lưu được 40.000 session, giảm tải giải mã CPU)
        ssl_session_cache shared:SSL:10m;
        ssl_session_timeout 1d;
        ssl_session_tickets off;

        # BẢO MẬT TIÊU CHUẨN CAO (SECURITY HEADERS)
        add_header X-Frame-Options "SAMEORIGIN" always; # Chống Clickjacking
        add_header X-Content-Type-Options "nosniff" always; # Chống MIME-sniffing
        add_header X-XSS-Protection "1; mode=block" always; # Bộ lọc XSS trình duyệt
        add_header Referrer-Policy "strict-origin-when-cross-origin" always;
        # Ép trình duyệt luôn nhớ và dùng HTTPS trong vòng 1 năm (HSTS)
        add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;

        # BỘ HEADERS PROXY DÙNG CHUNG CHO MỌI ROUTE
        proxy_http_version 1.1; # Bắt buộc HTTP/1.1 để hỗ trợ Keepalive và WebSocket
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto https;

        # ----------------------------------------------------------------------
        # ROUTE 1: API GRAPHQL (/graphql)
        # ----------------------------------------------------------------------
        location /graphql {
            # Giới hạn 15 r/s, cho phép vượt ngưỡng đột biến tối đa 25 request (burst)
            limit_req zone=api_limit burst=25 nodelay;

            # TUYỆT ĐỐI KHÔNG có dấu gạch chéo / ở đuôi
            proxy_pass http://backend_upstream;

            proxy_connect_timeout 60s;
            proxy_read_timeout 60s;
            proxy_send_timeout 60s;
        }

        # ----------------------------------------------------------------------
        # ROUTE 2: SOCKET.IO / WEBSOCKET (/socket.io/)
        # ----------------------------------------------------------------------
        location /socket.io/ {
            proxy_pass http://backend_upstream;

            # 2 Header bắt buộc để nâng cấp kết nối HTTP sang TCP WebSocket
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection "upgrade";

            # Tăng thời gian Timeout lên 24 giờ (86400s)
            # Ngăn Nginx ngắt kết nối WebSocket khi phòng chat/game đang idle
            proxy_read_timeout 86400s;
            proxy_send_timeout 86400s;

            # Tắt bộ đệm để tin nhắn real-time được truyền đi tức thì
            proxy_buffering off;
        }

        # ----------------------------------------------------------------------
        # ROUTE 3: UPLOAD FILE DUNG LƯỢNG LỚN (/upload)
        # ----------------------------------------------------------------------
        location /upload {
            limit_req zone=api_limit burst=10 nodelay;
            proxy_pass http://backend_upstream;

            # Cho phép upload riêng cho route này lên đến 50MB (nếu cần)
            client_max_body_size 50M;
            proxy_read_timeout 120s;
        }

        # ----------------------------------------------------------------------
        # ROUTE 4: TỐI ƯU HÓA TÀI NGUYÊN TĨNH NEXT.JS (STATIC ASSETS)
        # ----------------------------------------------------------------------
        location /_next/static/ {
            proxy_pass http://frontend_upstream;

            # Cache trên trình duyệt 365 ngày (vì file Next.js có mã hash độc nhất)
            expires 365d;
            add_header Cache-Control "public, max-age=31536000, immutable";

            # Tắt ghi log truy cập file tĩnh để giảm tải I/O ổ cứng
            access_log off;
        }

        # ----------------------------------------------------------------------
        # ROUTE 5: GIAO DIỆN CHÍNH / TRANG NEXT.JS (ROOT ROUTE)
        # ----------------------------------------------------------------------
        location / {
            proxy_pass http://frontend_upstream;

            # Hỗ trợ cả Hot Module Replacement (HMR) hoặc real-time trên frontend
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection "upgrade";
        }
    }
}
```

---

## 🔍 3. Phân Tích Chuyên Sâu Từng Khối Cấu Hình

### 3.1. Tối Ưu Hóa Upstream & Kết Nối Nội Bộ (`keepalive 32`)
Mặc định Nginx mở một kết nối TCP mới cho **mỗi request** gửi tới backend rồi đóng ngay.  
* Khi có `upstream { server ...; keepalive 32; }` kết hợp với `proxy_http_version 1.1;`: Nginx giữ lại 32 kết nối TCP sẵn sàng tái sử dụng.
* **Lợi ích:** Giảm tải CPU tiêu hao do bắt tay TCP 3 bước (3-way handshake) giữa Nginx và Container, **giảm độ trễ trung bình của API xuống 20% - 35%**.

---

### 3.2. Cấu Hình GraphQL (`/graphql`) – Tránh "Bẫy Lỗi 404"
* **Cú pháp:** `proxy_pass http://backend_upstream;` (Không có dấu `/` ở cuối).
* **Tại sao?**
  - Nếu viết `proxy_pass http://backend_upstream/;` $\rightarrow$ Nginx sẽ cắt bỏ chuỗi `/graphql`, biến request `POST /graphql` thành `POST /`, khiến NestJS / Apollo Server trả về `404 Not Found`!
* **Rate Limiting:**
  - `limit_req zone=api_limit burst=25 nodelay;`: Chống các công cụ dò quét introspect schema hoặc spam query phức tạp làm cạn kiệt tài nguyên database.

---

### 3.3. Cấu Hình Socket.IO (`/socket.io/`) – Duy Trì Kết Nối Thời Gian Thực
Giao thức WebSocket bắt đầu bằng một request HTTP và sau đó được "thăng cấp" (Upgrade) thành kết nối TCP 2 chiều liên tục:
```text
Client ──(HTTP GET /socket.io/?EIO=4&transport=websocket)──► Nginx
          [Headers: Upgrade: websocket, Connection: Upgrade]
                               │
                               ▼ Nginx chuyển tiếp nguyên vẹn 2 header sang NestJS
                      [ NestJS Gateway chấp thuận ]
                               │
                               ▼
Client ◄═══════════════ (KẾT NỐI TCP DỰNG SẴN 2 CHIỀU) ═══════════════► NestJS
```

* **2 Directive then chốt:**
  ```nginx
  proxy_set_header Upgrade $http_upgrade;
  proxy_set_header Connection "upgrade";
  ```
* **Vấn đề ngắt kết nối sau 60 giây:**  
  Mặc định `proxy_read_timeout` của Nginx là `60s`. Nếu phòng chat không có tin nhắn nào trong 60 giây, Nginx sẽ tự động ngắt kết nối!  
  $\rightarrow$ Tham số `proxy_read_timeout 86400s;` (24 giờ) đảm bảo kết nối luôn sống bền bỉ.
* `proxy_buffering off;`: Tắt bộ đệm để tin nhắn được truyền đi ngay lập tức (Zero Latency).

---

### 3.4. Tối Ưu Next.js Static Cache (`/_next/static/`)
Next.js tạo ra các file JavaScript/CSS với mã băm nội dung độc nhất (ví dụ: `framework-d380c25a7.js`):
* Cấu hình `expires 365d;` và `immutable` ra lệnh cho trình duyệt lưu vĩnh viễn trong cache mà không cần gửi request xác thực lại lên máy chủ.
* `access_log off;`: Tránh việc mỗi lượt tải trang ghi hàng chục dòng log file tĩnh vào ổ cứng, kéo dài tuổi thọ SSD và tiết kiệm dung lượng đĩa.

---

## 🐳 4. Mẫu `docker-compose.yml` Chuẩn Đi Kèm

Để chạy mượt mà cấu hình trên, file Docker Compose của bạn cần cấu hình tên container tương ứng trong mạng bridge nội bộ:

```yaml
version: '3.8'

services:
  nginx:
    image: nginx:alpine
    container_name: nano-nginx
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
      - ./ssl:/etc/nginx/ssl:ro
    depends_on:
      - nano-frontend
      - nano-backend
    networks:
      - app_network

  nano-frontend:
    image: your-nextjs-app:latest
    container_name: nano-frontend
    restart: unless-stopped
    expose:
      - "3000"
    networks:
      - app_network

  nano-backend:
    image: your-nestjs-backend:latest
    container_name: nano-backend
    restart: unless-stopped
    expose:
      - "5050"
    networks:
      - app_network

networks:
  app_network:
    driver: bridge
```

---

## 🧪 5. Kiểm Tra & Nghiệm Thu Cấu Hình

### 1. Kiểm tra cú pháp Nginx trước khi nạp lại
```bash
docker exec -it nano-nginx nginx -t
```
*Output mong đợi:*
```text
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
```

### 2. Nạp lại cấu hình không gián đoạn dịch vụ (Zero-Downtime)
```bash
docker exec -it nano-nginx nginx -s reload
```

### 3. Kiểm tra kết nối WebSocket bằng `wscat` hoặc `curl`
```bash
# Kiểm tra bắt tay HTTP Upgrade cho Socket.IO
curl -i -N \
  -H "Connection: Upgrade" \
  -H "Upgrade: websocket" \
  -H "Host: nano-boxes.id.vn" \
  https://nano-boxes.id.vn/socket.io/?EIO=4&transport=websocket
```
*Output mong đợi:* Trả về mã HTTP **`101 Switching Protocols`**.

### 4. Kiểm tra Header chuyển hướng HTTPS & HSTS
```bash
curl -I http://nano-boxes.id.vn/
# Output: HTTP/1.1 301 Moved Permanently -> Location: https://nano-boxes.id.vn/

curl -I https://nano-boxes.id.vn/
# Output: Strict-Transport-Security: max-age=31536000; includeSubDomains
```
