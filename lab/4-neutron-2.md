# Neutron OpenvSwitch
## Controller
Trong file `/etc/neutron/plugins/ml2/ml2_conf.ini`  
```
[ml2]
type_drivers = flat,vlan,vxlan
tenant_network_types = vxlan
mechanism_drivers = openvswitch,l2population
extension_drivers = port_security
```
## Network
Cài gói
```
apt install -y neutron-plugin-openvswitch-agent
```
Trong file `/etc/neutron/plugins/ml2/ml2_conf.ini`
```
[ml2]
type_drivers = flat,vlan,vxlan
tenant_network_types = vxlan
mechanism_drivers = openvswitch,l2population
extension_drivers = port_security
```
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
Tạo OVS provider bridge
```
ovs-vsctl add-br br-provider
ovs-vsctl add-port br-provider ens33
```
Trong file `/etc/network/interfaces`
```
auto ens33
iface ens33 inet manual
up ifconfig ens33 0.0.0.0 up
up ip link set ens33 promisc on
down ip link set ens33 promisc off
down ifconfig ens33 down

auto br-provider
iface br-provider inet dhcp
```
Trong file `/etc/neutron/plugins/ml2/openvswitch_agent.ini`
```
[ovs]
bridge_mappings = provider:br-provider
local_ip = 10.10.10.31

[agent]
tunnel_types = vxlan
l2_population = True

[securitygroup]
firewall_driver = iptables_hybrid
```
## Compute
Cài gói
```
apt install -y neutron-plugin-openvswitch-agent
```
Trong file `/etc/neutron/plugins/ml2/openvswitch_agent.ini`
```
[ovs]
local_ip = 10.10.10.21

[agent]
tunnel_types = vxlan
l2_population = True

[securitygroup]
firewall_driver = iptables_hybrid
```
