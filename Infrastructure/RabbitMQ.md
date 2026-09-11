# RabbitMQ

## Mục đích

RabbitMQ được sử dụng làm message queue để các OpenStack
services trao đổi message với nhau.

## Mô hình triển khai

RabbitMQ được triển khai trên Controller Node.

| Node | IP | Role |
|---|---|---|
| Controller | 10.168.36.11 | RabbitMQ |
| Compute01 | 10.168.36.12 | Compute |
| Compute02 | 10.168.36.13 | Compute |

## Cài đặt

Cập nhật package:

```bash
sudo apt update
```

Cài đặt RabbitMQ:

```bash
sudo apt install rabbitmq-server -y
```

## Tạo user OpenStack

```bash
sudo rabbitmqctl add_user openstack <RABBIT_PASSWORD>
```

## Cấp quyền cho user OpenStack

```bash
sudo rabbitmqctl set_permissions openstack ".*" ".*" ".*"
```


