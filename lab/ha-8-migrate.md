# Migrate
**Thực hiện trên 1 compute node bất kỳ**
Enable login nova user
```
usermod -s /bin/bash nova
```
Chuyển qua user nova
```
su nova
```
Gen key cho compute node 
```
nova@compute1:/home/congnt$ ssh-keygen -t rsa -b 2048
Generating public/private rsa key pair.
Enter file in which to save the key (/var/lib/nova/.ssh/id_rsa): 
Enter passphrase (empty for no passphrase): 
Enter same passphrase again: 
Your identification has been saved in /var/lib/nova/.ssh/id_rsa.
Your public key has been saved in /var/lib/nova/.ssh/id_rsa.pub.
```
Tạo file authorized_keys
```
cp /var/lib/nova/.ssh/id_rsa.pub /var/lib/nova/.ssh/authorized_keys
```
Change quyền thư mục
```
chown -R nova. /var/lib/nova/.ssh/
```
Copy cả thư mục `/var/lib/nova/.ssh/` sang Compute2, compute3
```
scp -rp /var/lib/nova/.ssh/ compute2:/var/lib/nova/.ssh/
scp -rp /var/lib/nova/.ssh/ compute3:/var/lib/nova/.ssh/
```
Change quyền
```
chown -R nova. /var/lib/nova/.ssh/
```
Trong file `/etc/libvirt/libvirtd.conf`, tắt listen tls và bật listen tcp
```
listen_tls = 0
listen_tcp = 1
auth_tcp = "none"
```
Trong file `/etc/default/libvirtd` cấu hình listen tcp
```
libvirtd_opts=" -l"
```
Restart libvirt
```
service libvirtd restart
```
Show VM
```
root@controller1:~# openstack server show 79bb21fd-8147-457f-922a-c21708b46799
+-------------------------------------+----------------------------------------------------------+
| Field                               | Value                                                    |
+-------------------------------------+----------------------------------------------------------+
| OS-DCF:diskConfig                   | MANUAL                                                   |
| OS-EXT-AZ:availability_zone         | nova                                                     |
| OS-EXT-SRV-ATTR:host                | compute2                                                 |
| OS-EXT-SRV-ATTR:hypervisor_hostname | compute2                                                 |
| OS-EXT-SRV-ATTR:instance_name       | instance-00000048                                        |
| OS-EXT-STS:power_state              | Running                                                  |
| OS-EXT-STS:task_state               | None                                                     |
| OS-EXT-STS:vm_state                 | active                                                   |
| OS-SRV-USG:launched_at              | 2018-10-04T09:20:27.000000                               |
| OS-SRV-USG:terminated_at            | None                                                     |
| accessIPv4                          |                                                          |
| accessIPv6                          |                                                          |
| addresses                           | selfservice=172.16.1.10                                  |
| config_drive                        |                                                          |
| created                             | 2018-10-04T09:20:14Z                                     |
| flavor                              | 1c_1g (42042581-b3e7-48fa-b814-08dcd72a0e78)             |
| hostId                              | 31f88791f8e6510fcb2f94bf6a1834d7de76a99466a98898676ad750 |
| id                                  | 79bb21fd-8147-457f-922a-c21708b46799                     |
| image                               |                                                          |
| key_name                            | None                                                     |
| name                                | instance1                                                |
| progress                            | 0                                                        |
| project_id                          | a5b1924d0f9743dabceea2627378bd91                         |
| properties                          |                                                          |
| security_groups                     | name='default'                                           |
| status                              | ACTIVE                                                   |
| updated                             | 2018-10-04T10:37:16Z                                     |
| user_id                             | 00ada8da9f374ed7b0fa0655116b3b99                         |
| volumes_attached                    | id='bbaa504f-7fed-4ef4-a05b-aa30c480dbbb'                |
+-------------------------------------+----------------------------------------------------------+
```
Live Migate sang compute3
```
openstack server migrate --live compute3 79bb21fd-8147-457f-922a-c21708b46799
```
Show lại VM thấy VM đang ở compute3
```
root@controller1:~# openstack server show 79bb21fd-8147-457f-922a-c21708b46799
+-------------------------------------+----------------------------------------------------------+
| Field                               | Value                                                    |
+-------------------------------------+----------------------------------------------------------+
| OS-DCF:diskConfig                   | MANUAL                                                   |
| OS-EXT-AZ:availability_zone         | nova                                                     |
| OS-EXT-SRV-ATTR:host                | compute3                                                 |
| OS-EXT-SRV-ATTR:hypervisor_hostname | compute3                                                 |
| OS-EXT-SRV-ATTR:instance_name       | instance-00000048                                        |
| OS-EXT-STS:power_state              | Running                                                  |
| OS-EXT-STS:task_state               | None                                                     |
| OS-EXT-STS:vm_state                 | active                                                   |
| OS-SRV-USG:launched_at              | 2018-10-04T09:20:27.000000                               |
| OS-SRV-USG:terminated_at            | None                                                     |
| accessIPv4                          |                                                          |
| accessIPv6                          |                                                          |
| addresses                           | selfservice=172.16.1.10                                  |
| config_drive                        |                                                          |
| created                             | 2018-10-04T09:20:14Z                                     |
| flavor                              | 1c_1g (42042581-b3e7-48fa-b814-08dcd72a0e78)             |
| hostId                              | 3e5a0dc5a725d4db4a6eedaa95d5e86340ed7a83f7ee7e4d1f7e319b |
| id                                  | 79bb21fd-8147-457f-922a-c21708b46799                     |
| image                               |                                                          |
| key_name                            | None                                                     |
| name                                | instance1                                                |
| progress                            | 0                                                        |
| project_id                          | a5b1924d0f9743dabceea2627378bd91                         |
| properties                          |                                                          |
| security_groups                     | name='default'                                           |
| status                              | ACTIVE                                                   |
| updated                             | 2018-10-04T10:43:57Z                                     |
| user_id                             | 00ada8da9f374ed7b0fa0655116b3b99                         |
| volumes_attached                    | id='bbaa504f-7fed-4ef4-a05b-aa30c480dbbb'                |
+-------------------------------------+----------------------------------------------------------+
```
Cold Migrate
```
openstack server migrate 79bb21fd-8147-457f-922a-c21708b46799
```
Khi show VM thấy ở trạng thái VERIFY_RESIZE ta sử dụng lệnh sau để confirm
```
openstack server resize --confirm 79bb21fd-8147-457f-922a-c21708b46799
```
Show lại VM thấy VM ở compute 1
```
root@controller1:~# openstack server show 79bb21fd-8147-457f-922a-c21708b46799
+-------------------------------------+----------------------------------------------------------+
| Field                               | Value                                                    |
+-------------------------------------+----------------------------------------------------------+
| OS-DCF:diskConfig                   | MANUAL                                                   |
| OS-EXT-AZ:availability_zone         | nova                                                     |
| OS-EXT-SRV-ATTR:host                | compute1                                                 |
| OS-EXT-SRV-ATTR:hypervisor_hostname | compute1                                                 |
| OS-EXT-SRV-ATTR:instance_name       | instance-00000048                                        |
| OS-EXT-STS:power_state              | Running                                                  |
| OS-EXT-STS:task_state               | None                                                     |
| OS-EXT-STS:vm_state                 | active                                                   |
| OS-SRV-USG:launched_at              | 2018-10-04T10:45:17.000000                               |
| OS-SRV-USG:terminated_at            | None                                                     |
| accessIPv4                          |                                                          |
| accessIPv6                          |                                                          |
| addresses                           | selfservice=172.16.1.10                                  |
| config_drive                        |                                                          |
| created                             | 2018-10-04T09:20:14Z                                     |
| flavor                              | 1c_1g (42042581-b3e7-48fa-b814-08dcd72a0e78)             |
| hostId                              | d202562755beb1c31029fa1c03e6864c9938d800d038994abc634e20 |
| id                                  | 79bb21fd-8147-457f-922a-c21708b46799                     |
| image                               |                                                          |
| key_name                            | None                                                     |
| name                                | instance1                                                |
| progress                            | 0                                                        |
| project_id                          | a5b1924d0f9743dabceea2627378bd91                         |
| properties                          |                                                          |
| security_groups                     | name='default'                                           |
| status                              | ACTIVE                                                   |
| updated                             | 2018-10-04T10:46:43Z                                     |
| user_id                             | 00ada8da9f374ed7b0fa0655116b3b99                         |
| volumes_attached                    | id='bbaa504f-7fed-4ef4-a05b-aa30c480dbbb'                |
+-------------------------------------+----------------------------------------------------------+
```
