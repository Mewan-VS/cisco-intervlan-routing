# Design and Configure a Secure Office Network Using VLANs


## 1. Topology

```
                     [R1 - Cisco 2911]
                        G0/0  (802.1Q trunk, subifs .10 .20 .30)
                         |
                        G0/1
   HR-PC1  ---Fa0/1  [SW1 - 2960]  G0/2 ======= trunk ======= G0/1  [SW2 - 2960]  Fa0/1--- HR-PC2
   IT-PC1  ---Fa0/2                                                               Fa0/2--- IT-PC2
   FIN-PC1 ---Fa0/3                                                               Fa0/3--- FIN-PC2
                                                                                  Fa0/4--- IT-SERVER (bonus)
```

Each department has one PC on each switch, as the brief requires.

### Cabling

| From | Port | To | Port | Cable | Mode |
|---|---|---|---|---|---|
| R1 | G0/0 | SW1 | G0/1 | Copper straight-through | Trunk |
| SW1 | G0/2 | SW2 | G0/1 | Copper cross-over | Trunk |
| HR-PC1 | Fa0 | SW1 | Fa0/1 | Straight-through | Access VLAN 10 |
| IT-PC1 | Fa0 | SW1 | Fa0/2 | Straight-through | Access VLAN 20 |
| FIN-PC1 | Fa0 | SW1 | Fa0/3 | Straight-through | Access VLAN 30 |
| HR-PC2 | Fa0 | SW2 | Fa0/1 | Straight-through | Access VLAN 10 |
| IT-PC2 | Fa0 | SW2 | Fa0/2 | Straight-through | Access VLAN 20 |
| FIN-PC2 | Fa0 | SW2 | Fa0/3 | Straight-through | Access VLAN 30 |
| IT-SERVER | Fa0 | SW2 | Fa0/4 | Straight-through | Access VLAN 20 |

## 2. VLAN Plan

| VLAN ID | Name | Department | Network | Gateway |
|---|---|---|---|---|
| 10 | HR | Human Resources | 192.168.10.0/24 | 192.168.10.1 (G0/0.10) |
| 20 | IT | Information Technology | 192.168.20.0/24 | 192.168.20.1 (G0/0.20) |
| 30 | FINANCE | Finance | 192.168.30.0/24 | 192.168.30.1 (G0/0.30) |
| 99 | NATIVE | Unused native VLAN for trunks (security) | n/a | n/a |
| 999 | PARKING | Unused, shut-down ports (security) | n/a | n/a |

## 3. IP Addressing

| Device | VLAN | IP Address | Mask | Gateway |
|---|---|---|---|---|
| HR-PC1 | 10 | 192.168.10.11 | 255.255.255.0 | 192.168.10.1 |
| HR-PC2 | 10 | 192.168.10.12 | 255.255.255.0 | 192.168.10.1 |
| IT-PC1 | 20 | 192.168.20.11 | 255.255.255.0 | 192.168.20.1 |
| IT-PC2 | 20 | 192.168.20.12 | 255.255.255.0 | 192.168.20.1 |
| FIN-PC1 | 30 | 192.168.30.11 | 255.255.255.0 | 192.168.30.1 |
| FIN-PC2 | 30 | 192.168.30.12 | 255.255.255.0 | 192.168.30.1 |
| IT-SERVER (bonus) | 20 | 192.168.20.100 | 255.255.255.0 | 192.168.20.1 |

Set these under PC → Desktop → IP Configuration (Static).

## 4. Configuration

The full scripts are in `configs/R1.txt`, `configs/SW1.txt` and `configs/SW2.txt`. Paste each one into the device's CLI tab.

### Router-on-a-Stick (R1)
```
interface GigabitEthernet0/0
 no shutdown
interface GigabitEthernet0/0.10
 encapsulation dot1Q 10
 ip address 192.168.10.1 255.255.255.0
interface GigabitEthernet0/0.20
 encapsulation dot1Q 20
 ip address 192.168.20.1 255.255.255.0
interface GigabitEthernet0/0.30
 encapsulation dot1Q 30
 ip address 192.168.30.1 255.255.255.0
```

### Switch VLANs, access ports and trunks (SW1; SW2 is the same pattern)
```
vlan 10 / name HR      vlan 20 / name IT      vlan 30 / name FINANCE
interface Fa0/1 -> switchport mode access ; switchport access vlan 10
interface Fa0/2 -> switchport mode access ; switchport access vlan 20
interface Fa0/3 -> switchport mode access ; switchport access vlan 30
interface G0/1, G0/2 -> switchport mode trunk
                        switchport trunk native vlan 99
                        switchport trunk allowed vlan 10,20,30,99
                        switchport nonegotiate
```

### Security hardening (the "Secure" in the title)
- **Native VLAN changed from 1 to 99** on both ends of every switch trunk. This mitigates VLAN double-tagging (VLAN hopping) attacks.
- **DTP disabled** (`switchport nonegotiate`) and ports hard-set to access or trunk. A rogue device can't negotiate a trunk.
- **Allowed-VLAN list** on trunks. Only VLANs 10, 20, 30 and 99 cross the links.
- **Unused ports** moved to VLAN 999 and `shutdown`.
- **PortFast** enabled on access ports only.
- `enable secret` and a console password are set, plus an MOTD banner.

## 5. Verification Commands – What Each Shows

| Device | Command | Information provided |
|---|---|---|
| Switch | `show vlan brief` | Every VLAN ID, name and status, plus the access ports assigned to each. Trunk ports are **not** listed. |
| Switch | `show interfaces trunk` | Trunking ports, mode, encapsulation (802.1q), native VLAN, allowed VLANs, and VLANs active and forwarding (STP) on each trunk. |
| Switch | `show interfaces status` | Each port's connected/notconnect/disabled state, VLAN (or "trunk"), duplex, speed and type. Good for spotting down or misassigned ports quickly. |
| Switch | `show mac address-table` | Learned MAC → VLAN → port mappings. Shows which hosts the switch learned on which port. The router's MAC appears on G0/1 in each VLAN. |
| Switch/Router | `show running-config` | The full active configuration in RAM: VLAN assignments, trunk settings, subinterfaces, passwords. |
| Router | `show ip interface brief` | Each interface and subinterface with its IP, status (up/down) and protocol state. Confirms G0/0 and G0/0.10/.20/.30 are up/up. |
| Router | `show ip route` | The routing table. Should show **C** (connected) and **L** (local) routes for 192.168.10.0/24, 20.0/24 and 30.0/24 via G0/0.10/.20/.30. |
| Router | `show interfaces` | Detailed layer 1/2 info per interface: status, MAC (BIA), MTU, bandwidth, encapsulation (802.1Q VLAN ID on subifs), and input/output and error counters. |

**Expected key outputs**

```
SW1# show vlan brief
VLAN Name        Status    Ports
1    default     active
10   HR          active    Fa0/1
20   IT          active    Fa0/2
30   FINANCE     active    Fa0/3
99   NATIVE      active
999  PARKING     active    Fa0/4 ... Fa0/24

SW1# show interfaces trunk
Port   Mode  Encapsulation  Status    Native vlan
Gig0/1 on    802.1q         trunking  99
Gig0/2 on    802.1q         trunking  99
Port   Vlans allowed on trunk
Gig0/1 10,20,30,99
Gig0/2 10,20,30,99

R1# show ip interface brief
GigabitEthernet0/0     unassigned    YES unset  up  up
GigabitEthernet0/0.10  192.168.10.1  YES manual up  up
GigabitEthernet0/0.20  192.168.20.1  YES manual up  up
GigabitEthernet0/0.30  192.168.30.1  YES manual up  up

R1# show ip route
C 192.168.10.0/24 is directly connected, GigabitEthernet0/0.10
L 192.168.10.1/32 is directly connected, GigabitEthernet0/0.10
C 192.168.20.0/24 is directly connected, GigabitEthernet0/0.20
L 192.168.20.1/32 is directly connected, GigabitEthernet0/0.20
C 192.168.30.0/24 is directly connected, GigabitEthernet0/0.30
L 192.168.30.1/32 is directly connected, GigabitEthernet0/0.30
```
*(Replace these with your own screenshots.)*

## 6. Testing Results

> The first ping often shows 1 timeout (`Request timed out`) while ARP resolves. That's normal. Run it again and you should get 4/4.

### Test A – Same-VLAN
| Source | Destination | Result |
|---|---|---|
| HR-PC1 | 192.168.10.12 (HR-PC2) | Success |
| IT-PC1 | 192.168.20.12 (IT-PC2) | Success |
| FIN-PC1 | 192.168.30.12 (FIN-PC2) | Success |

The two PCs in each VLAN sit on different switches, so their frames cross the SW1–SW2 trunk tagged with their VLAN ID. The router isn't involved.

### Test B – Inter-VLAN
| Source | Destination | Result |
|---|---|---|
| HR-PC1 | 192.168.20.11 (IT-PC1) | Success |
| HR-PC1 | 192.168.30.11 (FIN-PC1) | Success |
| FIN-PC1 | 192.168.10.11 (HR-PC1) | Success |

### Test C – VLAN Isolation (R1 G0/0 shut down, or SW1 G0/1 shut down)
1. **Can two PCs in the same VLAN still communicate?** Yes. Same-VLAN traffic is switched at layer 2 and crosses the SW1–SW2 trunk. It never needs the router.
2. **Can a PC in VLAN 10 reach a PC in VLAN 20?** No. The pings fail with "Request timed out" or "Destination host unreachable".
3. **Why?** Each VLAN is a separate broadcast domain and IP subnet. A PC that sends to another subnet must forward the frame to its default gateway, which is the router subinterface. With the router link down, nothing can do the layer-3 forwarding between VLANs, and the switches won't move frames between VLANs themselves. Traffic inside one subnet goes directly to the destination's MAC (learned via ARP), so it only needs the switches.

Restore with `no shutdown` afterwards.

### Test D – Simulation Mode (ping HR-PC1 → FIN-PC1)
1. **How the PC finds the gateway MAC:** HR-PC1 compares 192.168.30.11 with its own IP and mask and sees the destination is on a different subnet. It needs the MAC of its gateway, 192.168.10.1. If that isn't in its ARP cache, it broadcasts an **ARP request** ("Who has 192.168.10.1?") in VLAN 10. R1's G0/0.10 replies with a unicast **ARP reply** carrying its MAC, and the PC caches it.
2. **How the frame travels from the access switch to the router:** HR-PC1 sends an untagged frame to SW1 Fa0/1 (access, VLAN 10). SW1 looks up the destination MAC (the router's), which it learned on G0/1. It adds an **802.1Q tag with VLAN ID 10** and forwards the frame out the G0/1 trunk. R1 receives it on G0/0, reads tag 10 and hands it to subinterface **G0/0.10**.
3. **How the router picks the outgoing interface:** R1 removes the layer-2 header and looks up destination IP 192.168.30.11 in its routing table. The longest match is **192.168.30.0/24, directly connected via G0/0.30**. R1 ARPs for 192.168.30.11 if needed, then sends the frame out G0/0.30 tagged **VLAN 30**, back down the same physical link. SW1 forwards it across the trunk to SW2 if needed, and SW2 removes the tag and delivers it out the VLAN 30 access port to FIN-PC1.
4. **What happens to the MAC addresses:** they are **rewritten at every layer-3 hop**. On the HR side the frame is Src = HR-PC1 MAC, Dst = R1 G0/0 MAC. After routing it is Src = R1 G0/0 MAC, Dst = FIN-PC1 MAC. (All subinterfaces share G0/0's MAC. Only the VLAN tag changes, from 10 to 30.)
5. **Do the IP addresses change?** No. Without NAT, the source IP (192.168.10.11) and destination IP (192.168.30.11) stay the same end to end. Only the TTL drops by 1 and the header checksum is recalculated.

## 7. Troubleshooting Report

| # | Fault (symptom) | Cause | Diagnosis | Solution |
|---|---|---|---|---|
| 1 | Inter-VLAN pings fail. Same-VLAN pings work. | R1 G0/0 was left administratively down (router interfaces are shut by default). | `show ip interface brief` showed G0/0 and its subifs as *administratively down*. | `interface g0/0` → `no shutdown` |
| 2 | Same-VLAN pings fail between the two switches. | The SW1–SW2 link was still in access mode, VLAN 1, so only VLAN 1 crossed it. | `show interfaces trunk` didn't list the port. `show vlan brief` showed G0/2 in VLAN 1. | `switchport mode trunk` on both ends |
| 3 | Inter-VLAN fails from one VLAN only. | Wrong VLAN on a subinterface (for example, `encapsulation dot1Q 20` on G0/0.30). | `show running-config` / `show interfaces g0/0.30` showed the wrong VLAN ID. | Fix to `encapsulation dot1Q 30` |
| 4 | A PC can't ping its own gateway. | Wrong default gateway or mask on the PC. | PC `ipconfig` showed gateway 192.168.1.1. | Correct it under IP Configuration |
| 5 | A PC can't reach anything in its own department. | Access port assigned to the wrong VLAN. | `show vlan brief` showed Fa0/3 in VLAN 20. | `switchport access vlan 30` |
| 6 | CDP "Native VLAN mismatch" console messages. | Native VLAN 99 set on one trunk end only. | `show interfaces trunk` showed native 99 vs 1. | Set `switchport trunk native vlan 99` on both ends |
| 7 | A VLAN missing from the trunk. | VLAN not created on one switch, or not in the allowed list. | `show interfaces trunk` → "VLANs allowed and active" was missing it. | Create the VLAN / `switchport trunk allowed vlan add X` |
| 8 | First ping shows 1 timeout. | ARP resolution delay. This isn't a fault. | n/a | Expected. Repeat the ping. |

*(Add your own faults if your instructor injected specific ones.)*

## 8. Conclusion

VLANs split one physical switched network into separate logical broadcast domains: HR, IT and Finance. That gives:
- **Organization:** users are grouped by function, not location. HR-PC1 and HR-PC2 share a subnet even though they're on different switches.
- **Less broadcast traffic:** ARP and other broadcasts stay inside their VLAN.
- **Isolation and security:** departments can't reach each other at layer 2. All inter-department traffic has to pass through the router, where ACLs could be applied (for example, to block HR from Finance). Hardening such as a non-default native VLAN, DTP disabled and unused ports shut reduces the attack surface further.
- **Scalability:** Router-on-a-Stick routes between all VLANs over one physical interface using 802.1Q subinterfaces, and trunks carry every VLAN over a single link between switches.

---

## Appendix – Build Steps in Packet Tracer (about 10 minutes)
1. Place a **2911** router, two **2960-24TT** switches, six **PC-PT** and optionally one **Server-PT**. Rename them (R1, SW1, SW2, HR-PC1, ...).
2. Cable them as in the cabling table in section 1.
3. Paste `configs/R1.txt`, `configs/SW1.txt` and `configs/SW2.txt` into each device's CLI tab, after `enable` and `configure terminal` (the scripts end with `end` / `write memory`).
4. Set the static IPs on each PC/server (section 3).
5. Wait for the links to turn green (about 30 s for STP). Run Tests A–D and take screenshots.
6. Save as `StudentName_VLAN_Routing_Project.pkt`.
