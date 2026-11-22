# Build order and verification

## Paste order
DS1, DS2, AS1, AS2, AS3, AS4, then Server0 and the PCs (`configs/7_Server0_and_PCs.txt`).

Paste each switch in 3-4 chunks and check for `% Invalid input`: (1) basics, VLANs, STP, (2) interfaces, (3) SVIs/HSRP or VLAN 99,
(4) the SSH block on its own. For `crypto key generate rsa`, type `1024` when asked for the modulus.

## EtherChannel note
The Port-channel interface is created before the member ports join it. In my first attempt the ports stayed stand-alone (`I`) until
I rebuilt the bundle in this order. After that both ends showed `Po1(SU)` with `Fa0/1(P) Fa0/2(P)`.

## Verify
| Device | Command | Expected |
|---|---|---|
| DS1, DS2 | `show etherchannel summary` | Po1 `SU`, Fa0/1 and Fa0/2 `P` |
| DS1, DS2 | `show standby brief` | DS1 active 10, 30, 99; DS2 active 20, 50 |
| DS1, DS2 | `show spanning-tree summary` | Rapid-PVST, roots split between the two |
| AS1 | `show ip dhcp snooping` | VLANs 10, 20, 30; uplinks trusted |
| AS1 | `show port-security interface fa0/3` | Sticky, max 1, restrict |
| AS1 | `show interfaces trunk` | Native 999, allowed 10, 20, 30, 50, 99 |
| PC1 | `ipconfig`, `ping 10.10.50.10`, `ping` a VLAN 20 host | DHCP lease, TTL 127 |
| PC1 | Browser: `http://10.10.50.10` | Web page loads |

## Failover test
Take DS1 out of service (shut all its ports), wait ~15 s, then run `show standby brief` and `show spanning-tree summary` on DS2.
Re-enable DS1's ports afterwards; preempt returns VLANs 10, 30 and 99 to DS1.

## Port-security test
1. Add a new PC with a static address in VLAN 10 (for example 10.10.10.50 /24, gateway 10.10.10.1).
2. Disconnect PC1 from AS1 Fa0/3 and connect the new PC there.
3. Ping 10.10.10.1 from the new PC; it fails.
4. On AS1, run `show port-security interface fa0/3` and `show port-security`. The violation count rises and the last source address is the new PC's MAC.
5. Reconnect PC1 afterwards; its sticky MAC is still allowed.
