# Sử dụng ScaleIO
## Cài đặt SDC trên các node Controller, Compute
Đưa file `EMC-ScaleIO-sdc-2.0-14000.231.Ubuntu.16.04.x86_64.tar` vào các node và tar file. Extract deb file
```
./siob_extract [siob_file]
```
Cài đặt sdc
```
dpkg -i EMC-ScaleIO-sdc-2.0-14000.231.Ubuntu.16.04.x86_64.deb
```
Thêm IP mdm file `/bin/emc/scaleio/drv_cfg.txt`. Ví dụ
```
ini_guid 908312e6-aa0f-4f3c-bd3b-974957cd556f
mdm 10.5.9.61
```
Sửa file `/bin/emc/scaleio/scini_sync/driver_sync.conf`
```
# Repository address, prefixed by protocol
#repo_address = sftp://localhost/path/to/repo/dir
repo_address = ftp://ftp.emc.com/
#repo_address = ftp://localhost/path/to/repo/dir
#repo_address = file://local/path/to/repo/dir 

# Repository user (valid for ftp/sftp protocol)
repo_user = QNzgdxXix

# Repository password (valid for ftp protocol)
repo_password = Aw3wFAwAq3

# Local directory for modules
local_dir = /bin/emc/scaleio/scini_sync/driver_cache/

# User's RSA private key file (sftp protocol)
user_private_rsa_key = /bin/emc/scaleio/scini_sync/scini_key

# Repository host public key (sftp protocol)
repo_public_rsa_key = /bin/emc/scaleio/scini_sync/scini_repo_key.pub

# Should the fetched modules' signatures be checked [0, 1]
module_sigcheck = 0

# EMC public signature key (needed when module_sigcheck is 1)
emc_public_gpg_key = /bin/emc/scaleio/scini_sync/emc_key.pub

# Sync pattern (regular expression) for massive retrieve
sync_pattern = .*
```
Restart scini
```
service scini restart
```

## Cài đặt Cinder trên Controller
Sửa file `/etc/cinder/cinder.conf`
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
enabled_backends = scaleio
glance_api_servers = http://controller:9292
transport_url = rabbit://openstack:RABBIT_PASS@controller
my_ip = 172.16.78.23
auth_strategy = keystone


[database]
#connection = sqlite:////var/lib/cinder/cinder.sqlite
connection = mysql+pymysql://cinder:CINDER_DBPASS@controller/cinder

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

[oslo_concurrency]
# ...
lock_path = /var/lib/cinder/tmp

[scaleio]
volume_driver = cinder.volume.drivers.dell_emc.scaleio.driver.ScaleIODriver
volume_backend_name = scaleio
san_ip = 10.5.9.26
sio_protection_domain_name = domain1
sio_storage_pool_name = pool1
sio_storage_pools = domain1:pool1
san_login = admin
san_password = Vcc123**
```


