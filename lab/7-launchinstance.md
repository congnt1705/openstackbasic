# Launch instance
## Tạo network
### Tạo provider network
Chạy script biến môi trường của user admin 
```
. admin-openrc
```
Tạo network  
```
openstack network create --share --external --provider-physical-network provider --provider-network-type flat provider
+---------------------------+--------------------------------------+
| Field                     | Value                                |
+---------------------------+--------------------------------------+
| admin_state_up            | UP                                   |
| availability_zone_hints   |                                      |
| availability_zones        |                                      |
| created_at                | 2018-01-03T10:49:18Z                 |
| description               |                                      |
| dns_domain                | None                                 |
| id                        | fc79d828-abca-4f33-b030-14a809966d17 |
| ipv4_address_scope        | None                                 |
| ipv6_address_scope        | None                                 |
| is_default                | False                                |
| is_vlan_transparent       | None                                 |
| mtu                       | 1500                                 |
| name                      | provider                             |
| port_security_enabled     | True                                 |
| project_id                | 6e09da450c9e4d4ebdd71c74499e86be     |
| provider:network_type     | flat                                 |
| provider:physical_network | provider                             |
| provider:segmentation_id  | None                                 |
| qos_policy_id             | None                                 |
| revision_number           | 3                                    |
| router:external           | External                             |
| segments                  | None                                 |
| shared                    | True                                 |
| status                    | ACTIVE                               |
| subnets                   |                                      |
| tags                      |                                      |
| updated_at                | 2018-01-03T10:49:18Z                 |
+---------------------------+--------------------------------------+
```
Tạo subnet cho network provider
```
openstack subnet create --network provider \
--allocation-pool start=192.168.66.224,end=192.168.66.254 \
--dns-nameserver 8.8.8.8 --gateway 192.168.66.2 \
--subnet-range 192.168.66.0/24 provider
+-------------------------+--------------------------------------+
| Field                   | Value                                |
+-------------------------+--------------------------------------+
| allocation_pools        | 192.168.66.224-192.168.66.254        |
| cidr                    | 192.168.66.0/24                      |
| created_at              | 2018-01-03T10:54:29Z                 |
| description             |                                      |
| dns_nameservers         | 8.8.8.8                              |
| enable_dhcp             | True                                 |
| gateway_ip              | 192.168.66.2                         |
| host_routes             |                                      |
| id                      | 93f904c4-acaa-4333-be15-dcb587031379 |
| ip_version              | 4                                    |
| ipv6_address_mode       | None                                 |
| ipv6_ra_mode            | None                                 |
| name                    | provider                             |
| network_id              | fc79d828-abca-4f33-b030-14a809966d17 |
| project_id              | 6e09da450c9e4d4ebdd71c74499e86be     |
| revision_number         | 0                                    |
| segment_id              | None                                 |
| service_types           |                                      |
| subnetpool_id           | None                                 |
| tags                    |                                      |
| updated_at              | 2018-01-03T10:54:29Z                 |
| use_default_subnet_pool | None                                 |
+-------------------------+--------------------------------------+
```


### Tạo virtual network
Chạy script biến môi trường của user demo 
```
. demo-openrc
```
Tạo network
```
openstack network create selfservice
+---------------------------+--------------------------------------+
| Field                     | Value                                |
+---------------------------+--------------------------------------+
| admin_state_up            | UP                                   |
| availability_zone_hints   |                                      |
| availability_zones        |                                      |
| created_at                | 2018-01-03T10:16:47Z                 |
| description               |                                      |
| dns_domain                | None                                 |
| id                        | e994cc6e-41c8-48e2-b6c1-b5d0b2e046d8 |
| ipv4_address_scope        | None                                 |
| ipv6_address_scope        | None                                 |
| is_default                | False                                |
| is_vlan_transparent       | None                                 |
| mtu                       | 1450                                 |
| name                      | selfservice                          |
| port_security_enabled     | True                                 |
| project_id                | 3275c6545a6347658bd028569a1386ba     |
| provider:network_type     | None                                 |
| provider:physical_network | None                                 |
| provider:segmentation_id  | None                                 |
| qos_policy_id             | None                                 |
| revision_number           | 2                                    |
| router:external           | Internal                             |
| segments                  | None                                 |
| shared                    | False                                |
| status                    | ACTIVE                               |
| subnets                   |                                      |
| tags                      |                                      |
| updated_at                | 2018-01-03T10:16:47Z                 |
+---------------------------+--------------------------------------+
```
Tạo subnet trên network `selfservice` vừa tạo
```
openstack subnet create --network selfservice --dns-nameserver 8.8.8.8 \
--gateway 172.16.1.1 --subnet-range 172.16.1.0/24 subnetofselfservice
+-------------------------+--------------------------------------+
| Field                   | Value                                |
+-------------------------+--------------------------------------+
| allocation_pools        | 172.16.1.2-172.16.1.254              |
| cidr                    | 172.16.1.0/24                        |
| created_at              | 2018-01-03T10:33:27Z                 |
| description             |                                      |
| dns_nameservers         | 8.8.8.8                              |
| enable_dhcp             | True                                 |
| gateway_ip              | 172.16.1.1                           |
| host_routes             |                                      |
| id                      | a66260ff-09fb-44df-b17c-cf5b704eb579 |
| ip_version              | 4                                    |
| ipv6_address_mode       | None                                 |
| ipv6_ra_mode            | None                                 |
| name                    | subnetofselfservice                  |
| network_id              | e994cc6e-41c8-48e2-b6c1-b5d0b2e046d8 |
| project_id              | 3275c6545a6347658bd028569a1386ba     |
| revision_number         | 0                                    |
| segment_id              | None                                 |
| service_types           |                                      |
| subnetpool_id           | None                                 |
| tags                    |                                      |
| updated_at              | 2018-01-03T10:33:27Z                 |
| use_default_subnet_pool | None                                 |
+-------------------------+--------------------------------------+
```

### Tạo Router
Chạy script biến môi trường của user demo 
```
. demo-openrc
```
Tạo Router
```
openstack router create routertest
+-------------------------+--------------------------------------+
| Field                   | Value                                |
+-------------------------+--------------------------------------+
| admin_state_up          | UP                                   |
| availability_zone_hints |                                      |
| availability_zones      |                                      |
| created_at              | 2018-01-03T10:27:56Z                 |
| description             |                                      |
| distributed             | False                                |
| external_gateway_info   | None                                 |
| flavor_id               | None                                 |
| ha                      | False                                |
| id                      | b86738f5-abee-40a3-a507-24dce346df33 |
| name                    | routertest                           |
| project_id              | 3275c6545a6347658bd028569a1386ba     |
| revision_number         | None                                 |
| routes                  |                                      |
| status                  | ACTIVE                               |
| tags                    |                                      |
| updated_at              | 2018-01-03T10:27:56Z                 |
+-------------------------+--------------------------------------+
```
Add subnet selfservice vào interface của router
```
openstack router add subnet routertest subnetofselfservice
```
Set gateway của provider network vào router
```
neutron router-gateway-set routertest provider
```

### Verify
Trên Network node list network namespace
```
root@network:~# ip netns
qdhcp-fc79d828-abca-4f33-b030-14a809966d17 (id: 2)
qrouter-b86738f5-abee-40a3-a507-24dce346df33 (id: 1)
qdhcp-e994cc6e-41c8-48e2-b6c1-b5d0b2e046d8 (id: 0)
```
Chạy script biến môi trường user admin trên Controller
```
. admin-openrc
``` 
List port của router
```
neutron router-port-list routertest
+--------------------------------------+------+----------------------------------+-------------------+---------------------------------------------------------------------------------------+
| id                                   | name | tenant_id                        | mac_address       | fixed_ips                                                                             |
+--------------------------------------+------+----------------------------------+-------------------+---------------------------------------------------------------------------------------+
| 6a57239e-d823-4a24-8c2a-cedccaeef19d |      |                                  | fa:16:3e:7a:18:b2 | {"subnet_id": "93f904c4-acaa-4333-be15-dcb587031379", "ip_address": "192.168.66.231"} |
| ef6a5af4-1d93-4ca8-be63-11d27bb3537c |      | 3275c6545a6347658bd028569a1386ba | fa:16:3e:35:9a:af | {"subnet_id": "a66260ff-09fb-44df-b17c-cf5b704eb579", "ip_address": "172.16.1.1"}     |
+--------------------------------------+------+----------------------------------+-------------------+---------------------------------------------------------------------------------------+
```

## Tạo flavor
```
openstack flavor create --id 0 --vcpus 1 --ram 512 --disk 10 flavortest
+----------------------------+------------+
| Field                      | Value      |
+----------------------------+------------+
| OS-FLV-DISABLED:disabled   | False      |
| OS-FLV-EXT-DATA:ephemeral  | 0          |
| disk                       | 10         |
| id                         | 0          |
| name                       | flavortest |
| os-flavor-access:is_public | True       |
| properties                 |            |
| ram                        | 512        |
| rxtx_factor                | 1.0        |
| swap                       |            |
| vcpus                      | 1          |
+----------------------------+------------+
```

## Generate a key pair
Tạo keypair đưa vào Compute Node.  
Chạy script biến môi trường của user demo 
```
. demo-openrc
```  
Tạo keypair và add public key
```
ssh-keygen -q -N ""
openstack keypair create --public-key ~/.ssh/id_rsa.pub keytest
+-------------+-------------------------------------------------+
| Field       | Value                                           |
+-------------+-------------------------------------------------+
| fingerprint | 13:74:f3:bd:6f:34:d9:9a:37:35:1e:2a:88:ad:6f:40 |
| name        | keytest                                         |
| user_id     | b3082ea3decf453b9ee7e34384a8568b                |
+-------------+-------------------------------------------------+
```
Verify keypair 
```
openstack keypair list
+---------+-------------------------------------------------+
| Name    | Fingerprint                                     |
+---------+-------------------------------------------------+
| keytest | 13:74:f3:bd:6f:34:d9:9a:37:35:1e:2a:88:ad:6f:40 |
+---------+-------------------------------------------------+
```

## Add security group rules
Cho phép ICMP 
```
openstack security group rule create --proto icmp default
```
Cho phép ssh
```
openstack security group rule create --proto tcp --dst-port 22 default
```

## Launch an instance
Chạy script biến môi trường của user demo
```
. demo-openrc
```
List các flavor, image, network, security group va keypair được dùng
```
openstack flavor list
openstack image list
openstack network list
openstack security group list
openstack keypair list
```
Launch an instance
```
openstack server create --flavor m1.nano --image cirros --key-name keytest \
--nic net-id=e994cc6e-41c8-48e2-b6c1-b5d0b2e046d8 --security-group default instance1
```
Xem các server hiện có
```
openstack server list
```
lấy url console vào server
```
openstack console url show instance1
```

