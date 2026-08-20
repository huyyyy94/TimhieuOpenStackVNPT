# RabbitMQ

## 1. Mục đích

RabbitMQ được sử dụng làm message queue để các OpenStack
services trao đổi message với nhau.

## 2. Mô hình triển khai

RabbitMQ được triển khai trên Controller Node.

| Node | IP | Role |
|---|---|---|
| Controller | 10.168.36.11 | RabbitMQ |
| Compute01 | 10.168.36.12 | Compute |
| Compute02 | 10.168.36.13 | Compute |
