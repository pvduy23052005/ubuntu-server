# 📘 Phần 02: Linux Commands – Làm Chủ Hệ Thống Qua Dòng Lệnh Cốt Lõi

> **Motto cốt lõi:**  
> *Đọc và hiểu output > Học thuộc cú pháp | Giải quyết vấn đề thực tế > Ghi nhớ hàng trăm câu lệnh.*  
> Chu trình tiếp cận: **Hiểu bản chất → Thực hành lệnh → Quan sát hệ thống thay đổi → Tự kiểm chứng → Xử lý tình huống thực tế.**

---

## 🎯 1. Mục Tiêu Của Phần Học

Mục tiêu của phần này không phải là bắt bạn ghi nhớ một danh sách dài các câu lệnh, mà là giúp bạn **hiểu cách hệ điều hành Linux vận hành thông qua giao diện dòng lệnh (CLI)**.

Sau khi hoàn thành phần này, bạn sẽ:
1. **Phân biệt rõ ràng:** Terminal, Shell (Bash/Zsh) và CLI Command.
2. **Nắm vững cấu trúc câu lệnh Linux:** Cách kết hợp `command`, `options` (cờ tham số) và `arguments` (đối số).
3. **Thao tác thành thạo Filesystem:** Điều hướng cây thư mục Linux, phân biệt đường dẫn tuyệt đối vs tương đối, hiểu ý nghĩa của `/`, `~`, `.`, `..`.
4. **Quản lý tệp tin và thư mục:** Tạo, xem, sửa, sao chép, di chuyển và xóa dữ liệu một cách an toàn.
5. **Khai thác nội dung tệp & Log:** Đọc file dung lượng lớn với `less`, theo dõi log thời gian thực với `tail -f`.
6. **Xử lý và lọc dữ liệu văn bản:** Kết hợp `grep`, `wc`, `sort`, `uniq` để tìm kiếm thông tin chính xác.
7. **Làm chủ luồng dữ liệu (I/O Streams):** Hiểu rõ `stdin`, `stdout`, `stderr`, kỹ thuật chuyển hướng (Redirection `>` / `>>`) và ghép lệnh bằng đường ống (Pipe `|`).
8. **Hiểu sâu về Phân quyền (Permissions):** Giải mã chính xác ý nghĩa của `rwx`, mã số phân quyền `755`, `644`, vai trò của `sudo` và người dùng `root`.
9. **Kiểm soát Tiến trình (Process) & Tài nguyên:** Biết cách tra cứu PID, theo dõi mức tiêu thụ CPU/RAM và tắt tiến trình bị treo.
10. **Chẩn đoán Mạng cơ bản:** Kiểm tra IP, bảng định tuyến, test kết nối mạng và phát hiện các cổng dịch vụ đang lắng nghe (`ss -tulpn`).

---

## 🧠 2. Bản Chất: Linux Command Line Hoạt Động Như Thế Nào?

Khi bạn gõ một dòng lệnh và ấn `Enter`, điều gì thực sự xảy ra bên dưới hệ thống?

```text
┌─────────────────────────────────────────────────────────────────────────┐
│                           NGƯỜI DÙNG GÕ LỆNH                            │
└────────────────────────────────────┬────────────────────────────────────┘
                                     │ (Nhập ký tự qua bàn phím)
                                     ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                      TERMINAL (Giao diện hiển thị)                      │
│   (macOS Terminal, iTerm2, Alacritty - Cửa sổ nhận ký tự & vẽ font chữ) │
└────────────────────────────────────┬────────────────────────────────────┘
                                     │ (Chuyển chuỗi ký tự thô)
                                     ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                        SHELL (Bộ thông dịch lệnh)                       │
│   (Bash, Zsh, Sh - Phân tích cú pháp, tìm file thực thi, gọi kernel)    │
└────────────────────────────────────┬────────────────────────────────────┘
                                     │ (Tạo Process & System Calls)
                                     ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                      LINUX KERNEL & OPERATING SYSTEM                    │
│   (Cấp phát CPU/RAM, đọc ghi ổ đĩa, gửi nhận gói tin mạng)              │
└────────────────────────────────────┬────────────────────────────────────┘
                                     │ (Dữ liệu trả về stdout / stderr)
                                     ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                    KẾT QUẢ HIỂN THỊ TRÊN MÀN HÌNH                       │
└─────────────────────────────────────────────────────────────────────────┘
```

### 2.1. Phân biệt Terminal, Shell và Command

#### 1. Terminal là gì?
* **Bản chất:** Terminal (Trình giả lập thiết bị đầu cuối) chỉ là một **ứng dụng đồ họa hiển thị cửa sổ**.
* **Nhiệm vụ:** Nhận tín hiệu gõ phím từ bạn, gửi tín hiệu đó đến Shell, và nhận các ký tự phản hồi từ Shell để vẽ lên màn hình.
* *Ví dụ:* Ứng dụng `Terminal.app` có sẵn trên Mac, `iTerm2`, `Kitty`.

#### 2. Shell là gì?
* **Bản chất:** Shell là một **chương trình thông dịch lệnh (Command Interpreter)** chạy bên trong Terminal.
* **Nhiệm vụ:** Đọc chuỗi ký tự bạn gõ, phân tích cú pháp, tìm xem chương trình bạn yêu cầu nằm ở đâu trên đĩa cứng, sau đó yêu cầu Linux Kernel tạo một tiến trình (Process) mới để chạy chương trình đó.
* *Ví dụ:* 
  * `zsh`: Shell mặc định trên macOS.
  * `bash` (Bourne Again Shell): Shell chuẩn mặc định trên hầu hết các bản phân phối Ubuntu Server.
  * `sh`: Bản shell nguyên bản cơ bản nhất của Unix.

#### 3. Command (Câu lệnh) thực sự là gì?
* Khi bạn gõ `ls`, Linux không tự "đoán" hành động. `ls` thực chất là một **file nhị phân thực thi (Executable Binary)** nằm tại thư mục `/usr/bin/ls`.
* **Luồng xử lý chi tiết:**
  1. Bạn gõ `ls` và bấm `Enter`.
  2. Shell tìm kiếm từ khóa `ls` trong các thư mục được liệt kê ở biến môi trường `$PATH` (ví dụ: `/usr/bin/`).
  3. Khi tìm thấy file `/usr/bin/ls`, Shell yêu cầu Linux Kernel nạp file này vào RAM và tạo một tiến trình mới (Process).
  4. Tiến trình `ls` đọc danh sách các tệp trong thư mục hiện tại từ filesystem.
  5. Tiến trình xuất kết quả ra luồng đầu ra chuẩn (`stdout`).
  6. Shell nhận kết quả và Terminal vẽ các tên tệp lên màn hình của bạn.

---

## 📐 3. Cấu Trúc Chuẩn Của Một Lệnh Linux

Hầu hết mọi câu lệnh trong Linux đều tuân theo công thức chuẩn sau:

```bash
command [options] [arguments]
```

```text
┌──────────────┬───────────────────┬──────────────────────────────────────┐
│ Cú pháp      │ Tên gọi           │ Ý nghĩa                              │
├──────────────┼───────────────────┼──────────────────────────────────────┤
│ command      │ Tên lệnh          │ Chương trình bạn muốn thực thi.       │
│ [options]    │ Cờ tùy chọn (Flags)│ Thay đổi cách thức hoạt động của lệnh.│
│ [arguments]  │ Đối số (Mục tiêu) │ Đối tượng mà lệnh tác động lên.      │
└──────────────┴───────────────────┴──────────────────────────────────────┘
```

### Ví dụ phân tích:
```bash
ls -lah /var/log
```
* `ls`: **Command** (Liệt kê thư mục).
* `-lah`: **Options** (Gộp từ 3 cờ: `-l` hiển thị chi tiết dạng danh sách, `-a` hiện cả file ẩn, `-h` định dạng kích thước dễ đọc KB/MB/GB).
* `/var/log`: **Argument** (Thư mục mục tiêu mà ta muốn `ls` kiểm tra).

### Tự đọc tài liệu hướng dẫn của lệnh:
Đừng bao giờ cố gắng học thuộc lòng mọi tùy chọn (options). Hãy rèn luyện kỹ năng tự tra cứu:
* **Xem tóm tắt nhanh:** `command --help` (vd: `ls --help`, `grep --help`).
* **Đọc hướng dẫn chi tiết đầy đủ (Manual Page):** `man command` (vd: `man ls`).
  *(Bấm phím `q` để thoát khỏi trang man).*

---

## 🌲 4. Hệ Thống Tập Tin Linux (Filesystem Hierarchy Standard - FHS)

Khác với Windows phân chia ổ đĩa thành các ký tự độc lập (`C:\`, `D:\`), toàn bộ hệ thống Linux được tổ chức theo **một cây thư mục duy nhất**, bắt nguồn từ gốc gọi là **Root (`/`)**.

```text
/ (Gốc - Root Directory)
├── bin / sbin  ──► Chứa các file thực thi nhị phân cốt lõi (ls, cp, ping, ip)
├── home        ──► Thư mục chứa dữ liệu cá nhân của các người dùng (/home/ubuntu)
├── root        ──► Thư mục cá nhân độc quyền của tài khoản quản trị tối cao (root)
├── etc         ──► TRUNG TÂM CẤU HÌNH (Chứa file config: nginx.conf, sshd_config)
├── var         ──► Dữ liệu biến đổi liên tục (Logs hệ thống tại /var/log, database)
├── tmp         ──► Thư mục lưu file tạm thời (Tự động dọn dẹp khi reboot)
├── usr         ──► Chứa các phần mềm, thư viện và ứng dụng do người dùng cài đặt
├── opt         ──► Chứa các gói phần mềm cài thêm từ bên thứ 3
└── dev / proc  ──► Các thư mục ảo đại diện cho thiết bị phần cứng & tiến trình RAM
```

> [!IMPORTANT]
> **Quy tắc nằm lòng của Quản trị viên Linux:**
> * Muốn sửa cấu hình hệ thống $\rightarrow$ Vào `/etc/`.
> * Muốn kiểm tra nguyên nhân lỗi / xem log ứng dụng $\rightarrow$ Vào `/var/log/`.
> * Muốn lưu code / làm việc hàng ngày $\rightarrow$ Nằm trong `/home/<username>/`.

---

## 🧭 5. Điều Hướng Cây Thư Mục (Navigation)

### 5.1. Khái niệm cốt lõi:
* `/`: **Root** (Gốc cao nhất của toàn bộ hệ điều hành).
* `~`: **Home directory** của người dùng hiện tại (ví dụ: `/home/ubuntu`).
* `.`: Đại diện cho **thư mục hiện tại đang đứng**.
* `..`: Đại diện cho **thư mục cha (cấp trên liền kề)**.
* **Đường dẫn tuyệt đối (Absolute Path):** Bắt đầu bằng dấu `/` từ gốc (vd: `/var/log/syslog`).
* **Đường dẫn tương đối (Relative Path):** Tính từ vị trí hiện tại đang đứng (vd: `../etc/hosts`).

### 5.2. Thực hành chuỗi lệnh điều hướng

```bash
# 1. Kiểm tra vị trí hiện tại
pwd
# Kết quả: /home/ubuntu

# 2. Liệt kê các file trong thư mục hiện tại
ls -la

# 3. Lên thư mục cha (/home)
cd ..
pwd
# Kết quả: /home

# 4. Lên tiếp thư mục gốc (Root /)
cd ..
pwd
# Kết quả: /

# 5. Đi thẳng vào thư mục cấu hình bằng đường dẫn tuyệt đối
cd /etc
pwd
# Kết quả: /etc

# 6. Quay trở về thư mục Home của user bất kể đang đứng ở đâu
cd ~
pwd
# Kết quả: /home/ubuntu
```

---

## 📁 6. Tạo và Quản Lý File & Directory

```text
┌──────────────┬─────────────────────────────────────────────────────────────┐
│ Câu lệnh     │ Thao tác thực tế trong Filesystem                           │
├──────────────┼─────────────────────────────────────────────────────────────┤
│ mkdir        │ Tạo một hoặc nhiều thư mục mới.                             │
│ touch        │ Tạo một file rỗng mới (hoặc cập nhật timestamp của file cũ). │
│ cp           │ Tạo một bản sao chép (Copy) của file/thư mục.               │
│ mv           │ Di chuyển (Move) hoặc Đổi tên (Rename) file/thư mục.        │
│ rm           │ Xóa vĩnh viễn file khỏi hệ thống tập tin.                   │
│ rmdir        │ Xóa thư mục rỗng.                                           │
└──────────────┴─────────────────────────────────────────────────────────────┘
```

### 6.1. Thực hành thực tế theo kịch bản

```bash
# Bước 1: Tạo cây thư mục dự án lồng nhau (-p: parent)
mkdir -p my-project/src

# Bước 2: Tạo file code rỗng
touch my-project/src/index.js

# Bước 3: Kiểm tra cấu trúc vừa tạo
ls -R my-project

# Bước 4: Sao chép file làm bản backup (-r: recursive nếu sao chép cả thư mục)
cp my-project/src/index.js my-project/src/index.backup.js

# Bước 5: Đổi tên file (Rename)
mv my-project/src/index.backup.js my-project/src/index.old.js

# Bước 6: Di chuyển file ra thư mục ngoài
mv my-project/src/index.old.js my-project/

# Bước 7: Xóa file đơn lẻ
rm my-project/index.old.js

# Bước 8: Xóa toàn bộ thư mục cùng mọi file bên trong (-r: recursive, -f: force)
rm -rf my-project
```

> [!CAUTION]
> **Cực kỳ nguy hiểm:** Lệnh `rm` trong Linux **không có Thùng rác (Recycle Bin/Trash)**. File bị xóa bởi `rm` sẽ biến mất vĩnh viễn khỏi filesystem ngay lập tức. Hãy luôn cẩn thận tuyệt đối khi dùng cờ `rm -rf`.

---

## 📖 7. Đọc Nội Dung File & Theo Dõi Log

Khi vận hành server, bạn liên tục phải kiểm tra các file cấu hình và file nhật ký (logs).

```text
┌──────────────┬─────────────────────────────────────────────────────────────┐
│ Câu lệnh     │ Khi nào nên sử dụng?                                        │
├──────────────┼─────────────────────────────────────────────────────────────┤
│ cat          │ Đọc toàn bộ nội dung file ngắn (dưới 50 dòng).              │
│ less         │ Đọc file dài nhiều trang (cho phép cuộn lên/xuống, tìm kiếm)│
│ head -n 10   │ Chỉ xem 10 dòng đầu tiên của file.                          │
│ tail -n 10   │ Chỉ xem 10 dòng cuối cùng của file.                         │
│ tail -f      │ "Live-stream" log: Giữ màn hình mở và in ra dòng mới ngay   │
│              │ khi có dữ liệu vừa được ghi vào file.                       │
└──────────────┴─────────────────────────────────────────────────────────────┘
```

### 7.1. Thực hành:

```bash
# Xem thông tin hệ điều hành (file ngắn)
cat /etc/os-release

# Đọc file cấu hình dài có phân trang (Bấm Space để cuộn, phím 'q' để thoát)
less /etc/services

# Xem 5 dòng đầu tiên của file mật khẩu
head -n 5 /etc/passwd

# Xem 5 dòng log hệ thống gần nhất
sudo tail -n 5 /var/log/auth.log

# BẬT CHẾ ĐỘ LIVE DEBUG LOG (Cực kỳ quan trọng khi test API/Nginx):
sudo tail -f /var/log/syslog
# (Nhấn Ctrl + C để dừng theo dõi)
```

---

## 🔍 8. Xử Lý và Tìm Kiếm Dữ Liệu Văn Bản (Text Processing)

Linux nổi tiếng với triết lý: *"Mọi thứ cấu hình và nhật ký đều là văn bản thuần (Plain Text)"*. Do đó, thành thạo các công cụ xử lý văn bản là chìa khóa để làm chủ hệ thống.

```text
┌──────────────┬─────────────────────────────────────────────────────────────┐
│ Câu lệnh     │ Chức năng chính                                             │
├──────────────┼─────────────────────────────────────────────────────────────┤
│ grep         │ Tìm kiếm các dòng khớp với một chuỗi ký tự hoặc Regex.       │
│ wc           │ Đếm số dòng (-l), số từ (-w), số ký tự (-c).                │
│ sort         │ Sắp xếp các dòng theo thứ tự bảng chữ cái hoặc số học (-n). │
│ uniq         │ Lọc bỏ các dòng trùng lặp liên tiếp nhau (-c: đếm số lần).  │
│ cut          │ Cắt trích xuất các cột dữ liệu theo ký tự phân tách (-d, -f)│
└──────────────┴─────────────────────────────────────────────────────────────┘
```

### 8.1. Thực hành tìm kiếm:

```bash
# Tạo một file log giả lập để thực hành
cat << 'EOF' > app.log
2026-08-19 [INFO] Server started on port 3000
2026-08-19 [ERROR] Database connection failed: timeout
2026-08-19 [INFO] User logged in: admin
2026-08-19 [ERROR] Invalid token provided
2026-08-19 [WARNING] Memory usage above 80%
2026-08-19 [ERROR] Database connection failed: refused
EOF

# 1. Tìm tất cả các dòng có chứa chữ "ERROR"
grep "ERROR" app.log

# 2. Tìm không phân biệt chữ hoa/thường (-i) và in kèm số dòng (-n)
grep -in "database" app.log

# 3. Tìm các dòng KHÔNG chứa chữ "INFO" (-v: invert match)
grep -v "INFO" app.log
```

---

## 🚰 9. Đường Ống Lệnh (Pipe `|`): Sức Mạnh Kết Hợp

### 9.1. Bản chất của Pipe
Pipe (`|`) cho phép bạn lấy **đầu ra chuẩn (`stdout`) của tiến trình phía trước** làm **đầu vào chuẩn (`stdin`) cho tiến trình phía sau**, tạo thành một dây chuyền xử lý dữ liệu liên hoàn.

```text
┌──────────────────────┐   stdout   ┌──────────────────────┐   stdout   ┌──────────────────────┐
│      Command 1       ├───────────►│      Command 2       ├───────────►│      Command 3       │
│ (grep "ERROR" app.log│            │        (sort)        │            │       (wc -l)        │
└──────────────────────┘            └──────────────────────┘            └──────────────────────┘
```

### 9.2. Thực hành kết hợp:

```bash
# Bài toán 1: Đếm xem có bao nhiêu lỗi [ERROR] trong file log?
grep "ERROR" app.log | wc -l
# Kết quả: 3

# Bài toán 2: Xem danh sách các tiến trình hệ thống và lọc ra tiến trình ssh
ps aux | grep ssh

# Bài toán 3: Kiểm tra xem cổng nào đang mở và lọc trạng thái LISTEN
ss -tulpn | grep LISTEN
```

---

## 🔄 10. Chuyển Hướng Luồng Dữ Liệu (Redirection & I/O Streams)

Trong Linux, mọi tiến trình khi khởi chạy đều được gắn liền với **3 luồng truyền thông tiêu chuẩn (Standard Streams)**:

```text
                             ┌──► stdout (Mã luồng: 1) ──► Màn hình Terminal
                             │
Dữ liệu bàn phím ──► stdin ──┴── Tiến trình (Process)
 (Mã luồng: 0)               │
                             └──► stderr (Mã luồng: 2) ──► Màn hình Terminal
```

### 10.1. Các toán tử chuyển hướng:

* `>`: Chuyển hướng `stdout` vào file (**Ghi đè - Overwrite** toàn bộ nội dung cũ).
* `>>`: Chuyển hướng `stdout` vào file (**Ghi nối tiếp - Append** vào cuối file).
* `<`: Lấy dữ liệu từ file nạp vào `stdin` của lệnh.
* `2>`: Chuyển hướng riêng luồng thông báo lỗi `stderr` vào file.
* `2>&1` hoặc `&>`: Gom cả luồng kết quả đúng (`stdout`) và luồng báo lỗi (`stderr`) vào cùng một file.

### 10.2. Thực hành:

```bash
# 1. Ghi đè file
echo "Phien ban 1.0" > version.txt
cat version.txt

# 2. Ghi nối tiếp vào file
echo "Cap nhat ngay 2026" >> version.txt
cat version.txt

# 3. Tách biệt kết quả đúng và thông báo lỗi:
# Lệnh 'ls /etc /duong-dan-sai' sẽ vừa có output đúng vừa có lỗi:
ls /etc /duong-dan-sai > success.txt 2> error.txt

cat success.txt   # Chứa danh sách các file trong /etc
cat error.txt     # Chứa câu báo lỗi: "No such file or directory"
```

---

## 🔒 11. Phân Quyền Tập Tin & Người Dùng (Permissions)

Mọi tệp tin và thư mục trong Linux đều được bảo vệ nghiêm ngặt bởi mô hình phân quyền đa người dùng.

### 11.1. Giải mã chuỗi phân quyền từ lệnh `ls -l`

```bash
ls -l app.txt
-rw-r--r-- 1 ubuntu ubuntu 1200 Aug 19 09:00 app.txt
```

```text
 ┌── Loại tệp (-: file, d: directory, l: symlink)
 │  ┌── Quyền của Chủ sở hữu (Owner / User)
 │  │   ┌── Quyền của Nhóm (Group)
 │  │   │   ┌── Quyền của Những người khác (Others / World)
 ▼  ▼   ▼   ▼
 -  rw- r-- r--   1   ubuntu    ubuntu   1200   app.txt
                      ▲         ▲
                      │         └── Tên Group sở hữu
                      └──────────── Tên User sở hữu
```

### 11.2. Ý nghĩa của 3 ký tự `r`, `w`, `x` và Mã số Nhị phân:

| Quyền hạn | Ký hiệu | Giá trị số (Octal) | Ý nghĩa đối với File | Ý nghĩa đối với Directory |
| :--- | :---: | :---: | :--- | :--- |
| **Read** | `r` | **4** | Được mở và đọc nội dung file. | Được liệt kê danh sách file bên trong (`ls`). |
| **Write** | `w` | **2** | Được chỉnh sửa hoặc xóa nội dung file.| Được tạo mới, đổi tên hoặc xóa file trong thư mục.|
| **Execute**| `x` | **1** | Được chạy file như một chương trình/script.| Được truy cập (vào lệnh `cd`) vào thư mục. |
| **None** | `-` | **0** | Không có quyền. | Không có quyền. |

### 11.3. Giải mã các con số phân quyền kinh điển:

* **`chmod 755 file`:**
  * Owner: $4 + 2 + 1 = \mathbf{7}$ (Full: Read, Write, Execute).
  * Group: $4 + 0 + 1 = \mathbf{5}$ (Read, Execute).
  * Others: $4 + 0 + 1 = \mathbf{5}$ (Read, Execute).
  * *Thường dùng cho:* File Script chạy được, thư mục Web công khai.
* **`chmod 644 file`:**
  * Owner: $4 + 2 + 0 = \mathbf{6}$ (Read, Write).
  * Group: $4 + 0 + 0 = \mathbf{4}$ (Read only).
  * Others: $4 + 0 + 0 = \mathbf{4}$ (Read only).
  * *Thường dùng cho:* File tài liệu văn bản, file cấu hình thông thường.
* **`chmod 600 file`:**
  * Owner: $4 + 2 + 0 = \mathbf{6}$ (Read, Write).
  * Group & Others: $\mathbf{0}$ (Tuyệt đối cấm).
  * *Thường dùng cho:* File SSH Private Key, file chứa mật khẩu bí mật.

### 11.4. Thực hành phân quyền:

```bash
# Tạo một shell script đơn giản
echo 'echo "Xin chao Linux Server!"' > run.sh

# Thử chạy file (Sẽ bị từ chối quyền: Permission denied)
./run.sh

# Cấp quyền thực thi (+x hoặc 755)
chmod +x run.sh

# Chạy lại thành công
./run.sh

# Đổi chủ sở hữu file sang user khác (Yêu cầu quyền sudo)
# sudo chown <user>:<group> <file>
sudo chown ubuntu:ubuntu run.sh
```

---

## 🛡️ 12. Quản Trị Đặc Quyền: `sudo`, `root` và `su`

```text
┌─────────────────────────────────────────────────────────────┐
│                      ROOT SUPERUSER (UID: 0)                │
│             Toàn quyền tuyệt đối, có thể xóa toàn bộ OS     │
└──────────────────────────────▲──────────────────────────────┘
                               │
               ┌───────────────┴───────────────┐
               │                               │
        sudo command                           su
  (Mượn quyền root tạm thời             (Chuyển hẳn sang root,
   để chạy duy nhất 1 lệnh)              dễ gây mất an toàn)
               │                               │
┌──────────────┴───────────────────────────────┴──────────────┐
│                   STANDARD USER (User: ubuntu)              │
│       Chỉ được chỉnh sửa dữ liệu trong thư mục /home/       │
└─────────────────────────────────────────────────────────────┘
```

* **`root` là gì?** Tài khoản quản trị tối cao của hệ điều hành Linux (tương tự Administrator trên Windows). `root` có thể đọc, sửa, xóa bất kỳ file nào trong hệ thống mà không bị ngăn cản.
* **`sudo` (Superuser Do) là gì?** Cho phép một người dùng thông thường đã được cấp phép thực thi một câu lệnh cụ thể với đặc quyền của `root`.
* **Nguyên tắc bảo mật:** **Không bao giờ đăng nhập trực tiếp bằng root** cho mọi công việc hàng ngày. Luôn dùng user thông thường và chỉ thêm `sudo` trước các lệnh cần can thiệp hệ thống (như cài đặt phần mềm, sửa file trong `/etc/`).

---

## ⚙️ 13. Quản Lý Tiến Trình Hệ Thống (Process Management)

Mọi chương trình đang chạy trên Linux đều được biểu diễn dưới dạng một **Tiến trình (Process)** và được hệ điều hành gán một mã định danh duy nhất gọi là **PID (Process ID)**.

```text
┌──────────────┬─────────────────────────────────────────────────────────────┐
│ Câu lệnh     │ Chức năng                                                   │
├──────────────┼─────────────────────────────────────────────────────────────┤
│ ps aux       │ Chụp ảnh toàn bộ mọi tiến trình đang chạy trong hệ thống.   │
│ top          │ Bảng điều khiển tương tác theo dõi CPU/RAM của tiến trình.  │
│ htop         │ Phiên bản nâng cấp đẹp mắt, trực quan của `top`.            │
│ pgrep <tên>  │ Tìm nhanh mã PID của một tiến trình theo tên.               │
│ kill <PID>   │ Gửi tín hiệu yêu cầu một tiến trình dừng lại (SIGTERM 15).   │
│ kill -9 <PID>│ Cưỡng chế tiêu diệt tiến trình ngay lập tức (SIGKILL 9).    │
└──────────────┴─────────────────────────────────────────────────────────────┘
```

### 13.1. Thực hành quản lý tiến trình:

```bash
# 1. Chạy một chương trình giả lập chạy ngầm trong 500 giây (&: chạy background)
sleep 500 &

# 2. Tìm mã PID của tiến trình sleep vừa tạo
ps aux | grep sleep

# 3. Hoặc tìm nhanh PID bằng pgrep
pgrep sleep

# 4. Gửi tín hiệu dừng tiến trình một cách lịch sự (Graceful Termination)
kill $(pgrep sleep)

# 5. Kiểm tra lại xem tiến trình đã biến mất chưa
ps aux | grep sleep
```

---

## 📊 14. Kiểm Tra & Giám Sát Tài Nguyên Hệ Thống (System Resources)

Khi server gặp sự cố hoặc chạy chậm, đây là bộ lệnh đầu tiên bạn phải sử dụng để chẩn đoán:

```bash
# 1. Kiểm tra số nhân và thông số kiến trúc CPU
nproc
lscpu

# 2. Kiểm tra bộ nhớ RAM và Swap theo định dạng dễ đọc (Gi/Mi)
free -h

# 3. Kiểm tra dung lượng ổ đĩa của tất cả phân vùng (Disk Free)
df -h

# 4. Kiểm tra dung lượng thực tế của một thư mục cụ thể (Disk Usage)
du -sh /var/log

# 5. Xem thời gian máy chủ đã chạy liên tục (Uptime) và Tải trung bình (Load Average)
uptime
```

---

## 🌐 15. Bộ Lệnh Kiểm Tra Mạng Cốt Lõi (Networking Commands)

```text
┌──────────────┬─────────────────────────────────────────────────────────────┐
│ Câu lệnh     │ Mục đích chẩn đoán                                          │
├──────────────┼─────────────────────────────────────────────────────────────┤
│ ip addr      │ Xem toàn bộ card mạng và địa chỉ IP của server.             │
│ ip route     │ Xem bảng định tuyến (Routing Table) và Default Gateway.     │
│ ping <host>  │ Kiểm tra kết nối tầng mạng (ICMP) đến một địa chỉ/domain.  │
│ curl -I <url>│ Gửi HTTP Request để kiểm tra phản hồi từ Web Server/API.     │
│ ss -tulpn    │ Liệt kê toàn bộ các cổng mạng TCP/UDP đang MỞ và lắng nghe. │
└──────────────┴─────────────────────────────────────────────────────────────┘
```

### 15.1. Giải mã lệnh kiểm tra cổng mạng kinh điển `ss -tulpn`:
* `-t`: Lọc giao thức **TCP**.
* `-u`: Lọc giao thức **UDP**.
* `-l`: Chỉ hiện các socket đang **LISTEN** (Lắng nghe kết nối đến).
* `-p`: Hiển thị tên chương trình và **Process PID** đang chiếm giữ cổng đó.
* `-n`: Hiển thị số hiệu cổng dạng số (Numeric) thay vì tên dịch vụ (hiện `80` thay vì `http`).

```bash
sudo ss -tulpn
```

---

## 📦 16. Quản Lý Gói Phần Mềm Với APT (Package Management)

```bash
# 1. Cập nhật chỉ mục danh mục phần mềm mới nhất
sudo apt update

# 2. Cài đặt một công cụ mới (ví dụ: htop)
sudo apt install -y htop

# 3. Kiểm tra xem phần mềm đã hoạt động chưa
htop
# (Bấm phím 'q' hoặc F10 để thoát)

# 4. Gỡ bỏ phần mềm khỏi hệ thống
sudo apt remove -y htop
```

---

## 🌲 17. Biến Môi Trường (Environment Variables)

* **Biến môi trường là gì?** Là các cặp giá trị `KEY=VALUE` được nạp vào bộ nhớ của Shell, giúp hệ điều hành và các ứng dụng (NodeJS, Python, Go) biết được cấu hình môi trường đang chạy.
* **Biến `$PATH` là gì?** Là danh sách các đường dẫn thư mục ngăn cách nhau bởi dấu hai chấm `:`. Khi bạn gõ một lệnh, Shell sẽ lần lượt tìm kiếm file thực thi trong các thư mục của `$PATH`.

### 17.1. Thực hành:

```bash
# 1. Xem biến PATH hiện tại của hệ thống
echo $PATH

# 2. Khởi tạo một biến môi trường tạm thời cho phiên làm việc hiện tại
export NODE_ENV=production
export PORT=3000

# 3. Đọc giá trị biến môi trường
echo "Ứng dụng đang chạy ở môi trường: $NODE_ENV trên cổng: $PORT"

# 4. Xem toàn bộ biến môi trường đang có
printenv | head -n 10
```

---

## 🚨 18. Giải Quyết 5 Tình Huống Thực Tế Trên Server (Real-World Scenarios)

Dưới đây là cách phối hợp các câu lệnh để xử lý sự cố thực tế:

### 🔴 Tình huống 1: "Server báo động hết ổ cứng. Làm sao tìm thư mục đang chiếm dung lượng lớn nhất?"
```bash
# Bước 1: Xác định phân vùng nào bị đầy
df -h

# Bước 2: Quét top 10 thư mục/file chiếm dung lượng lớn nhất trong /var
sudo du -ah /var | sort -rh | head -n 10
```

---

### 🔴 Tình huống 2: "Khởi động ứng dụng NodeJS báo lỗi: `Error: listen EADDRINUSE: address already in use :::3000`. Phải xử lý thế nào?"
```bash
# Bước 1: Tìm xem Process nào đang chiếm giữ cổng 3000
sudo ss -tulpn | grep :3000
# (Hoặc dùng: sudo lsof -i :3000)

# Bước 2: Nhìn thấy PID (ví dụ PID = 4125), tiến hành tắt process đó:
sudo kill -9 4125
```

---

### 🔴 Tình huống 3: "Kiểm tra xem Web Server Nginx có đang thực sự chạy ngầm không?"
```bash
ps aux | grep nginx
```

---

### 🔴 Tình huống 4: "Khách hàng phản ánh ứng dụng lỗi. Cần đếm số lượng lỗi `500 Internal Server Error` xuất hiện trong log hôm nay."
```bash
grep "500" /var/log/nginx/access.log | wc -l
```

---

### 🔴 Tình huống 5: "Chạy script báo lỗi `bash: ./deploy.sh: Permission denied`."
```bash
# Bước 1: Kiểm tra quyền hạn hiện tại của file
ls -l deploy.sh

# Bước 2: Cấp quyền thực thi
chmod +x deploy.sh

# Bước 3: Chạy lại
./deploy.sh
```

---

## 🧪 19. Bài Thực Hành: Server Investigation Lab (Thử Thách 20 Nhiệm Vụ)

*Hãy truy cập vào Ubuntu Server VM của bạn (`multipass shell server`) và tự tìm câu trả lời cho 20 câu hỏi chẩn đoán sau mà không xem trước gợi ý:*

1. Tên user hiện tại của bạn là gì? Thư mục cá nhân của bạn nằm ở đâu?
2. Server của bạn đang chạy nhân Linux Kernel phiên bản bao nhiêu?
3. Bản phân phối Ubuntu chính xác là bản nào (Codename là gì)?
4. Server VM của bạn đang có bao nhiêu nhân vCPU?
5. Tổng dung lượng RAM của server là bao nhiêu và hiện còn trống bao nhiêu MB?
6. Phân vùng gốc `/` đã sử dụng bao nhiêu phần trăm (%) dung lượng?
7. Địa chỉ IPv4 của server là gì?
8. Địa chỉ IP của Gateway mạng là bao nhiêu?
9. Kiểm tra xem server có gửi nhận được gói tin đến `google.com` không?
10. Hiện tại có những cổng mạng nào đang ở trạng thái `LISTEN` trên server?
11. Tiến trình nào đang chiếm nhiều bộ nhớ RAM nhất tại thời điểm hiện tại?
12. Tìm tất cả các file có phần mở rộng `.log` nằm trong thư mục `/var/log`.
13. Đếm xem có bao nhiêu dòng có chứa chữ `fail` hoặc `error` trong file `/var/log/auth.log` (hoặc `/var/log/syslog`)?
14. Tạo một thư mục `/tmp/investigation` và tạo file `/tmp/investigation/report.txt`.
15. Ghi thông tin `uptime` của server vào cuối file `report.txt` bằng toán tử `>>`.
16. Thay đổi phân quyền của file `report.txt` thành: Chỉ có chủ sở hữu mới có quyền đọc và ghi (quyền `600`).
17. Chạy lệnh `sleep 600 &` ở chế độ background.
18. Tìm mã PID của tiến trình `sleep` vừa chạy.
19. Dùng lệnh `kill` để dừng tiến trình `sleep` đó.
20. Xóa hoàn toàn thư mục `/tmp/investigation` cùng toàn bộ nội dung bên trong.

---

## 🗺️ 20. Sơ Đồ Tổng Hợp Kiến Thức Cần Nhớ

```text
LINUX COMMAND LINE CỐT LÕI
│
├── 📂 Filesystem (Điều hướng & Quản lý tệp)
│   ├── pwd, cd (., .., ~, /)
│   ├── ls -lah
│   ├── mkdir -p, touch
│   └── cp -r, mv, rm -rf
│
├── 📖 File Content (Đọc nội dung)
│   ├── cat (file ngắn)
│   ├── less (file dài, phân trang)
│   └── head, tail -f (Live debug log)
│
├── 🔍 Text Processing (Xử lý chuỗi)
│   ├── grep -rnw (Tìm kiếm mẫu)
│   ├── wc -l (Đếm dòng)
│   └── sort, uniq, cut
│
├── 🔒 Permission & User (Bảo mật & Phân quyền)
│   ├── ls -l (rwx)
│   ├── chmod 755 / 644 / 600
│   ├── chown user:group
│   └── sudo, whoami, id
│
├── ⚙️ Process & System (Tiến trình & Tài nguyên)
│   ├── ps aux, top, htop
│   ├── kill <PID>, pgrep
│   ├── free -h, df -h, du -sh
│   └── uptime, nproc, lscpu
│
├── 🌐 Networking (Mạng & Dịch vụ)
│   ├── ip addr, ip route
│   ├── ping, curl
│   └── ss -tulpn (Kiểm tra cổng LISTEN)
│
└── 🚰 Stream Composition (Ghép nối luồng dữ liệu)
    ├── | (Pipe: Nối stdout -> stdin)
    ├── > (Ghi đè stdout)
    ├── >> (Ghi nối tiếp stdout)
    └── 2> (Chuyển hướng lỗi stderr)
```

---

## 📌 Tóm Tắt & Các Lỗi Kinh Điển Cần Tránh

1. **Nhầm lẫn giữa `>` và `>>`:** Dùng nhầm `>` khi muốn ghi thêm log sẽ làm **xóa sạch toàn bộ dữ liệu cũ** của file. Hãy luôn cẩn thận dùng `>>` khi append.
2. **Không phân biệt được `localhost` và IP mạng:** `localhost` trong VM là chính VM, `localhost` ngoài Mac là chính Mac.
3. **Thói quen lạm dụng `sudo`:** Không gắn `sudo` bừa bãi vào mọi câu lệnh. Chỉ dùng khi gặp lỗi quyền hạn can thiệp vào file hệ thống.
4. **Hiểu rõ luồng lệnh thay vì gõ vẹt:** Khi gặp sự cố, hãy bình tĩnh đọc thông báo lỗi (`stderr`), sử dụng `tail -f`, `grep`, `ss -tulpn` và `ps aux` để truy tìm gốc rễ vấn đề.
