# Cài đặt dịch vụ Image (Glance) trên controller node
## Tạo database
Đăng nhập vào MariaDB với password root đã tạo
```
mysql -u root -p
```
Tạo database cho Glance và gán quyền truy cập
```
CREATE DATABASE glance;
GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'localhost' IDENTIFIED BY 'GLANCE_DBPASS';
GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'%' IDENTIFIED BY 'GLANCE_DBPASS';
```
## Tạo user, gán role và tạo endpoint API 
Chạy script biến môi trường
```
. admin-openrc
```
Tạo User Glance với password là 123456
```
root@controller:~# openstack user create --domain default --password-prompt glance
User Password:
Repeat User Password:
+---------------------+----------------------------------+
| Field               | Value                            |
+---------------------+----------------------------------+
| domain_id           | default                          |
| enabled             | True                             |
| id                  | f7808557d9534d2badd225707a4fbed2 |
| name                | glance                           |
| options             | {}                               |
| password_expires_at | None                             |
+---------------------+----------------------------------+
```
Add `glance` user vào project `service` với role `admin`
```
openstack role add --project service --user glance admin
```
Tạo entity service có tên `glance` 
```
openstack service create --name glance --description "OpenStack Image" image
+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | OpenStack Image                  |
| enabled     | True                             |
| id          | 96b7e2a357774be09b682451acf54e35 |
| name        | glance                           |
| type        | image                            |
+-------------+----------------------------------+
```
Tạo Endpoint cho dịch vụ Glance
```
openstack endpoint create --region RegionOne image public http://controller:9292
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | e66c6106a18b4b64a5fccc681820353b |
| interface    | public                           |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | 96b7e2a357774be09b682451acf54e35 |
| service_name | glance                           |
| service_type | image                            |
| url          | http://controller:9292           |
+--------------+----------------------------------+

openstack endpoint create --region RegionOne image internal http://controller:9292
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | 71480ecde1944eb4b2afa4c829976188 |
| interface    | internal                         |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | 96b7e2a357774be09b682451acf54e35 |
| service_name | glance                           |
| service_type | image                            |
| url          | http://controller:9292           |
+--------------+----------------------------------+

openstack endpoint create --region RegionOne image admin http://controller:9292
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | 704e10a8d02444f5a3a7c608e6ca2888 |
| interface    | admin                            |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | 96b7e2a357774be09b682451acf54e35 |
| service_name | glance                           |
| service_type | image                            |
| url          | http://controller:9292           |
+--------------+----------------------------------+
```

## Cài đặt và cấu hình các thành phần của Glance
Cài đặt glance
```
apt install -y glance
```
### Cấu hình Glance API
Trong file `/etc/glance/glance-api.conf` cấu hình như sau:  
- Phần `[database]` cấu hình truy cập database  

```
connection = mysql+pymysql://glance:GLANCE_DBPASS@controller/glance
```
- Phần `[keystone_authtoken]` và `[paste_deploy]` cấu hình truy cập KeyStone

```
[keystone_authtoken]
auth_uri = http://controller:5000
auth_url = http://controller:35357
memcached_servers = controller:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = glance
password = 123456

[paste_deploy]
flavor = keystone
```
- Phần `[glance_store]` cấu hình lưu trữ cục bộ và đường dẫn của image file  

```
[glance_store]
stores = file,http
default_store = file
filesystem_store_datadir = /var/lib/glance/images/
```

### Cấu hình Glance registry
Trong file `/etc/glance/glance-registry.conf` cấu hình như sau:  
- Phần `[database]` cấu hình truy cập database

```
connection = mysql+pymysql://glance:GLANCE_DBPASS@controller/glance
```
- Phần `[keystone_authtoken]` và `[paste_deploy]` cấu hình truy cập KeyStone

```
[keystone_authtoken]
auth_uri = http://controller:5000
auth_url = http://controller:35357
memcached_servers = controller:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = glance
password = 123456

[paste_deploy]
flavor = keystone
```

### Đồng bộ database 
```
su -s /bin/sh -c "glance-manage db_sync" glance
```
Restart Glance
```
service glance-registry restart
service glance-api restart
```

## Verify Operation
Chạy script biến môi trường
```
. admin-openrc
```
Tải source image
```
wget http://download.cirros-cloud.net/0.3.5/cirros-0.3.5-x86_64-disk.img
```
Upload image vào Glance bằng định dạng QCOW2, bare container format, và public để tất cả các project có thể truy cập
```
openstack image create "cirros" \
  --file cirros-0.3.5-x86_64-disk.img \
  --disk-format qcow2 --container-format bare \
  --public
+------------------+------------------------------------------------------+
| Field            | Value                                                |
+------------------+------------------------------------------------------+
| checksum         | f8ab98ff5e73ebab884d80c9dc9c7290                     |
| container_format | bare                                                 |
| created_at       | 2018-01-02T10:57:17Z                                 |
| disk_format      | qcow2                                                |
| file             | /v2/images/6f7ad56c-3d46-4fe2-9e1b-b49ec75d0791/file |
| id               | 6f7ad56c-3d46-4fe2-9e1b-b49ec75d0791                 |
| min_disk         | 0                                                    |
| min_ram          | 0                                                    |
| name             | cirros                                               |
| owner            | 6e09da450c9e4d4ebdd71c74499e86be                     |
| protected        | False                                                |
| schema           | /v2/schemas/image                                    |
| size             | 13267968                                             |
| status           | active                                               |
| tags             |                                                      |
| updated_at       | 2018-01-02T10:57:17Z                                 |
| virtual_size     | None                                                 |
| visibility       | public                                               |
+------------------+------------------------------------------------------+

```
Kiểm tra lại xem image đã có hay chưa
```
openstack image list
+--------------------------------------+--------+--------+
| ID                                   | Name   | Status |
+--------------------------------------+--------+--------+
| 6f7ad56c-3d46-4fe2-9e1b-b49ec75d0791 | cirros | active |
+--------------------------------------+--------+--------+
```


