# OpenStack Nova

Nova là OpenStack Compute Service, dùng để quản lý và tạo máy ảo.

## 1. Mô hình triển khai

| Node | IP | Role |
|---|---|---|
| Controller | 10.168.36.11 | Nova API, Scheduler, Conductor |
| Compute01 | 10.168.36.12 | Nova Compute |
| Compute02 | 10.168.36.13 | Nova Compute |

Nova sử dụng:

- MariaDB làm database
- RabbitMQ làm message queue
- Keystone để authentication
- Placement để quản lý tài nguyên
- Glance để cung cấp image
- KVM làm hypervisor trên Compute Node

## 2. Tạo Database

```bash
sudo mariadb
```

```sql
CREATE DATABASE nova_api;
CREATE DATABASE nova;
CREATE DATABASE nova_cell0;

CREATE USER 'nova'@'localhost'
IDENTIFIED BY 'quanghuy94';

CREATE USER 'nova'@'%'
IDENTIFIED BY 'quanghuy94';

GRANT ALL PRIVILEGES ON nova_api.* TO 'nova'@'localhost';
GRANT ALL PRIVILEGES ON nova_api.* TO 'nova'@'%';

GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'localhost';
GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'%';

GRANT ALL PRIVILEGES ON nova_cell0.* TO 'nova'@'localhost';
GRANT ALL PRIVILEGES ON nova_cell0.* TO 'nova'@'%';

FLUSH PRIVILEGES;
EXIT;
```

## 3. Đăng ký Nova với Keystone

```bash
source ~/admin-openrc
```

```bash
openstack user create --domain default \
  --password quanghuy94 nova

openstack role add --project service \
  --user nova admin

openstack service create --name nova \
  --description "OpenStack Compute" \
  compute
```

Tạo endpoint:

```bash
openstack endpoint create --region RegionOne \
  compute public http://10.168.36.11:8774/v2.1

openstack endpoint create --region RegionOne \
  compute internal http://10.168.36.11:8774/v2.1

openstack endpoint create --region RegionOne \
  compute admin http://10.168.36.11:8774/v2.1
```

## 4. Cài đặt Nova trên Controller

```bash
sudo apt update
sudo apt install nova-api nova-conductor nova-novncproxy nova-scheduler -y
```

## 5. Cấu hình Nova

```bash
sudo nano /etc/nova/nova.conf
```

```ini
[DEFAULT]
transport_url = rabbit://openstack:quanghuy94@10.168.36.11:5672/
my_ip = 10.168.36.11
use_neutron = true
firewall_driver = nova.virt.firewall.NoopFirewallDriver

[api_database]
connection = mysql+pymysql://nova:quanghuy94@10.168.36.11/nova_api

[database]
connection = mysql+pymysql://nova:quanghuy94@10.168.36.11/nova

[api]
auth_strategy = keystone

[keystone_authtoken]
www_authenticate_uri = http://10.168.36.11:5000
auth_url = http://10.168.36.11:5000
memcached_servers = 10.168.36.11:11211
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = nova
password = quanghuy94

[placement]
auth_url = http://10.168.36.11:5000
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = placement
password = quanghuy94
region_name = RegionOne

[glance]
api_servers = http://10.168.36.11:9292

[vnc]
enabled = true
server_listen = 0.0.0.0
server_proxyclient_address = 10.168.36.11
```

## 6. Cấu hình Cells v2

```bash
sudo su -s /bin/sh -c "nova-manage api_db sync" nova

sudo su -s /bin/sh -c "nova-manage cell_v2 map_cell0" nova

sudo su -s /bin/sh -c \
"nova-manage cell_v2 create_cell --name=cell1 --verbose" nova

sudo su -s /bin/sh -c "nova-manage db sync" nova
```

Kiểm tra:

```bash
sudo su -s /bin/sh -c "nova-manage cell_v2 list_cells" nova
```

Kết quả gồm:

```text
cell0
cell1
```

## 7. Cài Nova Compute

Trên Compute Node:

```bash
sudo apt update
sudo apt install nova-compute -y
```

Cấu hình:

```bash
sudo nano /etc/nova/nova.conf
```

```ini
[DEFAULT]
transport_url = rabbit://openstack:quanghuy94@10.168.36.11:5672/
my_ip = 10.168.36.12
use_neutron = true
firewall_driver = nova.virt.firewall.NoopFirewallDriver

[api]
auth_strategy = keystone

[keystone_authtoken]
www_authenticate_uri = http://10.168.36.11:5000
auth_url = http://10.168.36.11:5000
memcached_servers = 10.168.36.11:11211
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = nova
password = quanghuy94

[placement]
auth_url = http://10.168.36.11:5000
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = placement
password = quanghuy94
region_name = RegionOne

[glance]
api_servers = http://10.168.36.11:9292

[vnc]
enabled = true
server_listen = 0.0.0.0
server_proxyclient_address = 10.168.36.12
novncproxy_base_url = http://10.168.36.11:6080/vnc_auto.html

[libvirt]
virt_type = kvm
```

