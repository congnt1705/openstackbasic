# Cài đặt dịch vụ dashboard (Horizon) trên Controller
Cài đặt Horizon
```
apt install -y openstack-dashboard
```
Trong file `/etc/openstack-dashboard/local_settings.py` sửa như sau:  
- Cấu hình dashboard sử dụng OpenStack service trên Controller Node  

```
OPENSTACK_HOST = "controller"
```
- Trong phần cấu hình Dashboard, chấp nhận tất cả các host truy cập dashboard  

```
ALLOWED_HOSTS = ['*', ]
```
- Cấu hình `memcached` session storage service

```
SESSION_ENGINE = 'django.contrib.sessions.backends.cache'

CACHES = {
    'default': {
         'BACKEND': 'django.core.cache.backends.memcached.MemcachedCache',
         'LOCATION': 'controller:11211',
    }
}
```
- Enable Identity API v3 

```
OPENSTACK_KEYSTONE_URL = "http://%s:5000/v3" % OPENSTACK_HOST
```
- Enable support for domains:

```
OPENSTACK_KEYSTONE_MULTIDOMAIN_SUPPORT = True
```
- Configure API versions:

```
OPENSTACK_API_VERSIONS = {
    "identity": 3,
    "image": 2,
    "volume": 2,
}
```
- Configure `Default` as the default domain for users that you create via the dashboard:

```
OPENSTACK_KEYSTONE_DEFAULT_DOMAIN = "Default"
```
- Configure user as the default role for users that you create via the dashboard:

```
OPENSTACK_KEYSTONE_DEFAULT_ROLE = "user"
```
- Optionally, configure the time zone:

```
TIME_ZONE = "Asia/Ho_Chi_Minh"
```

Restart Apache
```
service apache2 reload
```

## Verify
Sử dụng trình duyệt web truy cập vào url `http://controller/horizon` hoặc `http://10.10.10.11/horizon`.  
Đăng nhập bằng domain `default`, user `admin` với password là `ADMIN_PASS` hoặc user `demo` với password là `123456`
