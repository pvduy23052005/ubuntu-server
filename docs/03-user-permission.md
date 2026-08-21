# 📘 Phần 03: User & Permission – Quản Lý Người Dùng & Phân Quyền

> **Motto cốt lõi:**  
> *Hiểu tại sao được phép hoặc bị từ chối > Học vẹt mã số chmod | Tự tạo lỗi rồi tự sửa > Làm theo hướng dẫn một cách thụ động.*  
> Chu trình chuẩn: **Khái niệm → Nó thay đổi cái gì trong hệ thống → Thực hành → Kiểm tra kết quả.**

---

## 🎯 1. Mục Tiêu Của Phần Học

Trong môi trường máy chủ Linux, bảo mật bắt đầu từ **tài khoản người dùng (User)** và **quyền hạn tập tin (File Permissions)**.

Sau khi hoàn thành bài học này, bạn sẽ:
1. **Hiểu bản chất phân quyền:** Biết chính xác Linux dựa vào đâu để cho phép hoặc chặn một người dùng đọc, sửa hoặc thực thi một file.
2. **Quản lý User & Group chuyên nghiệp:** Tự tạo, chỉnh sửa, gán nhóm và xóa tài khoản người dùng một cách an toàn.
3. **Làm chủ quyền sở hữu (Ownership):** Hiểu rõ mối liên hệ giữa **Owner** và **Group**, thay đổi chủ sở hữu bằng `chown` và `chgrp`.
4. **Giải mã hoàn toàn `chmod`:** Phân biệt rõ cách dùng dạng ký hiệu (`u+x`, `g-w`) và dạng số bát phân (`755`, `644`, `700`, `600`) mà không cần học thuộc lòng.
5. **Nắm vững sự khác biệt sống còn:** Hiểu quyền `rwx` trên **Tập tin (File)** khác hoàn toàn thế nào so với trên **Thư mục (Directory)** (Đặc biệt là quyền xóa file và quyền `cd`).
6. **Sử dụng `sudo` chuẩn mực:** Biết cách kiểm tra quyền hạn (`sudo -l`) và hiểu tại sao không bao giờ nên làm việc trực tiếp bằng tài khoản `root`.
7. **Thực hành bài Lab phân quyền thực tế:** Xây dựng cấu trúc thư mục `/project` đa người dùng (Frontend, Backend, Developers Group) với phân quyền cô lập chuẩn production.

---

## 👤 2. User, Group & Tài Khoản `root`

### 2.1. Khái niệm cốt lõi

```text
┌─────────────────────────────────────────────────────────────┐
│                    MÔ HÌNH NGƯỜI DÙNG LINUX                 │
├──────────────────────────────┬──────────────────────────────┤
│  User (Người dùng)           │  Group (Nhóm người dùng)     │
│  - Đại diện cho 1 con người  │  - Tập hợp nhiều User        │
│    hoặc 1 dịch vụ (Nginx)    │  - Dùng để cấp quyền chung   │
│  - Định danh bằng: UID       │  - Định danh bằng: GID       │
└──────────────────────────────┴──────────────────────────────┘
```

* **User (Người dùng):** Là một tài khoản dùng để đăng nhập hoặc để chạy một tiến trình/dịch vụ nền (ví dụ: user `ubuntu`, user `nginx`, user `postgres`). Mỗi user có một mã số định danh duy nhất là **UID (User ID)**.
* **Group (Nhóm):** Là một tập hợp các user có chung nhu cầu truy cập tài nguyên. Một user có thể thuộc về nhiều group khác nhau. Mỗi group có mã định danh là **GID (Group ID)**.
* **Linux dùng User/Group để làm gì?** Để **cô lập dữ liệu** và **ngăn chặn truy cập trái phép**. User này không thể tự ý xem, xóa hoặc can thiệp vào file của User khác nếu không được cấp quyền.
* **`root` là gì?** Là tài khoản quản trị tối cao (Superuser) có `UID = 0`. `root` đứng ngoài mọi quy tắc phân quyền: có thể đọc, sửa, xóa bất kỳ file nào trong toàn bộ hệ điều hành.

---

### 2.2. Kiểm tra thông tin người dùng hiện tại

#### Câu lệnh: `whoami`, `id`, `groups`
* **Nó là gì?** Các công cụ kiểm tra danh tính và nhóm của tài khoản đang đăng nhập.
* **Nó tác động vào đâu?** Chỉ đọc thông tin từ `/etc/passwd` và `/etc/group` trong bộ nhớ, không làm thay đổi hệ thống.

```bash
# Xem tên user hiện tại
whoami
# Kết quả: ubuntu

# Xem chi tiết UID, GID và tất cả các nhóm mà user đang tham gia
id
# Kết quả mẫu: uid=1000(ubuntu) gid=1000(ubuntu) groups=1000(ubuntu),4(adm),27(sudo),...

# Xem danh sách các nhóm của user hiện tại
groups
```

---

### 2.3. Tạo và Xóa Người Dùng (`adduser` & `userdel`)

#### 1. Tạo user mới với `adduser`
* **Nó là gì?** Lệnh cấp cao giúp tạo user mới kèm theo: Thư mục cá nhân (`/home/<user>`), thiết lập mật khẩu và gán shell mặc định (`/bin/bash`).
* **Nó thay đổi gì trong hệ thống?**
  - Ghi thêm dòng mới vào `/etc/passwd` (thông tin user).
  - Ghi hash mật khẩu vào `/etc/shadow`.
  - Tạo group mới cùng tên trong `/etc/group`.
  - Tạo thư mục `/home/<tên_user>` và phân quyền cho user đó.

```bash
# Tạo user mới tên là "dev" (Hệ thống sẽ yêu cầu nhập mật khẩu và thông tin)
sudo adduser dev
```

*Quan sát kết quả tạo user:*
```text
Adding user `dev' ...
Adding new group `dev' (1001) ...
Adding new user `dev' (1001) with group `dev' ...
Creating home directory `/home/dev' ...
Copying files from `/etc/skel' ...
New password: 
```

#### 2. Xóa user với `userdel`
* **Nó là gì?** Lệnh xóa tài khoản người dùng khỏi hệ thống.
* **Cờ quan trọng:** `-r` (Remove home directory) – Xóa luôn thư mục `/home/<user>` của người dùng đó.

```bash
# Xóa user "dev" kèm toàn bộ thư mục cá nhân
sudo userdel -r dev
```

---

## 🏷️ 3. Quyền Sở Hữu Tập Tin (File Ownership)

Mỗi file và thư mục trong Linux **luôn luôn thuộc về đúng 1 User (Owner)** và **đúng 1 Group**.

```text
┌─────────────────────────────────────────────────────────────┐
│                       FILE: app.js                          │
├──────────────────────────────┬──────────────────────────────┤
│  Owner (Chủ sở hữu)          │  Group (Nhóm sở hữu)         │
│  ubuntu                      │  developers                  │
│  (Người tạo ra file)         │  (Nhóm được phép làm việc)   │
└──────────────────────────────┴──────────────────────────────┘
```

### 3.1. Đọc quyền sở hữu từ lệnh `ls -l`

```bash
touch demo.txt
ls -l demo.txt
```

**Kết quả hiển thị:**
```text
-rw-rw-r-- 1 ubuntu ubuntu 0 Aug 21 08:15 demo.txt
             ▲      ▲
             │      └── Group sở hữu (ubuntu)
             └───────── Owner sở hữu (ubuntu)
```

---

### 3.2. Thay đổi Owner và Group (`chown` & `chgrp`)

```text
┌──────────────┬─────────────────────────────────────────────────────────────┐
│ Câu lệnh     │ Chức năng & Tác động                                        │
├──────────────┼─────────────────────────────────────────────────────────────┤
│ chown        │ Change Owner: Thay đổi User sở hữu (hoặc cả User & Group). │
│ chgrp        │ Change Group: Thay đổi riêng Group sở hữu của file.        │
└──────────────┴─────────────────────────────────────────────────────────────┘
```

#### Thực hành thay đổi quyền sở hữu:

```bash
# 1. Tạo lại user 'dev' để thực hành
sudo adduser --disabled-password --gecos "" dev

# 2. Tạo một file thử nghiệm
touch test-owner.txt
ls -l test-owner.txt
# Hiện tại: Owner = ubuntu, Group = ubuntu

# 3. Đổi chủ sở hữu (Owner) sang user 'dev' (cần sudo)
sudo chown dev test-owner.txt
ls -l test-owner.txt
# Kết quả: Owner đã chuyển thành 'dev'

# 4. Đổi cả Owner và Group cùng một lúc bằng cú pháp: user:group
sudo chown dev:sudo test-owner.txt
ls -l test-owner.txt
# Kết quả: Owner = dev, Group = sudo

# 5. Dùng cờ -R để đổi đệ quy toàn bộ file bên trong thư mục
# sudo chown -R dev:dev /path/to/folder
```

---

## 🔐 4. Bản Chất Phân Quyền (Permissions): `r`, `w`, `x`

### 4.1. Cấu trúc chuỗi phân quyền

Khi chạy lệnh `ls -l`, hãy nhìn vào **10 ký tự đầu tiên**:

```text
- r w x r - x r - -
│ └──┬┘ └──┬┘ └──┬┘
│    │     │     └── 3. Others (Người dùng khác trên máy)
│    │     └──────── 2. Group (Các thành viên thuộc nhóm sở hữu)
│    └────────────── 1. Owner (Chính chủ sở hữu file)
└─────────────────── 0. Loại tệp (-: file, d: directory, l: link)
```

* **`r` (Read - Đọc):** Cho phép đọc nội dung file.
* **`w` (Write - Ghi):** Cho phép sửa đổi hoặc ghi đè nội dung file.
* **`x` (Execute - Thực thi):** Cho phép chạy file như một chương trình / script.
* **`-` (None):** Không có quyền tương ứng.

---

### 4.2. Phân tích ví dụ thực tế: `-rwxr-xr--`

```text
- r w x   r - x   r - -
  (Owner) (Group) (Others)
```

* **Owner (`rwx`):** Có toàn quyền: Đọc (`r`), Ghi/Sửa (`w`), Chạy file (`x`).
* **Group (`r-x`):** Chỉ được Đọc (`r`) và Chạy file (`x`), **không được sửa/xóa** (`-`).
* **Others (`r--`):** Chỉ được Đọc (`r`), không được sửa (`-`) và không được chạy (`-`).

---

## 🔢 5. Thay Đổi Phân Quyền Với `chmod` (Numeric vs Symbolic)

### 5.1. Bản chất dạng số (Octal / Numeric Mode)

Mỗi quyền hạn được gán một giá trị số nhị phân:

```text
r (Read)    = 4  (2^2)
w (Write)   = 2  (2^1)
x (Execute) = 1  (2^0)
- (None)    = 0
```

Mỗi nhóm người dùng sẽ có tổng điểm quyền từ **0 đến 7** ($4 + 2 + 1 = 7$):

$$\text{Tổng quyền} = (\text{Read} \times 4) + (\text{Write} \times 2) + (\text{Execute} \times 1)$$

```text
┌──────────────┬───────────────────┬──────────────────────────────────────────┐
│ Mã số bát phân│ Ký hiệu tương ứng │ Ý nghĩa thực tế                          │
├──────────────┼───────────────────┼──────────────────────────────────────────┤
│ 7 (4+2+1)    │ rwx               │ Toàn quyền (Đọc, Ghi, Thực thi).         │
│ 6 (4+2+0)    │ rw-               │ Đọc và Ghi (Không được thực thi).        │
│ 5 (4+0+1)    │ r-x               │ Đọc và Thực thi (Chỉ đọc & chạy, cấm sửa)│
│ 4 (4+0+0)    │ r--               │ Chỉ đọc (Read-only).                     │
│ 0 (0+0+0)    │ ---               │ Bị cấm hoàn toàn, không có quyền gì.     │
└──────────────┴───────────────────┴──────────────────────────────────────────┘
```

#### Giải mã 3 bộ số kinh điển trong quản trị Linux Server:

```text
1. chmod 755 script.sh   ──► Owner: 7 (rwx) | Group: 5 (r-x) | Others: 5 (r-x)
   (Dành cho file thực thi, script hoặc thư mục công khai)

2. chmod 644 config.txt  ──► Owner: 6 (rw-) | Group: 4 (r--) | Others: 4 (r--)
   (Chuẩn cho file văn bản/code: Chủ sửa được, mọi người chỉ được đọc)

3. chmod 600 id_ed25519  ──► Owner: 6 (rw-) | Group: 0 (---) | Others: 0 (---)
   (Bảo mật tối đa cho SSH Key, Password: Chỉ duy nhất chủ nhân được truy cập)
```

---

### 5.2. Bản chất dạng ký hiệu (Symbolic Mode)

Thay vì tính số, bạn có thể thêm (`+`), bớt (`-`), hoặc gán (`=`) quyền cụ thể cho từng đối tượng:
* `u`: User / Owner
* `g`: Group
* `o`: Others
* `a`: All (Tất cả mọi người)

```bash
touch script.sh

# Thêm quyền thực thi cho Owner (+x)
chmod u+x script.sh

# Bỏ quyền ghi của Group (g-w)
chmod g-w script.sh

# Gán quyền đọc duy nhất cho Others (o=r)
chmod o=r script.sh

# Thêm quyền thực thi cho tất cả mọi người (a+x)
chmod a+x script.sh
```

---

## 📁 6. Trọng Tâm: Phân Quyền Trên Directory Khác File Thế Nào?

Đây là phần kiến thức quan trọng nhất mà rất nhiều người học Linux hay bị nhầm lẫn:

```text
┌──────────────┬──────────────────────────────┬──────────────────────────────┐
│ Quyền        │ Ý nghĩa đối với FILE         │ Ý nghĩa đối với DIRECTORY    │
├──────────────┼──────────────────────────────┼──────────────────────────────┤
│ r (Read)     │ Mở xem nội dung file.        │ Xem danh sách file bên trong │
│              │ (ví dụ: cat, less, head).    │ bằng lệnh `ls`.              │
├──────────────┼──────────────────────────────┼──────────────────────────────┤
│ w (Write)    │ Sửa/xóa nội dung của file.   │ TẠO MỚI, ĐỔI TÊN, XÓA FILE   │
│              │ (ví dụ: vim, nano, echo >).  │ bên trong thư mục!           │
├──────────────┼──────────────────────────────┼──────────────────────────────┤
│ x (Execute)  │ Chạy file như 1 chương trình │ ĐƯỢC PHÉP TRUY CẬP VÀO       │
│              │ (ví dụ: ./app.sh).           │ THƯ MỤC bằng lệnh `cd`.      │
└──────────────┴──────────────────────────────┴──────────────────────────────┘
```

> [!WARNING]
> **2 sự thật gây sốc về quyền hạn Directory trong Linux:**
> 1. Nếu bạn có quyền `r` (Read) nhưng **không có quyền `x` (Execute)** trên một thư mục $\rightarrow$ Bạn gõ `ls` sẽ thấy tên file nhưng **không thể `cd` vào** và **không thể đọc nội dung file bên trong**!
> 2. Quyền xóa một file **nằm ở Thư mục cha chứa file đó (quyền `w` của Directory)**, chứ KHÔNG phụ thuộc vào quyền của chính file đó! Một user có thể xóa một file Read-only của user khác nếu user đó có quyền `w` trên thư mục cha!

---

### 6.1. Thực hành kiểm chứng quyền hạn Directory

Hãy tự tay thực hiện thí nghiệm sau để kiểm chứng bản chất:

```bash
# Bước 1: Tạo thư mục test và một file bên trong
mkdir /tmp/lab-dir
echo "Bi mat" > /tmp/lab-dir/secret.txt

# Bước 2: Tước bỏ quyền execute (x) của thư mục
chmod -x /tmp/lab-dir
ls -ld /tmp/lab-dir
# Quyền hiện tại: drw-rw-r-- (Có 'r', không có 'x')

# Bước 3: Thử truy cập vào thư mục (SẼ BỊ CHẶN NGAY)
cd /tmp/lab-dir
# Kết quả: bash: cd: /tmp/lab-dir: Permission denied

# Bước 4: Khôi phục lại quyền 'x'
chmod +x /tmp/lab-dir
cd /tmp/lab-dir
# Kết quả: Truy cập thành công!

# Dọn dẹp
cd ~ && rm -rf /tmp/lab-dir
```

---

## 🛡️ 7. Quản Trị Đặc Quyền: `sudo` và `root`

```text
┌─────────────────────────────────────────────────────────────┐
│                      ROOT SUPERUSER (UID: 0)                │
│            Toàn quyền tuyệt đối, nguy cơ phá hủy hệ thống   │
└──────────────────────────────▲──────────────────────────────┘
                               │
               ┌───────────────┴───────────────┐
               │                               │
        sudo command                           su -
  (Chạy 1 lệnh cụ thể với quyền root,   (Chuyển hẳn sang shell root,
   có log ghi lại vào /var/log/auth.log) dễ gây thảm họa nếu gõ nhầm)
               │                               │
┌──────────────┴───────────────────────────────┴──────────────┐
│                   STANDARD USER (User: ubuntu)              │
│       Chỉ được chỉnh sửa dữ liệu trong thư mục /home/       │
└─────────────────────────────────────────────────────────────┘
```

### 7.1. Bản chất của `sudo`
* **`sudo` (Superuser Do):** Cho phép user thực thi tạm thời một câu lệnh với quyền hạn của `root`.
* **Cơ chế xác thực:** Khi bạn gõ `sudo`, hệ thống hỏi **mật khẩu của chính bạn** (không phải mật khẩu của root), và ghi lại dấu vết hành động đó vào file nhật ký bảo mật `/var/log/auth.log` để kiểm toán (Audit Log).
* **Kiểm tra quyền `sudo` của bản thân:**
  ```bash
  sudo -l
  ```
  *(Lệnh này in ra danh sách chính xác các câu lệnh mà tài khoản của bạn được phép chạy với quyền root).*

### 7.2. Tại sao KHÔNG NÊN đăng nhập bằng `root` cho mọi thao tác?
1. **Tránh thảm họa do gõ nhầm:** Một câu lệnh sai cú pháp nhỏ như `rm -rf / tmp/trash` (thừa dấu cách sau `/`) khi chạy dưới quyền root sẽ **xóa sạch toàn bộ hệ điều hành ngay lập tức**.
2. **Không có trách nhiệm giải trình (Audit Trail):** Nếu nhiều kỹ sư cùng dùng chung tài khoản `root`, khi server gặp sự cố sẽ không thể biết ai là người đã sửa file cấu hình hoặc tắt dịch vụ.

---

## 🧪 8. Bài Thực Hành Tổng Hợp: Thiết Lập Dự Án Đa Người Dùng (Multi-User Lab)

Chúng ta sẽ mô phỏng một bài toán thực tế trong doanh nghiệp: Thiết lập môi trường làm việc chung cho nhóm phát triển gồm lập trình viên **Backend** và **Frontend**.

```text
/project
├── backend/   ──► Chỉ user 'backend' & group 'developers' được thao tác
└── frontend/  ──► Chỉ user 'frontend' & group 'developers' được thao tác
```

```text
┌─────────────────────────────────────────────────────────────┐
│                        YÊU CẦU BÀI TOÁN                     │
├─────────────────────────────────────────────────────────────┤
│ 1. Tạo 3 user: dev, backend, frontend.                      │
│ 2. Tạo group: developers.                                   │
│ 3. Thêm user backend và frontend vào group developers.      │
│ 4. Tạo thư mục /project, /project/backend, /project/frontend│
│ 5. Phân quyền:                                              │
│    - /project thuộc group developers (quyền 770).           │
│    - /project/backend thuộc user backend (quyền 770).       │
│    - /project/frontend thuộc user frontend (quyền 770).     │
│    - User 'dev' (không thuộc nhóm) bị CHẶN HOÀN TOÀN.       │
└─────────────────────────────────────────────────────────────┘
```

---

### Bước 8.1: Tạo các tài khoản User và Group

```bash
# 1. Tạo 3 user mới (dùng cờ tạo tự động không cần hỏi mật khẩu rườm rà)
sudo adduser --disabled-password --gecos "" dev
sudo adduser --disabled-password --gecos "" backend
sudo adduser --disabled-password --gecos "" frontend

# 2. Tạo group chung mang tên 'developers'
sudo groupadd developers

# 3. Thêm backend và frontend vào group developers (-aG: append to group)
sudo usermod -aG developers backend
sudo usermod -aG developers frontend

# 4. Kiểm tra lại danh sách group của user backend
groups backend
# Kết quả: backend : backend developers
```

---

### Bước 8.2: Tạo cây thư mục dự án và phân quyền

```bash
# 1. Tạo cây thư mục tại thư mục gốc /
sudo mkdir -p /project/backend /project/frontend

# 2. Gán Group sở hữu cho toàn bộ thư mục /project là 'developers'
sudo chgrp -R developers /project

# 3. Gán User sở hữu tương ứng cho từng thư mục con
sudo chown backend /project/backend
sudo chown frontend /project/frontend

# 4. Thiết lập phân quyền 770:
# Owner: 7 (rwx) | Group: 7 (rwx) | Others: 0 (---: Cấm hoàn toàn người ngoài)
sudo chmod 770 /project
sudo chmod 770 /project/backend
sudo chmod 770 /project/frontend

# 5. Kiểm tra lại cấu trúc phân quyền bằng ls -la
ls -la /project
```

*Quan sát kết quả phân quyền:*
```text
drwxrwx--- 4 root     developers 4096 Aug 21 08:30 .
drwxr-xr-x 3 root     root       4096 Aug 21 08:30 ..
drwxrwx--- 2 backend  developers 4096 Aug 21 08:30 backend
drwxrwx--- 2 frontend developers 4096 Aug 21 08:30 frontend
```

---

### Bước 8.3: Kiểm chứng thực tế (Đăng nhập từng user để test)

#### Thử nghiệm 1: User `backend` thao tác
```bash
# Chuyển sang phiên làm việc của user backend
sudo su - backend

# Đi vào thư mục backend và tạo file code
cd /project/backend
echo "console.log('NestJS API ready');" > app.js
ls -la
# Kết quả: Tạo file thành công!

# Thử đi vào thư mục frontend tạo file
cd /project/frontend
touch hack.js
# Kết quả: Vẫn tạo được vì cùng thuộc group 'developers'!

exit
```

#### Thử nghiệm 2: User `dev` (Người ngoài - Others) thử xâm nhập
```bash
# Chuyển sang user dev (không nằm trong group developers)
sudo su - dev

# Thử truy cập vào thư mục dự án
cd /project
# Kết quả ngay lập tức: bash: cd: /project: Permission denied

exit
```

> [!TIP]
> **Kết luận thực nghiệm:** Hệ thống phân quyền Linux đã chặn đứng hoàn toàn user `dev` từ cửa ngoài (`/project`), trong khi các thành viên nhóm `developers` làm việc trơn tru bên trong!

---

## 🚨 9. Xử Lý 3 Tình Huống Lỗi Permission Kinh Điển Trên Server

### 🔴 Tình huống 1: Web Server Nginx báo lỗi `403 Forbidden`
* **Triệu chứng:** Bạn deploy web tĩnh lên `/var/www/my-site`, nhưng mở trình duyệt truy cập chỉ thấy trang lỗi `403 Forbidden`.
* **Nguyên nhân:** Web Server Nginx chạy ngầm dưới quyền user `www-data`. Thư mục web của bạn đang bị phân quyền quá chặt khiến `www-data` không thể đọc file hoặc không có quyền `x` để `cd` vào thư mục.
* **Cách khắc phục:**
  ```bash
  # Cấp quyền sở hữu thư mục cho user www-data của Nginx
  sudo chown -R www-data:www-data /var/www/my-site

  # Đảm bảo phân quyền chuẩn: Thư mục là 755, File là 644
  sudo find /var/www/my-site -type d -exec chmod 755 {} \;
  sudo find /var/www/my-site -type f -exec chmod 644 {} \;
  ```

---

### 🔴 Tình huống 2: Lỗi `Permissions 0644 for 'id_ed25519' are too open` khi SSH
* **Triệu chứng:** Khi dùng SSH Key kết nối server, terminal báo lỗi đỏ và từ chối kết nối.
* **Nguyên nhân:** SSH Client phát hiện file Private Key đang để quyền `644` (cho phép các user khác trên máy đọc trộm).
* **Cách khắc phục:**
  ```bash
  chmod 600 ~/.ssh/id_ed25519
  ```

---

### 🔴 Tình huống 3: Script chạy báo lỗi `Permission denied`
* **Triệu chứng:** Chạy `./deploy.sh` bị báo lỗi không có quyền.
* **Nguyên nhân:** File mới tạo mặc định chỉ có quyền `rw-` (644), chưa được bật cờ thực thi (`x`).
* **Cách khắc phục:**
  ```bash
  chmod +x deploy.sh
  ./deploy.sh
  ```

---

## 🗺️ 10. Sơ Đồ Tổng Hợp & Lệnh Cốt Lõi Cần Nhớ

```text
QUẢN LÝ USER & PERMISSION
│
├── 👤 User & Group Management
│   ├── whoami, id, groups        ──► Kiểm tra danh tính & nhóm
│   ├── sudo adduser <user>       ──► Tạo người dùng mới
│   ├── sudo userdel -r <user>    ──► Xóa người dùng kèm thư mục Home
│   ├── sudo groupadd <group>     ──► Tạo nhóm mới
│   └── sudo usermod -aG <g> <u>  ──► Thêm user vào nhóm
│
├── 🏷️ Ownership (Quyền sở hữu)
│   ├── sudo chown user file      ──► Đổi Owner
│   ├── sudo chown user:group f   ──► Đổi cả Owner & Group
│   └── sudo chown -R u:g dir     ──► Đổi đệ quy toàn bộ thư mục
│
├── 🔢 Permission (Phân quyền)
│   ├── chmod 755 file/dir        ──► rwxr-xr-x (File thực thi, thư mục công khai)
│   ├── chmod 644 file            ──► rw-r--r-- (File dữ liệu, code chuẩn)
│   ├── chmod 600 file            ──► rw------- (Bảo mật tối đa, chỉ Owner)
│   └── chmod u+x, g-w, o=r       ──► Phân quyền bằng ký hiệu
│
└── 🛡️ Privilege Escalation
    ├── sudo -l                   ──► Kiểm tra quyền root của bản thân
    └── sudo command              ──► Thực thi an toàn dưới quyền root
```

---

## 📌 Tóm Tắt Bản Chất Cần Nhớ

1. **User = Định danh (UID)**, **Group = Nhóm quyền (GID)**, **Root = Quyền tối cao (UID 0)**.
2. **Chuỗi 9 ký tự phân quyền:** 3 ký tự đầu cho **Owner**, 3 ký tự giữa cho **Group**, 3 ký tự cuối cho **Others**.
3. **Công thức tính điểm quyền:** $\text{Read} = 4$, $\text{Write} = 2$, $\text{Execute} = 1$.
4. **Quyền trên Directory mang tính quyết định:** Cần `x` để `cd` vào trong, và quyền xóa file nằm ở **quyền `w` của Thư mục cha**.
5. **Không bao giờ làm việc trực tiếp bằng tài khoản `root`:** Luôn dùng tài khoản cá nhân kết hợp lệnh `sudo`.
