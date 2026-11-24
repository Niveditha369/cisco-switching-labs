# Enterprise Routing and Wireless: Build and Verify

## Lab Conventions

- Enable secret: `Cisco@123`.
- SSH user: `admin`, password `Admin@123`, domain `lab.local`.
- OSPF MD5 key 1: `Ospf@2026`.
- Router IDs: HQ-R1 1.1.1.1, HQ-R2 2.2.2.2, BR1-R 3.3.3.3, BR2-R 4.4.4.4, HQ-DSW 5.5.5.5.
- Switch management addresses: HQ-ASW 10.0.20.2, BR1-SW 10.1.10.2, BR2-SW 10.1.30.2.

## Cabling

| From | To |
|---|---|
| ISP-R Gig0/0 | HQ-R2 Gig0/1 |
| ISP-R Gig0/1 | Server0 (cross-over) |
| HQ-R2 Gig0/0 | HQ-DSW Fa0/2 (routed port) |
| HQ-R1 Gig0/0 | HQ-DSW Fa0/1 (routed port) |
| HQ-R1 Gig0/1 | BR1-R Gig0/0 |
| HQ-R1 Gig0/2 | BR2-R Gig0/0 |
| BR1-R Gig0/2 | BR2-R Gig0/2 |
| BR1-R Gig0/1 | BR1-SW Fa0/1 (trunk) |
| BR2-R Gig0/1 | BR2-SW Fa0/1 |
| HQ-DSW Fa0/3 and Fa0/4 | HQ-ASW Fa0/1 and Fa0/5 (PAgP bundle) |
| BR2-AP | BR2-SW Fa0/2 |

Access ports: HQ-ASW Fa0/2 Server1, Fa0/3 PC0, Fa0/4 PC1. BR1-SW Fa0/2 PC2, Fa0/3 PC3. BR2-SW Fa0/3 PC4, Fa0/4 PC5. The laptops have no cable.

## Build Order

1. ISP-R
2. HQ-R2
3. HQ-DSW
4. HQ-ASW
5. HQ-R1
6. BR1-R
7. BR1-SW
8. BR2-R
9. BR2-SW

## Step 1: Paste the Device Configuration

Open the device console and paste the whole file from the `configs/` folder. Each file is one block, and it ends with `end`.

## Step 2: Generate the SSH Key

Run this on every device after its configuration block. The full text is in `configs/ssh-key-block.txt`.

```
configure terminal
crypto key generate rsa
```

Type `1024` at the modulus prompt. Then enter `end` and `write memory`.

## Step 3: Host Settings

1. **Server0:** Open Desktop, then IP Configuration. Set static 198.51.100.10, mask 255.255.255.0 and gateway 198.51.100.1.
2. **Server1:** Set static 10.0.20.10, mask 255.255.255.0 and gateway 10.0.20.1.
3. **Server1 DHCP:** Open Services, then DHCP, and turn it on. Add a pool named HQ-VLAN10 with gateway 10.0.10.1, start IP 10.0.10.11, mask 255.255.255.0 and max users 100.
4. **Server1 HTTP:** Open Services, then HTTP, and turn it on.
5. **PC0, PC1, PC2, PC3, PC4 and PC5:** Open Desktop, then IP Configuration, and select DHCP. Wait about 30 seconds for the switch ports to forward, then renew if the address starts with 169.254.

## Step 4: Wireless Checklist

1. Click BR2-AP, open the Config tab, then Port 1.
2. Set SSID to `BR2-WiFi`.
3. Set Authentication to WPA2-PSK.
4. Set PSK Pass Phrase to `Branch@2026`.
5. Set Encryption Type to AES.
6. Keep the Port 0 cable on BR2-SW Fa0/2.
7. Click BR2-LAP1, open the Physical tab and switch the laptop off.
8. Replace the wired module with the wireless module PT-LAPTOP-NM-1W, then switch the laptop on.
9. Open Config, then Wireless0. Set the SSID, WPA2-PSK, the passphrase and AES, and select DHCP.
10. Repeat steps 7 to 9 for BR2-LAP2.
11. Keep both laptops inside the range of the access point.

## Step 5: Verification Commands

| Concept | Device | Command |
|---|---|---|
| OSPF neighbors | HQ-R1, HQ-DSW, HQ-R2, BR1-R, BR2-R | `show ip ospf neighbor` |
| OSPF routes | HQ-R1, BR1-R | `show ip route ospf` |
| OSPF interface | HQ-R1 | `show ip ospf interface gig0/1` |
| Summary route | HQ-DSW, HQ-R2 | `show ip route` |
| PAT | HQ-R2 | `show ip nat translations` and `show ip nat statistics` |
| VLANs | HQ-ASW, BR1-SW, HQ-DSW | `show vlan brief` |
| Trunks | HQ-ASW, BR1-SW | `show interfaces trunk` |
| EtherChannel | HQ-DSW, HQ-ASW | `show etherchannel summary` |
| Interfaces | HQ-DSW, BR1-R | `show ip interface brief` |
| DHCP | BR1-R, BR2-R | `show ip dhcp binding` |
| Host addresses | PC0, PC2, PC3, PC5, BR2-LAP1 | `ipconfig` |
| ACL counters | BR1-R | `enable`, then `show access-lists` |
| SSH | BR1-R | `show ip ssh` |

The `show access-lists` command only runs in privileged mode, so enter `enable` first.

## Step 6: Test Plan

**Internet and PAT**
1. From PC0, run `ping 198.51.100.10`.
2. On HQ-R2, run `show ip nat translations`.

**ACL on BR1 VLAN 10**
1. From PC2, open `http://10.0.20.10` in the Web Browser. The page should load.
2. From PC2, run `ping 10.0.20.10`. It should fail.
3. From PC3, run `ping 10.0.20.10`. It should work.
4. On BR1-R, run `enable` and then `show access-lists`.

**SSH and VTY access list**
1. From PC3, run `ssh -l admin 10.1.20.1`.
2. From Server0, run `ssh -l admin 198.51.100.1`. It should log in.
3. From Server0, run `ssh -l admin 203.0.113.1`. It should be refused.

**Wireless**
1. On BR2-LAP1, open Desktop, then Command Prompt.
2. Run `ipconfig` and check for a 10.1.30.x address.
3. Run `ping 10.1.30.1`.

**End to end**
1. From PC4, run `ping 10.0.10.11` and `ping 198.51.100.10`.
2. From PC2, run `ping 198.51.100.10`.

## Step 7: Redundancy Demo

**Before the failure**
1. On PC3, run `tracert 10.0.20.10`.

**Shut the link**
2. On HQ-R1, enter these commands:

```
enable
configure terminal
interface GigabitEthernet0/1
shutdown
end
```

3. On BR1-R, run `show ip ospf neighbor` and `show ip route ospf`.
4. On PC3, run `tracert 10.0.20.10` again. If the first attempt times out, wait a few seconds and run it again.

**Restore the link**
5. On HQ-R1, enter these commands:

```
configure terminal
interface GigabitEthernet0/1
no shutdown
end
```

After the restore, save the configuration on every device with `write memory` and proceed as described.
