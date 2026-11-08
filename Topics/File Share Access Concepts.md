Reference for what actually happens when a domain client opens a file share on the NAS. Written from the default-deny firewall session, but the model is general to any AD domain.

## The big idea

The domain does NOT hand out the files. It's easy to think the client gets the share "through the domain," but the jobs are split:

- **The DC finds and proves.** It tells the client where things are (DNS) and who the user is (Kerberos).
- **The NAS serves.** The actual file traffic goes straight from the client to the NAS.

## Step by step

A client on VLAN 20 opening `\\nas\share`:

1. **DHCP (20 -> 10, via relay).** Client gets its IP, subnet mask, default gateway, DNS server (the DC), domain name, and lease. Without this, nothing else can start.
2. **SRV lookup (20 -> 10).** "Where's the DC for this domain?"
3. **Kerberos at logon (20 -> 10).** Client gets a ticket proving who the user is and which groups they're in.
4. **A record lookup (20 -> 10).** "What's the IP of `nas`?" Computers never send to names, only IPs.
5. **Kerberos ticket for the NAS (20 -> 10).** A second ticket, just for the file service.
6. **SMB (20 -> 30).** File traffic goes straight to the NAS on TCP 445. The NAS checks the groups in the ticket against its ACL.

Steps 1-3 happen at boot/logon. Steps 4-6 happen when the share is opened.

The only step that touches the NAS is step 6. Everything else goes to the DC, so the firewall only needs **445** from clients to the NAS.

## DHCP relay

DHCP Discover is a broadcast, and broadcasts don't cross VLANs. The DHCP server lives on the DC (VLAN 10), but the clients are on VLAN 20.

The fix: a **DHCP relay** on the VLAN 20 interface of the MikroTik catches the broadcast and forwards it as unicast to 10.10.10.10. That's how the request crosses from 20 to 10.

The option that matters most is the **DNS server**. DHCP hands out the DC's IP as DNS, which is why every lookup after that goes to the DC.

## Record types: SRV vs A

- **A record** = name -> IPv4 address. (AAAA is the IPv6 version.)
- **SRV record** = service -> host. Instead of "what's the IP of X," it asks "who provides this service?"

The client knows its domain name but not which machine is the DC. So it asks DNS for the SRV record for LDAP in that domain, and DNS answers with the DC's hostname and port. Then a normal A lookup turns that hostname into an IP.

The DC registers its own SRV records when it's promoted, so I never had to create them.

Check it: `nslookup -type=SRV _ldap._tcp.dc._msdcs.<domain>`

## Where permissions actually live

Three different things, easy to mix up:

- **Security groups (AD)** = who you are. "Landon is in IT-Staff."
- **Group Policy (AD)** = can _map_ the drive for you. It does not grant access to the files.
- **ACL (on the NAS)** = the actual rule. "FileShare-Dept-RW can read/write this dataset." The NAS checks it.

The Kerberos ticket carries identity and group membership, NOT permissions. The NAS compares the groups in the ticket against its ACL.

## AGDLP

The standard way to organize it:

- **A**ccounts go into **G**lobal groups (HR-Staff, IT-Staff, Sales-Staff = who people are)
- Global groups go into **D**omain **L**ocal groups (FileShare-Dept-RW = access to a resource)
- **P**ermission is granted to the Domain Local group, in the ACL on the resource

AD holds the membership. The resource holds the rule.

Why bother: when someone changes departments, you change one group membership and every permission follows. Nobody touches the ACLs.

Design gotcha: one RW group for every department means every department can open every department's folder. Separate access needs separate Domain Local groups and separate ACLs.

## Seeing it in Wireshark

On a client:

1. `ipconfig /flushdns` (otherwise the name is cached and the DNS query never shows)
2. Start a capture, then open the share
3. Display filter: `dns || kerberos || tcp.port == 445`

Watch the source and destination IPs. DNS and Kerberos go to the DC. The 445 traffic lands on the NAS.

Time matters: Kerberos fails if clocks are more than about 5 minutes apart, so every domain member needs NTP from the DC.

Learned in: Sessions/2026-11-08 Default-Deny Firewall and Share Flow See also: Topics/Firewall 