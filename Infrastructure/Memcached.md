# Memcached

## Mục đích

Memcached được sử dụng làm caching service cho các
OpenStack services.

## Mô hình triển khai

Memcached được triển khai trên Controller Node.

| Node | IP | Role |
|---|---|---|
| Controller | 10.168.36.11 | Memcached |
| Compute01 | 10.168.36.12 | Compute |
| Compute02 | 10.168.36.13 | Compute |

## Cài đặt

Cập nhật package:

```bash
sudo apt update
```

Cài đặt Memcached và Python client:

```bash
sudo apt install memcached python3-memcache -y
```

## Cấu hình

Mở file cấu hình:

```bash
sudo nano /etc/memcached.conf
```

Tìm:

```text
-l 127.0.0.1
```

Thay bằng:

```text
-l 10.168.36.11
```

Khởi động lại Memcached:

```bash
sudo systemctl restart memcached
```

---
