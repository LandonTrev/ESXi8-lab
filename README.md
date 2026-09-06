# ESXi VSphere 8 Lab

This lab is built around a mini pc running ESXi as a type 1 bare metal hypervisor. The host connects to a MicroTik CRS Switch over a trunk (plus and access port for management). The switch handles VLAN filtering and L3 routing between the lab segments (VLAN10,VLAN20,VLAN30,etc.)

The list of hardware is as follows:

- Minisforum MS-01 (i9-13900H, 64GB DDR5), the only ESXi host.
- MikroTik CRS310-8G+2S+IN, doing VLAN filtering, routing between VLANs, and DHCP relay. CRS models having routing capabilities.
- A 10GbE SFP+ DAC from the switch to the host, carrying tagged traffic
- A 2.5 GBE cable for host management traffic. Not tagged, this is the access port.
- UGREEN DXP2800 with 2x 4TB in a ZFS mirror. Not set up yet see roadmap.

**Why I'm Building this:** Originally, I started out with a fully Virtualized VirtualBox lab. The networking was hard to get a grasp on as it gets abstracted behind Virtual Adapters and NIC's mapped to interfaces on a virtualized router/firewall. I built this to actually see how VLAN's, trunk ports, access ports, and switching works with gear I have right in front of me. Also to work with a hypervisor and Switch OS which are used in actual enterprise environments. Running VirtualBox VMs on my primary laptop was also painfully slow and tough to navigate with the 4-5 VM's I'd have open at any given time.

**Status:** active build, updated each session.

## Architecture

### Physical topology

Router, then switch (w/ L3 capability), then server. The switch is the core of the network instead of something traffic just passes through, which is how most real networks are laid out.

The host talks to the switch over a 10GbE DAC carrying tagged VLAN traffic. ESXi management gets its own 2.5GbE cord, though its down to 1GbE because the switch's ports top out there.

### Virtualization

ESXi 8.0 U3 sits directly on the hardware. One vSwitch (vSwitch-Lab) handles lab traffic on the 10GbE uplink, with a tagged port group for each VLAN. Host management stays on its own physical NIC.

The MS-01 has a mix of performance and efficiency cores, which ESXi does not love. Getting it stable took kernel tuning. Details are in the build logs.

### Identity

On VLAN10 DC01 runs Windows Server 2025 and holds the landon.lab forest, plus DNS and DHCP for everything. VLAN 20 holds clients, CLIENT01 is Windows 11 Enterprise, joined to the domain and managed from the DC. Users follow a naming standard, and my admin account is separate from my normal one.

### Network segmentation

VLANs are tagged with 802.1Q and enforced by bridge VLAN filtering on the MikroTik. Servers sit on VLAN 10, clients on VLAN 20, and they can't reach each other freely. Traffic between them gets routed and then filtered, with the firewall dropping everything except the AD services clients actually need.

DHCP was the interesting part. Client requests are broadcasts, so they die at the VLAN boundary. A relay picks them up and forwards them as unicast to DC01. Reads the scope table and selects an available address within the range.

### Storage

Not built yet. The plan is TrueNAS on the DXP2800 serving NFS and iSCSI datastores to ESXi off the ZFS mirror, plus SMB shares joined to the domain.

It's blocked on hardware. The NAS ships with UGREEN's own OS on the internal storage. Determining whether it is worth it to get hands on with TrueNAS. Other things come first.

## What works

- ESXi installed and configured for the CPU, datastore on its own NVMe
- vSwitch-Lab set up with a tagged port group per VLAN, management traffic on a separate NIC
- landon.lab forest with DNS and DHCP
- VLAN 10 for servers and VLAN 20 for clients, segmented and tested end to end
- DHCP relay working across the VLAN boundary
- Windows 11 client joined to the domain, renamed, and registered in DNS (so other machines and the domain can find it by name) 
- Separate standard and admin accounts
- Firewall between the VLANs dropping everything but the AD services clients need 

## Roadmap

- VLAN 30 for storage and VLAN 99 for management
- TrueNAS: NFS and iSCSI datastores on the ZFS mirror, SMB shares joined to the domain
- PowerShell. Moving day to day AD work out of the GUI and committing the scripts here
- AD Certificate Services on its own member server
- Tailscale for remote access, running on the NAS rather than the host. If I fat finger a firewall rule I don't want to lose the hypervisor along with it
- Server Core and Azure Arc
- SIEM and NIDS work in here eventually. TryHackMe is covering the basics for now

## Repository layout

| Note | Contents |
|---|---|
| [Hardware.md](Hardware.md) | Every device: specs, quirks, fixes |
| [Roadmap.md](Roadmap.md) | Certification track and lab build phases |
| [Sessions/](Sessions/) | Dated build and troubleshooting logs |
| [Topics/](Topics/) | Distilled reference notes per concept |
| [Images/](Images/) | Architecture and concept diagrams |

This is an [Obsidian](https://obsidian.md) vault. Clone it and open the folder as a vault for full wiki-link navigation or just browse the Markdown here on GitHub.
