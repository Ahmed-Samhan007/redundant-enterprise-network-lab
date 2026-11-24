# Redundant Enterprise Network Lab (Cisco Packet Tracer)

A small company with three departments (Accounting, Sales, and HR) needs a redundant and segmented network. I designed, configured, and verified this network in Cisco Packet Tracer.

![Network topology](images/topology.png)

## Overview
- **Goal:** build a redundant network with one VLAN per department, routing between VLANs, and basic access security.
- **Tool:** Cisco Packet Tracer
- **Devices:** 1 router (Cisco 1941), 2 multilayer switches (3560-24PS), 3 access switches (2960-24TT), 6 PCs, 1 management laptop

## Network Design
- **Core:** two multilayer switches (MLS1 and MLS2) connected by an EtherChannel (Port-channel 1, LACP) made of 6 links.
- **Access layer:** three access switches (SW1, SW2, SW3). Each one has two uplinks, one to each multilayer switch, so the network keeps working if one uplink fails. Spanning Tree Protocol blocks one path to prevent Layer 2 loops. MLS1 is the root bridge.
- **Routing:** router R1 connects to the core with two routed point-to-point links (10.0.0.0/30 and 11.0.0.0/30). OSPF (process 1, area 0) runs between R1 and the multilayer switches.
- **Inter-VLAN routing:** done on MLS1 with SVIs (switch virtual interfaces).
- **DHCP:** configured on MLS1, with one pool per VLAN. The pools also give clients the DNS server address 8.8.8.8.
- **Management:** Laptop0 is connected to R1 with a console cable.

## IP and VLAN Plan

| VLAN | Department | Subnet | Gateway (SVI on MLS1) |
|------|------------|--------|------------------------|
| 10 | Accounting | 192.168.1.0/24 | 192.168.1.1 |
| 20 | Sales | 192.168.2.0/24 | 192.168.2.1 |
| 30 | HR | 192.168.3.0/24 | 192.168.3.1 |

| Link | Network | Side A | Side B |
|------|---------|--------|--------|
| MLS1 to R1 | 10.0.0.0/30 | MLS1: 10.0.0.1 | R1: 10.0.0.2 |
| MLS2 to R1 | 11.0.0.0/30 | MLS2: 11.0.0.2 | R1: 11.0.0.1 |

## Port Map

| Device | Port | Connected to | Port | Mode |
|--------|------|--------------|------|------|
| R1 | Gig0/1 | MLS1 | Fa0/10 | Routed (10.0.0.0/30) |
| R1 | Gig0/0 | MLS2 | Fa0/10 | Routed (11.0.0.0/30) |
| MLS1 | Fa0/4 - Fa0/9 | MLS2 | Fa0/4 - Fa0/9 | Port-channel 1 (LACP) |
| MLS1 | Fa0/3 | SW1 | Fa0/4 | Trunk |
| MLS1 | Fa0/2 | SW2 | Fa0/4 | Trunk |
| MLS1 | Fa0/1 | SW3 | Fa0/4 | Trunk |
| MLS2 | Fa0/3 | SW1 | Fa0/3 | Trunk |
| MLS2 | Fa0/2 | SW2 | Fa0/3 | Trunk |
| MLS2 | Fa0/1 | SW3 | Fa0/3 | Trunk |
| SW1, SW2, SW3 | Fa0/1 - Fa0/2 | PCs | - | Access (VLAN 10, 20, 30) |

## Configuration Summary

| Feature | Details |
|---------|---------|
| VLANs | VLAN 10 (Accounting), VLAN 20 (Sales), VLAN 30 (HR) |
| Trunking | 802.1Q trunks between the access switches and the multilayer switches |
| EtherChannel | Port-channel 1, LACP, between MLS1 and MLS2 (Fa0/4 - Fa0/9) |
| STP | Rapid-PVST on all switches. MLS1 is the root bridge, with priority 24576 |
| Inter-VLAN routing | SVIs on MLS1 |
| OSPF | Process 1, area 0, between R1 and the multilayer switches |
| DHCP | Pools for VLAN 10, 20, and 30 on MLS1 |
| DNS | DHCP gives clients the DNS server address 8.8.8.8 |

## Security
- **Port Security** on the access ports (Fa0/1 and Fa0/2) of all three access switches (SW1, SW2, SW3): maximum 1 MAC address per port, violation mode `shutdown`, and sticky MAC learning.

## Verification
Screenshots of each check are in the `screenshots/` folder. Key results:

**EtherChannel (MLS1):** all six links are bundled in Port-channel 1 with LACP.
```
MLS1#show etherchannel summary
Group  Port-channel  Protocol    Ports
1      Po1(SU)       LACP        Fa0/4(P) Fa0/5(P) Fa0/6(P) Fa0/7(P) Fa0/8(P) Fa0/9(P)
```

**Spanning Tree (MLS1, VLAN 10):** MLS1 is the root bridge.
```
MLS1#show spanning-tree vlan 10
VLAN0010
  Spanning tree enabled protocol rstp
  Root ID    Priority    24586
             This bridge is the root
```

**OSPF (R1):** process 1, both core links in area 0.
```
R1>show ip protocols
Routing Protocol is "ospf 1"
  Router ID 11.0.0.1
  Routing for Networks:
    10.0.0.0 0.0.0.3 area 0
    11.0.0.0 0.0.0.3 area 0
```

**Other checks:**

| Check | Command | Device |
|-------|---------|--------|
| VLANs | `show vlan brief` | SW1, SW2, SW3 |
| Trunks | `show interfaces trunk` | MLS1, MLS2 |
| Routing table | `show ip route` | R1, MLS1 |
| DHCP | `show ip dhcp binding` | MLS1 |
| Port Security | `show port-security interface fa0/1` | SW1, SW2, SW3 |
| Connectivity | `ping` between PCs in different VLANs | PC0, PC2, PC4 |

## Limitations and Next Steps
- The gateways (SVIs) and the DHCP server are on MLS1 only. If MLS1 fails, the VLANs lose their gateway. Next step: add **HSRP** between MLS1 and MLS2.
- This lab has no DNS server. DHCP only gives clients the DNS address. Next step: add a server with the DNS service enabled.
- Next step: add ACLs to control traffic between VLANs.

## What I Learned
- How STP, EtherChannel, and OSPF work together to make a network redundant.
- How to check a network step by step with `show` commands to find where a problem is.
- In a previous test of this kind of lab, a wrong default gateway in one VLAN stopped the ping between VLANs. I found it by comparing the IP settings on each device.

## Repository Structure
```
.
├── README.md
├── network-lab.pkt
├── images/
│   └── topology.png
├── configs/            running-config of each device (.txt)
└── screenshots/        show command output and ping tests
```
