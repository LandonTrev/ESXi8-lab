## Goal

Lock down the client -> NAS path (VLAN 20 -> 30) and finish turning the inter-VLAN forward chain into default deny, without breaking logons, Group Policy, or the file share.
## What I did

- Traced the full client -> file share sequence (DHCP, SRV, Kerberos, A record, SMB) and confirmed it in Wireshark on a client VM. SMB landed on the NAS at 10.10.30.100 on TCP 445.
- Found that 20 -> 30 worked with no rules because MikroTik accepts forwarded traffic that matches nothing.
- Built allow + drop blocks for 20 -> 30 and 30 -> 10, on top of the existing 20 -> 10 block.
- Added an ICMP accept so clients can ping for troubleshooting. This changes the earlier note: a failed ping is no longer the expected result.
- Checked the AD groups: HR-Staff, IT-Staff, and Sales-Staff are nested in FileShare-Dept-RW (AGDLP).
- Got Pi-hole back up after the docker01 reboot.
- Decided to hold off on NFS/iSCSI datastores for ESXi until I understand them better.

## Rule set as built (forward chain)

|   |   |   |   |   |   |   |
|---|---|---|---|---|---|---|
|#|Action|Src|Dst|Protocol|Ports|Purpose|
|0|accept|any|any|—|—|Established/related|
|1|accept|10.10.20.0/24|10.10.10.0/24|TCP|53, 88, 135, 389, 445, 464, 636, 3269, 49152-65535|Clients to AD|
|2|accept|10.10.20.0/24|10.10.10.0/24|UDP|53, 88, 123, 389, 464|Clients to AD|
|3|accept|10.10.20.0/24|any|ICMP|—|Client ping|
|4|drop|10.10.20.0/24|10.10.10.0/24|—|—|Wall: 20 -> 10|
|5|accept|10.10.20.0/24|10.10.30.0/24|TCP|445|SMB to NAS|
|6|drop|10.10.20.0/24|10.10.30.0/24|—|—|Wall: 20 -> 30|
|7|accept|10.10.30.0/24|10.10.10.0/24|TCP|135, 389, 445, 464, 49152-65535|NAS as domain member|
|8|accept|10.10.30.0/24|10.10.10.0/24|UDP|123, 389, 464|NAS as domain member|
|9|drop|10.10.30.0/24|10.10.10.0/24|—|—|Wall: 30 -> 10|

Still open: 10 -> 30 has no rules, so it falls through to the default accept. Rules 7 and 8 are still missing 53 and 88 (see Next session).

## What I learned

- No rule = allowed on MikroTik's forward chain. A wall has to be built on purpose.
- RPC (135 + 49152-65535). Port 135 is the Endpoint Mapper, like a front desk. The client asks where a service lives, and the DC answers with a random high port. That's why the whole dynamic range has to be open. `netsh int ipv4 show dynamicport tcp` on the DC confirmed my rule matches.
- ICMP has no ports. It's its own protocol at Layer 3, filtered by type (echo-request = type 8).
- Stateful in action: the DC can ping the NAS even with a 30 -> 10 drop. The reply rides the established/related rule because the firewall already approved the request.
- Anything domain-joined needs about the same ports to the DC. That's why the NAS rules mirror the client rules.
- gpupdate is a network test. It pulls policy from SYSVOL over SMB, plus DNS and LDAP, so it proves rule 1 is complete.

## How I tested

- `gpupdate /force` on the client: works.
- Pinged the DC and NAS from a client: works (ICMP rule).
- Checked packet counters: accepts climb, drops stay near zero. A drop that keeps climbing means something needed is missing.

## Pi-hole after reboot

Rebooting docker01 left Pi-hole stopped, which breaks the DC's upstream DNS forwarding.

- `docker ps -a` shows stopped containers too
- docker compose up -d` from the compose folder
- A restart policy (`unless-stopped`) brings it back on its own

