# 📘 Phần 01: Tổng Quan Ubuntu & Thiết Lập Môi Trường với Multipass trên macOS

> **Motto cốt lõi:** *Hiểu bản chất > Nhớ câu lệnh | Thực hành thực tế > Đọc lý thuyết suông.*  
> Mỗi kiến thức trong bài học này đều đi qua chu trình: **Khái niệm → Bản chất bên trong → Thực hành → Quan sát kết quả → Giải thích cơ chế.**

---

## 🎯 1. Mục Tiêu Của Phần Học

Phần học này tập trung vào 2 mục tiêu duy nhất: **Hiểu bản chất hoạt động của hệ thống** và **Tự tay dựng, vận hành thành công một máy chủ Ubuntu Server VM trên máy Mac**.

Sau khi hoàn thành bài học này, bạn sẽ tự tin trả lời chính xác và rành mạch 10 câu hỏi cốt lõi:
1. **Ubuntu Server là gì** và nó khác gì so với hệ điều hành macOS bạn đang dùng?
2. **Virtual Machine (VM)** thực chất là gì dưới góc nhìn của phần cứng?
3. **Host** và **Guest** có mối quan hệ như thế nào?
4. **Multipass** làm nhiệm vụ gì và đứng ở đâu trong bức tranh toàn cảnh?
5. VM lấy **CPU, RAM, Ổ đĩa (Disk) và Mạng (Network)** từ đâu?
6. Toàn bộ dữ liệu của Ubuntu VM được **lưu trữ ở đâu** trên chiếc MacBook?
7. Khi VM bị **dừng (Stop)** thì dữ liệu có bị mất không?
8. Khi **xóa VM (Delete & Purge)** thì điều gì thực sự xảy ra với ổ cứng máy Mac?
9. VM lấy **địa chỉ IP** như thế nào và tại sao nó lại có Internet?
10. Làm thế nào để **máy Mac và máy ảo Ubuntu giao tiếp** được với nhau?

---

## 🏗️ 2. Mô Hình Phân Tầng: Host → Multipass → Ubuntu VM

Trước khi gõ bất kỳ câu lệnh nào, bạn cần nắm rõ bức tranh kiến trúc phân tầng trong máy tính của mình.

### 2.1. Sơ đồ kiến trúc phân tầng

```text
┌────────────────────────────────────────────────────────────────────────┐
│                             CHIẾC MACBOOK                              │
│   Phần cứng vật lý: Apple Silicon (M1/M2/M3/M4) hoặc Intel CPU,       │
│                     RAM vật lý, Ổ cứng SSD vật lý, Card Wi-Fi          │
│                                                                        │
│   ┌────────────────────────────────────────────────────────────────┐   │
│   │                      HOST OS: macOS                            │   │
│   │   (Hệ điều hành chính điều khiển trực tiếp phần cứng)          │   │
│   │                                                                │   │
│   │   ┌────────────────────────────────────────────────────────┐   │   │
│   │   │                MULTIPASS (Daemon & CLI)                │   │   │
│   │   │   (Phần mềm điều phối và quản lý máy ảo)               │   │   │
│   │   │                                                        │   │   │
│   │   │   ┌────────────────────────────────────────────────┐   │   │   │
│   │   │   │            GUEST VM: Ubuntu Server             │   │   │   │
│   │   │   │                                                │   │   │   │
│   │   │   │   ├── CPU: 2 vCPU (Ảo hóa từ nhân CPU của Mac) │   │   │   │
│   │   │   │   ├── RAM: 2 GB (Cấp phát từ RAM của Mac)      │   │   │   │
│   │   │   │   ├── Disk: 20 GB (1 file image trên SSD Mac)  │   │   │   │
│   │   │   │   └── Network: Card mạng ảo + IP riêng         │   │   │   │
│   │   │   │   ├── Filesystem riêng biệt (/etc, /var, /home)│   │   │   │
│   │   │   │   └── Tiến trình riêng (Process tree PID 1)    │   │   │   │
│   │   │   └────────────────────────────────────────────────┘   │   │   │
│   │   └────────────────────────────────────────────────────────┘   │   │
│   └────────────────────────────────────────────────────────────────┘   │
└────────────────────────────────────────────────────────────────────────┘
```

### 2.2. Giải thích bản chất từng thành phần

1. **Mac là Host (Máy chủ vật lý):**
   - Nắm giữ toàn bộ tài nguyên thật: Nhân CPU, thanh RAM, chip nhớ SSD và card mạng Wi-Fi.
   - macOS là hệ điều hành nền tảng chịu trách nhiệm chia sẻ một phần tài nguyên đó cho các ứng dụng và máy ảo.
2. **Ubuntu là Guest (Máy khách ảo):**
   - Là hệ điều hành chạy bên trong môi trường được mô phỏng.
   - Ubuntu Server hoàn toàn "nghĩ rằng" nó đang sở hữu một cỗ máy vật lý có 2 CPU, 2GB RAM và 20GB ổ cứng, dù thực tế nó chỉ đang dùng tài nguyên được macOS cho mượn.
3. **Môi trường cách ly độc lập (Sandbox):**
   - **Filesystem riêng:** Cây thư mục Linux (`/`, `/home`, `/etc`, `/var`) hoàn toàn nằm bên trong máy ảo, không hề trộn lẫn với thư mục `/Users/macbook/...` của macOS.
   - **Process riêng:** Tiến trình bên trong Ubuntu (như Systemd, Nginx, Database) chạy độc lập và không hiển thị trực tiếp trong Activity Monitor của Mac như một app thông thường, mà được gom trong tiến trình ảo hóa của Multipass.
   - **Network Interface riêng:** Ubuntu có card mạng ảo và địa chỉ IP riêng biệt, có thể ping và nhận kết nối từ macOS.
   - **Lifecycle riêng:** Bạn có thể khởi động lại (reboot) hoặc tắt Ubuntu mà máy Mac vẫn hoạt động bình thường, không hề bị gián đoạn.

---

## 💻 3. Cài Đặt Multipass

### 3.1. Khái niệm & Bản chất
* **Homebrew là gì?** Trình quản lý gói cho macOS giúp tự động tải, cài đặt và cập nhật các phần mềm/công cụ dòng lệnh mà không cần tải file `.dmg` hay kéo thả thủ công.
* **Multipass cài ở đâu?** Tiến trình nền (`multipassd`) được đăng ký vào hệ thống dịch vụ của macOS (`launchd`), còn công cụ dòng lệnh `multipass` được đặt trong đường dẫn hệ thống (thường là `/usr/local/bin/multipass` hoặc `/opt/homebrew/bin/multipass`).
* **Multipass CLI giao tiếp như thế nào?** Khi bạn gõ lệnh `multipass ...`, chương trình dòng lệnh (CLI) sẽ gửi tín hiệu qua một socket nội bộ (UNIX Domain Socket) đến tiến trình nền `multipassd` để yêu cầu Hypervisor thực thi thao tác tương ứng.

### 3.2. Thực hành cài đặt
Mở ứng dụng **Terminal** trên macOS và chạy lệnh:

```bash
brew install --cask multipass
```

### 3.3. Quan sát & Kiểm tra kết quả
Sau khi cài đặt xong, hãy kiểm tra phiên bản:

```bash
multipass version
```

**Kết quả hiển thị:**
```text
multipass   1.14.x+mac
multipassd  1.14.x+mac
```

### 3.4. Giải thích kết quả
* Dòng `multipass`: Cho biết phiên bản của công cụ dòng lệnh (Client CLI).
* Dòng `multipassd`: Cho biết tiến trình nền (Daemon) đang chạy ngầm trên macOS và sẵn sàng nhận lệnh khởi tạo máy ảo.

---

## 🚀 4. Khởi Tạo Ubuntu Server VM

### 4.1. Khái niệm & Bản chất các tham số

Chúng ta sẽ khởi tạo một máy ảo hoàn chỉnh với lệnh:

```bash
multipass launch \
  --name server \
  --cpus 2 \
  --memory 2G \
  --disk 20G
```

```text
┌──────────────┬─────────────────────────────────────────────────────────────┐
│ Tham số      │ Bản chất hệ thống bên dưới                                  │
├──────────────┼─────────────────────────────────────────────────────────────┤
│ --name       │ Đặt tên định danh duy nhất cho VM là "server".             │
│ --cpus 2     │ Cấp 2 vCPU: Hypervisor ánh xạ thời gian xử lý của 2 nhân    │
│              │ CPU vật lý trên Mac cho máy ảo.                             │
│ --memory 2G  │ Cấp 2GB RAM: macOS khóa tạm thời tối đa 2GB bộ nhớ vật lý   │
│              │ để cấp cho VM khi nó vận hành.                              │
│ --disk 20G   │ Tạo 1 file disk ảo có giới hạn co giãn tối đa là 20GB.     │
│              │ Ban đầu file này chỉ chiếm ~1.5GB SSD thật trên máy Mac.    │
└──────────────┴─────────────────────────────────────────────────────────────┘
```

> [!NOTE]
> Khi không chỉ định phiên bản, Multipass tự động tải **Ubuntu LTS mới nhất** (bản Cloud Image chính thức từ Canonical).

### 4.2. Thực hành khởi tạo
Chạy câu lệnh trên trong Terminal máy Mac. Quá trình tải image và khởi tạo sẽ diễn ra tự động.

### 4.3. Quan sát kết quả từ macOS
Chạy 2 lệnh sau trên Terminal máy Mac:

```bash
multipass list
```
**Kết quả hiển thị:**
```text
Name       State       IPv4             Image
server     Running     192.168.64.2     Ubuntu 24.04 LTS
```

Xem thông số chi tiết của VM vừa tạo:
```bash
multipass info server
```
**Kết quả hiển thị:**
```text
Name:           server
State:          Running
IPv4:           192.168.64.2
Release:        Ubuntu 24.04 LTS
Image hash:     ...
CPU(s):         2
Load:           0.00 0.00 0.00
Disk usage:     1.4GiB out of 19.3GiB
Memory usage:   178.5MiB out of 1.9GiB
Mounts:         --
```

### 4.4. Giải thích kết quả & Đối chiếu tài nguyên
* **State: Running:** Máy ảo đã được bật và hệ điều hành Ubuntu đang chạy ngầm.
* **Disk usage (1.4GiB / 19.3GiB):** Chứng minh cơ chế *Sparse Disk* (Cấp phát động): Dù bạn cấp 20GB, nó chỉ thực sự chiếm 1.4GB trên SSD máy Mac.
* **Memory usage (178.5MiB / 1.9GiB):** Chứng minh Ubuntu Server cực kỳ nhẹ: Khi vừa khởi động, hệ điều hành chỉ dùng chưa tới 200MB RAM.

---

## 🔑 5. Truy Cập Vào Bên Trong Ubuntu Server

### 5.1. Khái niệm & Bản chất
* Lệnh `multipass shell server` thực chất là mở một đường truyền an toàn (qua SSH/Socket nội bộ) từ cửa sổ Terminal của Mac nối thẳng vào phiên làm việc dòng lệnh (`bash shell`) của người dùng mặc định mang tên `ubuntu` bên trong máy ảo.

```text
Terminal trên Mac ──(multipass shell)──► Multipass Socket ──► Ubuntu VM Shell (ubuntu@server)
```

### 5.2. Thực hành truy cập
Trên Terminal máy Mac, chạy lệnh:

```bash
multipass shell server
```

### 5.3. Quan sát sự thay đổi
Dấu nhắc lệnh (Prompt) của Terminal lập tức thay đổi:
* **Trước khi vào (macOS):** `macbook@MacBook-Pro ~ %`
* **Sau khi vào (Ubuntu VM):** `ubuntu@server:~$ `

Hãy chạy tiếp 3 lệnh cơ bản để kiểm chứng:
```bash
whoami
hostname
pwd
```

**Kết quả hiển thị:**
```text
ubuntu
server
/home/ubuntu
```

### 5.4. Giải thích kết quả
1. **`whoami` trả về `ubuntu`:** Bạn đang hoạt động dưới tài khoản người dùng chuẩn có quyền `sudo` trong Linux, không còn là user macOS nữa.
2. **`hostname` trả về `server`:** Tên máy chủ bạn đã đặt khi dùng cờ `--name server`.
3. **`pwd` trả về `/home/ubuntu`:** Thư mục làm việc hiện tại nằm hoàn toàn trong hệ thống tập tin Linux (Filesystem của VM), hoàn toàn tách biệt với thư mục cá nhân trên macOS.

> [!TIP]
> Để thoát khỏi máy ảo và quay trở về Terminal máy Mac: Gõ lệnh `exit` hoặc bấm phím `Ctrl + D`.

---

## 🔍 6. Quan Sát Hệ Thống Từ Bên Trong Ubuntu Server

Đang đứng bên trong shell của Ubuntu (`ubuntu@server:~$`), hãy dùng các lệnh sau để "soi" trực tiếp vào phần cứng ảo:

### 6.1. Thực hành chuỗi lệnh quan sát

```bash
uname -a
cat /etc/os-release
nproc
free -h
df -h
ip addr
```

### 6.2. Quan sát & Giải thích từng lệnh

| Câu lệnh | Mục đích quan sát | Ý nghĩa dữ liệu trả về |
| :--- | :--- | :--- |
| `uname -a` | Xem thông tin Kernel & Kiến trúc | Thấy `Linux server ... aarch64` (nếu dùng chip Apple Silicon) hoặc `x86_64` (nếu dùng chip Intel). |
| `cat /etc/os-release` | Xem bản phân phối hệ điều hành | Xác nhận chính xác hệ điều hành là `Ubuntu 24.04 LTS (Noble Numbat)`. |
| `nproc` | Đếm số nhân CPU máy ảo nhìn thấy | Trả về `2` (Khớp chính xác với cấu hình `--cpus 2`). |
| `free -h` | Quan sát bộ nhớ RAM | Cột `total` là `1.9Gi`, cột `used` chỉ khoảng `180Mi`, còn trống hơn `1.6Gi`. |
| `df -h` | Quan sát các phân vùng ổ đĩa | Dòng `/dev/root` mount tại `/` có kích thước `19G` (Khớp với cấu hình `--disk 20G`). |
| `ip addr` | Xem danh sách card mạng | Thấy card `lo` (Loopback 127.0.0.1) và card mạng ảo (vd `enp0s1`) mang địa chỉ IP của VM. |

---

## 🌐 7. Bản Chất Network Của VM & Giao Tiếp Với macOS

```text
┌────────────────────────────────────────────────────────────────────────┐
│                              MÁY MAC                                   │
│   IP Wi-Fi thật: 192.168.1.50                                          │
│   localhost Mac = 127.0.0.1 (Chính chiếc máy Mac)                     │
│                                                                        │
│          │                                                             │
│          │ Giao tiếp nội bộ qua Virtual Bridge                         │
│          ▼                                                             │
│   ┌────────────────────────────────────────────────────────────────┐   │
│   │                      UBUNTU SERVER VM                          │   │
│   │   IP ảo do Multipass cấp: 192.168.64.2                         │   │
│   │   localhost VM = 127.0.0.1 (Chính cỗ máy ảo Ubuntu)            │   │
│   │                                                                │   │
│   │   Card mạng ảo: enp0s1 (192.168.64.2)                          │   │
│   └────────────────────────────────┬───────────────────────────────┘   │
└────────────────────────────────────┼───────────────────────────────────┘
                                     │ (Gói tin đi qua NAT của Mac)
                                     ▼
                            INTERNET (google.com)
```

### 7.1. Trả lời các câu hỏi cốt lõi về Mạng

1. **Địa chỉ IP của VM thuộc về ai và từ đâu ra?**
   - Khi VM khởi động, Hypervisor trên macOS đóng vai trò là một Virtual Router/DHCP Server. Nó cấp một địa chỉ IP riêng (thường là dải `192.168.64.x`) cho card mạng ảo của VM.
2. **`localhost` của Mac có phải `localhost` của Ubuntu không?**
   - **HOÀN TOÀN KHÔNG!**
   - `localhost` (`127.0.0.1`) là địa chỉ loopback tự trỏ về chính máy tính đang chạy lệnh đó.
   - Nếu bạn chạy ứng dụng web ở port 3000 trong Ubuntu, bạn đứng trong Ubuntu gọi `localhost:3000` sẽ được; nhưng đứng ở trình duyệt máy Mac gõ `localhost:3000` sẽ **không ra gì cả** (vì Mac đang tự tìm chính nó). Từ Mac, bạn phải truy cập qua: `http://<IP_CỦA_VM>:3000` (vd: `http://192.168.64.2:3000`).
3. **Vì sao Ubuntu VM truy cập được Internet?**
   - Hypervisor cấu hình cơ chế **NAT (Network Address Translation)**: Mọi gói tin từ VM muốn ra ngoài thế giới sẽ được máy Mac "mượn đường" chuyển tiếp qua card Wi-Fi của Mac ra Internet, sau đó nhận dữ liệu phản hồi trả lại cho VM.

### 7.2. Thực hành kiểm tra mạng

1. **Kiểm tra kết nối Internet từ bên trong Ubuntu:**
   ```bash
   ping -c 4 google.com
   ```
   *Quan sát kết quả:* Thấy các dòng `64 bytes from ... time=...ms` và `0% packet loss` → Kết nối ra Internet hoàn toàn thông suốt.

2. **Kiểm tra kết nối từ macOS vào Ubuntu:**
   Mở một tab Terminal mới trên máy Mac (không vào shell của VM) và gõ lệnh ping tới IP của VM:
   ```bash
   ping -c 4 192.168.64.2
   ```
   *(Thay `192.168.64.2` bằng IP thực tế bạn thấy trong `multipass list`).*  
   *Quan sát kết quả:* Máy Mac ping thành công tới VM → Hai máy đã thông mạng nội bộ với nhau!

---

## 📦 8. Cơ Chế Quản Lý Gói (Package Management với APT)

### 8.1. Khái niệm & Bản chất

```text
┌─────────────────────────────────────────────────────────────────────────┐
│                        CƠ CHẾ HOẠT ĐỘNG CỦA APT                         │
│                                                                         │
│  1. sudo apt update                                                     │
│     Ubuntu VM ────── Tải danh mục phiên bản mới nhất ──────► Repository │
│     (Chỉ cập nhật chỉ mục index trong /var/lib/apt/lists/)              │
│                                                                         │
│  2. sudo apt upgrade                                                    │
│     Ubuntu VM ────── Tải file cài đặt & Ghi đè phiên bản cũ ─► Server   │
│     (Thực sự nâng cấp các gói phần mềm đã cài)                          │
└─────────────────────────────────────────────────────────────────────────┘
```

* **Package (Gói phần mềm):** Là file `.deb` chứa code phần mềm đã biên dịch sẵn, file cấu hình và thông tin các thư viện phụ thuộc.
* **Repository (Kho phần mềm):** Máy chủ chính thức của Canonical lưu trữ hàng chục ngàn gói phần mềm.
* **APT (Advanced Package Tool):** Trình quản lý gói của Ubuntu, giúp tải và cài đặt phần mềm cùng mọi dependencies một cách tự động.

### 8.2. Phân biệt rõ `apt update` và `apt upgrade`

* **`sudo apt update`:**
  - Đi ra ngoài mạng, hỏi Repository xem "Hiện tại có những phần mềm nào có bản mới?".
  - Tải danh sách metadata về lưu vào đĩa cứng.
  - **Chưa hề thay đổi hay cài mới bất kỳ file phần mềm nào**.
* **`sudo apt upgrade -y`:**
  - Đối chiếu danh sách vừa tải với các phần mềm đang có trên máy.
  - Tải các gói phần mềm mới về và **tiến hành cài đặt đè/nâng cấp thực sự**.
  - Cờ `-y` tự động đồng ý cài đặt không cần dừng lại hỏi xác nhận.

> [!IMPORTANT]
> **Quy tắc vàng:** Luôn luôn chạy `sudo apt update` trước khi chạy `sudo apt upgrade` hoặc trước khi cài một phần mềm mới (`sudo apt install ...`).

### 8.3. Thực hành cập nhật Ubuntu
Bên trong shell của Ubuntu, chạy:
```bash
sudo apt update && sudo apt upgrade -y
```

---

## 🔄 9. Vòng Đời Của Server (Lifecycle): Chứng Minh STOP ≠ DELETE

Đây là một trong những bài học quan trọng nhất về bản chất lưu trữ của máy ảo.

```text
                       ┌───────────────────────┐
                       │  multipass launch ... │
                       └───────────┬───────────┘
                                   │
                                   ▼
                       ┌───────────────────────┐
            ┌─────────►│     RUNNING (Chạy)    │◄─────────┐
            │          └───────────┬───────────┘          │
            │                      │                      │
     multipass start        multipass stop         multipass restart
            │                      │                      │
            │                      ▼                      │
            │          ┌───────────────────────┐          │
            └──────────┤    STOPPED (Dừng)     ├──────────┘
                       └───────────┬───────────┘
                                   │
                             multipass delete
                                   │
                                   ▼
                       ┌───────────────────────┐
                       │   DELETED (Thùng rác) │
                       └───────────┬───────────┘
                                   │
                             multipass purge
                                   │
                                   ▼
                             [ XÓA VĨNH VIỄN ]
```

### 9.1. Thực hành kiểm chứng tính toàn vẹn dữ liệu

**Bước 1:** Vào bên trong VM và tạo một file thử nghiệm:
```bash
multipass shell server
echo "Du lieu quan trong cua server" > ~/important_data.txt
cat ~/important_data.txt
exit
```

**Bước 2:** Từ máy Mac, ra lệnh tắt máy ảo:
```bash
multipass stop server
```
Kiểm tra trạng thái bằng `multipass list`:
```text
Name       State       IPv4             Image
server     Stopped     --               Ubuntu 24.04 LTS
```

**Bước 3:** Bật lại máy ảo từ máy Mac:
```bash
multipass start server
```

**Bước 4:** Truy cập lại vào VM và kiểm tra file cũ:
```bash
multipass shell server
cat ~/important_data.txt
exit
```
*Kết quả:* File `important_data.txt` vẫn còn nguyên vẹn 100%!

### 9.2. Giải thích bản chất
* `multipass stop` chỉ tương đương với thao tác **Shut Down (Tắt nguồn)** máy tính.
* Bộ nhớ RAM được giải phóng trả lại cho macOS, nhưng toàn bộ dữ liệu trên ổ đĩa ảo (file `.qcow2` nằm trên SSD máy Mac) **hoàn toàn được giữ nguyên vẹn**.

---

## 🗑️ 10. Xóa VM Có Kiểm Soát: Phân Biệt `delete` và `purge`

### 10.1. Bản chất sự khác biệt
* **`multipass delete <tên_vm>`:**
  - Chuyển máy ảo vào trạng thái "Đã xóa" (tương tự như ném file vào Thùng rác trên macOS).
  - Dữ liệu ổ đĩa **vẫn còn trên máy Mac**, bạn hoàn toàn có thể khôi phục lại bằng lệnh `multipass recover server`.
* **`multipass purge`:**
  - Dọn sạch toàn bộ thùng rác (Empty Trash).
  - **Xóa vĩnh viễn file Virtual Disk khỏi SSD của máy Mac**, giải phóng 100% dung lượng. Thao tác này **không thể hoàn tác**.

### 10.2. Thực hành có kiểm soát

1. Đánh dấu xóa máy ảo:
   ```bash
   multipass delete server
   multipass list
   ```
   *Quan sát:* Cột `State` hiển thị `Deleted`.

2. Nếu muốn khôi phục lại (Thử nghiệm tính năng an toàn):
   ```bash
   multipass recover server
   multipass list
   ```
   *Quan sát:* Máy ảo quay trở lại trạng thái `Stopped`.

3. Xóa vĩnh viễn:
   ```bash
   multipass delete server
   multipass purge
   multipass list
   ```
   *Quan sát:* Danh sách hoàn toàn trống, máy ảo `server` đã bị xóa hoàn toàn.

---

## ⚡ 11. Bảng Tra Cứu Lệnh Multipass Cốt Lõi

| Lệnh | Nó làm gì? | Tác động đến VM? | Có ảnh hưởng dữ liệu? | Khi nào sử dụng? |
| :--- | :--- | :--- | :--- | :--- |
| `multipass launch` | Tạo & bật máy ảo mới | Khởi chạy 1 VM mới từ Cloud Image | Tạo mới 1 file disk ảo | Khi bắt đầu dựng một server mới |
| `multipass list` | Liệt kê danh sách VM | Không can thiệp, chỉ đọc trạng thái | **Không** | Khi cần xem IP hoặc xem VM nào đang chạy |
| `multipass info <tên>` | Xem thông số chi tiết | Đọc thông số CPU, RAM, Disk đã dùng | **Không** | Khi kiểm tra xem VM có bị đầy RAM/Disk không |
| `multipass shell <tên>` | Mở terminal vào VM | Mở phiên làm việc dòng lệnh | Theo thao tác của bạn trong shell | Khi cần cài đặt, gõ lệnh cấu hình server |
| `multipass stop <tên>` | Tắt máy ảo | Tắt nguồn VM, giải phóng RAM | **Không mất dữ liệu** | Khi học xong, muốn tiết kiệm pin/RAM cho Mac |
| `multipass start <tên>` | Bật lại máy ảo | Khởi động VM đã tắt | **Không mất dữ liệu** | Khi muốn tiếp tục học/thực hành |
| `multipass restart <tên>` | Khởi động lại VM | Reboot hệ điều hành Ubuntu | **Không mất dữ liệu** | Khi vừa cập nhật cấu hình hệ thống cấp thấp |
| `multipass delete <tên>` | Đưa VM vào thùng rác | Dừng VM và đánh dấu `Deleted` | Tạm ẩn, có thể khôi phục | Khi không cần dùng nhưng chưa muốn xóa hẳn |
| `multipass purge` | Dọn sạch thùng rác | Hủy bỏ hoàn toàn các VM `Deleted` | **XÓA VĨNH VIỄN DỮ LIỆU ĐĨA** | Khi muốn giải phóng hoàn toàn dung lượng SSD |

---

## 🛑 12. Những Thứ Chưa Cần Học Ở Giai Đoạn Này

Để việc học tập trung và không bị quá tải, bạn **CHƯA CẦN** cấu hình những thành phần sau ở phần này:
- ❌ **Cấu hình SSH Key & Đổi cổng SSH:** Sẽ học ở [Phần 04](./04-ssh.md) và [Phần 05](./05-ssh-key.md).
- ❌ **Tường lửa UFW & Fail2ban:** Sẽ học ở [Phần 07](./07-ufw-firewall.md) và [Phần 13](./13-fail2ban.md).
- ❌ **Web Server Nginx, Node.js, PostgreSQL, Docker:** Sẽ lần lượt được thực hành từ [Phần 08](./08-nginx.md) đến [Phần 16](./16-docker.md).
- ❌ **CI/CD, Cloud-init nâng cao, Kubernetes:** Thuộc giai đoạn nâng cao sau này.

*Mục tiêu của Phần 01 chỉ là: Làm chủ hoàn toàn vòng đời của một Ubuntu Server VM trên máy Mac.*

---

## 📝 13. Bài Thực Hành Cuối Phần (Hands-on Challenge)

*Hãy tự mình mở Terminal trên máy Mac và thực hiện tuần tự 16 bước sau mà không cần nhìn lại hướng dẫn bên trên:*

1. Cài đặt Multipass vào máy Mac (nếu chưa cài).
2. Tạo một máy ảo Ubuntu tên là `server` với: 2 vCPU, 2GB RAM, 20GB Disk.
3. Liệt kê danh sách VM và xác định địa chỉ IPv4 của máy ảo.
4. Xem thông số phần cứng chi tiết của `server` qua lệnh `info`.
5. Đăng nhập vào bên trong máy ảo bằng lệnh `shell`.
6. Dùng lệnh xác định số nhân CPU mà Ubuntu nhìn thấy.
7. Dùng lệnh xác định tổng dung lượng RAM và lượng RAM đang còn trống.
8. Dùng lệnh kiểm tra dung lượng phân vùng ổ đĩa gốc `/`.
9. Kiểm tra xem Ubuntu có ping ra được Internet (`google.com`) hay không.
10. Chạy lệnh cập nhật danh mục gói và nâng cấp toàn bộ hệ thống (`apt update && apt upgrade`).
11. Tạo một thư mục `~/workspace` và một file `~/workspace/lab01.txt` chứa nội dung `"Hoan thanh lab 01"`.
12. Thoát khỏi VM trở về macOS.
13. Dừng máy ảo `server`.
14. Khởi động lại máy ảo `server`.
15. Truy cập lại vào VM và kiểm tra xem file `~/workspace/lab01.txt` có còn nguyên vẹn không.
16. Thoát ra máy Mac, tiến hành `delete` và `purge` máy ảo sau khi hoàn thành.

---

## 🧠 14. Bộ Câu Hỏi Tự Kiểm Tra Bản Chất (Self-Assessment)

*Sau khi thực hành xong, hãy tự trả lời các câu hỏi sau để chắc chắn bạn đã hiểu sâu bản chất:*

1. *VM có phải là một máy tính vật lý riêng biệt không? Nó lấy CPU và RAM từ đâu?*
2. *Toàn bộ hệ thống tập tin của Ubuntu VM (ổ đĩa 20GB) thực chất là gì trên máy Mac?*
3. *Địa chỉ IP của VM do ai cấp? Tại sao máy Mac có thể ping được vào IP đó?*
4. *Nếu trong Ubuntu bạn chạy một web server ở cổng 3000, tại sao gõ `http://localhost:3000` trên trình duyệt máy Mac lại không xem được? Bạn phải gõ địa chỉ nào?*
5. *Lệnh `multipass stop` khác lệnh `multipass delete` và `multipass purge` như thế nào về mặt bảo toàn dữ liệu?*
6. *Vì sao lệnh `sudo apt update` không làm tăng dung lượng các phần mềm đã cài trên máy?*
7. *Multipass giải quyết vấn đề gì vượt trội hơn so với VirtualBox hay VMware truyền thống trên macOS?*
