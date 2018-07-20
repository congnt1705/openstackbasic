# Cài đặt dịch vụ block storage (Cinder) trên Controller
Đăng nhập vào MariaDB với password root đã tạo
```
mysql -u root -p
```
Tạo database cho Cinder và gán quyền truy cập
```
CREATE DATABASE cinder;
GRANT ALL PRIVILEGES ON cinder.* TO 'cinder'@'localhost' IDENTIFIED BY 'CINDER_DBPASS';
GRANT ALL PRIVILEGES ON cinder.* TO 'cinder'@'%' IDENTIFIED BY 'CINDER_DBPASS';
```
Chạy script biến môi trường của user `admin`
```
. admin-openrc
```
Tạo user `cinder` với pass là 123456
```
openstack user create --domain default --password-prompt cinder
+---------------------+----------------------------------+
| Field               | Value                            |
+---------------------+----------------------------------+
| domain_id           | default                          |
| enabled             | True                             |
| id                  | 1ae6ca104fc0408392d16c4036c5878b |
| name                | cinder                           |
| options             | {}                               |
| password_expires_at | None                             |
+---------------------+----------------------------------+
```
Add user `cinder` vào project `service` với role `admin`
```
openstack role add --project service --user cinder admin
```
Tạo service `cinderv2` và `cinderv3`
```
openstack service create --name cinderv2 --description "OpenStack Block Storage" volumev2
+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | OpenStack Block Storage          |
| enabled     | True                             |
| id          | 8056cd7bc6914f7d9e092ba38d6d336d |
| name        | cinderv2                         |
| type        | volumev2                         |
+-------------+----------------------------------+

openstack service create --name cinderv3 --description "OpenStack Block Storage" volumev3
+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | OpenStack Block Storage          |
| enabled     | True                             |
| id          | 8b0bed0ec73d4d5186e482a2479e3c78 |
| name        | cinderv3                         |
| type        | volumev3                         |
+-------------+----------------------------------+
```
Tạo Endpoint cho Cinder
```
openstack endpoint create --region RegionOne \
  volumev2 public http://controller:8776/v2/%\(project_id\)s
+--------------+------------------------------------------+
| Field        | Value                                    |
+--------------+------------------------------------------+
| enabled      | True                                     |
| id           | 29af87dc8bfe410da0d21144a44010ef         |
| interface    | public                                   |
| region       | RegionOne                                |
| region_id    | RegionOne                                |
| service_id   | 8056cd7bc6914f7d9e092ba38d6d336d         |
| service_name | cinderv2                                 |
| service_type | volumev2                                 |
| url          | http://controller:8776/v2/%(project_id)s |
+--------------+------------------------------------------+

openstack endpoint create --region RegionOne \
  volumev2 internal http://controller:8776/v2/%\(project_id\)s
+--------------+------------------------------------------+
| Field        | Value                                    |
+--------------+------------------------------------------+
| enabled      | True                                     |
| id           | 1eb535c58663444284caa750d8073181         |
| interface    | internal                                 |
| region       | RegionOne                                |
| region_id    | RegionOne                                |
| service_id   | 8056cd7bc6914f7d9e092ba38d6d336d         |
| service_name | cinderv2                                 |
| service_type | volumev2                                 |
| url          | http://controller:8776/v2/%(project_id)s |
+--------------+------------------------------------------+

openstack endpoint create --region RegionOne \
  volumev2 admin http://controller:8776/v2/%\(project_id\)s
+--------------+------------------------------------------+
| Field        | Value                                    |
+--------------+------------------------------------------+
| enabled      | True                                     |
| id           | 387f7258895c4f9b811857fb38549603         |
| interface    | admin                                    |
| region       | RegionOne                                |
| region_id    | RegionOne                                |
| service_id   | 8056cd7bc6914f7d9e092ba38d6d336d         |
| service_name | cinderv2                                 |
| service_type | volumev2                                 |
| url          | http://controller:8776/v2/%(project_id)s |
+--------------+------------------------------------------+

openstack endpoint create --region RegionOne \
  volumev3 public http://controller:8776/v3/%\(project_id\)s
+--------------+------------------------------------------+
| Field        | Value                                    |
+--------------+------------------------------------------+
| enabled      | True                                     |
| id           | 681816aca2ec4b158b65a71153ac1c18         |
| interface    | public                                   |
| region       | RegionOne                                |
| region_id    | RegionOne                                |
| service_id   | 8b0bed0ec73d4d5186e482a2479e3c78         |
| service_name | cinderv3                                 |
| service_type | volumev3                                 |
| url          | http://controller:8776/v3/%(project_id)s |
+--------------+------------------------------------------+

openstack endpoint create --region RegionOne \
  volumev3 internal http://controller:8776/v3/%\(project_id\)s
+--------------+------------------------------------------+
| Field        | Value                                    |
+--------------+------------------------------------------+
| enabled      | True                                     |
| id           | dfc5f91dc8aa4658a103868d5e26b418         |
| interface    | internal                                 |
| region       | RegionOne                                |
| region_id    | RegionOne                                |
| service_id   | 8b0bed0ec73d4d5186e482a2479e3c78         |
| service_name | cinderv3                                 |
| service_type | volumev3                                 |
| url          | http://controller:8776/v3/%(project_id)s |
+--------------+------------------------------------------+

openstack endpoint create --region RegionOne \
  volumev3 admin http://controller:8776/v3/%\(project_id\)s
+--------------+------------------------------------------+
| Field        | Value                                    |
+--------------+------------------------------------------+
| enabled      | True                                     |
| id           | 9c7e1db7cb9e45af8b058c03a1877f41         |
| interface    | admin                                    |
| region       | RegionOne                                |
| region_id    | RegionOne                                |
| service_id   | 8b0bed0ec73d4d5186e482a2479e3c78         |
| service_name | cinderv3                                 |
| service_type | volumev3                                 |
| url          | http://controller:8776/v3/%(project_id)s |
+--------------+------------------------------------------+
```
Cài đặt các thành phần của cinder
```
apt install -y cinder-volume cinder-api cinder-scheduler thin-provisioning-tools
```
Trong file `/etc/cinder/cinder.conf` cấu hình như sau:  
- [database] cấu hình truy cập database  

```
#connection = sqlite:////var/lib/cinder/cinder.sqlite
connection = mysql+pymysql://cinder:CINDER_DBPASS@controller/cinder
```
- [DEFAULT] cấu hình truy cập RABBIT MQ và cấu hình sử dụng IP manage của Controller 

```
transport_url = rabbit://openstack:RABBIT_PASS@controller
my_ip = 10.10.10.11
```
- Cấu hình truy cập KeyStone  

```
[DEFAULT]
# ...
auth_strategy = keystone

[keystone_authtoken]
# ...
auth_uri = http://controller:5000
auth_url = http://controller:35357
memcached_servers = controller:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = cinder
password = 123456
```
- Cấu hình lock path 

```
[oslo_concurrency]
# ...
lock_path = /var/lib/cinder/tmp
```
- Cấu hình vị trí của Glance API  

```
glance_api_servers = http://controller:9292
```
- Config LVM backend với LVM driver, volume group sử dụng `cinder-volume`, iSCSI protocol  

```
[lvm]
# ...
volume_driver = cinder.volume.drivers.lvm.LVMVolumeDriver
volume_group = cinder-volumes
iscsi_protocol = iscsi
iscsi_helper = tgtadm
```
Populate Cinder database
```
su -s /bin/sh -c "cinder-manage db sync" cinder
```
Cài đặt lvm
```
apt install -y lvm2
```
Tạo lvm physical volume 
```
pvcreate /dev/sdb
```
Tạo volume group tên `cinder-volumes`
```
vgcreate cinder-volumes /dev/sdb
```
Trong file `/etc/lvm/lvm.conf`, `devices` section cấu hình filter `/dev/sdb` và từ chối các thiết bị khác
```
devices {
#...
filter = [ "a/sdb/", "r/.*/"]
```
Restart các thành phần
```
service nova-api restart
service cinder-scheduler restart
service apache2 restart
service tgt restart
service cinder-volume restart
```

