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

### 🚀 4.4. Nâng Cấp Production: Kịch Bản Sao Lưu Đa Ứng Dụng (Multi-App Backup)

Khi máy chủ Ubuntu vận hành **nhiều ứng dụng đồng thời** (ví dụ: vừa có API NestJS, vừa có Dashboard React, vừa có WordPress/Python Service), mô hình sao lưu gộp 1 file duy nhất bộc lộ nhiều nhược điểm:
1. **Khó phục hồi cục bộ:** Một app hỏng buộc phải giải nén toàn bộ kho lưu trữ khổng lồ của cả server.
2. **Đa dạng công nghệ CSDL:** Mỗi app có thể dùng hệ quản trị khác nhau (PostgreSQL, MySQL/MariaDB, hoặc không dùng database).
3. **Phình to dung lượng ổ cứng:** Nếu không lọc bỏ các thư mục tạm như `node_modules`, `.git`, `.next`, `dist`, dung lượng backup sẽ tăng từ vài chục MB lên hàng chục GB không cần thiết.

#### Cấu trúc lưu trữ độc lập theo từng ứng dụng (Per-App Isolation)

```text
/var/backups/multi_apps/
├── apps/
│   ├── nestjs_api/           ──► [2026-09-05_020000_nestjs_api.tar.gz] (Code + DB Dump)
│   ├── react_admin/          ──► [2026-09-05_020000_react_admin.tar.gz] (Source/Build + Config)
│   └── wordpress_blog/       ──► [2026-09-05_020000_wordpress_blog.tar.gz] (Uploads + MySQL Dump)
└── system_shared/
    └── [2026-09-05_020000_system_configs.tar.gz] (Nginx, Let's Encrypt SSL, Systemd Services)
```

#### File Script: `/opt/scripts/backup-multi-apps.sh`

```bash
sudo nano /opt/scripts/backup-multi-apps.sh
```

Dán toàn bộ nội dung script chuẩn dưới đây:

```bash
#!/usr/bin/env bash
# ==============================================================================
# SCRIPT SAO LƯU ĐA ỨNG DỤNG (MULTI-APP PRODUCTION BACKUP)
# Tự động xuất CSDL, nén mã nguồn lọc rác, phân tách từng app & sao lưu cấu hình hệ thống
# ==============================================================================

set -u  # Dừng và cảnh báo nếu sử dụng biến chưa được định nghĩa

# 1. THIẾT LẬP MÔI TRƯỜNG & ĐƯỜNG DẪN HỆ THỐNG
export PATH="/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"
DATE=$(date +%Y-%m-%d_%H%M%S)
BACKUP_ROOT="/var/backups/multi_apps"
APPS_BACKUP_DIR="$BACKUP_ROOT/apps"
SYSTEM_BACKUP_DIR="$BACKUP_ROOT/system_shared"
TEMP_DIR="/tmp/backup_runner_$DATE"
LOG_FILE="/var/log/multi_app_backup.log"
RETENTION_DAYS=7

# 2. KHAI BÁO DANH SÁCH ỨNG DỤNG CẦN SAO LƯU
# Cú pháp định dạng: "TÊN_APP|ĐƯỜNG_DẪN_SOURCE|LOẠI_DB|TÊN_DB|DB_USER|TÊN_CONTAINER_DOCKER"
# - LOẠI_DB: postgres | mysql | none
# - TÊN_CONTAINER_DOCKER: Để trống nếu chạy trực tiếp trên máy chủ; Điền tên container nếu chạy Docker
APPS=(
    "nestjs_api|/var/www/nestjs-api|postgres|nest_prod_db|postgres|postgres_db"
    "react_admin|/var/www/react-admin|none|||"
    "wordpress_blog|/var/www/wordpress|mysql|wp_database|root|mysql_db"
)

# 3. HÀM GHI NHẬT KÝ & KHỞI TẠO THƯ MỤC
mkdir -p "$APPS_BACKUP_DIR" "$SYSTEM_BACKUP_DIR" "$TEMP_DIR"

log() {
    local level="$1"
    local message="$2"
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] [$level] $message" | tee -a "$LOG_FILE"
}

log "INFO" "==============================================================="
log "INFO" "🚀 BẮT ĐẦU CHU TRÌNH SAO LƯU ĐA ỨNG DỤNG: $DATE"

SUCCESS_COUNT=0
FAIL_COUNT=0

# 4. HÀM DUMP DATABASE ĐA NĂNG (POSTGRESQL / MYSQL / DOCKER / DIRECT)
dump_database() {
    local db_type="$1"
    local db_name="$2"
    local db_user="$3"
    local container="$4"
    local output_file="$5"

    case "$db_type" in
        postgres)
            if [ -n "$container" ] && docker ps -q -f name="^${container}$" &>/dev/null; then
                docker exec -t "$container" pg_dump -U "$db_user" "$db_name" > "$output_file"
            else
                pg_dump -U "$db_user" -h 127.0.0.1 "$db_name" > "$output_file"
            fi
            ;;
        mysql)
            if [ -n "$container" ] && docker ps -q -f name="^${container}$" &>/dev/null; then
                docker exec -t "$container" mysqldump -u "$db_user" "$db_name" > "$output_file"
            else
                mysqldump -u "$db_user" "$db_name" > "$output_file"
            fi
            ;;
        none)
            return 0
            ;;
        *)
            log "WARN" "Loại CSDL '$db_type' không hỗ trợ. Bỏ qua backup DB."
            return 1
            ;;
    esac
}

# 5. TIẾN TRÌNH XỬ LÝ TỪNG ỨNG DỤNG (CÔ LẬP LỖI - FAULT ISOLATION)
for app_entry in "${APPS[@]}"; do
    IFS='|' read -r APP_NAME APP_PATH DB_TYPE DB_NAME DB_USER DOCKER_CONTAINER <<< "$app_entry"

    log "INFO" "---------------------------------------------------------------"
    log "INFO" "📦 [App: $APP_NAME] Bắt đầu xử lý..."

    APP_TEMP="$TEMP_DIR/$APP_NAME"
    APP_TARGET_DIR="$APPS_BACKUP_DIR/$APP_NAME"
    mkdir -p "$APP_TEMP" "$APP_TARGET_DIR"

    # Bước 5.1: Sao lưu Cơ sở dữ liệu tương ứng
    if [ "$DB_TYPE" != "none" ]; then
        log "INFO" "  - Exporting Database ($DB_TYPE: $DB_NAME)..."
        DB_OUT="$APP_TEMP/database_${DB_NAME}.sql"
        if ! dump_database "$DB_TYPE" "$DB_NAME" "$DB_USER" "$DOCKER_CONTAINER" "$DB_OUT" 2>/dev/null; then
            log "ERROR" "  ❌ Lỗi khi export database $DB_NAME của app $APP_NAME!"
        fi
    fi

    # Bước 5.2: Đóng gói Source Code & Media (Loại bỏ triệt để thư mục rác)
    if [ -d "$APP_PATH" ]; then
        log "INFO" "  - Đóng gói mã nguồn từ: $APP_PATH (đã loại trừ node_modules, cache)"
        tar -czf "$APP_TEMP/source_code.tar.gz" \
            --exclude="node_modules" \
            --exclude=".git" \
            --exclude=".next" \
            --exclude="dist" \
            --exclude="build" \
            --exclude="cache" \
            --exclude="*.log" \
            -C "$APP_PATH" . 2>/dev/null || true
    else
        log "WARN" "  ⚠️ Thư mục mã nguồn $APP_PATH không tồn tại!"
    fi

    # Bước 5.3: Nén gói hoàn chỉnh của riêng ứng dụng này
    APP_FINAL_ARCHIVE="$APP_TARGET_DIR/${DATE}_${APP_NAME}.tar.gz"
    if tar -czf "$APP_FINAL_ARCHIVE" -C "$TEMP_DIR" "$APP_NAME"; then
        ARCHIVE_SIZE=$(du -sh "$APP_FINAL_ARCHIVE" | awk '{print $1}')
        log "INFO" "  ✅ [App: $APP_NAME] Thành công: $(basename "$APP_FINAL_ARCHIVE") ($ARCHIVE_SIZE)"
        ((SUCCESS_COUNT++))
    else
        log "ERROR" "  ❌ Thất bại khi nén gói backup của app $APP_NAME!"
        ((FAIL_COUNT++))
    fi

    # Bước 5.4: Dọn dẹp bản backup cũ của riêng app này (Retention Policy)
    find "$APP_TARGET_DIR" -type f -name "*_${APP_NAME}.tar.gz" -mtime +$RETENTION_DAYS -exec rm -f {} \;
done

# 6. SAO LƯU TOÀN BỘ CẤU HÌNH DÙNG CHUNG HỆ THỐNG
log "INFO" "---------------------------------------------------------------"
log "INFO" "⚙️ Đang sao lưu cấu hình dùng chung (Nginx, Let's Encrypt SSL, Systemd)..."

SYS_TEMP="$TEMP_DIR/system_shared"
mkdir -p "$SYS_TEMP"

# Nginx vhosts & configs
[ -d "/etc/nginx" ] && tar -czf "$SYS_TEMP/nginx.tar.gz" -C /etc/nginx . 2>/dev/null || true
# Chứng chỉ SSL Let's Encrypt
[ -d "/etc/letsencrypt" ] && tar -czf "$SYS_TEMP/letsencrypt.tar.gz" -C /etc/letsencrypt . 2>/dev/null || true
# Cấu hình dịch vụ Systemd của các app
tar -czf "$SYS_TEMP/systemd_units.tar.gz" /etc/systemd/system/*.service 2>/dev/null || true

SYS_FINAL_ARCHIVE="$SYSTEM_BACKUP_DIR/${DATE}_system_configs.tar.gz"
tar -czf "$SYS_FINAL_ARCHIVE" -C "$TEMP_DIR" "system_shared" 2>/dev/null || true
find "$SYSTEM_BACKUP_DIR" -type f -name "*_system_configs.tar.gz" -mtime +$RETENTION_DAYS -exec rm -f {} \;

# 7. DỌN DẸP BỘ NHỚ TẠM & TỔNG KẾT
rm -rf "$TEMP_DIR"

TOTAL_USAGE=$(du -sh "$BACKUP_ROOT" | awk '{print $1}')
log "INFO" "==============================================================="
log "INFO" "🏁 HOÀN TẤT SAO LƯU! Thành công: $SUCCESS_COUNT app | Thất bại: $FAIL_COUNT app"
log "INFO" "📊 Tổng dung lượng kho sao lưu: $TOTAL_USAGE"
log "INFO" "==============================================================="
```

#### Cấp quyền và chạy thử nghiệm:
```bash
sudo chmod +x /opt/scripts/backup-multi-apps.sh
sudo /opt/scripts/backup-multi-apps.sh
```

#### Quy trình phục hồi (Restore) độc lập 1 App khi gặp sự cố:
Khi một app (ví dụ `nestjs_api`) gặp trục trặc dữ liệu, bạn chỉ cần khôi phục đúng app đó mà không làm ảnh hưởng các dịch vụ khác:
```bash
# 1. Tạo thư mục tạm giải nén gói backup mới nhất của app
mkdir -p /tmp/restore_nest
tar -xzvf /var/backups/multi_apps/apps/nestjs_api/2026-*_nestjs_api.tar.gz -C /tmp/restore_nest/

# 2. Khôi phục mã nguồn và dữ liệu
tar -xzvf /tmp/restore_nest/nestjs_api/source_code.tar.gz -C /var/www/nestjs-api/

# 3. Khôi phục Database
cat /tmp/restore_nest/nestjs_api/database_nest_prod_db.sql | sudo -u postgres psql -d nest_prod_db

# 4. Khởi động lại service app
sudo systemctl restart nestjs-api
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
# Tùy chọn A: Dành cho máy chủ đơn ứng dụng (Single-App):
0 2 * * * /opt/scripts/backup.sh >> /var/log/cron_backup_debug.log 2>&1

# Tùy chọn B: Dành cho máy chủ đa ứng dụng (Multi-App Production):
0 2 * * * /opt/scripts/backup-multi-apps.sh >> /var/log/cron_backup_debug.log 2>&1
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

### 6.1. Thực Hành Khôi Phục Database (Restore Drill)

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

### 🔍 6.2. Cây Thư Mục Cần Lưu Ý & Những File Cần Soi Khi Kiểm Chứng Backup

Một bản sao lưu chỉ thực sự có giá trị khi nó được **kiểm chứng tính toàn vẹn (Verification)**. Dưới đây là cấu trúc thư mục toàn diện và các file trọng yếu cần giám sát:

#### 1. Cây thư mục tổng thể vòng đời Backup trên máy chủ:

```text
UBUNTU SERVER ROOT (/)
│
├── 📂 /etc/                                  ◄── [NGUỒN CẤU HÌNH CẦN BACKUP]
│   ├── nginx/
│   │   ├── nginx.conf                        ──► Cấu hình chính của Web Server
│   │   └── sites-available/                  ──► Cấu hình VirtualHost / Reverse Proxy từng app
│   ├── letsencrypt/live/                     ──► Chứng chỉ HTTPS/SSL (fullchain.pem, privkey.pem)
│   └── systemd/system/                       ──► File service chạy nền (*.service) của các app
│
├── 📂 /var/www/ (hoặc /home/ubuntu/apps/)     ◄── [NGUỒN MÃ NGUỒN & DỮ LIỆU APP]
│   ├── nestjs-api/
│   │   ├── .env                              ──► ⚠️ SỐNG CÒN: Biến môi trường, Secret Keys, DB Pass
│   │   └── uploads/                          ──► File tài liệu, ảnh người dùng tải lên
│   └── react-admin/
│       └── dist/                             ──► Bản build tĩnh của frontend
│
├── 📂 /opt/scripts/                          ◄── [NƠI ĐẶT SCRIPT TỰ ĐỘNG HÓA]
│   ├── backup.sh                             ──► Script backup đơn ứng dụng
│   └── backup-multi-apps.sh                  ──► Script backup đa ứng dụng
│
├── 📂 /var/log/                              ◄── [NƠI SOI LỖI & THEO DÕI TIẾN TRÌNH]
│   ├── server_backup.log                     ──► Log chạy script đơn lẻ
│   ├── multi_app_backup.log                  ──► Log chạy script đa ứng dụng
│   └── cron_backup_debug.log                 ──► Bắt toàn bộ lỗi Stderr từ tiến trình ngầm Crontab
│
└── 📂 /var/backups/                          ◄── [KHO LƯU TRỮ CÁC BẢN SAO LƯU]
    ├── system_backups/                       ──► Lưu bản backup đơn ứng dụng
    └── multi_apps/                           ──► Lưu bản backup đa ứng dụng
        ├── apps/
        │   ├── nestjs_api/
        │   │   └── 2026-09-05_020000_nestjs_api.tar.gz
        │   └── react_admin/
        └── system_shared/
            └── 2026-09-05_020000_system_configs.tar.gz
```

#### 2. Cấu trúc file bên trong một gói Backup (Sau khi giải nén ra thư mục tạm):

```text
/tmp/restore_test/
└── nestjs_api/
    ├── database_nest_prod_db.sql             ──► File Dump CSDL PostgreSQL/MySQL
    └── source_code.tar.gz                    ──► Mã nguồn + uploads (ĐÃ LOẠI TRỪ node_modules, .git)
        ├── .env                              ──► Bắt buộc phải có trong bản backup
        ├── package.json
        ├── uploads/                          ──► Dữ liệu người dùng tải lên
        └── dist/
```

#### 3. Bảng danh sách các file cốt lõi cần xem khi kiểm chứng:

| STT | File cần kiểm tra | Mục đích kiểm chứng | Dấu hiệu BẤT THƯỜNG cần cảnh giác |
| :---: | :--- | :--- | :--- |
| **1** | `/var/log/multi_app_backup.log`<br>hoặc `server_backup.log` | Xem chu trình backup có chạy đủ các bước và ghi nhận thành công | Xuất hiện log `ERROR`, `Permission denied`, `No space left on device` |
| **2** | `/var/log/cron_backup_debug.log` | Kiểm tra lỗi phát sinh từ môi trường thực thi ngầm của Cron | Xuất hiện lỗi `command not found` (do thiếu `$PATH` trong Cron) |
| **3** | File `*.tar.gz` (Kích thước file) | Xác minh file nén có dữ liệu thực tế hay không | File chỉ có kích thước **dưới 1 KB** $\rightarrow$ 99% lệnh dump DB hoặc nén đã lỗi |
| **4** | `database_*.sql` (File dump CSDL) | Xác nhận cấu trúc bảng và dữ liệu SQL đầy đủ | File rỗng (0 bytes) hoặc thiếu dòng hoàn tất dump ở cuối file |
| **5** | File `.env` (Biến môi trường) | Đảm bảo không bị thất lạc key bí mật, API token | File `.env` bị thiếu do cấu hình exclude nhầm |
| **6** | `system_configs.tar.gz` | Đảm bảo lưu đủ cấu hình để dựng lại server mới | Thiếu thư mục `/etc/nginx` hoặc thư mục `/etc/letsencrypt` |

#### 4. Lệnh kiểm tra nhanh 1 dòng (One-Liner Cheat Sheet):

```bash
# 1. Soi danh sách file bên trong gói nén mà KHÔNG cần giải nén ra đĩa:
tar -ztvf /var/backups/multi_apps/apps/nestjs_api/2026-*_nestjs_api.tar.gz

# 2. Kiểm tra dòng đầu và dòng cuối của file CSDL dump bên trong file nén:
tar -zxvf /var/backups/multi_apps/apps/nestjs_api/2026-*_nestjs_api.tar.gz -O nestjs_api/database_*.sql | head -n 20
tar -zxvf /var/backups/multi_apps/apps/nestjs_api/2026-*_nestjs_api.tar.gz -O nestjs_api/database_*.sql | tail -n 10
# (Lưu ý: PostgreSQL luôn có dòng "-- PostgreSQL database dump complete" ở cuối, MySQL có "-- Dump completed on ...")

# 3. Đếm số lượng bảng CSDL đã được sao lưu thành công:
tar -zxvf /var/backups/multi_apps/apps/nestjs_api/2026-*_nestjs_api.tar.gz -O nestjs_api/database_*.sql | grep -c "CREATE TABLE"

# 4. Soi 30 dòng nhật ký lỗi Cron gần nhất:
tail -n 30 /var/log/cron_backup_debug.log

# 5. Tạo và kiểm tra mã băm SHA256 chống hỏng file (Bit rot):
sha256sum /var/backups/multi_apps/apps/nestjs_api/*.tar.gz > /var/backups/multi_apps/checksums.sha256
sha256sum -c /var/backups/multi_apps/checksums.sha256
```

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
