# 📘 Phần 10: Systemd – Quản Lý Tiến Trình & Dịch Vụ Nền Tự Động (Process Manager)

> **Motto cốt lõi:**  
> *PID 1 là Cha của mọi tiến trình | Tự động khởi động cùng hệ thống – Tự động hồi sinh khi bị Crash | Luôn dùng đường dẫn tuyệt đối trong Service Unit!*  
> Chu trình chuẩn: **Bản chất PID 1 → Viết File `.service` → `daemon-reload` → Kích hoạt Boot → Kiểm tra tự hồi sinh → Đọc Log `journalctl`.**

---

## 🎯 1. Mục Tiêu Của Phần Học

Sau khi hoàn thành phần này, bạn sẽ:
1. **Hiểu bản chất Systemd & PID 1:** Biết tại sao Systemd là hệ thống khởi tạo (Init System) trung tâm điều khiển toàn bộ tiến trình trên Linux.
2. **So sánh Systemd vs PM2 vs Docker:** Hiểu ưu/nhược điểm và lý do vì sao Systemd là công cụ quản trị dịch vụ cấp hệ điều hành (OS-Level) chuẩn mực nhất.
3. **Thành thạo cấu trúc Unit File (`.service`):** Nắm vững 3 khối cấu hình cốt lõi: `[Unit]`, `[Service]` và `[Install]`.
4. **Đóng gói ứng dụng Backend (Node.js/NestJS) thành Background Service:** Ứng dụng chạy ngầm liên tục, độc lập hoàn toàn với phiên đăng nhập SSH.
5. **Làm chủ khả năng tự phục hồi (Self-Healing / Auto-Restart):** Thiết lập cơ chế tự động khởi chạy lại ứng dụng trong 5 giây nếu code bị lỗi sập app (Crash) hoặc máy chủ bị khởi động lại (Reboot).
6. **Thành thạo quản lý & soi log với `systemctl` và `journalctl`:** Xem trạng thái chi tiết, theo dõi Live Stream Log và lọc log lỗi.
7. **Xử lý các sự cố kinh điển:** Sửa lỗi sai đường dẫn nhị phân (`ExecStart`), lỗi quyền hạn user (`Permission Denied`), và lỗi sập vòng lặp (`CrashLoopBackoff`).

---

## 🧠 2. Bản Chất: Systemd, PID 1 & Vì Sao Cần Quản Lý Tiến Trình?

### 2.1. Systemd là gì? Cây tiến trình của Linux

Khi máy chủ Linux khởi động, nhân Linux (Kernel) sẽ nạp tiến trình đầu tiên mang **PID 1 (Process ID = 1)** mang tên **`systemd`**.

```text
                                LINUX KERNEL
                                     │
                                     ▼
                     ┌───────────────────────────────┐
                     │     SYSTEMD (PID = 1)         │
                     │  "Tiến trình Cha của vạn vật" │
                     └───────────────┬───────────────┘
                                     │
         ┌───────────────────────────┼───────────────────────────┐
         ▼                           ▼                           ▼
   ssh.service                 nginx.service               nestjs-api.service
 (Quản lý SSH Daemon)       (Quản lý Web Server)        (Quản lý Backend App)
```

* Mọi dịch vụ khác (Nginx, SSH, Database, Ứng dụng Backend của bạn) đều là **tiến trình con** do Systemd giám sát và quản lý.
* Nếu tiến trình con bị sập, Systemd lập tức phát hiện thông qua tín hiệu Kernel và tự động khởi động lại tiến trình đó.

---

### 2.2. So sánh: Chạy Thường vs PM2 vs Systemd

```text
┌───────────────────────────┬──────────────────────┬───────────────────────────┐
│ Tiêu chí                  │ PM2                  │ SYSTEMD (Chuẩn OS)        │
├───────────────────────────┼──────────────────────┼───────────────────────────┤
│ Tầng quản lý              │ Ứng dụng (Node.js)   │ Hệ điều hành (OS-Level)   │
│ Bộ nhớ tiêu thụ           │ Tốn thêm ~80MB RAM   │ 0MB (Tích hợp sẵn trong OS)│
│ Khởi động cùng OS         │ Cần cấu hình startup │ Mặc định (`systemctl enable`)│
│ Giám sát đa ngôn ngữ      │ Chỉ tối ưu JS/TS     │ Bất kỳ ngôn ngữ nào       │
│                           │                      │ (Go, Rust, Python, Node)  │
│ Chuẩn doanh nghiệp        │ Phổ biến ở môi trường│ Chuẩn mực bắt buộc cho    │
│                           │ Dev / Node.js        │ mọi SysAdmin & DevOps     │
└───────────────────────────┴──────────────────────┴───────────────────────────┘
```

---

## 📄 3. Cấu Trúc Chuẩn Của Một File `.service` (Unit File)

Các file service tùy biến do người dùng tạo sẽ được đặt tại thư mục:
```text
/etc/systemd/system/<tên-dịch-vụ>.service
```

```text
┌─────────────────────────────────────────────────────────────────────────┐
│                    3 KHỐI CỐT LÕI CỦA MỘT SERVICE FILE                  │
├─────────────────────────────────────────────────────────────────────────┤
│ [Unit]        ──► Mô tả dịch vụ, thứ tự khởi động (Sau Network/Database)│
│ [Service]     ──► Ai chạy? Chạy lệnh gì? Thư mục nào? Restart ra sao?  │
│ [Install]     ──► Dịch vụ kích hoạt ở chế độ nào khi bật máy            │
└─────────────────────────────────────────────────────────────────────────┘
```

---

### Mẫu File Cấu Hình Service Chuẩn Cho NestJS / Node.js

Hãy xem xét file mẫu `/etc/systemd/system/nestjs-api.service`:

```ini
[Unit]
Description=NestJS Production API Service
Documentation=https://docs.nestjs.com
# Chỉ khởi động ứng dụng sau khi hệ thống mạng đã sẵn sàng
After=network.target

[Service]
# Loại tiến trình (simple: tiến trình chạy liên tục ở foreground)
Type=simple

# Chạy dưới user thường (Tuyệt đối KHÔNG chạy bằng root để bảo mật)
User=ubuntu
Group=ubuntu

# Thư mục gốc chứa mã nguồn dự án
WorkingDirectory=/home/ubuntu/apps/nestjs-api

# LỆNH KHỞI CHẠY (BẮT BUỘC DÙNG ĐƯỜNG DẪN TUYỆT ĐỐI CỦA NODE VÀ CODE)
ExecStart=/usr/bin/node /home/ubuntu/apps/nestjs-api/dist/main.js

# CƠ CHẾ TỰ ĐỘNG HỒI SINH:
# Tự khởi động lại nếu tiến trình bị crash hoặc bị kill bất thường
Restart=always
# Đợi 5 giây trước khi khởi động lại (tránh crash-loop làm nghẽn CPU)
RestartSec=5s

# Nạp biến môi trường
Environment=NODE_ENV=production
Environment=PORT=3000
# (Tùy chọn) Nạp toàn bộ biến từ file .env:
EnvironmentFile=/home/ubuntu/apps/nestjs-api/.env

# Chuyển tiếp toàn bộ Console Log ra Journald của hệ thống
StandardOutput=journal
StandardError=journal
SyslogIdentifier=nestjs-api

[Install]
# Cho phép tự động chạy khi máy chủ khởi động ở chế độ đa người dùng
WantedBy=multi-user.target
```

> [!IMPORTANT]
> **Quy tắc sống còn:** Trong khối `[Service]`, lệnh thực thi `ExecStart` **bắt buộc phải sử dụng đường dẫn tuyệt đối** (ví dụ: `/usr/bin/node`, không được viết `node`).  
> Để tìm đường dẫn tuyệt đối của Node.js, gõ: `which node`.

---

## 🛠️ 4. Thực Hành: Đóng Gói NestJS Thành Dịch Vụ Nền Hoàn Chỉnh

Hãy cùng biến ứng dụng NestJS đã tạo ở **Phần 09** thành một dịch vụ chạy ngầm được quản lý 100% bởi Systemd.

---

### Bước 4.1: Xác định đường dẫn tuyệt đối của Node.js và Dự án
Chạy lệnh kiểm tra:
```bash
which node
# Output mẫu: /usr/bin/node

pwd
# Khi đứng trong thư mục app: /home/ubuntu/apps/nestjs-api
```

---

### Bước 4.2: Tạo file Service Unit trong `/etc/systemd/system/`
```bash
sudo nano /etc/systemd/system/nestjs-api.service
```

Dán toàn bộ nội dung cấu hình chuẩn:

```ini
[Unit]
Description=NestJS Backend API Service
After=network.target

[Service]
Type=simple
User=ubuntu
Group=ubuntu
WorkingDirectory=/home/ubuntu/apps/nestjs-api
ExecStart=/usr/bin/node /home/ubuntu/apps/nestjs-api/dist/main.js
Restart=always
RestartSec=5s
Environment=NODE_ENV=production
EnvironmentFile=/home/ubuntu/apps/nestjs-api/.env
StandardOutput=journal
StandardError=journal
SyslogIdentifier=nestjs-api

[Install]
WantedBy=multi-user.target
```

*Lưu file:* Bấm `Ctrl + O` $\rightarrow$ `Enter`, sau đó `Ctrl + X`.

---

### Bước 4.3: Nạp lại cấu hình Systemd (`daemon-reload`)

> [!NOTE]
> Bất cứ khi nào bạn tạo mới hoặc chỉnh sửa bất kỳ file `.service` nào, bạn **bắt buộc phải chạy lệnh `daemon-reload`** để Systemd quét và cập nhật lại cấu hình:

```bash
sudo systemctl daemon-reload
```

---

### Bước 4.4: Bật tự động khởi động cùng hệ thống (Enable) & Khởi chạy (Start)

```bash
# 1. Kích hoạt dịch vụ tự động chạy khi bật máy
sudo systemctl enable nestjs-api

# 2. Khởi động dịch vụ ngay lập tức
sudo systemctl start nestjs-api

# 3. Kiểm tra trạng thái hoạt động của dịch vụ
sudo systemctl status nestjs-api
```

**Quan sát Output chuẩn mực:**
```text
● nestjs-api.service - NestJS Backend API Service
     Loaded: loaded (/etc/systemd/system/nestjs-api.service; enabled; preset: enabled)
     Active: active (running) since Wed 2026-08-26 08:20:15 UTC; 10s ago
   Main PID: 18452 (node)
      Tasks: 11 (limit: 2248)
     Memory: 45.2M
        CPU: 420ms
     CGroup: /system.slice/nestjs-api.service
             └─18452 /usr/bin/node /home/ubuntu/apps/nestjs-api/dist/main.js

Aug 26 08:20:16 ubuntu-server nestjs-api[18452]: [Nest] Starting Nest application...
Aug 26 08:20:17 ubuntu-server nestjs-api[18452]: [Nest] Nest application successfully started
```

* **`Active: active (running)` màu xanh lá cây** $\rightarrow$ Dịch vụ đang chạy ngầm hoàn hảo!
* **`Main PID: 18452`** $\rightarrow$ Mã tiến trình hiện tại của ứng dụng.
* **`Memory: 45.2M`** $\rightarrow$ Lượng RAM thực tế mà ứng dụng đang tiêu thụ.

---

## 🔍 5. Kiểm Tra Khả Năng Tự Phục Hồi (Self-Healing) & Xem Log với `journalctl`

### 5.1. Thí nghiệm 1: Cố tình "Giết" tiến trình để kiểm tra Auto-Restart

Hãy xem Systemd tự động hồi sinh ứng dụng như thế nào khi gặp sự cố:

**Bước 1: Tìm PID của ứng dụng và tiêu diệt cưỡng bức:**
```bash
# Lấy PID hiện tại (ví dụ 18452) và dùng kill -9
sudo pkill -9 -f "dist/main.js"
```

**Bước 2: Kiểm tra lại trạng thái ngay sau 5 giây:**
```bash
sudo systemctl status nestjs-api
```

*Quan sát kết quả kỳ diệu:*  
$\rightarrow$ `Active: active (running)` vẫn hiển thị, nhưng **`Main PID` đã tự động đổi sang một số mới** (ví dụ: `18590`)!  
$\rightarrow$ Systemd đã tự động phát hiện tiến trình bị chết và spawn lại một tiến trình mới đúng sau 5 giây (`RestartSec=5s`).

---

### 5.2. Thí nghiệm 2: Thoát SSH và Khởi động lại Server (Reboot Test)

1. Gõ `exit` để thoát hoàn toàn khỏi SSH.
2. Từ trình duyệt máy Mac, truy cập: `http://192.168.64.2/` (hoặc `https://192.168.64.2/`).
   $\rightarrow$ Trang web **vẫn hoạt động 100%**, không hề bị gián đoạn khi thoát SSH!
3. Vào lại server và thử reboot máy:
   ```bash
   sudo reboot
   ```
4. Đợi 20 giây cho VM khởi động lại, sau đó mở lại trình duyệt:  
   $\rightarrow$ Ứng dụng NestJS và Nginx đã **tự động chạy lên sẵn sàng** mà bạn không cần phải đăng nhập vào gõ bất kỳ lệnh nào!

---

### 5.3. Làm chủ công cụ xem log `journalctl`

Mọi dòng `console.log()` hoặc lỗi của NestJS đều được Systemd gom về nhật ký tập trung của hệ điều hành:

```bash
# 1. Theo dõi log ứng dụng theo thời gian thực (Live Stream - Bấm Ctrl+C để thoát)
sudo journalctl -u nestjs-api -f

# 2. Xem 50 dòng log gần nhất
sudo journalctl -u nestjs-api -n 50 --no-pager

# 3. Xem log từ 30 phút trước đến nay
sudo journalctl -u nestjs-api --since "30 minutes ago"

# 4. Chỉ xem các dòng log báo LỖI (Error/Critical)
sudo journalctl -u nestjs-api -p err
```

---

## 🚨 6. Xử Lý 3 Sự Cố Phổ Biến Khi Dùng Systemd

### 🔴 Lỗi 1: `status=203/EXEC` (Không tìm thấy file thực thi)
* **Triệu chứng:** `Active: failed (Result: exit-code)`, mã lỗi `203/EXEC`.
* **Nguyên nhân:** Đường dẫn trong `ExecStart` bị sai hoặc không phải đường dẫn tuyệt đối.
* **Cách sửa:** Chạy `which node` để lấy chính xác đường dẫn Node.js và kiểm tra lại đường dẫn file `dist/main.js`.

---

### 🔴 Lỗi 2: `status=217/USER` (Không tìm thấy User)
* **Triệu chứng:** Service không chạy được do lỗi `217/USER`.
* **Nguyên nhân:** Khai báo `User=xxx` không tồn tại trên hệ thống.
* **Cách sửa:** Kiểm tra lại user hiện tại bằng lệnh `whoami` và điền đúng vào file `.service`.

---

### 🔴 Lỗi 3: `Unit file changed on disk, run 'systemctl daemon-reload'`
* **Triệu chứng:** Khi chạy `systemctl restart`, hệ thống cảnh báo file cấu hình đã bị sửa đổi.
* **Cách sửa:** Luôn chạy `sudo systemctl daemon-reload` trước khi restart dịch vụ.

---

## 🧪 7. Bài Thực Hành Lab (10 Bước Triển Khai)

*Hãy thực hiện toàn bộ kịch bản thiết lập Systemd Service cho NestJS:*

1. Đăng nhập vào Ubuntu Server VM và xác định đường dẫn: `which node`.
2. Tạo file cấu hình `/etc/systemd/system/nestjs-api.service`.
3. Cấu hình đầy đủ: `User=ubuntu`, `WorkingDirectory`, `ExecStart`, `Restart=always`, `RestartSec=5s`, `EnvironmentFile`.
4. Nạp cấu hình mới: `sudo systemctl daemon-reload`.
5. Kích hoạt tự khởi động khi boot: `sudo systemctl enable nestjs-api`.
6. Khởi chạy dịch vụ: `sudo systemctl start nestjs-api`.
7. Kiểm tra trạng thái: `sudo systemctl status nestjs-api` (xác nhận màu xanh `running`).
8. Mở Live Stream Log bằng `sudo journalctl -u nestjs-api -f` và gửi request từ máy Mac để xem log nhảy thời gian thực.
9. Thử nghiệm giết tiến trình bằng `sudo pkill -9 -f "dist/main.js"` và kiểm tra xem PID có tự động làm mới không.
10. Thử nghiệm khởi động lại máy ảo bằng `sudo reboot` và xác nhận ứng dụng tự động chạy lại hoàn hảo.

---

## 📌 8. Bảng Tra Cứu Lệnh `systemctl` & `journalctl` Cốt Lõi

| Lệnh | Ý nghĩa & Khi nào dùng |
| :--- | :--- |
| `sudo systemctl daemon-reload` | Nạp lại toàn bộ file service (Bắt buộc sau khi sửa file `.service`) |
| `sudo systemctl start <app>` | Khởi động dịch vụ |
| `sudo systemctl stop <app>` | Dừng dịch vụ |
| `sudo systemctl restart <app>` | Khởi động lại dịch vụ |
| `sudo systemctl reload <app>` | Nạp lại cấu hình ứng dụng không ngắt kết nối (với Nginx) |
| `sudo systemctl status <app>` | Xem trạng thái chi tiết, PID, RAM, CPU và log gần nhất |
| `sudo systemctl enable <app>` | Cho phép dịch vụ tự động chạy khi bật máy (Boot) |
| `sudo systemctl disable <app>` | Tắt tính năng tự chạy khi boot |
| `sudo systemctl is-active <app>` | Kiểm tra nhanh dịch vụ có đang chạy không (Trả về active/inactive) |
| `sudo journalctl -u <app> -f` | Theo dõi Live Stream log của dịch vụ theo thời gian thực |
| `sudo journalctl -u <app> -n 50` | Xem 50 dòng log gần nhất |
| `sudo journalctl -u <app> -p err`| Chỉ lọc các dòng log có mức độ cảnh báo lỗi (Error) |
