# QUIC: How the Transport Under HTTP/3 Works

QUIC is a general-purpose, connection-oriented, encrypted transport protocol that runs over UDP. Google created it around 2012. The IETF then reworked it heavily and standardized it in 2021 as RFC 9000 (the core transport), RFC 9001 (using TLS 1.3 to secure QUIC), and RFC 9002 (loss detection and congestion control). RFC 8999 describes the version-independent properties, and RFC 9369 defines QUIC version 2. HTTP/3 (RFC 9114) is the best-known application on top of it, but QUIC is a transport in its own right. DNS-over-QUIC, SMB, and media protocols such as Media over QUIC use it too.

This explainer covers what QUIC is for and how its pieces fit together, then walks through worked examples: connection setup, packet number spaces, streams, loss recovery, connection migration, and head-of-line blocking.

---

## 1. Why QUIC Exists

The classic web stack is HTTP/2 over TLS over TCP. It has three structural problems:

1. **Handshake latency.** TCP needs one round trip (SYN, SYN-ACK) before any data can flow. TLS 1.3 then adds another round trip, and TLS 1.2 added two. A fresh HTTPS connection therefore costs at least 2 RTTs before the first request byte arrives.
2. **Head-of-line blocking.** TCP delivers one ordered byte stream. HTTP/2 multiplexes many requests over that stream, so one lost TCP segment stalls *every* request until it is retransmitted, even requests whose bytes all arrived.
3. **Ossification.** Middleboxes such as NATs, firewalls, and "optimizing" proxies inspect and rewrite TCP headers. That made it nearly impossible to deploy new TCP features; TCP Fast Open and Multipath TCP both struggled for years. Changing TCP also means changing operating system kernels.

QUIC addresses all three. It combines the transport and cryptographic handshakes, so setup takes 1 RTT, or 0 RTT on resumption. It provides independent streams, so a loss on one stream does not block the others. It encrypts almost everything, including most transport headers, so middleboxes cannot come to depend on them. And because it runs over UDP in user space, implementations ship as libraries that can be updated as often as a browser.

---

## 2. Architecture at a Glance

| Layer | TCP stack | QUIC stack |
|---|---|---|
| Application | HTTP/2 | HTTP/3 |
| Security | TLS 1.3 (records) | TLS 1.3 handshake only; QUIC does the packet protection |
| Transport | TCP | QUIC |
| Network | IP | UDP / IP |

QUIC does not use TLS records. It takes the TLS 1.3 handshake messages, carries them in its own `CRYPTO` frames, and uses the keys TLS derives to protect QUIC packets directly with AEAD ciphers such as AES-128-GCM or ChaCha20-Poly1305.

### Key concepts

- **Connection:** shared state between a client and a server, identified by **connection IDs (CIDs)** rather than the IP/port 4-tuple.
- **Packet:** the unit of transmission. One UDP datagram can carry several QUIC packets ("coalescing"). Each packet has a packet number and carries one or more frames.
- **Frame:** the unit of meaning inside a packet. Examples are `STREAM`, `ACK`, `CRYPTO`, `MAX_DATA`, `PING`, `PADDING`, `NEW_CONNECTION_ID`, `PATH_CHALLENGE`, and `CONNECTION_CLOSE`.
- **Stream:** an ordered, reliable byte stream inside a connection. A connection can carry thousands of streams.

### Packet types

QUIC has two header forms:

- **Long header**, used during connection setup. It carries the version and both the destination and source CIDs. Its types are `Initial`, `0-RTT`, `Handshake`, and `Retry`. `Version Negotiation` is a special long-header packet with version 0.
- **Short header** ("1-RTT"), used once the handshake completes. It is minimal: a flags byte, the destination CID, a packet number of 1 to 4 bytes, and the protected payload. The flags byte includes the **spin bit**, an optional, deliberately exposed bit that lets on-path observers estimate RTT, and the **key phase** bit.

Payloads are encrypted, and **header protection** also masks the packet number and some flag bits using a sample of the ciphertext. An observer cannot even see packet numbers. This stops middleboxes from inferring loss or reordering and building behavior around it.

---

## 3. Worked Example: 1-RTT Connection Setup

Suppose a client is connecting to a server for the first time, and the RTT is 50 ms. Times are measured from the client's first send.

```
 t (ms)  Client                                                Server
 ------  ------                                                ------
   0     UDP datagram (padded to >= 1200 bytes):
           Initial[pn=0]: CRYPTO(ClientHello,
                          incl. quic_transport_parameters)
         --------------------------------------------------->
  25                                                           Receives ClientHello.
                                                               Derives handshake keys.
                                                               UDP datagram (coalesced):
                                                                 Initial[pn=0]:   ACK(0), CRYPTO(ServerHello)
                                                                 Handshake[pn=0]: CRYPTO(EncryptedExtensions,
                                                                                  Certificate, CertVerify,
                                                                                  Finished)
                                                                 1-RTT[pn=0]:     STREAM (optional
                                                                                  "0.5-RTT" data)
         <---------------------------------------------------
  50     Verifies certificate, derives 1-RTT keys.
         UDP datagram (coalesced):
           Initial[pn=1]:   ACK(0)
           Handshake[pn=0]: ACK(0), CRYPTO(Finished)
           1-RTT[pn=0]:     STREAM(id=0, "GET /index.html")
         --------------------------------------------------->
  75                                                           Verifies client Finished.
                                                               Handshake confirmed.
                                                               1-RTT[pn=1]: HANDSHAKE_DONE,
                                                                            ACK(0),
                                                                            STREAM(id=0, response...)
         <---------------------------------------------------
 100     Receives response bytes.
```

Things to notice:

- **The request leaves at t = 50 ms, one RTT after the start.** With TCP plus TLS 1.3, the TCP handshake would take 0–50 ms, the TLS handshake 50–100 ms, and the request would leave at 100 ms. QUIC saves a full round trip.
- **Initial packets are encrypted, but not secret.** Their keys are derived from the client's chosen Destination Connection ID with a fixed, published salt and HKDF. Anyone who sees the packet can decrypt it. This is obfuscation and integrity against accidental corruption, not confidentiality. Handshake packets use keys from the TLS key exchange and are properly confidential.
- **The 1200-byte padding** lets the client find out whether the path can carry 1200-byte UDP datagrams. It also gives the server a budget for the **anti-amplification limit**: until it has validated the client's address, a server may send at most **3 times** the bytes it has received. That stops attackers from using QUIC servers to flood a spoofed victim. In this example the server's first flight must fit in about 3600 bytes. A large certificate chain can exceed that, and then the server has to wait for more client bytes before sending the rest.
- **Transport parameters** travel inside the TLS handshake, so they are authenticated. They include the initial flow-control windows (`initial_max_data`, `initial_max_stream_data_*`), stream limits (`initial_max_streams_bidi`/`uni`), `max_idle_timeout`, `max_ack_delay`, `active_connection_id_limit`, and others.
- **Coalescing** puts packets from different encryption levels in one UDP datagram, which saves datagrams and round trips.
- **0.5-RTT data:** the server can send application data in its first flight, under 1-RTT keys, before it has verified the client's Finished message. This is useful for server push-like patterns, but the server does not yet know who the client is.

### Optional: Retry

If a server is under load or wants to verify the client's address before committing state, it can answer the first Initial with a **Retry** packet that contains an opaque token. The client then resends its Initial with the token included. That costs one extra RTT, and in return the server can confirm the client really owns its source address. The server can also hand out tokens in a `NEW_TOKEN` frame for future connections, which avoids the Retry round trip next time.

### Optional: Version Negotiation

If the server does not support the version in the client's Initial, it replies with a Version Negotiation packet that lists the versions it does support. The client then retries with one of them. RFC 9368 adds compatible version negotiation, which lets the upgrade happen without an extra round trip.

---

## 4. Worked Example: 0-RTT Resumption

During the earlier connection, the server sent a TLS `NewSessionTicket` that allows early data. Now the client reconnects:

```
 t (ms)  Client                                                Server
 ------  ------                                                ------
   0     UDP datagram:
           Initial[pn=0]: CRYPTO(ClientHello + PSK + early_data)
           0-RTT[pn=0]:   STREAM(id=0, "GET /index.html")
         --------------------------------------------------->
  25                                                           Validates ticket, accepts early data.
                                                               Processes GET immediately.
                                                               Initial[pn=0]:   ACK, CRYPTO(ServerHello)
                                                               Handshake[pn=0]: CRYPTO(EE, Finished)
                                                               1-RTT[pn=0]:     ACK(0-RTT pn 0),
                                                                                STREAM(id=0, response...)
         <---------------------------------------------------
  50     Response bytes arrive at 1 RTT.
         Handshake[pn=0]: CRYPTO(Finished)
         --------------------------------------------------->
```

The request leaves at t = 0, and response data arrives one RTT later, which is the theoretical minimum for request-response.

The 0-RTT packets are protected with keys derived from the resumption secret, and they have the same weaknesses as TLS 1.3 early data:

- **No forward secrecy** for the early data with respect to the ticket key.
- **Replay is possible.** An attacker can capture the first datagram and resend it to the server, possibly to a different server in the same cluster. Applications must send only idempotent requests in 0-RTT. HTTP/3 servers commonly accept only safe methods such as GET in early data and may reply with `425 Too Early`.
- **The server may reject 0-RTT.** The connection still completes as a 1-RTT handshake, and the client retransmits the early data's contents in 1-RTT packets.

The client must also remember the server's previous transport parameters, because 0-RTT data is sent before the new ones arrive. It is limited by the remembered flow-control limits.

---

## 5. Packet Number Spaces

TCP numbers bytes. QUIC numbers **packets**, and it keeps three separate **packet number spaces**:

| Space | Packet types | Keys |
|---|---|---|
| Initial | Initial | Derived from client's DCID |
| Handshake | Handshake | TLS handshake secrets |
| Application Data | 0-RTT and 1-RTT | Early secret / 1-RTT secrets |

Each space starts at packet number 0 and increases strictly. **Packet numbers are never reused**, not even for retransmissions. An ACK in one space acknowledges only packets in that same space, and ACKs are sent at the matching encryption level: Initial packets are ACKed in Initial packets, and so on.

### Why separate spaces?

Different encryption levels are installed and discarded at different times. If one space covered all levels, a receiver could see packet 7 (1-RTT) before it can decrypt packet 5 (Handshake), and it would wrongly read the gap as loss. Separate spaces keep each level's loss detection self-contained. When the handshake completes, the endpoint simply **discards** the Initial and Handshake spaces and their keys. Anything still in flight there no longer matters.

### Worked example

Take the 1-RTT trace from Section 3. After it, the client's state looks like this:

```
Initial space:      sent {0, 1}       received {0}       -> discarded after Handshake keys in use
Handshake space:    sent {0}          received {0}       -> discarded at handshake confirmation
Application space:  sent {0}          received {0, 1}
```

Suppose the server's 1-RTT packet number 0 (the 0.5-RTT data) had been lost. When the client receives application packet 1, it sees a gap at 0 in the application space only. That gap tells it nothing about the Handshake space, where everything arrived.

### Why "no reuse" matters

TCP has the **retransmission ambiguity** problem. When an ACK arrives for a retransmitted segment, the sender cannot tell whether it acknowledges the original or the retransmission, so RTT samples are corrupted and Karn's algorithm has to discard them. In QUIC, a retransmission carries the same *frames* in a *new packet* with a new number. Every ACK refers to exactly one transmission, so every ACK gives a clean RTT sample.

### Packet number encoding

On the wire, packet numbers are truncated to 1–4 bytes. The receiver reconstructs the full 62-bit value as the candidate closest to `largest_received + 1`. For example, if the largest number received so far is `0xa82f30ea` and a 2-byte encoded value `0x9b32` arrives, the receiver decodes it as `0xa82f9b32`. The sender chooses an encoding length large enough to cover twice the range of unacknowledged packets.

---

## 6. Streams and STREAM Frames

### Stream IDs

Stream IDs are 62-bit integers. The two least significant bits encode the stream type:

| Low bits | Type |
|---|---|
| `0x0` | Client-initiated, bidirectional |
| `0x1` | Server-initiated, bidirectional |
| `0x2` | Client-initiated, unidirectional |
| `0x3` | Server-initiated, unidirectional |

The client's bidirectional streams are therefore 0, 4, 8, 12, and so on. The server's unidirectional streams are 3, 7, 11, and so on. Opening a stream takes no handshake: an endpoint just sends a frame on a new ID, within the limit its peer has granted through `MAX_STREAMS`. HTTP/3 maps each request to a client-initiated bidirectional stream. It uses unidirectional streams for its control stream and for the QPACK encoder and decoder streams.

### STREAM frame format

The frame type ranges from `0x08` to `0x0f`, and its three low bits are flags:

- `0x04` **OFF**: an Offset field is present. If absent, the offset is 0.
- `0x02` **LEN**: a Length field is present. If absent, the frame extends to the end of the packet.
- `0x01` **FIN**: this frame ends the stream.

```
STREAM Frame {
  Type (i) = 0x08..0x0f,
  Stream ID (i),
  [Offset (i)],
  [Length (i)],
  Stream Data (..),
}
```

`(i)` marks a **variable-length integer**. Its two high bits select a length of 1, 2, 4, or 8 bytes, which leaves 6, 14, 30, or 62 bits for the value.

### Worked example: sending a 3,000-byte body

A client uploads 3,000 bytes on stream 4. With roughly 1,200 bytes of payload available per packet, it might send:

```
Packet 10: STREAM type=0x0a (LEN)          id=4          len=1180  data[0..1179]
Packet 11: STREAM type=0x0e (OFF|LEN)      id=4 off=1180 len=1180  data[1180..2359]
Packet 12: STREAM type=0x0f (OFF|LEN|FIN)  id=4 off=2360 len=640   data[2360..2999]  FIN
```

The first frame leaves out the offset because it is 0. Because each frame carries its own offset, the receiver can place the bytes correctly in any arrival order. If packet 11 arrives last, bytes 0–1179 are delivered at once. Bytes 2360–2999 are buffered until the gap fills, but only on **stream 4**. Data on other streams keeps flowing.

### Flow control

QUIC has flow control at two levels, both based on credit:

- **Per stream:** `MAX_STREAM_DATA(stream_id, max)` caps the highest offset the sender may send on that stream.
- **Per connection:** `MAX_DATA(max)` caps the sum of the highest offsets across all streams.

A sender that has hit a limit sends `STREAM_DATA_BLOCKED` or `DATA_BLOCKED` to signal it, which is useful for debugging and tuning. The number of streams is limited separately by `MAX_STREAMS`, with matching `STREAMS_BLOCKED`.

Here is an example. The connection limit is 10,000 bytes, and each stream's limit is 4,000. The client has sent 4,000 bytes on stream 0 and 4,000 on stream 4. It may still send only 2,000 more bytes in total, on stream 8 or any other stream with remaining stream credit. When the server's application reads data from its buffers, the server sends `MAX_DATA(20000)` to raise the ceiling.

### Stream termination

- **FIN**: graceful end of the sending direction.
- **`RESET_STREAM(id, error, final_size)`**: abrupt end of the sending direction. Data not yet delivered is discarded.
- **`STOP_SENDING(id, error)`**: the receiver asks the peer to stop sending. The peer typically answers with `RESET_STREAM`.

The final size has to be consistent across all of these, so that connection-level flow control stays accurate.

---

## 7. Loss Detection and Recovery

RFC 9002 defines QUIC's recovery. It borrows ideas from TCP (SACK, RACK, Tail Loss Probe, F-RTO) and simplifies them, because packet numbers are never reused and ACKs are richer.

### ACK frames

An ACK frame contains the **Largest Acknowledged** packet number, an **ACK Delay** (how long the receiver held the ACK before sending it), and a list of **ACK Ranges**. The ranges work much like unlimited SACK blocks. ACKs are never retracted: once a packet is acknowledged, it stays acknowledged. That is unlike TCP's SACK reneging. Receivers usually send an ACK after every second ack-eliciting packet, or after at most `max_ack_delay` (25 ms by default). They ACK immediately when they see a packet out of order.

### RTT estimation

```
latest_rtt   = ack_time - send_time_of_largest_acked
adjusted_rtt = latest_rtt - ack_delay   (only if result >= min_rtt)
smoothed_rtt = 7/8 * smoothed_rtt + 1/8 * adjusted_rtt
rttvar       = 3/4 * rttvar + 1/4 * |smoothed_rtt - adjusted_rtt|
```

Before any sample exists, `smoothed_rtt` defaults to 333 ms.

### Declaring loss

A packet is declared lost if it is unacknowledged, it was sent *before* an acknowledged packet, and either of these holds:

- **Packet threshold:** at least `kPacketThreshold = 3` later packets have been acknowledged.
- **Time threshold:** it was sent more than `9/8 × max(smoothed_rtt, latest_rtt)` before the most recent acknowledged packet, with a minimum of `kGranularity = 1 ms`.

### Probe Timeout (PTO)

If no ACK arrives at all, for example because the tail of a burst was lost, there is nothing to trigger the thresholds above. In that case the PTO timer fires after:

```
PTO = smoothed_rtt + max(4 * rttvar, kGranularity) + max_ack_delay
```

When it fires, the sender transmits one or two **probe packets** that elicit an ACK. These can carry new data or retransmitted frames. The PTO then doubles for each consecutive expiry. Unlike TCP's RTO, a PTO **does not by itself declare anything lost** and does not collapse the congestion window. Losses are still decided by the thresholds once ACKs return. This avoids the spurious-RTO problems TCP needed F-RTO to fix.

### Worked trace

Assume `smoothed_rtt = 40 ms`, `rttvar = 5 ms`, and `max_ack_delay = 25 ms`. The client sends application packets 20–27 on stream 8, one every 1 ms, starting at t = 0:

```
t=0..7     send pn 20,21,22,23,24,25,26,27 (each carries 1000 bytes of stream 8)
           pn 22 is dropped by the network.

t=41       ACK frame: Largest=23, Ranges: [23..23], gap, [20..21]
           -> 20,21,23 acked. RTT sample for pn 23 = 41-3 = 38 ms.
           -> pn 22 is unacked; only 1 later packet (23) acked -> not lost by count.
           -> time threshold = 9/8 * max(40, 38) = 45 ms; pn 22 sent at t=2,
              pn 23 acked; 41 - 2 = 39 ms < 45 ms -> not lost by time yet.
              Loss timer armed for t = 2 + 45 = 47 ms.

t=43       ACK: Largest=25, Ranges [23..25], [20..21]
           -> 3 packets (23,24,25) acked after pn 22 -> PACKET THRESHOLD MET.
           -> pn 22 declared lost at t=43 (before the 47 ms timer).

t=43       Sender queues the lost frame: STREAM(id=8, off=2000, len=1000).
           It is sent in a NEW packet: pn 28.
           Congestion controller: loss event -> cwnd reduced (NewReno: halve),
           enter recovery period starting at pn 28's send time.

t=45,47    ACKs for 26, 27 arrive. Further losses of packets sent before
           the recovery start would not reduce cwnd again.

t=83       ACK: Largest=28 -> retransmitted data acknowledged.
           Receiver delivered stream 8 bytes 2000..7999 to the app at ~t=63
           when pn 28 arrived; streams other than 8 were never blocked.
```

Three points stand out in this trace:

1. The data from pn 22 went out again in pn 28. **Frames are retransmitted, not packets.** If stream 8 had been reset in the meantime, the sender would just drop the frame instead of sending it again.
2. The ACK for pn 28 cannot be confused with an ACK for pn 22, so its RTT sample is valid.
3. If pn 27, the last packet, had also been lost and no further data sent, the PTO (40 + 20 + 25 = 85 ms after the last ack-eliciting send) would have sent a probe. The resulting ACK would then show the gap.

### Congestion control

RFC 9002 specifies a NewReno-style controller: slow start, congestion avoidance, and a recovery period during which the window is cut once per round trip of losses. It also defines **persistent congestion**. If every packet sent over a period longer than `(smoothed_rtt + max(4*rttvar, kGranularity) + max_ack_delay) × 3` is lost, the window collapses to its minimum of 2 × max_datagram_size. In practice, implementations often use CUBIC or BBR, which is easy to do because the controller lives in user space. ECN is supported: ACK frames can carry ECT(0), ECT(1), and CE counts.

---

## 8. Connection Migration

A TCP connection is bound to its 4-tuple, so when a phone moves from Wi-Fi to cellular, every TCP connection dies. A QUIC connection is identified by **connection IDs**, so it can survive a change of address.

### Connection IDs

Each endpoint issues CIDs to its peer through `NEW_CONNECTION_ID` frames. Each frame carries a sequence number and a 16-byte **stateless reset token**. The peer puts these CIDs in the Destination Connection ID field of the packets it sends. Endpoints should use a **fresh CID** on each new path. If they reused one, an observer could link the old path to the new one and track the user across networks. Old CIDs are retired with `RETIRE_CONNECTION_ID`.

Load balancers can route on the CID instead of the 4-tuple. The QUIC-LB draft defines ways to encode a server ID in the CID so that packets reach the right backend after migration.

### Path validation

Before sending significant data on a new path, an endpoint must check that the peer is reachable there. Otherwise an attacker could forge the source address and point a server's traffic at a victim.

- Send `PATH_CHALLENGE` with 8 bytes of unpredictable data.
- The peer echoes those bytes in `PATH_RESPONSE`.
- Until the path is validated, the anti-amplification limit (3×) applies to it.

### Worked example: Wi-Fi to cellular

```
Client on Wi-Fi 192.0.2.10:5000 talks to server 203.0.113.5:443 using server CID S1.
Client already holds spare server-issued CIDs S2, S3 (via NEW_CONNECTION_ID).

t=0     Wi-Fi drops. Client OS now routes via cellular, address 198.51.100.7:6000.

t=1     Client -> server (from 198.51.100.7:6000), DCID=S2:
          1-RTT[pn=150]: PATH_CHALLENGE(0x4b1d...),
                         STREAM(id=12, more upload data)
        Client resets its congestion controller and RTT estimate for the
        new path (they belong to the old path).

t=35    Server receives packet from an unknown address with a CID it issued.
        It decrypts successfully (proves it's the same peer's keys), so:
          - Treats this as a probable migration.
          - Sends from 203.0.113.5:443 to 198.51.100.7:6000:
              1-RTT[pn=210]: PATH_RESPONSE(0x4b1d...),
                             PATH_CHALLENGE(0x9e77...), ACK(150)
          - Limits itself to 3x the bytes received on this path until validated.

t=70    Client receives PATH_RESPONSE matching its challenge -> new path valid
        from client's view. Replies with PATH_RESPONSE(0x9e77...).

t=105   Server receives PATH_RESPONSE -> path valid. Full sending rate resumes
        (subject to a fresh congestion window). Client retires CID S1
        with RETIRE_CONNECTION_ID(seq of S1).
```

Streams, flow-control state, and keys all carry over. The application sees, at worst, a short pause.

Some related cases:

- **NAT rebinding:** a NAT may silently change the client's public port. The server sees a new 4-tuple with the same CID and validates the path in the same way. The client may not even notice.
- **Probing:** an endpoint can send probing frames (`PATH_CHALLENGE`, `PATH_RESPONSE`, `NEW_CONNECTION_ID`, `PADDING`) on a candidate path without migrating, to test it first.
- **Server preferred address:** a server can name a `preferred_address` in its transport parameters, such as a unicast address behind an anycast front, and the client migrates there after the handshake. Apart from that mechanism, RFC 9000 permits only clients to initiate migration.
- **`disable_active_migration`:** a server can signal that it cannot support migration, for example because of its load balancer setup.
- Multipath QUIC, an IETF draft, extends this so that several paths can be used *at the same time*.

---

## 9. Head-of-Line Blocking: QUIC vs TCP

### The scenario

A browser fetches three resources over one connection: `A` (CSS), `B` (JS), and `C` (image). Each is 3 packets long, and the sender interleaves them round-robin:

```
Send order:  A1 B1 C1 A2 B2 C2 A3 B3 C3
Lost:              ^^ C1
```

### HTTP/2 over TCP

TCP sees one byte stream. Segment C1 occupies, say, bytes 2400–3599. When it is lost, the receiving kernel buffers everything after byte 2399 and **delivers nothing past it** to the application until C1 is retransmitted:

```
t=0      A1 B1 [C1 lost] A2 B2 C2 A3 B3 C3 arrive
         Delivered to HTTP/2: A1 B1 only.
         A2, B2, A3, B3 sit in the kernel buffer despite being complete.
t=1 RTT  Duplicate ACKs / RACK trigger retransmit of C1.
t=1.5RTT C1 arrives. Kernel releases C1 A2 B2 C2 A3 B3 C3 at once.
         A and B complete only now.
```

Resources **A and B were delayed by about one RTT because of a loss in C**, even though none of their own bytes were lost. On lossy mobile networks (1–2% loss), this is why HTTP/2 can perform *worse* than HTTP/1.1 with six parallel connections.

### HTTP/3 over QUIC

Each resource lives on its own QUIC stream, and each STREAM frame carries its own offset:

```
t=0      Packets arrive; packet containing C1 lost.
         Stream A: offsets 0..end contiguous -> delivered immediately; A completes.
         Stream B: contiguous -> delivered; B completes.
         Stream C: gap at offset 0 -> C2, C3 buffered (stream C only).
t=~1 RTT C1's frame is re-sent in a new packet; stream C completes.
```

**Only C waits.** A and B finish as if no loss had happened. The CSS and JS can be parsed and executed while the image fills in.

### Remaining caveats

- **Within one stream**, QUIC is still ordered. A loss blocks later bytes on that stream, which is inherent to reliable ordered delivery.
- **Header compression:** HPACK in HTTP/2 needs in-order processing across streams. HTTP/3 therefore uses **QPACK**, which moves dynamic table updates onto a dedicated stream and lets the encoder control how much cross-stream blocking it risks (`SETTINGS_QPACK_BLOCKED_STREAMS`).
- **Congestion control is per connection**, so a loss on stream C still shrinks the window shared by every stream. HOL elimination applies to *delivery*, not to *sending rate*.
- For data that should not be retransmitted at all, such as real-time media, the **DATAGRAM extension** (RFC 9221) adds unreliable, congestion-controlled datagrams inside the encrypted QUIC connection.

---

## 10. Other Mechanisms Worth Knowing

- **Key update:** after the handshake, either side can roll the 1-RTT keys by flipping the **Key Phase** bit. The new keys are derived from the current secrets, and no handshake messages are needed. AEAD confidentiality and integrity limits make this mandatory for long connections. With AES-GCM, an endpoint must update keys before encrypting about 2^23 packets.
- **Idle timeout:** a connection closes silently once `max_idle_timeout` passes with no activity. `PING` frames keep it alive.
- **Connection close:** `CONNECTION_CLOSE` comes in two variants. Type `0x1c` signals transport errors, and type `0x1d` signals application errors. After sending or receiving it, an endpoint enters a short **closing/draining** period of three times the PTO and does not create new state.
- **Stateless reset:** if a server has lost the state for a connection, for instance after a reboot, it cannot send an authenticated `CONNECTION_CLOSE`. It instead sends a packet that looks random and ends with the 16-byte stateless reset token tied to that CID. The peer recognizes the token and tears down the connection.
- **Greasing and version 2:** RFC 9287 lets endpoints randomize the fixed "QUIC bit", and QUIC v2 (RFC 9369) changes salts and packet type codepoints. Both exist to keep middleboxes from ossifying around version 1's exact wire image.
- **Datagram size:** the minimum is 1200 bytes. DPLPMTUD (RFC 8899) can probe for larger sizes using padded packets.

---

## 11. Summary

| Problem | TCP + TLS | QUIC |
|---|---|---|
| New connection latency | 2 RTT (TLS 1.3) | 1 RTT |
| Resumed connection | 1 RTT + 0-RTT data (TFO rarely works) | 0 RTT |
| Head-of-line blocking | Across all multiplexed streams | Only within a single stream |
| Retransmission ambiguity | Yes (Karn's algorithm) | No (packet numbers never reused) |
| Network change | Connection dies | Connection migrates via CIDs |
| Header visibility | Plaintext TCP header | Nearly everything encrypted |
| Deployment | OS kernel | User-space library |

QUIC is not free. It costs more CPU per byte than kernel TCP, mainly because of user-space UDP I/O and per-packet cryptography, though GSO/GRO and kernel offloads have narrowed that gap. Some enterprise networks block UDP 443, so clients must fall back to TCP. And operators lose the passive visibility they used to get from TCP headers. Even so, for latency-sensitive, multiplexed, mobile-heavy traffic, QUIC's design choices hold up well. It now carries a large share of web traffic, and new transport-layer work increasingly targets it rather than TCP.
