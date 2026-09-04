# 📘 Phần 13: Fail2ban – Hệ Thống Tự Động Chống Tấn Công Dò Quét & Brute-Force

> **Motto cốt lõi:**  
> *UFW là cánh cửa tĩnh – Fail2ban là người gác cổng động | Tự động phát hiện hành vi độc hại từ Log và cấm (Ban) IP ngay lập tức | Luôn thêm IP của chính mình vào `ignoreip` để chống tự khóa ngoài!*  
> Chu trình chuẩn: **Bản chất IPS → Cấu trúc `jail.local` → Bảo vệ SSH & Nginx → Tự Viết Custom Filter → Quản trị `fail2ban-client` → Lab Thực Chiến.**

---

## 🎯 1. Mục Tiêu Của Phần Học

Sau khi hoàn thành phần này, bạn sẽ:
1. **Hiểu bản chất Hệ thống ngăn chặn xâm nhập (IPS - Intrusion Prevention System):** Cách Fail2ban biến các file log thụ động thành công cụ bảo vệ chủ động theo thời gian thực.
2. **Nắm vững tam giác thông số vàng:** `findtime` (Khoảng thời gian theo dõi), `maxretry` (Số lần thử sai tối đa) và `bantime` (Thời gian cấm IP).
3. **Làm chủ cấu trúc cấu hình Fail2ban:** Hiểu nguyên tắc kế thừa cấu hình `jail.conf` $\rightarrow$ `jail.local`, thư mục bộ lọc `filter.d/` và hành động `action.d/`.
4. **Thiết lập bảo vệ toàn diện các dịch vụ trọng yếu:**
   - **SSH:** Chống dò quét mật khẩu và tấn công brute-force tài khoản root.
   - **Nginx Botsearch:** Tự động khóa các bot chuyên quét file nhạy cảm (`.env`, `wp-login.php`, `phpmyadmin`, `.git`).
   - **Nginx HTTP Auth & Bad Request:** Chặn người dùng cố tình spam request lỗi.
5. **Thiết lập danh sách IP an toàn (`ignoreip`):** Đảm bảo IP nội bộ và máy tính cá nhân không bao giờ bị khóa nhầm.
6. **Thành thạo công cụ điều tra & quản trị `fail2ban-client`:** Tra cứu danh sách IP bị cấm, mở khóa (Unban), cấm thủ công (Ban) và kiểm tra bộ lọc bằng `fail2ban-regex`.
7. **Xử lý sự cố tự khóa mình (Self-Lockout):** Biết cách xử lý khẩn cấp khi lỡ tay gõ sai mật khẩu khiến chính mình bị cấm truy cập server.

---

## 🧠 2. Bản Chất Vận Hành Của Fail2ban

```text
┌─────────────────────────────────────────────────────────────────────────┐
│                    CHU TRÌNH 4 BƯỚC TỰ ĐỘNG CỦA FAIL2BAN                │
└────────────────────────────────────┬────────────────────────────────────┘
                                     │
    1. Kẻ tấn công gửi Request ──────┼──► SSH (Port 22) / Nginx (Port 80/443)
       (Thử sai pass / Quét .env)    │
                                     ▼
    2. Dịch vụ ghi log ──────────────┼──► /var/log/auth.log hoặc systemd-journald
                                     │
                                     ▼
    3. Fail2ban Daemon Quét Log ─────┼──► Khớp Regular Expression (failregex)
                                     │    Đếm: Thử sai >= 5 lần trong 10 phút?
                                     │
                                     ▼ (NẾU VƯỢT QUÁ GIỚI HẠN)
    4. Kích hoạt Action (UFW) ───────┴──► Bơm lệnh vào Firewall:
                                          "sudo ufw insert 1 deny from <IP_Attacker>"
                                          (Khóa toàn bộ kết nối của IP này trong 1 giờ!)
```

* **Tại sao UFW một mình là không đủ?** UFW mở cổng 22 để bạn làm việc. Hacker cũng nhìn thấy cổng 22 mở và dùng botnet thử hàng triệu mật khẩu. Fail2ban chính là mắt thần theo dõi để đóng cửa với riêng kẻ xấu.

---

## 📁 3. Cấu Trúc Thư Mục & Nguyên Tắc Cấu Hình

Hãy khám phá thư mục `/etc/fail2ban/`:

```text
/etc/fail2ban/
├── fail2ban.conf       ──► Cấu hình daemon cấp hệ thống (mức log, socket)
├── jail.conf           ──► FILE CẤU HÌNH GỐC (TUYỆT ĐỐI KHÔNG SỬA FILE NÀY!)
├── jail.local          ──► FILE CẤU HÌNH TÙY BIẾN (Tất cả thiết lập của bạn viết ở đây)
├── filter.d/           ──► Chứa các mẫu biểu thức chính quy (Regex) để nhận diện lỗi
└── action.d/           ──► Chứa các tập lệnh thực thi (UFW, iptables, gửi email cảnh báo)
```

> [!IMPORTANT]
> **Quy tắc bất biến:** Không bao giờ chỉnh sửa file `jail.conf` vì khi hệ thống chạy lệnh `apt upgrade fail2ban`, file này sẽ bị ghi đè làm mất toàn bộ cấu hình. Hãy luôn tạo và chỉnh sửa file **`/etc/fail2ban/jail.local`**.

---

## 🛠️ 4. Cài Đặt & Thiết Lập `jail.local` Chuẩn Production

### Bước 4.1: Cài đặt Fail2ban
```bash
sudo apt update
sudo apt install -y fail2ban
```

---

### Bước 4.2: Tạo file cấu hình tối ưu `/etc/fail2ban/jail.local`

```bash
sudo nano /etc/fail2ban/jail.local
```

Dán toàn bộ nội dung cấu hình chuẩn bảo mật cao sau:

```ini
# ==============================================================================
# 1. CẤU HÌNH MẶC ĐỊNH TOÀN CỤC (GLOBAL DEFAULTS)
# ==============================================================================
[DEFAULT]
# Danh sách IP an toàn không bao giờ bị cấm (Localhost + IP máy Mac / subnet mạng ảo)
ignoreip = 127.0.0.1/8 ::1 192.168.252.1 192.168.252.0/24

# Thời gian cấm IP cơ bản lần đầu (1h = 1 giờ, 1d = 1 ngày, 1w = 1 tuần)
bantime  = 1h

# Khoảng thời gian theo dõi vi phạm (10 phút)
findtime = 10m

# Số lần thử sai tối đa trong khoảng findtime trước khi bị cấm
maxretry = 5

# ==============================================================================
# BAN LŨY TIẾN: TÁI PHẠM THÌ TĂNG THỜI GIAN PHẠT LÊN CẤP SỐ NHÂN
# ==============================================================================
# Kích hoạt tính năng phạt tăng dần theo lịch sử vi phạm của IP
bantime.increment = true

# Hệ số nhân thời gian phạt mỗi lần tái phạm
bantime.factor = 2

# Thời gian cấm tối đa có thể áp dụng (4w = 4 tuần / gần 1 tháng)
bantime.maxtime = 4w

# Cơ chế đọc log hiện đại trên Ubuntu (dùng systemd-journald)
backend = systemd

# Hành động thực thi: Sử dụng UFW để chặn IP tại tầng Tường lửa
banaction = ufw

# ==============================================================================
# 2. BẢO VỆ DỊCH VỤ SSH (SSHD JAIL)
# ==============================================================================
[sshd]
enabled = true
port    = ssh
mode    = normal
maxretry = 3
bantime  = 24h

# ==============================================================================
# 3. BẢO VỆ NGINX: CHỐNG BOT DÒ QUÉT FILE NHẠY CẢM (.env, wp-login, phpmyadmin)
# ==============================================================================
[nginx-botsearch]
enabled  = true
port     = http,https
filter   = nginx-botsearch
logpath  = /var/log/nginx/error.log
backend  = auto
maxretry = 2
bantime  = 48h

# ==============================================================================
# 4. BẢO VỆ NGINX: CHỐNG DÒ MẬT KHẨU HTTP BASIC AUTH
# ==============================================================================
[nginx-http-auth]
enabled  = true
port     = http,https
filter   = nginx-http-auth
logpath  = /var/log/nginx/error.log
backend  = auto
maxretry = 3
bantime  = 12h

# ==============================================================================
# 5. BẢO VỆ NGINX: RATE LIMITING (FLOOD / SPAM REQUEST)
# ==============================================================================
[nginx-limit-req]
enabled  = true
port     = http,https
filter   = nginx-limit-req
logpath  = /var/log/nginx/error.log
backend  = auto
maxretry = 5
findtime = 2m
bantime  = 2h
```

*Lưu file:* Bấm `Ctrl + O` $\rightarrow$ `Enter`, sau đó `Ctrl + X` để thoát.

---

### 💡 Giải Thích Chi Tiết: Các Tham Số Này Làm Gì?

```text
┌─────────────────────────────────────────────────────────────────────────┐
│              CƠ CHẾ HOẠT ĐỘNG CỦA BAN LŨY TIẾN (INCREMENTAL BANNING)    │
└────────────────────────────────────┬────────────────────────────────────┘
                                     │
    Kẻ tấn công vi phạm lần 1 ───────┴──► Bị cấm: 1 GIỜ (bantime cơ bản)
                                     │
    Hết 1 giờ ──► Được mở khóa       │
                                     │
    Quay lại tấn công tiếp (Lần 2) ──┴──► Bị cấm: 1h x 2 = 2 GIỜ
                                     │
    Quay lại tấn công tiếp (Lần 3) ──┴──► Bị cấm: 2h x 2 = 4 GIỜ
                                     │
    Quay lại tấn công tiếp (Lần 4) ──┴──► Bị cấm: 4h x 2 = 8 GIỜ
                                     │
                                    ... (Tăng cấp số nhân)
                                     │
    Chạm ngưỡng tối đa (Lần n) ──────┴──► Bị cấm kịch trần: 4 TUẦN (bantime.maxtime)
```

1. **`ignoreip = 127.0.0.1/8 ::1 192.168.252.1 192.168.252.0/24` (Danh sách trắng / Whitelist):**
   - Định nghĩa các địa chỉ IP "miễn nhiễm" với mọi hình phạt. Dù bạn có lỡ tay gõ sai mật khẩu 100 lần từ máy Mac (`192.168.252.1` hoặc toàn bộ dải mạng `192.168.252.0/24`) hay từ các script nội bộ (`127.0.0.1`), Fail2ban cũng **không bao giờ chặn**. Điều này giúp bạn loại bỏ 100% rủi ro tự khóa mình ra khỏi máy chủ.

2. **`bantime = 1h` (Thời gian cấm cơ sở):**
   - Khoảng thời gian một IP bị nhốt ngoài cửa trong **lần vi phạm đầu tiên**. Ở đây đặt là 1 giờ (`1h`).

3. **`findtime = 10m` & `maxretry = 5` (Cửa sổ phát hiện):**
   - Fail2ban theo dõi trong khung thời gian **10 phút** (`findtime = 10m`). Nếu một IP có từ **5 lần** thử sai trở lên (`maxretry = 5`), hệ thống lập tức xác định đây là hành vi dò quét độc hại và thi hành án phạt.

4. **Bộ 3 tham số Ban Lũy Tiến (`bantime.increment`, `bantime.factor`, `bantime.maxtime`):**
   - **Vấn đề trong thực tế:** Các botnet chuyên nghiệp thường được lập trình để "ngủ đông". Sau khi bị cấm 1 tiếng và được thả ra, chúng lại tiếp tục chu kỳ dò mật khẩu mới. Nếu không có ban lũy tiến, server của bạn sẽ liên tục bị quấy rầy.
   - **`bantime.increment = true`:** Kích hoạt tính năng lưu lại "tiền án tiền sự" của từng IP trong cơ sở dữ liệu SQLite của Fail2ban (`/var/lib/fail2ban/fail2ban.sqlite3`).
   - **`bantime.factor = 2`:** Hệ số nhân thời gian phạt. Mỗi lần IP đó tái phạm sau khi vừa hết hạn cấm, thời gian phạt mới sẽ bằng **thời gian phạt lần trước nhân với 2** (1h $\rightarrow$ 2h $\rightarrow$ 4h $\rightarrow$ 8h $\rightarrow$ 16h...).
   - **`bantime.maxtime = 4w` (Thời gian cấm tối đa):** Giới hạn trần hình phạt tối đa là **4 tuần (1 tháng)**. Việc đặt trần này rất quan trọng để tránh cấm vĩnh viễn một địa chỉ IP (phòng trường hợp IP đó là IP động của nhà mạng sau này được cấp phát cho một người dùng hợp lệ khác).

5. **`backend = systemd` vs `backend = auto` (Quy tắc sống còn khi đọc Log):**
   - **Trong khối `[DEFAULT]`:** Ta đặt `backend = systemd` để Fail2ban đọc log trực tiếp từ **`systemd-journald`** của Linux Kernel (chuẩn mặc định trên Ubuntu 22.04 và 24.04 LTS). Dịch vụ SSH (`sshd`) ghi log vào journald nên hoạt động cực kỳ hoàn hảo.
   - **Tại sao các Jail của Nginx PHẢI ghi đè `backend = auto`?**
     - Nginx mặc định ghi nhật ký lỗi ra file vật lý trên đĩa cứng: `/var/log/nginx/error.log` (chứ không ném vào systemd-journald).
     - Nếu không ghi đè `backend = auto` trong các jail của Nginx, Fail2ban sẽ kế thừa `backend = systemd` từ `[DEFAULT]`, dẫn đến việc **bỏ qua hoàn toàn file `/var/log/nginx/error.log`** và cố tìm log trong journald $\rightarrow$ Fail2ban sẽ bị "mù", không bao giờ bắt được bot dò quét web!
     - Khai báo `backend = auto` ép Fail2ban sử dụng cơ chế lắng nghe file (`inotify`) để theo dõi trực tiếp file `/var/log/nginx/error.log` theo thời gian thực.

6. **`banaction = ufw`:**
   - Thay vì dùng các quy tắc `iptables` thô khó quản lý, Fail2ban sẽ tự động thực thi thông qua **UFW** (`sudo ufw insert 1 deny from <IP>`). Nhờ vậy bạn có thể dễ dàng kiểm tra danh sách chặn bằng lệnh `sudo ufw status`.

7. **`[nginx-limit-req]` (Chống Spam / Flood Request ở tầng Tường lửa):**
   - **Nguyên lý hoạt động:** Nginx có module giới hạn tốc độ truy cập (`limit_req_zone`). Khi một client spam quá nhiều request trong thời gian ngắn, Nginx sẽ ghi một dòng cảnh báo vào `/var/log/nginx/error.log`:
     `[error] ... limiting requests, excess: ... by zone ..., client: <IP>`
   - **Sức mạnh khi kết hợp với Fail2ban:** Nếu một IP cố tình spam và **chạm ngưỡng vi phạm 5 lần (`maxretry = 5`) trong vòng 2 phút (`findtime = 2m`)**, Fail2ban sẽ ra lệnh cho UFW **khóa chặt IP đó trong 2 giờ (`bantime = 2h`)**.
   - **Lợi ích tối thượng:** Bình thường khi bị rate limit, Nginx vẫn phải tốn CPU để nhận kết nối và trả về mã lỗi HTTP `503` hoặc `429`. Nhưng khi kết hợp với Fail2ban, kẻ tấn công sẽ bị **chặn đứng ngay tại cổng mạng bởi Kernel/Tường lửa** $\rightarrow$ Tiết kiệm 100% tài nguyên CPU và băng thông cho máy chủ!

> [!TIP]
> **Cách kích hoạt Rate Limiting trong Nginx để Fail2ban bắt log:**  
> Thêm vào file cấu hình Nginx (`/etc/nginx/nginx.conf` hoặc Server Block):
> ```nginx
> # 1. Khai báo vùng nhớ theo dõi IP (tối đa 10 request/giây)
> limit_req_zone $binary_remote_addr zone=mylimit:10m rate=10r/s;
> 
> # 2. Áp dụng vào location API:
> location / {
>     limit_req zone=mylimit burst=20 nodelay;
>     proxy_pass http://127.0.0.1:3000;
> }
> ```

---

### Bước 4.3: Khởi động và kích hoạt dịch vụ

```bash
# 1. Kích hoạt tự khởi động cùng OS
sudo systemctl enable --now fail2ban

# 2. Khởi động lại dịch vụ để nạp cấu hình mới
sudo systemctl restart fail2ban

# 3. Kiểm tra trạng thái
sudo systemctl status fail2ban
```

---

## 🎯 5. Làm Chủ Công Cụ Quản Trị `fail2ban-client`

Đây là công cụ CLI đắc lực giúp SysAdmin điều tra và xử lý sự cố vi phạm an ninh:

### 5.1. Tra cứu trạng thái các Jail đang hoạt động
```bash
sudo fail2ban-client status
```
*Output mẫu:*
```text
Status
|- Number of jail:      3
`- Jail list:           nginx-botsearch, nginx-http-auth, sshd
```

---

### 5.2. Soi chi tiết danh sách IP đang bị cấm của từng dịch vụ
```bash
# Xem tình hình bảo vệ của SSH
sudo fail2ban-client status sshd
```

**Quan sát Output chi tiết:**
```text
Status for the jail: sshd
|- Filter
|  |- Currently failed: 1
|  |- Total failed:     24
|  `- File list:        /var/log/auth.log
`- Actions
   |- Currently banned: 2
   |- Total banned:     5
   `- Banned IP list:   203.0.113.195 198.51.100.42
```
* `Currently banned: 2` $\rightarrow$ Đang có 2 kẻ tấn công bị nhốt ngoài cửa!
* `Banned IP list` $\rightarrow$ Hiển thị chính xác địa chỉ IP của kẻ xấu.

---

### 5.3. Thao tác Cấm (Ban) và Mở Khóa (Unban) bằng tay

```bash
# 1. Mở khóa (Unban) cho 1 địa chỉ IP (khi lỡ tay gõ sai pass)
sudo fail2ban-client set sshd unbanip 203.0.113.195

# 2. Cấm thủ công 1 địa chỉ IP đáng ngờ ngay lập tức
sudo fail2ban-client set sshd banip 198.51.100.99

# 3. Mở khóa TOÀN BỘ tất cả các IP bị cấm trên mọi Jail (Dùng khi khẩn cấp)
sudo fail2ban-client unban --all
```

---

## 🧪 6. Thử Nghiệm Tấn Công & Quan Sát Fail2ban Bắt Sống Kẻ Xấu

Hãy thực hiện một bài test thực tế ngay trên máy Mac:

### Thí nghiệm: Giả lập tấn công dò quét file nhạy cảm trên Nginx

1. Đảm bảo Nginx và Fail2ban đang chạy trên VM.
2. Từ Terminal của **máy Mac**, dùng `curl` gửi 3 request liên tiếp tìm kiếm các file cấm vào IP máy ảo (`192.168.252.3`):
   ```bash
   curl http://192.168.252.3/.env
   curl http://192.168.252.3/wp-login.php
   curl http://192.168.252.3/phpmyadmin
   ```
3. Quay trở lại **Ubuntu Server VM** và kiểm tra trạng thái jail `nginx-botsearch`:
   ```bash
   sudo fail2ban-client status nginx-botsearch
   ```
   $\rightarrow$ Bạn sẽ thấy IP của máy Mac (`192.168.252.1`) đã xuất hiện trong **`Banned IP list`**! *(Lưu ý: Để test thực tế bước này, hãy tạm thời xóa IP máy Mac khỏi dòng `ignoreip` trong `jail.local` rồi reload Fail2ban)*.
4. Từ máy Mac, thử gửi lại bất kỳ request nào:
   ```bash
   curl http://192.168.252.3/
   ```
   $\rightarrow$ Lệnh bị treo hoặc trả về `Connection refused` vì Tường lửa UFW đã chặn triệt để gói tin từ máy Mac!
5. Trên Server, mở khóa cho máy Mac:
   ```bash
   sudo fail2ban-client set nginx-botsearch unbanip 192.168.252.1
   ```
   $\rightarrow$ Kết nối của máy Mac lập tức được khôi phục bình thường.

---

## 🚨 7. Xử Lý Sự Cố Khi Sử Dụng Fail2ban

### 🔴 Sự cố 1: Tự khóa chính mình khỏi SSH (Self-Lockout)
* **Triệu chứng:** Bạn gõ sai mật khẩu SSH nhiều lần và bị mất kết nối hoàn toàn vào server.
* **Cách cứu hộ:**
  - Vì chúng ta đang dùng Multipass trên Mac, bạn có thể truy cập qua bảng điều khiển máy ảo không qua mạng SSH:
    ```bash
    multipass shell server
    ```
  - Sau khi vào được shell, mở khóa ngay lập tức:
    ```bash
    sudo fail2ban-client set sshd unbanip <IP_CỦA_MÁY_MAC>
    ```
  - Thêm IP của bạn vào dòng `ignoreip` trong `/etc/fail2ban/jail.local` để không bao giờ bị khóa lại.

---

### 🔴 Sự cố 2: Kiểm tra biểu thức chính quy (Regex) với `fail2ban-regex`
Khi bạn nghi ngờ một dòng log không được Fail2ban bắt, hãy dùng công cụ kiểm tra:
```bash
sudo fail2ban-regex /var/log/auth.log /etc/fail2ban/filter.d/sshd.conf
```
*Kết quả sẽ hiển thị số dòng log khớp (Matched) và số dòng bị bỏ qua.*

---

## 🧪 8. Bài Thực Hành Lab (10 Bước Triển Khai)

*Hãy thực hiện toàn bộ kịch bản thiết lập hệ thống phòng thủ tự động Fail2ban trên Ubuntu Server VM:*

1. Cài đặt Fail2ban qua APT: `sudo apt install -y fail2ban`.
2. Xác định IP của máy Mac (`192.168.252.1`), IP máy ảo (`192.168.252.3`) và `127.0.0.1`.
3. Tạo file `/etc/fail2ban/jail.local` với cấu hình `ignoreip`, `banaction = ufw`, `backend = systemd`.
4. Kích hoạt jail `[sshd]` và `[nginx-botsearch]`.
5. Khởi động lại dịch vụ: `sudo systemctl restart fail2ban`.
6. Tra cứu danh sách jail đang chạy: `sudo fail2ban-client status`.
7. Xem chi tiết trạng thái jail sshd: `sudo fail2ban-client status sshd`.
8. Thử nghiệm cấm thủ công một IP giả lập: `sudo fail2ban-client set sshd banip 192.0.2.1`.
9. Kiểm tra xem quy tắc UFW có tự động sinh ra không: `sudo ufw status verbose`.
10. Mở khóa cho IP giả lập: `sudo fail2ban-client set sshd unbanip 192.0.2.1` và kiểm tra lại trạng thái.

---

## 📌 9. Bảng Tra Cứu Lệnh Fail2ban Cốt Lõi

| Lệnh | Ý nghĩa & Khi nào dùng |
| :--- | :--- |
| `sudo fail2ban-client status` | Xem tổng quan số lượng Jail và danh sách các Jail đang hoạt động |
| `sudo fail2ban-client status <jail>` | Xem chi tiết số lượng vi phạm và danh sách các IP đang bị CẤM |
| `sudo fail2ban-client set <jail> banip <IP>` | Cấm thủ công một địa chỉ IP độc hại |
| `sudo fail2ban-client set <jail> unbanip <IP>` | Mở khóa (gỡ lệnh cấm) cho một địa chỉ IP |
| `sudo fail2ban-client unban --all` | Mở khóa khẩn cấp cho tất cả các IP trên toàn bộ hệ thống |
| `sudo fail2ban-client reload` | Nạp lại cấu hình sau khi chỉnh sửa file `jail.local` |
| `sudo fail2ban-regex <log> <filter>` | Kiểm tra và debug biểu thức chính quy lọc log |
| `sudo tail -f /var/log/fail2ban.log` | Theo dõi nhật ký hoạt động cấm/mở khóa thời gian thực |
