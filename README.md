# ESXi vSphere 8.0 v3 Lab

### Objectives

**What I've built so far**
- ESXi 8.0 U3 hypervisor on the MS-01 
- 10GbE trunk to the host, port group per VLAN, management on its own uplink
- 3 VLANs (servers, clients, NAS) routed on the Switch (L3 capabilities)
- Default deny between VLANs, only allowing what's needed
- landon.lab domain on DC01 with DNS + DHCP
- DHCP relay on VLAN's so clients get addresses from the DC across VLANs
- NAT Masquerade rules configured for egress to the internet on VLAN10, 20, and the management subnet.  
- Configured OU hierarchy (admins, computers, security groups, users)
- CLIENT01 joined to the domain
- Basic Group Policy (Welcome Message with a Security Deterrent and Screen Lock/Sleep)
- TrueNAS joined to the domain, per-department shares
- Mapped drives through Group Policy and configured permissions via domain local groups
- Pi-hole in Docker Container on docker01, DC forwards DNS to it
- Tailscale functioning as a subnet router to reach the lab remotely.

**What's next**
- Move management to its own VLAN 99
- Full default deny, including the switch itself and internet traffic
- Personal home drives for each user
- NFS or iSCSI storage from TrueNAS to ESXi
- Notes good enough to rebuild the whole lab from scratch

**Down the road: Azure**
- Site-to-site VPN from the MikroTik to Azure
- Plan IP ranges so nothing overlaps, lock down Azure like the lab
- DNS working both ways
- Sync domain users to Entra ID

### Detailed Overview
This lab is built around a mini pc running ESXi as a bare metal hypervisor. The host connects to a MicroTik CRS Switch over a trunk (plus an access port for management). The switch handles VLAN filtering and L3 routing between the lab segments (VLAN10, VLAN20, VLAN30, VLAN99)

The list of hardware is as follows:

- Minisforum MS-01 (i9-13900H, 64GB DDR5), the only ESXi host.
- MikroTik CRS310 which assists with VLAN filtering, routing between VLANs, DHCP relay, NAT Masquerade, and Firewall rules. 
- A 10GbE SFP+ DAC cable from the switch to the host (carrying tagged traffic)
- A 2.5 GBE cable for host management traffic. Not tagged this is the access port.
- UGREEN DXP2800 with 2x 4TB in a ZFS mirror

**Why I'm Building this:**
   Originally, I started out with a fully Virtualized VirtualBox Security lab. The networking was hard to get a grasp on as it gets abstracted behind Virtual Adapters and NIC's mapped to interfaces on a virtualized router/firewall. This was frustrating as I had such a keen interest in the networking portion after starting.
   
   I built this lab to actually see how VLAN's, trunk ports, access ports, and switching works with gear I have right in front of me. Also to work with a hypervisor and Switch OS which are used in actual enterprise environments. However if I were to go back I would purchase a Cisco Switch rather than a MicroTik. Possibly even with its own dedicated router instead of using the L3 capabilities on the CRS310. 
## Architecture
### Physical topology

Apartment Router -> switch (w/ L3 capability) -> then server. 
The switch is the core of the network instead of something traffic just passes through.

The host talks to the switch over a 10GbE DAC cable carrying tagged VLAN traffic. ESXi management gets its own 2.5GbE cord, though its down to 1GbE as the switch's ports top out there.
### Virtualization

ESXi 8.0 U3 runs on the MS-01. Lab traffic goes through vSwitch-Lab over the 10GbE uplink, with a tagged port group for each VLAN. Management gets its own vSwitch (vSwitch0) and its own uplink on the 2.5GbE port. Important separation as when things hit the fan on the lab side, I can still get into the host.

The MS-01 has a mix of performance and efficiency cores, which ESXi does not love. Getting it stable took kernel tuning. Details can be found here: [[ESXi on Hybrid CPUs]]
### Identity

On VLAN10 DC01 runs a Debian Machine for Docker(pi-hole)/Tailscale and Windows Server 2025 providing the landon.lab forest, plus DNS and DHCP. VLAN 20 holds clients. An example being CLIENT01 which is a Windows 11 Enterprise machine and is joined to the domain.

Active Directory is organized into a corporate structure with OU's in a hierarchical manner. The main OU Corp contains the Admins, Computers, Groups, Service Accounts, and Users OU's. Groups contains my Domain Local and Global Security groups to manage staff from each department and to assign permissions for policy like mapped department drives on startup.

### Network segmentation

VLANs are tagged with 802.1Q and enforced by bridge VLAN filtering on the MikroTik. Servers sit on VLAN 10, clients on VLAN 20, and they can't reach each other freely. Traffic between them gets routed and then filtered, with the firewall dropping everything except the AD services clients actually need.

DHCP was the interesting part. Client requests are broadcasts, so they die at the VLAN boundary. A relay picks them up and forwards them as unicast to DC01. Reads the scope table and selects an available address within the range.
### Storage

TrueNAS on the DXP2800 NAS with ZFS datasets serving SMB shares to clients on the domain. I will likely expand the use case of the NAS as there is plenty of storage to mess around with. Saving backups, NFS or iSCSI datastores for ESXi or hosting a media server are options that I have thought about. 

## Repository layout

| Note                       | Contents                             |
| -------------------------- | ------------------------------------ |
| [Hardware.md](Hardware.md) | Devices, Hardware and Specs          |
| [Sessions/](Sessions/)     | Dated build and troubleshooting logs |
| [Topics/](Topics/)         | Reference notes per concept          |
| [Images/](Images/)         | Architecture and concept diagrams    |
