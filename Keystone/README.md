# OpenStack Keystone

Keystone là OpenStack Identity service, chịu trách nhiệm quản lý:

- User
- Project
- Role
- Authentication
- Authorization
- Token
- Service Catalog

## 1. Mô hình triển khai

Keystone được triển khai trên Controller Node.

| Node | IP | Role |
|---|---|---|
| Controller | 10.168.36.11 | Keystone |
| Compute01 | 10.168.36.12 | Compute |
| Compute02 | 10.168.36.13 | Compute |

Keystone sử dụng:

- MariaDB làm database backend
- Apache HTTP Server làm web server
- WSGI để chạy Keystone API

---

# 2. Tạo Database cho Keystone

Đăng nhập MariaDB:

```bash
sudo mariadb
```

Tạo database:

```sql
CREATE DATABASE keystone;
```

Tạo user:

```sql
CREATE USER 'keystone'@'localhost' IDENTIFIED BY '<KEYSTONE_DBPASS>';
```

Cấp quyền:

```sql
GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'localhost';
```

Cho phép Keystone truy cập database từ Controller:

```sql
CREATE USER 'keystone'@'%' IDENTIFIED BY '<KEYSTONE_DBPASS>';
```

Cấp quyền:

```sql
GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'%';
```

Áp dụng quyền:

```sql
FLUSH PRIVILEGES;
```

Thoát:

```sql
EXIT;
```

> Không lưu password thật trong Git.

---

# 3. Cài đặt Keystone

Cập nhật package:

```bash
sudo apt update
```

Cài đặt Keystone:

```bash
sudo apt install keystone apache2 libapache2-mod-wsgi-py3 -y
```

Các thành phần được cài:

| Package | Purpose |
|---|---|
| `keystone` | OpenStack Identity service |
| `apache2` | Web server |
| `libapache2-mod-wsgi-py3` | Chạy Keystone thông qua WSGI |

---

# 4. Cấu hình Keystone

Mở file:

```bash
sudo nano /etc/keystone/keystone.conf
```

## Database

Tìm phần:

```ini
[database]
```

Cấu hình:

```ini
[database]
connection = mysql+pymysql://keystone:<KEYSTONE_DBPASS>@10.168.36.11/keystone
```

## Fernet Token

Cấu hình:

```ini
[token]
provider = fernet
```

---

# 5. Đồng bộ Database

Chuyển sang user `keystone`:

```bash
sudo su -s /bin/sh -c "keystone-manage db_sync" keystone
```

---

# 6. Khởi tạo Fernet Key Repository

Tạo thư mục:

```bash
sudo keystone-manage fernet_setup \
  --keystone-user keystone \
  --keystone-group keystone
```

Tạo credential keys:

```bash
sudo keystone-manage credential_setup \
  --keystone-user keystone \
  --keystone-group keystone
```

---

# 7. Bootstrap Keystone

Bootstrap Identity service:

```bash
sudo keystone-manage bootstrap \
  --bootstrap-password <ADMIN_PASS> \
  --bootstrap-admin-url http://10.168.36.11/identity/v3/ \
  --bootstrap-internal-url http://10.168.36.11/identity/v3/ \
  --bootstrap-public-url http://10.168.36.11/identity/v3/ \
  --bootstrap-region-id RegionOne
```

> Không lưu password thật trong Git.

---

# 8. Cấu hình Apache

Mở file:

```bash
sudo nano /etc/apache2/apache2.conf
```

Thêm:

```apache
ServerName controller
```

Restart Apache:

```bash
sudo systemctl restart apache2
```

Enable Apache:

```bash
sudo systemctl enable apache2
```

---

# 9. Thiết lập biến môi trường

Tạo file:

```bash
nano ~/admin-openrc
```

Nội dung:

```bash
export OS_USERNAME=admin
export OS_PASSWORD=<ADMIN_PASS>
export OS_PROJECT_NAME=admin
export OS_USER_DOMAIN_NAME=Default
export OS_PROJECT_DOMAIN_NAME=Default
export OS_AUTH_URL=http://10.168.36.11/identity
export OS_IDENTITY_API_VERSION=3
```

> Không commit file `admin-openrc` chứa password thật lên Git.

Load environment:

```bash
source ~/admin-openrc
```

---

# 10. Tạo Project

Tạo project `service`:

```bash
openstack project create --domain default \
  --description "Service Project" service
```

Tạo project `demo`:

```bash
openstack project create --domain default \
  --description "Demo Project" demo
```

---

# 11. Tạo User

Tạo user `demo`:

```bash
openstack user create --domain default \
  --password <DEMO_PASS> demo
```

> Không lưu password thật trong Git.

---

# 12. Tạo Role

Tạo role:

```bash
openstack role create user
```

Gán role cho user:

```bash
openstack role add --project demo --user demo user
```

---

# 13. Keystone trong OpenStack

Keystone cung cấp Identity service cho toàn bộ OpenStack.

Luồng cơ bản:

```text
User
  |
  v
Keystone
  |
  +---- Authentication
  |
  +---- Authorization
  |
  +---- Token
  |
  +---- Service Catalog
  |
  v
OpenStack Services
```

Các service khác sẽ sử dụng Keystone để xác thực
và xác định quyền truy cập của user.

---

# 14. Infrastructure Dependency

Keystone sử dụng các thành phần infrastructure đã triển khai trước đó:

```text
                 Keystone
                    |
          +---------+---------+
          |                   |
          v                   v
       MariaDB             Memcached
          |
          v
    Keystone Database
```

MariaDB lưu trữ dữ liệu của Keystone.

Memcached được sử dụng cho caching/token-related operations
trong môi trường OpenStack.

---

# 15. Kết quả

Sau khi hoàn thành các bước trên:

```text
Controller
|
+-- MariaDB
|     └── keystone database
|
+-- RabbitMQ
|
+-- Memcached
|
+-- Apache
|     └── Keystone API
|
└-- Keystone
      ├── User
      ├── Project
      ├── Role
      ├── Token
      └── Service Catalog
```

Keystone là Identity service nền tảng để các OpenStack
services sử dụng authentication và authorization.
