# The Border Gateway Protocol, Explained with Worked Examples

The Internet is not one network. It is roughly seventy-five thousand independently operated networks, called **Autonomous Systems** (ASes), that agree to carry each other's traffic. The protocol they use to tell one another which addresses they can reach is the **Border Gateway Protocol**, version 4 (BGP-4, RFC 4271). Every packet you send to a host outside your own provider follows a path that BGP chose.

This explainer covers the whole protocol: what problem it solves, how sessions come up, what messages it sends, how routes propagate, how a router picks one best path out of many, and how the system fails through route leaks and hijacks and what RPKI does about it. Everything is grounded in one running example.

All AS numbers and prefixes below come from the ranges reserved for documentation (RFC 5398 for ASNs, RFC 5737 and RFC 3849 for addresses), so none of them refer to a real network.

---

## 1. What BGP Is For

Inside a single organisation, routing protocols such as OSPF and IS-IS find the shortest path through a network whose topology every router can see. That approach does not scale to the Internet, and the operators would not want it to. No network wants to publish its internal topology to competitors, and "shortest" is rarely the goal anyway. Between networks, routing is driven by **business policy**: who pays whom, which links are cheap, which neighbours are trusted.

BGP is therefore a **path-vector** protocol. A route is not just "prefix X is reachable at cost 7." It is "prefix X is reachable, and here is the full list of ASes the announcement passed through to get to you." That list, the **AS_PATH**, does two jobs:

1. **Loop prevention.** If a router sees its own AS number in an incoming AS_PATH, it discards the route.
2. **Policy input.** Operators can see exactly whose network a route passes through and decide whether to accept, prefer, or reject it.

BGP runs over TCP port 179. Using TCP means BGP does not need its own retransmission or ordering logic. It also means BGP is **incremental**: after the initial exchange of full tables, peers send only changes.

### eBGP and iBGP

The same protocol is used in two settings:

- **eBGP (external BGP)** runs between routers in *different* ASes. It is where policy is applied and where the AS_PATH grows.
- **iBGP (internal BGP)** runs between routers in the *same* AS, so that every border router learns what the others learned. Routers do not prepend their AS number on iBGP, and by default a route learned over iBGP is not re-advertised to another iBGP peer. That rule means iBGP needs a full mesh or a scaling mechanism such as **route reflectors** (RFC 4456) or confederations.

---

## 2. The Running Example Topology

Here is a small Internet with five ASes:

```
                    AS64500 (Transit "T1")
                   /          \
           customer            customer
                 /              \
   AS64501 (ISP "A") ---peer--- AS64502 (ISP "B")
          |                          |
       customer                   customer
          |                          |
   AS64510 (Enterprise "E") ---------+
          (multihomed to both A and B)

   AS64511 (Content network "C"), customer of B only
```

Concretely:

| AS | Role | Originates |
|----|------|-----------|
| AS64500 | Tier-1 style transit, sells transit to A and B | `198.51.100.0/24` |
| AS64501 | Regional ISP A | `203.0.113.0/25` |
| AS64502 | Regional ISP B | `203.0.113.128/25` |
| AS64510 | Enterprise E, multihomed to A and B | `192.0.2.0/24`, `2001:db8:e::/48` |
| AS64511 | Content network C, single-homed to B | `2001:db8:c::/48` |

Relationships follow the standard **Gao-Rexford** model, which describes how most commercial BGP policy works:

- **Customer to provider:** the customer pays. The provider carries the customer's routes to everyone, and gives the customer routes to everywhere.
- **Peer to peer:** settlement-free. Each side carries only its *own and its customers'* routes to the other. Neither provides transit for the other.

These relationships lead to the export rule that keeps the Internet sane, and that a route leak violates:

> Routes learned from a **customer** may be exported to anyone. Routes learned from a **peer** or a **provider** may be exported only to **customers**.

Link addresses used below:

| Link | Address on each side |
|------|---------------------|
| E to A | E `192.0.2.254`, A `192.0.2.253` (a /30 out of E's space, for brevity) |
| A to B | A `198.51.100.249`, B `198.51.100.250` |
| A to T1 | A `198.51.100.1`, T1 `198.51.100.2` |

---

## 3. Session Establishment: The Finite State Machine

Before any routes flow, two BGP speakers must establish a session. RFC 4271 defines a six-state finite state machine (FSM):

| State | Meaning |
|-------|---------|
| **Idle** | Not trying. Waiting for a Start event (configuration, or a retry timer). Refuses incoming connections. |
| **Connect** | Trying to open the TCP connection to the peer. |
| **Active** | The TCP attempt failed or timed out. Listening for the peer to connect in, and will retry. Despite its name, a session "stuck in Active" is *not* working. |
| **OpenSent** | TCP is up. Our OPEN message is sent. Waiting for the peer's OPEN. |
| **OpenConfirm** | We received an acceptable OPEN and sent a KEEPALIVE. Waiting for the peer's KEEPALIVE. |
| **Established** | Up. UPDATEs and KEEPALIVEs flow. |

### Worked trace: A brings up its session with E

The configuration on A's border router, in a vendor-neutral sketch:

```
router bgp 64501
  neighbor 192.0.2.254 remote-as 64510
  neighbor 192.0.2.254 description Customer-E
  neighbor 192.0.2.254 hold-time 90
```

E has the mirror-image configuration pointing at `192.0.2.253` with `remote-as 64501`.

```
t=0.000  A: Idle -> Connect       (admin enabled the neighbor)
t=0.001  A: TCP SYN to 192.0.2.254:179
t=0.002  E: TCP SYN-ACK
t=0.002  A: TCP ACK; connection up
t=0.003  A: Connect -> OpenSent   ; sends OPEN
            OPEN { version=4, my_as=64501, hold_time=90,
                   bgp_id=198.51.100.1,
                   capabilities: [MP-BGP ipv4-unicast, MP-BGP ipv6-unicast,
                                  4-octet-AS 64501, route-refresh,
                                  graceful-restart] }
t=0.004  A: receives E's OPEN
            OPEN { version=4, my_as=64510, hold_time=180,
                   bgp_id=192.0.2.1, capabilities: [...] }
         A checks: version OK; my_as 64510 == configured remote-as;
                   bgp_id not equal to ours; hold time >= 3.
         Negotiated hold time = min(90, 180) = 90 s
         Keepalive interval  = 90 / 3 = 30 s
t=0.004  A: OpenSent -> OpenConfirm ; sends KEEPALIVE
t=0.005  A: receives KEEPALIVE from E
t=0.005  A: OpenConfirm -> Established
```

The other side runs the same machine at the same time. If both routers open TCP connections to each other at once, **collision detection** keeps the connection initiated by the speaker with the higher BGP Identifier and closes the other.

### The four message types

| Type | Purpose |
|------|---------|
| **OPEN** (1) | Parameters and capabilities. Sent once per session. |
| **UPDATE** (2) | Advertises new routes and/or withdraws old ones. |
| **NOTIFICATION** (3) | Fatal error; session closes. |
| **KEEPALIVE** (4) | "Still here." 19 bytes of header only. |

**ROUTE-REFRESH** (type 5, RFC 2918) is a negotiated extension that asks a peer to resend its routes, so new inbound policy can be applied without resetting the session.

---

## 4. The UPDATE Message and Path Attributes

An UPDATE carries three things:

1. **Withdrawn routes:** prefixes that are no longer reachable through the sender.
2. **Path attributes:** properties shared by every prefix in this UPDATE.
3. **NLRI (Network Layer Reachability Information):** the prefixes being advertised.

IPv6 and other address families travel inside the `MP_REACH_NLRI` and `MP_UNREACH_NLRI` attributes (RFC 4760). That is why one BGP session can carry IPv4, IPv6, VPN routes, and more.

The attributes you will meet constantly:

| Attribute | Category | Purpose |
|-----------|----------|---------|
| **ORIGIN** | well-known mandatory | How the route entered BGP: IGP (`i`), EGP (`e`), or INCOMPLETE (`?`, usually redistribution). |
| **AS_PATH** | well-known mandatory | Sequence of ASes the route has traversed, newest first. |
| **NEXT_HOP** | well-known mandatory | The address to forward packets to for this prefix. |
| **LOCAL_PREF** | well-known discretionary, iBGP only | Higher wins. The operator's main tool for choosing *outbound* exits. |
| **MED** (MULTI_EXIT_DISC) | optional non-transitive | Lower wins. A hint to a neighbouring AS about which of several links to use *inbound*. |
| **COMMUNITIES** | optional transitive | 32-bit tags (`ASN:value`) used to signal policy. Large Communities (RFC 8092) widen these to `ASN:x:y` for 4-byte ASNs. |
| **ATOMIC_AGGREGATE / AGGREGATOR** | — | Mark routes that were summarised. |
| **ORIGINATOR_ID / CLUSTER_LIST** | optional non-transitive | Loop prevention for route reflection. |

---

## 5. Propagation Trace: How E's Prefix Reaches the World

E originates `192.0.2.0/24` and wants the whole Internet to reach it, with A as the primary path and B as the backup.

### Step 1: E announces to both providers

E's router has the prefix in its routing table (a static route to Null0, or an internal route), and the operator configures `network 192.0.2.0/24`. E sends to A:

```
UPDATE  E(AS64510) -> A(AS64501)
  ORIGIN    = IGP
  AS_PATH   = 64510
  NEXT_HOP  = 192.0.2.254
  NLRI      = 192.0.2.0/24
```

To make B the backup, E **prepends** its own AS twice toward B, so the path looks longer to everyone downstream:

```
UPDATE  E(AS64510) -> B(AS64502)
  ORIGIN    = IGP
  AS_PATH   = 64510 64510 64510
  NEXT_HOP  = <E's address on the E-B link>
  NLRI      = 192.0.2.0/24
```

### Step 2: A processes the route

A's inbound filter for customer E allows only the prefixes E is entitled to originate, here `192.0.2.0/24` and `2001:db8:e::/48`. That list is typically built automatically from IRR objects and RPKI ROAs (Section 8). The route passes. A's inbound policy then:

- sets `LOCAL_PREF = 200` (A's customer routes are preferred over peer routes at 150 and provider routes at 100);
- tags it with community `64501:100` meaning "learned from a customer."

The route enters A's **Adj-RIB-In** for that neighbor, goes through best-path selection (it is the only candidate so far), and lands in A's **Loc-RIB** and forwarding table.

### Step 3: A exports

Because the route came from a customer, the export rule allows it to go everywhere: to T1 (provider), to B (peer), and to A's other customers. At each eBGP export, A prepends its own ASN and sets NEXT_HOP to its own address on that link:

```
UPDATE  A(AS64501) -> T1(AS64500)
  ORIGIN    = IGP
  AS_PATH   = 64501 64510
  NEXT_HOP  = 198.51.100.1
  NLRI      = 192.0.2.0/24
```

```
UPDATE  A(AS64501) -> B(AS64502)
  ORIGIN    = IGP
  AS_PATH   = 64501 64510
  NEXT_HOP  = 198.51.100.249
  NLRI      = 192.0.2.0/24
```

LOCAL_PREF is not sent over eBGP. Communities may or may not be, depending on A's export policy.

### Step 4: B now holds two candidates

B has heard `192.0.2.0/24` twice:

| From | AS_PATH | Relationship | LOCAL_PREF set by B's inbound policy |
|------|---------|--------------|----------------------------------------|
| E (customer) | `64510 64510 64510` | customer | 200 |
| A (peer) | `64501 64510` | peer | 150 |

Section 6 works through which one B picks. The answer is that B prefers the **direct customer route**, despite the longer path. Prepending is weak: it only affects networks that fall all the way down to the AS_PATH tie-breaker.

### Step 5: T1 and beyond

T1 hears the prefix from A (`64501 64510`) and, because B exports its customer route to its provider, also from B (`64502 64510 64510 64510`). Both are customer routes for T1, so LOCAL_PREF ties, and T1 picks the shorter AS_PATH through A. Prepending achieves its goal here. Content network C, a customer of B only, reaches E through B directly.

### Step 6: A withdrawal

Suppose the E–A link fails. A's session to E drops (the Hold Timer expires, or the interface goes down immediately and BFD tears the session within milliseconds). A deletes the route from its Adj-RIB-In, finds no other candidate in Loc-RIB except B's peer-learned path, and sends:

```
UPDATE  A(AS64501) -> T1(AS64500)
  WITHDRAWN = 192.0.2.0/24
```

A does *not* advertise B's path to T1, because a peer route may only go to customers. T1 now falls back to its remaining path via B (`64502 64510 64510 64510`). Traffic shifts; E stays reachable. Making that failover actually happen is the whole point of multihoming.

During convergence, routers can briefly explore longer, stale alternatives (**path hunting**). The **MRAI** (Minimum Route Advertisement Interval) timer rate-limits updates to damp this, which is why BGP convergence is usually measured in seconds to minutes rather than milliseconds.

---

## 6. Best-Path Selection, Walked Through

When a router holds several valid routes to the same prefix, it runs a deterministic sequence of tie-breakers. The first step that produces a single winner ends the process. The exact list varies by vendor; this order is the common one, with vendor-specific steps in italics:

0. **Discard ineligible routes:** NEXT_HOP unreachable, AS_PATH loop, rejected by policy, or (with RPKI enforcement) RPKI-invalid.
1. *Highest Weight* (Cisco-specific, local to the router).
2. **Highest LOCAL_PREF.**
3. *Prefer locally originated routes.*
4. **Shortest AS_PATH** (an AS_SET counts as 1; confederation segments count as 0).
5. **Lowest ORIGIN** (IGP < EGP < INCOMPLETE).
6. **Lowest MED**, compared by default only among routes from the *same* neighbouring AS.
7. **eBGP over iBGP.**
8. **Lowest IGP metric to the NEXT_HOP** ("hot-potato" routing: hand the packet off at the nearest exit).
9. *Oldest eBGP route* (stability preference, used by some vendors).
10. **Lowest BGP Router ID** of the advertising peer (or ORIGINATOR_ID if reflected).
11. **Shortest CLUSTER_LIST.**
12. **Lowest neighbor address.**

### Walk-through 1: B choosing a path to `192.0.2.0/24`

| Step | Route 1 (from E, customer) | Route 2 (from A, peer) | Result |
|------|----------------------------|------------------------|--------|
| 0. Eligible? | yes | yes | both continue |
| 2. LOCAL_PREF | **200** | 150 | **Route 1 wins** |

Done at step 2. AS_PATH length (3 vs 2) is never compared. This is the most important lesson about BGP policy: **LOCAL_PREF, which encodes commercial relationships, beats path length**. If E really wants B to avoid the direct link, prepending will not do it. E must use a **community** that B publishes for that purpose, for example `64502:70` meaning "set LOCAL_PREF to 70 on this route." Many providers publish such a list of action communities.

E re-announces to B with the community attached:

```
UPDATE  E -> B
  AS_PATH     = 64510
  COMMUNITIES = 64502:70
  NLRI        = 192.0.2.0/24
```

B's inbound policy matches `64502:70` and sets LOCAL_PREF to 70. Now:

| Step | Route 1 (from E) | Route 2 (from A) | Result |
|------|------------------|------------------|--------|
| 2. LOCAL_PREF | 70 | **150** | **Route 2 wins** |

B now sends traffic for E across the peering link to A, and the direct E–B link carries only backup traffic. Note the side effect: B now prefers a *peer* route, and peer routes go only to customers, so B stops advertising `192.0.2.0/24` to T1. Inbound traffic from the wider Internet comes through A alone, which is exactly what E wanted. If the A–E link fails, A withdraws, B's remaining candidate is the direct route, and B re-advertises it to T1.

### Walk-through 2: Inside A, two exits to T1

Suppose A has two border routers, R1 in the city of Northport and R2 in Southport, each with an eBGP session to T1, and T1 originates `198.51.100.0/24`. T1 sets MED 10 on the Northport link and MED 50 on the Southport link, because the server for that prefix sits nearer Northport. A router R3 inside A learns both via iBGP:

| Step | Via R1 (Northport) | Via R2 (Southport) | Result |
|------|--------------------|--------------------|--------|
| 2. LOCAL_PREF | 100 | 100 | tie |
| 4. AS_PATH | `64500` (1) | `64500` (1) | tie |
| 5. ORIGIN | IGP | IGP | tie |
| 6. MED (same neighbor AS 64500) | **10** | 50 | **R1 wins** |

If A ignored MED, which it may legitimately do since MED is only a hint, step 8 would decide. R3 sits in Southport, so its IGP cost to R2 is lower and it would hand traffic off locally (hot potato). Honouring MED is a cooperative choice; many networks honour it only from customers.

---

## 7. Route Leaks and Hijacks

BGP was designed in an era of mutual trust. By default, a router believes whatever its neighbours tell it. Two classes of incident follow from that.

### Route leaks

RFC 7908 defines a route leak as the propagation of routing announcements beyond their intended scope. The classic case is a breach of the export rule.

**Worked leak:** E, multihomed to A and B, has a misconfigured export policy toward A. Instead of exporting only its own prefixes, E exports *everything it learned from B*, including `2001:db8:c::/48`, the prefix of C (B's customer):

```
UPDATE  E -> A
  AS_PATH = 64510 64502 64511
  NLRI    = 2001:db8:c::/48
```

A's view of that prefix:

| Candidate | AS_PATH | Relationship | LOCAL_PREF |
|-----------|---------|--------------|------------|
| via E | `64510 64502 64511` | **customer** | 200 |
| via B | `64502 64511` | peer | 150 |
| via T1 | `64500 64502 64511` | provider | 100 |

The leaked route wins at step 2, because A prefers customer routes. A then exports it to T1 and to every other customer, since customer routes go everywhere. Now a large part of the Internet sends C's traffic through E, a small enterprise with two modest links that were never sized as transit. Result: congestion, packet loss, and possibly visibility of the traffic to E, even though no one acted maliciously.

The AS_PATH is perfectly genuine; nothing here is forged. The leak is a *policy* violation. Defences include:

- **Strict customer prefix filters.** If A only accepted from E what E may originate (`192.0.2.0/24`, `2001:db8:e::/48`), the leak would have stopped at the first hop. This is the single most effective control and is part of the **MANRS** baseline.
- **Max-prefix limits.** If E normally sends 2 prefixes, a limit of, say, 10 tears down the session when a full table leaks.
- **RFC 8212 default.** eBGP sessions with no explicit policy must neither accept nor send routes.
- **BGP Roles and the OTC attribute (RFC 9234).** Peers declare their relationship in the OPEN message (provider, customer, peer, RS, RS-client). Routes sent to a customer or peer are stamped with an **Only to Customer** attribute, and a router that receives a route carrying OTC from a customer knows it is a leak and drops it.
- **ASPA (Autonomous System Provider Authorization).** An RPKI object in which an AS lists its authorised providers. Routers can then check whether an AS_PATH is "valley-free." Here, AS64510 has no ASPA listing AS64502 as a valid customer of it, so the path `64510 64502 64511` received from a customer fails verification. ASPA is in standardisation and early deployment.

### Prefix hijacks

A hijack is an announcement of address space by an AS that is not authorised to originate it. It may be malicious (to intercept traffic or steal cryptocurrency) or accidental (a typo in a prefix list).

**Origin hijack:** suppose a compromised AS64511 announces E's prefix:

```
UPDATE  C -> B
  AS_PATH = 64511
  NLRI    = 192.0.2.0/24
```

At B the hijack (`64511`, customer, LOCAL_PREF 200) competes with E's genuine route (`64510`, also customer, LOCAL_PREF 200, or 70 if E is still sending the backup community). Path lengths are equal, so the tie goes to later steps, and some networks will pick the hijack. The Internet splits: part reaches E, part reaches C.

**More-specific hijack:** worse, C announces `192.0.2.0/25` and `192.0.2.128/25`. Forwarding uses **longest-prefix match**, before BGP preference is even considered. Every network that accepts the /25s sends E's traffic to C, whatever the AS_PATH or LOCAL_PREF of the legitimate /24. This is why more-specific hijacks spread so far.

**Forged-origin hijack:** a sophisticated attacker announces `AS_PATH = 64511 64510`, faking E as the origin. Origin checks pass; only path validation (ASPA, or BGPsec) catches it.

---

## 8. RPKI: Cryptographic Route Origin Validation

The **Resource Public Key Infrastructure** (RFC 6480 and family) ties address space and AS numbers to cryptographic certificates issued down the same chain as the allocations themselves: IANA, then the Regional Internet Registries (AFRINIC, APNIC, ARIN, LACNIC, RIPE NCC), then members.

### ROAs

A holder of address space signs a **Route Origin Authorization** (ROA) saying which AS may originate it, and up to what prefix length. E creates:

```
ROA
  prefix:     192.0.2.0/24
  maxLength:  24
  origin AS:  64510

ROA
  prefix:     2001:db8:e::/48
  maxLength:  48
  origin AS:  64510
```

`maxLength` matters. Setting it to 24 means any /25 is *not* authorised. A loose `maxLength` (for example 26 when only the /24 is announced) would let an attacker announce forged-origin /25s that still look valid; RFC 9319 recommends keeping `maxLength` equal to the prefix length you actually announce.

### Validation pipeline

1. **Relying-party software** (Routinator, rpki-client, FORT, OctoRPKI) fetches the signed objects from every RIR's repository via RRDP or rsync, verifies the signature chains, and builds a list of **Validated ROA Payloads** (VRPs): `(prefix, maxLength, ASN)` triples.
2. It serves the VRPs to routers over the **RPKI-to-Router protocol** (RTR, RFC 8210).
3. Each router evaluates every received route (RFC 6811):

| State | Condition |
|-------|-----------|
| **Valid** | Some VRP covers the prefix, the prefix length ≤ maxLength, and the origin AS (the last AS in the AS_PATH) matches. |
| **Invalid** | At least one VRP covers the prefix, but none matches both origin AS and length. |
| **NotFound** | No VRP covers the prefix at all. |

### Applying it to the hijacks

| Announcement | Origin | Covering VRP | State | Outcome with "drop invalid" |
|--------------|--------|--------------|-------|-----------------------------|
| `192.0.2.0/24` from E | 64510 | `192.0.2.0/24-24 AS64510` | **Valid** | accepted |
| `192.0.2.0/24` from C | 64511 | same | **Invalid** (wrong AS) | dropped |
| `192.0.2.0/25` from C | 64511 | same | **Invalid** (wrong AS, too long) | dropped |
| `192.0.2.0/25` with path `64511 64510` | 64510 | same | **Invalid** (length > maxLength) | dropped |
| `192.0.2.0/24` with path `64511 64510` | 64510 | same | **Valid** | accepted: forged origin passes |
| `198.51.100.0/24` from T1 (no ROA) | 64500 | none | **NotFound** | accepted |

Note the second-last row. Route origin validation stops origin hijacks and more-specific attacks, but it only checks the *last* AS on the path. A forged path ending in the right AS gets through, as long as the prefix length is valid. A tight `maxLength` at least forces such an attacker to announce the same /24, competing on path length rather than winning outright by longest-prefix match.

### Deploying ROV

A typical inbound policy on B:

```
route-map FROM-CUSTOMER-C
  match rpki invalid      -> deny
  match prefix-list C-ALLOWED  -> permit, set local-pref 200
  default                 -> deny
```

Dropping invalids, rather than just lowering their preference, matters. A depreferenced invalid /25 still wins by longest-prefix match whenever there is no valid /25 to compete with it. Most large transit networks now drop RPKI-invalid routes, which means a hijack announced with a ROA in place usually spreads only through networks that do not.

### Beyond origin validation

**ASPA** extends RPKI to the *path*. **BGPsec** (RFC 8205) has each AS sign the path, but needs every AS to participate and is barely deployed. Operators also use **Peerlock**-style AS_PATH filters for large networks.

---

## 9. Summary

BGP is a small protocol with large consequences. It has four message types, a six-state session machine, and a handful of attributes. From these come the Internet's global reachability, its business structure, and its most dramatic outages.

The protocol trusts by default. A safe Internet depends on each operator filtering what it accepts, limiting what it exports, and publishing cryptographic statements about what it owns.
