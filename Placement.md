# OpenStack Placement

Placement là OpenStack service dùng để quản lý và theo dõi tài nguyên của các Compute Node.

## 1. Mô hình triển khai

| Node | IP | Role |
|---|---|---|
| Controller | 10.168.36.11 | Placement |
| Compute01 | 10.168.36.12 | Compute |
| Compute02 | 10.168.36.13 | Compute |

Placement sử dụng:

- MariaDB làm database
- Keystone để authentication
- Apache HTTP Server làm web server
- API chạy trên port `8778`

## 2. Tạo Database

```bash
sudo mariadb
```

```sql
CREATE DATABASE placement;

CREATE USER 'placement'@'localhost'
IDENTIFIED BY 'quanghuy94';

GRANT ALL PRIVILEGES ON placement.* TO 'placement'@'localhost';

CREATE USER 'placement'@'%'
IDENTIFIED BY 'quanghuy94';

GRANT ALL PRIVILEGES ON placement.* TO 'placement'@'%';

FLUSH PRIVILEGES;
EXIT;
```

## 3. Đăng ký Placement với Keystone

```bash
source ~/admin-openrc
```

```bash
openstack user create --domain default \
  --password quanghuy94 placement

openstack role add --project service \
  --user placement admin

openstack service create --name placement \
  --description "Placement API" \
  placement
```

Tạo endpoint:

```bash
openstack endpoint create --region RegionOne \
  placement public http://10.168.36.11/placement

openstack endpoint create --region RegionOne \
  placement internal http://10.168.36.11/placement

openstack endpoint create --region RegionOne \
  placement admin http://10.168.36.11/placement
```

## 4. Cài đặt Placement

```bash
sudo apt update
sudo apt install placement-api -y
```

## 5. Cấu hình Placement

```bash
sudo nano /etc/placement/placement.conf
```

### Database

```ini
[placement_database]
connection = mysql+pymysql://placement:quanghuy94@10.168.36.11/placement
```

### Keystone

```ini
[api]
auth_strategy = keystone

[keystone_authtoken]
auth_url = http://10.168.36.11:5000
memcached_servers = 10.168.36.11:11211
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = placement
password = quanghuy94
```

## 6. Khởi tạo Database

```bash
sudo su -s /bin/sh -c \
"placement-manage db sync" placement
```

## 7. Cấu hình Apache

File:

```bash
sudo nano /etc/apache2/sites-available/placement-api.conf
```

Placement sử dụng Apache để cung cấp API thông qua:

```text
http://10.168.36.11/placement
```

API backend chạy trên:

```text
Port: 8778
```

Enable site:

```bash
sudo a2ensite placement-api
```

Restart Apache:

```bash
sudo systemctl restart apache2
```

## 8. Kết quả

Placement API hoạt động trên Controller.

```text
Keystone
    |
    v
Apache
    |
    v
Placement API :8778
    |
    +--- MariaDB
```
