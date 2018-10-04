# Cài đặt dịch vụ block storage (Cinder) tich hợp Ceph RBD. Đồng thời tích hợp glance vào Ceph
## Prepare Ceph. 
Trước tiên cần 1 cụm Ceph đang chạy trên Ceph Node. Tham khảo link [này](http://docs.ceph.com/docs/mimic/start/) để cài đặt Ceph bằng ceph-deploy.  
Sau khi có 1 Ceph Cluster chạy, trước tiên cần tạo pool cần dùng
```
ceph osd pool create volumes 128
ceph osd pool create images 128
ceph osd pool create vms 128
```
Các pool mới cần được khởi tạo trước khi sử dụng
```
rbd pool init volumes
rbd pool init images
rbd pool init vms
```
Cài đặt python-rbd trên 3 node controller
```
sudo apt-get install python-rbd
```
Cài đặt ceph-common trên 3 node controller và 3 node compute
```
sudo apt-get install ceph-common -y
```
Trên node ceph đửa file ceph.conf vào controller và compute
```
for i in {1..3}; do scp /etc/ceph/ceph.conf controller$i:/etc/ceph/ 
for i in {1..3}; do scp /etc/ceph/ceph.conf compute$i:/etc/ceph/ 
```
Tạo user trên ceph quản lý các pools
```
ceph auth get-or-create client.glance mon 'profile rbd' osd 'profile rbd pool=images'
ceph auth get-or-create client.cinder mon 'profile rbd' osd 'profile rbd pool=volumes, profile rbd pool=vms, profile rbd pool=images'
```
Add keyring vào các node controller và change quyền
```
for i in {1..3}; do
  ceph auth get-or-create client.cinder | ssh controller$i sudo tee /etc/ceph/ceph.client.cinder.keyring
  ssh controller$i sudo chown cinder:cinder /etc/ceph/ceph.client.cinder.keyring;
  ceph auth get-or-create client.glance | ssh controller$i sudo tee /etc/ceph/ceph.client.glance.keyring
  ssh controller$i sudo chown glance:glance /etc/ceph/ceph.client.glance.keyring
done
```
Đưa key vào các node compute
```
for i in {1..3}; do
  ceph auth get-or-create client.cinder | ssh compute$i sudo tee /etc/ceph/ceph.client.cinder.keyring
  ceph auth get-key client.cinder | ssh compute$i tee client.cinder.key;
done
```
Trên 3 node compute làm như sau:
```
cat > secret.xml <<EOF
<secret ephemeral='no' private='no'>
  <uuid>6c10f553-08a5-46e0-a885-a251d67e5f06</uuid>
  <usage type='ceph'>
    <name>client.cinder secret</name>
  </usage>
</secret>
EOF

sudo virsh secret-define --file secret.xml
Secret 6c10f553-08a5-46e0-a885-a251d67e5f06 createda

sudo virsh secret-set-value --secret 6c10f553-08a5-46e0-a885-a251d67e5f06 --base64 $(cat client.cinder.key) && rm client.cinder.key secret.xml
```
Trên 3 node compute thêm section sau vào file `/etc/nova/nova.conf`. **Lưu ý mục rbd_secret_uuid phải cấu hình theo uuid chuẩn bị ở trên**
```
[libvirt]
images_type = rbd
images_rbd_pool = vms
images_rbd_ceph_conf = /etc/ceph/ceph.conf
rbd_user = cinder
rbd_secret_uuid = 6c10f553-08a5-46e0-a885-a251d67e5f06
disk_cachemodes="network=writeback"
inject_password = false
inject_key = false
inject_partition = -2
live_migration_flag="VIR_MIGRATE_UNDEFINE_SOURCE,VIR_MIGRATE_PEER2PEER,VIR_MIGRATE_LIVE,VIR_MIGRATE_PERSIST_DEST,VIR_MIGRATE_TUNNELLED"
hw_disk_discard = unmap
```
## Cầu hình Glance sử dụng Ceph trên 3 node controller
Trong file `/etc/glance/glance-api.conf` thêm section sau:
```
[glance_store]
stores = rbd
default_store = rbd
rbd_store_pool = images
rbd_store_user = glance
rbd_store_ceph_conf = /etc/ceph/ceph.conf
rbd_store_chunk_size = 8
```
Thêm cấu hình sau vào section [DEFAULT] để enable copy-on-write
```
[DEFAULT]
show_image_direct_url = True`
```
Restart glance để áp dụng cấu hình
```
service glance-api restart
```
Download cirros image, convert image sang raw và tạo image
```
wget http://download.cirros-cloud.net/0.3.4/cirros-0.3.4-x86_64-disk.img
qemu-img convert cirros-0.3.4-x86_64-disk.img cirros-0.3.4-x86_64-disk.raw
openstack image create "Cirros 0.3.4" \
  --file cirros-0.3.4-x86_64-disk.raw \
  --disk-format raw --container-format bare \
  --public
```

## Prepare Cinder. Thực hiện lệnh sau trên 1 trong 3 node Controller
Đăng nhập vào MySQL với password root đã tạo
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
## Cài đặt và cấu hình Cinder trên 3 node Controller
Cài đặt các thành phần của cinder
```
apt install -y cinder-volume cinder-api cinder-scheduler thin-provisioning-tools
```
Trong file `/etc/cinder/cinder.conf` cấu hình như sau.**Thay my_ip bằng IP management của Controller đang cấu hình Và Chú ý cấu hình rbd_secret_uuid (Nếu bước prepare Ceph trên Compute bạn sử dụng UUID khác bạn phải cấu hình UUID đó vào config này**:  
```
[DEFAULT]
rootwrap_config = /etc/cinder/rootwrap.conf
api_paste_confg = /etc/cinder/api-paste.ini
iscsi_helper = tgtadm
volume_name_template = volume-%s
volume_group = cinder-volumes
verbose = True
state_path = /var/lib/cinder
lock_path = /var/lock/cinder
volumes_dir = /var/lib/cinder/volumes
enabled_backends = ceph
glance_api_version = 2
transport_url = rabbit://openstack:RABBIT_PASS@controller1,openstack:RABBIT_PASS@controller2,openstack:RABBIT_PASS@controller3,
auth_strategy = keystone
my_ip = 172.16.69.90
glance_api_servers = http://controller:9292
[database]
connection = mysql+pymysql://cinder:CINDER_DBPASS@controller/cinder
[keystone_authtoken]
auth_url = http://controller:5000
memcached_servers = controller:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = cinder
password = 123456
[oslo_concurrency]
lock_path = /var/lib/cinder/tmp
[ceph]
volume_driver = cinder.volume.drivers.rbd.RBDDriver
rbd_cluster_name = ceph
rbd_pool = volumes
rbd_user = cinder
rbd_ceph_conf = /etc/ceph/ceph.conf
rbd_flatten_volume_from_snapshot = false
rbd_secret_uuid = 6c10f553-08a5-46e0-a885-a251d67e5f06
rbd_max_clone_depth = 5
rbd_store_chunk_size = 4
rados_connect_timeout = -1
```
Populate Cinder database. **Thực hiện trên 1 trong 3 node Controller là được**
```
su -s /bin/sh -c "cinder-manage db sync" cinder
```
Restart các thành phần sau trên controller
```
service cinder-scheduler restart
service cinder-volume restart
service glance-api restart
service apache2 restart
```
Restart nova-compute trên compute node
```
service nova-compute restart
```
