# 📘 Phần 12: HTTPS & SSL/TLS – Mã Hóa & Quản Trị Chứng Chỉ Bảo Mật

> **Motto cốt lõi:**  
> *HTTP là bản rõ (Plaintext) – HTTPS là sự an toàn tuyệt đối | Hiểu bản chất bắt tay TLS (TLS Handshake) | Tự động hóa 100% chứng chỉ với Let's Encrypt Certbot & Auto-renewal!*  
> Chu trình chuẩn: **Bản chất Mã hóa & PKI → Cơ chế Let's Encrypt (ACME) → Cài đặt Certbot → Cấu hình Nginx Đạt Chuẩn A+ → Tự Động Gia Hạn → Debug & Lab.**

---

## 🎯 1. Mục Tiêu Của Phần Học

Sau khi hoàn thành phần này, bạn sẽ:
1. **Hiểu bản chất HTTP vs HTTPS:** Biết dữ liệu bị nghe lén (Sniffing) và giả mạo (Man-in-the-Middle) như thế nào nếu không có mã hóa.
2. **Nắm vững cơ chế TLS Handshake & PKI:** Hiểu sự phối hợp giữa mã hóa bất đối xứng (RSA/ECC) để trao đổi khóa và mã hóa đối xứng (AES) để truyền tải dữ liệu tốc độ cao.
3. **Phân biệt các loại Chứng chỉ số (X.509):** Hiểu rõ vai trò của Root CA, Intermediate CA, Certificate (`.crt`), Private Key (`.key`) và Fullchain (`.pem`).
4. **Làm chủ Let's Encrypt & Certbot:** Triển khai cấp phát chứng chỉ SSL hoàn toàn miễn phí, tự động xác thực qua giao thức ACME (HTTP-01 Challenge).
5. **Cấu hình Nginx HTTPS chuẩn A+ Security:** Bật TLS 1.2/1.3, thiết lập HSTS, HTTP-to-HTTPS 301 Redirect và OCSP Stapling.
6. **Thiết lập chu trình tự động gia hạn (Auto-renewal):** Đảm bảo chứng chỉ 90 ngày của Let's Encrypt không bao giờ bị hết hạn (Expired) với `systemd timer`.
7. **Xử lý các sự cố kinh điển:** Sửa lỗi xác thực Certbot thất bại, lỗi cổng 80 bị chặn, lỗi Mixed Content và cách thu hồi/xóa chứng chỉ an toàn.

---

## 🔐 2. Bản Chất: HTTP vs HTTPS & Cơ Chế Bắt Tay TLS (TLS Handshake)

### 2.1. Tại sao HTTP lại cực kỳ nguy hiểm?

```text
┌─────────────────────────────────────────────────────────────────────────┐
│                           HTTP (PORT 80) - NGUY HIỂM                    │
│   Client (Mac) ───[ Mật khẩu: "123456" (Bản rõ Plaintext) ]───► Server  │
│                                 │                                       │
│                                 ▼                                       │
│                    Hacker ở giữa (Wireshark/Router)                     │
│                    ──► Đọc trộm được 100% mật khẩu & Cookie!             │
├─────────────────────────────────────────────────────────────────────────┤
│                           HTTPS (PORT 443) - AN TOÀN                    │
│   Client (Mac) ───[ Dữ liệu mã hóa: "a8#9!zK$1x90..." ]───────► Server  │
│                                 │                                       │
│                                 ▼                                       │
│                    Hacker ở giữa (Wireshark/Router)                     │
│                    ──► Chỉ thấy chuỗi rác vô nghĩa!                     │
└─────────────────────────────────────────────────────────────────────────┘
```

---

### 2.2. Cơ chế Bắt tay TLS 1.3 (TLS Handshake Lifecycle)

Làm sao Client và Server có thể trao đổi mật mã an toàn qua đường truyền công cộng mà không bị lộ khóa?

```text
    CLIENT (Trình duyệt trên Mac)                   SERVER (Ubuntu Nginx)
                 │                                           │
                 │ 1. Client Hello (Các bộ mã hỗ trợ, TLS1.3)│
                 ├──────────────────────────────────────────►│
                 │                                           │
                 │ 2. Server Hello + Certificate (Fullchain) │
                 │    + Trao đổi khóa công khai (ECDHE)      │
                 │◄──────────────────────────────────────────┤
                 │                                           │
                 │ 3. Client xác thực chứng chỉ với Root CA  │
                 │    (Đảm bảo đúng Server thật, không giả mạo)
                 │                                           │
                 │ 4. Cả 2 bên tự sinh ra Khóa Phiên Đối Xứng│
                 │    (Symmetric Session Key - AES 256)      │
                 │                                           │
                 │ 5. Bắt đầu truyền dữ liệu siêu tốc độ đã mã hóa (HTTPS)
                 │◄═════════════════════════════════════════►│
```

* **Mã hóa bất đối xứng (Asymmetric):** Dùng ở giai đoạn đầu để xác thực danh tính server và thỏa thuận một "Khóa phiên" bí mật.
* **Mã hóa đối xứng (Symmetric - AES):** Dùng để truyền dữ liệu thực tế vì tốc độ xử lý nhanh hơn mã hóa bất đối xứng hàng ngàn lần.

---

## 📜 3. Giải Mã Hệ Thống Chứng Chỉ Số (PKI & X.509 Files)

Khi bạn cấu hình SSL cho Nginx, bạn sẽ làm việc với các file chứng chỉ:

```text
/etc/letsencrypt/live/yourdomain.com/
├── privkey.pem     ──► PRIVATE KEY (Khóa bí mật của Server - TUYỆT ĐỐI KHÔNG LỘ!)
├── cert.pem        ──► CERTIFICATE (Chứng chỉ cấp riêng cho domain của bạn)
├── chain.pem       ──► INTERMEDIATE CA (Chứng chỉ của đơn vị trung gian)
└── fullchain.pem   ──► FULLCHAIN = cert.pem + chain.pem (FILE DÙNG ĐỂ GẮN VÀO NGINX)
```

```text
┌─────────────────────────────────────────────────────────────────────────┐
│                       CHUỖI XÁC THỰC TIN CẬY (CHAIN OF TRUST)           │
├─────────────────────────────────────────────────────────────────────────┤
│  ROOT CA (Được tích hợp sẵn trong MacOS/Windows/iOS/Android từ trước)   │
│     │                                                                   │
│     ▼ (Ký xác nhận)                                                     │
│  INTERMEDIATE CA (Let's Encrypt Authority X3 / R3)                     │
│     │                                                                   │
│     ▼ (Ký cấp phát)                                                     │
│  YOUR DOMAIN CERTIFICATE (mycompany.com)                                │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 🤖 4. Let's Encrypt & Certbot: Cơ Chế Xác Thực ACME

**Let's Encrypt** là tổ chức phi lợi nhuận (CA) cấp chứng chỉ SSL miễn phí tự động. Để chứng minh bạn thực sự sở hữu tên miền, công cụ **Certbot** sử dụng cơ chế **HTTP-01 Challenge**:

```text
                             CƠ CHẾ HTTP-01 CHALLENGE
                             
   1. Certbot gửi yêu cầu cấp SSL cho domain "api.example.com" ──► Let's Encrypt CA
                                                                         │
   2. Let's Encrypt tạo ra 1 mã bí mật Token ngẫu nhiên ───────────────┤
      "Hãy đặt file Token này vào:                                       │
       http://api.example.com/.well-known/acme-challenge/<Token>"        │
                                                                         │
   3. Nginx tự động phục vụ file Token đó cho Let's Encrypt đọc        │
                                                                         │
   4. Let's Encrypt đọc đúng Token ──► Xác nhận chủ sở hữu hợp lệ! ◄───┘
                                       Cấp ngay Fullchain SSL Certificate!
```

---

## 🛠️ 5. Cài Đặt Certbot & Cấp Chứng Chỉ SSL Tự Động Cho Nginx

> [!NOTE]
> Để cấp chứng chỉ Let's Encrypt thật, bạn cần:
> 1. Có một tên miền thật (Domain) đã trỏ bản ghi **A Record** về địa chỉ IP Public của máy chủ.
> 2. Đã mở cổng **80/tcp** và **443/tcp** trên Tường lửa UFW.

---

### Bước 5.1: Cài đặt Certbot qua Snap (Khuyến nghị chuẩn của Canonical)

```bash
# 1. Cài đặt snapd (nếu chưa có)
sudo apt update
sudo apt install -y snapd

# 2. Cài đặt Certbot bản mới nhất
sudo snap install --classic certbot

# 3. Tạo symlink để gọi lệnh certbot trực tiếp
sudo ln -sf /snap/bin/certbot /usr/bin/certbot
```

---

### Bước 5.2: Chạy Certbot tự động cấu hình Nginx

Chỉ với 1 câu lệnh duy nhất, Certbot sẽ tự động giải mã HTTP Challenge, lấy chứng chỉ và chèn toàn bộ cấu hình HTTPS vào Nginx:

```bash
# Thay yourdomain.com bằng tên miền thật của bạn
sudo certbot --nginx -d yourdomain.com -d www.yourdomain.com
```

*Quá trình tương tác của Certbot:*
1. **Nhập Email:** Dùng để nhận thông báo cảnh báo khi chứng chỉ sắp hết hạn.
2. **Đồng ý điều khoản (Terms of Service):** Gõ `Y`.
3. **Chia sẻ Email cho EFF (Electronic Frontier Foundation):** Gõ `N` (hoặc `Y`).
4. **Tự động cấu hình Redirect:** Certbot sẽ tự động sửa file cấu hình Nginx để ép mọi truy cập HTTP chuyển sang HTTPS 301!

---

### Bước 5.3: Cấu hình Thủ Công Nginx Chuẩn A+ Security (Nâng Cao)

Nếu bạn tự viết cấu hình hoặc muốn tối ưu hóa bảo mật đạt điểm **A+ trên SSL Labs**, hãy sử dụng file cấu hình chuẩn sau:

```nginx
# 1. HTTP: Tự động chuyển hướng toàn bộ sang HTTPS
server {
    listen 80;
    listen [::]:80;
    server_name yourdomain.com www.yourdomain.com;

    return 301 https://$host$request_uri;
}

# 2. HTTPS: Xử lý mã hóa và Reverse Proxy
server {
    listen 443 ssl http2;
    listen [::]:443 ssl http2;
    server_name yourdomain.com www.yourdomain.com;

    # Đường dẫn chứng chỉ Let's Encrypt
    ssl_certificate /etc/letsencrypt/live/yourdomain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/yourdomain.com/privkey.pem;

    # =========================================================================
    # THIẾT LẬP BẢO MẬT CHUẨN A+ (TLS 1.2 / TLS 1.3 & Modern Ciphers)
    # =========================================================================
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_prefer_server_ciphers on;
    ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384;

    # SSL Session Cache: Giúp Client kết nối lại cực nhanh không cần bắt tay lại từ đầu
    ssl_session_timeout 1d;
    ssl_session_cache shared:SSL:10m;
    ssl_session_tickets off;

    # HSTS (HTTP Strict Transport Security): Ép trình duyệt luôn vào HTTPS trong 1 năm
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;

    # OCSP Stapling: Tăng tốc độ kiểm tra tính hợp lệ của chứng chỉ
    ssl_stapling on;
    ssl_stapling_verify on;

    # Reverse Proxy về ứng dụng NestJS / Node.js
    location / {
        proxy_pass http://127.0.0.1:3000;
        
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto https;
    }
}
```

---

## 🔄 6. Cơ Chế Tự Động Gia Hạn (Auto-Renewal)

Chứng chỉ của Let's Encrypt chỉ có thời hạn **90 ngày** (để đảm bảo tính bảo mật và buộc hệ thống phải tự động hóa).

### 6.1. Kiểm tra Systemd Timer tự động gia hạn của Certbot

Khi cài bằng Snap, Certbot tự động cài một **Systemd Timer** chạy ngầm 2 lần mỗi ngày để kiểm tra các chứng chỉ còn dưới 30 ngày và tự động gia hạn:

```bash
# Kiểm tra timer tự động gia hạn
sudo systemctl status snap.certbot.renew.timer
```

---

### 6.2. Kiểm tra thử nghiệm gia hạn giả lập (Dry Run)

Bạn có thể chạy thử nghiệm quy trình gia hạn mà không làm ảnh hưởng đến chứng chỉ đang hoạt động:

```bash
sudo certbot renew --dry-run
```

*Output chuẩn khi thành công:*
```text
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
Simulating renewal of an existing certificate for yourdomain.com
The dry run was successful.
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
```

---

## 🧪 7. Thực Hành Lab Môi Trường Cục Bộ: OpenSSL Self-Signed SSL

Trong môi trường Lab trên máy Mac qua Multipass VM (không có domain công khai), chúng ta thực hành sinh chứng chỉ **Self-Signed (Tự ký)** chuẩn cấu trúc X.509.

### Bước 7.1: Sinh cặp khóa và chứng chỉ bảo mật
```bash
sudo mkdir -p /etc/nginx/ssl
sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout /etc/nginx/ssl/lab-server.key \
  -out /etc/nginx/ssl/lab-server.crt \
  -subj "/C=VN/ST=Hanoi/L=Hanoi/O=SecurityLab/CN=192.168.64.2"

# Phân quyền nghiêm ngặt: Chỉ root đọc được Private Key!
sudo chmod 600 /etc/nginx/ssl/lab-server.key
sudo chmod 644 /etc/nginx/ssl/lab-server.crt
```

### Bước 7.2: Soi nội dung chi tiết của chứng chỉ số X.509
Để đọc thông tin tổ chức, ngày hết hạn, mã băm (Fingerprint) của chứng chỉ:
```bash
openssl x509 -in /etc/nginx/ssl/lab-server.crt -text -noout | grep -E "Issuer|Subject|Not After"
```

---

## 🚨 8. Xử Lý 3 Sự Cố Phổ Biến Khi Cấu Hình HTTPS

### 🔴 Lỗi 1: `Certbot failed to authenticate some domains (HTTP-01 failed)`
* **Nguyên nhân:** Let's Encrypt không thể kết nối tới server qua cổng 80 để đọc token xác thực.
* **Cách khắc phục:**
  1. Kiểm tra DNS: Tên miền đã thực sự trỏ về IP máy chủ chưa (`dig +short yourdomain.com` hoặc `ping yourdomain.com`).
  2. Kiểm tra Tường lửa: Cổng 80 đã mở trên UFW chưa (`sudo ufw allow 80/tcp`).
  3. Kiểm tra Nginx: Nginx có đang chạy không (`sudo systemctl status nginx`).

---

### 🔴 Lỗi 2: `Mixed Content Warning` (Trang có ổ khóa vàng / mất ổ khóa xanh)
* **Nguyên nhân:** Trang web chạy HTTPS nhưng trong mã nguồn HTML/JS lại gọi ảnh hoặc API qua giao thức `http://` không mã hóa.
* **Cách khắc phục:** Đổi toàn bộ đường dẫn ảnh/API trong code sang đường dẫn tương đối (`/api/...`) hoặc dùng `https://`.

---

### 🔴 Lỗi 3: Xóa hoặc Thu hồi chứng chỉ (Revoke / Delete)
Khi bạn không dùng tên miền đó nữa hoặc muốn dọn dẹp chứng chỉ cũ:
```bash
# Xem danh sách chứng chỉ đang có
sudo certbot certificates

# Xóa chứng chỉ an toàn khỏi hệ thống
sudo certbot delete --cert-name yourdomain.com
```

---

## 🧪 9. Bài Thực Hành Lab (10 Bước Triển Khai)

1. Kiểm tra trạng thái cổng 80 và 443 trên UFW: `sudo ufw status`.
2. Đảm bảo cổng 443 đã được mở: `sudo ufw allow 443/tcp`.
3. Sinh cặp chứng chỉ SSL Self-signed với `openssl`.
4. Soi thông tin chi tiết của chứng chỉ bằng `openssl x509`.
5. Cấu hình Nginx Server Block tại `/etc/nginx/sites-available/secure-app.conf`.
6. Cấu hình khối Server chuyển hướng HTTP 80 $\rightarrow$ HTTPS 443 với `return 301`.
7. Gắn `ssl_certificate` và `ssl_certificate_key` vào khối HTTPS 443.
8. Kiểm tra cú pháp `sudo nginx -t` và nạp lại `sudo systemctl reload nginx`.
9. Thử nghiệm request HTTPS từ máy Mac với `curl -k -I https://<IP_VM>`.
10. Kiểm tra trình tự động gia hạn của Certbot: `sudo certbot renew --dry-run`.

---

## 📌 10. Bảng Tra Cứu Lệnh SSL / Certbot Cốt Lõi

| Lệnh | Ý nghĩa & Khi nào dùng |
| :--- | :--- |
| `sudo certbot --nginx -d <domain>` | Cấp chứng chỉ SSL và tự động cấu hình Nginx |
| `sudo certbot certonly --standalone -d <domain>` | Cấp chứng chỉ độc lập (không cần Nginx) |
| `sudo certbot certificates` | Liệt kê tất cả các chứng chỉ SSL đang quản lý và hạn dùng |
| `sudo certbot renew --dry-run` | Kiểm tra thử nghiệm tính năng tự động gia hạn (Mô phỏng) |
| `sudo certbot delete --cert-name <name>` | Xóa bỏ một chứng chỉ SSL không còn sử dụng |
| `openssl req -x509 ...` | Tạo chứng chỉ SSL tự ký (Self-Signed) cho môi trường Lab |
| `openssl x509 -in cert.crt -text -noout` | Đọc nội dung chi tiết của file chứng chỉ số X.509 |
