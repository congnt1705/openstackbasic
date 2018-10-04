# Cài đặt dịch vụ Networking (Neutron)
## Cài đặt Neutron trên Controller
Đăng nhập vào MySQL với password root đã tạo
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
Trong file `/etc/neutron/neutron.conf` cấu hình như sau (bind_host thay bằng IP của controller đang cấu hình):
```
[DEFAULT]
core_plugin = ml2
service_plugins = router
allow_overlapping_ips = true
transport_url = rabbit://openstack:RABBIT_PASS@controller1,openstack:RABBIT_PASS@controller2,openstack:RABBIT_PASS@controller3,
auth_strategy = keystone
notify_nova_on_port_status_changes = true
notify_nova_on_port_data_changes = true
bind_host = [IP Management Controller]
l3_ha = True
allow_automatic_l3agent_failover = True
max_l3_agents_per_router = 2
min_l3_agents_per_router = 2
dhcp_agents_per_network = 2
max_allowed_address_pair = 25
[agent]
root_helper = "sudo /usr/bin/neutron-rootwrap /etc/neutron/rootwrap.conf"
[database]
connection = mysql+pymysql://neutron:NEUTRON_DBPASS@controller/neutron
[keystone_authtoken]
auth_uri = http://controller:5000
auth_url = http://controller:5000
memcached_servers = controller:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = neutron
password = 123456
[nova]
auth_url = http://controller:5000
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
```
[DEFAULT]
[ml2]
type_drivers = flat,vlan,vxlan
tenant_network_types = vxlan
mechanism_drivers = openvswitch,l2population
extension_drivers = port_security
[ml2_type_flat]
flat_networks = provider
[ml2_type_gre]
[ml2_type_vlan]
[ml2_type_vxlan]
vni_ranges = 1:1000
[securitygroup]
enable_ipset = true
```
### Cấu hình Compute Service sử dụng Networking trên Controller Node
Edit file `/etc/nova/nova.conf`, section [neutron] 
```
[neutron]
url = http://controller:9696
auth_url = http://controller:5000
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
apt install -y neutron-server neutron-plugin-ml2 neutron-plugin-openvswitch-agent neutron-l3-agent neutron-dhcp-agent neutron-metadata-agent
```
### Cấu hình Neutron trên Network Node
Trong file `/etc/neutron/neutron.conf` cấu hình như sau:  
```
[DEFAULT]
core_plugin = ml2
service_plugins = router
allow_overlapping_ips = true
transport_url = rabbit://openstack:RABBIT_PASS@controller1,openstack:RABBIT_PASS@controller2,openstack:RABBIT_PASS@controller3,
auth_strategy = keystone
notify_nova_on_port_status_changes = true
notify_nova_on_port_data_changes = true
bind_host = [IP Management Network Node]
[agent]
root_helper = "sudo /usr/bin/neutron-rootwrap /etc/neutron/rootwrap.conf"
[database]
connection = mysql+pymysql://neutron:NEUTRON_DBPASS@controller/neutron
[keystone_authtoken]
auth_uri = http://controller:5000
auth_url = http://controller:5000
memcached_servers = controller:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = neutron
password = 123456
[nova]
auth_url = http://controller:5000
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
```
[DEFAULT]
[ml2]
type_drivers = flat,vlan,vxlan
tenant_network_types = vxlan
mechanism_drivers = openvswitch,l2population
extension_drivers = port_security
[ml2_type_flat]
flat_networks = provider
[ml2_type_gre]
[ml2_type_vlan]
[ml2_type_vxlan]
vni_ranges = 1:1000
[securitygroup]
enable_ipset = true
```

### Cấu hình OpenvSwitch
Trong file `/etc/neutron/l3_agent.ini`
```
[DEFAULT]
interface_driver = openvswitch
```
Trong file `/etc/neutron/dhcp_agent.ini`
```
[DEFAULT]
interface_driver = openvswitch
dhcp_driver = neutron.agent.linux.dhcp.Dnsmasq
enable_isolated_metadata = true
```
Tạo OVS provider bridge. Thay eth0 bằng card mạng provider
```
ovs-vsctl add-br br-provider
ovs-vsctl add-port br-provider eth0
```
Trong file `/etc/network/interfaces`
```
auto eth0
iface eth0 inet manual
up ifconfig eth0 0.0.0.0 up
up ip link set eth0 promisc on
down ip link set eth0 promisc off
down ifconfig eth0 down

auto br-provider
iface br-provider inet static
address [IP provider network node]
netmask 255.255.252.0
gateway 10.5.8.1
dns-nameservers 8.8.8.8
```
Trong file `/etc/neutron/plugins/ml2/openvswitch_agent.ini`
```
[DEFAULT]
[agent]
tunnel_types = vxlan
l2_population = True
[ovs]
bridge_mappings = provider:br-provider
local_ip = [IP management Network node]
[securitygroup]
firewall_driver = iptables_hybrid
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
service nova-api status
service neutron-server status
```
Restart Networking service trên Network Node
```
service neutron-server restart
service neutron-openvswitch-agent restart
service neutron-dhcp-agent restart
service neutron-metadata-agent restart
service neutron-l3-agent restart
service neutron-server status
service neutron-openvswitch-agent status
service neutron-dhcp-agent status
service neutron-metadata-agent status
service neutron-l3-agent status
```

## Cài đặt và cấu hình Neutron trên Compute
Cài đặt các thành phần
```
apt install -y neutron-plugin-openvswitch-agent
```
### Cấu hình thành phần cơ bản
Trong file `/etc/neutron/neutron.conf` cấu hình như sau:  
```
[DEFAULT]
core_plugin = ml2
auth_strategy = keystone
transport_url = rabbit://openstack:RABBIT_PASS@controller1,openstack:RABBIT_PASS@controller2,openstack:RABBIT_PASS@controller3,
[agent]
root_helper = "sudo /usr/bin/neutron-rootwrap /etc/neutron/rootwrap.conf"
[keystone_authtoken]
auth_uri = http://controller:5000
auth_url = http://controller:5000
memcached_servers = controller:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = neutron
password = 123456
```
### Cấu hình OpenvSwitch agent
Trong file `/etc/neutron/plugins/ml2/openvswitch_agent.ini`
```
[DEFAULT]
[ovs]
local_ip = [IP Manament Compute Node]
[agent]
tunnel_types = vxlan
l2_population = True
[securitygroup]
firewall_driver = iptables_hybrid
```

### Cấu hình Nova sử dụng neutron
Trong file `/etc/nova/nova.conf`, thêm section [neutron] như sau:  
```
[neutron]
url = http://controller:9696
auth_url = http://controller:5000
auth_type = password
project_domain_name = default
user_domain_name = default
region_name = RegionOne
project_name = service
username = neutron
password = 123456
```
Restart nova compute & neutron-openvswitch
```
service nova-compute restart
service neutron-openvswitch-agent restart
```
