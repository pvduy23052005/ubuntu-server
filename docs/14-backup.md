# 📘 Phần 14: Backup – Chiến Lược & Tự Động Hóa Sao Lưu Dữ Liệu Máy Chủ

> **Motto cốt lõi:**  
> *Một bản sao lưu chưa từng được thử nghiệm khôi phục (Restore) thì coi như CHƯA CÓ BACKUP!*  
> Quy tắc vàng: **Chiến lược 3-2-1 | Tự động hóa hoàn toàn với Bash Script & Crontab | Thiết lập chính sách lưu trữ (Retention Policy) để không làm tràn ổ đĩa!**

---

## 🎯 1. Mục Tiêu Của Phần Học

Sau khi hoàn thành phần này, bạn sẽ:
1. **Nắm vững Chiến lược Sao lưu 3-2-1:** Tiêu chuẩn công nghiệp về an toàn dữ liệu và phòng chống thảm họa (Disaster Recovery).
2. **Phân biệt các loại sao lưu:** Full Backup (Toàn phần), Incremental Backup (Gia tăng) và Differential Backup (Vi sai).
3. **Xác định các thành phần bắt buộc phải Backup:** Cơ sở dữ liệu (PostgreSQL/MySQL), Mã nguồn & Tệp tải lên (Uploads/Volumes), và File cấu hình hệ thống (`/etc/`).
4. **Làm chủ các công cụ lưu trữ & đồng bộ:** `tar`, `gzip`, `rsync` (đồng bộ qua SSH) và `rclone` (đồng bộ lên Cloud S3/Google Drive).
5. **Viết Bash Script tự động hóa chuẩn Production:** Tự động xuất CSDL, nén tệp, đặt tên theo ngày giờ, dọn dẹp các bản backup cũ quá 7 ngày và ghi log.
6. **Lập lịch chạy ngầm với Crontab:** Hiểu cú pháp 5 dấu sao (`* * * * *`) và thiết lập lịch backup tự động vào 02:00 sáng mỗi ngày.
7. **Thực hành Diễn tập Khôi phục (Disaster Recovery Drill):** Tự tay phục hồi hệ thống từ file backup khi xảy ra sự cố mất dữ liệu.

---

## 🏛️ 2. Bản Chất: Chiến Lược Sao Lưu 3-2-1 & Các Thành Phần Cốt Lõi

### 2.1. Quy tắc vàng 3-2-1 trong Quản trị Hệ thống

```text
┌─────────────────────────────────────────────────────────────────────────┐
│                    CHIẾN LƯỢC SAO LƯỢC CHUẨN 3-2-1                      │
├─────────────────────────────────────────────────────────────────────────┤
│  3 BẢN SAO (Copies)      ──► 1 Bản chính đang chạy + 2 Bản sao lưu      │
│  2 LOẠI ĐỊNH DẠNG/THIẾT BỊ ─► Lưu trên SSD máy chủ + Lưu trên NAS/Ổ cứng │
│  1 BẢN TÁCH BIỆT (Offsite) ─► 1 Bản đẩy lên Cloud (AWS S3, Cloudflare R2│
│                              hoặc Google Drive ở vị trí địa lý khác)    │
└─────────────────────────────────────────────────────────────────────────┘
```

> [!WARNING]
> **Sai lầm chết người:** Lưu file backup ngay trên chính ổ đĩa của máy chủ mà không đẩy ra ngoài. Khi máy chủ bị cháy phần cứng, lỗi ổ đĩa hoặc bị hacker chiếm quyền xóa trắng, bạn sẽ mất cả hệ thống lẫn file backup!

---

### 2.2. Danh mục 3 nhóm dữ liệu bắt buộc phải sao lưu

```text
UBUNTU SERVER CỦA BẠN
   │
   ├── 1. CƠ SỞ DỮ LIỆU (Database) ──────► PostgreSQL (pg_dump), MySQL, MongoDB
   │                                       (Dữ liệu thay đổi từng giây)
   │
   ├── 2. TỆP NGƯỜI DÙNG TẢI LÊN (Media) ─► /var/www/uploads/, Docker Named Volumes
   │                                       (Ảnh đại diện, tài liệu, video)
   │
   └── 3. CẤU HÌNH HỆ THỐNG (Configs) ───► /etc/nginx/, /etc/systemd/system/,
                                           /etc/fail2ban/, các file .env
```

---

## 🧰 3. Bộ Công Cụ Sao Lưu Cốt Lõi Trên Linux

### 3.1. Đóng gói và Nén với `tar` & `gzip`

Lệnh `tar` (Tape Archive) dùng để gom hàng ngàn file/thư mục thành 1 file duy nhất và nén lại:

```bash
# Cú pháp tạo file nén .tar.gz:
# -c: Create (tạo mới)
# -z: Gzip (nén bằng gzip)
# -v: Verbose (hiển thị tiến trình)
# -f: File (tên file đầu ra)

# 1. Nén toàn bộ thư mục cấu hình Nginx
sudo tar -czvf /backup/nginx_config_$(date +%F).tar.gz /etc/nginx/

# 2. Giải nén file backup vào một thư mục cụ thể (-x: Extract)
sudo tar -xzvf /backup/nginx_config_2026-08-26.tar.gz -C /tmp/restore/
```

---

### 3.2. Đồng bộ dữ liệu thông minh qua SSH với `rsync`

`rsync` là công cụ đồng bộ siêu nhanh vì nó chỉ truyền tải các phần file có sự thay đổi (Delta Transfer):

```bash
# Đồng bộ thư mục /var/www/ từ Ubuntu Server sang máy Mac (hoặc Remote Backup Server):
# -a: Archive (giữ nguyên quyền sở hữu, timestamps)
# -v: Verbose
# -z: Nén dữ liệu khi truyền qua mạng
# --delete: Xóa file ở đích nếu nguồn đã xóa (giữ 2 bên giống hệt nhau)

rsync -avz --delete -e ssh ubuntu@192.168.64.2:/var/www/ /Users/macbook/ServerBackups/www/
```

---

## 📜 4. Xây Dựng Script Sao Lưu Tự Động Chuẩn Production

Chúng ta sẽ viết một kịch bản Bash Script toàn diện: Tự động dump database, nén thư mục, xoay vòng xóa file cũ và ghi log.

---

### Bước 4.1: Tạo thư mục lưu trữ và file script
```bash
# Tạo thư mục chứa backup và script
sudo mkdir -p /var/backups/system_backups
sudo mkdir -p /opt/scripts
sudo chown -R ubuntu:ubuntu /var/backups/system_backups /opt/scripts

nano /opt/scripts/backup.sh
```

---

### Bước 4.2: Dán nội dung Bash Script chuẩn

```bash
#!/bin/bash

# ==============================================================================
# SCRIPT SAO LƯU TỰ ĐỘNG CHO UBUNTU SERVER
# ==============================================================================

# 1. Khai báo các biến thiết lập
DATE=$(date +%Y-%m-%d_%H%M%S)
BACKUP_DIR="/var/backups/system_backups"
LOG_FILE="/var/log/server_backup.log"
RETENTION_DAYS=7 # Chỉ giữ lại các bản backup trong 7 ngày gần nhất

# Thư mục và Database cần backup
APP_DIR="/home/ubuntu/apps"
DB_NAME="nestjs_db"
DB_USER="devuser"

# Tạo timestamp ghi log
log() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] $1" | tee -a "$LOG_FILE"
}

log "=================================================="
log "🚀 BẮT ĐẦU QUY TRÌNH SAO LƯU HỆ THỐNG: $DATE"

# Tạo thư mục tạm cho bản backup hôm nay
CURRENT_BACKUP="$BACKUP_DIR/backup_$DATE"
mkdir -p "$CURRENT_BACKUP"

# 2. BƯỚC 1: SAO LƯU CƠ SỞ DỮ LIỆU POSTGRESQL
log "📦 Đang sao lưu Database: $DB_NAME..."
# (Dùng lệnh Docker exec nếu chạy qua Docker, hoặc pg_dump trực tiếp)
if command -v docker &> /dev/null && docker ps | grep -q production_postgres; then
    docker exec -t production_postgres pg_dump -U $DB_USER $DB_NAME > "$CURRENT_BACKUP/database_$DB_NAME.sql"
else
    pg_dump -U $DB_USER -h 127.0.0.1 $DB_NAME > "$CURRENT_BACKUP/database_$DB_NAME.sql" 2>/dev/null || true
fi

# 3. BƯỚC 2: SAO LƯU MÃ NGUỒN VÀ FILE CẤU HÌNH
log "📁 Đang sao lưu thư mục ứng dụng và file cấu hình Nginx..."
tar -czf "$CURRENT_BACKUP/app_code.tar.gz" -C "$APP_DIR" . 2>/dev/null || true
sudo tar -czf "$CURRENT_BACKUP/nginx_config.tar.gz" -C /etc/nginx . 2>/dev/null || true

# 4. BƯỚC 3: ĐÓNG GÓI TẤT CẢ VÀO 1 FILE DUY NHẤT
FINAL_ARCHIVE="$BACKUP_DIR/FULL_BACKUP_$DATE.tar.gz"
log "🗜️ Đang nén toàn bộ thành: $FINAL_ARCHIVE..."
tar -czf "$FINAL_ARCHIVE" -C "$BACKUP_DIR" "backup_$DATE"

# Xóa thư mục tạm không nén
rm -rf "$CURRENT_BACKUP"

# 5. BƯỚC 4: THIẾT LẬP CHÍNH SÁCH LƯU TRỮ (RETENTION POLICY)
log "🧹 Đang dọn dẹp các bản backup cũ hơn $RETENTION_DAYS ngày..."
find "$BACKUP_DIR" -type f -name "FULL_BACKUP_*.tar.gz" -mtime +$RETENTION_DAYS -exec rm -f {} \;

# Tính dung lượng file backup vừa tạo
BACKUP_SIZE=$(du -sh "$FINAL_ARCHIVE" | awk '{print $1}')
log "✅ SAO LƯU THÀNH CÔNG! Dung lượng file: $BACKUP_SIZE"
log "=================================================="
```

*Lưu file:* Bấm `Ctrl + O` $\rightarrow$ `Enter`, sau đó `Ctrl + X` để thoát.

---

### Bước 4.3: Cấp quyền thực thi và chạy thử nghiệm

```bash
# 1. Cấp quyền thực thi cho script
chmod +x /opt/scripts/backup.sh

# 2. Chạy thử nghiệm trực tiếp bằng tay
/opt/scripts/backup.sh
```

**Kiểm tra kết quả:**
```bash
# Xem file backup vừa sinh ra
ls -lh /var/backups/system_backups/

# Xem log hoạt động
cat /var/log/server_backup.log
```

---

## ⏰ 5. Lập Lịch Tự Động Hóa Với Crontab

**Cron** là tiến trình nền (Daemon) của Linux chuyên thực thi các tác vụ định kỳ theo thời gian biểu.

```text
┌─────────────────────────────────────────────────────────────────────────┐
│                       CÚ PHÁP 5 TRƯỜNG CỦA CRONTAB                      │
├─────────────────────────────────────────────────────────────────────────┤
│   *           *           *           *           *                     │
│   │           │           │           │           │                     │
│   │           │           │           │           └── Ngày trong tuần   │
│   │           │           │           │               (0 - 7, 0 & 7: CN)│
│   │           │           │           └────────────── Tháng (1 - 12)    │
│   │           │           └────────────────────────── Ngày trong tháng  │
│   │           │                                       (1 - 31)          │
│   │           └────────────────────────────────────── Giờ (0 - 23)      │
│   └────────────────────────────────────────────────── Phút (0 - 59)     │
└─────────────────────────────────────────────────────────────────────────┘
```

*Ví dụ kinh điển:*
* `0 2 * * *` $\rightarrow$ Chạy vào đúng **02:00 sáng mỗi ngày** (Thời điểm ít người truy cập nhất).
* `0 0 * * 0` $\rightarrow$ Chạy vào đúng **00:00 đêm Chủ nhật hàng tuần**.
* `*/15 * * * *` $\rightarrow$ Chạy đều đặn **mỗi 15 phút một lần**.

---

### Bước 5.1: Cấu hình Crontab cho hệ thống

Mở bảng điều khiển Crontab của user:
```bash
crontab -e
```
*(Nếu lần đầu mở, hãy bấm số `1` để chọn trình soạn thảo `nano`).*

Thêm dòng sau vào cuối cùng của file:

```cron
# Tự động chạy backup vào lúc 02:00 sáng mỗi ngày và ghi log
0 2 * * * /opt/scripts/backup.sh >> /var/log/cron_backup_debug.log 2>&1
```

*Lưu file:* `Ctrl + O` $\rightarrow$ `Enter`, sau đó `Ctrl + X`.

Kiểm tra danh sách lịch đã nạp:
```bash
crontab -l
```

---

## 🚑 6. Diễn Tập Khôi Phục Thảm Họa (Disaster Recovery Drill)

Hãy cùng giả lập tình huống nguy cấp: **Database bị xóa nhầm hoặc hỏng hóc và tiến hành khôi phục 100%!**

```text
┌─────────────────────────────────────────────────────────────────────────┐
│                      QUY TRÌNH KHÔI PHỤC DỮ LIỆU                        │
└────────────────────────────────────┬────────────────────────────────────┘
                                     │
    1. Phát hiện sự cố ──────────────┴──► Database bị mất / Trống rỗng
                                     │
    2. Giải nén file Backup ─────────┴──► tar -xzvf FULL_BACKUP_*.tar.gz
                                     │
    3. Nạp lại CSDL (Restore) ───────┴──► psql / mysql < database.sql
                                     │
    4. Kiểm tra toàn vẹn (Verify) ───┴──► SELECT COUNT(*) FROM users;
```

### Các bước thực hành khôi phục:

1. **Giả lập xóa dữ liệu:**
   ```bash
   # Đăng nhập vào CSDL và xóa bảng dữ liệu
   sudo -u postgres psql -d nestjs_db -c "DROP TABLE IF EXISTS test;"
   ```
2. **Tìm và giải nén file backup gần nhất:**
   ```bash
   mkdir -p /tmp/recovery
   tar -xzvf /var/backups/system_backups/FULL_BACKUP_*.tar.gz -C /tmp/recovery/
   ```
3. **Nạp lại dữ liệu vào Database:**
   ```bash
   # Tìm file .sql bên trong thư mục vừa giải nén
   cat /tmp/recovery/backup_*/database_nestjs_db.sql | sudo -u postgres psql -d nestjs_db
   ```
4. **Xác nhận khôi phục thành công:**
   ```bash
   sudo -u postgres psql -d nestjs_db -c "\dt"
   ```
   $\rightarrow$ Toàn bộ các bảng dữ liệu đã quay trở lại nguyên vẹn!

---

## 🚨 7. Xử Lý 3 Sự Cố Thường Gặp Khi Làm Backup

### 🔴 Sự cố 1: Crontab không chạy script
* **Nguyên nhân phổ biến nhất:** Crontab chạy trong môi trường tối giản, không có sẵn các biến môi trường và đường dẫn `$PATH` đầy đủ như khi bạn gõ trong Terminal.
* **Cách sửa:** Luôn khai báo `PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin` ở đầu file script hoặc dùng đường dẫn tuyệt đối cho mọi câu lệnh (`/usr/bin/tar`, `/usr/bin/docker`).

---

### 🔴 Sự cố 2: Tràn ổ cứng máy chủ (No space left on device)
* **Nguyên nhân:** Viết script backup nhưng quên không xóa các file backup cũ, sau 1-2 tháng dung lượng backup làm đầy 100% ổ cứng.
* **Cách sửa:** Luôn đưa lệnh `find ... -mtime +7 -delete` vào cuối script để tự động dọn dẹp.

---

## 🧪 8. Bài Thực Hành Lab (10 Bước Triển Khai)

*Hãy thực hiện toàn bộ kịch bản thiết lập hệ thống sao lưu tự động trên Ubuntu Server VM:*

1. Tạo thư mục sao lưu `/var/backups/system_backups` và thư mục `/opt/scripts`.
2. Viết file script sao lưu `/opt/scripts/backup.sh`.
3. Tích hợp lệnh dump Database PostgreSQL và nén mã nguồn ứng dụng.
4. Tích hợp chính sách Retention Policy tự động xóa file cũ hơn 7 ngày với lệnh `find`.
5. Cấp quyền thực thi `chmod +x /opt/scripts/backup.sh`.
6. Chạy thử nghiệm script bằng tay và kiểm tra file `.tar.gz` được sinh ra.
7. Soi nội dung nhật ký log tại `/var/log/server_backup.log`.
8. Thiết lập lịch chạy tự động bằng `crontab -e`.
9. Thử nghiệm diễn tập khôi phục: Giải nén file backup vào `/tmp/recovery/`.
10. Nạp lại Database từ file `.sql` đã backup và kiểm tra tính toàn vẹn dữ liệu.

---

## 📌 9. Bảng Tra Cứu Lệnh Backup Cốt Lõi

| Lệnh | Ý nghĩa & Mục đích sử dụng |
| :--- | :--- |
| `tar -czvf <file.tar.gz> <dir>` | Đóng gói và nén thư mục bằng chuẩn Gzip |
| `tar -xzvf <file.tar.gz> -C <dest>` | Giải nén file nén vào thư mục chỉ định |
| `rsync -avz -e ssh <src> <dest>` | Đồng bộ dữ liệu gia tăng thông minh qua mạng SSH |
| `crontab -e` | Chỉnh sửa lịch trình tác vụ tự động Cron |
| `crontab -l` | Xem danh sách các lịch Cron đang hoạt động |
| `find /dir -type f -mtime +7 -delete` | Tìm và xóa tự động các file cũ hơn 7 ngày |
| `du -sh /var/backups/*` | Kiểm tra dung lượng đĩa của các file backup |
