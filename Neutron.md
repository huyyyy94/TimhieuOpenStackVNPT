# OpenStack Neutron

Neutron là OpenStack Networking Service, dùng để quản lý network, subnet, router, port và kết nối mạng cho VM.

## 1. Mô hình triển khai

| Node | IP | Role |
|---|---|---|
| Controller | 10.168.36.11 | Neutron Server, Neutron API, OVN DB |
| Compute01 | 10.168.36.12 | OVN Controller, Open vSwitch |
| Compute02 | 10.168.36.13 | OVN Controller, Open vSwitch |

Neutron sử dụng:

- MariaDB làm database
- RabbitMQ làm message queue
- Keystone để authentication
- ML2 làm plugin networking
- OVN làm mechanism driver
- Geneve làm tenant network encapsulation
- Open vSwitch làm virtual switch trên Compute Node

---

## 2. Kiểm tra phiên bản OpenStack

```bash
openstack --version
```

Kết quả:

```text
openstack 7.4.0
```

---

## 3. Kiểm tra Neutron package

Trên Controller:

```bash
dpkg -l | grep -E '^ii  neutron|^ii  ovn'
```

Kiểm tra version package:

```bash
apt-cache policy neutron-server neutron-api neutron-rpc-server
```

Trong quá trình lab, package Neutron được cài từ Ubuntu Cloud Archive.

---

# 4. Configure ML2

File cấu hình:

```text
/etc/neutron/plugins/ml2/ml2_conf.ini
```

Kiểm tra file:

```bash
sudo grep -n -C 3 'max_header_size' /etc/neutron/plugins/ml2/ml2_conf.ini
```

Ban đầu cấu hình có:

```ini
#max_header_size = 30
```

Trong comment của file có ghi:

```text
The maximum allowed Geneve encapsulation header size
...
The default is 30
...
for OVN which requires at least 38
```

Vì sử dụng OVN + Geneve nên đổi thành:

```ini
max_header_size = 38
```

Sửa bằng:

```bash
sudo sed -i 's/^#max_header_size = 30/max_header_size = 38/' /etc/neutron/plugins/ml2/ml2_conf.ini
```

Kiểm tra lại:

```bash
sudo grep -n -C 3 'max_header_size' /etc/neutron/plugins/ml2/ml2_conf.ini
```

Kết quả:

```text
243-# Geneve-based networks. The default is 30, which is the size of the Geneve
244-# header without any additional option headers. Note the default is not enough
245-# for OVN which requires at least 38. (integer value)
246:max_header_size = 38
```

### Lưu ý

Nếu lệnh:

```bash
sudo grep -n '^[[:space:]]*max_header_size' /etc/neutron/plugins/ml2/ml2_conf.ini
```

không trả về gì thì không có nghĩa là file không có option.

Có thể option đang bị comment:

```ini
#max_header_size = 30
```

Do đó nên dùng:

```bash
sudo grep -n -C 3 'max_header_size' /etc/neutron/plugins/ml2/ml2_conf.ini
```

---

# 5. Kiểm tra ML2 type

Kiểm tra section:

```bash
sudo grep -n -E '^\[ml2_type_(geneve|gre)\]' /etc/neutron/plugins/ml2/ml2_conf.ini
```

Kiểm tra Geneve:

```bash
sudo sed -n '228,250p' /etc/neutron/plugins/ml2/ml2_conf.ini
```

Kết quả có:

```ini
[ml2_type_geneve]

vni_ranges = 1:65535

max_header_size = 38
```

---

# 6. Neutron Database Migration

Kiểm tra database migration hiện tại:

```bash
sudo neutron-db-manage current
```

Kiểm tra offline migration:

```bash
sudo neutron-db-manage has_offline_migrations
```

Kết quả:

```text
No offline migrations pending.
```

Nếu database chưa ở head:

```bash
sudo neutron-db-manage upgrade heads
```

Sau đó kiểm tra lại:

```bash
sudo neutron-db-manage current
```

Kết quả cuối cùng:

```text
5c85685d616d (head)
```

---

# 7. Kiểm tra Neutron Server

```bash
sudo systemctl status neutron-server --no-pager
```

Expected:

```text
Active: active (running)
```

Tuy nhiên cần lưu ý:

> `neutron-server` đang `active` không đồng nghĩa với việc Neutron API đang listen trên port 9696.

Phải kiểm tra riêng:

```bash
sudo ss -lntp | grep 9696
```

---

# 8. Lỗi Connection Refused port 9696

Khi chạy:

```bash
openstack network agent list
```

đã gặp:

```text
Unable to establish connection to http://10.168.36.11:9696/v2.0/agents
Connection refused
```

Tương tự:

```bash
openstack network list
```

cũng gặp:

```text
Unable to establish connection to http://10.168.36.11:9696/v2.0/networks
Connection refused
```

Kiểm tra:

```bash
sudo ss -lntp | grep 9696
```

Không có output.

Trong khi:

```bash
sudo systemctl status neutron-server
```

lại cho thấy:

```text
Active: active (running)
```

### Kết luận

Lúc này vấn đề không nằm ở việc `neutron-server` chết.

Vấn đề nằm ở:

```text
Neutron API endpoint :9696
```

chưa được expose đúng.

---

# 9. Kiểm tra Apache

Kiểm tra Apache:

```bash
sudo systemctl status apache2 --no-pager
```

Kiểm tra VirtualHost:

```bash
sudo apachectl -S
```

Kết quả có:

```text
*:9696
    controller (/etc/apache2/sites-enabled/neutron-api.conf:...)
```

Kiểm tra port:

```bash
sudo ss -lntp | grep 9696
```

Sau khi Apache hoạt động:

```text
LISTEN
*:9696
```

---

# 10. Kiểm tra Apache modules

Kiểm tra:

```bash
sudo a2query -m proxy
```

Kiểm tra:

```bash
sudo a2query -m proxy_uwsgi
```

Nếu chưa có thì cài:

```bash
sudo apt install libapache2-mod-proxy-uwsgi
```

Enable:

```bash
sudo a2enmod proxy
sudo a2enmod proxy_uwsgi
```

Kiểm tra:

```bash
sudo a2query -m proxy
sudo a2query -m proxy_uwsgi
```

---

# 11. Configure Neutron API với uWSGI

File:

```text
/etc/neutron/neutron-api-uwsgi.ini
```

Nội dung:

```ini
[uwsgi]
chmod-socket = 666
socket = /var/run/uwsgi/neutron-api.socket
start-time = $t
lazy-apps = true
add-header = Connection: close
buffer-size = 65535
hook-master-start = unix_signal:15 gracefully_kill_them_all
thunder-lock = true
plugins = http,python3
enable-threads = true
worker-reload-mercy = 80
exit-on-reload = false
die-on-term = true
master = true
processes = 2
module = neutron.wsgi.api:application
```

Các thông số quan trọng:

```text
Socket:
 /var/run/uwsgi/neutron-api.socket

Application:
 neutron.wsgi.api:application

Workers:
 2
```

---

# 12. Kiểm tra uWSGI socket

Kiểm tra:

```bash
sudo ss -lxnp | grep neutron-api
```

Expected:

```text
/var/run/uwsgi/neutron-api.socket
```

Kiểm tra trực tiếp:

```bash
ls -l /var/run/uwsgi/neutron-api.socket
```

---

# 13. Test uWSGI thủ công

Trong quá trình troubleshooting có thể chạy:

```bash
sudo uwsgi --procname-prefix neutron-api \
    --ini /etc/neutron/neutron-api-uwsgi.ini
```

Kiểm tra:

```bash
ps -ef | grep '[u]wsgi.*neutron-api'
```

Kết quả có process:

```text
uwsgi --procname-prefix neutron-api --ini /etc/neutron/neutron-api-uwsgi.ini
```

Kiểm tra socket:

```bash
sudo ss -lxnp | grep neutron-api
```

Nếu socket tồn tại thì uWSGI đã start được.

---

# 14. Apache Neutron API configuration

File:

```text
/etc/apache2/sites-enabled/neutron-api.conf
```

Cấu hình cũ có dạng:

```apache
Listen 9696

<VirtualHost *:9696>

    WSGIDaemonProcess neutron-api processes=4 threads=1 user=neutron group=neutron display-name=${GROUP}

    WSGIProcessGroup neutron-api

    WSGIScriptAlias / /usr/bin/neutron-api

    WSGIApplicationGroup %{GLOBAL}

    WSGIPassAuthorization On

</VirtualHost>
```

Đây là mô hình:

```text
Apache
   |
   v
mod_wsgi
   |
   v
neutron-api
```

Trong quá trình lab chuyển sang mô hình:

```text
Apache
   |
   v
mod_proxy_uwsgi
   |
   v
uWSGI
   |
   v
neutron.wsgi.api:application
```

---

# 15. Apache configuration mới

File:

```text
/etc/apache2/sites-enabled/neutron-api.conf
```

Cấu hình:

```apache
Listen 9696

LogFormat "%h %l %u %t \"%r\" %>s %b \"%{Referer}i\" \"%{User-agent}i\" %D(us)" neutron_combined

<VirtualHost *:9696>

    WSGIDaemonProcess neutron-api processes=4 threads=1 user=neutron group=neutron display-name=${GROUP}

    WSGIProcessGroup neutron-api

    WSGIScriptAlias / /usr/bin/neutron-api

    WSGIApplicationGroup %{GLOBAL}

    WSGIPassAuthorization On

    LimitRequestBody 114688

    ErrorLog /var/log/neutron/neutron-api.log
    CustomLog /var/log/neutron/neutron_access.log neutron_combined

</VirtualHost>
```

Sau đó Apache được cấu hình để phục vụ Neutron API thông qua uWSGI socket.

---

# 16. Validate Apache configuration

```bash
sudo apachectl configtest
```

Kết quả:

```text
Syntax OK
```

Restart:

```bash
sudo systemctl restart apache2
```

Kiểm tra:

```bash
sudo systemctl status apache2 --no-pager
```

Expected:

```text
Active: active (running)
```

---

# 17. Kiểm tra port 9696

```bash
sudo ss -lntp | grep ':9696'
```

Kết quả:

```text
LISTEN 0 511 *:9696 *:*
users:(("apache2",...))
```

Điều này chứng minh Apache đang listen port 9696.

---

# 18. Test Neutron API bằng curl

```bash
curl -i http://10.168.36.11:9696/
```

Kết quả:

```text
HTTP/1.1 200 OK
Date: ...
Server: Apache/2.4.58 (Ubuntu)
Content-Type: application/json
```

Response:

```json
{
    "versions": [
        {
            "id": "v2.0",
            "status": "CURRENT"
        }
    ]
}
```

=> Neutron API đã hoạt động.

---

# 19. Lỗi 503 Service Unavailable

Sau khi Apache listen được:

```text
*:9696
```

nhưng uWSGI process/socket không chạy, lệnh:

```bash
openstack network agent list
```

trả:

```text
503 Service Unavailable
```

Điều này khác với:

```text
Connection refused
```

## Connection refused

```text
Client
   |
   X
   |
  :9696
```

Không có service listen port 9696.

## 503 Service Unavailable

```text
Client
   |
   v
Apache :9696
   |
   X
uWSGI socket
```

Apache hoạt động nhưng backend không hoạt động.

---

# 20. Kiểm tra uWSGI process khi gặp 503

```bash
ps -ef | grep '[u]wsgi.*neutron-api'
```

Nếu không có process thì kiểm tra socket:

```bash
sudo ss -lxnp | grep neutron-api
```

Nếu không có socket:

```text
Apache
  |
  v
:9696
  |
  X
/var/run/uwsgi/neutron-api.socket
```

=> uWSGI đang không chạy.

---

# 21. Tạo systemd service cho uWSGI

Tạo:

```text
/etc/systemd/system/neutron-api-uwsgi.service
```

Nội dung:

```ini
[Unit]
Description=OpenStack Neutron API uWSGI
After=network.target neutron-server.service
Wants=network.target

[Service]
Type=simple
User=root
Group=root
RuntimeDirectory=uwsgi
RuntimeDirectoryMode=0755
ExecStart=/usr/bin/uwsgi --procname-prefix neutron-api --ini /etc/neutron/neutron-api-uwsgi.ini
Restart=on-failure
RestartSec=5
KillSignal=SIGTERM
TimeoutStopSec=30

[Install]
WantedBy=multi-user.target
```

Reload:

```bash
sudo systemctl daemon-reload
```

Start:

```bash
sudo systemctl start neutron-api-uwsgi
```

Check:

```bash
sudo systemctl status neutron-api-uwsgi --no-pager
```

Expected:

```text
Active: active (running)
```

Enable:

```bash
sudo systemctl enable neutron-api-uwsgi
```

Check:

```bash
sudo systemctl is-enabled neutron-api-uwsgi
```

Expected:

```text
enabled
```

---

# 22. Kiểm tra lại uWSGI

```bash
ps -ef | grep '[u]wsgi.*neutron-api'
```

Kiểm tra socket:

```bash
sudo ss -lxnp | grep neutron-api
```

Expected:

```text
/var/run/uwsgi/neutron-api.socket
```

---

# 23. Kiểm tra Neutron API

```bash
curl -i http://10.168.36.11:9696/
```

Expected:

```text
HTTP/1.1 200 OK
```

---

# 24. Kiểm tra Neutron Agent

```bash
openstack network agent list
```

Kết quả cuối cùng:

```text
+--------------------------------------+----------------------+-----------+--------------------+-------+----------------+
| ID                                   | Agent Type           | Host      | Availability Zone  | Alive | State          |
+--------------------------------------+----------------------+-----------+--------------------+-------+----------------+
| ...                                  | OVN Controller agent | compute02 |                    | :-)   | UP             |
| ...                                  | OVN Controller agent | compute01 |                    | :-)   | UP             |
+--------------------------------------+----------------------+-----------+--------------------+-------+----------------+
```

Điều này chứng minh Neutron đã nhìn thấy OVN Controller trên:

```text
compute01
compute02
```

---

# 25. Kiểm tra OVN Controller trên Compute

Trên `compute01`:

```bash
sudo systemctl status ovn-controller --no-pager
```

Trên `compute02`:

```bash
sudo systemctl status ovn-controller --no-pager
```

Expected:

```text
Active: active (running)
```

---

# 26. Kiểm tra Open vSwitch

Trên Compute Node:

```bash
sudo ovs-vsctl show
```

Kiểm tra OVN controller:

```bash
sudo systemctl status ovn-controller --no-pager
```

---

# 27. Kiểm tra OVN database

Trên Controller:

```bash
ovn-nbctl show
```

```bash
ovn-sbctl show
```

Kiểm tra port:

```bash
sudo ss -lntp | grep -E '6641|6642'
```

Expected:

```text
10.168.36.11:6641
10.168.36.11:6642
```

---

# 28. Kiểm tra Neutron kết nối OVN

Trong log:

```text
/var/log/neutron/neutron-api.log
```

có các dòng:

```text
tcp:10.168.36.11:6641: connected
```

và:

```text
tcp:10.168.36.11:6642: connected
```

=> Neutron API/ML2 OVN đã kết nối được tới OVN database.

---

# 29. Tổng hợp các lỗi đã gặp

## Lỗi 1 - Port 9696 Connection Refused

### Symptom

```text
Connection refused
10.168.36.11:9696
```

### Kiểm tra

```bash
sudo systemctl status neutron-server
```

Neutron Server vẫn:

```text
active (running)
```

Nhưng:

```bash
sudo ss -lntp | grep 9696
```

không có output.

### Root cause

Neutron API chưa được expose đúng qua Apache/uWSGI.

### Solution

Triển khai:

```text
Apache
   |
   v
uWSGI
   |
   v
Neutron API
```

---

# Lỗi 2 - Apache 503

### Symptom

```text
503 Service Unavailable
```

### Root cause

Apache đang listen:

```text
*:9696
```

nhưng uWSGI không chạy hoặc socket không tồn tại.

### Kiểm tra

```bash
ps -ef | grep '[u]wsgi.*neutron-api'
```

```bash
sudo ss -lxnp | grep neutron-api
```

### Solution

Tạo systemd service:

```text
neutron-api-uwsgi.service
```

và:

```bash
sudo systemctl start neutron-api-uwsgi
```

---

# Lỗi 3 - Apache thiếu uWSGI proxy module

### Kiểm tra

```bash
sudo a2query -m proxy_uwsgi
```

### Solution

```bash
sudo apt install libapache2-mod-proxy-uwsgi
```

```bash
sudo a2enmod proxy
sudo a2enmod proxy_uwsgi
```

---

# Lỗi 4 - Geneve `max_header_size`

Ban đầu:

```ini
#max_header_size = 30
```

Đã sửa:

```ini
max_header_size = 38
```

Kiểm tra:

```bash
sudo grep -n -C 3 'max_header_size' /etc/neutron/plugins/ml2/ml2_conf.ini
```

---

# Lỗi 5 - Database migration

Kiểm tra:

```bash
sudo neutron-db-manage current
```

Upgrade:

```bash
sudo neutron-db-manage upgrade heads
```

Kiểm tra:

```bash
sudo neutron-db-manage current
```

Offline migration:

```bash
sudo neutron-db-manage has_offline_migrations
```

Expected:

```text
No offline migrations pending.
```

---

# 30. Bộ lệnh troubleshooting nhanh

Khi Neutron API lỗi, chạy theo thứ tự:

## Bước 1 - Neutron

```bash
sudo systemctl status neutron-server --no-pager
```

## Bước 2 - uWSGI

```bash
sudo systemctl status neutron-api-uwsgi --no-pager
```

## Bước 3 - uWSGI process

```bash
ps -ef | grep '[u]wsgi.*neutron-api'
```

## Bước 4 - uWSGI socket

```bash
sudo ss -lxnp | grep neutron-api
```

## Bước 5 - Apache

```bash
sudo systemctl status apache2 --no-pager
```

## Bước 6 - Port 9696

```bash
sudo ss -lntp | grep ':9696'
```

## Bước 7 - Apache config

```bash
sudo apachectl configtest
```

## Bước 8 - API

```bash
curl -i http://10.168.36.11:9696/
```

## Bước 9 - OpenStack CLI

```bash
openstack network list
```

## Bước 10 - Agent

```bash
openstack network agent list
```

---

