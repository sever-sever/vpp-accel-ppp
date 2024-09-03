# vpp-accel-ppp
VPP accel-ppp tests

- [L3](#l3)
  - [First](#first)
  - [Second](#second)
- [L2](#l2)
  - [First shared](#first-shared)
  - [Seconf vlan-per-user](#second-vlan-per-user)


# L3
## First 
With this config accel-ppp does not see packets.
Packets go directly via vpp, accel-ppp does nothing.
Without vpp, accel-ppp creates packets

### VyOS config:
```
set interfaces ethernet eth0 address '10.0.2.2/30'
set interfaces ethernet eth0 description 'WAN'
set interfaces ethernet eth1 address '10.0.0.1/30'
set interfaces ethernet eth1 description 'LAN'

set vpp settings interface eth0 driver 'dpdk'
set vpp settings interface eth1 driver 'dpdk'
```
### Accel-ppp config:
```
[ipoe]
verbose=1
interface=eth1,shared=1,mode=L3,ifcfg=1,start=up
noauth=1
local-net=100.64.0.0/28
# ip-pool=ONE
# gw-ip-address=100.64.0.14/28
# proxy-arp=1

# [ip-pool]
# gw-ip-address=100.64.0.14
# 100.64.0.1-10,name=ONE
```
### issues
The `ifcfg` parameter is not affected to the L3 mode at all
It does not matter if you set `1` or `0`

## Second
With this config accel-ppp does not see packets.
The same with ipoe `start=up` option
### VyOS config:
```
set interfaces ethernet eth0 address '10.0.2.2/30'
set interfaces ethernet eth0 description 'WAN'
set interfaces ethernet eth2 address '100.64.0.14/28'
set interfaces ethernet eth2 description 'LAN-direct'

set service ipoe-server authentication mode 'noauth'
set service ipoe-server client-ip-pool ONE range '100.64.0.1-100.64.0.10'
set service ipoe-server default-pool 'ONE'
set service ipoe-server gateway-address '100.64.0.14/28'
set service ipoe-server interface eth2 mode 'l3'
set service ipoe-server interface eth2 network 'shared'
set service ipoe-server name-server '1.1.1.1'
set service ipoe-server name-server '8.8.8.8'

set vpp settings interface eth0 driver 'dpdk'
set vpp settings interface eth1 driver 'dpdk'
set vpp settings interface eth2 driver 'dpdk'
set vpp settings unix poll-sleep-usec '10'
```
### Accel-ppp config:
```
[ipoe]
verbose=1
interface=eth2,shared=1,mode=L3,ifcfg=1,start=dhcpv4
# the same with start=up
noauth=1
ip-pool=ONE
gw-ip-address=100.64.0.14/28
proxy-arp=1

[ip-pool]
gw-ip-address=100.64.0.14
100.64.0.1-10,name=ONE
```

# L2
## first shared
With this configuration IPoE does not work.
Accel-ppp does not see packets (vpp drops unicast packets)
### VyOS config:
```
set interfaces ethernet eth0 address '10.0.2.2/30'
set interfaces ethernet eth0 description 'WAN'
set interfaces ethernet eth2 description 'LAN-direct'

set service ipoe-server authentication mode 'noauth'
set service ipoe-server client-ip-pool ONE range '100.64.0.1-100.64.0.10'
set service ipoe-server default-pool 'ONE'
set service ipoe-server gateway-address '100.64.0.14/28'
set service ipoe-server interface eth2 mode 'l2'
set service ipoe-server interface eth2 network 'shared'
set service ipoe-server name-server '1.1.1.1'
set service ipoe-server name-server '8.8.8.8'

set vpp settings interface eth0 driver 'dpdk'
set vpp settings interface eth1 driver 'dpdk'
set vpp settings interface eth2 driver 'dpdk'
set vpp settings unix poll-sleep-usec '10'
```
### Accel-ppp config:
```
[ipoe]
verbose=1
interface=eth2,shared=1,mode=L2,ifcfg=1,start=dhcpv4,ipv6=1
noauth=1
ip-pool=ONE
gw-ip-address=100.64.0.14/28
proxy-arp=1

[ip-pool]
gw-ip-address=100.64.0.14
100.64.0.1-10,name=ONE

```
### half-fix
To get it working we have to set address on eth2, otherwise unicast packets are dropped by vpp
```
set interfaces ethernet eth2 address 100.64.0.14/28
```
### issues
This config will have issues as we have 2 gateways
the first on the `eth2`, and the second on the `ipoeX` interface
```
vyos@ipoe-server:~$ ip r
default nhid 47 via 10.0.2.1 dev eth0 proto static metric 20 
10.0.0.0/30 dev eth1 proto kernel scope link src 10.0.0.1 dead linkdown 
10.0.2.0/30 dev eth0 proto kernel scope link src 10.0.2.2 
100.64.0.0/28 dev eth2 proto kernel scope link src 100.64.0.14 
100.64.0.2 dev ipoe0 proto kernel scope link src 100.64.0.14 
vyos@ipoe-server:~$ 


# the client site after reboot cannot ping the Internet, but can ping only gw
vyos@client-11:~$ ping 1.1.1.1
PING 1.1.1.1 (1.1.1.1) 56(84) bytes of data.
From 100.64.0.14 icmp_seq=1 Destination Net Unreachable
From 100.64.0.14 icmp_seq=2 Destination Net Unreachable
```
vpp does not delete old subscriber addresses, when subsciber off
in this example the previous client address 100.64.0.1 still in fib
but we do not have it in the kernel.
Routes exist even if we restart accel-ppp
```
vyos@ipoe-server:~$ sudo systemctl restart accel-ppp@ipoe.service

26@100.64.0.15/32
  unicast-ip4-chain
  [@0]: dpo-load-balance: [proto:ip4 index:27 buckets:1 uRPF:39 to:[0:0]]
    [0] [@0]: dpo-drop ip4
27@100.64.0.14/32
  unicast-ip4-chain                                                            
  [@0]: dpo-load-balance: [proto:ip4 index:28 buckets:1 uRPF:29 to:[2:656]]
    [0] [@12]: dpo-receive: 100.64.0.14 on eth2
28@100.64.0.1/32
  unicast-ip4-chain
  [@0]: dpo-load-balance: [proto:ip4 index:29 buckets:1 uRPF:32 to:[19:1596]]
    [0] [@5]: ipv4 via 100.64.0.1 eth2: mtu:1500 next:7 flags:[] 0c31799f00000c647d1000020800
29@100.64.0.0/28
  unicast-ip4-chain
  [@0]: dpo-load-balance: [proto:ip4 index:30 buckets:1 uRPF:40 to:[0:0]]
    [0] [@4]: ipv4-glean: [src:100.64.0.0/28] eth2: mtu:1500 next:3 flags:[] ffffffffffff0c647d1000020806
30@100.64.0.2/32
  unicast-ip4-chain
  [@0]: dpo-load-balance: [proto:ip4 index:31 buckets:1 uRPF:46 to:[0:0]]
    [0] [@5]: ipv4 via 100.64.0.2 eth2: mtu:1500 next:7 flags:[] 0c31799f00000c647d1000020800
vpp#   
```
## second vlan-per-user
### VyOS configuration:
```
set interfaces ethernet eth0 address '10.0.2.2/30'
set interfaces ethernet eth0 description 'WAN'
set interfaces ethernet eth2 description 'LAN-direct'

set service ipoe-server authentication mode 'noauth'
set service ipoe-server client-ip-pool ONE range '100.64.0.1-100.64.0.10'
set service ipoe-server default-pool 'ONE'
set service ipoe-server gateway-address '100.64.0.14/28'
set service ipoe-server interface eth2.11 network 'vlan'
set service ipoe-server interface eth2.12 network 'vlan'
set service ipoe-server name-server '1.1.1.1'
set service ipoe-server name-server '8.8.8.8'

set vpp settings interface eth0 driver 'dpdk'
set vpp settings interface eth2 driver 'dpdk'
set vpp settings lcp route-no-paths
set vpp settings unix poll-sleep-usec '10'
```
### Accel-ppp config:
```
[ipoe]
verbose=1
interface=eth2.11,shared=0,mode=L2,ifcfg=1,start=dhcpv4,ipv6=1
interface=eth2.12,shared=0,mode=L2,ifcfg=1,start=dhcpv4,ipv6=1
noauth=1
ip-pool=ONE
gw-ip-address=100.64.0.14/28
proxy-arp=1

[ip-pool]
gw-ip-address=100.64.0.14
100.64.0.1-10,name=ONE

```
### issues
- accel-ppp does not see packets
- vpp does not see packets in VLANs
### half-fix
Enable VLANs in vpp:
```
sudo vppctl
set interface promiscuous on eth2
```
Add any unicast address to VLAN interfaces:
```
set interfaces ethernet eth2 vif 11 address 192.0.2.1/32
set interfaces ethernet eth2 vif 12 address 192.0.2.1/32
```
Sometimes only one subscriber works, sometime all does not work.
Logs:
```
Sep 03 14:53:35 ipoe-server vpp[17570]: interface: hw_add_del_mac_address: vnet_hw_interface_add_del_mac_address: Secondary MAC Addresses not supported for interface index 0
Sep 03 14:53:35 ipoe-server vpp[17570]: interface: hw_add_del_mac_address: vnet_hw_interface_add_del_mac_address: Secondary MAC Addresses not supported for interface index 0
Sep 03 14:53:35 ipoe-server vpp[17570]: interface: hw_add_del_mac_address: vnet_hw_interface_add_del_mac_address: Secondary MAC Addresses not supported for interface index 0
Sep 03 14:53:35 ipoe-server vpp[17570]: interface: hw_add_del_mac_address: vnet_hw_interface_add_del_mac_address: Secondary MAC Addresses not supported for interface index 0
Sep 03 14:53:35 ipoe-server vpp[17570]: interface: hw_add_del_mac_address: vnet_hw_interface_add_del_mac_address: Secondary MAC Addresses not supported for interface index 0
Sep 03 14:53:35 ipoe-server vpp[17570]: interface: hw_add_del_mac_address: vnet_hw_interface_add_del_mac_address: Secondary MAC Addresses not supported for interface index 0
Sep 03 14:53:35 ipoe-server vpp[17570]: interface: hw_add_del_mac_address: vnet_hw_interface_add_del_mac_address: Secondary MAC Addresses not supported for interface index 0
Sep 03 14:53:35 ipoe-server vpp[17570]: interface: hw_add_del_mac_address: vnet_hw_interface_add_del_mac_address: Secondary MAC Addresses not supported for interface index 0
Sep 03 14:53:35 ipoe-server vpp[17570]: interface: hw_add_del_mac_address: vnet_hw_interface_add_del_mac_address: Secondary MAC Addresses not supported for interface index 0
Sep 03 14:53:35 ipoe-server vpp[17570]: interface: hw_add_del_mac_address: vnet_hw_interface_add_del_mac_address: Secondary MAC Addresses not supported for interface index 0
Sep 03 14:53:35 ipoe-server vpp[17570]: interface: hw_add_del_mac_address: vnet_hw_interface_add_del_mac_address: Secondary MAC Addresses not supported for interface index 0
Sep 03 14:53:35 ipoe-server vpp[17570]: interface: hw_add_del_mac_address: vnet_hw_interface_add_del_mac_address: Secondary MAC Addresses not supported for interface index 0
Sep 03 14:53:35 ipoe-server vpp[17570]: interface: hw_add_del_mac_address: vnet_hw_interface_add_del_mac_address: Secondary MAC Addresses not supported for interface index 0
Sep 03 14:53:35 ipoe-server vpp[17570]: interface: hw_add_del_mac_address: vnet_hw_interface_add_del_mac_address: Secondary MAC Addresses not supported for interface index 0
Sep 03 14:53:35 ipoe-server zebra[1363]: [VTVCM-Y2NW3] Configuration Read in Took: 00:00:00

```
Unicast asddresses `192.0.2.x/32` are droped by accel-ppp:
```
vyos@ipoe-server# run show int
Codes: S - State, L - Link, u - Up, D - Down, A - Admin Down
Interface    IP Address      MAC                VRF        MTU  S/L    Description
-----------  --------------  -----------------  -------  -----  -----  -------------
eth0         10.0.2.2/30     0c:64:7d:10:00:00  default   1500  u/u    WAN
eth1         10.0.0.1/30     0c:64:7d:10:00:01  default   1500  u/D    LAN
eth2         100.64.0.14/28  0c:64:7d:10:00:02  default   1500  u/u    LAN-direct
eth2.11      -               0c:64:7d:10:00:02  default   1500  u/u
eth2.12      -               0c:64:7d:10:00:02  default   1500  u/u

```