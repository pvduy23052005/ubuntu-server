# 📘 Phần 08: Nginx – Web Server & Reverse Proxy Cho Ứng Dụng Backend

> **Motto cốt lõi:**  
> *Hiểu luồng đi của Request > Học thuộc cấu hình | Nginx làm người gác cổng phía trước – Node.js/NestJS xử lý logic phía sau.*  
> Chu trình chuẩn: **Nó là gì? → Request đi như thế nào? → Cấu hình → Thực hành → Quan sát → Debug.**

---

## 🎯 1. Mục Tiêu Của Phần Học

Trong kiến trúc triển khai ứng dụng hiện đại, **Nginx** là thành phần quan trọng bậc nhất đứng giữa Internet và ứng dụng Backend của bạn.

Sau khi hoàn thành bài học này, bạn sẽ:
1. **Hiểu rõ bản chất:** Phân biệt được **Web Server** (Nginx) và **Application Server** (Node.js / NestJS / Python).
2. **Làm chủ Reverse Proxy & SSL Termination:** Hiểu tại sao Nginx đứng trước giải mã HTTPS và chuyển tiếp traffic HTTP nội bộ về Node.js (cổng 3000).
3. **Quản lý cấu hình chuyên nghiệp:** Nắm vững cấu trúc `/etc/nginx/`, cơ chế tách biệt giữa `sites-available` (kho cấu hình) và `sites-enabled` (kích hoạt qua Symlink).
4. **Phục vụ Static Files & Cấu hình HTTPS:** Cấu hình Server Block phục vụ file tĩnh, tự động chuyển hướng HTTP $\rightarrow$ HTTPS 301 và gán chứng chỉ SSL.
5. **Thành thạo kỹ năng kiểm tra & Reload không gián đoạn:** Sử dụng `sudo nginx -t` và `sudo systemctl reload nginx` (Zero-downtime).
6. **Xử lý triệt để 2 lỗi kinh điển:** Giải quyết tận gốc lỗi **`403 Forbidden`** (Permission) và **`502 Bad Gateway`** (Upstream App Down) thông qua phân tích nhật ký `/var/log/nginx/`.

---

## 🌐 2. Bản Chất: Web Server, Reverse Proxy & Luồng Đi Của Request

### 2.1. Nginx là gì? Web Server vs Application Server

```text
┌─────────────────────────────────────────────────────────────────────────┐
│              PHÂN BIỆT NGINX VÀ APPLICATION SERVER (NODE.JS)             │
├────────────────────────────────────┬────────────────────────────────────┤
│  NGINX (Web Server / Reverse Proxy)│  NODE.JS / NESTJS (App Server)     │
│  - Viết bằng C, cực nhẹ, xử lý hàng│  - Chạy môi trường JavaScript/V8.  │
│    chục ngàn kết nối đồng thời.    │  - Chuyên xử lý Business Logic,    │
│  - Giỏi nhất: Phục vụ file tĩnh,   │    Database Query, Tính toán.      │
│    SSL/TLS Termination, Proxy.     │  - Không tối ưu cho việc xử lý     │
│  - Đóng vai trò "Mặt tiền / Lễ tân"│    file tĩnh hoặc bảo mật mạng thô.│
└────────────────────────────────────┴────────────────────────────────────┘
```

---

### 2.2. Reverse Proxy là gì?

* **Forward Proxy (Proxy xuôi):** Đứng cạnh **Client** (Người dùng) để thay mặt Client đi lấy dữ liệu từ Internet (ví dụ: VPN, Proxy công ty để vượt tường lửa).
* **Reverse Proxy (Proxy ngược):** Đứng trước **Server** (Hạ tầng Backend) để thay mặt Server tiếp nhận tất cả request từ Internet, sau đó phân phối nội bộ tới các ứng dụng bên trong.

---

### 2.3. Sơ đồ luồng đi của một Request (Request Lifecycle)

```text
Trình duyệt trên Mac / Client
              │
              │ 1. Gửi HTTP Request tới Port 80 (hoặc HTTPS 443)
              ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                           UBUNTU SERVER VM                              │
│                                                                         │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │                     NGINX (Lắng nghe Port 80)                   │   │
│   │                                                                 │   │
│   │   ├── Request: / (Trang chủ)  ──► Đọc file tĩnh: /var/www/html/ │   │
│   │   │                               (Trả về HTML/CSS trong 1ms)   │   │
│   │   │                                                             │   │
│   │   └── Request: /api/...       ──► proxy_pass                    │   │
│   │                                   (Chuyển tiếp nội bộ)          │   │
│   └───────────────────────────────────┬─────────────────────────────┘   │
│                                       │ 2. Chuyển tiếp tới 127.0.0.1:3000
│                                       ▼
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │                NODE.JS / NESTJS APPLICATION                     │   │
│   │             (Lắng nghe nội bộ tại localhost:3000)               │   │
│   │                                                                 │   │
│   │   └── Xử lý API Logic ──► Query Database ──► Trả về JSON Data  │   │
│   └───────────────────────────────────┬─────────────────────────────┘   │
│                                       │ 3. Trả kết quả JSON về Nginx
│                                       ▼
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │   Nginx nhận JSON ──► Gửi trả lại cho Trình duyệt Client        │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

> [!TIP]
> **Tại sao không mở cổng 3000 trực tiếp ra Internet?**
> 1. **Bảo mật:** Ẩn giấu hoàn toàn hạ tầng Backend phía sau. Hacker không biết bạn đang dùng Node.js, Python hay Go.
> 2. **Tối ưu hiệu năng:** Nginx phục vụ ảnh, video, file tĩnh nhanh hơn Node.js gấp nhiều lần và tự động nén dữ liệu (`gzip`).
> 3. **Tập trung SSL:** Nginx đảm nhận giải mã HTTPS/SSL, giảm tải tính toán mã hóa cho ứng dụng Node.js.

---

## 🛠️ 3. Cài Đặt & Cấu Trúc Thư Mục Của Nginx

### Bước 3.1: Cài đặt Nginx
Đăng nhập vào Ubuntu Server VM và chạy lệnh:

```bash
sudo apt update
sudo apt install -y nginx
```

### Bước 3.2: Kiểm tra trạng thái hoạt động
```bash
# 1. Kiểm tra dịch vụ systemd
sudo systemctl status nginx

# 2. Kiểm tra xem Nginx có đang chiếm cổng 80 không
sudo ss -tulpn | grep :80

# 3. Gửi request thử nghiệm từ nội bộ server
curl -I http://localhost
```
*Quan sát thấy:* `HTTP/1.1 200 OK` kèm `Server: nginx/...` $\rightarrow$ Nginx đã hoạt động hoàn hảo!

---

### 3.3. Giải mã cấu trúc thư mục `/etc/nginx/`

Hãy chạy lệnh `ls -la /etc/nginx/` để quan sát cấu trúc:

```text
/etc/nginx/
├── nginx.conf          ──► FILE CẤU HÌNH GỐC (Chứa thiết lập toàn cục: User, Worker Process, Gzip...)
├── conf.d/             ──► Thư mục chứa các file cấu hình phụ (*.conf)
├── sites-available/    ──► "KHO LƯU TRỮ" tất cả các cấu hình Server Block (Website) bạn đã viết
└── sites-enabled/      ──► "DANH SÁCH KÍCH HOẠT": Chỉ các website có Symlink ở đây mới thực sự CHẠY!
```

```text
┌─────────────────────────────────────────────────────────────────────────┐
│                   CƠ CHẾ SITES-AVAILABLE VÀ SITES-ENABLED               │
└────────────────────────────────────┬────────────────────────────────────┘
                                     │
    1. Viết cấu hình tại:            │
       /etc/nginx/sites-available/my-app.conf
                                     │
                                     │ 2. Tạo Symbolic Link (ln -s)
                                     ▼
    3. Kích hoạt website tại:
       /etc/nginx/sites-enabled/my-app.conf ──► (Trỏ về file gốc)
```

> [!NOTE]
> **Tại sao phải chia làm 2 thư mục?**  
> Giúp bạn dễ dàng "bật/tắt" một website: Khi cần bảo trì một trang web, bạn chỉ cần xóa symlink trong `sites-enabled/` mà **không hề làm mất file cấu hình gốc** trong `sites-available/`.

---

## 📄 4. Thực Hành 1: Phục Vụ Trang Web Tĩnh (Static Web Server)

Chúng ta sẽ tạo một Server Block phục vụ trang HTML tĩnh độc lập.

### Bước 4.1: Tạo thư mục chứa mã nguồn web
```bash
sudo mkdir -p /var/www/my-static-site

# Tạo 1 file HTML đơn giản
sudo tee /var/www/my-static-site/index.html << 'EOF'
<!DOCTYPE html>
<html>
<head>
    <title>Hello Ubuntu Nginx</title>
    <style>
        body { font-family: sans-serif; text-align: center; margin-top: 50px; background: #f4f6f8; }
        h1 { color: #0070f3; }
    </style>
</head>
<body>
    <h1>🚀 Nginx Static Web Server Đang Chạy!</h1>
    <p>Trang web tĩnh được phục vụ trực tiếp từ /var/www/my-static-site</p>
</body>
</html>
EOF
```

### Bước 4.2: Phân quyền cho User `www-data`
Nginx chạy dưới tài khoản `www-data`, do đó cần cấp quyền đọc cho user này:
```bash
sudo chown -R www-data:www-data /var/www/my-static-site
sudo chmod -R 755 /var/www/my-static-site
```

---

### Bước 4.3: Viết cấu hình Server Block
Tạo file cấu hình mới tại `sites-available`:

```bash
sudo nano /etc/nginx/sites-available/static-site.conf
```

Dán nội dung cấu hình sau:
```nginx
server {
    listen 80;
    server_name _; # Khớp với mọi Domain/IP truy cập

    root /var/www/my-static-site;
    index index.html;

    location / {
        try_files $uri $uri/ =404;
    }
}
```

*Lưu file:* Bấm `Ctrl + O` $\rightarrow$ `Enter`, sau đó `Ctrl + X` để thoát.

---

### Bước 4.4: Kích hoạt website & Kiểm tra an toàn

```bash
# 1. Tắt cấu hình mặc định (Default) của Nginx để tránh xung đột cổng 80
sudo rm -f /etc/nginx/sites-enabled/default

# 2. Tạo Symbolic Link để kích hoạt website mới
sudo ln -s /etc/nginx/sites-available/static-site.conf /etc/nginx/sites-enabled/

# 3. KIỂM TRA CÚ PHÁP CẤU HÌNH (Bắt buộc trước khi reload!)
sudo nginx -t
```
*Khi thấy thông báo:* `nginx: configuration file /etc/nginx/nginx.conf test is successful`.

```bash
# 4. Nạp lại cấu hình (Zero-downtime Reload)
sudo systemctl reload nginx
```

### Bước 4.5: Kiểm tra kết quả
Mở trình duyệt trên máy Mac và truy cập vào địa chỉ IP của VM: `http://192.168.64.2` *(thay bằng IP thực tế của bạn)*.  
$\rightarrow$ Giao diện trang HTML màu xanh sẽ xuất hiện ngay lập tức!

---

## 🔀 5. Thực Hành 2: Cấu Hình Nginx Làm Reverse Proxy Cho Node.js

Bây giờ chúng ta sẽ mô phỏng mô hình Production thực tế: Một ứng dụng Backend Node.js chạy ngầm ở cổng `3000`, và Nginx đứng trước làm Reverse Proxy chuyển tiếp traffic.

```text
Browser (Mac) ──(HTTP:80)──► Nginx ──(proxy_pass:3000)──► Node.js App (127.0.0.1:3000)
```

---

### Bước 5.1: Tạo một ứng dụng Node.js mẫu chạy ở Port 3000

Tạo nhanh một server Node.js HTTP đơn giản:

```bash
# 1. Cài đặt Node.js nhanh (nếu chưa có)
sudo apt install -y nodejs

# 2. Tạo thư mục app
mkdir -p ~/demo-app && cd ~/demo-app

# 3. Viết file server.js
cat << 'EOF' > server.js
const http = require('http');
const PORT = 3000;

const server = http.createServer((req, res) => {
    res.writeHead(200, { 'Content-Type': 'application/json' });
    res.end(JSON.stringify({
        status: 'success',
        message: 'Xin chao tu Backend Node.js!',
        client_ip: req.headers['x-real-ip'] || req.socket.remoteAddress,
        requested_url: req.url,
        timestamp: new Date().toISOString()
    }));
});

server.listen(PORT, '127.0.0.1', () => {
    console.log(`Backend Server dang lang nghe tai http://127.0.0.1:${PORT}`);
});
EOF

# 4. Chạy ứng dụng Node.js ở chế độ chạy ngầm (Background)
node server.js &
```

Kiểm tra xem Node.js đã lắng nghe tại cổng 3000 chưa:
```bash
curl http://127.0.0.1:3000
```
*Kết quả:* Trả về chuỗi JSON chứa `status: success`.

---

### Bước 5.2: Cấu hình Nginx Reverse Proxy (`proxy_pass`)

Sửa file cấu hình `/etc/nginx/sites-available/static-site.conf`:

```bash
sudo nano /etc/nginx/sites-available/static-site.conf
```

Thay đổi nội dung thành cấu hình Reverse Proxy chuyên nghiệp:

```nginx
server {
    listen 80;
    server_name _;

    # Chuyển tiếp toàn bộ request tới ứng dụng Node.js
    location / {
        proxy_pass http://127.0.0.1:3000;

        # Bộ 4 Headers chuẩn mực bắt buộc phải có:
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # Thiết lập thời gian timeout kết nối
        proxy_connect_timeout 60s;
        proxy_read_timeout 60s;
    }
}
```

```text
┌───────────────────────────┬─────────────────────────────────────────────────┐
│ Directive                 │ Ý nghĩa & Bản chất                              │
├───────────────────────────┼─────────────────────────────────────────────────┤
│ proxy_pass                │ Địa chỉ và cổng nội bộ của ứng dụng Backend.    │
│ proxy_set_header Host     │ Giữ nguyên tên miền gốc mà Client gõ.           │
│ proxy_set_header X-Real-IP│ Chuyển địa chỉ IP THẬT của Client cho Node.js   │
│                           │ (nếu không có, Node.js sẽ tưởng IP là 127.0.0.1)│
└───────────────────────────┴─────────────────────────────────────────────────┘
```

---

### Bước 5.3: Kiểm tra và Reload Nginx
```bash
sudo nginx -t
sudo systemctl reload nginx
```

### Bước 5.4: Kiểm chứng thực tế từ máy Mac
Từ trình duyệt trên máy Mac, truy cập: `http://192.168.64.2` (Port 80 mặc định).  
$\rightarrow$ Bạn sẽ nhận được JSON từ Node.js với `client_ip` hiển thị chính xác IP máy Mac của bạn!

---

## 🔐 6. Cấu Hình HTTPS / SSL Cho Reverse Proxy (SSL Termination)

Trong mô hình thực tế, **Node.js KHÔNG CẦN tự cấu hình HTTPS**. Nginx sẽ đứng phía trước chịu trách nhiệm giải mã toàn bộ HTTPS (gọi là **SSL Termination**), sau đó chuyển tiếp nội bộ qua HTTP thông thường về cho Node.js (cổng `3000`).

```text
Trình duyệt trên Mac
       │
       ├── (HTTP:80) ──► Nginx (Tự động chuyển hướng Redirect 301 sang HTTPS)
       │
       └── (HTTPS:443 Mã hóa SSL) ──► Nginx (Giải mã SSL / SSL Termination)
                                        │
                                        │ (proxy_pass qua HTTP nội bộ cực nhanh)
                                        ▼
                             Node.js App (127.0.0.1:3000)
```

Tùy thuộc vào môi trường của bạn, hãy chọn phương án phù hợp:
* **Phương án A (Môi trường Lab nội bộ):** Dùng OpenSSL tạo chứng chỉ tự ký (Self-Signed) khi chỉ có IP máy ảo (ví dụ `192.168.64.2`).
* **Phương án B (Môi trường Production thực tế):** Dùng **Certbot (Let's Encrypt)** tự động cấp và cấu hình SSL miễn phí 100% khi có Tên miền (Domain).

---

### 🅰️ Phương Án A: Dùng OpenSSL Self-Signed (Dành Cho Môi Trường Lab / IP VM)

#### Bước A.1: Tạo cặp chứng chỉ SSL tự ký
```bash
# 1. Tạo thư mục chứa chứng chỉ
sudo mkdir -p /etc/nginx/ssl

# 2. Sinh Private Key và Certificate có hạn 365 ngày
sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout /etc/nginx/ssl/server.key \
  -out /etc/nginx/ssl/server.crt \
  -subj "/C=VN/ST=Hanoi/L=Hanoi/O=DevOpsLab/CN=192.168.64.2"

# 3. Phân quyền bảo mật cho file khóa
sudo chmod 600 /etc/nginx/ssl/server.key
sudo chmod 644 /etc/nginx/ssl/server.crt
```

#### Bước A.2: Cấu hình Server Block thủ công
Mở file `/etc/nginx/sites-available/static-site.conf`:
```nginx
# 1. Chuyển hướng toàn bộ HTTP (Port 80) sang HTTPS
server {
    listen 80;
    server_name _;
    return 301 https://$host$request_uri;
}

# 2. Xử lý HTTPS (Port 443) & Reverse Proxy về Node.js (Port 3000)
server {
    listen 443 ssl;
    server_name _;

    ssl_certificate /etc/nginx/ssl/server.crt;
    ssl_certificate_key /etc/nginx/ssl/server.key;

    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;
    ssl_prefer_server_ciphers on;

    location / {
        proxy_pass http://127.0.0.1:3000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto https;
        proxy_connect_timeout 60s;
        proxy_read_timeout 60s;
    }
}
```

---

### 🅱️ Phương Án B: Dùng Certbot (Let's Encrypt) Chuẩn Production (Khi Có Domain)

> [!IMPORTANT]
> **Điều kiện tiên quyết để dùng Certbot:**
> 1. Bạn sở hữu một tên miền thật (ví dụ: `api.yourdomain.com`).
> 2. Đã trỏ bản ghi **A Record** của tên miền về địa chỉ IP Public của máy chủ.
> 3. Tường lửa UFW đã mở cả 2 cổng: `sudo ufw allow 80/tcp` và `sudo ufw allow 443/tcp`.

#### Bước B.1: Cài đặt Certbot qua Snap
```bash
# 1. Cài đặt snapd và certbot
sudo apt update
sudo apt install -y snapd
sudo snap install --classic certbot

# 2. Tạo symlink để sử dụng lệnh certbot toàn cục
sudo ln -sf /snap/bin/certbot /usr/bin/certbot
```

#### Bước B.2: Chuẩn bị Server Block cơ bản với Domain của bạn
Trước khi chạy Certbot, đảm bảo file cấu hình Nginx (ví dụ `/etc/nginx/sites-available/my-app.conf`) đã khai báo đúng tên miền trong `server_name`:

```nginx
server {
    listen 80;
    server_name api.yourdomain.com;

    location / {
        proxy_pass http://127.0.0.1:3000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```
*Kích hoạt và nạp lại Nginx:*
```bash
sudo ln -sf /etc/nginx/sites-available/my-app.conf /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl reload nginx
```

#### Bước B.3: Chạy Certbot tự động cấp SSL & Tự cấu hình Nginx
Chỉ cần chạy 1 câu lệnh duy nhất:

```bash
sudo certbot --nginx -d api.yourdomain.com
```

*Quá trình tự động của Certbot:*
1. Certbot kết nối tới Let's Encrypt CA để xác thực quyền sở hữu tên miền qua HTTP-01 Challenge.
2. Certbot tự động tải chứng chỉ SSL về thư mục `/etc/letsencrypt/live/api.yourdomain.com/`.
3. Certbot **tự động sửa file cấu hình Nginx**, kích hoạt cổng 443 SSL và thêm luật chuyển hướng 301 từ HTTP sang HTTPS!
4. Certbot tự động nạp lại (Reload) Nginx.

#### Bước B.4: Kiểm tra tính năng tự động gia hạn (Auto-renewal)
Chứng chỉ Let's Encrypt có hạn 90 ngày. Certbot đã tự động cài đặt Systemd timer để tự gia hạn khi còn dưới 30 ngày. Kiểm tra giả lập:
```bash
sudo certbot renew --dry-run
```

---

### Bước 6.3: Mở cổng 443 trên Firewall và Reload Nginx

```bash
# 1. Mở cổng 443 trên UFW
sudo ufw allow 443/tcp

# 2. Kiểm tra cú pháp Nginx
sudo nginx -t

# 3. Reload Nginx
sudo systemctl reload nginx
```

---

### Bước 6.4: Kiểm chứng kết quả từ máy Mac
* **Trên trình duyệt:** Gõ `http://192.168.64.2` $\rightarrow$ Trình duyệt tự động nhảy sang `https://192.168.64.2`.
* **Trên Terminal máy Mac:**
  ```bash
  curl -k -I https://192.168.64.2
  ```
  *(Cờ `-k` để bỏ qua cảnh báo chứng chỉ tự ký; kết quả trả về `HTTP/1.1 200 OK`).*

---

## 🔒 7. Quyền Hạn User `www-data` & Xử Lý Lỗi `403 Forbidden`

### 7.1. Tại sao Nginx chạy dưới user `www-data`?
* **Nguyên tắc đặc quyền tối thiểu (Least Privilege):** Nếu Nginx chạy dưới quyền `root`, khi một hacker tìm ra lỗ hổng bảo mật trong Nginx, họ sẽ lập tức kiểm soát toàn bộ máy chủ.
* Bằng cách gán Nginx chạy dưới user không có quyền quản trị `www-data`, hacker sẽ bị "nhốt" trong phạm vi đọc các file web tĩnh.

---

### 7.2. Xử lý lỗi `403 Forbidden` (Bị cấm truy cập)

* **Triệu chứng:** Mở web thấy báo `403 Forbidden`.
* **Nguyên nhân phổ biến:**
  1. Thư mục web không có quyền đọc (`r`) hoặc không có quyền truy cập (`x`) cho user `www-data`.
  2. Thư mục không có file `index.html` và Nginx tắt tính năng duyệt file (`autoindex off`).
* **Cách sửa chuẩn xác:**
  ```bash
  # 1. Đổi chủ sở hữu cho www-data
  sudo chown -R www-data:www-data /var/www/my-site

  # 2. Phân quyền chuẩn: Thư mục 755, File 644
  sudo find /var/www/my-site -type d -exec chmod 755 {} \;
  sudo find /var/www/my-site -type f -exec chmod 644 {} \;
  ```

---

## 🚨 8. Phân Tích Lỗi `502 Bad Gateway` & Quy Trình Debug Chuyên Nghiệp

```text
┌─────────────────────────────────────────────────────────────────────────┐
│                      BẢN CHẤT LỖI 502 BAD GATEWAY                       │
└────────────────────────────────────┬────────────────────────────────────┘
                                     │
    Client ──(Gửi Request)──► Nginx (Hoạt động tốt ở Port 80/443)
                                     │
                                     │ (Nginx cố gắng kết nối tới 127.0.0.1:3000)
                                     ▼
                        [ ỨNG DỤNG NODE.JS BỊ SẬP / CHƯA BẬT! ]
                                     │
                                     ▼
    Nginx không nhận được phản hồi ──► Báo lỗi: 502 BAD GATEWAY cho Client
```

### 8.1. Thực hành tự tạo lỗi `502` và quan sát

**Bước 1: Tắt tiến trình Node.js đang chạy ngầm:**
```bash
# Tìm PID của node và tắt
pkill -f "node server.js"
```

**Bước 2: Mở trình duyệt máy Mac truy cập lại:** `https://192.168.64.2`  
$\rightarrow$ Trình duyệt hiển thị ngay lập tức: **`502 Bad Gateway` (Nginx/...)**.

---

### 8.2. Bộ 5 câu lệnh Debug Nginx kinh điển:

Khi gặp lỗi web, hãy thực hiện theo đúng 5 bước sau:

```bash
# 1. Kiểm tra cú pháp file cấu hình
sudo nginx -t

# 2. Kiểm tra trạng thái của tiến trình Nginx
sudo systemctl status nginx

# 3. Soi log lỗi hệ thống của Nginx
sudo journalctl -u nginx -n 20 --no-pager

# 4. Xem nhật ký lỗi chi tiết của Web (Nơi ghi rõ nguyên nhân 502/403)
sudo tail -n 20 /var/log/nginx/error.log
```

*Đọc dòng log lỗi 502 thực tế:*
```text
[error] 1420#1420: *1 connect() failed (111: Connection refused) while connecting to upstream, client: 192.168.64.1, server: _, request: "GET / HTTP/1.1", upstream: "http://127.0.0.1:3000/"
```
$\rightarrow$ `Connection refused while connecting to upstream http://127.0.0.1:3000/` nghĩa là: **Ứng dụng Backend ở cổng 3000 chưa được bật!**

```bash
# 5. Theo dõi Live Access Log (Mỗi khi có ai click vào web sẽ hiện lên)
sudo tail -f /var/log/nginx/access.log
```

**Bước 3: Bật lại Node.js để sửa lỗi:**
```bash
node ~/demo-app/server.js &
```
$\rightarrow$ Tải lại trang web, lỗi 502 biến mất hoàn toàn!

---

## 🧪 9. Bài Thực Hành Lab (11 Bước Triển Khai Hoàn Chỉnh)

*Hãy thực hiện kịch bản xây dựng Web Server, Reverse Proxy và HTTPS hoàn chỉnh trên Ubuntu Server VM:*

1. Cài đặt Nginx và xác nhận trạng thái `active (running)`.
2. Tạo thư mục `/var/www/lab-site` chứa file `index.html`.
3. Phân quyền sở hữu `www-data:www-data` và quyền `755` cho thư mục.
4. Tạo Server Block phục vụ static file tại `/etc/nginx/sites-available/lab.conf`.
5. Kích hoạt website bằng `ln -s` vào `sites-enabled/` và xóa site `default`.
6. Khởi chạy ứng dụng Node.js chạy ở cổng `3000`.
7. Chuyển đổi cấu hình `lab.conf` sang làm **Reverse Proxy** trỏ về `http://127.0.0.1:3000`.
8. Sinh chứng chỉ SSL tự ký bằng `openssl` và lưu vào `/etc/nginx/ssl/`.
9. Cấu hình chuyển hướng HTTP $\rightarrow$ HTTPS 301 và kích hoạt SSL trên cổng 443.
10. Mở cổng 443 trên UFW và reload Nginx: `sudo nginx -t && sudo systemctl reload nginx`.
11. Cố tình tắt Node.js (`pkill node`), quan sát lỗi `502`, đọc log `/var/log/nginx/error.log` và bật lại ứng dụng để khôi phục dịch vụ.

---

## 📌 10. Bảng Tra Cứu Lệnh Nginx Cốt Lõi

| Lệnh | Ý nghĩa & Khi nào dùng |
| :--- | :--- |
| `sudo nginx -t` | Kiểm tra cú pháp toàn bộ file config (Bắt buộc chạy trước khi reload) |
| `sudo systemctl reload nginx` | Nạp lại cấu hình không gián đoạn kết nối (Zero-downtime) |
| `sudo systemctl restart nginx` | Khởi động lại toàn bộ tiến trình Nginx |
| `sudo systemctl status nginx` | Kiểm tra trạng thái tiến trình |
| `sudo tail -f /var/log/nginx/access.log` | Theo dõi lưu lượng truy cập thời gian thực |
| `sudo tail -f /var/log/nginx/error.log` | Theo dõi và chẩn đoán log lỗi (403, 502, 504) |
| `sudo ln -s <available> <enabled>` | Kích hoạt một Server Block |
| `sudo rm /etc/nginx/sites-enabled/<site>` | Hủy kích hoạt một Server Block |

