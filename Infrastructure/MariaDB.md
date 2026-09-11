# MariaDB

## Mục đích

MariaDB được sử dụng làm SQL database backend
cho các OpenStack services.

## Mô hình triển khai

MariaDB được triển khai trên Controller Node.

| Node | IP | Role |
|---|---|---|
| Controller | 10.168.36.11 | MariaDB |
| Compute01 | 10.168.36.12 | Compute |
| Compute02 | 10.168.36.13 | Compute |


## Cài đặt

Cập nhật package:

```bash
sudo apt update
```

Cài đặt MariaDB và PyMySQL:

```bash
sudo apt install mariadb-server python3-pymysql -y

## Cấu hình

Tạo file cấu hình:

```bash
sudo nano /etc/mysql/mariadb.conf.d/99-openstack.cnf
```

Nội dung:

```ini
[mysqld]
bind-address = 10.168.36.11

default-storage-engine = innodb
innodb_file_per_table = on
max_connections = 4096
collation-server = utf8_general_ci
character-set-server = utf8
```

Khởi động lại MariaDB:

```bash
sudo systemctl restart mariadb
```

## Bảo mật

```bash
sudo mysql_secure_installation
```
