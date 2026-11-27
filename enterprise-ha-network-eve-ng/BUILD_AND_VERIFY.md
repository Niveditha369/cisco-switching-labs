# Build and Verify Guide

## 1. Requirements

- EVE-NG with Cisco IOSv (routers), Cisco IOSvL2 (switches), VPCS and Linux nodes
- Console access to every node (`terminal length 0` on routers and switches keeps output on one screen)

## 2. Nodes

| Node | Template | Role |
|---|---|---|
| ISP4, ISP5, ISP6 | IOSv | Internet side, eBGP |
| R1, R2, R3 | IOSv | Edge routers (PAT, default route, OSPF) |
| CORE1, CORE2 | IOSvL2 | Layer 3 cores (SVIs, HSRP, STP roots) |
| ACC-SW, SALES-SW, SRV-SW | IOSvL2 | Layer 2 access switches |
| DHCP-SRV | IOSv | DHCP server |
| FILE-SRV, QNAP-NAS | Linux | Server room hosts |
| VPC12-14 | VPCS | Accounts users (VLAN 100) |
| VPC15-17 | VPCS | Sales users (VLAN 200) |

## 3. Cabling (28 links)

**Internet side**

| From | To | Subnet |
|---|---|---|
| ISP4 Gi0/0 | ISP5 Gi0/0 | 100.1.1.0/30 |
| ISP5 Gi0/1 | ISP6 Gi0/0 | 200.1.1.0/30 |
| ISP4 Gi0/1 | R1 Gi0/0 | 10.1.1.0/30 |
| ISP5 Gi0/2 | R2 Gi0/0 | 20.1.1.0/30 |
| ISP6 Gi0/1 | R3 Gi0/0 | 30.1.1.0/30 |

**Edge routers to cores (routed links)**

| From | To |
|---|---|
| R1 Gi0/1 | CORE1 Gi0/0 |
| R1 Gi0/2 | CORE2 Gi0/0 |
| R2 Gi0/1 | CORE1 Gi0/1 |
| R2 Gi0/2 | CORE2 Gi0/1 |
| R3 Gi0/1 | CORE1 Gi0/2 |
| R3 Gi0/2 | CORE2 Gi0/2 |

**Core to core (EtherChannel Po1)**

| From | To |
|---|---|
| CORE1 Gi1/0 | CORE2 Gi1/0 |
| CORE1 Gi1/1 | CORE2 Gi1/1 |

**Cores to access switches (dual-homed trunks)**

| From | To |
|---|---|
| ACC-SW Gi0/0 | CORE1 Gi2/0 |
| ACC-SW Gi0/1 | CORE2 Gi2/0 |
| SALES-SW Gi0/0 | CORE1 Gi2/1 |
| SALES-SW Gi0/1 | CORE2 Gi2/1 |
| SRV-SW Gi0/0 | CORE1 Gi2/2 |
| SRV-SW Gi0/1 | CORE2 Gi2/2 |

**Endpoints**

| From | To | VLAN |
|---|---|---|
| ACC-SW Gi1/0, Gi1/1, Gi1/2 | VPC12, VPC13, VPC14 | 100 |
| SALES-SW Gi1/0, Gi1/1, Gi1/2 | VPC15, VPC16, VPC17 | 200 |
| SRV-SW Gi1/0 | DHCP-SRV | 10 |
| SRV-SW Gi1/1 | FILE-SRV | 10 |
| SRV-SW Gi1/2 | QNAP-NAS | 10 |

## 4. Addressing

| Item | Value |
|---|---|
| AS numbers | ISP4 65004, ISP5 65005, ISP6 65006 |
| "Web servers" | ISP4 Lo0 4.4.4.4/32, ISP6 Lo0 8.8.8.8/32 |
| Router to CORE1 | 172.16.1.0/30, 172.16.2.0/30, 172.16.3.0/30 (router .1, core .2) |
| Router to CORE2 | 172.16.1.4/30, 172.16.2.4/30, 172.16.3.4/30 (router .5, core .6) |
| VLAN 10 Servers | 192.168.10.0/24 (VIP .1, CORE1 .2, CORE2 .3, DHCP-SRV .10, FILE-SRV .20, QNAP-NAS .30) |
| VLAN 100 Accounts | 192.168.100.0/24 (VIP .1, CORE1 .2, CORE2 .3) |
| VLAN 200 Sales | 192.168.200.0/24 (VIP .1, CORE1 .2, CORE2 .3) |

**Roles**

| | CORE1 | CORE2 |
|---|---|---|
| VLAN 10 and 100 | STP root, HSRP active | STP secondary, HSRP standby |
| VLAN 200 | STP secondary, HSRP standby | STP root, HSRP active |

## 5. Build order

Paste the matching section of [configs/all-configs.txt](configs/all-configs.txt) into each device, then run `write memory`. Start the nodes in stages: ISPs and routers, then cores, then access switches, then end devices.

1. ISP4, ISP5, ISP6
2. R1, R2, R3
3. CORE1, CORE2
4. ACC-SW, SALES-SW, SRV-SW
5. DHCP-SRV, then FILE-SRV and QNAP-NAS
6. Run `ip dhcp` on VPC12-17

## 6. Verification

### 6.1 BGP

| Device | Command | Expected result |
|---|---|---|
| ISP4 | `show ip bgp summary`, `show ip route bgp` | Neighbor 100.1.1.2 established; routes to 5.5.5.5, 8.8.8.8, 20.1.1.0 and 30.1.1.0 marked `B` |
| ISP5 | `show ip bgp summary` | Neighbors 100.1.1.1 and 200.1.1.2 established |
| ISP6 | `show ip bgp summary` | Neighbor 200.1.1.1 established |

![ISP4](images/02-isp4-bgp-summary-and-routes.png)
![ISP5](images/03-isp5-bgp-summary.png)
![ISP6](images/04-isp6-bgp-summary.png)

### 6.2 Edge routers

| Device | Command | Expected result |
|---|---|---|
| R1 | `show ip interface brief` | Gi0/0, Gi0/1, Gi0/2 up/up |
| R1, R2, R3 | `ping 8.8.8.8 source GigabitEthernet0/0`, `ping 4.4.4.4 source GigabitEthernet0/0` | 100% success |

![R1](images/05-r1-interfaces-and-internet-ping.png)
![R2](images/06-r2-internet-ping.png)
![R3](images/07-r3-internet-ping.png)

### 6.3 Layer 2 (VLANs, EtherChannel, Rapid PVST)

| Device | Command | Expected result |
|---|---|---|
| CORE1 | `show vlan brief`, `show etherchannel summary` | VLANs 10, 100, 200; Po1 `SU` (LACP) with Gi1/0 and Gi1/1 `P` |
| CORE2 | `show etherchannel summary` | Po1 `SU` (LACP) with Gi1/0 and Gi1/1 `P` |
| CORE1 | `show spanning-tree vlan 10,100,200` | Root for VLAN 10 and 100 |
| CORE2 | `show spanning-tree vlan 10,100,200` | Root for VLAN 200 |
| ACC-SW | `show interfaces trunk`, `show spanning-tree vlan 100` | Both uplinks trunking; Gi0/0 root port, Gi0/1 alternate |
| SALES-SW | `show spanning-tree vlan 200` | Gi0/1 root port, Gi0/0 alternate |
| SRV-SW | `show spanning-tree vlan 10` | Gi0/0 root port, Gi0/1 alternate |

![CORE1 VLANs and EtherChannel](images/08-core1-vlans-and-etherchannel.png)
![CORE2 EtherChannel](images/09-core2-etherchannel.png)
![CORE1 STP](images/10-core1-stp.png)
![CORE2 STP](images/11-core2-stp.png)
![ACC-SW](images/12-accsw-trunks-and-stp.png)
![SALES-SW](images/13-salessw-stp.png)
![SRV-SW](images/14-srvsw-stp.png)

### 6.4 HSRP and OSPF

| Device | Command | Expected result |
|---|---|---|
| CORE1 | `show standby brief` | Active for Vl10 and Vl100, Standby for Vl200 |
| CORE2 | `show standby brief` | Active for Vl200, Standby for Vl10 and Vl100 |
| CORE1 | `show ip ospf neighbor`, `show ip route ospf` | R1, R2, R3 in FULL; `O*E2 0.0.0.0/0` via R1 (172.16.1.1) |
| R1 | `show ip ospf neighbor`, `show ip route ospf` | CORE1 and CORE2 in FULL; 192.168.10.0, 192.168.100.0 and 192.168.200.0 learned |

![CORE1 HSRP and OSPF](images/15-core1-hsrp-ospf-routes.png)
![CORE2 HSRP](images/16-core2-hsrp.png)
![R1 OSPF](images/17-r1-ospf-neighbors-and-routes.png)

### 6.5 DHCP

| Device | Command | Expected result |
|---|---|---|
| VPC12-14 | `ip dhcp`, `show ip` | 192.168.100.x/24, gateway 192.168.100.1 |
| VPC15-17 | `ip dhcp`, `show ip` | 192.168.200.x/24, gateway 192.168.200.1 |
| DHCP-SRV | `show ip dhcp pool`, `show ip dhcp binding` | Both pools show 3 leases; six bindings listed |

![DHCP-SRV](images/18-dhcp-pools-and-bindings.png)
![VPC12](images/19-vpc12-dhcp.png)
![VPC15](images/20-vpc15-dhcp.png)

### 6.6 End-to-end connectivity

From VPC12:

| Command | Expected result |
|---|---|
| `ping 192.168.100.1` | Replies from the HSRP virtual gateway |
| `ping 192.168.200.4` | Replies (Accounts reaches Sales before the ACL is applied) |
| `ping 8.8.8.8`, `ping 4.4.4.4` | Replies via PAT and BGP |
| `trace 8.8.8.8` | Path: CORE1, R1 (172.16.1.1), ISP4 (10.1.1.1), ISP5 (100.1.1.2), ISP6 (200.1.1.2) |

From VPC15: `ping 8.8.8.8` replies.
On R1: `show ip nat translations` and `show ip nat statistics` show 192.168.x.x addresses translated to 10.1.1.2.

![VPC12 tests](images/21-vpc12-connectivity-tests.png)
![VPC15 internet](images/22-vpc15-internet-ping.png)
![R1 NAT](images/23-r1-nat.png)

### 6.7 Server reachability

From VPC12 (and VPC15), run `ping 192.168.10.20` and `ping 192.168.10.30`.
Expected result: replies from FILE-SRV and QNAP-NAS through the HSRP gateway.

### 6.8 Failover tests

Restore each failure before starting the next one.

**A. ISP link failure**

1. On CORE1, run `show ip route 0.0.0.0`. The next hop is R1 (172.16.1.1).
2. On R1, shut `GigabitEthernet0/0` and wait about 15 seconds.
3. Run `show ip route 0.0.0.0` on CORE1 again. Expected result: the next hop moves to R2 (172.16.2.1).
4. On VPC12, run `trace 8.8.8.8`. Expected result: the path now goes through R2 and ISP5.
5. Run `no shutdown` on R1 Gi0/0 to restore.

**B. Core switch failure**

1. On VPC12, start a continuous ping: `ping 8.8.8.8 -t`.
2. Shut all interfaces on CORE1 (or stop the node).
3. Expected result: a few timeouts, then replies resume. `show standby brief` on CORE2 shows Active for all three VLANs, and `show spanning-tree vlan 100` on CORE2 reports "This bridge is the root".
4. Restore CORE1. Expected result: `show standby brief` on CORE1 shows Active for Vl10 and Vl100 again (preempt).

**C. EtherChannel member failure**

1. On CORE1, shut `GigabitEthernet1/0`.
2. Run `show etherchannel summary`. Expected result: Po1 stays `SU` with Gi1/1 `P`, and traffic keeps flowing.
3. Run `no shutdown` on Gi1/0 to restore.

### 6.9 ACL isolation (optional)

Apply the following on **both CORE1 and CORE2**, because either core can be the active gateway:

```
configure terminal
ip access-list extended BLOCK-ACC-TO-SALES
 deny ip 192.168.100.0 0.0.0.255 192.168.200.0 0.0.0.255
 permit ip any any
ip access-list extended BLOCK-SALES-TO-ACC
 deny ip 192.168.200.0 0.0.0.255 192.168.100.0 0.0.0.255
 permit ip any any
interface Vlan100
 ip access-group BLOCK-ACC-TO-SALES in
interface Vlan200
 ip access-group BLOCK-SALES-TO-ACC in
end
write memory
```

Then run these from VPC12:

| Command | Expected result |
|---|---|
| `ping 192.168.200.4` | No replies (Accounts blocked from Sales) |
| `ping 192.168.10.20` | Replies (servers still reachable) |
| `ping 8.8.8.8` | Replies (internet still reachable) |

On CORE1, `show access-lists` shows match counters on the deny line.
