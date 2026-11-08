# ESXi Basics

What it is: VMware ESXi is a bare-metal (Type 1) hypervisor - an OS whose only job is running virtual machines. It installs directly on hardware; there is no Windows or Linux underneath. It is not Linux: the kernel is VMware's own VMkernel, even though the management shell feels Unix-like.

In my lab: ESXi 8.0.3 (free vSphere Hypervisor 8.0U3e, build 24677879) on the Minisforum MS-01. Boot drive is a 256GB NVMe; VMs live on a separate 2TB NVMe datastore. All real operating systems (Windows Server for AD, Linux servers) run as guests on top.

## Key concepts
- Hypervisor vs guest: ESXi is the platform; the VMs are the actual servers you administer.
- Datastore: storage ESXi uses for VM files (VMFS on local disks, or NFS/iSCSI from a NAS).
- DCUI: the yellow and grey console on the physical machine - F2 configure, F12 shutdown, Alt+F1 shell.
- Web UI (Host Client): https://host-IP/ui - day-to-day management, no vCenter needed.
- Free license: full hypervisor, but no vCenter connectivity and 8 vCPU max per VM. Fine for a lab. For vCenter practice later, run the VCSA appliance on its 60-day eval.

## Getting the free version (post-Broadcom)
support.broadcom.com
free account
division set to VMware Cloud Foundation
search "VMware vSphere Hypervisor"
download the .iso (not the offline-bundle ZIP)
Write with Rufus: GPT, UEFI (non-CSM), ISO Image mode.

## Commands I have actually used
```
esxcli system settings kernel set -s SETTING -v VALUE    # set a kernel boot option (plural "settings")
esxcli system settings kernel list -o SETTING            # verify: check the Configured column
```

Learned in: [[Sessions/2026-07-08 ESXi Install]]
