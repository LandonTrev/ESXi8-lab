# Hardware

All lab hardware in one place. Reference note - update only when something changes.

---

## Minisforum MS-01 - ESXi host
- CPU: Intel i9-13900H (hybrid P/E-core)
- RAM: 64GB DDR5 (2x32GB; 5600-rated, runs at 5200 - normal for the platform)
- NICs: 2x 2.5GbE copper, 2x SFP+ 10GbE (Intel X710)
- Boot drive: Fanxiang S500Pro 256GB NVMe - ESXi 8.0.3 (build 24677879, free 8.0U3e)
- Datastore drive: Samsung 990 PRO 2TB NVMe - VM datastore (pending setup)

Quirks and fixes (important):
- Hybrid-CPU purple screen: ESXi panics with "Fatal CPU mismatch" on P/E cores. Fixed permanently with:
  esxcli system settings kernel set -s cpuUniformityHardCheckPanic -v FALSE
  (see [[Topics/ESXi on Hybrid CPUs]])
- BIOS shows no NVMe drive list: normal for this AMI BIOS. "UEFI NVME" in boot options means detected. Real confirmation is the ESXi installer disk screen.
- VMD must stay Disabled (Advanced, Onboard Devices, VMD setup menu) or ESXi cannot see the NVMe drives.

Access: DCUI on HDMI (F2 config, F12 shutdown, Alt+F1 shell). Web UI at https://IP/ui (pending network).

---

## MikroTik CRS310-8G+2S+IN - core switch
- 8x 2.5GbE copper plus 2x SFP+ 10GbE, runs RouterOS
- Connected: SFP+ DAC to MS-01 X710 (done). Router uplink: not yet (this is why the host has no DHCP)
- No power button - unplug to power off; designed for 24/7
- Factory default IP: 192.168.88.1 (WinBox or WebFig)
- Status: unconfigured

---

## UGREEN DXP2800 - NAS
- 2-bay, 2x WD Red Plus 4TB (WD40EFZZ)
- Plan: replace stock OS with TrueNAS, ZFS mirror, NFS/iSCSI datastores for ESXi plus AD-integrated SMB shares
- Status: pending (after ESXi is networked)

---

## Boot media and spares

| Item | Status |
|---|---|
| Sony 8GB USB | Known-good ESXi installer stick - keep ||

Lesson: verify new flash media with H2testw or ValiDrive before trusting it.

---

## Cabling / topology (target)

```
Home Router -- CRS310 (2.5GbE uplink)
                 |-- SFP+ 10G DAC -- MS-01 (ESXi)
                 |-- 2.5GbE -- DXP2800 (TrueNAS)
```

Interim (until switch is configured): MS-01 copper port direct to router.
