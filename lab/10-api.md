# API
Lấy token bằng user admin, tạo image, flavor, provider network, list domain, tạo domain CS-Lab-Domain, project CS-Lab-Project. Sau đó sử dụng token để tạo 1 user CS-Lab-User nằm trong Domain CS-Lab-Domain, project CS-Lab-Project. Lấy token bằng user CS-Lab-User, tạo network, subnet, router và server.  
## Lấy token bằng user admin
```
curl -i \
  -H "Content-Type: application/json" \
  -d '
{ "auth": {
    "identity": {
      "methods": ["password"],
      "password": {
        "user": {
          "name": "admin",
          "domain": { "id": "default" },
          "password": "ADMIN_PASS"
        }
      }
    },
    "scope": {
      "project": {
        "name": "admin",
        "domain": { "id": "default" }
      }
    }
  }
}' \
  "http://controller:5000/v3/auth/tokens" ; echo
```
## Tạo image
```
curl -i -X POST \
  -H "X-Auth-Token: $OS-Token" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "cirros1",
    "container_format": "bare",
    "disk_format": "qcow2",
    "location": "/home/congnt/cirros-0.3.5-x86_64-disk.img",
    "visibility": "public"
  }' \
   "http://controller:9292/v2/images" ; echo
```
## Tạo flavor 
```
curl -i -X POST \
  -H "X-Auth-Token: gAAAAABabpWd4CDiwW118gopirUQzbcMM0OtG21z70rB9t-giU9m45cNl6IoAYHa4GTQuaSRrkskVGWiSGMTRwUBNCxgCzYXkmY4CLwNJLriOF7lR2acGUQhD0MOhgAS3r-SZsTYnood4yJMyi4MUwn-rFzeF4xjWuWcJ3I-zRnPSOz_LIUutso" \
  -H "Content-Type: application/json" \
  -d '{
    "flavor": {
        "name": "test_flavor",
        "ram": 128,
        "vcpus": 1,
        "disk": 0,
        "id": "10"
       }
   }' \
   "http://controller:8774/v2.1/flavors" ; echo
```
## Tạo Provider network
```
curl -i -X POST \
  -H "X-Auth-Token: gAAAAABabokYwagrdZn7iswqlfWnLuuEfux5G-VHweIj2hkSTwpDy7TZWQyNAWGTJMGCb_rN16gGO8hzvCj67O32HiQpHy8D6dqo4eKXxcACdiac0NUze1Ze8ky2U3-kiIp_pab5mNaRK5_IhFwo_KGhGBrDRuGwIbq0wNchPUPr9SVATLdNJvY" \
  -H "Content-Type: application/json" \
  -d '{
    "network": {
        "admin_state_up": true,
        "router:external": true,
        "name": "provider",
        "shared": true,
        "provider:network_type": "flat",
        "provider:physical_network": "provider"
      }
  }' \
   "http://controller:9696/v2.0/networks" ; echo
```
## Tạo subnet cho provider
```
curl -i -X POST \
  -H "X-Auth-Token: gAAAAABabokYwagrdZn7iswqlfWnLuuEfux5G-VHweIj2hkSTwpDy7TZWQyNAWGTJMGCb_rN16gGO8hzvCj67O32HiQpHy8D6dqo4eKXxcACdiac0NUze1Ze8ky2U3-kiIp_pab5mNaRK5_IhFwo_KGhGBrDRuGwIbq0wNchPUPr9SVATLdNJvY" \
  -H "Content-Type: application/json" \
  -d '{
    "subnet": {
      "name": "provider",
      "network_id": "114d81e8-e903-41e7-9469-efa64682a3cf",
      "dns_nameservers": ["8.8.8.8"],
      "allocation_pools": [{
        "start": "192.168.66.224",
        "end": "192.168.66.254"
      }],
     "gateway_ip": "192.168.66.2",
     "ip_version": 4,
     "cidr": "192.168.66.0/24"
    }
  }' \
   "http://controller:9696/v2.0/subnets" ; echo
```
## List domain
```
curl -s \
  -H "X-Auth-Token: $OS-Token" \
  "http://controller:5000/v3/domains" | python -mjson.tool
```

## Tạo domain
```
curl -s \
  -H "X-Auth-Token: $OS-Token" \
  -H "Content-Type: application/json" \
  -d '{ "domain": { "name": "CS-Lab-Domain"}}' \
  "http://controller:5000/v3/domains" | python -mjson.tool
```

## List Project
```
curl -s \
 -H "X-Auth-Token: $OS-Token" \
 "http://controller:5000/v3/projects" | python -mjson.tool
```

## Tạo Project
```
curl -s \
  -H "X-Auth-Token: $OS-Token" \
  -H "Content-Type: application/json" \
  -d '{
    "project": {
        "domain_id": "28e87a35508f4e2d9d8cbb66657c59fa",
	"name": "CS-Lab-Project"
        }
    }' \
  "http://controller:5000/v3/projects" | python -mjson.tool
```

## Tạo User
```
curl -s \
 -H "X-Auth-Token: $OS-Token" \
 -H "Content-Type: application/json" \
 -d '{
   "user": {
       "domain-id": "28e87a35508f4e2d9d8cbb66657c59fa",
       "name": "CS-Lab-User",
       "password": "123456"
     }
   }' \
 "http://controller:5000/v3/users" | python -mjson.tool
```

## List User
```
curl -s \
 -H "X-Auth-Token: $OS-Token" \
 "http://controller:5000/v3/users" | python -mjson.tool
```

## Đổi pass user
```
USER_ID=ebfddf9247eb42ca88ca4c2da299ee56
ORIG_PASS=passwd
NEW_PASS=123456

curl \
 -H "X-Auth-Token: $OS-Token" \
 -H "Content-Type: application/json" \
 -d '{ "user": {"password": "'passwd'", "original_password": "'123456'"} }' \
 "http://controller:5000/v3/users/ebfddf9247eb42ca88ca4c2da299ee56/password"
```

## Reset password
```
USER_ID=ebfddf9247eb42ca88ca4c2da299ee56
NEW_PASS=123456

curl -s -X PATCH \
 -H "X-Auth-Token: $OS-Token" \
 -H "Content-Type: application/json" \
 -d '{ "user": {"password": "'123456'"} }' \
 "http://controller:5000/v3/users/ebfddf9247eb42ca88ca4c2da299ee56" | python -mjson.tool
```

## Tạo Role
```
curl -s \
 -H "X-Auth-Token: $OS-Token" \
 -H "Content-Type: application/json" \
 -d '{
   "role": {
        "name": "CS-Lab-Role"
     }
   }' \
 "http://controller:5000/v3/roles" | python -mjson.tool
```

## Gán role
```
curl -s -X PUT -H "X-Auth-Token: $OS-Token" \
 "http://controller:5000/v3/projects/4eff20217ee242f5bf87776061c96dc8/users/ebfddf9247eb42ca88ca4c2da299ee56/roles/ca379f8ef2ca4fddaf631d5812c8750f"
```

## Lấy token bằng user CS-Lab-User
```
curl -i \
  -H "Content-Type: application/json" \
  -d '
{ "auth": {
    "identity": {
      "methods": ["password"],
      "password": {
        "user": {
          "name": "CS-Lab-User",
          "domain": { "id": "28e87a35508f4e2d9d8cbb66657c59fa" },
          "password": "123456"
        }
      }
    },
    "scope": {
      "project": {
        "name": "CS-Lab-Project",
        "domain": { "id": "28e87a35508f4e2d9d8cbb66657c59fa" }
      }
    }
  }
}' \
  "http://controller:5000/v3/auth/tokens" ; echo
```
## Tạo selfservice network
```
curl -i -X POST \
  -H "X-Auth-Token: $OS-Token"\ 
  -H "Content-Type: application/json" \
  -d '{
    "network": {
        "admin_state_up": true,
        "name": "selfservice"
      }
  }' \
   "http://controller:9696/v2.0/networks" ; echo
```
## Tạo subnet cho selfservice network
```
curl -i -X POST \
  -H "X-Auth-Token: $OS-Token" \
  -H "Content-Type: application/json" \
  -d '{
    "subnet": {
      "name": "subnetofselfservice",
      "network_id": "8388e180-68a1-49e5-b28a-f8a2e71f670f",
      "dns_nameservers": ["8.8.8.8"],
      "allocation_pools": [{
        "start": "172.16.1.10",
        "end": "172.16.1.254"
      }],
     "gateway_ip": "172.16.1.1",
     "ip_version": 4,
     "cidr": "172.16.1.0/24"
    }
  }' \
   "http://controller:9696/v2.0/subnets" ; echo
```
## Tạo router
```
curl -i -X POST \
  -H "X-Auth-Token: $OS-Token" \
  -H "Content-Type: application/json" \
  -d '{
    "router": {
     "name": "router",
     "admin_state_up": true
    }
  }' \
   "http://controller:9696/v2.0/routers" ; echo
```
## Gán subnet selfservice cho router
```
curl -i -X PUT \
  -H "X-Auth-Token: $OS-Token" \
  -H "Content-Type: application/json" \
  -d '{
    "subnet_id": "62e82854-79c9-472c-8ed0-c14feab6c40d"
  }' \
   "http://controller:9696/v2.0/routers/ebdcb9fb-4d62-4d59-a5be-11ca59af76a2/add_router_interface" ; echo

```
## Set gateway của provider network cho router
```
curl -i -X PUT \
  -H "X-Auth-Token: $OS-Token" \
  -H "Content-Type: application/json" \
  -d '{
    "router": {
     "external_gateway_info": {
       "network_id": "114d81e8-e903-41e7-9469-efa64682a3cf"
      }
    }
  }' \
   "http://controller:9696/v2.0/routers/ebdcb9fb-4d62-4d59-a5be-11ca59af76a2" ; echo
```
## Tạo volume
```
curl -i -X POST \
  -H "X-Auth-Token: $OS-Token" \
  -H "Content-Type: application/json" \
  -d '{
    "volume": {
     "name": "volumetest",
     "imageRef": "2ebb8c2b-c586-4995-8132-64263320165e",
     "size": 2
    }
  }' \
   "http://controller:8776/v3/4eff20217ee242f5bf87776061c96dc8/volumes" ; echo

```
## Tạo Server
```
curl -i -X POST \
  -H "X-Auth-Token: gAAAAABabp6UM6aFGP_POIShvYohZTYhw2FbQjP4Qt_nAVeRUJK9tMQq2pKHi1B315j8szPpezZ5Pj_zGlEIEX64FLa4l7ZbaIFor8_JcKavklIs7qei-5Hl_ebka-VB2ynanjc-pDyifSn--kpbSxNz0F29SJAig5rQ6Dc11qKkIKjN2CScya4" \
  -H "Content-Type: application/json" \
  -d '{
    "server": {
      "name": "instance1",
      "imageRef": "",
      "block_device_mapping_v2": [{
        "source_type": "volume",
        "boot_index": "0",
        "uuid": "e5332148-c787-47dc-b5ed-4c7b0918329b",
        "destination_type": "volume"}],
      "flavorRef": "10",
      "max_count": 1,
      "min_count": 1,
      "networks": [{
        "uuid": "8388e180-68a1-49e5-b28a-f8a2e71f670f"
      }]
   }
}' \
   http://controller:8774/v2.1/os-volumes_boot
```
