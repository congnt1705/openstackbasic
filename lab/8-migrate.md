# Add Compute Node
Cấu hình compute node giống như trong link [này](3-nova.md).  
Trên **Controller** sử dụng lệnh `nova-manage cell_v2 discover_hosts` để đăng ký Compute Node mới.  
Cấu hình Neutron giống như trong link [này](4-neutron.md) nếu sử dụng Linux Bridge hoặc link [này](4-neutron-2.md) nếu sử dụng OVS.  

# Migrate
**Cấu hình trẻn Compute Node**
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
nova@compute:/home/congnt$ ssh-keygen -t rsa -b 2048
Generating public/private rsa key pair.
Enter file in which to save the key (/var/lib/nova/.ssh/id_rsa): 
Enter passphrase (empty for no passphrase): 
Enter same passphrase again: 
Your identification has been saved in /var/lib/nova/.ssh/id_rsa.
Your public key has been saved in /var/lib/nova/.ssh/id_rsa.pub.
```
Copy key sang Compute2
```
nova@compute:~/.ssh$ ssh-copy-id -i id_rsa.pub nova@10.10.10.41
/usr/bin/ssh-copy-id: INFO: Source of key(s) to be installed: "id_rsa.pub"
/usr/bin/ssh-copy-id: INFO: attempting to log in with the new key(s), to filter out any that are already installed
/usr/bin/ssh-copy-id: INFO: 1 key(s) remain to be installed -- if you are prompted now it is to install the new keys
nova@10.10.10.41's password: 

Number of key(s) added: 1

Now try logging into the machine, with:   "ssh 'nova@10.10.10.41'"
and check to make sure that only the key(s) you wanted were added.
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
**Làm tương tự với Compute2**

