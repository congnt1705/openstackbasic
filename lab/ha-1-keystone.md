# Cài đặt dịch vụ Identity (KeyStone) trên 3 node controller
## Tạo Database
Việc tạo database và user chỉ cần làm trên 1 node, các node khác sẽ tự đồng bộ. Đăng nhập vào MySQL với password root đã tạo
```
mysql -u root -p
```
Tạo database cho KeyStone và gán quyền truy cập
```
CREATE DATABASE keystone;
GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'localhost' IDENTIFIED BY 'KEYSTONE_DBPASS';
GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'%' IDENTIFIED BY 'KEYSTONE_DBPASS';
```
## Cài đặt và cấu hình Keystone
Cài đặt và cấu hình keystone trên cả 3 node.
```
apt install -y keystone  apache2 libapache2-mod-wsgi
```
Cấu hình KeyStone trong file `/etc/keystone/keystone.conf` như sau:
```
[DEFAULT]
log_dir = /var/log/keystone
[database]
connection = mysql+pymysql://keystone:KEYSTONE_DBPASS@controller/keystone
[token]
provider = fernet
```
Đồng bộ database cho KeyStone (1 node)
```
su -s /bin/sh -c "keystone-manage db_sync" keystone
```
Khởi tạo Fernet key repo (3 node)
```
keystone-manage fernet_setup --keystone-user keystone --keystone-group keystone
keystone-manage credential_setup --keystone-user keystone --keystone-group keystone
```
Sau khi khởi tạo Fernet key cần đưa fernet key từ 1 node sang 2 node còn lại vì nếu không đồng bộ fernet key khi sử dụng haproxy roundrobin sẽ không authen được.  
Khởi động KeyStone (1 node)
```
keystone-manage bootstrap --bootstrap-password ADMIN_PASS \
  --bootstrap-admin-url http://controller:5000/v3/ \
  --bootstrap-internal-url http://controller:5000/v3/ \
  --bootstrap-public-url http://controller:5000/v3/ \
  --bootstrap-region-id RegionOne
```
## Cấu hình Apache HTTP Server
Thêm dòng sau vào cuối  file `/etc/apache2/apache2.conf` trên cả 3 node controller
```
ServerName controller
```
Restart apache
```
service apache2 restart
```
Export biến môi trường để chạy command
```
export OS_USERNAME=admin
export OS_PASSWORD=ADMIN_PASS
export OS_PROJECT_NAME=admin
export OS_USER_DOMAIN_NAME=Default
export OS_PROJECT_DOMAIN_NAME=Default
export OS_AUTH_URL=http://controller:5000/v3
export OS_IDENTITY_API_VERSION=3
```

## Create domain, project, users, roles
Tạo project service nằm trong domain default
```
openstack project create --domain default --description "Service Project" service
+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | Service Project                  |
| domain_id   | default                          |
| enabled     | True                             |
| id          | 6295d0455d7e4c159cd41a1d4b5a8bf1 |
| is_domain   | False                            |
| name        | service                          |
| parent_id   | default                          |
+-------------+----------------------------------+
```
Các nhiệm vụ thông thường nên sử dụng project & user không có đặc quyền. Tạo project `demo` và user `demo` có password là `123456`. Sau đó tạo role có tên là `user` và gán role này vào project và user `demo`.
```
# Tạo project demo nằm trong domain default
openstack project create --domain default --description "Demo Project" demo
+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | Demo Project                     |
| domain_id   | default                          |
| enabled     | True                             |
| id          | 3275c6545a6347658bd028569a1386ba |
| is_domain   | False                            |
| name        | demo                             |
| parent_id   | default                          |
+-------------+----------------------------------+

# Tạo user demo và đặt password. Ở đây dùng pass 123456
openstack user create --domain default --password-prompt demo
User Password:
Repeat User Password:
+---------------------+----------------------------------+
| Field               | Value                            |
+---------------------+----------------------------------+
| domain_id           | default                          |
| enabled             | True                             |
| id                  | a38089449c7c433f9c75216b24737f17 |
| name                | demo                             |
| options             | {}                               |
| password_expires_at | None                             |
+---------------------+----------------------------------+

# Tạo role với tên user
openstack role create user
+-----------+----------------------------------+
| Field     | Value                            |
+-----------+----------------------------------+
| domain_id | None                             |
| id        | c2855bfbf57f4f399b285aba82ff9e54 |
| name      | user                             |
+-----------+----------------------------------+

# Add user demo vào project demo
openstack role add --project demo --user demo user
```

## Verify Operation
Bỏ cài đắt các biến môi trường tạm thời
```
unset OS_AUTH_URL OS_PASSWORD
```
Yêu cầu Token bằng `admin` với pass là ADMIN_PASS
```
openstack --os-auth-url http://controller:5000/v3 \
--os-project-domain-name Default --os-user-domain-name Default \
--os-project-name admin --os-username admin token issue

Password:
+------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| Field      | Value                                                                                                                                                                                   |
+------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| expires    | 2018-01-02T10:43:14+0000                                                                                                                                                                |
| id         | gAAAAABaS1Qy5rcKBwVcoSTaNl5zZgXy7YZXZT37DLN1WYBbXlHu96jiAP3DdmvEcaP90rH3y10DK0cZUzAwWX-YsOxiMdhT0LgFbLOkzv7yIx1nSIvb92LR_RQdGfY3t4lWBxBLO_IhDn4C3JyNm9MC0zAwPw-qrGoZsTW0zUhMbqeAc-uT8N0 |
| project_id | 6e09da450c9e4d4ebdd71c74499e86be                                                                                                                                                        |
| user_id    | b3082ea3decf453b9ee7e34384a8568b                                                                                                                                                        |
+------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
```
Yêu cầu Token bằng user `demo` với pass là 123456
```
openstack --os-auth-url http://controller:5000/v3 \
--os-project-domain-name Default --os-user-domain-name Default \
--os-project-name demo --os-username demo token issue
Password: 
+------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| Field      | Value                                                                                                                                                                                   |
+------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| expires    | 2018-01-02T10:49:08+0000                                                                                                                                                                |
| id         | gAAAAABaS1WUGaCjAFMXFBrzrdBORvgK7ezV2yYUuA5h22FhgFAC2I_LWHCyC3ZyqeDI1cYRtKUYhP71jW1XOSJAyPYdKzImGkxr4b1UkZc6kCCfWuxbqALKetQyXPAWVMeeZ9YSb61VK_W5DPCo8woafZ2j-1SMmlGNWD3eYAWSPnm00vDcGg4 |
| project_id | 3275c6545a6347658bd028569a1386ba                                                                                                                                                        |
| user_id    | a38089449c7c433f9c75216b24737f17                                                                                                                                                        |
+------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
```

## Create OpenStack client environment scripts
Tạo Script cho admin user trong file `admin-openrc`
```
export OS_PROJECT_DOMAIN_NAME=Default
export OS_USER_DOMAIN_NAME=Default
export OS_PROJECT_NAME=admin
export OS_USERNAME=admin
export OS_PASSWORD=ADMIN_PASS
export OS_AUTH_URL=http://controller:5000/v3
export OS_IDENTITY_API_VERSION=3
export OS_IMAGE_API_VERSION=2
```
Tạo Script cho demo user trong file `demo-openrc`
```
export OS_PROJECT_DOMAIN_NAME=Default
export OS_USER_DOMAIN_NAME=Default
export OS_PROJECT_NAME=demo
export OS_USERNAME=demo
export OS_PASSWORD=123456
export OS_AUTH_URL=http://controller:5000/v3
export OS_IDENTITY_API_VERSION=3
export OS_IMAGE_API_VERSION=2
```
