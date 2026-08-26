# 📘 Phần 11: Database Servers – Quản Trị Cơ Sở Dữ Liệu (PostgreSQL, MySQL, MongoDB & Quản Lý Qua Docker)

> **Motto cốt lõi:**  
> *Dữ liệu là trái tim của hệ thống | Tuyệt đối không mở cổng Database (5432, 3306, 27017) trực tiếp ra Internet | Dù chạy Native hay Docker, Dữ liệu phải luôn được gắn Persistent Storage!*  
> Chu trình chuẩn: **Bản chất Database → Quản trị Native (PostgreSQL) → Mở rộng MySQL & MongoDB → Triển khai hiện đại qua Docker & Compose → Bảo mật & Backup.**

---

## 🎯 1. Mục Tiêu Của Phần Học

Sau khi hoàn thành phần này, bạn sẽ:
1. **Hiểu bản chất Database Server:** Cách các hệ quản trị CSDL quản lý tiến trình, lắng nghe kết nối mạng (Socket UNIX vs Port TCP) và lưu trữ dữ liệu xuống đĩa cứng (Data Directory).
2. **Làm chủ PostgreSQL (Native):** Cài đặt qua APT, quản trị Roles/Users, Database, cấu hình lắng nghe và phân quyền truy cập an toàn trong `pg_hba.conf`.
3. **Nắm vững MySQL & MongoDB cơ bản:** Biết cách khởi tạo, phân quyền user và bảo mật cho CSDL quan hệ (MySQL) và phi quan hệ (MongoDB).
4. **Quản trị Database hiện đại bằng Docker & Docker Compose:** Hiểu vì sao Docker là xu hướng triển khai CSDL hàng đầu hiện nay, cơ chế ánh xạ cổng (Port Mapping) và gắn vùng lưu trữ bền vững (**Named Volumes**).
5. **So sánh Native vs Docker:** Biết khi nào nên cài trực tiếp vào hệ điều hành và khi nào nên dùng Docker cho Database.
6. **Bảo mật Database cấp độ Production:** Chỉ cho phép kết nối từ `localhost` (`127.0.0.1`) hoặc qua Docker Network nội bộ, cô lập hoàn toàn khỏi Internet qua UFW.
7. **Sao lưu dữ liệu cốt lõi (Backup & Restore):** Thành thạo các lệnh sao lưu nhanh `pg_dump`, `mysqldump`, `mongodump`.

---

## 🧠 2. So Sánh: PostgreSQL vs MySQL vs MongoDB

```text
┌───────────────────────────┬───────────────────────────┬─────────────────────────────┐
│ Tiêu chí                  │ POSTGRESQL                │ MYSQL / MARIADB             │ MONGODB                     │
├───────────────────────────┼───────────────────────────┼─────────────────────────────┼─────────────────────────────┤
│ Loại CSDL                 │ RDBMS (Quan hệ nâng cao)  │ RDBMS (Quan hệ phổ biến)    │ NoSQL (Document Store)      │
│ Định dạng dữ liệu         │ Bảng, Cột, Hỗ trợ JSONB   │ Bảng, Cột, Khoá ngoại       │ JSON / BSON Documents       │
│ Cổng mặc định (Port)      │ 5432/tcp                  │ 3306/tcp                    │ 27017/tcp                   │
│ Tiến trình / Daemon       │ postgres.service          │ mysql.service               │ mongod.service              │
│ CLI tương tác             │ psql                      │ mysql                       │ mongosh                     │
│ Trường hợp sử dụng tốt    │ Hệ thống tài chính, ERP,  │ Web thương mại điện tử, CMS,│ Dữ liệu linh hoạt, Log,     │
│                           │ phức tạp, toàn vẹn cao    │ ứng dụng đọc nhanh          │ Chat, IoT, Schema động      │
└───────────────────────────┴───────────────────────────┴─────────────────────────────┴─────────────────────────────┘
```

---

## 🐘 3. Quản Trị PostgreSQL Cài Đặt Trực Tiếp (Native Setup)

PostgreSQL là CSDL quan hệ mạnh mẽ, chuẩn mực bậc nhất cho các ứng dụng Backend hiện đại (NestJS TypeORM/Prisma).

---

### Bước 3.1: Cài đặt PostgreSQL trên Ubuntu Server
```bash
sudo apt update
sudo apt install -y postgresql postgresql-contrib

# Kiểm tra trạng thái dịch vụ
sudo systemctl status postgresql
```

*Khi cài xong, PostgreSQL tự động tạo một user hệ thống mang tên **`postgres`**.*

---

### Bước 3.2: Cơ chế xác thực Peer Authentication & Đăng nhập `psql`

Mặc định trên Linux, PostgreSQL sử dụng cơ chế **Peer Authentication**: Nếu bạn là user Linux `postgres`, bạn sẽ tự động đăng nhập vào CSDL `postgres` mà không cần mật khẩu.

```bash
# Chuyển sang user postgres và mở công cụ dòng lệnh psql
sudo -u postgres psql
```

**Các câu lệnh `psql` cốt lõi cần nhớ:**
```sql
-- 1. Xem danh sách CSDL hiện có
\l

-- 2. Xem danh sách Users / Roles
\du

-- 3. Tạo một User mới với mật khẩu
CREATE USER devuser WITH PASSWORD 'MySecurePassword2026!';

-- 4. Tạo một Database mới thuộc sở hữu của devuser
CREATE DATABASE nestjs_db OWNER devuser;

-- 5. Cấp toàn bộ quyền trên database cho user
GRANT ALL PRIVILEGES ON DATABASE nestjs_db TO devuser;

-- 6. Kết nối vào Database vừa tạo
\c nestjs_db

-- 7. Thoát khỏi psql
\q
```

---

### Bước 3.3: Bảo mật cấu hình mạng PostgreSQL (`pg_hba.conf` & `postgresql.conf`)

Mặc định, PostgreSQL chỉ lắng nghe tại `localhost` (`127.0.0.1`), điều này rất an toàn!

* File cấu hình cổng và địa chỉ lắng nghe:
  ```bash
  sudo nano /etc/postgresql/16/main/postgresql.conf
  # Đảm bảo: listen_addresses = 'localhost' (không đổi thành '*' trừ khi có Private Network)
  ```
* File cấu hình quyền truy cập (Host-Based Authentication):
  ```bash
  sudo nano /etc/postgresql/16/main/pg_hba.conf
  # Đảm bảo các kết nối cục bộ sử dụng phương thức mã hóa md5 hoặc scram-sha-256
  ```

---

## 🐬 4. Quản Trị MySQL & 🍃 MongoDB Cài Đặt Trực Tiếp (Native)

### 4.1. Cài đặt & Thiết lập MySQL an toàn

```bash
# 1. Cài đặt MySQL Server
sudo apt install -y mysql-server

# 2. Chạy kịch bản bảo mật mặc định (Xóa user ẩn danh, tắt remote root)
sudo mysql_secure_installation

# 3. Đăng nhập vào MySQL với quyền root
sudo mysql
```

**Tạo Database và cấp quyền trong MySQL:**
```sql
CREATE DATABASE shop_db CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
CREATE USER 'shop_user'@'localhost' IDENTIFIED BY 'StrongMySQLPass2026!';
GRANT ALL PRIVILEGES ON shop_db.* TO 'shop_user'@'localhost';
FLUSH PRIVILEGES;
EXIT;
```

---

### 4.2. Cài đặt & Thiết lập MongoDB Community

MongoDB trên Ubuntu cần nạp khóa GPG chính thức từ kho của MongoDB:

```bash
# 1. Nạp khóa bảo mật và thêm kho lưu trữ MongoDB 7.0
curl -fsSL https://www.mongodb.org/static/pgp/server-7.0.asc | sudo gpg -o /usr/share/keyrings/mongodb-server-7.0.gpg --dearmor
echo "deb [ arch=amd64,arm64 signed-by=/usr/share/keyrings/mongodb-server-7.0.gpg ] https://repo.mongodb.org/apt/ubuntu jammy/mongodb-org/7.0 multiverse" | sudo tee /etc/apt/sources.list.d/mongodb-org-7.0.list

# 2. Cài đặt MongoDB
sudo apt update
sudo apt install -y mongodb-org

# 3. Kích hoạt và khởi chạy
sudo systemctl enable --now mongod

# 4. Mở công cụ tương tác MongoDB Shell
mongosh
```

**Tạo User quản trị trong MongoDB:**
```javascript
use admin
db.createUser({
  user: "mongo_admin",
  pwd: "MongoSecretPass2026!",
  roles: [ { role: "userAdminAnyDatabase", db: "admin" }, "readWriteAnyDatabase" ]
})
exit
```

---

## 🐳 5. Quản Trị Database Bằng Docker & Docker Compose (Hiện Đại)

Trong môi trường thực tế ngày nay, việc triển khai Database qua Docker Container đem lại sự linh hoạt tuyệt đối:
* **Không làm rác hệ điều hành gốc (Clean Host OS).**
* **Dễ dàng nâng/hạ phiên bản CSDL chỉ bằng 1 dòng cấu hình.**
* **Cô lập hoàn toàn mạng nội bộ qua Docker Bridge Network.**

```text
┌─────────────────────────────────────────────────────────────────────────┐
│              KIẾN TRÚC DATABASE CHẠY TRONG DOCKER VỚI VOLUME            │
└────────────────────────────────────┬────────────────────────────────────┘
                                     │
    Ứng dụng Backend (Port 3000) ───► Kết nối tới localhost:5432
                                     │
                                     ▼
             ┌───────────────────────────────────────────────┐
             │       DOCKER CONTAINER (Postgres 16)          │
             │                                               │
             │   Lắng nghe: Container Port 5432              │
             │   Ghi dữ liệu vào: /var/lib/postgresql/data   │
             └───────────────────────┬───────────────────────┘
                                     │
                                     │ (Ánh xạ vùng nhớ liên tục - Persistent)
                                     ▼
             ┌───────────────────────────────────────────────┐
             │       DOCKER NAMED VOLUME: postgres_data      │
             │       (/var/lib/docker/volumes/...)           │
             │  (Dù Container bị XÓA, DỮ LIỆU VẪN CÒN 100%)  │
             └───────────────────────────────────────────────┘
```

> [!CAUTION]
> **Quy tắc vàng của Database trên Docker:** Container có tính chất tạm thời (Ephemeral). Bạn **BẮT BUỘC PHẢI DÙNG VOLUME** để lưu dữ liệu ra ngoài ổ đĩa máy chủ. Nếu không có Volume, khi container bị restart hoặc xóa, toàn bộ dữ liệu sẽ biến mất vĩnh viễn!

---

### 5.1. Cài đặt Docker & Docker Compose trên Ubuntu Server

```bash
# 1. Cài đặt Docker engine
sudo apt update
sudo apt install -y docker.io docker-compose-v2

# 2. Cho phép user ubuntu chạy docker không cần gõ sudo
sudo usermod -aG docker ubuntu

# 3. Kích hoạt dịch vụ
sudo systemctl enable --now docker
```
*(Ghi chú: Đăng xuất SSH và đăng nhập lại để cập nhật group `docker`).*

---

### 5.2. File `docker-compose.yml` Quản Lý Đồng Thời PostgreSQL, MySQL, MongoDB

Hãy tạo một thư mục quản trị hạ tầng CSDL tập trung:

```bash
mkdir -p ~/databases && cd ~/databases
nano docker-compose.yml
```

Dán toàn bộ nội dung cấu hình chuẩn sau:

```yaml
version: '3.8'

services:
  # ==========================================
  # 1. POSTGRESQL SERVICE
  # ==========================================
  postgres_db:
    image: postgres:16-alpine
    container_name: production_postgres
    restart: always
    environment:
      POSTGRES_USER: devuser
      POSTGRES_PASSWORD: MySecurePassword2026!
      POSTGRES_DB: nestjs_db
    ports:
      # Ánh xạ: Chỉ mở cổng nội bộ 127.0.0.1 để tránh bị hacker từ ngoài quét
      - "127.0.0.1:5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U devuser -d nestjs_db"]
      interval: 10s
      timeout: 5s
      retries: 5

  # ==========================================
  # 2. MYSQL SERVICE
  # ==========================================
  mysql_db:
    image: mysql:8.0
    container_name: production_mysql
    restart: always
    environment:
      MYSQL_ROOT_PASSWORD: RootSecretPass2026!
      MYSQL_DATABASE: shop_db
      MYSQL_USER: shop_user
      MYSQL_PASSWORD: StrongMySQLPass2026!
    ports:
      - "127.0.0.1:3306:3306"
    volumes:
      - mysql_data:/var/lib/mysql

  # ==========================================
  # 3. MONGODB SERVICE
  # ==========================================
  mongo_db:
    image: mongo:7.0
    container_name: production_mongo
    restart: always
    environment:
      MONGO_INITDB_ROOT_USERNAME: mongo_admin
      MONGO_INITDB_ROOT_PASSWORD: MongoSecretPass2026!
    ports:
      - "127.0.0.1:27017:27017"
    volumes:
      - mongo_data:/data/db

# ==========================================
# KHAI BÁO CÁC VÙNG LƯU TRỮ BỀN VỮNG (VOLUMES)
# ==========================================
volumes:
  postgres_data:
    name: postgres_persistent_data
  mysql_data:
    name: mysql_persistent_data
  mongo_data:
    name: mongo_persistent_data
```

---

### 5.3. Các Lệnh Quản Trị Database Bằng Docker Compose

```bash
# 1. Khởi chạy toàn bộ CSDL ở chế độ chạy ngầm (-d)
docker compose up -d

# 2. Xem trạng thái các container CSDL đang chạy
docker compose ps

# 3. Xem log hoạt động của PostgreSQL container
docker compose logs -f postgres_db

# 4. Truy cập trực tiếp vào công cụ psql bên trong Docker Container
docker exec -it production_postgres psql -U devuser -d nestjs_db

# 5. Dừng các CSDL (Dữ liệu vẫn được bảo vệ nguyên vẹn trong volume)
docker compose down
```

---

## 💾 6. Chiến Lược Sao Lưu & Khôi Phục Dữ Liệu (Backup & Restore)

### 6.1. Sao lưu PostgreSQL với `pg_dump`
```bash
# Xuất toàn bộ dữ liệu ra 1 file SQL
pg_dump -U devuser -h 127.0.0.1 -d nestjs_db > ~/backup_nestjs_$(date +%F).sql

# Khôi phục dữ liệu từ file backup
psql -U devuser -h 127.0.0.1 -d nestjs_db < ~/backup_nestjs_2026-08-26.sql
```

*(Nếu dùng Docker):*
```bash
docker exec -t production_postgres pg_dump -U devuser nestjs_db > ~/backup_postgres.sql
```

---

### 6.2. Sao lưu MySQL với `mysqldump`
```bash
# Backup
mysqldump -u shop_user -p shop_db > ~/backup_mysql_$(date +%F).sql

# Restore
mysql -u shop_user -p shop_db < ~/backup_mysql_2026-08-26.sql
```

---

## 🧪 7. Bài Thực Hành Lab (10 Bước Triển Khai)

*Hãy thực hành xây dựng môi trường Database chuẩn mực trên Ubuntu Server VM:*

1. Kiểm tra tài nguyên RAM và đĩa trống của VM: `free -h` và `df -h`.
2. Cài đặt Docker và Docker Compose trên máy chủ.
3. Tạo thư mục quản trị `~/databases` và tạo file `docker-compose.yml`.
4. Cấu hình container PostgreSQL với user `devuser`, pass `MySecurePassword2026!`, volume `postgres_data`.
5. Khởi chạy CSDL bằng `docker compose up -d postgres_db`.
6. Kiểm tra trạng thái container bằng `docker compose ps`.
7. Đăng nhập vào psql bên trong container bằng `docker exec -it production_postgres psql -U devuser -d nestjs_db`.
8. Tạo một bảng mẫu `CREATE TABLE test (id SERIAL PRIMARY KEY, name VARCHAR(50));` và chèn 1 dòng dữ liệu.
9. Xóa container bằng `docker compose down` rồi bật lại bằng `docker compose up -d postgres_db`.
10. Truy vấn lại bảng `test` để chứng minh dữ liệu vẫn còn 100% nhờ Docker Volume.

---

## 📌 8. Bảng Tra Cứu Lệnh Database Cốt Lõi

| Lệnh | Ý nghĩa & Mục đích sử dụng |
| :--- | :--- |
| `sudo -u postgres psql` | Đăng nhập psql cục bộ dưới quyền admin postgres (Native) |
| `docker compose up -d` | Khởi chạy tất cả CSDL chạy ngầm (Docker) |
| `docker compose ps` | Kiểm tra tình trạng sức khỏe các container CSDL |
| `docker compose logs -f <service>` | Xem nhật ký log của CSDL |
| `docker exec -it <container> psql ...` | Mở CLI tương tác bên trong container CSDL |
| `docker volume ls` | Liệt kê các vùng lưu trữ dữ liệu bền vững |
| `pg_dump -U <user> -d <db> > file.sql` | Xuất file sao lưu cơ sở dữ liệu PostgreSQL |
| `psql -U <user> -d <db> < file.sql` | Nạp file khôi phục dữ liệu PostgreSQL |
