# 📘 CẨM NANG TRIỂN KHAI PRODUCTION: UBUNTU VM / DOCKER QUA CLOUDFLARE TUNNEL (FULL END-TO-END SSL)

> **Motto cốt lõi:**  
> *Zero Open Ports – Không mở cổng Router/Modem | Ẩn IP máy chủ 100% | Mã hóa HTTPS 2 chiều toàn vẹn (Full End-to-End SSL từ Client $\rightarrow$ Cloudflare $\rightarrow$ Nginx).*

---

## 🏛️ 1. Mô Hình Kiến Trúc & Luồng Dữ Liệu (Zero Trust Architecture)

```text
┌───────────────────────────────┐
│   Client (Trình duyệt/Mobile) │
└───────────────┬───────────────┘
                │ (1) HTTPS - Cổng 443 / Cloudflare Edge Certificate (Mã hóa công khai)
                ▼
┌───────────────────────────────┐
│    Cloudflare Edge Server     │  ◄── WAF, Anti-DDoS, Caching, CDN toàn cầu
└───────────────┬───────────────┘
                │ (2) Đường hầm mã hóa 2 chiều QUIC/TLS Outbound (Đường hầm Tunnel)
                ▼
┌─────────────────────────────────────────────────────────────────────────┐
│ UBUNTU SERVER / VIRTUAL MACHINE                                         │
│                                                                         │
│   [ cloudflared Service ] (Daemon chạy ngầm dạng systemd)               │
│         │                                                               │
│         │ (3) HTTPS - https://localhost:443 (SNI: nano-boxes.id.vn)     │
│         ▼                                                               │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │ Docker Container: Nginx (Cài Cloudflare Origin CA Certificate)  │   │
│   └───────────────┬─────────────────────────────────┬───────────────┘   │
│                   │                                 │                   │
│      (4a) Route: /│                     (4b) Route: │/graphql           │
│                   ▼                                 ▼                   │
│   ┌───────────────────────────────┐ ┌───────────────────────────────┐   │
│   │ Container: Frontend (Next.js) │ │ Container: Backend            │   │
│   │ Port: 3000                    │ │ (Express / NestJS) Port: 5050 │   │
│   └───────────────────────────────┘ └───────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────┘
```

### 🌟 Ưu Điểm Tuyệt Đối Của Kiến Trúc Này:
1. **Zero Open Ports:** Không cần cấu hình NAT, Port Forwarding (mở cổng 80/443) trên Modem Wi-Fi hay Router nhà mạng (kể cả mạng dính CGNAT).
2. **Ẩn IP Gốc Tuyệt Đối:** Địa chỉ IP Public của bạn (ví dụ `42.115.213.41`) được che giấu hoàn toàn khỏi Internet. Mọi kẻ tấn công chỉ có thể quét thấy dải IP Anycast của Cloudflare.
3. **Full End-to-End Encryption:** Dữ liệu được mã hóa 2 lớp:
   - Từ Client đến Cloudflare: Sử dụng Cloudflare Universal Edge SSL.
   - Từ Cloudflare đến Nginx: Sử dụng **Cloudflare Origin CA Certificate** hợp lệ, đảm bảo chuẩn mã hóa tối đa **Full (Strict)**.

---

## 🛠️ 2. Quy Trình Cấu Hình Chuẩn Từng Bước

---

### Bước 1: Tạo Cloudflare Origin CA Certificate

Chứng chỉ Origin CA được cấp phát bởi chính Cloudflare nhằm đảm bảo kết nối giữa Cloudflare Edge và máy chủ của bạn an toàn 100%.

1. Truy cập **Cloudflare Dashboard** $\rightarrow$ Chọn tên miền của bạn (`nano-boxes.id.vn`).
2. Điều hướng tới **SSL/TLS** $\rightarrow$ **Origin Server** $\rightarrow$ Bấm nút **Create Certificate**.
3. Cấu hình các thông số:
   - **Private key type:** Chọn **RSA (2048)** (hoặc ECC).
   - **Hostnames:** Điền tên miền chính và wildcard:
     ```text
     nano-boxes.id.vn
     *.nano-boxes.id.vn
     ```
   - **Certificate Validity:** Chọn 15 năm (mặc định).
4. Bấm **Create**. Cloudflare sẽ hiển thị 2 đoạn mã:
   - **Origin Certificate:** Sao chép và lưu vào file `ssl/cert.pem` trên server.
   - **Private Key:** Sao chép và lưu vào file `ssl/key.pem` trên server.

> [!CAUTION]
> **Bảo mật Private Key:**  
> Chuỗi Private Key chỉ hiển thị **DUY NHẤT MỘT LẦN**. Ngay sau khi lưu file, bạn phải phân quyền hạn chế truy cập:
> ```bash
> chmod 600 ssl/key.pem
> chmod 644 ssl/cert.pem
> ```

---

### Bước 2: Cấu Hình Nginx & Docker Compose

#### 1. File cấu hình `nginx.conf`:

> [!IMPORTANT]
> **Lưu ý sống còn về `proxy_pass` trong Nginx:**  
> Tuyệt đối **KHÔNG** thêm dấu gạch chéo `/` ở cuối directive `proxy_pass` của các route API/GraphQL (dùng `proxy_pass http://backend:5050;` thay vì `proxy_pass http://backend:5050/;`). Nếu thêm `/`, Nginx sẽ cắt bỏ tiền tố `/graphql`, khiến ứng dụng backend nhận request vào `/` và trả về lỗi **404 Not Found**.

```nginx
events {
    worker_connections 1024;
}

http {
    # 1. Chuyển hướng HTTP sang HTTPS nội bộ (nếu có truy cập qua port 80)
    server {
        listen 80;
        server_name nano-boxes.id.vn *.nano-boxes.id.vn;
        return 301 https://$host$request_uri;
    }

    # 2. Cấu hình HTTPS End-to-End với Cloudflare Origin Certificate
    server {
        listen 443 ssl;
        server_name nano-boxes.id.vn *.nano-boxes.id.vn;

        # Đường dẫn chứng chỉ mount từ host vào container
        ssl_certificate     /etc/nginx/ssl/cert.pem;
        ssl_certificate_key /etc/nginx/ssl/key.pem;

        # Chuẩn bảo mật TLS hiện đại
        ssl_protocols TLSv1.2 TLSv1.3;
        ssl_ciphers HIGH:!aNULL:!MD5;
        ssl_prefer_server_ciphers on;

        # -----------------------------------------------------------
        # ROUTE 1: API / GraphQL sang Backend
        # -----------------------------------------------------------
        location /graphql {
            proxy_pass http://backend:5050;   # BẮT BUỘC: KHÔNG có dấu / ở đuôi
            proxy_http_version 1.1;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection 'upgrade';
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
            proxy_cache_bypass $http_upgrade;
        }

        # -----------------------------------------------------------
        # ROUTE 2: Giao diện người dùng sang Frontend (Next.js)
        # -----------------------------------------------------------
        location / {
            proxy_pass http://frontend:3000;
            proxy_http_version 1.1;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection 'upgrade';
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
        }
    }
}
```

#### 2. File `docker-compose.yml` (Đoạn cấu hình Nginx):

Đảm bảo container map đầy đủ cả 2 cổng 80 và 443, đồng thời mount thư mục SSL ở chế độ Read-Only (`ro`):

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
      - frontend
      - backend

  frontend:
    # Container Next.js chạy cổng nội bộ 3000
    image: your-nextjs-image
    container_name: nano-frontend
    restart: unless-stopped
    expose:
      - "3000"

  backend:
    # Container NestJS/Express chạy cổng nội bộ 5050
    image: your-backend-image
    container_name: nano-backend
    restart: unless-stopped
    expose:
      - "5050"
```

Khởi chạy hệ thống container:
```bash
docker compose up -d
```

---

### Bước 3: Cài Đặt Cloudflare Tunnel Chạy Nền Tự Động (systemd)

> [!WARNING]
> **Tuyệt đối không chạy lệnh `cloudflared tunnel run` thủ công trên Terminal!**  
> Khi bạn tắt cửa sổ Terminal hoặc ngắt kết nối SSH, phiên làm việc sẽ gửi tín hiệu `SIGHUP` tiêu diệt tiến trình tunnel, khiến toàn bộ website bị sập ngay lập tức. **Bắt buộc phải cài đặt làm Systemd Service**.

Chạy chuỗi lệnh sau trên Ubuntu VM:

```bash
# 1. Cài đặt tunnel service với Token lấy từ Cloudflare Dashboard (Zero Trust > Networks > Tunnels)
sudo cloudflared service install <CHUỖI_TOKEN_DÀI_eyJhI...>

# 2. Nạp lại cấu hình systemd và khởi chạy dịch vụ
sudo systemctl daemon-reload
sudo systemctl enable --now cloudflared

# 3. Kiểm tra trạng thái hoạt động ngầm
sudo systemctl status cloudflared
```

*Yêu cầu kiểm tra:* Dòng `Active:` phải hiển thị màu xanh lá cây: **`active (running)`**.

---

### Bước 4: Cấu Hình Route & TLS Chuẩn Trên Cloudflare Dashboard

#### 1. Thiết lập Chế độ SSL Tổng thể:
- Truy cập: **SSL/TLS** $\rightarrow$ **Overview**:
  - Chọn chế độ **Full** hoặc **Full (strict)** *(Do Nginx đã có Origin CA Certificate chính chủ)*.
- Truy cập: **SSL/TLS** $\rightarrow$ **Edge Certificates**:
  - Bật tùy chọn **Always Use HTTPS** sang **ON**.

#### 2. Cấu hình Route cho Tunnel:
- Vào **Zero Trust Dashboard** $\rightarrow$ **Networks** $\rightarrow$ **Tunnels** $\rightarrow$ Chọn Tunnel của bạn $\rightarrow$ Tab **Public Hostnames** (hoặc Routes) $\rightarrow$ Bấm **Add a public hostname**:
  - **Subdomain:** Để trống (hoặc điền subdomain nếu muốn).
  - **Domain:** Chọn `nano-boxes.id.vn`.
  - **Path:** Để trống hoặc `*`.
  - **Type:** Chọn `HTTPS`.
  - **URL:** Điền `localhost:443` (hoặc `127.0.0.1:443`).

#### 3. Cấu hình TLS Nâng Cao (CỰC KỲ QUAN TRỌNG):
Bấm mở rộng mục **Additional application settings** $\rightarrow$ chọn thẻ **TLS**:
- **Origin Server Name:** Điền chính xác `nano-boxes.id.vn`.
  - *Ý nghĩa:* Do Tunnel kết nối vào `localhost:443`, tham số này gửi trường **SNI (Server Name Indication)** tương ứng với domain thật, giúp Nginx trả về đúng chứng chỉ Origin CA của `nano-boxes.id.vn` thay vì báo lỗi lệch tên.
- **Disable TLS certificate verification:** Giữ nguyên **OFF** (Không tắt xác thực chứng chỉ, giữ kết nối an toàn tối đa).

Bấm **Save changes**.

---

## 🚨 3. Sổ Tay Tra Cứu Lỗi & Khắc Phục Nhanh (Troubleshooting)

| Tên lỗi | Biểu hiện | Nguyên nhân gốc rễ | Cách xử lý dứt điểm |
| :--- | :--- | :--- | :--- |
| **`SEC_ERROR_UNKNOWN_ISSUER`** | Trình duyệt hiện màn hình đỏ/vàng cảnh báo chứng chỉ SSL không an toàn, không được tin cậy. | File `/etc/hosts` trên máy Mac còn lưu dòng trỏ domain về IP mạng LAN (`192.168.x.x`). Trình duyệt trên Mac bắt tay trực tiếp với Nginx nội bộ (nơi cài Origin CA – vốn chỉ Cloudflare tin tưởng). | 1. Xóa dòng `nano-boxes.id.vn` trong file `/etc/hosts` trên Mac.<br>2. Xóa cache DNS của Mac:<br>`sudo dscacheutil -flushcache; sudo killall -HUP mDNSResponder` |
| **`Error 1033 (Tunnel Error)`** | Cloudflare báo lỗi không tìm thấy máy chủ đích (Host unreachable). | Tiến trình `cloudflared` trên Ubuntu VM bị chết (do chạy thủ công trong terminal và bị tắt SSH). | Cài đặt `cloudflared` thành systemd service:<br>`sudo cloudflared service install <TOKEN>`<br>`sudo systemctl enable --now cloudflared` |
| **`The page isn't redirecting properly`** | Trình duyệt báo lỗi vòng lặp chuyển hướng vô tận (Redirect Loop). | Tunnel gửi request HTTP (port 80) vào Nginx. Nginx thấy port 80 liền bắn HTTP 301 chuyển sang HTTPS về Cloudflare, tạo thành vòng lặp 301 luẩn quẩn. | Chuyển **Service URL** trên Cloudflare Tunnel sang **`https://localhost:443`** (để Tunnel nói chuyện trực tiếp qua HTTPS với Nginx). |
| **`502 Bad Gateway`** | Màn hình Cloudflare hiển thị thông báo Host Error / Bad Gateway. | Lệch định danh chứng chỉ SSL (SNI Name Mismatch). Tunnel gọi `https://localhost:443` nhưng Nginx chỉ cấp chứng chỉ cho tên miền `nano-boxes.id.vn`. | Vào Tunnel Dashboard $\rightarrow$ **Additional application settings** $\rightarrow$ **TLS** $\rightarrow$ Điền trường **Origin Server Name** là `nano-boxes.id.vn`. |
| **`Response code 404 (API/GraphQL)`** | Giao diện Next.js tải bình thường nhưng đăng nhập, gửi form hoặc gọi GraphQL trả về lỗi `404 Not Found`. | Directive `proxy_pass` trong Nginx bị gõ thừa dấu gạch chéo `/` ở cuối (`proxy_pass http://backend:5050/;`), khiến Nginx xóa mất tiền tố `/graphql`. | Sửa file `nginx.conf`: đổi thành `proxy_pass http://backend:5050;` (bỏ dấu gạch chéo ở đuôi) rồi reload Nginx:<br>`docker restart nano-nginx` |

---

## 📌 4. Bảng Kiểm Tra Nghiệm Thu Cuối Cùng (Production Checklist)

Trước khi bàn giao hoặc đưa hệ thống vào phục vụ người dùng thực tế:

```bash
# 1. Kiểm tra service tunnel đang chạy nền ổn định
sudo systemctl is-active cloudflared
# Output mong đợi: active

# 2. Kiểm tra container Nginx, Frontend, Backend đều UP
docker compose ps
# Output mong đợi: nano-nginx (Up), nano-frontend (Up), nano-backend (Up)

# 3. Kiểm tra từ máy tính bên ngoài qua HTTPS
curl -I https://nano-boxes.id.vn/
# Output mong đợi: HTTP/2 200 (hoặc HTTP/1.1 200), Server: cloudflare

# 4. Kiểm tra endpoint GraphQL
curl -I https://nano-boxes.id.vn/graphql
# Output mong đợi: HTTP 200 hoặc 400 (do chưa gửi body), TUYỆT ĐỐI KHÔNG ĐƯỢC LÀ 404
```
