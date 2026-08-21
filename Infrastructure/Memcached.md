# Memcached

## 1. Mục đích

Memcached được sử dụng làm caching service cho các
OpenStack services.

## 2. Mô hình triển khai

Memcached được triển khai trên Controller Node.

| Node | IP | Role |
|---|---|---|
| Controller | 10.168.36.11 | Memcached |
| Compute01 | 10.168.36.12 | Compute |
| Compute02 | 10.168.36.13 | Compute |
