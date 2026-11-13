Up until this point, my management network was the default at 192.168.88.0/24 on VLAN 1. It worked well so far but there is a specific reason I am moving it. The issue is that VLAN 1 is where all traffic lands by default. Say in the future I forget to configure a port, or I plug in a new device. This device will end up on the same network as my ESXi host with absolutely no firewall rules governing it. This is why Cisco and the CCNA say not to use VLAN 1 for management. 

In this session we will be moving management to VLAN99. It'll get its own subnet to match the rest of the lab (10.10.99.0/24), and it'll be tagged just like everything else. This will allow firewall rules to be setup so my laptop and Tailscale can reach ESXi but nothing else can. Another step towards the default deny structure planned.

### Step 1 - VLAN 99 
The first step we need to take is building a new VLAN on the switch. VLAN 99 interface will be the routers connection into VLAN 99. When we set the interface as bridge we are allowing the interface to watch for frames tagged 99, strip the tag, and hand it over to the router. Vise versa when the router sends something going out on vlan99, tag it, and push to the bridge.

**Interfaces > VLAN > New**
Name: vlan99
VLAN ID: 99
Interface: bridge

### Step 2 - Add an IP
We need to give the VLAN a gateway address like (10.10.99.1). This is where every device on VLAN 99 will send traffic to if the destination is outside of its own subnet. The switch will take the traffic to the right VLAN or out to the internet.

**IP > Addresses > New**
Address: 10.10.99.1/24
Network: 10.10.99.0
Interface: vlan99

### Step 3 - Bridge VLAN Table Entry
At this moment we now have an interface which gives the router an address on VLAN 99. However this still doesn't tell the bridge that VLAN 99 exists or what ports belong to it. This ensures the bridge will know which ports are in VLAN 99, in turn allowing it to receive traffic. 

**Bridge > VLANs > New**
Bridge: bridge
VLAN ID's: 99
Tagged: bridge
Untagged: ether2(copper cable -> MS-01 running ESXi), ether4(Copper cable -> Docking station -> Personal Laptop)

Before moving on I want to reinforce why certain interfaces/ports are tagged or untagged. Lets think...
Starting off with the bridge and why its tagged. A tag is how the router knows which VLAN a frame belongs to so it knows the proper interface to send it to. The bridge being tagged allows the bridge to send VLAN-labeled packets to the router. 

**Note to remember:** Putting in an interface in the tagged column means "do not strip the tag when crossing this port"

Second, we have untagged featuring ether 2 and ether 4 which will be on VLAN 99. The key point here is that my laptop and the ESXi management interface don't know anything about VLAN's. **They only know to receive and send untagged frames.** The devices will be on VLAN 99 but wont ever see a tag as it needs to be stripped off before receiving it. 

### Step 4 - Firewall Rules
Now that VLAN 99 exists, I want the firewall ready before anything actually moves onto it. Nothing lives on 10.10.99.0/24 yet besides the gateway, and traffic to the gateway goes through the input chain, not forward. So these rules just sit there until devices move over, which makes this safe to do now.

While looking at my rules I found something I didn't realize. MikroTik's forward chain accepts anything that doesn't match a rule. I had drops for specific pairs (20 to 10, 20 to 30, etc.) but nothing stopping anyone from reaching management. So CLIENT01 could pull up the ESXi web UI. No rule allowed it, nothing blocked it either.

I added three rules:

**IP > Firewall > Filter Rules > New**

```
Chain:        forward
Src. Address: 10.10.10.20
Dst. Address: 10.10.99.10
Protocol:     tcp
Dst. Port:    443,902
Action:       accept
Comment:      Tailscale: docker01 to ESXi mgmt
```

**IP > Firewall > Filter Rules > New**

```
Chain:        forward
Dst. Address: 10.10.99.0/24
Action:       drop
Comment:      Drop all other to mgmt
```

**IP > Firewall > Filter Rules > New**

```
Chain:        forward
Src. Address: 10.10.99.0/24
Action:       accept
Comment:      Mgmt to all
```

Then I dragged them right under the established/related rule so the order is:

```
0   accept established/related
1   accept docker01 -> ESXi
2   drop   anything -> mgmt
3   accept mgmt -> anything
4+  everything that was already there
```

Order matters here because MikroTik reads top to bottom and the first match wins. The docker01 allow has to be above the drop or Tailscale gets blocked. All three sit above my ICMP rules too, otherwise clients could still ping management.

Lets think about the replies though... if my laptop RDPs into DC01, DC01's reply is headed into VLAN 99. Why doesn't the drop catch it? Because rule 0 already let it through as part of a conversation my laptop started. So management can reach out to anything, but nothing can start a conversation with it.

> **Note to remember:** "Mgmt to all" technically already happens since the default is accept. I added it anyways so the policy is written down in the firewall, and so it keeps working once I add a final drop for full default deny.

**Mistake I caught:** the first time, I put 10.10.99.0/24 in the Src. Address column on the drop rule instead of Dst. That would have dropped everything _coming from_ management, so my laptop couldn't reach anything once it moved. Caught it before moving anything.

I also removed the old docker01 to 192.168.88.10 rule since ESXi was moving.

## Step 5 - Moving My Laptop

I was originally going to move ESXi first, but once the firewall was in it made more sense to move the laptop first. If ESXi moved first, my laptop would still be on 192.168.88.x and would get caught by the "drop all other to mgmt" rule. Laptop first means it's already on VLAN 99, so it can still reach ESXi at the old IP through "Mgmt to all".

> **Note to remember:** Safe Mode OFF for this one. Moving my own port is guaranteed to drop WinBox, and Safe Mode would just undo the change.

**Bridge > Ports > ether4**

```
PVID: 99
```

WinBox disconnected (expected). Then on the laptop:

```
IP:      10.10.99.20
Mask:    255.255.255.0
Gateway: 10.10.99.1
DNS:     <!-- DC01 (10.10.10.10) or 8.8.8.8, and why -->
```

Reconnected WinBox to 10.10.99.1.

Quick note on PVID since it confused me. Untagged membership in the VLAN table only covers frames going **out** of the port. The PVID decides which VLAN untagged frames coming **in** get put into. An access port needs both. That's why nothing moved in Step 3. The PVID change is the moment a device actually moves.

**Tested:**

- `ping 10.10.99.1` (gateway) - worked
- `ping 10.10.10.10` (DC01) - worked
- `https://192.168.88.10` (ESXi at its old IP) - still loaded
- **Bridge > Hosts:** laptop MAC `AC:91:A1:98:19:76` on ether4, VID 99

## Step 6 - Moving ESXi

The plan was to do this from the DCUI on the MS-01 but plugging in HDMI gave me no signal. Most likely because the host booted without a monitor, so the iGPU never turned on the display. I didn't want to reboot with all my VMs running, so I had to do it remotely.

This is where things went sideways. Changing vmk0's IP in the ESXi web UI just gave me **"Failed to save configuration changes."** From what I can tell, ESXi won't let you set a management IP that can't reach the current default gateway (192.168.88.1). And I couldn't change the gateway first because 10.10.99.1 wasn't reachable from the old IP. Each one was blocked by the other.

Somewhere in the middle of that I also changed things back the wrong way and lost the host at both IPs. Lets think about why... the IP and the port's PVID have to match:

|ESXi IP|ether2 PVID|Reachable?|
|---|---|---|
|10.10.99.10|99|yes|
|192.168.88.10|1|yes (routed)|
|10.10.99.10|1|no|
|192.168.88.10|99|no (where I ended up)|

Set ether2 back to PVID 1, pinged 192.168.88.10 from the switch terminal to confirm, and started over.

What actually worked was setting the IP and gateway together over SSH in one command:

**ESXi Web UI > Host > Actions > Services > Enable Secure Shell (SSH)**

```
ssh root@192.168.88.10
```

```
esxcli network ip interface ipv4 set -i vmk0 -t static -I 10.10.99.10 -N 255.255.255.0 -g 10.10.99.1 ; esxcfg-route 10.10.99.1
```

The first part sets the new IP and gateway on vmk0. The second sets the host's default gateway. Having them on one line means both run even when the SSH session hangs, which it did (expected).

Then on the switch:

**Bridge > Ports > ether2**

```
PVID: 99
```

```
/ping 10.10.99.10
```

Got replies, then logged in at `https://10.10.99.10`.

**ESXi Web UI > Networking > TCP/IP stacks > Default TCP/IP stack**

```
Gateway: 10.10.99.1
DNS:     10.10.10.10
```

Turned SSH back off after.

> **Note to remember:** The Management Network port group on vSwitch0 stays at VLAN 0. ether2 is an untagged access port, so frames show up at ESXi with no tag. If I set the port group to 99, ESXi would expect tagged frames and drop everything. Both sides have to agree on whether the cable carries tags.

## Step 7 - Tailscale

Remote access to ESXi broke after the move. docker01 only routes the subnets it advertises, and 10.10.99.0/24 wasn't one of them.

```
sudo tailscale set --advertise-routes=<!-- full list -->,10.10.99.0/24
```

> **Note to remember:** `--advertise-routes` replaces the whole list. If I only typed the new subnet, I'd lose remote access to everything else.

**Tailscale Admin Console > Machines > docker01 > Edit route settings** → approve 10.10.99.0/24

No firewall change was needed since rule 1 already allows docker01 to ESXi. docker01 rewrites the source address (SNAT), so the MikroTik only ever sees 10.10.10.20, never my phone.

Tested from my phone on cellular with Tailscale on: `https://10.10.99.10` - <!-- result -->

## Step 8 - Cleanup

Backed up the config first:

```
/export hide-sensitive file=pre-cleanup
```

**Files** → dragged `pre-cleanup.rsc` to my laptop and committed it to the repo.

- Disabled unused ports (ether5-8)
- Disabled the default DHCP server on the bridge
- Disabled 192.168.88.1 on the bridge (disabled, not removed, in case something depends on it)
- Removed 192.168.88.0/24 from Tailscale's advertised routes

I decided not to keep a rescue port on the old network. The config export is my way back if I ever lock myself out.

## Testing

| Test                                                | Expected         | Result                    |
| --------------------------------------------------- | ---------------- | ------------------------- |
| Laptop → `https://10.10.99.10`                      | loads            | worked                    |
| Laptop → DC01, NAS, internet                        | works            | worked                    |
| CLIENT01 → `https://10.10.99.10`                    | times out        | no connection as expected |
| Phone (Tailscale, cellular) → `https://10.10.99.10` | loads            | worked                    |
| Bridge > Hosts: ESXi MAC `38:05:25:3A:72:FF`        | VID 99 on ether2 | worked                    |

## Things I Found Along the Way

- **The input chain has no drop at all,** so WinBox is reachable from every VLAN. CLIENT01 could open 10.10.20.1 and get a login prompt. The docker01 WinBox rule doesn't actually do anything yet.
- **Mystery MAC `38:05:25:3A:72:FD`** is showing up untagged on VLAN 1 over the SFP+ trunk. It's one number off the ESXi management MAC, so probably another MS-01 NIC. Something on the ESXi side is sending untagged frames down the trunk.
- **The SFP+ is tagged on VLAN 30** but the NAS is physical on ether3. Does anything on ESXi actually need VLAN 30?

## Next

- Full default deny: final drop in forward, lock down the input chain (WinBox only from VLAN 99 and docker01)
- Figure out the untagged traffic on the trunk (look into `frame-types` on the SFP+ port)
- DNS A record for `esxi01.landon.lab` pointing to 10.10.99.10










