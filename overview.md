# 📘 Tổng Quan Kiến Thức Cốt Lõi: Cấu Hình Ubuntu Server Nhanh & An Toàn Chuẩn Production

> **Motto cốt lõi:**  
> *Hạ tầng an ninh vững chắc $\rightarrow$ Môi trường tối ưu $\rightarrow$ Vận hành tự động | Tối giản thao tác nhưng đạt tiêu chuẩn an toàn cao nhất trong môi trường thực tế.*

---

## 🎯 1. Triết Lý Thiết Kế & Mô Hình Phòng Thủ Chiều Sâu (Defense in Depth)

Một máy chủ Production đạt chuẩn không phụ thuộc vào một giải pháp duy nhất, mà là sự phối hợp của nhiều lớp bảo vệ độc lập:

```text
INTERNET (Người dùng & Botnet / Hacker)
  │
  ▼
┌─────────────────────────────────────────────────────────────────────────┐
│ [LỚP 1] UFW FIREWALL (Tường lửa mạng)                                   │
│ ──► Chặn 65.532 cổng, chỉ mở duy nhất: 22 (SSH), 80 (HTTP), 443 (HTTPS) │
└────────────────────────────────────┬────────────────────────────────────┘
                                     │
                                     ▼
┌─────────────────────────────────────────────────────────────────────────┐
│ [LỚP 2] FAIL2BAN (Hệ thống ngăn chặn xâm nhập động)                     │
│ ──► Tự động quét log, khóa IP nếu gõ sai SSH 3 lần hoặc dò quét file cấm │
└────────────────────────────────────┬────────────────────────────────────┘
                                     │
                                     ▼
┌─────────────────────────────────────────────────────────────────────────┐
│ [LỚP 3] SSH HARDENING (Bảo mật truy cập quản trị)                       │
│ ──► Cấm root login, tắt xác thực bằng Password, chỉ nhận SSH Key        │
└────────────────────────────────────┬────────────────────────────────────┘
                                     │
                                     ▼
┌─────────────────────────────────────────────────────────────────────────┐
│ [LỚP 4] NGINX REVERSE PROXY & CERTBOT (Mặt tiền Web Server)             │
│ ──► Đón tiếp traffic port 80/443, giải mã SSL (SSL Termination),        │
│     chuyển tiếp HTTP nội bộ vào ứng dụng                                │
└────────────────────────────────────┬────────────────────────────────────┘
                                     │
                                     ▼ (Nội bộ 127.0.0.1:3000)
┌─────────────────────────────────────────────────────────────────────────┐
│ [LỚP 5] SYSTEMD PROCESS MANAGER (Quản lý tiến trình ứng dụng)           │
│ ──► Chạy user thường, tự động hồi sinh khi crash, tự chạy khi reboot    │
└────────────────────────────────────┬────────────────────────────────────┘
                                     │
                                     ▼ (Mạng Docker Bridge nội bộ)
┌─────────────────────────────────────────────────────────────────────────┐
│ [LỚP 6] DOCKER DATABASE (Hạ tầng lưu trữ dữ liệu cô lập)                │
│ ──► PostgreSQL/MySQL gắn Named Volume, chỉ bind 127.0.0.1               │
└────────────────────────────────────┬────────────────────────────────────┘
                                     │
                                     ▼ (Chạy ngầm định kỳ 02:00 AM)
┌─────────────────────────────────────────────────────────────────────────┐
│ [LỚP 7] AUTOMATED BACKUP (Sao lưu dữ liệu tự động)                      │
│ ──► Tự động dump DB, nén code/uploads, tự dọn dẹp file cũ > 7 ngày      │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## ⚡ 2. Quy Trình Cấu Hình Nhanh 5 Bước (Fast-Track Runbook)

### BƯỚC 1: Củng Cố An Ninh Hệ Thống (5 Phút Đầu Tiên)

#### 1. Cập nhật hệ điều hành & Tạo tài khoản Deploy riêng
Tuyệt đối không sử dụng tài khoản `root` trực tiếp để chạy các dịch vụ.

```bash
# Cập nhật danh sách gói và nâng cấp hệ thống
sudo apt update && sudo apt upgrade -y

# Tạo user chuyên dùng để deploy ứng dụng
sudo adduser deployer

# Cấp quyền quản trị có kiểm soát qua sudo
sudo usermod -aG sudo deployer
```

#### 2. Thiết lập SSH Key Authentication & Khóa xác thực bằng Password
Từ máy tính cá nhân (Mac/Linux), chuyển Public Key lên server:
```bash
ssh-copy-id deployer@<IP_SERVER>
```

Đăng nhập vào server bằng user `deployer` và tạo file cấu hình bảo vệ SSH tại `/etc/ssh/sshd_config.d/hardening.conf`:
```bash
sudo tee /etc/ssh/sshd_config.d/hardening.conf << 'EOF'
# Cấm đăng nhập trực tiếp bằng tài khoản root
PermitRootLogin no

# Tắt hoàn toàn đăng nhập bằng mật khẩu (chống Brute-force 100%)
PasswordAuthentication no

# Chỉ cho phép đăng nhập bằng SSH Key
PubkeyAuthentication yes
AuthorizedKeysFile .ssh/authorized_keys
EOF

# Kiểm tra cú pháp an toàn trước khi nạp lại dịch vụ
sudo sshd -t && sudo systemctl reload ssh
```

#### 3. Bật Tường lửa UFW (Nguyên tắc Default Deny)
> [!CAUTION]
> **Quy tắc sống còn:** Luôn cho phép cổng SSH (`22/tcp`) trước khi kích hoạt `ufw enable` để không bị ngắt kết nối và vĩnh viễn khóa ngoài máy chủ!

```bash
# Chính sách: Chặn mọi kết nối vào, cho phép mọi kết nối ra
sudo ufw default deny incoming
sudo ufw default allow outgoing

# Mở cổng SSH với cơ chế Rate-Limit (chống spam kết nối)
sudo ufw limit 22/tcp

# Mở cổng Web chuẩn
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp

# Kích hoạt tường lửa
sudo ufw enable
```

#### 4. Kích hoạt Fail2ban tự động khóa IP tấn công
```bash
sudo apt install -y fail2ban

sudo tee /etc/fail2ban/jail.local << 'EOF'
[DEFAULT]
ignoreip = 127.0.0.1/8 ::1
bantime  = 1h
findtime = 10m
maxretry = 5

# Ban lũy tiến: Tái phạm tăng gấp đôi thời gian cấm
bantime.increment = true
bantime.factor = 2
bantime.maxtime = 4w

backend  = systemd
banaction = ufw

[sshd]
enabled  = true
port     = ssh
maxretry = 3
bantime  = 24h

[nginx-botsearch]
enabled  = true
port     = http,https
filter   = nginx-botsearch
logpath  = /var/log/nginx/error.log
backend  = auto
maxretry = 2
bantime  = 48h

[nginx-limit-req]
enabled  = true
port     = http,https
filter   = nginx-limit-req
logpath  = /var/log/nginx/error.log
backend  = auto
maxretry = 5
findtime = 2m
bantime  = 2h
EOF

sudo systemctl enable --now fail2ban
```

---

### BƯỚC 2: Cài Đặt Môi Trường Thực Thi (NVM + Node.js 24 + PNPM)

Thực hiện dưới tài khoản `deployer` (không cần và không nên dùng `sudo`):

```bash
# 1. Tải và cài đặt NVM v0.40.7
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.7/install.sh | bash

# Nạp môi trường NVM vào shell hiện tại
source ~/.bashrc

# 2. Cài đặt Node.js 24 LTS và đặt làm phiên bản mặc định
nvm install 24
nvm alias default 24
nvm use 24

# Kiểm tra phiên bản
node -v # In ra: v24.x.x
npm -v  # In ra: 11.x.x

# 3. Cài đặt PNPM (Trình quản lý package siêu nhanh và tiết kiệm đĩa)
npm install -g pnpm @nestjs/cli

pnpm -v
```

---

### BƯỚC 3: Dựng Cơ Sở Dữ Liệu Qua Docker (An Toàn & Bền Vững)

Không cài đặt trực tiếp Database vào OS để giữ sạch máy chủ. Sử dụng Docker với **Named Volumes** để đảm bảo dữ liệu không bao giờ bị mất:

```bash
# 1. Cài đặt Docker Engine và Compose plugin
sudo apt install -y docker.io docker-compose-v2
sudo usermod -aG docker deployer
newgrp docker

# 2. Tạo thư mục quản trị Database tập trung
mkdir -p ~/databases && cd ~/databases

tee docker-compose.yml << 'EOF'
version: '3.8'

services:
  postgres_db:
    image: postgres:16-alpine
    container_name: production_postgres
    restart: always
    environment:
      POSTGRES_USER: app_user
      POSTGRES_PASSWORD: StrongDbPassword2026!
      POSTGRES_DB: app_production
    ports:
      # CHỈ mở cho localhost nội bộ (127.0.0.1), Internet không thể tiếp cận
      - "127.0.0.1:5432:5432"
    volumes:
      - pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U app_user -d app_production"]
      interval: 5s
      timeout: 5s
      retries: 5

volumes:
  pgdata:
    name: postgres_persistent_data
EOF

# Khởi chạy Database ngầm
docker compose up -d
```

---

### BƯỚC 4: Triển Khai Backend & Quản Trị Bằng Systemd

#### 1. Đóng gói mã nguồn bằng PNPM
```bash
cd ~/apps/my-app

# Cài đặt dependencies chuẩn xác 100% theo lockfile
pnpm install --frozen-lockfile

# Biên dịch TypeScript sang JavaScript thuần
pnpm run build

# Khóa quyền đọc file môi trường (chỉ owner đọc được)
chmod 600 .env
```

#### 2. Tạo Systemd Service Unit
Tạo file `/etc/systemd/system/backend.service`:
```ini
[Unit]
Description=Backend Production Application Service
After=network.target

[Service]
Type=simple
User=deployer
Group=deployer
WorkingDirectory=/home/deployer/apps/my-app

# Dùng đường dẫn tuyệt đối của Node.js (kiểm tra bằng `which node`)
ExecStart=/home/deployer/.nvm/versions/node/v24.20.0/bin/node dist/main.js

# Cơ chế tự phục hồi khi crash
Restart=always
RestartSec=5s

# Nạp biến môi trường
Environment=NODE_ENV=production
EnvironmentFile=/home/deployer/apps/my-app/.env

# Ghi log tập trung về journald
StandardOutput=journal
StandardError=journal
SyslogIdentifier=backend-api

[Install]
WantedBy=multi-user.target
```

```bash
# Nạp cấu hình và khởi chạy dịch vụ
sudo systemctl daemon-reload
sudo systemctl enable --now backend

# Kiểm tra trạng thái
sudo systemctl status backend
```

---

### BƯỚC 5: Nginx Reverse Proxy & Tự Động Hóa SSL Với Certbot

#### 1. Cấu hình Nginx Server Block
Tạo file `/etc/nginx/sites-available/app.conf`:
```nginx
server {
    listen 80;
    server_name api.yourdomain.com;

    location / {
        proxy_pass http://127.0.0.1:3000;

        # Bộ Headers chuẩn bắt buộc để giữ thông tin Client
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        proxy_connect_timeout 60s;
        proxy_read_timeout 60s;
    }
}
```

```bash
# Kích hoạt cấu hình mới và xóa site mặc định
sudo ln -sf /etc/nginx/sites-available/app.conf /etc/nginx/sites-enabled/
sudo rm -f /etc/nginx/sites-enabled/default

# Kiểm tra cú pháp và nạp lại Nginx (Zero-Downtime)
sudo nginx -t && sudo systemctl reload nginx
```

#### 2. Kích hoạt HTTPS tự động bằng Certbot (Let's Encrypt)
```bash
# Cài đặt Certbot qua Snap
sudo apt install -y snapd
sudo snap install --classic certbot
sudo ln -sf /snap/bin/certbot /usr/bin/certbot

# Tự động xin chứng chỉ, nạp vào Nginx và cấu hình 301 Redirect HTTP->HTTPS
sudo certbot --nginx -d api.yourdomain.com
```

---

## 💾 3. Tự Động Hóa Sao Lưu (Backup Retention Script)

Tạo file script `/opt/scripts/backup.sh`:
```bash
#!/bin/bash
DATE=$(date +%Y-%m-%d_%H%M%S)
BACKUP_DIR="/var/backups/system"
mkdir -p "$BACKUP_DIR"

# 1. Dump Database từ Docker
docker exec -t production_postgres pg_dump -U app_user app_production > "$BACKUP_DIR/db_$DATE.sql"

# 2. Nén CSDL và Mã nguồn
tar -czf "$BACKUP_DIR/backup_$DATE.tar.gz" -C "$BACKUP_DIR" "db_$DATE.sql" /home/deployer/apps/my-app/.env
rm -f "$BACKUP_DIR/db_$DATE.sql"

# 3. Tự động xóa các bản sao lưu cũ hơn 7 ngày (Chống đầy ổ cứng)
find "$BACKUP_DIR" -type f -name "backup_*.tar.gz" -mtime +7 -delete
```

Cấp quyền và lập lịch qua Crontab:
```bash
sudo chmod +x /opt/scripts/backup.sh

# Mở crontab (crontab -e) và thêm dòng:
# 0 2 * * * /opt/scripts/backup.sh > /dev/null 2>&1
```

---

## 📌 4. Bảng So Sánh & Nguyên Tắc Vàng (Best Practices)

| Hạng mục | ✅ Chuẩn Production | ❌ Sai lầm thường gặp |
| :--- | :--- | :--- |
| **Quyền truy cập** | Dùng user `deployer` có sudo, xác thực bằng SSH Key | Đăng nhập trực tiếp bằng `root`, dùng mật khẩu yếu |
| **Tường lửa** | `default deny incoming`, chỉ mở cổng 22, 80, 443 | Mở toang các cổng CSDL (5432, 3306, 27017) ra ngoài |
| **Quản trị App** | Quản lý bằng **Systemd**, tự hồi sinh khi crash | Chạy bằng `node` thủ công, dùng `nohup` hoặc `&` |
| **Môi trường JS** | Dùng `pnpm install --frozen-lockfile`, chạy `node dist/main.js` | Chạy `npm run start:dev` trên máy chủ thật |
| **Cơ sở dữ liệu** | Chạy Docker bind `127.0.0.1`, lưu qua **Named Volume** | Chạy container không mount volume (mất trắng data khi xóa container) |
| **Nạp cấu hình Web** | Kiểm tra `nginx -t` trước, nạp bằng `systemctl reload` | Chạy `systemctl restart nginx` gây rớt kết nối người dùng |
| **Bí mật (.env)** | Phân quyền `chmod 600 .env`, đưa vào GitHub Secrets | Commit `.env` hoặc Private Key lên Git repository |

---

## 🔍 5. Checklist 60 Giây Kiểm Tra & Nghiệm Thu Hệ Thống

Trước khi đưa máy chủ vào hoạt động chính thức, chạy chuỗi 5 câu lệnh sau để nghiệm thu:

```bash
# 1. Tường lửa đang bật và chỉ mở đúng 3 cổng (22, 80, 443)?
sudo ufw status

# 2. Cổng Database 5432 có được bảo vệ và chỉ lắng nghe nội bộ 127.0.0.1?
sudo ss -tulpn | grep -E ':(22|80|443|5432|3000)'

# 3. Fail2ban đang theo dõi và bảo vệ cổng SSH?
sudo fail2ban-client status sshd

# 4. Backend Systemd đang Active và tự động chạy khi khởi động lại?
sudo systemctl status backend

# 5. Website / API đã có HTTPS hợp lệ?
curl -I https://api.yourdomain.com
```
