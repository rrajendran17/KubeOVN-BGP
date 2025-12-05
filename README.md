### KubeOVN-BGP

<img width="1536" height="1024" alt="image" src="https://github.com/user-attachments/assets/0282e9fa-61b1-46d7-80c1-1fc9534689ed" />

Use KubeOVN's BGP support to advertise routes from Harvester clusters to external hosts.This eliminates the need for NAT and provides external connectivity through L3 Integration and when combined with ECMP and BFP provides faster convergence and high availabilty.


- Label Harvester (or Kubernetes) nodes to run kube-ovn-speaker.
```
kubectl label nodes harvester-node-0 ovn.kubernetes.io/bgp=true
```
- Deploy BGP Speaker in Harvester
  - Apply the speaker.yaml with neighbor address and neighbor as from peer bgp configured using FRR.
- Configure BGP with upstream routers (lab or physical network) using FRR.
- Apply the vswitchbgp,vpcbgp,subnetbgp yamls.
- Create a VM attached to overlay network on subnetbgp
- Apply the provider network and vlan yaml
- Edit the subnet and add vlan

  ```
  26a5f988-c11b-4027-b221-ac5ddc1fa241
    Bridge br-pn1
        Port patch-localnet.subnetbgp-to-br-int
            Interface patch-localnet.subnetbgp-to-br-int
                type: patch
                options: {peer=patch-br-int-to-localnet.subnetbgp}
        Port ens7
            trunks: [0]
            Interface ens7
        Port br-pn1
            Interface br-pn1
                type: internal
    Bridge br-int
        fail_mode: secure
        datapath_type: system
        Port br-int
            Interface br-int
                type: internal
        Port ovn0
            Interface ovn0
                type: internal
        Port "7eff0_37a8eec_h"
            Interface "7eff0_37a8eec_h"
        Port patch-br-int-to-localnet.subnetbgp
            Interface patch-br-int-to-localnet.subnetbgp
                type: patch
                options: {peer=patch-localnet.subnetbgp-to-br-int}
        Port mirror0
            Interface mirror0
                type: internal
    ovs_version: "3.5.3"
  ```
- Annotate the pod and subnetbgp, so that kuebovn advertises the route to peer.
  - kubectl annotate pod sample ovn.kubernetes.io/bgp=true
  - kubectl annotate subnet subnetbgp ovn.kubernetes.io/bgp=true
- Validate that VM / Pod / Subnet IPs are being advertised externally.
- No NAT needed,you get true L3 integration.

- FRR config on neighbor

```
Hello, this is FRRouting (version 8.1).
Copyright 1996-2005 Kunihiro Ishiguro, et al.

renuka-ThinkPad-P14s-Gen-4# show running-config
Building configuration...

Current configuration:
!
frr version 8.1
frr defaults traditional
hostname renuka-ThinkPad-P14s-Gen-4
log syslog informational
no ipv6 forwarding
service integrated-vtysh-config
!
router bgp 65010
 bgp router-id 192.168.1.74
 neighbor 192.168.0.30 remote-as 65000
 !
 address-family ipv4 unicast
  neighbor 192.168.0.30 route-map ACCEPT-ALL in
  neighbor 192.168.0.30 route-map ACCEPT-ALL out
 exit-address-family
exit
!
route-map ACCEPT-ALL permit 10
exit
!
end

```

- Config on external host (bgp peer)
```
1.Install FRR
2.add config to /etc/frr/frr.conf
3.sudo systemctl restart frr
4.sudo vtysh
5.show running-config
6.show ip bgp summary
7.show ip bgp neighbors
8.show bgp neighbors 192.168.0.30 received-routes (will show routes received from harvester)
9.show ip fib (routes installed in kernel - should see 172.50.10.0/24 installed via 192.168.0.30)

```

- Config on Harvester node (kubeovn bgp speaker)

```
1.Create a dummy interface ens7
sudo ip link add dev ens7 type dummy
sudo ip link set ens7 up
sudo ip addr add 172.50.10.1/24 dev ens7

Once created a provider network with ens7, br-pn1 will automatically get the ip addr from ens7
a route will be created for 172.50.10.0/24 via br-pn1 (if not remove addr from ens7 and add to br-pn1 directly)

ip addr show ens7
157: ens7: <BROADCAST,NOARP,UP,LOWER_UP> mtu 1500 qdisc noqueue master ovs-system state UNKNOWN group default qlen 1000
    link/ether ae:e3:a0:9e:6c:1c brd ff:ff:ff:ff:ff:ff
harvester-node-0:~/bgptests # ip addr show br-pn1
158: br-pn1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UNKNOWN group default qlen 1000
    link/ether ae:e3:a0:9e:6c:1c brd ff:ff:ff:ff:ff:ff
    inet 172.50.10.1/24 scope global br-pn1
       valid_lft forever preferred_lft forever
harvester-node-0:~/bgptests # ip route show 172.50.10.0/24
172.50.10.0/24 dev br-pn1 proto kernel scope link src 172.50.10.1


Now external traffic for 172.50.10.2(VM) will be reaching br-pn1 and via patch port will be forwarded to VM.
But VM needs a default route to send ICMP reply back

add a default route on the VM (172.50.10.2)

sudo ip route add default via 172.50.10.1 dev enp1s0

```

- Neighbors learnt and routes received from Harvester node

```
show ip bgp summary

IPv4 Unicast Summary (VRF default):
BGP router identifier 192.168.1.74, local AS number 65010 vrf-id 0
BGP table version 2
RIB entries 3, using 552 bytes of memory
Peers 1, using 723 KiB of memory

Neighbor        V         AS   MsgRcvd   MsgSent   TblVer  InQ OutQ  Up/Down State/PfxRcd   PfxSnt Desc
192.168.0.30    4      65000        68        72        0    0    0 00:07:42            2        2 N/A

Total number of neighbors 1
renuka-ThinkPad-P14s-Gen-4# show bgp neighbors
192.168.0.30      graceful-restart  json
renuka-ThinkPad-P14s-Gen-4# show bgp neighbors 192.168.0.30 received
% No such neighbor or address family
renuka-ThinkPad-P14s-Gen-4# show bgp neighbors 192.168.0.30 received-routes
% No such neighbor or address family
renuka-ThinkPad-P14s-Gen-4# show ip fib
Codes: K - kernel route, C - connected, S - static, R - RIP,
       O - OSPF, I - IS-IS, B - BGP, E - EIGRP, N - NHRP,
       T - Table, v - VNC, V - VNC-Direct, A - Babel, F - PBR,
       f - OpenFabric,
       > - selected route, * - FIB route, q - queued, r - rejected, b - backup
       t - trapped, o - offload failure

K>* 0.0.0.0/0 [0/600] via 192.168.1.254, wlp2s0, 01:53:02
K>* 10.0.0.0/8 [0/50] via 10.116.240.1, tun0, 00:11:24
B>* 10.52.0.143/32 [20/0] via 192.168.0.30, virbr2, weight 1, 00:03:55
C>* 10.116.240.0/20 is directly connected, tun0, 00:11:24
K>* 147.2.0.0/16 [0/50] via 10.116.240.1, tun0, 00:11:24
K>* 149.44.0.0/16 [0/50] via 10.116.240.1, tun0, 00:11:24
K>* 169.254.0.0/16 [0/1000] is directly connected, virbr0, 02:08:10
B>* 172.50.10.0/24 [20/0] via 192.168.0.30, virbr2, weight 1, 00:03:45
C>* 192.168.0.0/24 is directly connected, virbr2, 00:53:03
C>* 192.168.1.0/24 is directly connected, wlp2s0, 01:53:03
K>* 192.168.1.254/32 [0/50] is directly connected, wlp2s0, 00:11:24
C>* 192.168.121.0/24 is directly connected, virbr1, 00:53:03
K>* 195.135.220.137/32 [0/50] via 192.168.1.254, wlp2s0, 00:11:24

```
