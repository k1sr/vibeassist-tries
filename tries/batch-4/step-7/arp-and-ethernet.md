# Ethernet Framing, Switching and ARP: A Worked Explainer

To cross one hop of a LAN, an IP datagram is wrapped in an Ethernet frame, a switch picks the port it leaves on, and the sender must first learn which MAC address belongs to the next-hop IP. That last step is ARP. This explainer works through all three with concrete bytes, then covers VLAN tags and ARP spoofing.

---

## 1. Addresses: MAC vs IP

An Ethernet interface has a 48-bit **MAC address**, written as six hex octets such as `00:1a:2b:00:00:0a`. Two bits of the first octet matter:

- **I/G bit** (least significant bit of the first octet): `0` = individual (unicast), `1` = group (multicast/broadcast). `01:00:5e:...` is an IPv4 multicast address. `ff:ff:ff:ff:ff:ff` is broadcast.
- **U/L bit** (second least significant bit): `0` = universally administered (vendor-assigned, with the first three octets forming the OUI), `1` = locally administered. `02:...` addresses are local, which is why virtual machines and randomised Wi-Fi MACs often start with `02`, `06`, `0a` and so on.

Ethernet sends each octet least-significant bit first, so the I/G bit is literally the first address bit on the wire. A switch can know a frame is multicast or broadcast after one bit.

An IP address identifies a host's position in a routed network. A MAC address identifies an interface on one link. **IP addresses survive end to end. MAC addresses are rewritten at every router hop.** Keep that asymmetry in mind and most of ARP explains itself.

For the rest of this document we use this LAN:

| Host | IP | MAC | Switch port |
|---|---|---|---|
| A | 192.168.1.10 | 00:1a:2b:00:00:0a | 1 |
| B | 192.168.1.20 | 00:1a:2b:00:00:0b | 2 |
| C | 192.168.1.30 | 00:1a:2b:00:00:0c | 3 |
| R (router, LAN side) | 192.168.1.1 | 00:1a:2b:00:00:01 | 4 |

The subnet is 192.168.1.0/24, and the default gateway on every host is 192.168.1.1.

---

## 2. The Ethernet II Frame

### 2.1 Layout

On the wire, a frame looks like this:

```
+----------+-----+---------+---------+-----------+----------------+-----+
| Preamble | SFD | Dst MAC | Src MAC | EtherType | Payload        | FCS |
| 7 bytes  | 1   | 6       | 6       | 2         | 46 - 1500      | 4   |
+----------+-----+---------+---------+-----------+----------------+-----+
             ^-- the "frame" proper starts at Dst MAC ---------------^
```

- **Preamble** (`55 55 55 55 55 55 55`) is an alternating 1010... pattern. It lets the receiver's clock lock on.
- **Start Frame Delimiter** (`d5`) ends the pattern with `...11`, which marks where the frame begins.
- **Destination MAC** comes first so that hardware can start filtering early.
- **Source MAC** is the sender's interface address.
- **EtherType** names the payload protocol: `0x0800` for IPv4, `0x0806` for ARP, `0x86dd` for IPv6 and `0x8100` for an 802.1Q VLAN tag. A value of 1500 (`0x05dc`) or less in this position means an IEEE 802.3 *length* field instead, followed by an LLC header. Values of `0x0600` and above are types. That is how both formats share a wire.
- **Payload** must be at least 46 bytes. A shorter payload is padded with zeros. It may be at most 1500 bytes, which is the standard MTU.
- **FCS** (Frame Check Sequence) is a CRC-32 over destination through payload. A receiver that computes a different CRC silently drops the frame. Ethernet has no retransmission. That is left to higher layers.

The minimum frame (destination through FCS) is therefore 6+6+2+46+4 = **64 bytes**, and the maximum is 6+6+2+1500+4 = **1518 bytes**, or 1522 with a VLAN tag. The 64-byte minimum comes from half-duplex CSMA/CD. A frame had to last long enough for the sender to still be transmitting when a collision from the far end of a maximum-length segment came back. Full-duplex switched Ethernet has no collisions, but the minimum was kept for compatibility. Between frames, the sender must also leave an **interframe gap** of 12 byte-times. Preamble, SFD and gap together cost 20 bytes of overhead per frame that never appear in a packet capture.

### 2.2 A real frame, byte by byte

Here is the ARP request that host A broadcasts to find B (ARP itself is explained in Section 4). It is shown as a capture tool would display it, without preamble and SFD:

```
Offset  Bytes                                             Meaning
0000    ff ff ff ff ff ff                                 Dst MAC: broadcast
0006    00 1a 2b 00 00 0a                                 Src MAC: host A
000c    08 06                                             EtherType: ARP
000e    00 01                                             ARP HTYPE: Ethernet (1)
0010    08 00                                             ARP PTYPE: IPv4
0012    06                                                HLEN: 6
0013    04                                                PLEN: 4
0014    00 01                                             OPER: 1 = request
0016    00 1a 2b 00 00 0a                                 Sender HW addr (A)
001c    c0 a8 01 0a                                       Sender IP 192.168.1.10
0020    00 00 00 00 00 00                                 Target HW addr (unknown)
0026    c0 a8 01 14                                       Target IP 192.168.1.20
002a    00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00   Padding (18 bytes)
003a    00 00
003c    xx xx xx xx                                       FCS (CRC-32)
```

Count the pieces. The header is 14 bytes. The ARP message is 28 bytes, which is below the 46-byte minimum, so the NIC adds 18 bytes of zero padding. The FCS adds 4, for a total of exactly 64 bytes. Captures taken on the sending host often show a 42-byte frame. The capture point sits above the NIC, which adds the padding and FCS later.

The IPv4 addresses are plain big-endian bytes: 192 = `c0`, 168 = `a8`, 1 = `01`, 10 = `0a`, 20 = `14`.

---

## 3. How a Switch Forwards Frames

A switch is a multi-port **transparent bridge**. Hosts do not know it exists. They address frames to each other, and the switch works out where to send them by watching traffic. It keeps a **MAC address table** (also called the CAM table or forwarding database) that maps MAC address and VLAN to a port, along with an age.

The algorithm, applied to every frame that arrives on port *P*:

1. **Learn.** Record *source MAC → P* and reset that entry's age timer. If the MAC was previously on another port, move it, because the host has moved.
2. **Forward.**
   - If the destination is broadcast or an unknown multicast, **flood** the frame out every port in the VLAN except *P*.
   - If the destination is unicast and in the table, send it out that one port. If that port is *P* itself, **filter** (drop) the frame. The destination is on the segment it came from.
   - If the destination is unicast and *not* in the table, **flood** it. This is called "unknown unicast flooding."
3. **Age.** Entries that have not been refreshed for the aging time (300 seconds is the common default) are removed.

### 3.1 MAC-learning trace

Start with an empty table on our 4-port switch and follow this sequence.

**Event 1: A sends the ARP request (broadcast) for B.**

- The frame arrives on port 1 with source `..:0a`. The switch learns `0a → 1`.
- The destination is `ff:ff:ff:ff:ff:ff`, so it floods out ports 2, 3 and 4.

| MAC | Port | Age |
|---|---|---|
| 00:1a:2b:00:00:0a | 1 | 0 s |

B, C and R all receive the frame. C and R see that the target IP is not theirs and discard it.

**Event 2: B sends the ARP reply (unicast) to A.**

- The frame arrives on port 2 with source `..:0b`. The switch learns `0b → 2`.
- The destination `..:0a` is in the table at port 1, so the frame is forwarded out port 1 only.

| MAC | Port | Age |
|---|---|---|
| 00:1a:2b:00:00:0a | 1 | 0 s |
| 00:1a:2b:00:00:0b | 2 | 0 s |

**Event 3: A sends an IPv4 packet to B.**

- The frame arrives on port 1. A's entry is refreshed.
- The destination `..:0b` is known at port 2, so the frame is forwarded out port 2 only. C and R never see it.

**Event 4: C sends a unicast frame to R. Neither has transmitted yet.**

- The frame arrives on port 3. The switch learns `0c → 3`.
- The destination `..:01` is unknown, so the frame floods to ports 1, 2 and 4. A and B receive it, see that the destination MAC is not theirs, and their NICs drop it in hardware.

**Event 5: R replies to C.**

- The switch learns `01 → 4` and forwards out port 3 only.

| MAC | Port | Age |
|---|---|---|
| 00:1a:2b:00:00:0a | 1 | ~0 s |
| 00:1a:2b:00:00:0b | 2 | ~0 s |
| 00:1a:2b:00:00:0c | 3 | 0 s |
| 00:1a:2b:00:00:01 | 4 | 0 s |

**Event 6: B is unplugged from port 2 and plugged into port 3 (via a small hub alongside C), then sends a frame.**

- The frame arrives on port 3 with source `..:0b`. The table said port 2, so the switch updates it to `0b → 3`. Until B transmits, frames for B still go to the dead port 2. That is why hosts send a gratuitous ARP after a link comes up (Section 4.4). It refreshes switch tables as well as ARP caches.

Two consequences follow:

- **A switch never "discovers" silent hosts.** It learns only from source addresses. A printer that never transmits causes every frame sent to it to be flooded.
- **The table is finite.** Fill it with millions of fake source MACs (*MAC flooding*) and real entries get evicted. The switch then floods everything, which turns it into a hub for an eavesdropper. **Port security**, which limits the number of MACs per port, is the defence.

Loops are the other hazard: Ethernet has no TTL, so the **Spanning Tree Protocol** blocks redundant links.

---

## 4. ARP: Turning IP Addresses into MAC Addresses

### 4.1 The packet format

ARP (RFC 826) runs directly over Ethernet with EtherType `0x0806`. It is not carried in IP. The message has nine fields:

| Field | Size | Ethernet/IPv4 value |
|---|---|---|
| HTYPE (hardware type) | 2 | `0x0001` |
| PTYPE (protocol type) | 2 | `0x0800` |
| HLEN | 1 | 6 |
| PLEN | 1 | 4 |
| OPER | 2 | 1 = request, 2 = reply |
| SHA (sender hardware address) | 6 | sender MAC |
| SPA (sender protocol address) | 4 | sender IP |
| THA (target hardware address) | 6 | target MAC (zeros in a request) |
| TPA (target protocol address) | 4 | target IP |

The design is generic. HTYPE and PTYPE let the same packet resolve other protocol and hardware combinations, although IPv4 over Ethernet is the only one most people meet.

### 4.2 The ARP cache

Every host keeps an **ARP cache**, a table of *IP → MAC* with a lifetime. On Linux you can see it with `ip neigh`, and on Windows or macOS with `arp -a`:

```
$ ip neigh
192.168.1.1  dev eth0 lladdr 00:1a:2b:00:00:01 REACHABLE
192.168.1.20 dev eth0 lladdr 00:1a:2b:00:00:0b STALE
```

Linux keeps an entry REACHABLE for roughly 30 seconds after confirmation, then STALE (still used, but re-verified). If no entry exists, the host queues the packet, sends a request, and reports "host unreachable" after a few unanswered retries.

### 4.3 Worked example: A talks to B on the same LAN

A runs `ping 192.168.1.20`.

**Step 1: the routing decision.** A compares the destination against its own interface: `192.168.1.20 AND 255.255.255.0 = 192.168.1.0`, the same as its own network. The destination is **on-link**, so the next hop is B itself.

**Step 2: cache lookup.** A has no entry for 192.168.1.20.

**Step 3: ARP request.** A broadcasts the frame shown byte by byte in Section 2.2. In words: *"Who has 192.168.1.20? Tell 192.168.1.10 (00:1a:2b:00:00:0a)."* The switch floods it.

**Step 4: receivers process it.** Under RFC 826, every receiver that already has a cache entry for the sender's IP refreshes it using SHA and SPA. Only the host whose IP equals TPA, which is B, does more:

- B adds `192.168.1.10 → 00:1a:2b:00:00:0a` to its own cache. It expects to reply to A soon, so learning A now saves a second ARP exchange.
- B sends a unicast reply.

**Step 5: ARP reply.** B swaps the roles of the fields:

```
0000  00 1a 2b 00 00 0a   Dst MAC: A (unicast)
0006  00 1a 2b 00 00 0b   Src MAC: B
000c  08 06               EtherType: ARP
000e  00 01 08 00 06 04   HTYPE, PTYPE, HLEN, PLEN
0014  00 02               OPER: 2 = reply
0016  00 1a 2b 00 00 0b   SHA: B's MAC   <- the answer
001c  c0 a8 01 14         SPA: 192.168.1.20
0020  00 1a 2b 00 00 0a   THA: A's MAC
0026  c0 a8 01 0a         TPA: 192.168.1.10
002a  (18 bytes padding) + FCS
```

**Step 6: A caches and transmits.** A stores `192.168.1.20 → 00:1a:2b:00:00:0b`, releases the queued ICMP Echo Request, and sends it in a frame with destination `..:0b`, source `..:0a` and EtherType `0x0800`. The first ping usually takes slightly longer than the rest because of this exchange.

### 4.4 Worked example: A talks to a server off the LAN

Now A runs `ping 203.0.113.50`, a host on the Internet.

**Step 1: the routing decision.** `203.0.113.50 AND 255.255.255.0 = 203.0.113.0`, which is not A's network. The best route is the default route, `0.0.0.0/0 via 192.168.1.1`. The next hop is the **router**, not the destination.

This is the step that confuses people most. **A never ARPs for 203.0.113.50.** ARP only works within a broadcast domain, and the server is not in A's. A ARPs for the *gateway's* IP instead.

**Step 2: ARP for the gateway.** A broadcasts "Who has 192.168.1.1? Tell 192.168.1.10." R replies with `00:1a:2b:00:00:01`. Hosts send traffic through the gateway constantly, so this entry is almost always already in the cache.

**Step 3: the frame to the router.** A builds:

```
Ethernet: dst 00:1a:2b:00:00:01 (R)   src 00:1a:2b:00:00:0a (A)   type 0800
IPv4:     src 192.168.1.10            dst 203.0.113.50           TTL 64
```

The *Ethernet* destination is the router, while the *IP* destination is the far server. The packet is addressed to the server, but the frame is delivered to the router.

**Step 4: the router forwards.** R receives the frame, because the destination MAC is its own. It strips the Ethernet header, verifies the IP checksum, decrements the TTL to 63, recomputes the checksum and looks up 203.0.113.50 in its routing table. Suppose the next hop is the ISP's router at 198.51.100.1 on R's WAN interface. R checks *its* ARP cache for 198.51.100.1 (ARPing if needed) and builds a **brand-new frame**:

```
Ethernet: dst <ISP router MAC>   src <R's WAN MAC>   type 0800
IPv4:     src 192.168.1.10       dst 203.0.113.50    TTL 63
```

In a typical home network, R would also apply NAT and rewrite the source IP to its public address. NAT is a separate mechanism. Without NAT, the IP addresses are unchanged from end to end and only the TTL and checksum differ. **The MAC addresses are entirely different at every hop.** Each hop repeats ARP on its own link, until the last router ARPs for 203.0.113.50 itself on the server's LAN.

**Replies come back the same way.** When the Echo Reply arrives, R looks up 192.168.1.10, finds it on its LAN interface, takes A's MAC from its ARP cache and delivers the frame.

### 4.5 Gratuitous ARP and other variants

- **Gratuitous ARP**: a host announces itself (SPA = TPA = its own IP) at link-up or failover, to detect address conflicts and to update neighbours' caches and switch tables.
- **ARP probe** (RFC 5227): a request with SPA `0.0.0.0`, sent before using a new address, which pollutes no caches.
- **Proxy ARP**: a router answers for addresses behind it with its own MAC. It hides misconfiguration and is usually disabled.
- **IPv6** uses Neighbor Discovery (ICMPv6) instead of ARP, with the same trust problems.

---

## 5. VLANs and 802.1Q Tagging

A **VLAN** splits one physical switch into several isolated broadcast domains. Ports in VLAN 10 and ports in VLAN 20 cannot exchange frames at layer 2. Broadcasts, including ARP requests, stay inside the VLAN, and traffic between VLANs must go through a router or a layer-3 switch. Because each VLAN is a separate broadcast domain, it is normally also a separate IP subnet.

### 5.1 Access ports and trunk ports

- An **access port** belongs to one VLAN. The host attached to it sends ordinary untagged frames and has no idea VLANs exist. The switch adds the VLAN association internally.
- A **trunk port** carries several VLANs, usually between switches or to a router or hypervisor. Frames on a trunk carry an explicit **802.1Q tag** saying which VLAN they belong to. One VLAN per trunk, the **native VLAN**, can be sent untagged.

The MAC table is really keyed on *(VLAN, MAC)*. The same MAC can legitimately appear in two VLANs on different ports, and flooding is confined to ports that are members of the frame's VLAN.

### 5.2 The tag format

The 4-byte tag is inserted between the source MAC and the EtherType:

```
+---------+---------+------------------+-----------+---------+-----+
| Dst MAC | Src MAC | 802.1Q tag       | EtherType | Payload | FCS |
| 6       | 6       | TPID 2 | TCI 2   | 2         | 42-1500 | 4   |
+---------+---------+------------------+-----------+---------+-----+

TCI (16 bits):  | PCP (3) | DEI (1) | VID (12) |
```

- **TPID** = `0x8100`. It sits where the EtherType would be, so a device that does not understand VLANs sees an unknown type.
- **PCP** (Priority Code Point), 0 to 7, is the class-of-service priority used for QoS.
- **DEI** (Drop Eligible Indicator) marks frames that may be dropped first under congestion.
- **VID**, the VLAN ID, is 1 to 4094. Values 0 ("priority tag only, no VLAN") and 4095 are reserved.

Because the tag adds 4 bytes, the maximum frame becomes 1522 bytes, and the minimum *payload* drops to 42 bytes so that the minimum frame stays at 64. The switch must recompute the FCS whenever it adds or removes a tag.

### 5.3 Worked example: encoding a tag

Say a voice phone's traffic is in VLAN 100 with priority 5:

```
PCP = 5   -> 101
DEI = 0   -> 0
VID = 100 -> 0000 0110 0100

TCI = 101 0 0000 0110 0100 = 1010 0000 0110 0100 = 0xa064
```

A tagged IPv4 frame on the trunk then begins:

```
0000  00 1a 2b 00 00 01        Dst MAC
0006  00 1a 2b 00 00 50        Src MAC (the phone)
000c  81 00                    TPID: 802.1Q
000e  a0 64                    TCI: PCP 5, DEI 0, VID 100
0010  08 00                    EtherType: IPv4
0012  45 ...                   IPv4 header begins
```

A plain data frame in VLAN 20 at default priority has TCI `00 14`. When this frame leaves the trunk on an access port in VLAN 100, the switch removes bytes `0c` to `0f`, recomputes the FCS and delivers an ordinary untagged frame.

### 5.4 Double tagging and VLAN hopping

An attacker on an access port in a trunk's native VLAN can send a frame with two tags: outer = native VLAN, inner = victim VLAN. The first switch strips the outer tag (native traffic goes untagged on the trunk), and the second switch reads the inner one and delivers the frame into the victim VLAN. The defences are to put the native VLAN on an unused ID, tag it explicitly, and disable automatic trunk negotiation on access ports.

---

## 6. ARP Spoofing (ARP Cache Poisoning)

### 6.1 Why it works

ARP has **no authentication**, and most implementations are generous about what they accept:

- They accept **unsolicited replies** and gratuitous ARPs, because failover depends on that.
- They overwrite an existing entry when new information arrives.
- They learn from the sender fields of *requests* as well as replies.

Anyone on the broadcast domain can therefore announce "192.168.1.1 is at *my* MAC," and neighbours will believe it.

### 6.2 Worked example: man-in-the-middle between A and the router

Host C (the attacker, MAC `..:0c`) wants to read A's Internet traffic.

**Before the attack:**

```
A's cache:  192.168.1.1  -> 00:1a:2b:00:00:01  (R)
R's cache:  192.168.1.10 -> 00:1a:2b:00:00:0a  (A)
```

**The attack:** C sends two forged ARP replies, and repeats them every second or two so that genuine traffic cannot repair the caches.

To A:

```
Ethernet: dst 00:1a:2b:00:00:0a  src 00:1a:2b:00:00:0c  type 0806
ARP:      OPER 2
          SHA 00:1a:2b:00:00:0c   SPA 192.168.1.1     <- lie
          THA 00:1a:2b:00:00:0a   TPA 192.168.1.10
```

To R:

```
ARP:      OPER 2
          SHA 00:1a:2b:00:00:0c   SPA 192.168.1.10    <- lie
          THA 00:1a:2b:00:00:01   TPA 192.168.1.1
```

**After the attack:**

```
A's cache:  192.168.1.1  -> 00:1a:2b:00:00:0c  (C!)
R's cache:  192.168.1.10 -> 00:1a:2b:00:00:0c  (C!)
```

Now when A sends a packet to 203.0.113.50, the frame goes to destination MAC `..:0c`. The switch is working correctly and delivers it to port 3. C has IP forwarding enabled. It reads or modifies the packet, then sends it to R's real MAC. Return traffic flows the same way in reverse. A and R notice nothing except perhaps slightly higher latency. The switch MAC table is *not* affected, because every frame really does come from the port it says it does. The poison is entirely in the hosts' ARP caches.

Variations include denial of service (point the gateway's IP at a nonexistent MAC) and tampering with plaintext traffic such as DNS once it is intercepted.

### 6.3 Detecting it

Warning signs are the same MAC appearing for two IPs (check `ip neigh` for duplicate `lladdr` values), a gateway MAC that changes without a known failover, and a burst of unsolicited replies in a capture. `arpwatch` logs IP/MAC pairing changes on a segment.

### 6.4 Defending against it

- **Dynamic ARP Inspection (DAI)** on managed switches drops ARP packets whose SHA/SPA pair does not match the **DHCP snooping** binding table, and rate-limits ARP per port.
- **Static ARP entries** for the gateway cannot be overwritten, but are hard to maintain.
- **Port security, 802.1X and small VLANs** limit who can be on the segment at all.
- **End-to-end encryption** (TLS, SSH, IPsec, WireGuard) makes interception yield only ciphertext, though it does not stop DoS.

The lesson: anything sharing your broadcast domain can lie about addresses, so protect sensitive traffic at a higher layer.

---

## 7. Putting It All Together

For A to reach the Internet, A decides the destination is off-subnet, ARPs for the gateway (a padded 64-byte broadcast the switch floods while learning A's port), and frames the IP packet to the router's MAC. The switch forwards it out one port using its *(VLAN, MAC)* table. The router strips the frame, decrements the TTL and re-frames the packet for the next link, with its own ARP lookup. Every ARP answer along the way could have been forged by a neighbour, which is why DAI, segmentation and encryption matter.
