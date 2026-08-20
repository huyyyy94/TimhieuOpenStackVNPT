# MariaDB

## 1. Mục đích

MariaDB được sử dụng làm SQL database backend
cho các OpenStack services.

## 2. Mô hình triển khai

MariaDB được triển khai trên Controller Node.

| Node | IP | Role |
|---|---|---|
| Controller | 10.168.36.11 | MariaDB |
| Compute01 | 10.168.36.12 | Compute |
| Compute02 | 10.168.36.13 | Compute |
