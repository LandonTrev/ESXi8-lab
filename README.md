# ESXi VSphere 8 Lab

This lab is built around a mini pc running ESXi as a type 1 bare metal hypervisor. The host connects to a MicroTik CRS Switch over a trunk (plus and access port for management). The switch handles VLAN filtering and L3 routing between the lab segments (VLAN10,VLAN20,VLAN30,etc.)

The list of hardware is as follows:

- Minisforum MS-01 (i9-13900H, 64GB DDR5), the only ESXi host.
- MikroTik CRS310-8G+2S+IN, doing VLAN filtering, routing between VLANs, and DHCP relay. CRS models having routing capabilities.
- A 10GbE SFP+ DAC from the switch to the host, carrying tagged traffic
- A 2.5 GBE cable for host management traffic. Not tagged, this is the access port.
- UGREEN DXP2800 with 2x 4TB in a ZFS mirror

**Why I'm Building this:** Originally, I started out with a fully Virtualized VirtualBox lab. The networking was hard to get a grasp on as it gets abstracted behind Virtual Adapters and NIC's mapped to interfaces on a virtualized router/firewall. I built this to actually see how VLAN's, trunk ports, access ports, and switching works with gear I have right in front of me. Also to work with a hypervisor and Switch OS which are used in actual enterprise environments. Running VirtualBox VMs on my primary laptop was also painfully slow and tough to navigate with the 4-5 VM's I'd have open at any given time.
## Architecture

### Physical topology

Router, then switch (w/ L3 capability), then server. The switch is the core of the network instead of something traffic just passes through.

The host talks to the switch over a 10GbE DAC carrying tagged VLAN traffic. ESXi management gets its own 2.5GbE cord, though it runs at 1GbE as the switch's ports top out there.

### Virtualization

ESXi 8.0 U3 sits on the MS-01. One vSwitch (vSwitch-Lab) handles lab traffic on the 10GbE uplink, with a tagged port group for each VLAN. Host management stays on its own physical NIC.

The MS-01 has a mix of performance and efficiency cores, which ESXi does not love. Getting it stable took kernel tuning. Details are in the build logs.

### Identity

On VLAN10 DC01 runs Windows Server 2025 and holds the landon.lab forest, plus DNS and DHCP for everything. VLAN 20 holds clients, CLIENT01 is Windows 11 Enterprise, joined to the domain and managed from the DC. Users follow a naming standard, and my admin account is separate from my normal one.

### Network segmentation

VLANs are tagged with 802.1Q and enforced by bridge VLAN filtering on the MikroTik. Servers sit on VLAN 10, clients on VLAN 20, and they can't reach each other freely. Traffic between them gets routed and then filtered, with the firewall dropping everything except the AD services clients actually need.

DHCP was the interesting part. Client requests are broadcasts, so they die at the VLAN boundary. A relay picks them up and forwards them as unicast to DC01. Reads the scope table and selects an available address within the range.

### Storage

TrueNAS on the DXP2800 NAS serving SMB shares on the domain. 

## Repository layout

| Note                       | Contents                              |
| -------------------------- | ------------------------------------- |
| [Hardware.md](Hardware.md) | Every device: specs, quirks, fixes    |
| [Sessions/](Sessions/)     | Dated build and troubleshooting logs  |
| [Topics/](Topics/)         | Distilled reference notes per concept |
| [Images/](Images/)         | Architecture and concept diagrams     |
