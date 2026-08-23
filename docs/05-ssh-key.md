# 📘 Phần 05: SSH Key Authentication – Thực Hành Xác Thực Bằng Khóa (90% Hands-on)

> **Motto cốt lõi:**  
> *90% Thực hành – 10% Lý thuyết | Tự tạo lỗi → Quan sát triệu chứng → Tìm nguyên nhân → Sửa lỗi.*  
> Khóa bí mật (Private Key) ở lại trên Mac | Ổ khóa công khai (Public Key) đặt lên Server.

---

## 🎯 1. Mục Tiêu Của Phần Học

Sau khi hoàn thành phần này, bạn sẽ:
1. Tự tay thiết lập luồng **SSH Key Authentication chuẩn Production** từ máy Mac tới Ubuntu Server mà không cần dùng mật khẩu.
2. Nắm vững vai trò và phân biệt chính xác: **Private Key**, **Public Key**, và file **`authorized_keys`**.
3. Thành thạo kỹ thuật cấu hình **`~/.ssh/config`** trên macOS để đăng nhập 1 chạm bằng Alias.
4. Hiểu sâu bản chất cơ chế bảo mật nghiêm ngặt (StrictModes) thông qua bài thực hành **cố tình làm sai phân quyền để quan sát lỗi và tự khắc phục**.
5. Thành thạo kỹ năng đọc log chẩn đoán với **`ssh -v`** và **`journalctl -u ssh`**.

---

## 💡 2. Khái Niệm Cốt Lõi (10% Lý Thuyết Nhanh)

```text
┌─────────────────────────────────────────────────────────────────────────┐
│                        CẶP KHÓA BẤT ĐỐI XỨNG                            │
├────────────────────────────────────┬────────────────────────────────────┤
│  1. PRIVATE KEY (Khóa bí mật)      │  2. PUBLIC KEY (Khóa công khai)    │
│  - Lưu DUY NHẤT trên máy Mac       │  - Đưa lên Ubuntu Server           │
│  - File: ~/.ssh/id_ed25519         │  - File: ~/.ssh/id_ed25519.pub     │
│  - TUYỆT ĐỐI KHÔNG CHIA SẺ         │  - Gắn vào ~/.ssh/authorized_keys  │
│  - Giống như "Chìa khóa"           │  - Giống như "Lỗ tra ổ khóa"       │
└────────────────────────────────────┴────────────────────────────────────┘
```

* **`authorized_keys`:** File danh bạ trên Ubuntu Server chứa danh sách tất cả các Public Key được phép đăng nhập vào tài khoản đó.
* **Quy tắc phân quyền vàng:**
  * Thư mục `~/.ssh`: Phân quyền **`700`** (`drwx------`) – Chỉ chủ sở hữu được vào.
  * File `authorized_keys` & Private Key: Phân quyền **`600`** (`-rw-------`) – Chỉ chủ sở hữu được đọc/sửa.

---

## 🛠️ 3. Thực Hành Từng Bước (90% Hands-on)

---

### Bước 1: Kiểm Tra Trạng Thái SSH Hiện Tại

#### 1.1. Trên máy Mac:
Mở Terminal trên máy Mac và kiểm tra thư mục SSH hiện có:

```bash
ls -la ~/.ssh
```

```text
┌───────────────────────────┬─────────────────────────────────────────────┐
│ Tên file trên Mac         │ Chức năng & Mục đích                        │
├───────────────────────────┼─────────────────────────────────────────────┤
│ known_hosts               │ Lưu danh sách "dấu vân tay" (Host Key) của  │
│                           │ các server bạn từng kết nối (Chống giả mạo) │
│ config                    │ File cấu hình Alias tùy biến (nếu đã tạo)   │
│ id_rsa / id_ed25519       │ Private Key (Khóa bí mật cá nhân của bạn)   │
│ id_rsa.pub / ...pub       │ Public Key (Khóa công khai để chia sẻ)      │
└───────────────────────────┴─────────────────────────────────────────────┘
```

#### 1.2. Trên Ubuntu Server:
Mở một cửa sổ Terminal khác hoặc dùng `multipass shell server` để kiểm tra:

```bash
ls -la ~/.ssh
```
*(Nếu báo `No such file or directory`, đừng lo lắng, hệ thống sẽ tự tạo ở các bước tiếp theo).*

---

### Bước 2: Tạo Cặp Khóa SSH Mới Trên Máy Mac

Thực hiện lệnh sau tại Terminal của **máy Mac**:

```bash
ssh-keygen -t ed25519 -C "admin@macbook"
```

```text
Generating public/private ed25519 key pair.
Enter file in which to save the key (/Users/macbook/.ssh/id_ed25519): [Bấm ENTER để chọn mặc định]
Enter passphrase (empty for no passphrase): [Bấm ENTER để trống hoặc đặt mật khẩu bảo vệ khóa]
Enter same passphrase again: [Bấm ENTER]
```

Kiểm tra lại trên máy Mac:
```bash
ls -l ~/.ssh/id_ed25519*
```

*Kết quả:*
```text
-rw-------  1 macbook  staff  419 Aug 23 17:00 /Users/macbook/.ssh/id_ed25519
-rw-r--r--  1 macbook  staff   98 Aug 23 17:00 /Users/macbook/.ssh/id_ed25519.pub
```

> [!CAUTION]
> **Cảnh báo an toàn số 1:** File `id_ed25519` (không có đuôi `.pub`) là **Private Key**. Tuyệt đối **KHÔNG** copy file này lên bất kỳ server nào, không gửi qua chat và không chia sẻ cho bất kỳ ai!

---

### Bước 3: Đưa Public Key Lên Ubuntu Server

Giả sử địa chỉ IP Ubuntu Server của bạn là `192.168.64.2` và user là `ubuntu`.

#### Cách 3.1: Dùng lệnh tự động `ssh-copy-id` (Khuyến nghị)
Chạy trực tiếp từ Terminal máy Mac:

```bash
ssh-copy-id -i ~/.ssh/id_ed25519.pub ubuntu@192.168.64.2
```
*(Nhập mật khẩu của user `ubuntu` một lần cuối cùng. Lệnh này sẽ tự động tải public key lên và phân quyền chuẩn).*

---

#### Cách 3.2: Thực hiện thủ công (Nếu máy Mac chưa có `ssh-copy-id`)

1. **Trên máy Mac:** In nội dung public key ra màn hình và copy chuỗi ký tự đó:
   ```bash
   cat ~/.ssh/id_ed25519.pub
   ```
   *(Chuỗi có dạng: `ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAA... admin@macbook`)*

2. **Trên Ubuntu Server:** Đăng nhập vào server và thêm key vào danh bạ:
   ```bash
   mkdir -p ~/.ssh
   chmod 700 ~/.ssh
   echo "dán-toàn-bộ-chuỗi-public-key-vào-đây" >> ~/.ssh/authorized_keys
   chmod 600 ~/.ssh/authorized_keys
   ```

3. **Kiểm tra trên Ubuntu Server:**
   ```bash
   cat ~/.ssh/authorized_keys
   ```

---

### Bước 4: Kiểm Tra Phân Quyền Bắt Buộc Trên Ubuntu

Đứng bên trong **Ubuntu Server**, chạy lệnh kiểm tra phân quyền chi tiết:

```bash
ls -ld ~/.ssh
ls -l ~/.ssh/authorized_keys
```

**Kết quả bắt buộc phải chuẩn xác:**
```text
drwx------ 2 ubuntu ubuntu 4096 Aug 23 17:05 /home/ubuntu/.ssh
-rw------- 1 ubuntu ubuntu   98 Aug 23 17:05 /home/ubuntu/.ssh/authorized_keys
```

Nếu quyền chưa đúng, hãy sửa ngay:
```bash
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys
```

---

### Bước 5: Đăng Nhập Thử Nghiệm Bằng SSH Key

Từ Terminal của **máy Mac**, chạy lệnh kết nối:

```bash
ssh ubuntu@192.168.64.2
```

Hoặc chỉ định tường minh file Private Key:
```bash
ssh -i ~/.ssh/id_ed25519 ubuntu@192.168.64.2
```

**Quan sát:**
* Hệ thống **không hề hỏi mật khẩu** và vào thẳng shell của Ubuntu trong tích tắc!
* Kiểm tra xác nhận:
  ```bash
  whoami    # Trả về: ubuntu
  hostname  # Trả về: server
  ```

Thoát ra máy Mac:
```bash
exit
```

---

### Bước 6: Cấu Hình Tối Ưu Với `~/.ssh/config`

Để không phải gõ IP và Username mỗi lần kết nối, hãy tạo Alias trên máy Mac.

Trên **máy Mac**, mở file cấu hình bằng `nano`:
```bash
nano ~/.ssh/config
```

Dán nội dung sau vào file:
```text
Host lab-server
    HostName 192.168.64.2
    User ubuntu
    IdentityFile ~/.ssh/id_ed25519
    Port 22
```

*Lưu file:* Bấm `Ctrl + O` $\rightarrow$ `Enter`, sau đó `Ctrl + X` để thoát.

Phân quyền an toàn cho file config trên Mac:
```bash
chmod 600 ~/.ssh/config
```

#### Trải nghiệm kết nối một chạm:
```bash
ssh lab-server
```
*(Ngay lập tức bạn đã có mặt bên trong Ubuntu Server!)*

Thoát ra máy Mac:
```bash
exit
```

---

## 💥 7. Thực Hành Tạo Lỗi Phân Quyền & Tự Sửa (Break & Fix Lab)

Đây là bài học quan trọng nhất giúp bạn hiểu tại sao `sshd` kiểm soát phân quyền cực kỳ khắt khe (**StrictModes**).

```text
Cố tình làm sai quyền ──► Thử SSH lại ──► Quan sát lỗi ──► Đọc nguyên nhân ──► Tự sửa lại
```

### Bước 7.1: Cố tình phá hỏng phân quyền trên Ubuntu Server
Đăng nhập vào Ubuntu Server (qua `multipass shell server`) và cấp quyền "quá mở" (777):

```bash
chmod 777 ~/.ssh
chmod 777 ~/.ssh/authorized_keys
ls -la ~/.ssh
```

### Bước 7.2: Thử SSH từ máy Mac và quan sát
Quay lại Terminal máy Mac và thử kết nối:

```bash
ssh lab-server
```

**Quan sát triệu chứng lỗi:**
```text
ubuntu@192.168.64.2: Permission denied (publickey).
```
*(Mặc dù khóa hoàn toàn đúng, nhưng server dứt khoát từ chối nhận key và đá bạn ra ngoài!)*

### Bước 7.3: Tìm hiểu nguyên nhân
Bên trong Ubuntu Server, kiểm tra log hệ thống của SSH:
```bash
sudo journalctl -u ssh -n 10 --no-pager
```
*Bạn sẽ thấy dòng cảnh báo bảo mật đắt giá:*
```text
Authentication refused: bad ownership or modes for directory /home/ubuntu/.ssh
```

### Bước 7.4: Sửa lại phân quyền chuẩn
Bên trong Ubuntu Server, sửa lại quyền chuẩn:
```bash
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys
```

### Bước 7.5: Kiểm tra lại từ máy Mac
```bash
ssh lab-server
```
*Kết quả:* Đăng nhập thành công trở lại ngay lập tức!

---

## 🔍 8. Kỹ Năng Debug SSH Key Bằng Verbose Mode (`ssh -v`)

Khi gặp sự cố kết nối, hãy thêm cờ `-v` (Verbose) để "bật kính lúp" soi từng gói tin:

```bash
ssh -v lab-server
```

### Cách đọc log chẩn đoán theo từng tầng:

```text
1. TẦNG MẠNG & CỔNG:
   debug1: Connecting to 192.168.64.2 [192.168.64.2] port 22.
   debug1: Connection established.
   ==> Kết nối TCP thành công!

2. TẦNG NẠP PRIVATE KEY TRÊN MAC:
   debug1: identity file /Users/macbook/.ssh/id_ed25519 type 3
   ==> Client đã tìm thấy và nạp đúng file Private Key trên Mac!

3. TẦNG THƯƠNG LƯỢNG & XÁC THỰC:
   debug1: Authentications that can continue: publickey
   debug1: Offering public key: /Users/macbook/.ssh/id_ed25519 ED25519 SHA256:...
   debug1: Server accepts key: pkalg ssh-ed25519 blen 51
   ==> Server chấp nhận chìa khóa và xác thực thành công!

4. TẦNG MỞ SHELL:
   debug1: Authentication succeeded (publickey).
```

---

## 🧪 9. Bài Lab Thử Thách Cuối Phần (Hands-on Challenge)

*Hãy tự mình mở Terminal và thực hiện kịch bản tạo user chuyên dụng `deploy` độc lập từ đầu đến cuối mà không xem lại lệnh mẫu:*

```text
┌─────────────────────────────────────────────────────────────┐
│                    KỊCH BẢN THỬ THÁCH                       │
├─────────────────────────────────────────────────────────────┤
│ 1. Trên Ubuntu: Tạo user mới tên là `deploy` có quyền sudo. │
│ 2. Trên Mac: Sinh một cặp SSH Key riêng tên:                │
│    `~/.ssh/id_deploy_demo`                                  │
│ 3. Đưa Public Key (`id_deploy_demo.pub`) lên cho user       │
│    `deploy` trên Ubuntu Server.                             │
│ 4. Phân quyền chuẩn xác cho thư mục `/home/deploy/.ssh`.    │
│ 5. Cấu hình file `~/.ssh/config` trên Mac với alias:        │
│    `Host deploy-box`                                        │
│ 6. Đăng nhập thử bằng: `ssh deploy-box` (Không hỏi pass).   │
│ 7. Chạy lệnh xác nhận: `whoami` (phải trả về `deploy`).     │
│ 8. Cố tình đổi quyền `authorized_keys` thành 666 và test lỗi│
│ 9. Khôi phục quyền 600 và hoàn tất bài lab.                 │
└─────────────────────────────────────────────────────────────┘
```

---

## 📋 10. Checklist Thực Hành Nhanh

- [ ] Private Key (`id_ed25519`) nằm trên máy Mac, quyền `600`.
- [ ] Public Key (`id_ed25519.pub`) nằm trên Ubuntu trong `~/.ssh/authorized_keys`.
- [ ] Thư mục `~/.ssh` trên Ubuntu có quyền `700` (`drwx------`).
- [ ] File `~/.ssh/authorized_keys` trên Ubuntu có quyền `600` (`-rw-------`).
- [ ] File `~/.ssh/config` trên Mac có quyền `600` và chứa đúng Alias.
- [ ] Đăng nhập thành công không cần nhập mật khẩu: `ssh <alias>`.

---

## 🧠 11. Câu Hỏi Tự Kiểm Tra Bản Chất (Self-Assessment)

1. *File `id_ed25519` và `id_ed25519.pub` file nào được phép chia sẻ công khai, file nào phải giữ bí mật tuyệt đối?*
2. *Nếu một đồng nghiệp gửi cho bạn file `id_ed25519.pub` của họ, bạn sẽ dán nội dung file đó vào đâu trên Ubuntu Server để họ có thể đăng nhập được?*
3. *Tại sao khi bạn cấp quyền `777` cho `~/.ssh`, dịch vụ SSH lại từ chối cho bạn đăng nhập bằng Key? Cơ chế nào của `sshd` chịu trách nhiệm việc này?*
4. *File `~/.ssh/config` nằm trên máy Mac hay nằm trên máy Ubuntu Server? Nó giải quyết sự bất tiện gì?*
5. *Khi chạy lệnh `ssh -v`, dòng log nào chứng minh rằng máy Mac đã kết nối thành công tới cổng 22 của Server?*
