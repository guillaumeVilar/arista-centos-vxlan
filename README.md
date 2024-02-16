# arista-centos-vxlan
A lab to demonstrate how to create a VxLAN tunnel between an Arista device and a CentOS server using containerlab.
## Topology
![Topology](topology.png)

## To run on the CentOS host

### Packages to install
```
# for ip link commands
yum install -y iproute

# optional: for packet capture on CentOS side
yum install -y tcpdump
```

### Physical interface setup
```
ip addr add 10.0.0.2/24 dev et2
ip route add 172.17.0.1/32 via 10.0.0.1
```
### Vxlan setup
```
ip link del vxlan1
ip link add vxlan1 type vxlan id 10000 remote 172.17.0.1 local 10.0.0.2 dstport 4789 dev et2
bridge fdb append 00:00:00:00:00:00 dev vxlan1 dst 172.17.0.1
ip link set up dev vxlan1

ip link add br-vxlan1 type bridge
ip link set dev vxlan1 master br-vxlan1
ip link set dev et1 master br-vxlan1 
ip link set dev br-vxlan1 up
```



### Optional: VxLAN setup with trunk interface
Additionally, it is also possible to make the CentOS bridge VLAN aware.  
This is useful for example if we want to send some traffic tagged to a host (to et1 in this case) or to isolate the traffic.
```
# Re-create the bridge with vlan filtering enabled
ip link del br-vxlan1 
ip link add br-vxlan1 type bridge  vlan_filtering 1
ip link set dev vxlan1 master br-vxlan1
ip link set dev et1 master br-vxlan1 

# Remove default VLAN
bridge vlan del dev br-vxlan1 vid 1 self
bridge vlan del dev et1 vid 1 master
bridge vlan del dev vxlan1 vid 1 master

# Enabling the vlan 100 in the bridge itself
bridge vlan add dev br-vxlan1 vid 100 tagged self
# Equivalent to switchport mode trunk, and allowing vlan 100
bridge vlan add dev et1 vid 100 tagged master
# Equivalent to switchport mode access, and access vlan 100
bridge vlan add dev vxlan1 vid 100 pvid untagged master

# Finally, enable the bridge
ip link set dev br-vxlan1 up
```
Verification: 
```
# bridge vlan show
port    vlan ids
et1      100

vxlan1   100 PVID Egress Untagged

br-vxlan1        100
```
With that config, traffic from/to et1 is expected to be tagged with VLAN 100.

## Verification
### On VTEP2: 
```
ip -d link show vxlan1
ip -d link show br-vxlan1
ip -d link show et1

# bridge fdb show dev vxlan1   
00:00:00:00:00:00 dst 172.17.0.1 via et2 self permanent
00:00:00:00:00:00 dst 172.17.0.1 self permanent

# bridge link
21: et1 state UP @(null): <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9500 master br-vxlan1 state forwarding priority 32 cost 2 
2: vxlan1 state UNKNOWN : <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9450 master br-vxlan1 state forwarding priority 32 cost 2
```


### On host1: 
```
# ping 192.168.0.2
```

### On ceos-vtep1:
```
# show int vxlan1
# show ip route 
# ping 10.0.0.2
```

## Troubleshooting tips if it is not working on CentOS side
1. Even if you are sending some traffic unidirectional (only from host1 to host2) for example, the CentOS needs to have a route to reach the remote VTEP, even if that route is not used.  
In the example, make sure to have this route configured on the CentOS side: 
```
ip route add 172.17.0.1/32 via 10.0.0.1
```

2. Make sure some iptables / firewall on the CentOS side is not blocking some traffic. 