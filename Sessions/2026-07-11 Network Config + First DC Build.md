# Session — ESXi Networking Fix + First Domain Controller Build

**Date:** 2026-07-11 **Goal:** Get ESXi on the network, access the web UI, build the first VM as an AD Domain Controller **Outcome:** First DC promoted, domain live

> **Note:** Network addressing in this repo is sanitized. Real values live in the private vault only. Placeholders below use `10.0.0.x` style; substitute your own.

---

## 1. ESXi Management Networking

### Problem

Host showed a `169.254.x.x` (APIPA) address with "Waiting for DHCP..." and a DHCP lookup failed warning. Web UI unreachable from the workstation.
### Diagnosis

DCUI → **F2** → Configure Management Network → Network Adapters

|Device|Status|
|---|---|
|vmnic0|Disconnected|
|vmnic1|**Connected (...)** ← was bound [X], but not the router port|
|vmnic2|Disconnected|
|vmnic3|**Connected** ← actual router uplink|

Management (vmk0) was bound to the wrong NIC.

**Why two NICs showed "Connected":** one was the 2.5GbE copper router port, the other was the SFP+ DAC to the switch. The DAC reports link-up even though the switch is unconfigured. a direct point-to-point copper link electrically detects both ends without any switch config. But no DHCP server sits behind it, hence the APIPA fallback.

### Fix

- Deselected vmnic1, selected **only vmnic3**
- Did NOT team both NICs (one uplink for management)
- Enter → Esc → **Y** to restart management network
- Host pulled a valid DHCP lease ✓
- Web UI reachable at `https://<host-ip>/ui`

---

## 2. Storage - Datastore Provisioning

Initial state: only one datastore (110 GB) — that was the small **boot SSD**. The 2TB NVMe was installed but never formatted (intentionally left untouched during the ESXi install).

**Actions:**

- Renamed boot datastore → **`ESXiBoot`** (110 GB)
- Created new datastore → **`vmstore`** on the 2TB NVMe — **1.82 TB, VMFS6**
- Created `vmstore/ISOs/` directory
- Uploaded Windows Server 2025 eval ISO (~7.7 GB) — build `26100.32230.260111-0550`

`vmstore` is now the VM home. `ESXiBoot` holds ESXi only.

---

## 3. VM Build - DC01

### OS choice: Windows Server 2025 Standard (Desktop Experience)

Rationale:

- Server 2022 mainstream support **ends October 2026** a fresh 2022 build enters maintenance mode almost immediately
- Most orgs upgrading are going straight to 2025 rather than stopping at 2022
- 2025 changed AD security defaults in ways directly relevant to the security track: **Credential Guard on by default**, **NTLM deprecated in favor of Kerberos**, **SMB signing required by default**
- 180-day eval, no license needed for lab

**Standard** (not Datacenter): Datacenter only matters for unlimited VM licensing / Storage Replica at scale. **Desktop Experience** (not Core) GUI while learning AD. Core is a separate future exercise.

### VM specs

| Resource  | Value                  |
| --------- | ---------------------- |
| Name      | `DC01`                 |
| vCPU      | 2                      |
| RAM       | 4 GB                   |
| Disk      | 60 GB,                 |
| Datastore | `vmstore`              |
| NIC       | VMXNET3 → `VM Network` |
| Firmware  | EFI                    |

---

## 4. AD DS Install + DC Promotion

- Server Manager → Manage → Add Roles and Features → **Active Directory Domain Services**
- Promoted to DC: **Add a new forest**
- DNS Server role installed automatically (AD depends on DNS)
- Global Catalog: yes (forced — first DC in a forest)
- **DSRM password set** — stored in a password manager, never in the repo

> The promotion wizard has a **View script** button on the Review Options screen. It emits the equivalent PowerShell (`Install-ADDSForest`). Worth reading; that's the automation path.

### Prereq check warnings (both expected)

1. **No static IP**: legitimate, fixed immediately after promotion (see below)
2. **DNS delegation cannot be created**: No parent zone exists for a `.lab` domain because it isn't a real TLD. The wizard itself says "no action is required."

Server rebooted automatically. Login switched from a local account to the domain: `<NETBIOS>\Administrator`.

---

## 5. Static IP (post-promotion)

A DC must have a static IP. It's also the DNS server for the domain. If the address shifts on a DHCP lease renewal, every client loses name resolution and the domain effectively breaks. 

`ncpa.cpl` → Ethernet adapter → IPv4 Properties:

| Field           | Value                                           |
| --------------- | ----------------------------------------------- |
| IP address      | `10.10.10.10` _(static, outside the DHCP pool)_ |
| Subnet mask     | `255.255.255.0`                                 |
| Default gateway | `10.10.10.1`                                    |
| Preferred DNS   | `127.0.0.1`                                     |

DNS points at loopback, not the router. The DC is the domain's DNS server and it resolves against itself. Pointing it at the router would break AD name resolution.

Chose a low host address to sit outside the router's DHCP pool (pools commonly start at `.100`). Verify your own pool before picking.

### Verification

```
ipconfig /all
  DHCP Enabled . . . . . . . . . . : No
  IPv4 Address . . . . . . . . . . : 10.10.10.10 (Preferred)
  Default Gateway  . . . . . . . . : 10.10.10.1
  DNS Servers  . . . . . . . . . . : ::1
                                     127.0.0.1
  Primary DNS Suffix . . . . . . . : <domain>
  Description  . . . . . . . . . . : vmxnet3 Ethernet Adapter

nslookup <domain>
  Name:    <domain>
  Address: 10.10.10.10
```

`nslookup` resolving the domain back to the DC's own IP confirms AD created its DNS zone and the DC registered itself. **Domain is live.**

---

## Key takeaways

- **"Connected" in the ESXi DCUI means physical link only** - not a working uplink. An SFP+ DAC shows link-up even with the switch on the other end completely unconfigured, because point-to-point copper detects both ends electrically. Use **`D` (View Details)** on a vmnic to check link speed and disambiguate: DAC ≈ 10000 Mbps, 2.5GbE copper ≈ 2500 Mbps.
- **Creating a VM builds hardware, not an OS.** A fresh `.vmdk` is a blank disk  - "No Media" on the virtual disk at first boot is _expected_. The ISO is the installer that fills it. Once the OS is installed the ISO becomes irrelevant and the VM boots from disk.
- **A VM's folder on the datastore IS the VM.** `DC01.vmx` (config), `DC01-flat.vmdk` (disk data), `DC01.nvram` (UEFI state), `vmware.log`. Copy the folder, register the `.vmx`, and the VM comes back elsewhere. That's file-level migration/backup.
- **Edit settings changes don't take effect until Save is clicked.** Cost ~20 minutes of "No Media" debugging on a config screen that _displayed_ correct settings which had never been committed.
- **Domain controllers get static IPs, and point DNS at themselves.** Non-negotiable.
- **Don't put AD CS (PKI) on a domain controller.** Best practice is a dedicated member server, and a real design uses a two-tier PKI (offline root CA + issuing subordinate CA). Build the domain first, then the PKI on top.

---
