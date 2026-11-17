# The Domain Name System, Explained

The Domain Name System (DNS) turns names people can remember, such as `www.example.com`, into the data machines need: IP addresses, mail server names, cryptographic keys and more. It is one of the oldest distributed databases still in daily use, and it is easy to take for granted. Almost every connection on the Internet starts with a DNS lookup, and when DNS fails, everything seems to fail with it.

This explainer covers DNS from the ground up: the namespace, the servers involved, a full recursive resolution traced step by step, the common record types with a sample zone file, caching and TTLs, the wire format down to individual bytes, and an overview of DNSSEC.

> **A note on addresses.** The root and `.com` server addresses below are real. Addresses for `example.com` hosts come from the documentation ranges (`192.0.2.0/24`, `2001:db8::/32`). They show the mechanics; they are not live data.

---

## 1. Why DNS exists

Before DNS, every host downloaded one shared `HOSTS.TXT` file, which could not scale. Paul Mockapetris designed DNS in 1983 (now RFC 1034 and RFC 1035) around three ideas:

1. **A hierarchical namespace**, so that names can be managed in pieces.
2. **Delegation**, so that each piece is run by whoever owns it.
3. **Caching**, so that the system scales by remembering answers rather than asking for them again and again.

Those three ideas still explain nearly everything DNS does.

## 2. The namespace

DNS names form a tree. Each node has a **label** of up to 63 bytes, and a full name, read from the node up to the root, may be at most 255 bytes on the wire. The root has an empty label, which is why a **fully qualified domain name** strictly ends with a dot:

```
                        . (root)
          ┌─────────────┼──────────────┐
         com           org             uk
       ┌──┴───┐         │              │
   example  google   wikipedia         co
     ┌─┴──┐                            │
    www   mail                        bbc
```

`www.example.com.` names the path `www` → `example` → `com` → root. The trailing dot is usually left off in browsers and applications, but it matters in zone files, as shown later.

A **domain** is a node plus everything below it. A **zone** is the part one set of servers is authoritative for. They differ because of **delegation**: the `com` zone does not hold `example.com`'s records, only a pointer saying "ask these servers." That **zone cut** lets millions of organisations run their own corners of the tree.

## 3. The cast of servers

Four roles appear in nearly every lookup:

- **Stub resolver.** The small DNS client in your operating system or application. It does not chase referrals itself. It sends one recursive query to a configured server and waits for a final answer.
- **Recursive resolver** (also called a full-service resolver or caching resolver). Usually run by your ISP, your company, or a public provider such as `1.1.1.1`, `8.8.8.8` or `9.9.9.9`. It does the real work, walking down the tree from the root, and it caches everything it learns.
- **Root servers.** Thirteen named server identities, `a.root-servers.net` through `m.root-servers.net`, run by twelve organisations. Each identity is served from hundreds of locations through **anycast**, so "13 root servers" really means more than 1,900 instances worldwide. The root zone itself is small: it mostly lists the name servers for each top-level domain.
- **Authoritative servers.** The servers that hold a zone's data and answer for it with the **AA** (Authoritative Answer) flag set. TLD servers such as `a.gtld-servers.net` are authoritative for `com`. Your hosting provider's name servers are authoritative for your domain.

A resolver needs one piece of preconfigured knowledge to begin: the **root hints**, a list of root server names and addresses that ships with the resolver software.

## 4. A full recursive resolution, step by step

Suppose a laptop has just booted, its resolver's cache is empty, and the user types `www.example.com` into a browser. The steps below follow the query.

### Step 0: The stub asks the resolver

The browser calls `getaddrinfo("www.example.com")`. The OS stub checks its local cache and `/etc/hosts`, finds nothing, and sends a UDP packet to port 53 on its configured resolver, say `10.0.0.1`:

```
Query: www.example.com. IN A    flags: RD (recursion desired)
```

The **RD** bit means: "please do the whole job for me."

### Step 1: The resolver asks a root server

The resolver's cache is empty, so it starts at the top. It picks a root server from its hints, `a.root-servers.net` at `198.41.0.4`. Modern resolvers use **QNAME minimisation** (RFC 9156) and send the root only as much of the name as it needs to see:

```
→ 198.41.0.4    Query: com. IN NS
```

(An older resolver would send the full `www.example.com. IN A`. The root's answer would be the same referral.)

The root is not authoritative for `com`, but it knows who is. It returns a **referral**: no answer section, an authority section listing the `com` name servers, and an additional section with their addresses (called **glue**):

```
;; flags: qr; QUERY: 1, ANSWER: 0, AUTHORITY: 13, ADDITIONAL: 26
;; AUTHORITY SECTION:
com.                  172800  IN  NS  a.gtld-servers.net.
com.                  172800  IN  NS  b.gtld-servers.net.
...                                     (13 in total)
;; ADDITIONAL SECTION:
a.gtld-servers.net.   172800  IN  A     192.5.6.30
a.gtld-servers.net.   172800  IN  AAAA  2001:503:a83e::2:30
...
```

Note the TTL of 172,800 seconds (two days). The resolver caches all of this, so for the next two days it can skip the root entirely for any `.com` name.

### Step 2: The resolver asks a `.com` server

```
→ 192.5.6.30    Query: example.com. IN NS
```

The `.com` servers are authoritative for `com` but not for `example.com`. They return another referral:

```
;; flags: qr; QUERY: 1, ANSWER: 0, AUTHORITY: 2, ADDITIONAL: 2
;; AUTHORITY SECTION:
example.com.        172800  IN  NS  ns1.example.com.
example.com.        172800  IN  NS  ns2.example.com.
;; ADDITIONAL SECTION:
ns1.example.com.    172800  IN  A   192.0.2.53
ns2.example.com.    172800  IN  A   198.51.100.53
```

Here the glue records are essential. The name servers for `example.com` are themselves *inside* `example.com`. Without their addresses in this referral, the resolver would need to resolve `ns1.example.com` in order to find the server that would tell it the address of `ns1.example.com`, which is a circular dependency. The parent zone breaks the loop by holding copies of those addresses.

### Step 3: The resolver asks the authoritative server

```
→ 192.0.2.53    Query: www.example.com. IN A
```

This time the server owns the data:

```
;; flags: qr aa; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 0
;; ANSWER SECTION:
www.example.com.    3600  IN  A  192.0.2.10
```

The **aa** flag confirms the answer is authoritative.

### Step 4: The resolver answers the stub

The resolver caches the record for up to 3,600 seconds and returns it to the laptop with the **RA** (Recursion Available) flag set. The **AA** flag is clear, because the resolver is not the authority. It is passing on what it learned:

```
;; flags: qr rd ra; QUERY: 1, ANSWER: 1
www.example.com.    3600  IN  A  192.0.2.10
```

The browser now opens a TCP or QUIC connection to `192.0.2.10`.

### What the trace teaches

- **Iteration and recursion are different modes.** The stub made one *recursive* query. The resolver made a series of *iterative* queries, each of which returned either a referral or an answer.
- **Most of the work happens once.** On a second lookup, for example `mail.example.com`, the resolver already knows the `example.com` servers and goes straight to step 3. On a third lookup of `www.example.com` within the hour, it sends no upstream queries at all.
- **CNAMEs restart the walk.** If `www` were a CNAME to a CDN name, the resolver would begin a fresh resolution for the target.

### Doing it yourself

`dig +trace www.example.com` makes your machine act as the resolver and prints each referral as it is followed. It is the best single tool for learning DNS.

## 5. Record types and a sample zone file

Every piece of DNS data is a **resource record** (RR) with five fields:

```
<owner name>  <TTL>  <class>  <type>  <RDATA>
www.example.com.  3600  IN  A  192.0.2.10
```

The class is almost always `IN` (Internet). A group of records with the same owner, class and type is an **RRset**. RRsets are the unit of caching, of answers, and of DNSSEC signatures.

### Common types

| Type | Purpose | Example RDATA |
|---|---|---|
| `A` | IPv4 address | `192.0.2.10` |
| `AAAA` | IPv6 address | `2001:db8::10` |
| `CNAME` | Alias to another name (the canonical name) | `www.example.net.` |
| `NS` | Delegates a zone to a name server | `ns1.example.com.` |
| `SOA` | Start of Authority: zone metadata | see below |
| `MX` | Mail exchanger, with preference | `10 mail.example.com.` |
| `TXT` | Free text, used for SPF, DKIM and site verification | `"v=spf1 mx -all"` |
| `SRV` | Service location: priority, weight, port, target | `10 5 5060 sip.example.com.` |
| `PTR` | Reverse mapping, address to name | `www.example.com.` |
| `CAA` | Which CAs may issue certificates | `0 issue "letsencrypt.org"` |
| `HTTPS`/`SVCB` | Service bindings: ALPN, ports, address hints | `1 . alpn="h2,h3"` |
| `DNSKEY`, `DS`, `RRSIG`, `NSEC`, `NSEC3` | DNSSEC (section 8) | |

Two rules trip people up. First, a `CNAME` cannot coexist with any other record at the same name, which is why you cannot put a `CNAME` at a zone apex (the apex must hold `SOA` and `NS`). Many providers offer a proprietary "ALIAS" or "flattening" feature to work around this. Second, `MX` and `NS` targets must be host names with address records, never `CNAME`s.

### A sample zone file

The master file format from RFC 1035 is still what BIND, NSD, Knot and most tools read:

```zone
$ORIGIN example.com.
$TTL 3600                         ; default TTL for records below

@   IN  SOA  ns1.example.com. hostmaster.example.com. (
            2026111701   ; serial  (YYYYMMDDnn)
            7200         ; refresh (secondaries check every 2 h)
            900          ; retry   (if refresh fails, retry every 15 min)
            1209600      ; expire  (stop answering after 14 days unreachable)
            300 )        ; minimum / negative-caching TTL (5 min)

; --- Name servers and their glue ---
@           IN  NS     ns1.example.com.
@           IN  NS     ns2.example.com.
ns1         IN  A      192.0.2.53
ns2         IN  A      198.51.100.53

; --- Web ---
@           IN  A      192.0.2.10
@           IN  AAAA   2001:db8::10
www         IN  CNAME  example.com.
@           IN  HTTPS  1 . alpn="h2,h3"

; --- Mail ---
@           IN  MX     10 mail.example.com.
@           IN  MX     20 mail-backup.example.net.
mail        IN  A      192.0.2.25
@           IN  TXT    "v=spf1 mx -all"
_dmarc      IN  TXT    "v=DMARC1; p=reject; rua=mailto:dmarc@example.com"

; --- Services ---
_sip._tcp   IN  SRV    10 5 5060 sip.example.com.
sip    300  IN  A      192.0.2.60      ; short TTL: may move soon

; --- Certificate policy ---
@           IN  CAA    0 issue "letsencrypt.org"

; --- Delegation of a subzone ---
lab         IN  NS     ns.lab.example.com.
ns.lab      IN  A      192.0.2.200     ; glue
```

Points to notice:

- **`@`** stands for the current `$ORIGIN`, here `example.com.`.
- **Names without a trailing dot are relative.** `mail` becomes `mail.example.com.`. The classic mistake is writing `mail.example.com` without the dot in RDATA, which the server reads as `mail.example.com.example.com.`.
- **The SOA's second field** is the zone contact's email address with the `@` replaced by a dot: `hostmaster.example.com.` means `hostmaster@example.com`.
- **The serial** must increase on every change. Secondary servers compare serials to decide whether to transfer the zone (via `AXFR` for a full transfer or `IXFR` for an incremental one).
- **The `lab` delegation** is a zone cut inside this file. The `example.com` servers will answer queries for `anything.lab.example.com` with a referral, exactly as `.com` did in the trace above.

### Reverse DNS

Reverse lookups reverse the address into a name under `in-addr.arpa`: `192.0.2.10` becomes `10.2.0.192.in-addr.arpa.`, whose `PTR` record points back to `www.example.com.`. IPv6 uses `ip6.arpa`, one label per hex nibble.

## 6. Caching and TTLs

Caching is what makes DNS work at Internet scale. The root servers would collapse if every lookup started there.

### How TTL works

Every record carries a **Time To Live** in seconds, set by the zone owner. A resolver that receives a record with TTL 3600 may serve it from cache for up to an hour. When it serves a cached record, it reports the *remaining* TTL. If you query the same resolver twice, 100 seconds apart, you will see something like:

```
$ dig +noall +answer www.example.com @10.0.0.1
www.example.com.   3600  IN  A  192.0.2.10
$ sleep 100; dig +noall +answer www.example.com @10.0.0.1
www.example.com.   3500  IN  A  192.0.2.10
```

The countdown is the clearest sign you are seeing a cached answer.

### A worked timeline

Take the zone above and a busy resolver:

| Time | Event | Upstream queries |
|---|---|---|
| 09:00:00 | First lookup of `www.example.com`. Resolver walks root → `.com` → `ns1`. Caches `.com` NS (2 days), `example.com` NS (2 days), the `www` CNAME (1 h), and the apex A (1 h). | 3 |
| 09:00:05 | Thousands more clients ask for `www.example.com`. | 0 |
| 09:30:00 | Client asks for `sip.example.com` (TTL 300). Resolver goes straight to `ns1`. | 1 |
| 09:35:01 | `sip` has expired; next request re-fetches it. | 1 |
| 10:00:01 | `www` CNAME and apex A expire; next request goes to `ns1` only. The NS records are still cached. | 1 |

### Choosing TTLs

TTL is a trade-off between **agility** and **load and latency**:

- **Long TTLs** (hours to days) mean fewer queries, faster lookups for users, and resilience: if your authoritative servers go down, cached answers keep working until they expire.
- **Short TTLs** (30 to 300 seconds) let you change records quickly, which matters for failover, load balancing and CDNs. The cost is more queries and more first-lookup latency.

A standard migration technique: **lower the TTL first.** If a record has TTL 86400 and you plan to change the IP address on Friday, drop the TTL to 300 on Wednesday. After a day, every cache holding the old long TTL has expired, so when you switch on Friday the whole world follows within five minutes. Once things are stable, raise the TTL again.

"DNS propagation" is a misnomer: nothing is pushed. Changes appear as old cached copies expire, so the delay is set by the *old* TTL.

### Negative caching

A resolver also caches the fact that something *does not exist* (RFC 2308). There are two kinds of negative answer:

- **NXDOMAIN** (RCODE 3): the name does not exist at all.
- **NODATA**: the name exists but has no records of the requested type (RCODE 0 with an empty answer section).

Both include the zone's SOA record in the authority section. The negative-caching TTL is the *lesser* of the SOA's own TTL and its `minimum` field, 300 seconds in our zone. If you query for `new.example.com` before creating it, you may have to wait up to five minutes after creating it before that resolver sees it.

## 7. The wire format, byte by byte

DNS messages are compact binary structures, defined in RFC 1035 section 4. Queries and responses share one layout:

```
+---------------------+
|        Header       |  12 bytes, always
+---------------------+
|       Question      |  what is being asked
+---------------------+
|        Answer       |  RRs answering the question
+---------------------+
|      Authority      |  RRs pointing at an authority (referrals, SOA)
+---------------------+
|      Additional     |  helpful extra RRs (glue, EDNS OPT)
+---------------------+
```

### The header

```
 0  1  2  3  4  5  6  7  8  9 10 11 12 13 14 15
+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+
|                      ID                       |
+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+
|QR|  Opcode   |AA|TC|RD|RA| Z|AD|CD|   RCODE   |
+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+
|                    QDCOUNT                    |
|                    ANCOUNT                    |
|                    NSCOUNT                    |
|                    ARCOUNT                    |
+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+
```

- **ID**: a 16-bit value chosen by the client and echoed in the reply so responses can be matched to queries.
- **QR**: 0 for a query, 1 for a response.
- **Opcode**: 0 for a standard query.
- **AA**: authoritative answer. **TC**: truncated (retry over TCP). **RD**: recursion desired. **RA**: recursion available.
- **AD**/**CD**: DNSSEC "authentic data" and "checking disabled".
- **RCODE**: 0 NOERROR, 2 SERVFAIL, 3 NXDOMAIN, 5 REFUSED, among others.
- Four counts give the number of entries in each section.

### A query, fully decoded

Here is the stub's query from step 0, asking for the `A` record of `www.example.com` with ID `0x1a2b`. It is 33 bytes:

```
1a 2b  01 00  00 01  00 00  00 00  00 00
03 77 77 77  07 65 78 61 6d 70 6c 65  03 63 6f 6d  00
00 01  00 01
```

Decoded:

| Bytes | Meaning |
|---|---|
| `1a 2b` | ID = 0x1a2b |
| `01 00` | Flags = `0000 0001 0000 0000`: QR=0 (query), Opcode=0, RD=1, everything else 0 |
| `00 01` | QDCOUNT = 1 |
| `00 00` ×3 | ANCOUNT, NSCOUNT, ARCOUNT = 0 |
| `03 77 77 77` | Label of length 3: `www` |
| `07 65 78 61 6d 70 6c 65` | Label of length 7: `example` |
| `03 63 6f 6d` | Label of length 3: `com` |
| `00` | Zero-length label: the root, end of name |
| `00 01` | QTYPE = 1 (A) |
| `00 01` | QCLASS = 1 (IN) |

Names are encoded as a series of length-prefixed labels, with no dots on the wire. The final zero byte is the empty root label, which is the trailing dot made literal.

### The response

The response is 49 bytes:

```
1a 2b  81 80  00 01  00 01  00 00  00 00
03 77 77 77  07 65 78 61 6d 70 6c 65  03 63 6f 6d  00  00 01  00 01
c0 0c  00 01  00 01  00 00 0e 10  00 04  c0 00 02 0a
```

| Bytes | Meaning |
|---|---|
| `1a 2b` | Same ID, so the client can match it |
| `81 80` | Flags = `1000 0001 1000 0000`: QR=1, RD=1, RA=1, RCODE=0 |
| `00 01 00 01 00 00 00 00` | 1 question, 1 answer, 0 authority, 0 additional |
| (21 bytes) | The question, copied back unchanged |
| `c0 0c` | **Compression pointer** to offset 12 (see below) |
| `00 01` | TYPE = A |
| `00 01` | CLASS = IN |
| `00 00 0e 10` | TTL = 0x0e10 = 3600 seconds |
| `00 04` | RDLENGTH = 4 bytes |
| `c0 00 02 0a` | RDATA = 192.0.2.10 |

### Name compression

Repeating `www.example.com` in full in every record would waste space, so RFC 1035 adds **compression pointers**. A length byte whose top two bits are `11` is not a label length. Together with the next byte it forms a 14-bit offset from the start of the message. `c0 0c` is `1100 0000 0000 1100`: the top bits mark it as a pointer, and the offset is 12, exactly where the question's name begins, right after the 12-byte header. Two bytes replace seventeen. In a referral with thirteen NS records all ending in `gtld-servers.net`, compression saves hundreds of bytes.

### Size limits, TCP and EDNS

Classic DNS over UDP limited messages to 512 bytes. A larger response is cut short with the **TC** bit set, and the client retries over TCP (also port 53), where each message is prefixed with a two-byte length.

**EDNS(0)** (RFC 6891) extends the protocol without changing the header. The client adds a pseudo-record of type `OPT` (41) to the additional section. Its CLASS field is reused to advertise the client's UDP buffer size (1232 bytes is the current recommendation, chosen to avoid IP fragmentation). Its TTL field carries extended flags, including the **DO** ("DNSSEC OK") bit. Its RDATA holds options such as cookies, padding and EDNS Client Subnet. Without EDNS, DNSSEC would not fit in UDP at all.

### Encrypted transports

DoT (port 853), DoH (RFC 8484) and DoQ carry these same bytes inside encryption between stub and resolver. They hide queries from the path but do not prove the data is genuine; that is DNSSEC's job.

## 8. DNSSEC: an overview

Plain DNS has no way to prove an answer is genuine. A resolver accepts any response that matches its question, ID and port. In 2008 Dan Kaminsky showed how an attacker could flood a resolver with forged responses and poison its cache for an entire domain. Source port randomisation made this harder, but the real fix is cryptographic. **DNS Security Extensions** (RFC 4033–4035) add **origin authentication and integrity**: a validating resolver can prove an answer came from the zone owner and was not modified. DNSSEC does *not* provide confidentiality. The answers are still readable by anyone.

### The new record types

- **`DNSKEY`**: a zone's public keys, published at the zone apex.
- **`RRSIG`**: a signature over one RRset, with an inception and expiry time.
- **`DS`** (Delegation Signer): a hash of a child zone's key, published in the *parent* zone.
- **`NSEC`/`NSEC3`**: signed proof that a name or type does *not* exist.

### Signing a zone

Our zone, signed, would gain records like these (keys and signatures shortened):

```zone
example.com.  3600 IN DNSKEY 257 3 13 mdsswUyr3DPW...==   ; KSK (flag 257)
example.com.  3600 IN DNSKEY 256 3 13 oJMRESz5E4gY...==   ; ZSK (flag 256)
example.com.  3600 IN RRSIG  DNSKEY 13 2 3600 (
                  20261217000000 20261117000000 31406 example.com.
                  kLQ7...== )
www.example.com. 3600 IN RRSIG CNAME 13 3 3600 (
                  20261217000000 20261117000000 2371 example.com.
                  Pq8v...== )
```

Algorithm 13 is ECDSA P-256 with SHA-256, the common modern choice. The RRSIG fields are: type covered, algorithm, number of labels, original TTL, expiry, inception, key tag, signer name, and the signature itself.

Most zones use two keys:

- The **Zone Signing Key (ZSK)** signs the ordinary RRsets. It can be rolled often, because nothing outside the zone refers to it.
- The **Key Signing Key (KSK)** signs only the `DNSKEY` RRset. Its hash goes to the parent as a `DS` record, so changing it requires coordinating with the registrar.

### The chain of trust

Signatures alone prove nothing unless you trust the key that made them. DNSSEC links keys together down the same tree that delegation uses:

```
Root zone   DNSKEY (KSK) ── trust anchor, built into every validator
   │  signs root DNSKEY set, which contains root ZSK
   │  root ZSK signs:  com. DS
   ▼
.com zone   DNSKEY (KSK) ── must hash to com. DS
   │  .com ZSK signs:  example.com. DS
   ▼
example.com DNSKEY (KSK) ── must hash to example.com. DS
   │  example.com ZSK signs:  www.example.com. CNAME, example.com. A, ...
   ▼
the answer
```

A validating resolver resolving `www.example.com` with DNSSEC asks the same servers as in section 4, but sets the **DO** bit and additionally fetches `DNSKEY` and `DS` sets at each level:

1. Verify the root `DNSKEY` set against the built-in **trust anchor** (the root KSK, key tag 20326 since the 2018 rollover; the next generation, key tag 38696, was introduced in 2024–2025).
2. Use the root ZSK to check the RRSIG over `com. DS`.
3. Fetch `com. DNSKEY`, check that its KSK hashes to that `DS`, and check the RRSIG over the DNSKEY set.
4. Repeat for `example.com. DS` and `example.com. DNSKEY`.
5. Finally, check the RRSIG over the answer RRset with `example.com`'s ZSK.

If every link verifies, the resolver sets the **AD** (Authentic Data) bit in its reply. If any link fails, because of a bad signature, an expired RRSIG or a DS mismatch, the resolver returns **SERVFAIL** rather than data it cannot trust. If the chain ends cleanly at a parent that proves there is *no* DS record for the child, the child is **insecure**: answers are still returned, just without the AD bit. That is how signed and unsigned zones coexist.

### Proving non-existence

Signing "there is nothing here" is harder, since the server cannot sign every possible missing name ahead of time. **NSEC** records solve this by listing each name in the zone together with the next name in canonical order and the types that exist at it:

```zone
mail.example.com. 300 IN NSEC sip.example.com. A RRSIG NSEC
```

A query for `newhost.example.com` falls between `mail` and `sip` in sorted order, so this signed record proves it does not exist. The drawback is that a client can follow the chain and list every name in the zone ("zone walking"). **NSEC3** (RFC 5155) hashes the names first, which makes walking harder, though not impossible, since hashes can be cracked offline. Many large operators instead generate minimal "black lies" or compact denial responses on the fly, signing them in real time.

### Costs and caveats

DNSSEC adds operational weight: signatures expire and must be refreshed, key rollovers must respect TTLs, and a stale DS record at the parent makes a domain vanish for validating users. Responses are larger, too. Most TLDs are signed, but only a minority of second-level domains are. Where it is deployed, it enables DANE (TLSA records), which pins TLS certificates in DNS and is widely used for mail between servers.

## 9. Putting it together

One lookup of `www.example.com` uses all of it: the namespace gives a path, referrals walk it, zone records hold the data, TTL-governed caches make the walk rare, a compact wire format carries it, and DNSSEC proves it genuine. To watch it happen, run `dig +trace +dnssec` on a name you use every day.
