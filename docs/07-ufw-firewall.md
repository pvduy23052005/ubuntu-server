# 📘 Phần 07: UFW Firewall – Thiết Lập Tường Lửa Bảo Vệ Ubuntu Server

> **Motto cốt lõi:**  
> *Nguyên tắc bảo mật mặc định: "Chặn toàn bộ truy cập vào (Deny Incoming) – Cho phép toàn bộ kết nối ra (Allow Outgoing)".*  
> Quy tắc sống còn: **Luôn mở Port SSH trước khi kích hoạt Firewall (`ufw enable`) để tránh bị khóa ngoài server!**

---

## 🎯 1. Mục Tiêu Của Phần Học

Sau khi hoàn thành phần này, bạn sẽ:
1. **Hiểu bản chất Firewall & UFW:** Biết Tường lửa làm nhiệm vụ gì và tại sao UFW (Uncomplicated Firewall) là công cụ chuẩn mực trên Ubuntu.
2. **Nắm vững nguyên lý "Default Deny":** Thiết lập chính sách bảo mật tối thiểu (Least Privilege) cho mạng.
3. **Thành thạo cấu hình quy tắc (Rules):** Mở/Đóng cổng SSH (22), Web (80, 443), giới hạn IP truy cập hoặc giới hạn dải cổng (Port Range).
4. **Làm chủ kỹ thuật Rate Limiting (`ufw limit`):** Chống tấn công dò quét mật khẩu (Brute-force) trên cổng SSH.
5. **Quản lý quy tắc dễ dàng:** Liệt kê danh sách có đánh số thứ tự (`ufw status numbered`) và xóa quy tắc an toàn.
6. **Xử lý sự cố & Bật Logging:** Đọc nhật ký `/var/log/ufw.log` để phát hiện các truy cập đáng ngờ bị chặn.
7. **Tích hợp Fail2ban tự động hóa:** Kết hợp sức mạnh của UFW với Fail2ban để tự động phát hiện và chặn (Ban) các IP tấn công Brute-force.

---

## 🛡️ 2. UFW Là Gì? Bản Chất Hoạt Động

```text
                                MẠNG INTERNET / MÁY MAC
                                           │
                                           │ Gói tin gửi đến (Incoming Traffic)
                                           ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                           UBUNTU SERVER VM                              │
│                                                                         │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │                 UFW FIREWALL (Bộ lọc cổng mạng)                 │   │
│   │                                                                 │   │
│   │   Port 22/tcp (SSH)      ──► ALLOW   ──► Chuyển tới sshd        │   │
│   │   Port 80/tcp (HTTP)     ──► ALLOW   ──► Chuyển tới Nginx       │   │
│   │   Port 443/tcp (HTTPS)   ──► ALLOW   ──► Chuyển tới Nginx       │   │
│   │   Port 5432 (Postgres)   ──► DENY    ──► CHẶN & DROP GÓI TIN    │   │
│   │   Mọi cổng khác          ──► DENY    ──► CHẶN TOÀN BỘ           │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 2.1. Khái niệm cốt lõi:
* **Firewall (Tường lửa):** Là người bảo vệ đứng ở cửa ngõ mạng của máy chủ, kiểm tra từng gói tin đi vào/đi ra và quyết định: **Cho phép (Allow)** hoặc **Từ chối (Deny/Reject)** dựa trên tập quy tắc được định nghĩa trước.
* **UFW (Uncomplicated Firewall):** Là giao diện dòng lệnh thân thiện được Canonical phát triển nhằm đơn giản hóa việc cấu hình `iptables` / `nftables` phức tạp của nhân Linux.

---

## ⚡ 3. Chu Trình Thiết Lập Tường Lửa Chuẩn (5 Bước)

Truy cập vào Ubuntu Server (`ssh lab-server` hoặc `multipass shell server`) và thực hiện theo đúng thứ tự sau:

---

### Bước 1: Kiểm tra trạng thái hiện tại của UFW

```bash
sudo ufw status verbose
```
*Mặc định trên máy mới, bạn sẽ thấy:* `Status: inactive` (Tường lửa đang tắt).

---

### Bước 2: Thiết lập chính sách bảo vệ mặc định (Default Policies)

Áp dụng nguyên tắc vàng của bảo mật hệ thống:

```bash
# 1. Chặn toàn bộ mọi kết nối từ bên ngoài cố tình đi vào server
sudo ufw default deny incoming

# 2. Cho phép máy chủ tự do gửi kết nối ra ngoài (để update apt, gọi API)
sudo ufw default allow outgoing
```

---

### Bước 3: MỞ CỔNG SSH (Bắt buộc làm trước khi Enable!)

> [!CAUTION]
> **CẢNH BÁO SỐNG CÒN:** Nếu bạn chạy `ufw enable` ở Bước 4 mà quên chưa mở cổng SSH ở bước này, kết nối SSH hiện tại của bạn sẽ **bị ngắt ngay lập tức và bạn vĩnh viễn bị khóa ngoài máy chủ**!

```bash
# Mở cổng SSH chuẩn (Port 22/tcp)
sudo ufw allow 22/tcp

# Hoặc dùng tên dịch vụ
sudo ufw allow ssh

# Nếu bạn đã đổi sang cổng khác (ví dụ: 2222), hãy mở đúng cổng đó:
# sudo ufw allow 2222/tcp
```

---

### Bước 4: Mở thêm các cổng dịch vụ Web cần thiết

```bash
# Mở cổng Web HTTP (Port 80)
sudo ufw allow 80/tcp

# Mở cổng Web bảo mật HTTPS (Port 443)
sudo ufw allow 443/tcp

# Hoặc mở đồng thời cả HTTP và HTTPS qua profile Nginx:
# sudo ufw allow 'Nginx Full'
```

---

### Bước 5: Kích hoạt Tường lửa (Enable UFW)

Sau khi đã mở đầy đủ SSH và các cổng cần thiết, tiến hành kích hoạt:

```bash
sudo ufw enable
```
*Hệ thống sẽ hỏi xác nhận:* `Command may disrupt existing ssh connections. Proceed with operation (y|n)?`  
$\rightarrow$ Gõ `y` rồi bấm `Enter`.

Kiểm tra lại trạng thái hoạt động:
```bash
sudo ufw status verbose
```

**Output chuẩn mực:**
```text
Status: active
Logging: on (low)
Default: deny (incoming), allow (outgoing), disabled (routed)
New profiles: skip

To                         Action      From
--                         ------      ----
22/tcp                     ALLOW IN    Anywhere                  
80/tcp                     ALLOW IN    Anywhere                  
443/tcp                    ALLOW IN    Anywhere                  
22/tcp (v6)                ALLOW IN    Anywhere (v6)             
80/tcp (v6)                ALLOW IN    Anywhere (v6)             
443/tcp (v6)               ALLOW IN    Anywhere (v6)             
```

---

## 🔧 4. Các Kỹ Thuật Cấu Hình UFW Thực Chiến

### 4.1. Chống Brute-force SSH với `ufw limit`
Thay vì cho phép kết nối thoải mái vào cổng 22, bạn có thể áp dụng luật **Rate Limiting**:
```bash
sudo ufw limit ssh
```
* **Cơ chế hoạt động:** Nếu một địa chỉ IP cố gắng thử kết nối SSH sai **quá 6 lần trong vòng 30 giây**, UFW sẽ tự động chặn IP đó ngay lập tức!

---

### 4.2. Chỉ cho phép một địa chỉ IP cụ thể truy cập (Whitelist IP)
*Ví dụ: Cổng Database PostgreSQL `5432` chỉ cho phép duy nhất máy tính nội bộ của bạn (`192.168.64.1`) kết nối:*

```bash
sudo ufw allow from 192.168.64.1 to any port 5432 proto tcp
```

---

### 4.3. Mở một dải cổng (Port Range)
*Ví dụ: Mở dải cổng từ 3000 đến 3005 cho các microservices:*

```bash
sudo ufw allow 3000:3005/tcp
```

---

### 4.4. Xóa một quy tắc (Delete Rule)

Cách an toàn nhất là xóa theo số thứ tự (Numbered List):

```bash
# 1. Liệt kê quy tắc kèm số thứ tự [ 1], [ 2], ...
sudo ufw status numbered
```

**Output mẫu:**
```text
     To                         Action      From
     --                         ------      ----
[ 1] 22/tcp                     ALLOW IN    Anywhere
[ 2] 80/tcp                     ALLOW IN    Anywhere
[ 3] 443/tcp                    ALLOW IN    Anywhere
```

```bash
# 2. Xóa quy tắc số 2 (cổng 80)
sudo ufw delete 2
```

---

### 4.5. Bật/Tắt và Reset UFW

```bash
# Tạm dừng tường lửa (không làm mất các quy tắc đã tạo)
sudo ufw disable

# Nạp lại cấu hình sau khi sửa đổi
sudo ufw reload

# Khôi phục tường lửa về trạng thái trắng ban đầu (Xóa toàn bộ rule)
sudo ufw reset
```

---

## 🔍 5. Xem Nhật Ký Tường Lửa (UFW Logging)

Khi nghi ngờ có ai đó quét cổng hoặc kết nối của bạn bị chặn nhầm, hãy kiểm tra nhật ký tường lửa.

> [!NOTE]
> **Lưu ý quan trọng trên Ubuntu mới (22.04 / 24.04 LTS trở lên):**  
> Mặc định hệ thống sử dụng **`systemd-journald`** để ghi nhận log trực tiếp từ Kernel thay vì dùng `rsyslog` để ghi ra file riêng `/var/log/ufw.log`. Do đó, cách xem log UFW chuẩn xác và nhanh nhất là dùng lệnh **`journalctl`**.

### 5.1. Bật tính năng ghi log của UFW
```bash
# Bật ghi log (mức độ cơ bản: on / low)
sudo ufw logging on

# (Tùy chọn) Bật ghi log chi tiết hơn nếu cần debug sâu
# sudo ufw logging medium
```

---

### 5.2. Các câu lệnh xem log UFW chuẩn nhất hiện nay

```bash
# 1. Xem 20 dòng log UFW gần nhất từ Kernel
sudo journalctl -k -g "\[UFW" -n 20

# 2. Theo dõi log UFW trực tiếp theo thời gian thực (Live Stream - Bấm Ctrl+C để thoát)
sudo journalctl -k -f -g "\[UFW"

# 3. Xem log UFW trong vòng 1 giờ qua
sudo journalctl -k -g "\[UFW" --since "1 hour ago"
```

*(Mẹo: Nếu bạn vẫn muốn ghi log ra file truyền thống `/var/log/ufw.log`, bạn chỉ cần cài thêm rsyslog: `sudo apt install -y rsyslog && sudo systemctl restart rsyslog`).*

---

### 5.3. Phân tích chi tiết một dòng log UFW bị chặn

```text
[UFW BLOCK] IN=enp0s1 OUT= MAC=52:54:00:... SRC=192.168.64.50 DST=192.168.64.2 PROTO=TCP SPT=54120 DPT=3306 ...
```

* **`[UFW BLOCK]`:** Gói tin đã bị Tường lửa từ chối và chặn lại.
* **`IN=enp0s1`:** Card mạng nhận gói tin này.
* **`SRC=192.168.64.50`:** Địa chỉ IP của máy gửi gói tin đến (Source IP).
* **`DST=192.168.64.2`:** Địa chỉ IP đích (chính là máy chủ Ubuntu của bạn).
* **`PROTO=TCP`:** Giao thức truyền tải (TCP hoặc UDP).
* **`SPT=54120`:** Cổng gửi từ máy client (Source Port).
* **`DPT=3306`:** Cổng đích trên server bị cố tình truy cập (Destination Port - ở đây là cổng MySQL 3306 chưa được mở).

---

## 🛑 6. Tích Hợp Fail2ban Cùng UFW – Tự Động Chặn IP Tấn Công

```text
┌─────────────────────────────────────────────────────────────────────────┐
│                   MÔ HÌNH PHỐI HỢP GIỮA UFW VÀ FAIL2BAN                 │
└────────────────────────────────────┬────────────────────────────────────┘
                                     │
                                     ▼
                      Kẻ tấn công (Dò quét mật khẩu SSH)
                                     │ (Thử sai nhiều lần)
                                     ▼
                         Ubuntu SSH Daemon (sshd)
                                     │ (Ghi nhận lỗi đăng nhập)
                                     ▼
                         Nhật ký: /var/log/auth.log
                                     │
                                     ▼
                         Fail2ban Daemon (Quét log)
                                     │ (Phát hiện thử sai 5 lần / 10 phút)
                                     ▼
                      Fail2ban kích hoạt lệnh UFW
                                     │
                                     ▼
                         UFW FIREWALL (Tự động BAN IP)
              (Chặn hoàn toàn IP của kẻ tấn công trong 1 giờ)
```

### 6.1. Tại sao cần kết hợp UFW và Fail2ban?
* **UFW (Tường lửa tĩnh):** Chỉ có nhiệm vụ mở hoặc đóng các cổng (như mở cổng 22 cho SSH, mở cổng 80 cho Web). UFW không tự phân biệt được ai là người dùng thật, ai là hacker đang cố tình dò mật khẩu.
* **Fail2ban (Hệ thống ngăn chặn xâm nhập động - IPS):** Đóng vai trò là "mắt thần" liên tục soi các file log hệ thống (`/var/log/auth.log`). Khi phát hiện một địa chỉ IP có hành vi bất thường (như đăng nhập sai quá 5 lần), Fail2ban sẽ **tự động yêu cầu UFW tạo quy tắc chặn ngay lập tức địa chỉ IP đó** trong một khoảng thời gian xác định (ví dụ: 1 giờ hoặc 24 giờ).

---

### 6.2. Cài Đặt & Cấu Hình Fail2ban Sử Dụng UFW

#### Bước 1: Cài đặt Fail2ban qua APT
```bash
sudo apt update
sudo apt install -y fail2ban
```

#### Bước 2: Tạo file cấu hình tùy biến `jail.local`
> [!NOTE]
> Không nên chỉnh sửa trực tiếp file `/etc/fail2ban/jail.conf` vì nó sẽ bị ghi đè khi cập nhật phần mềm. Hãy tạo file `/etc/fail2ban/jail.local` để ghi đè cấu hình an toàn:

```bash
sudo nano /etc/fail2ban/jail.local
```

Dán nội dung cấu hình chuẩn sau vào file:

```ini
[DEFAULT]
# Thời gian cấm IP (1h = 1 giờ, 1d = 1 ngày)
bantime = 1h

# Khoảng thời gian theo dõi (10 phút)
findtime = 10m

# Số lần thử sai tối đa trước khi bị cấm
maxretry = 5

# Chỉ định Fail2ban sử dụng UFW để thực thi chặn IP
banaction = ufw

[sshd]
# Kích hoạt bảo vệ cho dịch vụ SSH
enabled = true
port = ssh
mode = normal
```

*Lưu file:* Bấm `Ctrl + O` $\rightarrow$ `Enter`, sau đó `Ctrl + X` để thoát `nano`.

#### Bước 3: Khởi động và kích hoạt Fail2ban
```bash
sudo systemctl enable --now fail2ban
sudo systemctl restart fail2ban
```

---

### 6.3. Quản Lý & Kiểm Tra IP Bị Chặn Bằng `fail2ban-client`

```bash
# 1. Kiểm tra trạng thái chung của Fail2ban và các Jail đang bật
sudo fail2ban-client status
# Output mẫu: Number of jail: 1, Jail list: sshd

# 2. Xem chi tiết danh sách các IP đang bị CẤM (Banned IP list) của SSH
sudo fail2ban-client status sshd
```

**Output mẫu khi có kẻ tấn công bị cấm:**
```text
Status for the jail: sshd
|- Filter
|  |- Currently failed: 0
|  |- Total failed:     15
|  `- File list:        /var/log/auth.log
`- Actions
   |- Currently banned: 1
   |- Total banned:     1
   `- Banned IP list:   203.0.113.50
```

```bash
# 3. Gỡ lệnh cấm (Unban) cho một địa chỉ IP (nếu lỡ tay thử sai mật khẩu chính mình)
sudo fail2ban-client set sshd unbanip 203.0.113.50

# 4. Cấm thủ công một địa chỉ IP đáng ngờ ngay lập tức
sudo fail2ban-client set sshd banip 198.51.100.25
```

---

## 🧪 7. Bài Thực Hành Lab Toàn Diện (12 Bước)

*Hãy thực hiện toàn bộ kịch bản thiết lập an ninh mạng phối hợp UFW + Fail2ban trên Ubuntu Server VM:*

1. Đăng nhập vào Ubuntu Server và kiểm tra trạng thái ban đầu: `sudo ufw status`.
2. Thiết lập chính sách: `default deny incoming` và `default allow outgoing`.
3. Mở quy tắc rate limit cho SSH: `sudo ufw limit 22/tcp`.
4. Mở cổng web: `sudo ufw allow 80/tcp` và `sudo ufw allow 443/tcp`.
5. Kích hoạt tường lửa: `sudo ufw enable`.
6. Mở một terminal mới trên máy Mac và thử kết nối lại: `ssh lab-server` *(đảm bảo kết nối vẫn mượt mà)*.
7. Cài đặt Fail2ban: `sudo apt install -y fail2ban`.
8. Tạo file `/etc/fail2ban/jail.local` cấu hình `banaction = ufw` và kích hoạt jail `[sshd]`.
9. Khởi động lại dịch vụ: `sudo systemctl restart fail2ban`.
10. Kiểm tra trạng thái bảo vệ: `sudo fail2ban-client status sshd`.
11. Thử nghiệm cấm một IP giả lập: `sudo fail2ban-client set sshd banip 192.0.2.1`.
12. Kiểm tra danh sách bị cấm và gỡ lệnh cấm: `sudo fail2ban-client set sshd unbanip 192.0.2.1`.

---

## 📌 8. Bảng Tra Cứu Lệnh UFW & Fail2ban Cốt Lõi

### Lệnh UFW:
| Lệnh | Ý nghĩa |
| :--- | :--- |
| `sudo ufw status verbose` | Xem trạng thái chi tiết và toàn bộ rule |
| `sudo ufw default deny incoming` | Chặn toàn bộ kết nối đi vào |
| `sudo ufw default allow outgoing` | Cho phép toàn bộ kết nối đi ra |
| `sudo ufw allow <port>/tcp` | Mở một cổng TCP cụ thể |
| `sudo ufw limit ssh` | Bật chống brute-force cho cổng SSH |
| `sudo ufw allow from <IP> to any port <port>` | Chỉ mở cổng cho 1 địa chỉ IP |
| `sudo ufw status numbered` | Liệt kê rule có đánh số thứ tự |
| `sudo ufw delete <number>` | Xóa rule theo số thứ tự |
| `sudo ufw enable` / `disable` | Bật / Tắt tường lửa |
| `sudo ufw reload` | Nạp lại cấu hình tường lửa |

### Lệnh Fail2ban:
| Lệnh | Ý nghĩa |
| :--- | :--- |
| `sudo fail2ban-client status` | Xem các Jail đang hoạt động |
| `sudo fail2ban-client status sshd` | Xem danh sách IP đang bị cấm của SSH |
| `sudo fail2ban-client set sshd banip <IP>` | Cấm thủ công 1 địa chỉ IP |
| `sudo fail2ban-client set sshd unbanip <IP>` | Mở khóa (gỡ cấm) cho 1 địa chỉ IP |
| `sudo systemctl restart fail2ban` | Khởi động lại dịch vụ Fail2ban |

