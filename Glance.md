# OpenStack Glance

Glance là OpenStack Image Service, dùng để quản lý các image phục vụ việc tạo máy ảo.

## 1. Mô hình triển khai

| Node | IP | Role |
|---|---|---|
| Controller | 10.168.36.11 | Glance |
| Compute01 | 10.168.36.12 | Compute |
| Compute02 | 10.168.36.13 | Compute |

Glance sử dụng:

- MariaDB làm database
- Keystone để authentication
- Filesystem để lưu image
- API chạy trên port `9292`

## 2. Tạo Database

```bash
sudo mariadb
```

```sql
CREATE DATABASE glance;

CREATE USER 'glance'@'localhost'
IDENTIFIED BY 'quanghuy94';

GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'localhost';

CREATE USER 'glance'@'%'
IDENTIFIED BY 'quanghuy94';

GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'%';

FLUSH PRIVILEGES;
EXIT;
```

## 3. Đăng ký Glance với Keystone

```bash
source ~/admin-openrc
```

```bash
openstack user create --domain default \
  --password quanghuy94 glance

openstack role add --project service \
  --user glance admin

openstack service create --name glance \
  --description "OpenStack Image" \
  image
```

Tạo endpoint:

```bash
openstack endpoint create --region RegionOne \
  image public http://10.168.36.11:9292

openstack endpoint create --region RegionOne \
  image internal http://10.168.36.11:9292

openstack endpoint create --region RegionOne \
  image admin http://10.168.36.11:9292
```

## 4. Cài đặt Glance

```bash
sudo apt update
sudo apt install glance -y
```

## 5. Cấu hình Glance

```bash
sudo nano /etc/glance/glance-api.conf
```

### Database

```ini
[database]
connection = mysql+pymysql://glance:quanghuy94@10.168.36.11/glance
```

### Keystone

```ini
[keystone_authtoken]
www_authenticate_uri = http://10.168.36.11:5000
auth_url = http://10.168.36.11:5000
memcached_servers = 10.168.36.11:11211
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = glance
password = quanghuy94
```

### Storage

```ini
[glance_store]
stores = file
default_store = file
filesystem_store_datadir = /var/lib/glance/images/
```

## 6. Khởi tạo Database và Storage

```bash
sudo su -s /bin/sh -c "glance-manage db_sync" glance
```

```bash
sudo mkdir -p /var/lib/glance/images
sudo chown -R glance:glance /var/lib/glance/images
```

## 7. Khởi động Glance

```bash
sudo systemctl enable glance-api
sudo systemctl restart glance-api
```

## 8. Upload Ubuntu Image

Download Ubuntu 24.04:

```bash
wget https://cloud-images.ubuntu.com/noble/current/noble-server-cloudimg-amd64.img
```

Upload image:

```bash
openstack image create "Ubuntu 24.04" \
  --file noble-server-cloudimg-amd64.img \
  --disk-format qcow2 \
  --container-format bare \
  --public
```

## 9. Kết quả

Glance đã được triển khai trên Controller và upload thành công image Ubuntu 24.04.

```text
Glance API
    |
    +--- MariaDB
    |
    +--- Filesystem
           |
           +--- Ubuntu 24.04
```
