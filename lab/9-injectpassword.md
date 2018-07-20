# Inject Password
Để truyền được password khi khởi tạo instance, image cần có gói cloud-init  
Tạo image ubuntu bằng virt-manager, cấu hình grub console để có thể console vào, cấu hình ssh cho root, update package...  
Cài gói cloud init bằng lệnh
```
sudo apt install -y cloud-init
```
Sau khi cài có thể vào file `/etc/cloud/cloud.cfg` để cấu hình user, lab này sửa default_user thành root chứ không phải ubuntu
```
default_user:
    name: root
    lock_passwd: True
    gecos: Root
    groups: [adm, audio, cdrom, dialout, dip, floppy, lxd, netdev, plugdev, sudo, video]
    sudo: ["ALL=(ALL) NOPASSWD:ALL"]
    shell: /bin/bash
```
Đưa image vào Controller Node, sau đó tạo image cho openstack
```
openstack image create "ubuntu" \
  --file ubuntu.qcow2 \
  --container-format bare \
  --public
```
Khi tạo instance, để truyền password vào ta cấu hình như sau:  
- TH1: Chỉ cấu hình default user  

```
#cloud-config
password: rootpasswd
chpasswd: { expire: False }
ssh_pwauth: True
```
- TH2: Cấu hình password của 1 list user  

```
#cloud-config
ssh_pwauth: True
chpasswd:
  list: |
    root:root1
    congnt:congnt1
    ubuntu:ubuntu1
  expire: False
```

# Reset password
## Cách 1
Đầu tiên shutdown instance (Hoặc có thể reboot instance sau khi thực hiện xong)  
Trên Compute Node chứa instance, mount disk của instance vào 1 thư mục nào đó
```
sudo mount /dev/sdb1 /media/test
```
Chuyển sang chạy lệnh trên thư mục đó
```
sudo chroot /media/test
```
Sau đó có thể sử dụng lệnh thay đổi pass như bình thường
```
root@compute:/# passwd root
Enter new UNIX password: 
Retype new UNIX password: 
passwd: password updated successfully

root@compute:/# passwd congnt
Enter new UNIX password: 
Retype new UNIX password: 
passwd: password updated successfully
```
Exit khỏi mode và vào lại server để kiểm tra, đồng thời umount thư mục.

## Cách 2
Để thay đổi password bằng openstack, instance cần có guest agent để cho phép giao tiếp giữa host và VM. Vì vậy khi build image cần cài thêm gói guest agent
```
sudo apt-get install qemu-guest-agent 
```
Sau khi đưa image vào controller, upload lên glance ta cần thêm 2 metadata
```
hw_qemu_guest_agent=yes
os_type=linux với image OS linux
[os_type=windows với image OS Windows]
```
Tạo instance với image bình thường, khi muốn đổi pass sử dụng lệnh
```
root@controller:~# openstack server set --root-password d9699743-82bd-49c8-8ab9-33653e7737e6
New password: 
Retype new password:
```
