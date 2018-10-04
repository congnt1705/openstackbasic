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
```
Tạo subnet cho network provider
```
openstack subnet create --network provider \
--allocation-pool start=10.5.10.70,end=10.5.10.75 \
--dns-nameserver 8.8.8.8 --gateway 10.5.8.1 \
--subnet-range 10.5.8.0/22 provider
```

### Tạo virtual network
Chạy script biến môi trường của user demo 
```
. demo-openrc
```
Tạo network
```
openstack network create selfservice
```
Tạo subnet trên network `selfservice` vừa tạo
```
openstack subnet create --network selfservice --dns-nameserver 8.8.8.8 \
--gateway 172.16.1.1 --subnet-range 172.16.1.0/24 subnetofselfservice
```

### Tạo Router
Chạy script biến môi trường của user demo 
```
. demo-openrc
```
Tạo Router
```
openstack router create routertest
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
Trên Network node list network namespace. **Có thể khi list trên 1 node sẽ không ra đủ như dưới, bạn phải kiểm tra trên cả 3 node**
```
root@network1:~# ip netns
qrouter-7bf6be7f-b592-45f2-b1dd-0c1489357ee0
qdhcp-1f4b04c2-b677-48e1-ac00-e00ad67d6ef9
qdhcp-b234f4b1-93c4-41e7-8979-5ab8cbc2925e
```
Chạy script biến môi trường user admin trên Controller
```
. admin-openrc
``` 
List port của router
```
neutron router-port-list routertest
```

## Tạo flavor
```
openstack flavor create --vcpus 1 --ram 1024 1c_1g
```

## Add security group rules
Chạy script biến môi trường của user demo
```
. demo-openrc
```
Cho phép ICMP 
```
openstack security group rule create --proto icmp default
```
Cho phép ssh
```
openstack security group rule create --proto tcp --dst-port 22 default
```
## Tạo Volume với image cirros
Chạy script biến môi trường của user demo
```
. demo-openrc
```
Tạo volume với image cirross
```
openstack volume create --size 1 --image 35487019-b456-4b99-a285-0cc962bc406d volume_test
```
## Launch an instance
Chạy script biến môi trường của user demo
```
. demo-openrc
```
List các flavor, image, network, security group va keypair được dùng
```
openstack flavor list
openstack network list
openstack volume list
openstack security group list
```
Launch an instance
```
openstack server create --volume bbaa504f-7fed-4ef4-a05b-aa30c480dbbb --flavor 42042581-b3e7-48fa-b814-08dcd72a0e78\
 --nic net-id=1f4b04c2-b677-48e1-ac00-e00ad67d6ef9 --security-group default instance1
```
Xem các server hiện có
```
openstack server list
```
lấy url console vào server
```
openstack console url show instance1
```

