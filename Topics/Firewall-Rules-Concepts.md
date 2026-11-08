# Firewall Rules - Core Concepts

Reference for how MikroTik (RouterOS) firewall filter rules work. Written from the inter-VLAN lab build, but the model is general.

## A rule is an IF -> THEN

Every firewall rule is one sentence:

> IF a packet looks a certain way -> THEN allow it (accept) or block it (drop).

The General tab describes WHICH packets (the IF). The Action tab is the verb (the THEN).

## The four boxes that describe a packet

You narrow down which packets a rule applies to by filling boxes. A packet must match ALL filled boxes for the rule to apply.

1. Chain - which direction of traffic
2. Src. Address - coming from where
3. Dst. Address - going to where
4. Protocol / Dst. Port - which "door" (service) it is knocking on

Analogy: describing a car as "a red Toyota heading north." Each detail narrows it further.

## Chains (the "direction" box)

- input - traffic aimed AT the router itself (WinBox sessions, pinging the router's own IP).
- output - traffic the router itself generates. Rarely touched.
- forward - traffic PASSING THROUGH the router between networks. Client-to-server traffic across VLANs is forward.

Safety note: forward rules cannot lock you out of the switch, because your management session is input-chain traffic. Hardening the input chain is the session that carries real lockout risk - be careful there.

## First match wins

Rules are read top to bottom. The first rule a packet matches decides its fate, and nothing below matters for that packet. So ORDER is everything.

Standard design pattern:
1. Accept established/related (handles most packets cheaply)
2. Accept the specific new connections you bless (e.g. clients to servers on AD ports)
3. Drop everything else (the default-deny floor)

Because rules 1 and 2 are pure accepts, nothing is actually blocked until the drop rule goes in at the bottom. This lets you build safely and only "turn the key" as the final step.

## The established/related rule (the tricky one)

Core idea: conversations have TWO directions.

- The question travels one way (e.g. client -> server).
- The answer travels back the other way (server -> client).

The problem: rules that allow traffic going 20 -> 10 do not allow the reply, which travels 10 -> 20. Without help, every conversation would be one-way and useless.

The solution: the firewall remembers conversations it has already approved the start of (connection tracking). When a reply comes back, it recognizes it as part of an already-approved conversation and lets it through.

- "established" = traffic belonging to a conversation already in progress.
- "related" = a legitimate side-channel spawned by an approved conversation.

Plain-English rule:
> If this packet is part of a conversation I already approved the start of, wave it through without re-checking.

Why it is first: most traffic on any network is replies and ongoing chatter, so this one rule handles the majority of packets instantly. Only brand-new connection attempts get inspected by the door rules below.

Security payoff: this rule only allows replies to conversations the CLIENT started. The server can answer all day but can never START a conversation into client land uninvited. That asymmetry (clients initiate, servers respond, servers never initiate) comes for free.

## The drop rule (the wall)

- Chain forward, Src client subnet, Dst server subnet, protocol and ports left BLANK.
- Blank = match anything. So this rule catches every packet from clients to servers that did not match an allow rule above it.
- Must sit LAST. If it lands above the accepts, first-match-wins means it blocks everything, including logins.

## Testing a wall

- Test something that SHOULD work (an allowed service) - it should succeed.
- Test something that should NOT work (a service not on the allow list, e.g. ICMP ping) - it should fail. A failed ping is the good result: it proves the drop rule is doing its job.

Learned in: [[Sessions/2026-07-16 Client Join and Inter-VLAN Firewall]]
See also: [[Topics/AD-Ports-and-Services]]
