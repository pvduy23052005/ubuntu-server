# 📘 Phần 06: Networking – Quản Trị Mạng & Cấu Hình Netplan Trên Ubuntu Server

> **Motto cốt lõi:**  
> *Hiểu luồng dữ liệu gói tin > Ghi nhớ lệnh mạng | Sửa mạng an toàn với `netplan try` > Tự cô lập mình khỏi server.*  
> Chu trình chuẩn: **Khái niệm → Bản chất hoạt động → Thực hành → Quan sát output → Debug sự cố.**

---

## 🎯 1. Mục Tiêu Của Phần Học

Một máy chủ không thể hoạt động độc lập; nó chỉ phát huy giá trị khi được kết nối vào mạng để phục vụ người dùng. Quản trị mạng là kỹ năng nền tảng sống còn trước khi bạn dựng Web Server (Nginx), Database (PostgreSQL) hay Container (Docker).

Sau khi hoàn thành bài học này, bạn sẽ:
1. **Nắm vững 5 khái niệm cốt lõi:** Network Interface, IP/CIDR Subnet, Default Gateway, DNS Resolver và Bảng định tuyến (Routing Table).
2. **Hiểu rõ luồng đi của gói tin:** Từ ứng dụng $\rightarrow$ Card mạng $\rightarrow$ Router/Gateway $\rightarrow$ DNS $\rightarrow$ Internet.
3. **Thành thạo bộ công cụ chẩn đoán mạng hiện đại:** `ip addr`, `ip route`, `ping`, `curl`, `ss -tulpn` và `resolvectl`.
4. **Làm chủ Netplan:** Công cụ cấu hình mạng YAML chuẩn của Ubuntu Server.
5. **Phân biệt & tự cấu hình:** Cấu hình nhận IP động (**DHCP**) và gán IP tĩnh (**Static IP**).
6. **Nắm vững quy tắc an toàn:** Biết cách dùng `sudo netplan try` để tự động khôi phục cấu hình (Rollback) nếu xảy ra lỗi làm mất kết nối SSH.
7. **Xử lý sự cố mạng (Troubleshooting):** Tự tin khoanh vùng lỗi nằm ở tầng IP, Routing, DNS hay Tường lửa.

---

## 🌐 2. Hệ Thống Khái Niệm Mạng Cốt Lõi

```text
┌─────────────────────────────────────────────────────────────────────────┐
│                      MÔ HÌNH KẾT NỐI MẠNG CỦA SERVER                    │
└────────────────────────────────────┬────────────────────────────────────┘
                                     │
           ┌─────────────────────────┼─────────────────────────┐
           ▼                         ▼                         ▼
  ┌───────────────────┐    ┌───────────────────┐    ┌───────────────────┐
  │ Network Interface │    │    IP Address     │    │  Default Gateway  │
  │  (enp0s1, eth0)   │    │  (192.168.64.2)   │    │  (192.168.64.1)   │
  └─────────┬─────────┘    └─────────┬─────────┘    └─────────┬─────────┘
            │                        │                        │
            └────────────────────────┼────────────────────────┘
                                     │
                                     ▼
                           ┌───────────────────┐
                           │   DNS Resolver    │
                           │(8.8.8.8 / 1.1.1.1)│
                           └─────────┬─────────┘
                                     │
                                     ▼
                           ┌───────────────────┐
                           │     INTERNET      │
                           │   (google.com)    │
                           └───────────────────┘
```

### 2.1. Network Interface (Giao diện mạng / Card mạng)
* **Nó là gì?** Là điểm kết nối phần cứng hoặc phần mềm giữa máy chủ và hệ thống mạng.
* **Các loại phổ biến trên Ubuntu:**
  * `lo` (Loopback): Card mạng ảo nội bộ (địa chỉ `127.0.0.1`), dùng cho các tiến trình trên cùng 1 máy tự giao tiếp với nhau mà không cần đi qua dây mạng/Wi-Fi.
  * `eth0` / `enp0s1` / `ens3`: Card mạng Ethernet vật lý hoặc card mạng ảo nối với máy Mac/Router.

---

### 2.2. IP Address & Subnet Mask / CIDR
* **IP Address (Địa chỉ IP):** Là "địa chỉ nhà" duy nhất của máy chủ trong mạng (ví dụ: `192.168.64.2`).
* **Subnet Mask & Ký hiệu CIDR (`/24`):**
  * Định nghĩa độ rộng của mạng nội bộ (máy nào được xem là "hàng xóm cùng xóm").
  * Ký hiệu `/24` (tương đương `255.255.255.0`): 24 bit đầu cố định dải mạng (`192.168.64.x`), 8 bit cuối dành cho các thiết bị (cho phép tối đa 254 máy tính cùng dải mạng nói chuyện trực tiếp với nhau mà không cần qua Gateway).

---

### 2.3. Default Gateway (Cổng mặc định)
* **Nó là gì?** Là "cánh cửa thoát hiểm" (thường là Router hoặc Virtual Switch trên macOS) để máy chủ gửi gói tin ra thế giới bên ngoài khi địa chỉ đích không nằm trong mạng nội bộ.
* **Ví dụ:** Server có IP `192.168.64.2` muốn gửi gói tin đến máy chủ Google `142.250.190.46`. Vì Google không nằm trong mạng `192.168.64.x`, Server sẽ chuyển gói tin tới Default Gateway `192.168.64.1` để Gateway chuyển tiếp tiếp đi.

---

### 2.4. DNS (Domain Name System - Hệ thống phân giải tên miền)
* **Nó là gì?** Là cuốn "danh bạ điện thoại của Internet", giúp dịch tên miền dễ nhớ (như `google.com`) thành địa chỉ IP mà máy tính hiểu được (như `142.250.190.46`).
* **Các DNS Server phổ biến:** `8.8.8.8` / `8.8.4.4` (Google), `1.1.1.1` (Cloudflare).

---

### 2.5. DHCP vs Static IP (IP Động vs IP Tĩnh)
* **DHCP (Dynamic Host Configuration Protocol):** Máy chủ tự động xin cấp phát IP, Gateway, DNS từ Router khi khởi động. (Thuận tiện, nhưng IP có thể bị thay đổi sau khi khởi động lại).
* **Static IP (IP Tĩnh):** Bạn cấu hình cố định IP vĩnh viễn cho máy chủ. (Chuẩn mực bắt buộc cho Production Server để client luôn tìm thấy).

---

### 2.6. Phân biệt `localhost`, `127.0.0.1` và IP của Server
* **`localhost` & `127.0.0.1`:** Đại diện cho chính bản thân cỗ máy đang chạy.
  * Nếu Nginx lắng nghe tại `127.0.0.1:80`: **Chỉ nội bộ bên trong Ubuntu VM mới gọi được**, máy Mac bên ngoài gõ IP cũng không thể truy cập!
  * Nếu Nginx lắng nghe tại `0.0.0.0:80` (Mọi Interface): Máy Mac và các thiết bị ngoài mạng mới có thể truy cập qua địa chỉ IP của VM (`192.168.64.2:80`).

---

## 🛠️ 3. Bộ Lệnh Kiểm Tra & Chẩn Đoán Mạng Thực Tế

Hãy truy cập vào Ubuntu Server VM (`ssh lab-server` hoặc `multipass shell server`) để thực thi các lệnh sau:

---

### 3.1. `ip addr` (hoặc `ip a`) – Kiểm tra Card mạng & IP

```bash
ip -c addr
```
*(Cờ `-c` bật chế độ tô màu output giúp dễ đọc).*

```text
┌─────────────────────────────────────────────────────────────┐
│                      PHÂN TÍCH OUTPUT                       │
└─────────────────────────────────────────────────────────────┘
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 ...
    inet 127.0.0.1/8 scope host lo
2: enp0s1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 ...
    inet 192.168.64.2/24 metric 100 brd 192.168.64.255 scope global enp0s1
```

* **`enp0s1`:** Tên card mạng chính.
* **`state UP`:** Card mạng đang được bật và hoạt động tốt.
* **`inet 192.168.64.2/24`:** Địa chỉ IPv4 của server kèm Subnet Mask `/24`.

---

### 3.2. `ip route` – Kiểm tra Bảng định tuyến & Default Gateway

```bash
ip route
```

**Output mẫu:**
```text
default via 192.168.64.1 dev enp0s1 proto dhcp src 192.168.64.2 metric 100 
192.168.64.0/24 dev enp0s1 proto kernel scope link src 192.168.64.2 metric 100 
```

* **`default via 192.168.64.1`:** Bất kỳ gói tin nào đi ra Internet đều phải đi qua **Default Gateway là `192.168.64.1`** thông qua card mạng `enp0s1`.

---

### 3.3. `resolvectl status` – Kiểm tra máy chủ DNS

Trên Ubuntu Server sử dụng `systemd-resolved`, ta kiểm tra DNS bằng:

```bash
resolvectl status enp0s1
```

**Output mẫu:**
```text
Link 2 (enp0s1)
    Current Scopes: DNS
         Protocols: +DefaultRoute +LLMNR -mDNS -DNSOverTLS DNSSEC=no/unsupported
Current DNS Server: 192.168.64.1
       DNS Servers: 192.168.64.1
```
*Cho biết Server đang sử dụng máy chủ DNS nào để phân giải tên miền.*

---

### 3.4. `ping` – Kiểm tra kết nối tầng Mạng (ICMP)

```bash
# 1. Ping Gateway nội bộ (Kiểm tra kết nối tới máy Mac / Router)
ping -c 3 192.168.64.1

# 2. Ping một IP công khai ngoài Internet (Kiểm tra thông mạng ra ngoài, chưa cần DNS)
ping -c 3 8.8.8.8

# 3. Ping theo tên miền (Kiểm tra xem DNS có phân giải được không)
ping -c 3 google.com
```

> [!TIP]
> **Kỹ thuật bắt bệnh mạng siêu nhanh:**
> * Ping `8.8.8.8` thành công **NHƯNG** ping `google.com` thất bại $\rightarrow$ **100% lỗi do DNS**!
> * Ping `192.168.64.1` thất bại $\rightarrow$ **Lỗi card mạng hoặc mất kết nối nội bộ**!

---

### 3.5. `curl` – Kiểm tra kết nối tầng Ứng dụng (HTTP/HTTPS)

```bash
# Gửi request lấy Header của trang web (-I: Head only)
curl -I https://google.com
```
*Thấy trả về `HTTP/2 200` hoặc `HTTP/1.1 301 Moved Permanently` $\rightarrow$ Kết nối Web ra thế giới hoàn hảo.*

---

### 3.6. `ss -tulpn` – Kiểm tra các cổng đang MỞ và lắng nghe (LISTEN)

```bash
sudo ss -tulpn
```

```text
┌─────────────────────────────────────────────────────────────────────────┐
│                      GIẢI MÃ BẢNG CỔNG MẠNG                             │
├─────────┬──────────────┬────────────────────────┬───────────────────────┤
│ Netid   │ State        │ Local Address:Port     │ Process               │
├─────────┼──────────────┼────────────────────────┼───────────────────────┤
│ tcp     │ LISTEN       │ 0.0.0.0:22             │ users:(("sshd",pid=..))│
│ tcp     │ LISTEN       │ 127.0.0.53:53          │ users:(("systemd-reso"│
└─────────┴──────────────┴────────────────────────┴───────────────────────┘
```
* **`0.0.0.0:22` (sshd):** SSH đang lắng nghe trên mọi card mạng (bên ngoài kết nối vào được).
* **`127.0.0.53:53` (DNS stub):** Dịch vụ DNS nội bộ chỉ phục vụ cho chính máy chủ.

---

## ⚙️ 4. Quản Trị Cấu Hình Mạng Với Netplan

### 4.1. Netplan là gì?
* **Netplan** là công cụ cấu hình mạng mặc định trên Ubuntu (từ bản 18.04 trở lên).
* Thay vì phải viết các file cấu hình phức tạp của `systemd-networkd` hay `NetworkManager`, bạn chỉ cần viết một file cấu hình duy nhất theo định dạng **YAML** đặt trong thư mục: `/etc/netplan/`.

```text
┌────────────────────────────────────────────────────────┐
│               FILE CẤU HÌNH YAML                       │
│             /etc/netplan/*.yaml                        │
└───────────────────────────┬────────────────────────────┘
                            │
                            ▼
┌────────────────────────────────────────────────────────┐
│                        NETPLAN                         │
└───────────────────────────┬────────────────────────────┘
                            │
                            ▼ (Tạo config tương ứng)
┌────────────────────────────────────────────────────────┐
│             BACKEND ENGINE: systemd-networkd           │
│        (Hệ thống trực tiếp điều khiển Card mạng)       │
└────────────────────────────────────────────────────────┘
```

---

### 4.2. Khảo sát cấu hình Netplan hiện tại

Kiểm tra thư mục Netplan:
```bash
ls /etc/netplan/
```
*(Thường sẽ có file như `50-cloud-init.yaml` hoặc `00-installer-config.yaml`).*

Xem nội dung cấu hình mặc định (DHCP):
```bash
cat /etc/netplan/*.yaml
```

**Nội dung mẫu của cấu hình DHCP:**
```yaml
network:
  version: 2
  ethernets:
    enp0s1:
      dhcp4: true
```

---

### 4.3. Cấu hình Static IP (IP Tĩnh) Chuẩn

> [!CAUTION]
> **Quy tắc thụt lề trong YAML:**
> * YAML sử dụng **khoảng trắng (Spaces)** để định nghĩa cấu trúc phân cấp (thường là 2 hoặc 4 spaces).
> * **TUYỆT ĐỐI KHÔNG dùng phím TAB**. Dùng phím Tab sẽ khiến Netplan báo lỗi cú pháp ngay lập tức!

#### Cấu trúc file cấu hình Static IP (`/etc/netplan/50-cloud-init.yaml`):

```yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    enp0s1:
      dhcp4: false
      addresses:
        - 192.168.64.100/24
      routes:
        - to: default
          via: 192.168.64.1
      nameservers:
        addresses:
          - 8.8.8.8
          - 1.1.1.1
```

```text
┌───────────────────────┬─────────────────────────────────────────────────┐
│ Cú pháp               │ Ý nghĩa                                         │
├───────────────────────┼─────────────────────────────────────────────────┤
│ dhcp4: false          │ Tắt chế độ tự động nhận IP qua DHCP.            │
│ addresses: [IP/CIDR]  │ Gán IP tĩnh và Subnet Mask cố định cho server.  │
│ routes -> via: Gateway│ Chỉ định địa chỉ IP của Default Gateway.        │
│ nameservers -> address│ Chỉ định danh sách DNS Server (Google/Cloudflare│
└───────────────────────┴─────────────────────────────────────────────────┘
```

---

### 4.4. Quy Tắc Vàng Áp Dụng Cấu Hình Mạng An Toàn: `netplan try`

Khi bạn đang kết nối SSH từ máy Mac vào Server, nếu bạn cấu hình sai IP hoặc sai cú pháp YAML rồi chạy lệnh `netplan apply`, bạn sẽ **lập tức bị ngắt kết nối SSH và vĩnh viễn không vào lại được server từ xa nữa**!

$\rightarrow$ **Giải pháp cứu mạng:** Luôn sử dụng lệnh `sudo netplan try`.

```bash
sudo netplan try
```

```text
┌─────────────────────────────────────────────────────────────┐
│                 CƠ CHẾ HOẠT ĐỘNG CỦA NETPLAN TRY            │
└─────────────────────────────────────────────────────────────┘
1. Netplan áp dụng cấu hình mới tạm thời.
2. Bộ đếm ngược 120 giây (Countdown) bắt đầu chạy.
3. Nếu bạn vẫn còn kết nối và bấm phím ENTER để xác nhận ==> Cấu hình được LƯU.
4. Nếu kết nối bị đứt hoặc bạn không bấm gì trong 120s ==> Netplan TỰ ĐỘNG ROLLBACK
   quay về cấu hình mạng cũ!
```

Sau khi `netplan try` thành công, bạn có thể áp dụng vĩnh viễn bằng:
```bash
sudo netplan apply
```

---

## 🧪 5. Bài Thực Hành Mạng Toàn Diện (Networking Lab 10 Bước)

*Hãy mở Terminal và thực hiện lần lượt 10 bài tập chẩn đoán & cấu hình mạng thực tế sau:*

```text
┌─────────────────────────────────────────────────────────────┐
│                     LAB 10 BƯỚC THỰC HÀNH                   │
├─────────────────────────────────────────────────────────────┤
│ 1. Xác định tên chính xác của Card mạng chính (enp0s1).     │
│ 2. Đọc địa chỉ IPv4 hiện tại của VM.                        │
│ 3. Xác định địa chỉ Default Gateway trong bảng routing.     │
│ 4. Kiểm tra máy chủ DNS mà hệ thống đang sử dụng.           │
│ 5. Kiểm tra kết nối Internet bằng ping và curl.             │
│ 6. Kiểm tra xem port 22 có đang LISTEN trên 0.0.0.0 không. │
│ 7. Đọc file cấu hình Netplan hiện tại trong /etc/netplan/.  │
│ 8. Tạo bản sao lưu file cấu hình Netplan (.yaml.backup).    │
│ 9. Thử nghiệm lệnh kiểm tra cú pháp: `netplan generate`.    │
│ 10. Tự tạo lỗi cố tình và thực hành debug.                  │
└─────────────────────────────────────────────────────────────┘
```

---

### Hướng dẫn chi tiết bước 10: Tự tạo lỗi DNS và tự debug

#### Bước 10.1: Cố tình cấu hình sai DNS
Chỉnh sửa file cấu hình Netplan, đặt DNS thành một IP không tồn tại (ví dụ: `192.0.2.1`):
```bash
sudo nano /etc/netplan/50-cloud-init.yaml
```
*(Đổi phần nameservers thành `addresses: [192.0.2.1]` rồi chạy `sudo netplan apply`).*

#### Bước 10.2: Quan sát triệu chứng
Thử ping Google:
```bash
ping -c 3 8.8.8.8       # VẪN THÀNH CÔNG (Vì mạng và Gateway vẫn đúng!)
ping -c 3 google.com   # BÁO LỖI: Temporary failure in name resolution
```

#### Bước 10.3: Debug và sửa chữa
Nhận diện ngay lỗi nằm ở khâu phân giải tên miền (DNS). Sửa lại DNS thành `8.8.8.8`, chạy `sudo netplan apply` và xác nhận ping `google.com` hoạt động trở lại!

---

## 🔍 6. Sơ Đồ Khắc Phục Sự Cố Mạng (Network Troubleshooting Flowchart)

Khi Server của bạn bị mất mạng, hãy kiểm tra theo đúng thứ tự 4 bước sau:

```text
               ┌───────────────────────────────┐
               │    BƯỚC 1: KIỂM TRA IP & UP   │
               │         `ip addr`             │
               └───────────────┬───────────────┘
                               │
                Card mạng có UP và có IP không?
                               │
                      ┌────────┴────────┐
                     CÓ                KHÔNG ──► Kiểm tra dây mạng ảo / DHCP
                      │
               ┌──────▼────────────────────────┐
               │    BƯỚC 2: PING GATEWAY       │
               │   `ping <Default_Gateway>`    │
               └───────────────┬───────────────┘
                               │
                     Gateway có phản hồi không?
                               │
                      ┌────────┴────────┐
                     CÓ                KHÔNG ──► Sai IP Gateway trong Netplan
                      │
               ┌──────▼────────────────────────┐
               │  BƯỚC 3: PING IP INTERNET     │
               │       `ping 8.8.8.8`          │
               └───────────────┬───────────────┘
                               │
                   Ra được Internet bằng IP không?
                               │
                      ┌────────┴────────┐
                     CÓ                KHÔNG ──► Router mất kết nối ra ngoài
                      │
               ┌──────▼────────────────────────┐
               │     BƯỚC 4: PING DOMAIN       │
               │     `ping google.com`         │
               └───────────────┬───────────────┘
                               │
                    Phân giải được Domain không?
                               │
                      ┌────────┴────────┐
                     CÓ                KHÔNG ──► LỖI DNS! (Sửa lại DNS Server)
                      │
             [ MẠNG HOÀN TOÀN TỐT! ]
```

---

## 📌 7. Tóm Tắt Bản Chất Cần Nhớ

1. **`ip addr`** xem Card mạng & IP | **`ip route`** xem Gateway | **`resolvectl`** xem DNS | **`ss -tulpn`** xem cổng dịch vụ.
2. **`localhost` (127.0.0.1)** chỉ phục vụ nội bộ máy đó. Muốn mở cho máy Mac truy cập, dịch vụ phải lắng nghe trên **`0.0.0.0`** hoặc IP của card mạng.
3. **Netplan** sử dụng cú pháp **YAML**: Cấm dùng phím `Tab`, thụt lề bằng khoảng trắng `Space`.
4. **Luôn dùng `sudo netplan try`** khi chỉnh sửa cấu hình mạng từ xa qua SSH để đảm bảo tính năng tự động rollback nếu gặp sự cố.
5. **Công thức khoanh vùng lỗi mạng:** Ping được IP nhưng không ping được Domain $\rightarrow$ Lỗi ở máy chủ DNS!
