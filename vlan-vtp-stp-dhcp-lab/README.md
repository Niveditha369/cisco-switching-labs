# Redundant Switched Network: VLAN, VTP, STP/BPDU and DHCP (Cisco Packet Tracer)

A campus-style LAN with 4 VLANs, 6 access/distribution switches plus 1 multilayer switch, redundant links,
a different STP root bridge per VLAN, VTP in server/client/transparent modes, and centralized DHCP.

![Topology](images/topology.png)

## Concepts

| Concept | Implementation |
|---|---|
| VLAN | VLANs 10 marketing, 20 sales, 30 engineering, 40 finance |
| VTP | Domain `cisco.com`, version 2. MLS = server; Switch1, 2, 5, 6 = client; Switch3, 4 = transparent |
| STP / BPDU | PVST. Root bridge per VLAN (priority 4096), backup root (8192). PortFast + BPDU Guard on access ports |
| DHCP | Centralized on the multilayer switch, one pool per VLAN, gateway address excluded |
| Inter-VLAN routing | SVIs on the multilayer switch (`ip routing`) |

## Addressing

| VLAN | Name | Subnet | Gateway | Hosts |
|---|---|---|---|---|
| 10 | marketing | 10.1.1.0/24 | 10.1.1.1 | PC2, PC3 |
| 20 | sales | 20.1.1.0/24 | 20.1.1.1 | PC0, PC1 |
| 30 | engineering | 30.1.1.0/24 | 30.1.1.1 | PC4, PC5 |
| 40 | finance | 40.1.1.0/24 | 40.1.1.1 | PC6, PC7 |

## Device roles

| Device | VTP mode | STP |
|---|---|---|
| MLS (3560-24PS) | Server | DHCP server and gateway |
| Switch1 | Client | Access (VLAN 10, 20) |
| Switch2 | Client | Root VLAN 10, backup VLAN 20 |
| Switch3 | Transparent | Root VLAN 20, backup VLAN 10 |
| Switch4 | Transparent | Root VLAN 30, backup VLAN 40 |
| Switch5 | Client | Root VLAN 40, backup VLAN 30 |
| Switch6 | Client | Access (VLAN 30, 40) |

## Configurations
`configs/` has one file per device, in the order they should be pasted (see `BUILD_AND_VERIFY.md`).

## Verification (real output from the lab)

| Check | Result | Evidence |
|---|---|---|
| VTP server | MLS: Server, cisco.com, v2, revision 17, 9 VLANs | `images/mls_vtp_dhcp.png` |
| VTP client sync | Switch5: Client, revision 17, MD5 digest identical to the server | `images/switch5_vtp_client.png` |
| VTP transparent | Switch3: Transparent, revision 0, own database | `images/switch3_vtp_transparent.png` |
| Root VLAN 10 | Switch2 "Root bridge for: marketing", priority 4106 (4096 + VLAN 10) | `images/switch2_stp_summary.png`, `images/switch2_stp_vlan10.png` |
| Root VLAN 20 | Switch3 "Root bridge for: sales" | `images/switch3_stp_summary.png` |
| Root VLAN 30 | Switch4 "Root bridge for: engineering" | `images/switch4_stp_summary.png` |
| Root VLAN 40 | Switch5 "Root bridge for: finance" | `images/switch5_stp_summary.png` |
| Loop prevention | Switch3 blocks ports in VLANs 1, 30, 40; Switch5 in VLANs 1, 10, 20; Switch1 blocks one uplink per VLAN | STP summaries |
| DHCP | 8 leases, 2 per VLAN | `images/mls_vtp_dhcp.png` |
| Inter-VLAN routing | PC0 (20.1.1.3, VLAN 20) pings 30.1.1.2, TTL 127 (one routed hop). First request times out while ARP resolves | `images/pc0_ipconfig_ping.png` |

## Lessons learned

1. **Transparent switches need their VLANs created locally.** When I revisited the lab, Switch4 only had VLAN 1, so it could not be the VLAN 30 root. `Root bridge for:` was empty until the VLANs were added.
2. **BPDU Guard works.** A trunk port left with old access/PortFast/BPDU Guard settings was err-disabled the moment its neighbor sent a BPDU.
3. **VTP revision and MD5 digest prove sync.** Server and client matched exactly; the transparent switch stayed at revision 0.
4. **Per-VLAN roots balance traffic.** Different VLANs block different ports, so redundant links are actually used.
5. **Trust the DHCP bindings over the diagram.** The leases and MAC addresses showed which VLAN each PC really sat in, and two diagram labels were swapped. Fixing the diagram was enough.
6. **Design trade-off.** The STP roots sit on access-layer switches while the gateways live on the multilayer switch, so some traffic takes a longer path. Moving the roots to the multilayer or distribution layer would be cleaner.

## Next steps
EtherChannel, HSRP/VRRP with two multilayer switches, Rapid-PVST+, DHCP snooping and port security.

## Tools
Cisco Packet Tracer, Cisco IOS (2960 / 3560)
