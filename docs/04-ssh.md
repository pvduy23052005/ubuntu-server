# 📘 Phần 04: SSH Fundamentals – Nền Tảng Giao Thức SSH & Quản Trị Từ Xa

> **Motto cốt lõi:**  
> *Hiểu luồng kết nối và xác thực > Học thuộc lệnh ssh | Tự thiết lập, phân quyền và debug lỗi kết nối thực tế.*  
> Chu trình chuẩn: **Nó là gì? → Bản chất hoạt động → Thực hành → Quan sát → Debug khi lỗi.**

---

## 🎯 1. Mục Tiêu Của Phần Học

Trong thế giới quản trị hạ tầng và triển khai hệ thống (DevOps / Server Administration), **SSH (Secure Shell)** là "cánh cổng duy nhất" để bạn truy cập, điều khiển và vận hành máy chủ Linux từ xa một cách an toàn.

Sau khi hoàn thành bài học này, bạn sẽ:
1. **Hiểu rõ bản chất SSH:** Nắm vững mô hình kiến trúc **SSH Client (macOS)** $\leftrightarrow$ **SSH Server (`sshd` trên Ubuntu)** qua cổng mạng TCP Port `22`.
2. **Giải mã luồng kết nối SSH:** Hiểu tường tận các giai đoạn từ bắt tay TCP, trao đổi Host Key, xác thực danh tính (Authentication) đến phiên làm việc mã hóa (Encrypted Session).
3. **Làm chủ xác thực bằng SSH Key:** Hiểu bản chất cặp khóa bất đối xứng (**Private Key** trên Mac vs **Public Key** trên Server), tại sao phương pháp này an toàn vượt trội so với mật khẩu truyền thống.
4. **Phân quyền chuẩn xác cho thư mục SSH:** Hiểu tại sao `sshd` từ chối hoạt động nếu `~/.ssh` (quyền `700`) và `authorized_keys` (quyền `600`) bị cấp quyền sai.
5. **Cấu hình `~/.ssh/config` chuyên nghiệp:** Thiết lập các bí danh (Alias) trên máy Mac để kết nối server chỉ với 1 câu lệnh ngắn gọn: `ssh my-server`.
6. **Nắm vững file cấu hình máy chủ `/etc/ssh/sshd_config`:** Hiểu các thông số bảo vệ server cốt lõi và nguyên tắc "không bao giờ tự khóa mình ngoài cửa" (Lockout Prevention).
7. **Thành thạo kỹ năng Debug kết nối:** Biết cách đọc log chẩn đoán với `ssh -v` để truy tìm nguồn gốc lỗi (lỗi Mạng, lỗi Port, lỗi Quyền hạn hay lỗi Khóa xác thực).

---

## 🌐 2. SSH Là Gì? Mô Hình SSH Client → SSH Server

### 2.1. Khái niệm cốt lõi

```text
SSH = Secure Shell (Vỏ bọc bảo mật)
```

* **SSH là gì?** Là một giao thức mạng mật mã (Cryptographic Network Protocol) cho phép hai máy tính giao tiếp, truyền nhận dữ liệu và thực thi dòng lệnh từ xa thông qua một kênh truyền được **mã hóa 100%**.
* **SSH giải quyết vấn đề gì?** Trước khi có SSH, người ta dùng giao thức **Telnet** hoặc **Rlogin**. Các giao thức cũ này gửi toàn bộ thông tin (bao gồm cả Username và Password) dưới dạng văn bản thô (Plain Text) qua mạng, khiến bất kỳ ai ở cùng mạng Wi-Fi/LAN đều có thể bắt gói tin (Sniffing) và đánh cắp tài khoản. SSH mã hóa mọi dữ liệu truyền đi, biến thông tin thành chuỗi ký tự vô nghĩa đối với kẻ nghe lén.

---

### 2.2. Mô hình hoạt động Client - Server

```text
┌──────────────────────────────────────────────┐
│             MÁY MAC CỦA BẠN                  │
│  ┌────────────────────────────────────────┐  │
│  │           SSH CLIENT                   │  │
│  │  (Chương trình `ssh` có sẵn trên macOS)│  │
│  └───────────────────┬────────────────────┘  │
└──────────────────────┼───────────────────────┘
                       │
                       │ 1. Khởi tạo kết nối TCP qua Port 22
                       │ 2. Trao đổi khóa & Xác thực danh tính
                       │ 3. Tạo đường hầm mã hóa an toàn (Encrypted Tunnel)
                       │
┌──────────────────────┼───────────────────────┐
│                      ▼                       │
│  ┌────────────────────────────────────────┐  │
│  │           SSH SERVER (`sshd`)          │  │
│  │  (Tiến trình daemon chạy trên Ubuntu)  │  │
│  └───────────────────┬────────────────────┘  │
│                      │ 4. Mở phiên Bash Shell
│                      ▼                       │
│             UBUNTU SERVER VM                 │
└──────────────────────────────────────────────┘
```

* **SSH Client (Máy Mac):** Ứng dụng dòng lệnh `ssh` đóng vai trò là "người gọi", chủ động khởi tạo kết nối tới máy chủ.
* **SSH Server / `sshd` (Ubuntu Server):** Tiến trình nền `sshd` (SSH Daemon) liên tục chạy ngầm trên Ubuntu, mở và lắng nghe các kết nối gửi tới cổng mặc định **Port 22**.
* **TCP (Transmission Control Protocol):** SSH hoạt động trên nền giao thức TCP ở tầng Transport (Lớp 4), đảm bảo các gói tin lệnh được truyền tải đầy đủ, đúng thứ tự và không bị thất lạc.

---

## 🔐 3. Bản Chất Một Kết Nối SSH: Điều Gì Diễn Ra Phía Sau?

Khi bạn gõ lệnh `ssh ubuntu@192.168.64.2` và nhấn `Enter`, 5 bước sau sẽ diễn ra liên hoàn:

```text
┌──────────────┐                                            ┌──────────────┐
│  SSH CLIENT  │                                            │  SSH SERVER  │
│   (macOS)    │                                            │   (Ubuntu)   │
└──────┬───────┘                                            └──────┬───────┘
       │                                                           │
       │ 1. TCP 3-Way Handshake (SYN -> SYN/ACK -> ACK)            │
       ├──────────────────────────────────────────────────────────►│
       │                                                           │
       │ 2. Trao đổi Host Key & Thuật toán mã hóa                  │
       │◄─────────────────────────────────────────────────────────►│
       │ (Client kiểm tra Host Key trong ~/.ssh/known_hosts)       │
       │                                                           │
       │ 3. Thiết lập Kênh mã hóa đối xứng (Session Key)           │
       │◄═════════════════════════════════════════════════════════►│
       │                                                           │
       │ 4. Xác thực người dùng (User Authentication)               │
       │    (Bằng Mật khẩu HOẶC Cặp khóa SSH Key)                 │
       ├──────────────────────────────────────────────────────────►│
       │                                                           │
       │ 5. Cấp quyền truy cập & Mở phiên Bash Shell               │
       │◄═════════════════════════════════════════════════════════►│
       │                                                           │
```

### 3.1. Chi tiết từng giai đoạn:

1. **Giai đoạn 1: Kết nối TCP (Port 22):** Máy Mac kết nối tới IP và Port 22 của Ubuntu qua cơ chế bắt tay 3 bước của TCP.
2. **Giai đoạn 2: Xác minh máy chủ (Host Key Verification):**
   * Server gửi cho Client một "chứng minh thư" gọi là **Host Key**.
   * Client lưu Host Key này vào file `~/.ssh/known_hosts` trên máy Mac.
   * *Mục đích:* Giúp máy Mac đảm bảo rằng nó đang kết nối đúng vào máy chủ thật của bạn, ngăn chặn kẻ xấu giả mạo máy chủ (Chống tấn công *Man-in-the-Middle*).
3. **Giai đoạn 3: Thiết lập kênh truyền mã hóa (Key Exchange):**
   * Client và Server sử dụng thuật toán trao đổi khóa (như Diffie-Hellman) để tự sinh ra một **Khóa phiên (Session Key)** bí mật chung.
   * **Từ thời điểm này, 100% dữ liệu truyền qua lại đều được mã hóa**, kể cả username và password bạn chuẩn bị nhập.
4. **Giai đoạn 4: Xác thực người dùng (Authentication):**
   * Server yêu cầu Client chứng minh danh tính: Bạn là ai và có quyền đăng nhập vào tài khoản này không?
   * Có 2 phương thức chính: **Nhập mật khẩu (Password)** hoặc **Dùng khóa điện tử (SSH Key)**.
5. **Giai đoạn 5: Mở phiên làm việc (Session):**
   * Xác thực thành công $\rightarrow$ Server khởi tạo một tiến trình Bash Shell mới cho user đó và kết nối luồng I/O về màn hình Terminal máy Mac.

> [!NOTE]
> **Phân biệt rạch ròi:**
> * **Encryption (Mã hóa):** Bảo vệ nội dung dữ liệu không bị đọc trộm trên đường truyền mạng.
> * **Authentication (Xác thực):** Kiểm tra xem người kết nối có đúng là chủ nhân của tài khoản hay không.

---

## 🛠️ 4. Thực Hành SSH Cơ Bản Bằng Mật Khẩu (Password Authentication)

### Bước 4.1: Kiểm tra SSH Server trên Ubuntu Server VM

Đăng nhập vào VM bằng Multipass để kiểm tra xem dịch vụ `sshd` đã sẵn sàng chưa:

```bash
multipass shell server
```

Chạy chuỗi lệnh kiểm tra sau bên trong Ubuntu:

```bash
# 1. Kiểm tra trạng thái tiến trình SSH Daemon
sudo systemctl status ssh
```
*Quan sát thấy dòng màu xanh:* `Active: active (running)`.

```bash
# 2. Kiểm tra xem sshd có đang lắng nghe cổng 22 không
sudo ss -tulpn | grep :22
```
*Quan sát thấy:* `LISTEN 0 128 0.0.0.0:22 ... users:(("sshd",pid=...))`

```bash
# 3. Lấy địa chỉ IP của máy ảo
ip addr show enp0s1
```
*(Giả sử địa chỉ IP của VM là `192.168.64.2`).*

```bash
# 4. Đặt mật khẩu cho user ubuntu để có thể test đăng nhập bằng password
sudo passwd ubuntu
# (Nhập mật khẩu bạn muốn đặt, ví dụ: ubuntu123)
```

Thoát khỏi máy ảo để quay lại Terminal của máy Mac:
```bash
exit
```

---

### Bước 4.2: Thực hiện kết nối SSH từ máy Mac

Tại cửa sổ Terminal trên macOS, gõ lệnh:

```bash
ssh ubuntu@192.168.64.2
```

```text
┌─────────────────────────────────────────────────────────────┐
│                 KỊCH BẢN KẾT NỐI LẦN ĐẦU                    │
└─────────────────────────────────────────────────────────────┘
The authenticity of host '192.168.64.2 (192.168.64.2)' can't be established.
ED25519 key fingerprint is SHA256:abcd1234efgh5678...
This key is not known by any other names.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '192.168.64.2' (ED25519) to the list of known hosts.
ubuntu@192.168.64.2's password: [Nhập mật khẩu bạn vừa tạo]
```

Sau khi đăng nhập thành công, prompt đổi thành `ubuntu@server:~$`. Hãy chạy kiểm chứng:
```bash
whoami
hostname
pwd
```

> [!IMPORTANT]
> **Câu hỏi tư duy:** *Những câu lệnh `whoami`, `hostname`, `pwd` trên đang chạy ở đâu?*  
> **Trả lời:** Chúng đang được **thực thi 100% bên trong CPU và RAM của Ubuntu Server VM**. Cửa sổ Terminal trên máy Mac lúc này chỉ đóng vai trò là một "màn hình hiển thị từ xa".

---

### 4.3. Tại sao môi trường Production NÓI KHÔNG với Password Authentication?

1. **Nguy cơ tấn công dò quét mật khẩu (Brute-force Attack):** Các botnet trên Internet liên tục quét cổng 22 và thử hàng triệu mật khẩu phổ biến mỗi ngày.
2. **Mật khẩu yếu và dễ lộ:** Con người thường có xu hướng đặt mật khẩu ngắn, dễ đoán hoặc dùng chung mật khẩu.
3. **Bất tiện khi tự động hóa (CI/CD):** Không thể viết script tự động deploy nếu mỗi lần chạy lệnh lại phải có người ngồi gõ mật khẩu.

$\rightarrow$ **Giải pháp chuẩn công nghiệp:** Sử dụng **Xác thực bằng cặp khóa (SSH Key Authentication)**.

---

## 🔑 5. Xác Thực Bằng SSH Key (Public Key Authentication)

### 5.1. Bản chất của cặp khóa Bất Đối Xứng (Asymmetric Key Pair)

Khi bạn tạo SSH Key, hệ thống sẽ sinh ra một cặp gồm **2 file khóa có mối liên hệ toán học mật thiết với nhau**:

```text
┌─────────────────────────────────────────────────────────────────────────┐
│                        CẶP KHÓA SSH (KEY PAIR)                          │
├────────────────────────────────────┬────────────────────────────────────┤
│  1. PRIVATE KEY (Khóa bí mật)      │  2. PUBLIC KEY (Khóa công khai)    │
│  - Lưu DUY NHẤT trên máy Mac       │  - Đặt trên Ubuntu Server          │
│  - File: ~/.ssh/id_ed25519         │  - File: ~/.ssh/id_ed25519.pub     │
│  - TUYỆT ĐỐI KHÔNG CHIA SẺ VỚI AI  │  - Có thể gửi công khai cho mọi    │
│  - Giống như "Chìa khóa vạn năng"  │    server mà không sợ mất cắp      │
│                                    │  - Giống như "Ổ khóa"              │
└────────────────────────────────────┴────────────────────────────────────┘
```

```text
MacBook (Host)                                                 Ubuntu Server
┌─────────────────────────┐                                ┌─────────────────────────┐
│  PRIVATE KEY            │                                │  PUBLIC KEY             │
│  (~/.ssh/id_ed25519)    │                                │  (~/.ssh/authorized_keys│
│                         │                                │                         │
│  1. Client gửi yêu cầu  │                                │                         │
│  ───────────────────────┼───────────────────────────────►│                         │
│                         │                                │                         │
│                         │   2. Server tạo câu đố ngẫu    │                         │
│                         │      nhiên, mã hóa bằng        │                         │
│                         │      Public Key rồi gửi về     │                         │
│  ◄──────────────────────┼────────────────────────────────┤                         │
│                         │                                │                         │
│  3. Client dùng Private │                                │                         │
│     Key giải câu đố     │                                │                         │
│     và gửi kết quả lại  │                                │                         │
│  ───────────────────────┼───────────────────────────────►│                         │
│                         │                                │  4. Server khớp kết quả │
│                         │                                │     ==> ĐĂNG NHẬP THÀNH │
│                         │                                │         CÔNG! (No Pass) │
└─────────────────────────┘                                └─────────────────────────┘
```

> [!TIP]
> **Tại sao kẻ xấu có Public Key trên server cũng không thể đăng nhập được?**  
> Vì dữ liệu được mã hóa bằng Public Key **chỉ có thể giải mã được bằng Private Key tương ứng**. Server không cần biết Private Key của bạn là gì, nó chỉ cần kiểm tra xem bạn có giải mã được "câu đố" mà nó gửi hay không!

---

### 5.2. Tạo SSH Key trên máy Mac

Mở Terminal trên máy Mac (đảm bảo đang ở máy Mac, không phải trong VM) và chạy lệnh:

```bash
ssh-keygen -t ed25519 -C "admin@macbook"
```

```text
┌─────────────────────────────────────────────────────────────┐
│                   GIẢI NGHĨA TỪNG THAM SỐ                   │
├───────────────┬─────────────────────────────────────────────┤
│ -t ed25519    │ Sử dụng thuật toán đường cong Elliptic      │
│               │ (ED25519): Nhanh nhất, cực kỳ an toàn, khóa │
│               │ ngắn gọn hơn nhiều so với chuẩn cũ RSA 4096.│
│ -C "comment"  │ Ghi chú định danh vào cuối file public key. │
└───────────────┴─────────────────────────────────────────────┘
```

*Quá trình sinh khóa:*
1. **File lưu:** Bấm `Enter` để lưu vào đường dẫn mặc định (`/Users/macbook/.ssh/id_ed25519`).
2. **Passphrase:** Nhập mật khẩu bảo vệ khóa (hoặc bấm `Enter` 2 lần để trống nếu bạn muốn tự động hóa hoàn toàn).

Kiểm tra cặp khóa vừa tạo trên máy Mac:
```bash
ls -la ~/.ssh/id_ed25519*
```
*Bạn sẽ thấy 2 file:*
* `id_ed25519`: Private key (Quyền phải là `600` hoặc `-rw-------`).
* `id_ed25519.pub`: Public key (Quyền `644` hoặc `-rw-r--r--`).

---

## 📥 6. Đưa Public Key Lên Server (`~/.ssh/authorized_keys`)

Để máy chủ nhận diện chìa khóa của bạn, bạn phải sao chép nội dung file **Public Key (`id_ed25519.pub`)** từ máy Mac vào file `~/.ssh/authorized_keys` của tài khoản tương ứng trên Ubuntu Server.

### Cách 1: Sử dụng lệnh tự động `ssh-copy-id` (Khuyến nghị)

Chạy trực tiếp từ Terminal máy Mac:
```bash
ssh-copy-id -i ~/.ssh/id_ed25519.pub ubuntu@192.168.64.2
```
*(Nhập mật khẩu user `ubuntu` một lần duy nhất. Lệnh này sẽ tự động kết nối, tạo thư mục `~/.ssh`, đẩy public key vào file `authorized_keys` và phân quyền chuẩn xác).*

---

### Cách 2: Thiết lập thủ công (Hiểu sâu bản chất)

Nếu server không có `ssh-copy-id`, đây là các bước thủ công bên trong Ubuntu Server:

```bash
# 1. Đăng nhập vào Ubuntu Server
# 2. Tạo thư mục .ssh (nếu chưa có) và phân quyền 700
mkdir -p ~/.ssh
chmod 700 ~/.ssh

# 3. Mở file authorized_keys và dán toàn bộ nội dung file id_ed25519.pub vào
nano ~/.ssh/authorized_keys

# 4. Phân quyền bắt buộc 600 cho file authorized_keys
chmod 600 ~/.ssh/authorized_keys
```

---

### 6.1. Tại sao phân quyền `700` và `600` là SỐNG CÒN?

Tiến trình `sshd` trên Ubuntu được trang bị tính năng bảo vệ nghiêm ngặt gọi là **StrictModes**:
* Thư mục `~/.ssh` **phải là `700` (`drwx------`)**: Chỉ chủ sở hữu mới có quyền vào và đọc.
* File `authorized_keys` **phải là `600` (`-rw-------`)**: Chỉ chủ sở hữu mới có quyền đọc/sửa.

> [!WARNING]
> **Nguyên nhân lỗi số 1 khi SSH:** Nếu bạn lỡ tay đặt quyền `chmod 777 ~/.ssh` hoặc `chmod 777 authorized_keys`, tiến trình `sshd` sẽ **lập tức từ chối nhận SSH Key** vì nó phát hiện file chứa chìa khóa đang ở trạng thái không an toàn (bất kỳ ai trên máy cũng có thể ghi đè khóa của họ vào để hack tài khoản).

---

## ⚡ 7. Cấu Hình Tối Ưu Với File `~/.ssh/config`

Thay vì mỗi lần kết nối bạn phải nhớ và gõ: `ssh -i ~/.ssh/id_ed25519 -p 22 ubuntu@192.168.64.2`, bạn có thể tạo một **Bí danh (Alias)** cực kỳ tiện lợi trên máy Mac.

### 7.1. Thực hành tạo file cấu hình trên máy Mac

Mở file cấu hình SSH trên máy Mac bằng `nano`:
```bash
nano ~/.ssh/config
```

Thêm vào nội dung cấu hình sau:
```text
Host my-server
    HostName 192.168.64.2
    User ubuntu
    Port 22
    IdentityFile ~/.ssh/id_ed25519
```

*Cách lưu file trong `nano`:*
1. Nhấn `Ctrl + O` rồi bấm `Enter` để ghi file.
2. Nhấn `Ctrl + X` để thoát `nano`.

Phân quyền an toàn cho file config trên Mac:
```bash
chmod 600 ~/.ssh/config
```

### 7.2. Tận hưởng kết quả:

Bây giờ, từ bất kỳ đâu trên Terminal máy Mac, bạn chỉ cần gõ:
```bash
ssh my-server
```
$\rightarrow$ Máy Mac tự động đọc file config, tìm đúng IP, đúng User, đúng Private Key và đăng nhập vào Ubuntu Server trong **0.5 giây mà không cần hỏi mật khẩu**!

---

## 🛡️ 8. Tăng Cường Bảo Mật SSH Server (`/etc/ssh/sshd_config`)

File cấu hình chính của máy chủ SSH nằm tại: `/etc/ssh/sshd_config`.

```text
┌───────────────────────────┬───────────────────┬─────────────────────────────┐
│ Tham số cấu hình          │ Giá trị khuyến nghị│ Ý nghĩa bảo mật             │
├───────────────────────────┼───────────────────┼─────────────────────────────┤
│ Port                      │ 22 (hoặc đổi 2222)│ Cổng lắng nghe kết nối SSH. │
│ PermitRootLogin           │ no                │ Cấm tuyệt đối login bằng root│
│ PasswordAuthentication    │ no                │ Tắt đăng nhập bằng mật khẩu │
│ PubkeyAuthentication      │ yes               │ Bắt buộc dùng SSH Key       │
│ MaxAuthTries              │ 3                 │ Số lần thử sai tối đa trước │
│                           │                   │ khi bị ngắt kết nối.        │
└───────────────────────────┴───────────────────┴─────────────────────────────┘
```

### 8.1. Quy tắc sống còn chống bị "Khóa ngoài cửa" (Lockout Prevention)

> [!CAUTION]
> **Quy tắc vàng của Quản trị viên hệ thống:**
> Khi chỉnh sửa file `/etc/ssh/sshd_config` (đặc biệt là khi đổi Port hoặc tắt `PasswordAuthentication`):
> 1. **KHÔNG BAO GIỜ** đóng cửa sổ Terminal đang SSH hiện tại!
> 2. Mở một **cửa sổ Terminal MỚI** trên máy Mac và thử kết nối: `ssh my-server`.
> 3. Chỉ khi nào cửa sổ mới đăng nhập thành công 100% thì bạn mới được đóng cửa sổ cũ. Nếu có lỗi, bạn vẫn còn cửa sổ cũ để sửa lại cấu hình và cứu server!

---

## 🔍 9. Kỹ Năng Debug Kết Nối SSH Chuyên Nghiệp

Khi lệnh SSH không thành công, đừng đoán mò! Hãy sử dụng cờ Verbose Mode:

```bash
ssh -v my-server      # Chế độ debug cơ bản
ssh -vvv my-server    # Chế độ debug chi tiết mức tối đa (In từng gói tin)
```

```text
┌─────────────────────────────────────────────────────────────────────────┐
│                      SƠ ĐỒ TRUY TÌM NGUYÊN NHÂN LỖI                     │
└────────────────────────────────────┬────────────────────────────────────┘
                                     │
           ┌─────────────────────────┼─────────────────────────┐
           ▼                         ▼                         ▼
  [ Lỗi Tầng Mạng ]         [ Lỗi SSH Server ]       [ Lỗi Xác Thực ]
  - Connection timed out    - Connection refused     - Permission denied
  - No route to host                                   (publickey)
```

### Bảng xử lý 5 lỗi kinh điển:

| Thông báo lỗi | Vị trí lỗi | Nguyên nhân & Cách khắc phục |
| :--- | :--- | :--- |
| **`Connection timed out`** | Tầng Mạng / Firewall | **Nguyên nhân:** Sai địa chỉ IP, VM chưa bật, hoặc Tường lửa (Firewall) đang drop gói tin.<br>**Xử lý:** Dùng lệnh `ping <IP>` kiểm tra thông mạng; kiểm tra VM đã `Running` chưa. |
| **`Connection refused`** | Cổng / SSH Daemon | **Nguyên nhân:** Máy chủ đang bật nhưng dịch vụ `sshd` bị tắt, hoặc server đang đổi sang port khác.<br>**Xử lý:** Vào VM kiểm tra: `sudo systemctl status ssh` và `sudo ss -tulpn`. |
| **`Permission denied (publickey)`** | Xác thực Key / Quyền | **Nguyên nhân:** Server không nhận SSH Key. Thường do: Chưa copy public key vào `authorized_keys`, sai user đăng nhập, hoặc phân quyền `~/.ssh` sai.<br>**Xử lý:** Kiểm tra quyền `chmod 700 ~/.ssh` và `chmod 600 ~/.ssh/authorized_keys`. |
| **`Host key verification failed`** | Dữ liệu `known_hosts` | **Nguyên nhân:** Bạn vừa cài lại Ubuntu Server nhưng vẫn dùng IP cũ khiến Host Key bị thay đổi (Mac nghi ngờ bị tấn công Man-in-the-Middle).<br>**Xử lý:** Xóa dòng key cũ trên Mac bằng lệnh: `ssh-keygen -R <IP_SERVER>`. |
| **`Permissions 0644 for id_ed25519 are too open`** | Client Private Key | **Nguyên nhân:** File Private Key trên máy Mac bị phân quyền quá lỏng lẻo.<br>**Xử lý:** Chạy lệnh trên Mac: `chmod 600 ~/.ssh/id_ed25519`. |

---

## 🧪 10. Bài Thực Hành Tổng Hợp (Lab 10 Bước)

*Hãy thực hiện toàn bộ quy trình thiết lập SSH chuẩn Production từ máy Mac vào Ubuntu Server VM:*

```text
MacBook (Terminal) ──[SSH Key ed25519]──► Ubuntu VM (User: deploy)
```

1. **Khởi động máy ảo** và kiểm tra trạng thái dịch vụ SSH (`systemctl status ssh`).
2. **Tạo user mới chuyên dùng để deploy** trên Ubuntu:
   ```bash
   sudo adduser --disabled-password --gecos "" deploy
   sudo usermod -aG sudo deploy
   ```
3. **Sinh cặp khóa SSH mới trên máy Mac** với tên riêng `~/.ssh/id_deploy_lab`:
   ```bash
   ssh-keygen -t ed25519 -f ~/.ssh/id_deploy_lab -C "deploy@lab"
   ```
4. **Đưa Public Key lên server** cho user `deploy`:
   ```bash
   ssh-copy-id -i ~/.ssh/id_deploy_lab.pub deploy@<IP_VM>
   ```
5. **Kiểm tra phân quyền trên Ubuntu VM**:
   ```bash
   ls -ld /home/deploy/.ssh
   ls -l /home/deploy/.ssh/authorized_keys
   ```
   *(Đảm bảo thư mục là `700` và file là `600`, thuộc sở hữu `deploy:deploy`).*
6. **Tạo cấu hình Alias trong `~/.ssh/config` trên máy Mac**:
   ```text
   Host lab-server
       HostName <IP_VM>
       User deploy
       IdentityFile ~/.ssh/id_deploy_lab
   ```
7. **Đăng nhập một chạm từ máy Mac**:
   ```bash
   ssh lab-server
   ```
8. **Chạy kiểm tra xác nhận danh tính bên trong server**: `whoami` (trả về `deploy`).
9. **Thử nghiệm tính năng Debug**: Thoát ra Mac và gõ `ssh -v lab-server` để quan sát toàn bộ quá trình trao đổi khóa và nạp public key.
10. **Kiểm tra file nhật ký đăng nhập trên Ubuntu**:
    ```bash
    sudo journalctl -u ssh -n 20 --no-pager
    ```
    *Quan sát thấy dòng log ghi nhận: `Accepted publickey for deploy from ... ssh2: ED25519 ...`*.

---

## 📌 11. Tóm Tắt Bản Chất Cần Nhớ

1. **SSH Client** (macOS) kết nối tới **SSH Server `sshd`** (Ubuntu) qua cổng **TCP Port 22**.
2. **Private Key giữ bí mật tuyệt đối trên máy Mac**, **Public Key đặt trong `~/.ssh/authorized_keys` trên Ubuntu Server**.
3. **Bộ đôi phân quyền vàng:** Thư mục `~/.ssh` bắt buộc là **`700`**, file `authorized_keys` và Private Key bắt buộc là **`600`**.
4. **Tối ưu thao tác quản trị hàng ngày** bằng file cấu hình `~/.ssh/config` trên máy Mac.
5. **Luôn nhớ quy tắc an toàn:** Không bao giờ ngắt kết nối SSH cũ khi đang thử nghiệm sửa đổi cấu hình `/etc/ssh/sshd_config`.
