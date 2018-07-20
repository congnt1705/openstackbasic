# Cài đặt dịch vụ Networking (Neutron)
## Cài đặt Neutron trên Controller
Đăng nhập vào MariaDB với password root đã tạo
```
mysql -u root -p
```
Tạo database cho Neutron và gán quyền truy cập
```
CREATE DATABASE neutron;
GRANT ALL PRIVILEGES ON neutron.* TO 'neutron'@'localhost' IDENTIFIED BY 'NEUTRON_DBPASS';
GRANT ALL PRIVILEGES ON neutron.* TO 'neutron'@'%' IDENTIFIED BY 'NEUTRON_DBPASS';
```
Chạy script biến môi trường `
```
. admin-openrc
```
Tạo user `neutron` với password 123456
```
openstack user create --domain default --password-prompt neutron 
User Password:
Repeat User Password:
+---------------------+----------------------------------+
| Field               | Value                            |
+---------------------+----------------------------------+
| domain_id           | default                          |
| enabled             | True                             |
| id                  | fe4f776ff8574281b49902914f1469ff |
| name                | neutron                          |
| options             | {}                               |
| password_expires_at | None                             |
+---------------------+----------------------------------+
```
Add user `neutron` vào project `service` với role `admin`  
```
openstack role add --project service --user neutron admin
```
Tạo Neutron service
```
openstack service create --name neutron --description "OpenStack Networking" network
+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | OpenStack Networking             |
| enabled     | True                             |
| id          | 674d138c046a4de4b23b1e063db34cab |
| name        | neutron                          |
| type        | network                          |
+-------------+----------------------------------+
```
Tạo Endpoint cho Neutron
```
openstack endpoint create --region RegionOne network public http://controller:9696
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | b8a07b1f9e7f4a59996e91a373e9c76b |
| interface    | public                           |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | 674d138c046a4de4b23b1e063db34cab |
| service_name | neutron                          |
| service_type | network                          |
| url          | http://controller:9696           |
+--------------+----------------------------------+

openstack endpoint create --region RegionOne network internal http://controller:9696
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | 1a476497249e41698b14227ce68c8150 |
| interface    | internal                         |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | 674d138c046a4de4b23b1e063db34cab |
| service_name | neutron                          |
| service_type | network                          |
| url          | http://controller:9696           |
+--------------+----------------------------------+

openstack endpoint create --region RegionOne network admin http://controller:9696
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | 7bcbaefc02a4408b817b43c10d37ae97 |
| interface    | admin                            |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | 674d138c046a4de4b23b1e063db34cab |
| service_name | neutron                          |
| service_type | network                          |
| url          | http://controller:9696           |
+--------------+----------------------------------+
```
### Cấu hình neutron trên Controller
Cài đặt neutron server
```
apt install -y neutron-server
```
Trong file `/etc/neutron/neutron.conf` cấu hình như sau:
- [database] section, cấu hình neutron truy cập database

```
[database]
#connection = sqlite:////var/lib/neutron/neutron.sqlite
connection = mysql+pymysql://neutron:NEUTRON_DBPASS@controller/neutron
```
- [DEFAULT] section, enable Modular Layer 2 (ML2) plugin, router service, overlapping IP và cấu hình truy cập RABBIT MQ

```
[DEFAULT]
core_plugin = ml2
service_plugins = router
allow_overlapping_ips = true
transport_url = rabbit://openstack:RABBIT_PASS@controller
```
- Cấu hình truy cập KeyStone

```
[DEFAULT]
auth_strategy = keystone

[keystone_authtoken]
auth_uri = http://controller:5000
auth_url = http://controller:35357
memcached_servers = controller:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = neutron
password = 123456
```
- Section [DEFAULT] và [nova] cấu hình để thông báo khi Compute thay đổi topo mạng

```
[DEFAULT]
notify_nova_on_port_status_changes = true
notify_nova_on_port_data_changes = true

[nova]
auth_url = http://controller:35357
auth_type = password
project_domain_name = default
user_domain_name = default
region_name = RegionOne
project_name = service
username = nova
password = 123456
```
### Cấu hình ML2 Plug-in trên Controller
Trong file `/etc/neutron/plugins/ml2/ml2_conf.ini` cấu hình như sau:
- Enable flat, vlan, vxlan trong [ml2] section
- Enable VXLAN self-service trong [ml2] section
- Enable Linux Bridge & Layer 2 population trong [ml2] section
- Enable port-security trong [ml2] section

```
[ml2]
type_drivers = flat,vlan,vxlan
tenant_network_types = vxlan
mechanism_drivers = linuxbridge,l2population
extension_drivers = port_security
```
- [ml2_type_flat] cấu hình mạng ảo provider như flat network

```
[ml2_type_flat]
flat_networks = provider
```
- [ml2_type_vxlan] cấu hình range VXLAN cho self-service network

```
[ml2_type_vxlan]
vni_ranges = 1:1000
```
- Enable ipset để tăng hiệu quả của các rule security group trong phần [security group]

```
[securitygroup]
enable_ipset = true
```
### Cấu hình Compute Service sử dụng Networking trên Controller Node
Edit file `/etc/nova/nova.conf`, section [neutron] 
```
[neutron]
# ...
url = http://controller:9696
auth_url = http://controller:35357
auth_type = password
project_domain_name = default
user_domain_name = default
region_name = RegionOne
project_name = service
username = neutron
password = 123456
service_metadata_proxy = true
metadata_proxy_shared_secret = METADATA_SECRET
```

## Cài đặt và cấu hình Neutron trên Network Node sử dụng self service network
Cài đặt các thành phần neutron
```
apt install -y neutron-server neutron-plugin-ml2 neutron-linuxbridge-agent neutron-l3-agent neutron-dhcp-agent neutron-metadata-agent
```
### Cấu hình Neutron trên Network Node
Trong file `/etc/neutron/neutron.conf` cấu hình như sau:  
- [database] section, cấu hình neutron truy cập database  

```
[database]
#connection = sqlite:////var/lib/neutron/neutron.sqlite
connection = mysql+pymysql://neutron:NEUTRON_DBPASS@controller/neutron
```
- [DEFAULT] section, enable Modular Layer 2 (ML2) plugin, router service, overlapping IP và cấu hình truy cập RABBIT MQ

```
[DEFAULT]
core_plugin = ml2
service_plugins = router
allow_overlapping_ips = true
transport_url = rabbit://openstack:RABBIT_PASS@controller
```
- Cấu hình truy cập KeyStone 

```
[DEFAULT]
auth_strategy = keystone

[keystone_authtoken]
auth_uri = http://controller:5000
auth_url = http://controller:35357
memcached_servers = controller:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = neutron
password = 123456
```
- Section [DEFAULT] và [nova] cấu hình để thông báo khi Compute thay đổi topo mạng

```
[DEFAULT]
notify_nova_on_port_status_changes = true
notify_nova_on_port_data_changes = true

[nova]
auth_url = http://controller:35357
auth_type = password
project_domain_name = default
user_domain_name = default
region_name = RegionOne
project_name = service
username = nova
password = 123456
```
### Cấu hình ML2 Plug-in trên Network Node
Trong file `/etc/neutron/plugins/ml2/ml2_conf.ini` cấu hình như sau:  
- Enable flat, vlan, vxlan trong [ml2] section  
- Enable VXLAN self-service trong [ml2] section  
- Enable Linux Bridge & Layer 2 population trong [ml2] section  
- Enable port-security trong [ml2] section  

```
[ml2]
type_drivers = flat,vlan,vxlan
tenant_network_types = vxlan
mechanism_drivers = linuxbridge,l2population
extension_drivers = port_security
```
- [ml2_type_flat] cấu hình mạng ảo provider như flat network  

```
[ml2_type_flat]
flat_networks = provider
```
- [ml2_type_vxlan] cấu hình range VXLAN cho self-service network  

```
[ml2_type_vxlan]
vni_ranges = 1:1000
```
- Enable ipset để tăng hiệu quả của các rule security group trong phần [security group]  

```
[securitygroup]
enable_ipset = true
```

### Cấu hình Linux Bridge 
Trong file `/etc/neutron/plugins/ml2/linuxbridge_agent.ini` cấu hình như sau:  
- Trong [linux_bridge], map provider virtual network vào cổng vật lý của mạng provider (Lab này là ens33)  

```
[linux_bridge]
physical_interface_mappings = provider:ens33
```
- Enable VXLAN, cấu hình IP của giao diện mạng sẽ xử lý các data overlay network (Lab này là IP management của Network node)  

```
[vxlan]
enable_vxlan = true
local_ip = 10.10.10.31
l2_population = true
```
- Enable security group và cấu hình driver Linux Bridge iptables firewall  

```
[securitygroup]
enable_security_group = true
firewall_driver = neutron.agent.linux.iptables_firewall.IptablesFirewallDriver
```

### Cấu hình Layer-3 agent
L3 cung cấp định tuyến và NAT và các mạng ảo self-service. Trong file `/etc/neutron/l3_agent.ini`, tại section [DEFAULT]  
```
[DEFAULT]
interface_driver = linuxbridge
```

### Cấu hình DHCP agent
Trong file `/etc/neutron/dhcp_agent.ini`, tại section [DEFAULT]
```
[DEFAULT]
interface_driver = linuxbridge
dhcp_driver = neutron.agent.linux.dhcp.Dnsmasq
enable_isolated_metadata = true
```

### Cấu hình metadata agent
Edit file `/etc/neutron/metadata_agent.ini`, section [DEFAULT]
```
[DEFAULT]
nova_metadata_host = controller
metadata_proxy_shared_secret = METADATA_SECRET
```

## Hoàn tất cài đặt
Populate database trên Controller Node
```
su -s /bin/sh -c "neutron-db-manage --config-file /etc/neutron/neutron.conf --config-file /etc/neutron/plugins/ml2/ml2_conf.ini upgrade head" neutron
```
Restart nova-api và neutron-server trên Controller
```
service nova-api restart
service neutron-server restart
```
Restart Networking service trên Network Node
```
service neutron-server restart
service neutron-linuxbridge-agent restart
service neutron-dhcp-agent restart
service neutron-metadata-agent restart
service neutron-l3-agent restart
```

## Cài đặt và cấu hình Neutron trên Compute
Cài đặt các thành phần
```
apt install -y neutron-linuxbridge-agent
```
### Cấu hình thành phần cơ bản
Trong file `/etc/neutron/neutron.conf` cấu hình như sau:  
- [database] comment các `connection` vì compute không trực tiếp truy cập CSDL  

```
#connection = sqlite:////var/lib/neutron/neutron.sqlite
```
- [DEFAULT] cấu hình truy cập RABBIT MQ  

```
transport_url = rabbit://openstack:RABBIT_PASS@controller
```

- [DEFAULT] và [keystone_authtoken] section, cấu hình truy cập KeyStone  

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
username = neutron
password = 123456
```

### Cấu hình Linux Bridge agent
Trong file `/etc/neutron/plugins/ml2/linuxbridge_agent.ini` cấu hình như sau:  
- Trong [linux_bridge], map provider virtual network vào cổng vật lý của mạng provider (Lab này là ens33)

```
[linux_bridge]
physical_interface_mappings = provider:ens33
``` 
- Enable VXLAN, cấu hình IP của giao diện mạng sẽ xử lý các data overlay network (Lab này là IP management của Compute node)

```
[vxlan]
enable_vxlan = true
local_ip = 10.10.10.21
l2_population = true
```
- Enable security group và cấu hình driver Linux Bridge iptables firewall

```
[securitygroup]
enable_security_group = true
firewall_driver = neutron.agent.linux.iptables_firewall.IptablesFirewallDriver
```

### Cấu hình Nova sử dụng neutron
Trong file `/etc/nova/nova.conf`, section [neutron]  
```
[neutron]
url = http://controller:9696
auth_url = http://controller:35357
auth_type = password
project_domain_name = default
user_domain_name = default
region_name = RegionOne
project_name = service
username = neutron
password = 123456
```
Restart Compute service
```
service nova-compute restart
```
Restart Linux bridge agent
```
service neutron-linuxbridge-agent restart
```
