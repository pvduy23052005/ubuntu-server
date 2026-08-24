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

Khi nghi ngờ có ai đó quét cổng hoặc kết nối của bạn bị chặn nhầm, hãy kiểm tra log:

```bash
# Đảm bảo logging đã được bật
sudo ufw logging on

# Xem 20 dòng log chặn gói tin gần nhất
sudo tail -n 20 /var/log/ufw.log
```

*Phân tích 1 dòng log bị chặn:*
```text
[UFW BLOCK] IN=enp0s1 OUT= MAC=... SRC=192.168.64.50 DST=192.168.64.2 PROTO=TCP SPT=54120 DPT=3306 ...
```
* **`[UFW BLOCK]`:** Gói tin đã bị tường lửa từ chối.
* **`SRC=192.168.64.50`:** Địa chỉ IP của máy gửi đến.
* **`DPT=3306`:** Cổng đích bị cố tình truy cập (cổng MySQL).

---

## 🧪 6. Bài Thực Hành Lab (10 Bước)

*Hãy thực hiện toàn bộ kịch bản thiết lập an ninh mạng trên Ubuntu Server VM:*

1. Đăng nhập vào Ubuntu Server và kiểm tra trạng thái ban đầu: `sudo ufw status`.
2. Thiết lập chính sách: `default deny incoming` và `default allow outgoing`.
3. Mở quy tắc rate limit cho SSH: `sudo ufw limit 22/tcp`.
4. Mở cổng web: `sudo ufw allow 80/tcp` và `sudo ufw allow 443/tcp`.
5. Kích hoạt tường lửa: `sudo ufw enable`.
6. Mở một terminal mới trên máy Mac và thử kết nối lại: `ssh lab-server` *(đảm bảo kết nối vẫn mượt mà)*.
7. Thử mở cổng ứng dụng mẫu 3000: `sudo ufw allow 3000/tcp`.
8. Kiểm tra danh sách đánh số: `sudo ufw status numbered`.
9. Xóa quy tắc cổng 3000 vừa tạo bằng số thứ tự: `sudo ufw delete <số>`.
10. Kiểm tra lại trạng thái cuối cùng với `sudo ufw status verbose`.

---

## 📌 7. Bảng Tra Cứu Lệnh UFW Cốt Lõi

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
