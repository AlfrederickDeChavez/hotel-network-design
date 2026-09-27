# Hotel Network - Intermediate Capstone Lab
## Scenario

You are the network engineer for the **Grand Azure Hotel**, a 5-star hotel with guest rooms, front office, restaurant/POS, hotel staff, IP phones, employee Wi-Fi, guest Wi-Fi, IoT/cameras, PMS/ERP servers, and network-management systems.

The hotel requires:

- High availability at the core
- Redundant uplinks
- VLAN segmentation
- IPv4 and IPv6 dual stack
- Dynamic routing
- First-hop gateway redundancy
- Layer-2 security
- Guest isolation
- Protected POS/PMS networks
- Secure management access
- Internet/NAT simulation
- Monitoring and logging
- Deliberate failure and troubleshooting exercises

**Platform:** Cisco Packet Tracer

**Difficulty:** Intermediate CCNA

**Important:** This is a build-and-troubleshoot lab. Requirements are provided; final device configurations are intentionally not provided.

---

# 1. Learning Objectives

By completing this capstone you should be able to:

1. Design and implement a hierarchical enterprise network.
2. Create and troubleshoot VLANs and 802.1Q trunks.
3. Configure EtherChannel using LACP.
4. Implement Rapid PVST+ and deliberately select root bridges.
5. Configure inter-VLAN routing using SVIs.
6. Implement HSRP for IPv4 gateway redundancy.
7. Implement IPv6 first-hop redundancy where supported by the Packet Tracer image.
8. Configure OSPFv2 and OSPFv3.
9. Configure DHCP and DHCP relay.
10. Configure NAT/PAT and a default route.
11. Implement IPv4/IPv6 ACL-based segmentation.
12. Configure SSH management.
13. Configure port security, DHCP Snooping, DAI, PortFast and BPDU Guard where supported.
14. Use NTP, Syslog and SNMP conceptually/practically where Packet Tracer supports them.
15. Troubleshoot Layer 1, Layer 2, Layer 3 and security failures.
16. Document and verify an enterprise network.

---

# 2. Required Topology

## Logical topology

```text
                              INTERNET
                            /           \
                       ISP-1             ISP-2
                         |                 |
                         |                 |
                    +----+-----------------+----+
                    |       EDGE / FIREWALL      |
                    | NAT / ACL / VPN simulation |
                    +------------+---------------+
                                 |
                         +-------+-------+
                         |               |
                   +-----+-----+   +-----+-----+
                   |  CORE-01  |===|  CORE-02  |
                   |  PRIMARY  |===| SECONDARY |
                   +-----+-----+   +-----+-----+
                      /   |\          /   |\
                     /    | \        /    | \
                    /     |  \      /     |  \
                   /      |   \    /      |   \
             +----+-------+----+--+-------+----+
             |       DISTRIBUTION LAYER       |
             |                                |
             | DIST-01 = primary L2 path      |
             | DIST-02 = secondary L2 path    |
             +--+------+-----+-----+------+---+
                |      |     |     |      |
              SW01   SW02  SW03  SW04   SW05/SW06
               |      |     |     |       |
             Guest  Staff Voice POS      IoT/WiFi

                    SERVER FARM
          +----------------------------------+
          | DNS | WEB | MAIL | PMS | NTP     |
          | SYSLOG | MONITORING              |
          +----------------------------------+
```

## Physical/port topology

Use these port assignments as the target design. If your Packet Tracer model has different interface numbering, preserve the logical connection and document the substitution.

| Device | Interface | Connects to | Remote Interface | Link Type |
|---|---|---|---|---|
| CORE-01 | Gi1/0/1 | CORE-02 | Gi1/0/1 | EtherChannel member |
| CORE-01 | Gi1/0/2 | CORE-02 | Gi1/0/2 | EtherChannel member |
| CORE-01 | Gi1/0/3 | DIST-01 | Gi1/0/1 | Routed/L3 |
| CORE-01 | Gi1/0/4 | DIST-02 | Gi1/0/1 | Routed/L3 |
| CORE-02 | Gi1/0/3 | DIST-01 | Gi1/0/2 | Routed/L3 |
| CORE-02 | Gi1/0/4 | DIST-02 | Gi1/0/2 | Routed/L3 |
| DIST-01 | Gi1/0/3 | DIST-02 | Gi1/0/3 | Trunk/EtherChannel |
| DIST-01 | Gi1/0/4 | SW01 | Gi0/1 | Trunk |
| DIST-01 | Gi1/0/5 | SW02 | Gi0/1 | Trunk |
| DIST-01 | Gi1/0/6 | SW03 | Gi0/1 | Trunk |
| DIST-02 | Gi1/0/4 | SW04 | Gi0/1 | Trunk |
| DIST-02 | Gi1/0/5 | SW05 | Gi0/1 | Trunk |
| DIST-02 | Gi1/0/6 | SW06 | Gi0/1 | Trunk |
| EDGE | Gi0/0 | CORE-01 | Gi1/0/5 | L3 |
| EDGE | Gi0/1 | CORE-02 | Gi1/0/5 | L3 |
| EDGE | Gi0/2 | ISP-1 | Gi0/0 | WAN |
| EDGE | Gi0/3 | ISP-2 | Gi0/0 | WAN |

**If your Packet Tracer device lacks these exact interfaces, do not redesign the network. Substitute equivalent interfaces and record the change.**

---

# 3. Device Inventory

Recommended Packet Tracer inventory:

| Device | Quantity | Role |
|---|---:|---|
| Multilayer switch | 2 | Core |
| Multilayer switch | 2 | Distribution |
| Layer-2 switch | 6 | Access |
| Router | 1 | Edge/Internet |
| Router | 2 | ISP simulation |
| Server | 5–7 | Services |
| PCs/laptops | 10+ | Test clients |
| IP phones | 2+ | Voice |
| Wireless APs | 2+ | Guest/employee Wi-Fi |
| IoT/camera endpoints | 2+ | IoT |
| Firewall | Optional | Use if your Packet Tracer image supports the desired feature set |

**Note:** The primary CCNA version can simulate firewall functions using the Edge router with ACL/NAT. Do not let firewall hardware availability block the lab.

---

# 4. VLAN Plan

| VLAN | Name | Purpose | IPv4 Subnet | IPv6 Prefix | Virtual Gateway |
|---:|---|---|---|---|---|
| 10 | GUEST | Guest rooms/Wi-Fi | 192.168.10.0/24 | 2001:db8:10::/64 | 192.168.10.1 / 2001:db8:10::1 |
| 20 | STAFF | Hotel staff | 192.168.20.0/24 | 2001:db8:20::/64 | 192.168.20.1 / 2001:db8:20::1 |
| 30 | VOICE | IP phones | 192.168.30.0/24 | 2001:db8:30::/64 | 192.168.30.1 / 2001:db8:30::1 |
| 40 | POS | Restaurant/POS | 192.168.40.0/24 | 2001:db8:40::/64 | 192.168.40.1 / 2001:db8:40::1 |
| 50 | SERVER | General servers | 192.168.50.0/24 | 2001:db8:50::/64 | 192.168.50.1 / 2001:db8:50::1 |
| 60 | PMS | PMS/ERP | 192.168.60.0/24 | 2001:db8:60::/64 | 192.168.60.1 / 2001:db8:60::1 |
| 70 | MONITOR | Monitoring/logging | 192.168.70.0/24 | 2001:db8:70::/64 | 192.168.70.1 / 2001:db8:70::1 |
| 80 | IOT | Cameras/IoT | 192.168.80.0/24 | 2001:db8:80::/64 | 192.168.80.1 / 2001:db8:80::1 |
| 90 | MGMT | Network management | 192.168.90.0/24 | 2001:db8:90::/64 | 192.168.90.1 / 2001:db8:90::1 |
| 100 | EMP-WIFI | Employee Wi-Fi | 192.168.100.0/24 | 2001:db8:100::/64 | 192.168.100.1 / 2001:db8:100::1 |
| 999 | BLACKHOLE | Unused/native | No client subnet | None | None |

---

# 5. HSRP Design

Use HSRP for IPv4 gateway redundancy.

The virtual gateway is always `.1`.

| VLAN | CORE-01 | CORE-02 | Active |
|---:|---|---|---|
| 10 | 192.168.10.2 | 192.168.10.3 | CORE-01 |
| 20 | 192.168.20.2 | 192.168.20.3 | CORE-02 |
| 30 | 192.168.30.2 | 192.168.30.3 | CORE-01 |
| 40 | 192.168.40.2 | 192.168.40.3 | CORE-02 |
| 50 | 192.168.50.2 | 192.168.50.3 | CORE-01 |
| 60 | 192.168.60.2 | 192.168.60.3 | CORE-02 |
| 70 | 192.168.70.2 | 192.168.70.3 | CORE-01 |
| 80 | 192.168.80.2 | 192.168.80.3 | CORE-02 |
| 90 | 192.168.90.2 | 192.168.90.3 | CORE-01 |
| 100 | 192.168.100.2 | 192.168.100.3 | CORE-02 |

### HSRP requirement

- Configure HSRP on every client VLAN.
- Use priorities so the table above is respected.
- Enable preemption.
- Test failover by disabling the active core's relevant SVI or uplink.
- Verify the standby core takes over.

---

# 6. STP Design

Protocol:

**Rapid PVST+**

Root bridge distribution:

| VLANs | Primary Root | Secondary Root |
|---|---|---|
| 10, 30, 50, 70, 90 | CORE-01 | CORE-02 |
| 20, 40, 60, 80, 100 | CORE-02 | CORE-01 |

Requirements:

- Configure explicit root primary/secondary behavior.
- Do not rely on default switch priorities.
- Access ports must use PortFast where appropriate.
- Enable BPDU Guard on edge ports where supported.
- Do not enable PortFast on switch-to-switch links.
- Verify root bridge selection with `show spanning-tree`.

---

# 7. EtherChannel Design

Use:

**LACP**

CORE-01 ↔ CORE-02:

- Gi1/0/1
- Gi1/0/2
- Port-channel 1

DIST-01 ↔ DIST-02:

- Gi1/0/3
- Gi1/0/4 if available
- Port-channel 2

Requirements:

- Use LACP, not static EtherChannel.
- Ensure all member interfaces have compatible settings.
- Configure the logical Port-channel consistently.
- Use trunking where the bundle carries VLANs.
- Verify using EtherChannel summary commands.

---

# 8. Layer-3 Routed Links

Use point-to-point /30 IPv4 networks.

| Link | IPv4 Network | Side A | Side B |
|---|---|---|---|
| CORE-01 ↔ DIST-01 | 10.255.1.0/30 | 10.255.1.1 | 10.255.1.2 |
| CORE-01 ↔ DIST-02 | 10.255.2.0/30 | 10.255.2.1 | 10.255.2.2 |
| CORE-02 ↔ DIST-01 | 10.255.3.0/30 | 10.255.3.1 | 10.255.3.2 |
| CORE-02 ↔ DIST-02 | 10.255.4.0/30 | 10.255.4.1 | 10.255.4.2 |
| EDGE ↔ CORE-01 | 10.255.10.0/30 | 10.255.10.1 | 10.255.10.2 |
| EDGE ↔ CORE-02 | 10.255.11.0/30 | 10.255.11.1 | 10.255.11.2 |

IPv6 point-to-point prefixes:

| Link | IPv6 Prefix |
|---|---|
| CORE-01 ↔ DIST-01 | 2001:db8:ff:1::/64 |
| CORE-01 ↔ DIST-02 | 2001:db8:ff:2::/64 |
| CORE-02 ↔ DIST-01 | 2001:db8:ff:3::/64 |
| CORE-02 ↔ DIST-02 | 2001:db8:ff:4::/64 |
| EDGE ↔ CORE-01 | 2001:db8:ff:10::/64 |
| EDGE ↔ CORE-02 | 2001:db8:ff:11::/64 |

---

# 9. OSPF Design

## IPv4

Protocol:

**OSPFv2**

Process ID:

**10**

Area:

**0**

Requirements:

- Use explicit router IDs.
- Establish OSPF neighbors over routed links.
- Advertise all internal IPv4 networks.
- Use passive interfaces toward user VLANs.
- Originate or propagate the default route from the Edge where appropriate.
- Verify neighbors and the routing table.
- Observe alternate paths when a routed link fails.

Suggested router IDs:

| Device | Router ID |
|---|---|
| CORE-01 | 1.1.1.1 |
| CORE-02 | 2.2.2.2 |
| DIST-01 | 3.3.3.3 |
| DIST-02 | 4.4.4.4 |
| EDGE | 5.5.5.5 |

## IPv6

Protocol:

**OSPFv3**

Use the same logical area:

**Area 0**

Requirements:

- Enable IPv6 routing.
- Establish OSPFv3 neighbors.
- Advertise VLAN and point-to-point IPv6 networks.
- Verify IPv6 neighbors.
- Verify OSPF-learned IPv6 routes.
- Test end-to-end IPv6 connectivity.

---

# 10. IPv6 Addressing

Use these /64 prefixes for VLANs:

```text
VLAN 10  2001:db8:10::/64
VLAN 20  2001:db8:20::/64
VLAN 30  2001:db8:30::/64
VLAN 40  2001:db8:40::/64
VLAN 50  2001:db8:50::/64
VLAN 60  2001:db8:60::/64
VLAN 70  2001:db8:70::/64
VLAN 80  2001:db8:80::/64
VLAN 90  2001:db8:90::/64
VLAN 100 2001:db8:100::/64
```

Use:

```text
::1 = virtual gateway
::2 = CORE-01
::3 = CORE-02
```

where the address is used on the relevant SVI.

---

# 11. Server Addressing

| Server | VLAN | IPv4 | IPv6 | Function |
|---|---:|---|---|---|
| SRV-DNS | 50 | 192.168.50.10 | 2001:db8:50::10 | DNS |
| SRV-WEB | 50 | 192.168.50.20 | 2001:db8:50::20 | Hotel website |
| SRV-MAIL | 50 | 192.168.50.30 | 2001:db8:50::30 | Mail simulation |
| SRV-PMS | 60 | 192.168.60.10 | 2001:db8:60::10 | PMS/ERP |
| SRV-MON | 70 | 192.168.70.10 | 2001:db8:70::10 | Monitoring |
| SRV-NTP | 70 | 192.168.70.20 | 2001:db8:70::20 | NTP |
| SRV-SYSLOG | 70 | 192.168.70.30 | 2001:db8:70::30 | Syslog |

Use static addresses for servers.

---

# 12. Management Addressing

Management VLAN:

**VLAN 90 — 192.168.90.0/24**

Suggested addresses:

| Device | IPv4 |
|---|---|
| CORE-01 | 192.168.90.2 |
| CORE-02 | 192.168.90.3 |
| DIST-01 | 192.168.90.11 |
| DIST-02 | 192.168.90.12 |
| SW01 | 192.168.90.21 |
| SW02 | 192.168.90.22 |
| SW03 | 192.168.90.23 |
| SW04 | 192.168.90.24 |
| SW05 | 192.168.90.25 |
| SW06 | 192.168.90.26 |

Management requirements:

- SSH only.
- Disable Telnet.
- Use a local administrative account.
- Use an appropriate domain name.
- Generate RSA keys.
- Restrict VTY access to the management network.
- Use encrypted/hashed credentials where supported.

---

# 13. DHCP Design

Use DHCP for:

- VLAN 10 Guest
- VLAN 20 Staff
- VLAN 30 Voice
- VLAN 40 POS where appropriate
- VLAN 80 IoT
- VLAN 100 Employee Wi-Fi

Reserve:

```text
.1 - .99
```

for infrastructure/static addresses.

Suggested DHCP client range:

```text
.100 - .200
```

Example:

```text
VLAN 20
Network: 192.168.20.0/24
Gateway: 192.168.20.1
DHCP: 192.168.20.100–192.168.20.200
DNS: 192.168.50.10
```

If the DHCP server is centralized, use DHCP relay on the SVIs.

---

# 14. Guest Security Policy

Guest devices are untrusted.

Required behavior:

| Traffic | Result |
|---|---|
| Guest → Internet | ALLOW |
| Guest → DNS | ALLOW |
| Guest → Hotel Web | ALLOW |
| Guest → PMS | DENY |
| Guest → POS | DENY |
| Guest → Staff | DENY |
| Guest → Management | DENY |
| Guest → Server Farm | DENY except explicitly permitted services |
| Guest → Other Guest | Prefer isolation where supported |

Document your ACL logic before implementing it.

---

# 15. POS Security Policy

POS VLAN 40 is sensitive.

Allow only required traffic, such as:

```text
POS → DNS
POS → NTP
POS → PMS
POS → required Internet services
```

Deny:

```text
POS → Guest
POS → Employee Wi-Fi
POS → Network Management
POS → unrelated server networks
```

Do not use a broad `permit ip any any` after the restrictive rules unless you have a specific documented reason.

---

# 16. IoT Security Policy

VLAN 80 contains cameras/IoT.

Required concept:

```text
IoT → Monitoring      ALLOW
IoT → NTP             ALLOW
IoT → DNS             ALLOW
IoT → Internet        RESTRICT
IoT → Staff           DENY
IoT → POS             DENY
IoT → Management      DENY
IoT → PMS             DENY
```

---

# 17. Layer-2 Security Requirements

On access ports:

### Port Security

Implement:

- Maximum MAC addresses
- Sticky MAC
- Appropriate violation mode
- Verify learned secure MAC addresses

### DHCP Snooping

- Enable on client VLANs.
- Trust only the legitimate DHCP-facing interface/uplink.
- Verify the binding table.

### Dynamic ARP Inspection

- Enable for appropriate VLANs.
- Ensure legitimate DHCP bindings are available.

### IP Source Guard

Enable where supported and appropriate.

### Edge protection

Configure:

- PortFast
- BPDU Guard

on true endpoint ports.

### Unused ports

Move unused access ports to VLAN 999 and shut them down.

---

# 18. Trunk Security

All switch-to-switch trunks must:

- Use 802.1Q.
- Use native VLAN 999.
- Explicitly allow only required VLANs.
- Avoid unnecessary VLANs.
- Be documented.

Example allowed VLAN concept:

```text
10,20,30,40,50,60,70,80,90,100,999
```

Do not blindly allow every VLAN.

---

# 19. Voice Network

Use:

**VLAN 30 — VOICE**

Example access port requirement:

```text
PC → Data VLAN 20
IP Phone → Voice VLAN 30
```

Configure an appropriate voice VLAN on ports that connect to phones.

Verify that:

- PC traffic remains in VLAN 20.
- Phone traffic uses VLAN 30.
- The voice network receives the appropriate addressing.

---

# 20. WAN and NAT

Use:

**203.0.113.0/24** for simulated public addressing.

Example:

| Device | Address |
|---|---|
| ISP-1 | 203.0.113.1 |
| EDGE ISP-1 | 203.0.113.2 |
| ISP-2 | 203.0.114.1 |
| EDGE ISP-2 | 203.0.114.2 |

Use NAT/PAT for inside private networks.

Requirements:

- Configure inside/outside interfaces.
- Configure PAT for internal users.
- Configure a default route toward the primary ISP.
- Create a backup/default path toward ISP-2 as a redundancy exercise.
- Verify translated sessions.

---

# 21. Administrative Services

Configure where Packet Tracer supports them:

### SSH

All network devices:

- SSH version 2
- Local authentication
- Management VLAN restriction

### NTP

Point network devices toward:

```text
192.168.70.20
```

### Syslog

Point devices toward:

```text
192.168.70.30
```

### SNMP

Configure a read-only monitoring community for the lab if supported.

### DNS

Use:

```text
192.168.50.10
```

for internal name resolution.

---

# 22. Configuration Tasks

Complete these in order.

## Phase 1 — Build

- Place all devices.
- Connect all links.
- Label every link.
- Record actual Packet Tracer interface substitutions.

## Phase 2 — Basic device configuration

Configure:

- Hostnames
- Enable secret
- Local administrator
- Console protection
- SSH
- Domain name
- RSA keys
- Management SVI
- Disable unused interfaces

## Phase 3 — VLANs

Create:

```text
10 GUEST
20 STAFF
30 VOICE
40 POS
50 SERVER
60 PMS
70 MONITOR
80 IOT
90 MGMT
100 EMP-WIFI
999 BLACKHOLE
```

## Phase 4 — Access ports

Assign endpoint ports to the correct VLANs.

## Phase 5 — Trunks

Configure and verify all switch-to-switch trunks.

## Phase 6 — EtherChannel

Build the two planned LACP bundles.

## Phase 7 — STP

Configure Rapid PVST+ and verify the planned roots.

## Phase 8 — Inter-VLAN routing

Configure SVIs on the core.

## Phase 9 — HSRP

Configure gateway redundancy.

## Phase 10 — IPv4 OSPF

Build the OSPFv2 topology.

## Phase 11 — IPv6

Enable IPv6 routing and OSPFv3.

## Phase 12 — DHCP

Configure DHCP and relay.

## Phase 13 — ACLs

Implement guest/POS/IoT restrictions.

## Phase 14 — Layer-2 security

Implement:

- Port Security
- DHCP Snooping
- DAI
- IP Source Guard
- PortFast
- BPDU Guard

where supported.

## Phase 15 — NAT/WAN

Configure Internet simulation and PAT.

## Phase 16 — Management

Configure:

- SSH
- NTP
- Syslog
- SNMP

## Phase 17 — Validation

Run all tests in the verification matrix.

---

# 23. Verification Matrix

Do not consider the lab complete until you can demonstrate each item.

| Test | Expected Result |
|---|---|
| PC → local gateway IPv4 | Success |
| PC → local gateway IPv6 | Success |
| Staff → PMS | Success |
| POS → PMS | Success |
| Guest → PMS | Blocked |
| Guest → POS | Blocked |
| Guest → Internet | Success |
| IoT → Monitoring | Success |
| Unauthorized VLAN traffic | Blocked |
| SSH from MGMT | Success |
| SSH from Guest | Blocked |
| DHCP client | Receives correct subnet |
| DNS lookup | Success |
| NTP synchronization | Success |
| Syslog messages | Received |
| OSPF neighbor | Established |
| OSPFv3 neighbor | Established |
| HSRP state | Correct active/standby |
| STP root | Matches design |
| EtherChannel | Bundled |
| NAT/PAT | Translations appear |
| IPv4 end-to-end | Success |
| IPv6 end-to-end | Success |

---

# 24. Failure Scenarios

After the network works, deliberately introduce these failures.

## Failure 1 — Wrong VLAN

Move a Staff PC into the Guest VLAN.

**Task:** Diagnose why it receives the wrong network.

## Failure 2 — Trunk VLAN missing

Remove VLAN 40 from an uplink.

**Task:** Determine why POS traffic fails across the distribution layer.

## Failure 3 — Native VLAN mismatch

Create a native VLAN mismatch.

**Task:** Identify the warning and correct the configuration.

## Failure 4 — EtherChannel mismatch

Change one member's trunk configuration.

**Task:** Diagnose the bundle.

## Failure 5 — Wrong STP root

Change bridge priority.

**Task:** Identify the unexpected root bridge and restore the design.

## Failure 6 — HSRP failure

Disable the active core SVI.

**Task:** Confirm gateway failover.

## Failure 7 — OSPF failure

Shut one routed link.

**Task:** Verify that OSPF chooses the alternate path.

## Failure 8 — DHCP failure

Remove/disable the DHCP relay configuration.

**Task:** Determine why clients stop receiving addresses.

## Failure 9 — ACL failure

Add an incorrect ACL rule that blocks legitimate Staff → PMS traffic.

**Task:** Locate and fix the rule.

## Failure 10 — Port security

Connect a different endpoint to a secured port.

**Task:** Determine the violation and recover the port.

## Failure 11 — IPv6 routing

Disable OSPFv3 on one routed interface.

**Task:** Explain why IPv4 continues working while IPv6 fails.

## Failure 12 — NAT failure

Break the inside/outside classification.

**Task:** Diagnose why internal clients cannot reach the simulated Internet.

---

# 25. Required Show/Verification Commands

Build your own command checklist, but you should eventually be comfortable using commands such as:

```text
show vlan brief
show interfaces trunk
show etherchannel summary
show spanning-tree
show spanning-tree vlan 10
show spanning-tree vlan 20

show ip interface brief
show ip route
show ip ospf neighbor
show ip ospf interface
show ip protocols

show standby
show standby brief

show ipv6 interface brief
show ipv6 route
show ipv6 ospf neighbor

show access-lists
show ip interface

show port-security
show port-security interface
show ip dhcp snooping
show ip dhcp snooping binding
show ip arp inspection

show cdp neighbors
show lldp neighbors

show ip nat translations
show ip nat statistics

show logging
show ntp status
show users
show ssh
```

Use the commands to **prove** the network works rather than merely assuming it works.

---

# 26. Acceptance Criteria

The capstone is complete when:

### Layer 2

- All VLANs exist.
- Trunks work.
- Native VLAN is consistent.
- EtherChannel is operational.
- STP roots match the design.
- Endpoint protection is configured.

### Layer 3

- All SVIs are operational.
- HSRP works.
- OSPFv2 converges.
- OSPFv3 converges.
- IPv4 works end-to-end.
- IPv6 works end-to-end.

### Security

- Guest is isolated.
- POS is restricted.
- IoT is restricted.
- Management access is protected.
- Port security works.
- DHCP Snooping/DAI work where supported.
- BPDU Guard works on edge ports.

### Services

- DHCP works.
- DNS works.
- NTP works.
- Syslog works.
- SSH works.
- NAT/PAT works.

### Resiliency

- Core failure does not completely isolate users.
- HSRP fails over.
- OSPF finds alternate routes.
- STP reconverges.
- EtherChannel survives an individual member-link failure.

---

# 27. Deliverables

Submit a small network-engineering portfolio package:

```text
Grand-Azure-Hotel/
│
├── topology.pkt
│
├── README.md
│
├── addressing-plan.md
│
├── security-policy.md
│
├── troubleshooting.md
│
├── configs/
│   ├── CORE-01.txt
│   ├── CORE-02.txt
│   ├── DIST-01.txt
│   ├── DIST-02.txt
│   ├── SW01.txt
│   ├── SW02.txt
│   ├── SW03.txt
│   ├── SW04.txt
│   ├── SW05.txt
│   ├── SW06.txt
│   └── EDGE.txt
│
└── screenshots/
    ├── ospf.png
    ├── ospfv3.png
    ├── hsrp.png
    ├── spanning-tree.png
    ├── etherchannel.png
    ├── acl.png
    ├── dhcp-snooping.png
    └── ipv6-connectivity.png
```

---

# 28. Final Challenge

Once everything works, answer these questions without looking at a configuration:

1. Why does the hotel need both HSRP and STP?
2. Why should HSRP Active and STP Root generally be aligned?
3. What happens if CORE-01 fails?
4. What happens if only one EtherChannel member fails?
5. What happens if a distribution uplink fails?
6. Why is Guest VLAN unable to access PMS?
7. Why does DHCP Snooping need trusted interfaces?
8. How does DAI use DHCP Snooping information?
9. Why should the native VLAN not be a production user VLAN?
10. Why is OSPF useful here instead of a collection of static routes?
11. Why can IPv4 work while IPv6 fails?
12. What is the difference between a Layer-2 failure and a Layer-3 failure?
13. Why is the management VLAN separated from Staff?
14. Why should POS be isolated from Guest?
15. What happens to the default gateway when HSRP fails over?
16. What happens to an OSPF route when a link goes down?
17. Why does EtherChannel not simply mean "two independent links"?
18. Why should access ports use BPDU Guard?
19. What is the security difference between Port Security and DHCP Snooping?
20. Why should ACLs be designed around business requirements rather than arbitrary deny rules?

---

# 29. Recommended Build Order Summary

```text
PHYSICAL
   ↓
HOSTNAMES / MANAGEMENT
   ↓
VLANs
   ↓
ACCESS PORTS
   ↓
TRUNKS
   ↓
LACP
   ↓
RAPID PVST+
   ↓
SVIs
   ↓
HSRP
   ↓
IPv4 OSPF
   ↓
IPv6 + OSPFv3
   ↓
DHCP / RELAY
   ↓
ACLs
   ↓
L2 SECURITY
   ↓
NAT/PAT
   ↓
SSH / NTP / SYSLOG / SNMP
   ↓
END-TO-END TESTING
   ↓
FAILURE INJECTION
   ↓
TROUBLESHOOTING
   ↓
DOCUMENTATION
```

# 30. Engineer's Rule

Do not paste a complete configuration from the Internet or from an AI.

For every configuration you add, be able to explain:

**What problem does this command solve?**

**What would break if I removed it?**

**How can I verify that it actually works?**

That is the standard this capstone is designed around.
