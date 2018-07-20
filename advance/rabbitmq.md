# Introduce
RabbitMQ là 1 message broker, nó giúp nhận và gửi message. RabbitMQ dùng 1 số thuật ngữ:  
- Producing: chương trình gửi message  
- Queue: giống như hộp thư trong rabbitmq, message được lưu trữ trong queue trước khi được gửi đi. Queue bị ràng buộc bởi memory và disk.  
- Consuming: chương trình nhận message  
- channel: là 1 kết nối ảo nằm trong 1 kết nối. Khi gửi và nhận message từ queue, nó sẽ thực hiện qua channel.  
- Exchange: nhận tin nhắn từ Produce và đẩy tin nhắn tới queue. Có 4 loại exchange là direct, topic, headers và fanout.  
	- fanout: đưa message tới tất cả các queue nó biết  
	- direct: khi message đến có routing key match với binding key, message sẽ đi đến hàng đợi đó  
	- topic: message gửi đến exchange k được có routing key tùy ý, nó phải là 1 list các từ cách nhau bởi dấu ".". Logic của topic cũng giống exchange. Dấu `*` thay thế cho chính xác 1 từ, dấu `#` thay thế cho không hoặc nhiều từ
	- headers: Sử dụng các header để định tuyến các message đến queue  

- binding: là liên kết giữa exchange và queue.  
- routing key: thuộc tính của message, exchange sẽ nhìn vào routing key để định tyến message vài queue.  
- virtual host: tách các ứng dụng bằng cách sử dụng rabbitmq instance  

# Monitoring
Rabbitmq cung cấp plugin giúp giám sát, quản lý các hoạt động của rabbitmq. Rabbitmq-management cung cấp HTTP-based API để quản lý và giám sát server rabbitmq. Các chức năng của rabbitmq-managemant gồm:  
- List, delete các exchanges, queue, binding, user, virtual host & permission  
- Monitor queue length, message rate, data rate...  
- Monitor tài nguyên sử dụng  
- Quản lý user  
- Force close connection, purge queue  
- Gửi và nhận message  

## Getting Started
Enable rabbitmq-management
```
rabbitmq-plugins enable rabbitmq_management
```
Truy cập http://IP:15672/ để vào giao diện web  

## Permission
User được gán các tag trong RabbitMQ. Có 4 loại tag là management, policymaker, monitoring, administrator.  
- management:  
	- List virtual host mà user được login  
	- View all queue, exchange, binding trong virtual host  
	- View & close channel, connection  
	- View global statistics   

- policymaker: Các quyền của tag management cộng thêm:  
	- Xem, tạo, xóa các policy và thông số cho virtual host mà họ có thể đăng nhập  

- monitoring: Các quyền của tag monitoring cộng thêm:  
	- List all virtual host (kể cả những virtual host k được login)  
	- Xem các connection & channel của user khác  
	- View node-level data  
	- View global statistic cho mọi virtual host   

- administrator: Các quyền của tag policymaker và monitoring công thêm:  
	- Tạo và xóa virtual host  
	- Xem, tạo, xóa user  
	- Xem, tạo, xóa quyền  
	- Close connection của các user khác  

Để gán tag cho user sử dụng lệnh
```
rabbitmq set_user_tags [username] [tag]
```
Để xóa tag của user dùng lệnh
```
rabbitmq set_user_tags [username]
```
## HTTP API
Truy cập `http://IP:15672/api` để xem thông tin về API.  
API được sử dụng cho mục đích monitor và cảnh báo. Nó cung cấp truy cập vào thông tin chi tiết về trạng thái của các node, channel, connection...  
HTTP API chỉ được sử dụng trên các node enable management plugin. Nó có thể cung cấp các chỉ số của bất kỳ node nào. Khi monitor cluster node, chỉ cần contact tới 1 node bất kỳ hoặc contact tới load balancer để xem thông tin.  


## Install RabbitMQ Cluster
Install erlang
```
wget https://packages.erlang-solutions.com/erlang-solutions_1.0_all.deb
sudo dpkg -i erlang-solutions_1.0_all.deb
sudo apt-get update
sudo apt-get install erlang erlang-ssl
```
Pin erlang version in file `/etc/apt/preferences`
```
Package: *
Pin: release o=Erlang Solutions Ltd.
Pin-Priority: 999
```
Add repo rabbitmq
```
#Ubuntu 16.04
echo "deb https://dl.bintray.com/rabbitmq/debian xenial main" | sudo tee /etc/apt/sources.list.d/bintray.rabbitmq.list
```
Add key 
```
wget -O- https://dl.bintray.com/rabbitmq/Keys/rabbitmq-release-signing-key.asc |
     sudo apt-key add -
```
Install rabbitmq-server
```
sudo apt-get update
sudo apt-get install rabbitmq-server
```
Make sure service rabbitmq-server running, after that stop service
```
service rabbitmq-server status
service rabbitmq-server stop
```
Edit file `/etc/hosts`
```
10.5.8.2    rabbitmq-1
10.5.8.3    rabbitmq-2
```
Get erlang cookie from master node
```
(master)root@rabbitmq-1:~# cat /var/lib/rabbitmq/.erlang.cookie
```
Replace erlang cookie on all slave node with erlang cookie of master
```
(slave)root@rabbitmq-2:~# sudo sh -c "echo 'COOKIE_FROM_MASTER' > /var/lib/rabbitmq/.erlang.cookie"
```
Start service rabbitmq-server on slave node
```
service rabbitmq-server start
```
Run command on slave node
```
(slave)root@rabbitmq-2:~# rabbitmqctl stop_app
# If use disk to store message
(slave)root@rabbitmq-2:~# rabbitmqctl join_cluster rabbit@rabbitmq-1
# If use ram to store message
(slave)root@rabbitmq-2:~# rabbitmqctl join_cluster --ram rabbit@rabbitmq-1
(slave)root@rabbitmq-2:~# rabbitmqctl start_app
```
Check the cluster status from all node
```
rabbitmqctl cluster_status
```
Set policy to ha all queues (replicate 3)
```
rabbitmqctl set_policy ha-queue "" '{"ha-mode":"exactly","ha-params":3,"ha-sync-mode":"automatic"}' --apply-to queues
```
Enable plugin rabbitmq management & create user
```
rabbitmq-plugins enable rabbitmq_management
# Add user
rabbitmqctl add_user your_user your_password
# Set tag for user: administrator, monitoring, policymaker, management
rabbitmqctl set_user_tags your_user user_tag
```
Rabbitmq has memory watermark default is 0.4 (2GB-->800MB). If your rabbitmq server use over 0.4, rabbitmq will stop receive message. You can increase memory high watermark
```
# Not persistent
rabbitmqctl -n rabbit@rabbitmq-1  set_vm_memory_high_watermark 0.7
# Persistent. Add following line in `/etc/rabbitmq/rabbitmq.conf`
vm_memory_high_watermark.relative = 0.7
```
Restart rabbitmq-server
```
service rabbitmq-server restart
```

## SSL Rabbitmq
```
mkdir -p /etc/rabbitmq/tls-rabbitmq/tls-rabbitmq/
cd /etc/rabbitmq/tls-rabbitmq/tls-rabbitmq/
mkdir certs private
chmod 700 private
echo 01 > serial
touch index.txt
```
Create file `openssl.cnf` in `/etc/rabbitmq/tls-rabbitmq/tls-rabbitmq/`
```
[ ca ]
default_ca = tls-rabbitmq

[ tls-rabbitmq ]
dir = .
certificate = $dir/cacert.pem
database = $dir/index.txt
new_certs_dir = $dir/certs
private_key = $dir/private/cakey.pem
serial = $dir/serial

default_crl_days = 7
default_days = 365
default_md = sha256

policy = tls-rabbitmq_policy
x509_extensions = certificate_extensions

[ tls-rabbitmq_policy ]
commonName = supplied
stateOrProvinceName = optional
countryName = optional
emailAddress = optional
organizationName = optional
organizationalUnitName = optional
domainComponent = optional

[ certificate_extensions ]
basicConstraints = CA:false

[ req ]
default_bits = 2048
default_keyfile = ./private/cakey.pem
default_md = sha256
prompt = yes
distinguished_name = root_ca_distinguished_name
x509_extensions = root_ca_extensions

[ root_ca_distinguished_name ]
commonName = rabbitmq-ca

[ root_ca_extensions ]
basicConstraints = CA:true
keyUsage = keyCertSign, cRLSign

[ client_ca_extensions ]
basicConstraints = CA:false
keyUsage = digitalSignature,keyEncipherment
extendedKeyUsage = 1.3.6.1.5.5.7.3.2

[ server_ca_extensions ]
basicConstraints = CA:false
keyUsage = digitalSignature,keyEncipherment
extendedKeyUsage = 1.3.6.1.5.5.7.3.1
```
Gen key & certificates
```
openssl req -x509 -config /etc/rabbitmq/tls-rabbitmq/tls-rabbitmq/openssl.cnf -newkey rsa:4096 -days 365 \
    -out /etc/rabbitmq/tls-rabbitmq/tls-rabbitmq/cacert.pem -outform PEM -subj /CN=rabbitmq-ca/ -nodes
openssl x509 -in cacert.pem -out cacert.cer -outform DER
```
Create server certificates
```
cd /etc/rabbitmq/tls-rabbitmq/
mkdir server
cd server
openssl genrsa -out key.pem 4096
openssl req -new -key key.pem -out req.pem -outform PEM \
    -subj /CN=rabbitmq/O=server/ -nodes
cd ../tls-rabbitmq/
openssl ca -config openssl.cnf -in ../server/req.pem -out \
    ../server/cert.pem -notext -batch -extensions server_ca_extensions
```
Create client certificates
```
cd /etc/rabbitmq/tls-rabbitmq/
mkdir client
cd client
openssl genrsa -out key.pem 4096
openssl req -new -key key.pem -out req.pem -outform PEM \
    -subj /CN=rabbitmq/O=client/ -nodes
cd ../tls-rabbitmq
openssl ca -config openssl.cnf -in ../client/req.pem -out \
    ../client/cert.pem -notext -batch -extensions client_ca_extensions
```
Enable TLS in `/etc/rabbitmq/rabbitmq.conf`
```
listeners.tcp.local = 127.0.0.1:5672
listeners.ssl.default = 5671

ssl_options.cacertfile = /etc/rabbitmq/tls-rabbitmq/tls-rabbitmq/cacert.pem
ssl_options.certfile = /etc/rabbitmq/tls-rabbitmq/server/cert.pem
ssl_options.keyfile = /etc/rabbitmq/tls-rabbitmq/server/key.pem
ssl_options.verify = verify_peer
ssl_options.fail_if_no_peer_cert = true
```
Restart rabbitmq-server
```
service rabbitmq-server restart
```
