# Enterprise Routing and Wireless (Cisco Packet Tracer)

A headquarters and two branch offices connected with multi-area OSPF, internet access through PAT, inter-VLAN routing, DHCP, ACLs, SSH, a PAgP EtherChannel and a wireless access point.


## Topology

![Network topology](images/01-topology.png)

## Devices

| Device | Model | Role |
|---|---|---|
| ISP-R | 2911 | Provider router with the public Server0 |
| HQ-R2 | 2911 | Internet edge with static default route and PAT |
| HQ-DSW | 3560-24PS | Layer 3 switch with SVIs and routed ports |
| HQ-ASW | 2960-24TT | HQ access switch |
| HQ-R1 | 2911 | WAN hub and OSPF area border router |
| BR1-R | 2911 | Branch 1 router with router-on-a-stick |
| BR1-SW | 2960-24TT | Branch 1 switch |
| BR2-R | 2911 | Branch 2 router |
| BR2-SW | 2960-24TT | Branch 2 switch |
| BR2-AP | Access Point-PT | Branch 2 wireless access point |

Hosts are PC0 and PC1 (HQ VLAN 10), Server1 (HQ VLAN 20), PC2 (BR1 VLAN 10), PC3 (BR1 VLAN 20), PC4, PC5 and the laptops BR2-LAP1 and BR2-LAP2 (BR2), and the public Server0.

## Addressing

| Link or LAN | Network | Addresses |
|---|---|---|
| ISP-R to HQ-R2 | 203.0.113.0/30 | ISP-R .2, HQ-R2 .1 |
| HQ-R2 to HQ-DSW | 192.168.1.4/30 | HQ-R2 .6, HQ-DSW .5 |
| HQ-R1 to HQ-DSW | 192.168.1.0/30 | HQ-R1 .2, HQ-DSW .1 |
| HQ-R1 to BR1-R | 192.168.2.0/30 | HQ-R1 .1, BR1-R .2 |
| HQ-R1 to BR2-R | 192.168.2.4/30 | HQ-R1 .5, BR2-R .6 |
| BR1-R to BR2-R | 192.168.2.8/30 | BR1-R .9, BR2-R .10 |
| ISP-R to Server0 | 198.51.100.0/24 | ISP-R .1, Server0 .10 |
| HQ VLAN 10 | 10.0.10.0/24 | Gateway .1 on the HQ-DSW SVI |
| HQ VLAN 20 | 10.0.20.0/24 | Gateway .1, Server1 .10 |
| BR1 VLAN 10 | 10.1.10.0/24 | Gateway .1 on BR1-R Gig0/1.10 |
| BR1 VLAN 20 | 10.1.20.0/24 | Gateway .1 on BR1-R Gig0/1.20 |
| BR2 LAN | 10.1.30.0/24 | Gateway .1 on BR2-R Gig0/1 |

## Concepts Covered

- **OSPF:** Process 1 with area 0 (HQ-R2, HQ-DSW, HQ-R1) and area 1 (HQ-R1, BR1-R, BR2-R), an area range summary of 10.1.0.0/16, passive LAN interfaces, MD5 authentication and point-to-point networks.
- **Internet access:** A static default route and PAT on HQ-R2, with `default-information originate` to share the default route.
- **Inter-VLAN routing:** Router-on-a-stick on BR1-R, and SVIs with routed ports on HQ-DSW.
- **DHCP:** A relay to Server1 at HQ and router pools at the branches.
- **Security:** An extended ACL on BR1 VLAN 10, a standard ACL on the VTY lines, SSH version 2, enable secret and a banner.
- **EtherChannel:** PAgP (mode desirable) between HQ-DSW and HQ-ASW.
- **Wireless:** BR2-AP with SSID BR2-WiFi, WPA2-PSK and AES.
- **Redundancy:** BR1 reaching HQ through BR2 when the direct link is shut.

## Verification Evidence

### OSPF Neighbors and Routes

![HQ-R1 neighbors and routes](images/02-ospf-neighbors-routes-hq-r1.png)

HQ-R1 has three neighbors in FULL state: 5.5.5.5 (HQ-DSW), 3.3.3.3 (BR1-R) and 4.4.4.4 (BR2-R). It learned the default route as `O*E2` through 192.168.1.1. Gig0/1 shows area 1, network type POINT-TO-POINT and message digest authentication with key id 1.

![BR1-R OSPF routes](images/03-ospf-routes-br1-r.png)

BR1-R learns the HQ networks as inter-area routes through 192.168.2.1. It reaches 10.1.30.0 directly through 192.168.2.10, and it learned the default route through HQ-R1.

![HQ-DSW OSPF summary](images/04-ospf-summary-hq-dsw.png)

HQ-DSW has FULL neighbors 1.1.1.1 and 2.2.2.2. It holds one inter-area route, 10.1.0.0/16, in place of the three branch LANs. This is the area range summary from HQ-R1.

![HQ-R2 default route](images/05-ospf-default-route-hq-r2.png)

HQ-R2 has a FULL neighbor 5.5.5.5, the static default route `S* 0.0.0.0/0` through 203.0.113.2, and the same 10.1.0.0/16 summary.

![BR2-R neighbors](images/06-ospf-neighbors-br2-r.png)
![BR1-R neighbors](images/07-ospf-neighbors-br1-r.png)

BR2-R and BR1-R both show FULL neighbors toward HQ-R1 and toward each other.

### PAT and Internet Reachability

![PC0 ping to Server0](images/08-pat-ping-pc0-server0.png)

PC0 reaches Server0 at 198.51.100.10 with 0% loss on the repeat run.

![PAT translations on HQ-R2](images/09-pat-translations-hq-r2.png)

The translations table maps PC0 (10.0.10.11) to 203.0.113.1 for each ICMP session. The statistics show GigabitEthernet0/1 as the outside interface and GigabitEthernet0/0 as the inside interface.

### VLANs, Trunks and EtherChannel

![HQ-ASW VLANs, EtherChannel and trunk](images/10-vlans-etherchannel-trunk-hq-asw.png)

HQ-ASW has VLAN 10 on Fa0/3 and Fa0/4 and VLAN 20 on Fa0/2. Po1 is in use with protocol PAgP and both members, Fa0/1 and Fa0/5, bundled. The trunk on Po1 is up with 802.1q.

![HQ-DSW EtherChannel and interfaces](images/12-etherchannel-interfaces-hq-dsw.png)

HQ-DSW shows Po1 with PAgP and members Fa0/3 and Fa0/4 bundled. Fa0/1 and Fa0/2 are routed ports, and Vlan10 and Vlan20 are up.

![HQ-DSW VLANs](images/14-vlans-hq-dsw.png)

![BR1-SW VLANs and trunk](images/11-vlans-trunk-br1-sw.png)

BR1-SW has VLAN 10 on Fa0/2 and VLAN 20 on Fa0/3, with Fa0/1 trunking VLANs 1, 10 and 20.

![BR1-R subinterfaces](images/13-subinterfaces-br1-r.png)

On BR1-R, Gig0/1 has no address and the subinterfaces Gig0/1.10 and Gig0/1.20 are up with 10.1.10.1 and 10.1.20.1.

### DHCP

![BR1-R DHCP bindings](images/15-dhcp-binding-br1-r.png)
![BR2-R DHCP bindings](images/16-dhcp-binding-br2-r.png)

BR1-R holds two bindings, 10.1.10.11 and 10.1.20.11. BR2-R holds four bindings, from 10.1.30.11 to 10.1.30.15.

![PC0 ipconfig](images/17-ipconfig-pc0.png)
![PC3 ipconfig](images/18-ipconfig-pc3.png)
![PC5 ipconfig](images/19-ipconfig-pc5.png)

PC0 received 10.0.10.11 with gateway 10.0.10.1. PC3 received 10.1.20.11 with gateway 10.1.20.1. PC5 received 10.1.30.12 with gateway 10.1.30.1.

### ACL on BR1 VLAN 10

![PC2 opens the web page](images/20-acl-http-allowed-pc2.png)
![PC2 ping is blocked](images/21-acl-ping-blocked-pc2.png)
![PC3 ping is allowed](images/22-acl-ping-allowed-pc3.png)

PC2 opens `http://10.0.20.10`. Its ping to the same address returns "Destination host unreachable" from 10.1.10.1. PC3, in VLAN 20, pings Server1 with 0% loss.

![ACL counters on BR1-R](images/23-acl-counters-br1-r.png)

The extended ACL BR1-V10-IN shows 12 matches on the HTTP permit, 4 matches on the deny and 2 matches on the final permit.

### SSH and VTY Access

![SSH login from PC3](images/24-ssh-login-pc3.png)
![SSH version on BR1-R](images/25-ssh-version-br1-r.png)

PC3 logs in to BR1-R with SSH and sees the banner. `show ip ssh` reports SSH enabled, version 2.0.

![Server0 SSH attempts](images/26-ssh-vty-acl-server0.png)

From Server0, SSH to ISP-R at 198.51.100.1 succeeds. SSH to HQ-R2 at 203.0.113.1 returns "Connection refused by remote host", because the VTY access list on HQ-R2 does not permit that source.

### Wireless

![BR2-AP Port 1](images/27-wireless-ap-config.png)

BR2-AP Port 1 uses SSID BR2-WiFi, WPA2-PSK and AES.

![BR2-LAP1 settings](images/28-wireless-laptop-config.png)
![BR2-LAP1 ipconfig and ping](images/29-wireless-laptop-ping.png)

BR2-LAP1 uses DHCP on Wireless0 and received 10.1.30.14 with gateway 10.1.30.1. Its ping to 10.1.30.1 has 0% loss, with replies between 32 ms and 78 ms.

### End to End Reachability

![PC4 pings](images/30-end-to-end-pc4.png)
![PC2 ping and ipconfig](images/33-end-to-end-pc2.png)

PC4 reaches PC0 at 10.0.10.11 and Server0 at 198.51.100.10. PC2 has 10.1.10.11 and reaches Server0 with 0% loss.

### Redundancy

![BR1-R after the link was shut](images/31-resilience-br1-r-after-failure.png)

With the HQ-R1 to BR1-R link shut, BR1-R keeps only the neighbor 4.4.4.4. The HQ routes and the default route now go through 192.168.2.10.

![Traceroute from PC3 to Server1](images/32-resilience-tracert-pc3.png)

The first trace from PC3 to Server1 goes through 10.1.20.1, 192.168.2.1 and 192.168.1.1. The second trace goes through 10.1.20.1, 192.168.2.10, 192.168.2.5 and 192.168.1.1, so it passes through BR2-R.

## Lessons Learned

1. **Multi-area OSPF:** An area border router can summarize the branch LANs into one 10.1.0.0/16 route.
2. **Default route:** One static route and `default-information originate` give every router an internet exit.
3. **PAT:** Many private hosts share one public address, and the translation table lists each session.
4. **Inter-VLAN routing:** It can be done with router-on-a-stick or with SVIs on a Layer 3 switch.
5. **DHCP:** A relay serves many VLANs from one server, and branch routers can hold their own pools.
6. **ACLs:** Rule order matters, and the match counters show which rules were used. The `show access-lists` command only runs in privileged mode.
7. **EtherChannel:** PAgP bundles two cables into one logical link.
8. **SSH and VTY ACL:** Each router has its own list, so the same source can be allowed on one router and refused on another.
9. **Wireless:** The access point places wireless clients on the same LAN and DHCP pool as wired PCs.
10. **Redundancy:** OSPF finds a new path through BR2-R when the direct link goes down.

## Repository Contents

| Item | Description |
|---|---|
| `README.md` | This summary with verification evidence. |
| `BUILD_AND_VERIFY.md` | Build order, host settings, wireless checklist and verification commands. |
| `configs/` | Build configuration for each device, plus the SSH key block. |
| `images/` | Topology and verification screenshots. |
