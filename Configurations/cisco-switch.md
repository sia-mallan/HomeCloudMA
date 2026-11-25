# Cisco Catalyst 2960L: 8 Port

Layer 2 switch for the homelab. It carries the VLAN trunk from pfSense and provides access ports for Proxmox nodes, the Raspberry Pi, and storage.

## Overview

| Item | Value |
|---|---|
| Model | Cisco Catalyst 2960L, 8-port |
| Hostname | `SW1` |
| Domain | `example.arpa` |
| IOS | 15.2 |
| Management IP | `10.0.10.2/24` (MGMT Vlan) |
| Gateway | `10.0.10.1` (pfSense) |
| Management access | SSH, restricted by ACL 10 to `10.0.10.0/24` | Physical configuration by RJ45 console cable |

## VLAN Plan

| VLAN ID | Name | Subnet / Gateway | Purpose |
|---|---|---|---|
| 10 | MGMT | `X.X.10.1` | Network management |
| 20 | PROXMOX_NODES | `X.X.20.1` | Proxmox host management UI |
| 30 | SERVICES | `X.X.30.1` | VMs and containers (apps) |
| 40 | STORAGE | `10.0.40.1` | NAS / Open Media Vault |
| 50 | RPI MGMT / DNS Server | `10.0.50.1` | Raspberry Pi (Docker, NetBird, Pi-hole) |
| 100 | DMZ | `10.0.100.1` | Reverse proxy for exposed services |
| 999 | SINK | none | Unused ports parked here, no routing |

## Security Hardening

- SSH only on the VTY lines; Telnet disabled.
- ACL 10 limits management access to `10.0.0.0/24`.
- BPDU Guard on access ports Gi0/2 to Gi0/8, so a rogue switch can't alter spanning tree.
- Unused ports are shut down and placed in VLAN 999 (SINK).
- DHCP snooping is currently disabled.
- Only the VLANs that are needed are allowed on the trunk (no `allowed vlan all`).

## Cli Config

```
hostname SW1
ip domain-name example.arpa
!
enable secret <REDACTED>
username <user> privilege 15 secret <REDACTED>
!
crypto key generate rsa modulus 2048
ip ssh version 2
!
vlan 10
 name MGMT
vlan 20
 name PROXMOX_NODES
vlan 30
 name SERVICES
vlan 40
 name STORAGE
vlan 50
 name RPI
vlan 100
 name DMZ
vlan 999
 name SINK
!
interface Vlan 10
 ip address 10.0.10.2 255.255.255.0
 no shutdown
ip default-gateway 10.0.10.1
!
! Uplink to pfSense
interface GigabitEthernet0/1
 description pfSense-LAN
 switchport mode trunk
 switchport trunk allowed vlan 1,10,20,30,40,50,100
!
! Proxmox Mangement Interface
interface GigabitEthernet0/2-5
 description Proxmox
 switchport mode trunk
 switchport trunk allowed vlan 20,30
 spanning-tree portfast
 spanning-tree bpduguard enable
!
! Raspberry Pi 5
interface GigabitEthernet0/6
 description rpi1
 switchport mode access
 switchport access vlan 50
 spanning-tree portfast
 spanning-tree bpduguard enable
!
! NAS
interface GigabitEthernet0/7
 description NAS
 switchport mode access
 switchport access vlan 40
 spanning-tree portfast
 spanning-tree bpduguard enable
!
! Reverse Proxy / Public Apps
interface GigabitEthernet0/8
 description DMZ
 switchport mode access
 switchport access vlan 100
 spanning-tree portfast
 spanning-tree bpduguard enable
!
access-list 10 permit 10.0.0.0 0.0.0.255
line vty 0 15
 access-class 10 in
 transport input ssh
 login local
```

## Verification Commands

```
show vlan brief
show interfaces trunk
show interfaces status
show running-config interface Gi0/1
show mac address-table vlan 50
show spanning-tree summary
```

## Troubleshooting Log

### Pi gets no DHCP on VLAN 50

- **Symptom:** The Raspberry Pi on Gi0/6 (access VLAN 50) received no DHCP lease, but worked when placed on VLAN 1.
- **Checked:** Switch config (VLAN exists, trunk allows 50, port is access VLAN 50, DHCP snooping off). All looked correct.
- **Suspect:** pfSense's Realtek 2.5G NIC (`rge0` driver) carrying the VLAN trunk, with VLAN 50 replies not reaching the switch.
- **Plan:** Move the VLAN trunk to the Intel NIC (`em0`) and use the Realtek as WAN.
- Install PFsense within a virtual machine and pass realtek NIC in as Virtual Nic for more natural drivers using Linux for better compatibility than freeBSD
- Eventually install an Intel NIC for native support and less headache in the future
- **Result:** Many issues with Vlan tagging running the realtek NIC. After many hours of reading and trouble shooting, orgianlly thinking my configurations was the issue, I learned the issues were due to driver issues between Realtek and freeBSD software

## Lessons Learned

- Start with the switch config, then verify the firewall side. A correct switch config doesn't guarantee the trunk path works end to end.
- NIC and driver choice matters for VLAN trunking on pfSense.
- Slowing down the setup process and confirming successful configuration was my biggest lesson. I originally created all planned VLANs at once, which made troubleshooting difficult when issues arose with specific VLANs or firewall communication rules between virtual networks.
