# Build order and verification checklist

## Before you start
On every switch: `enable` > `erase startup-config` > `delete vlan.dat` > `reload`
(A leftover VTP revision number can overwrite your VLANs, so always start clean.)
Cable the lab exactly as in the topology diagram, then paste configs in this order.

## Paste order
1. `1_MLS.txt`        (server creates the VLANs)
2. `4_Switch3.txt` and `5_Switch4.txt`   (transparent, local VLANs)
3. `3_Switch2.txt`, `6_Switch5.txt`
4. `2_Switch1.txt`, `7_Switch6.txt`
5. On each PC: Desktop > IP Configuration > DHCP

Wait about 30-50 seconds for STP to settle before verifying.

## Verify and screenshot (these become the slides)

| Device | Command | You should see |
|---|---|---|
| MLS | `show vtp status` | Server, domain cisco.com, 9 VLANs |
| Switch1/2/5/6 | `show vtp status` | Client, SAME revision and MD5 digest as MLS |
| Switch3/4 | `show vtp status` | Transparent |
| Any switch | `show vlan brief` | VLANs 10-40 with names |
| Any switch | `show interfaces trunk` | Trunk ports listed |
| Switch2 | `show spanning-tree summary` | Root bridge for: marketing |
| Switch3 | `show spanning-tree summary` | Root bridge for: sales |
| Switch4 | `show spanning-tree summary` | Root bridge for: engineering |
| Switch5 | `show spanning-tree summary` | Root bridge for: finance |
| Any switch | `show spanning-tree vlan 10` | Root / Designated / Blocking ports |
| Switch1 | `show spanning-tree summary` | PortFast / BPDU guard info |
| MLS | `show ip dhcp binding` | 2 leases per VLAN (8 total) |
| MLS | `show ip interface brief` | Vlan10-40 up/up |
| PC0 | `ipconfig`, `ping 30.1.1.2` | DHCP address, inter-VLAN ping works |

## If something fails
- VLANs missing on a client: check `vtp password` and domain match, and that the trunk is up.
- No DHCP lease: check the PC's access VLAN and `show ip dhcp pool` on MLS.
- Root bridge blank on a transparent switch: the VLANs were not created locally.
