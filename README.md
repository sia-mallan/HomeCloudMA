# Homelab
 
A segmented, self-hosted home network built around a pfSense firewall, a managed Cisco switch, and a small Proxmox cluster. I use it to practice network design, virtualization, and self-hosting, and to document what I learn.
 
> **Skills demonstrated:** VLAN segmentation, firewall rule design, Cisco IOS, Proxmox virtualization, Docker, reverse proxies, zero-trust remote access, DNS, TrueNAS storage, troubleshooting.
 
## Network Topology
 
| Diagram PlaceHolder |
 
```
Internet
   |
[pfSense firewall]  (HP EliteDesk 800 G1 SFF)
   |  802.1Q trunk
[Cisco Catalyst 2960L]
   |-- Proxmox nodes     (VLAN 20)
   |-- Services          (VLAN 30)
   |-- Storage / NAS     (VLAN 40)
   |-- Raspberry Pi 5    (VLAN 50)
   '-- DMZ               (VLAN 100)
```
 
## VLAN Design
 
| VLAN | Name | Gateway | Purpose |
|---|---|---|---|
| 10 | MGMT | TBD | Network device management |
| 20 | PROXMOX_NODES | TBD | Proxmox host management |
| 30 | SERVICES | TBD | VMs and containers |
| 40 | STORAGE | 10.0.40.1 | NAS traffic |
| 50 | RPI | 10.0.50.1 | Raspberry Pi (Docker, NetBird, Pi-hole) |
| 100 | DMZ | 10.0.100.1 | Reverse proxy for exposed services |
| 999 | SINK | none | Unused ports, no routing |
 
Firewall rules are default-deny between VLANs, with explicit allows per service. See [`configurations/network.md`](configurations/network.md).
 
## Hardware
 
| Role | Device |
|---|---|
| Firewall | HP EliteDesk 800 G1 SFF running pfSense 2.9.0 |
| Switch | Cisco Catalyst 2960L (8-port) |
| Proxmox node | HP EliteDesk 800 G4 Mini |
| NAS | HP EliteDesk 800 G4 SFF running TrueNAS (bare metal) |
| Utility server | Raspberry Pi 5 (16 GB) running Docker |
 
## Services
 
| Service | Where | Purpose |
|---|---|---|---|
| pfSense | Dedicated HP 800 G1 | Firewall, routing, inter-VLAN rules |
| Proxmox VE | HP 800 G4 Mini | Virtualization | 
| Jellyfin | Proxmox | Media server |
| Immich | Proxmox | Photo backup |
| Pi-hole | Raspberry Pi | Network-wide DNS filtering |
| NetBird | Raspberry Pi | Zero-trust remote access and reverse proxy |
| Cloudflare Tunnel | Raspberry Pi | Public access without port forwarding |
| Portainer | Raspberry Pi | Docker management | 
| Dashy | Raspberry Pi | Single dashboard for the whole lab |
| Uptime Kuma | Raspberry Pi | Service monitoring |
| OpenMediaVault + ArozOS | HP 800 G4 SFF | Shared storage for all nodes |
| Proxmox Datacenter Manager | TBD | Manage all nodes from one place |
| DMZ reverse proxy | VLAN 100 | Front services on other VLANs |
 
## Remote Access Approach
 
- No inbound port forwarding. Remote access goes through NetBird (WireGuard-based mesh VPN).
- Selected services are published on a custom domain through NetBird's reverse proxy.
- Internal-only services resolve through local DNS and are never exposed publicly.
## Documentation
 
| Doc | Description |
|---|---|
| [`configurations/cisco-switch.md`](configurations/cisco-switch.md) | Cisco switch configuration |
| [`configurations/pfsense.md`](configurations/pfsense.md) | Firewall setup and rules |
 
## Challenges and Fixes
 
**VLAN 50 devices received no DHCP.** The switch config checked out, but replies from pfSense weren't reaching the Pi. I traced it to the pfSense Realtek 2.5G NIC (`rge` driver) carrying the VLAN trunk, and planned to move the trunk to the Intel NIC.
 
**Flat network to VLANs.** I started flat (10.0.0.0/24) and migrated to segmented VLANs once I had more services. The design notes are in [`configurations/decisions.md`](configurations/decisions.md).
 
## Security Notes
 
All secrets, keys, and credentials are excluded from this repo. Configs are sanitized and use placeholders.