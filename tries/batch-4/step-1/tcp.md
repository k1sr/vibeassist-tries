# TCP: The Transmission Control Protocol, Explained with Worked Examples

TCP is the protocol behind most of the internet's reliable traffic: web pages over HTTP/1.1 and HTTP/2, email, SSH, database connections, and much more. It runs on top of IP, which delivers individual packets on a "best effort" basis. Packets can be lost, duplicated, reordered, or delayed. TCP turns that unreliable service into an **ordered, reliable, flow-controlled, congestion-aware byte stream** between two processes.

This explainer covers the whole protocol: the segment format, connection setup, data transfer, flow control, retransmission, congestion control, and teardown. Each mechanism comes with a worked example that uses concrete numbers.

---

## 1. What TCP Promises (and What It Doesn't)

TCP provides:

- **Connection orientation.** Two endpoints agree to talk before any data flows, and they keep state about each other for the life of the connection.
- **A reliable byte stream.** Every byte the sender writes arrives exactly once, in order, or the connection reports an error. TCP has no notion of messages. If you write 100 bytes and then 200 bytes, the receiver may read 300 bytes at once, or 50 and then 250.
- **Full duplex.** Each direction is an independent stream with its own sequence numbers.
- **Flow control.** A fast sender cannot overrun a slow receiver's buffer.
- **Congestion control.** Senders back off when the network is overloaded, so the internet doesn't collapse.

TCP does **not** provide message boundaries, guaranteed latency, encryption (that's TLS, layered on top), or multicast.

A connection is identified by a 4-tuple: **(source IP, source port, destination IP, destination port)**. Two connections from the same client to the same server port are distinguished by the client's ephemeral source port.

---

## 2. The TCP Segment Header

Every TCP segment carries a header of at least 20 bytes:

```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|          Source Port          |       Destination Port        |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                        Sequence Number                        |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                    Acknowledgment Number                      |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
| Offset|  Rsv  |C|E|U|A|P|R|S|F|            Window             |
|       |       |W|C|R|C|S|S|Y|I|                               |
|       |       |R|E|G|K|H|T|N|N|                               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|           Checksum            |         Urgent Pointer        |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                    Options (0-40 bytes)                       |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

The fields that matter most:

| Field | Purpose |
|---|---|
| **Sequence number** | Position in the sender's byte stream of the first data byte in this segment. SYN and FIN each consume one sequence number. |
| **Acknowledgment number** | The *next* byte the sender of this segment expects to receive. It acknowledges everything before it (cumulative ACK). Valid only when ACK is set. |
| **Data offset** | Header length in 32-bit words (5 to 15), which tells you where options end. |
| **Flags** | SYN (synchronize/open), ACK, FIN (finish sending), RST (reset/abort), PSH (push data to the application), URG (urgent data, rarely used), ECE and CWR (Explicit Congestion Notification). |
| **Window** | How many bytes, starting at the ACK number, the sender of this segment is willing to receive. |
| **Checksum** | Covers header, data, and a pseudo-header of IP addresses. |

### Common options

- **MSS (Maximum Segment Size):** sent in SYNs; the largest payload the sender wants to receive. On Ethernet with a 1500-byte MTU it's typically 1460 (1500 − 20 IP − 20 TCP).
- **Window Scale:** the 16-bit window field caps out at 65,535 bytes, which is far too small for fast, long-distance links. Window scale (negotiated in the SYNs) left-shifts the advertised window by up to 14 bits, allowing windows of about 1 GiB.
- **SACK-Permitted / SACK:** selective acknowledgment lets the receiver report non-contiguous blocks it holds, so the sender retransmits only what's missing.
- **Timestamps:** used for RTT measurement and for PAWS (Protection Against Wrapped Sequences) on very fast links where 32-bit sequence numbers can wrap within a connection's lifetime.

---

## 3. Opening a Connection: The Three-Way Handshake

Before data flows, both sides must agree on **initial sequence numbers (ISNs)**. ISNs are chosen pseudo-randomly. This defends against attackers guessing sequence numbers, and it keeps segments from an old connection with the same 4-tuple from being mistaken for part of a new one.

### Worked example

A client at 10.0.0.2:51000 connects to a web server at 93.184.216.34:443. The client picks ISN **1000**. The server picks ISN **5000**. (Real ISNs are random 32-bit values; small numbers keep the arithmetic readable.)

```
Client                                              Server
(CLOSED)                                            (LISTEN)

  |  SYN  seq=1000                                     |
  |  win=64240, MSS=1460, WS=7, SACK_PERM, TS          |
  |--------------------------------------------------->|
(SYN-SENT)                                          (SYN-RECEIVED)

  |  SYN+ACK  seq=5000  ack=1001                       |
  |  win=65160, MSS=1460, WS=7, SACK_PERM, TS          |
  |<---------------------------------------------------|
(ESTABLISHED)

  |  ACK  seq=1001  ack=5001                           |
  |--------------------------------------------------->|
                                                    (ESTABLISHED)
```

Step by step:

1. **SYN.** The client sends `seq=1000`. The SYN flag consumes one sequence number, so the client's first data byte will be numbered **1001**. The client also advertises its options.
2. **SYN+ACK.** The server acknowledges with `ack=1001` ("I've got everything up to 1000; send 1001 next"), and sends its own SYN with `seq=5000`.
3. **ACK.** The client acknowledges the server's SYN with `ack=5001`. Its own sequence number is now 1001. A pure ACK carries no data and consumes no sequence number.

Both sides now know:

- Client → server stream starts at byte 1001.
- Server → client stream starts at byte 5001.
- MSS is 1460 in each direction; window scaling (factor 2⁷ = 128), SACK, and timestamps are enabled because *both* sides offered them.

Why three messages and not two? Each side has to send its ISN and get it acknowledged. The server's ACK of the client's SYN is combined with the server's own SYN, so four logical messages become three packets.

### Data after the handshake

The client sends an HTTP request of 500 bytes:

```
Client → Server:  seq=1001  ack=5001  len=500   (bytes 1001–1500)
Server → Client:  seq=5001  ack=1501  len=0     (ACK)
Server → Client:  seq=5001  ack=1501  len=300   (bytes 5001–5300, the response)
Client → Server:  seq=1501  ack=5301  len=0     (ACK)
```

The rule is simple: **next sequence number = seq + payload length (+1 for SYN or FIN).** The ACK number always names the next byte expected.

In practice the server would often merge the ACK into the response segment ("piggybacking"). Delayed ACK, covered below, encourages this.

### SYN floods and SYN cookies

A server in SYN-RECEIVED must remember the half-open connection. Attackers exploit this by sending floods of SYNs from spoofed addresses, filling the backlog queue. **SYN cookies** defend against it. The server encodes the connection parameters (a hash of the 4-tuple, a timestamp, and an MSS index) into its ISN and keeps no state. If a valid ACK returns with `ack = cookie + 1`, the server rebuilds the connection from it.

---

## 4. Reliability: Sequence Numbers, ACKs, and the Receive Buffer

The receiver keeps a buffer and tracks `RCV.NXT`, the next in-order byte it expects. When a segment arrives:

- If it starts at `RCV.NXT`, the data is appended, `RCV.NXT` advances, and any buffered out-of-order data that is now contiguous gets delivered too.
- If it starts beyond `RCV.NXT` (a gap exists), the data is buffered (if within the window), and the receiver immediately sends a **duplicate ACK** for `RCV.NXT`, with SACK blocks describing what it holds.
- If it's entirely below `RCV.NXT`, it's a duplicate. The receiver discards it and re-ACKs.

The sender keeps every unacknowledged byte until a cumulative ACK covers it.

---

## 5. Flow Control: The Sliding Window

The receiver advertises a **receive window (rwnd)**: the free space in its buffer. The sender may have at most `rwnd` bytes "in flight" (sent but not acknowledged). The sender's view of its stream looks like this:

```
          SND.UNA                SND.NXT            SND.UNA + SND.WND
             |                      |                       |
  ... acked  |  sent, not yet acked |  usable, may send now |  not yet allowed
-------------+----------------------+-----------------------+-----------------
```

- `SND.UNA`: oldest unacknowledged byte.
- `SND.NXT`: next byte to send.
- `SND.WND`: the window the peer advertised.
- **Usable window** = `SND.UNA + SND.WND − SND.NXT`.

As ACKs arrive, the left edge slides right. As the receiver's application reads data, buffer space opens up, and the advertised window lets the right edge slide right too.

### Worked example

Use MSS = 1000 bytes for round numbers. The sender's stream starts at byte 1 (relative numbering, the way Wireshark shows it). The receiver has a 4000-byte buffer and advertises `rwnd=4000`.

**t=0.** `SND.UNA=1`, `SND.NXT=1`, window = [1, 4001). Usable = 4000. The sender transmits four segments:

```
Seg A: bytes    1–1000
Seg B: bytes 1001–2000
Seg C: bytes 2001–3000
Seg D: bytes 3001–4000
```

Now `SND.NXT=4001`. Usable window = 1 + 4000 − 4001 = **0**. The sender must stop.

**t=1.** The receiver gets A and B, but its application has read only 1000 bytes. Buffer: 2000 bytes received, 1000 consumed, so 1000 bytes still occupied and 3000 free. It sends:

```
ACK ack=2001 win=3000
```

The sender updates `SND.UNA=2001`, `SND.WND=3000`. The window is now [2001, 5001). Usable = 2001 + 3000 − 4001 = **1000**. The sender sends Seg E: bytes 4001–5000.

**t=2.** The receiver gets C, D, and E, and its application is stalled, reading nothing. The buffer holds 1000 (from before) + 3000 = 4000 bytes, so it is full:

```
ACK ack=5001 win=0
```

This is a **zero window**. The sender may not send new data. It starts a **persist timer** and periodically sends a **window probe** (one byte, or an empty segment) to provoke a fresh window update. Without probes, if the receiver's later "window open" ACK were lost, both sides would wait forever.

**t=3.** The application reads 4000 bytes. The receiver sends `ACK ack=5001 win=4000`, and the transfer resumes.

### Nagle and delayed ACK

**Nagle's algorithm** stops a sender from flooding the network with tiny segments: while unacknowledged data is outstanding, small writes are buffered until a full MSS accumulates or the outstanding data is ACKed. **Delayed ACK** lets a receiver wait briefly (up to 500 ms per the RFC; around 40 ms on Linux) or until every second full segment before ACKing, hoping to piggyback the ACK on response data. Combined with Nagle, this causes a well-known stall. A sender that writes a header and then a small body has the body held by Nagle until the header is ACKed, while the receiver holds the ACK waiting for more data. Latency-sensitive applications set `TCP_NODELAY` to disable Nagle.

---

## 6. Retransmission

TCP detects loss in two ways: a **retransmission timeout (RTO)** and **fast retransmit** on duplicate ACKs.

### Computing the RTO (RFC 6298)

The sender measures round-trip time samples (R) and keeps a smoothed estimate (SRTT) and a variance estimate (RTTVAR):

- First sample: `SRTT = R`, `RTTVAR = R/2`.
- After that:
  - `RTTVAR = (1 − β)·RTTVAR + β·|SRTT − R|`, with β = 1/4
  - `SRTT = (1 − α)·SRTT + α·R`, with α = 1/8
- `RTO = SRTT + max(G, 4·RTTVAR)`, where G is the clock granularity.
- The RFC says RTO should be at least 1 second. Linux uses a 200 ms floor in practice.

Note that RTTVAR is updated *before* SRTT, using the old SRTT.

**Worked example** (assume G is negligible):

| Sample | R | RTTVAR | SRTT | 4·RTTVAR | Raw RTO |
|---|---|---|---|---|---|
| 1 | 100 ms | 50 | 100 | 200 | 300 ms |
| 2 | 120 ms | 0.75·50 + 0.25·20 = 42.5 | 0.875·100 + 0.125·120 = 102.5 | 170 | 272.5 ms |
| 3 | 300 ms | 0.75·42.5 + 0.25·197.5 = 81.25 | 0.875·102.5 + 0.125·300 = 127.2 | 325 | 452.2 ms |

A single slow sample (300 ms) raises the RTO sharply because of the variance term. That's intentional: jittery paths need more slack to avoid spurious retransmissions.

**Karn's algorithm:** never take an RTT sample from a retransmitted segment, because you can't tell which transmission the ACK belongs to. Timestamps remove this ambiguity.

**Exponential backoff:** each time the RTO fires for the same segment, the RTO doubles (300 → 600 → 1200 ms...), up to a maximum. After enough failures (on Linux, governed by `tcp_retries2`, about 15 minutes by default), the connection is aborted.

### Timeout example

```
t=0      Send seq=1001 len=1000     (RTO = 300 ms)
t=300    Timer fires, no ACK. Retransmit seq=1001. RTO → 600 ms.
t=380    ACK ack=2001 arrives. Timer stops; RTO is recomputed
         from the next clean sample.
```

A timeout is expensive. The sender has sat idle for a full RTO, and (as section 7 shows) congestion control treats it as a severe signal.

### Fast retransmit

Waiting for a timeout wastes time. If one segment is lost but later ones arrive, the receiver produces a stream of duplicate ACKs. **Three duplicate ACKs** are taken as strong evidence of loss rather than mere reordering.

**Worked example.** MSS = 1000. The sender transmits segments starting at 1, 1001, 2001, 3001, 4001, 5001. The segment at **2001** is lost.

```
Receiver gets 1–1000      → ACK 1001
Receiver gets 1001–2000   → ACK 2001
(2001–3000 lost)
Receiver gets 3001–4000   → ACK 2001  SACK 3001–4001   (dup #1)
Receiver gets 4001–5000   → ACK 2001  SACK 3001–5001   (dup #2)
Receiver gets 5001–6000   → ACK 2001  SACK 3001–6001   (dup #3)
```

On the third duplicate ACK, the sender immediately retransmits 2001–3000 without waiting for the RTO. When it arrives, the receiver can deliver everything through byte 6000 and replies `ACK 6001`, a single cumulative ACK that covers the retransmitted segment and all the buffered ones.

With **SACK**, the sender knows exactly which ranges are missing. If several segments are lost in one window, it can fill every hole in one round trip rather than discovering them one RTT at a time.

Modern stacks also use **RACK-TLP** (RFC 8985), which detects loss from time ("this segment was sent well before one that has been acknowledged") rather than counting duplicate ACKs, and sends a **Tail Loss Probe** to recover quickly when the last segments of a burst are lost and no duplicate ACKs will ever come.

---

## 7. Congestion Control

Flow control protects the *receiver*. Congestion control protects the *network*. In 1986 the internet suffered "congestion collapse": throughput on some links fell by a factor of about 1000 because senders retransmitted aggressively into overloaded routers. Van Jacobson's fixes (1988) are the basis of what TCP does today.

The sender maintains a **congestion window (cwnd)**. The amount in flight is limited by:

```
in_flight ≤ min(cwnd, rwnd)
```

and a **slow-start threshold (ssthresh)** that separates two growth phases.

### The classic Reno/NewReno algorithm

1. **Slow start** (`cwnd < ssthresh`): cwnd grows by 1 MSS for each ACK, so it **doubles every RTT**. The name is ironic; growth is exponential.
2. **Congestion avoidance** (`cwnd ≥ ssthresh`): cwnd grows by about 1 MSS **per RTT** (additive increase).
3. **On 3 duplicate ACKs (fast retransmit / fast recovery):** `ssthresh = max(FlightSize/2, 2·MSS)`, `cwnd = ssthresh` (temporarily inflated by 3 MSS for the segments that have left the network), then return to congestion avoidance once the loss is repaired. This is the **multiplicative decrease**.
4. **On RTO timeout:** `ssthresh = max(FlightSize/2, 2·MSS)`, `cwnd = 1 MSS` (the loss window), and go back to slow start.

Together these give **AIMD**: additive increase, multiplicative decrease. AIMD is what makes competing TCP flows converge toward a fair share of a bottleneck.

### Worked trace

Units are MSS. To make the shape easy to see, initial cwnd = 1 and initial ssthresh = 32. (Modern stacks start at cwnd = 10 per RFC 6928, and ssthresh starts effectively infinite.)

| RTT round | Phase | cwnd at start of round | Event |
|---|---|---|---|
| 1 | Slow start | 1 | |
| 2 | Slow start | 2 | |
| 3 | Slow start | 4 | |
| 4 | Slow start | 8 | |
| 5 | Slow start | 16 | |
| 6 | Cong. avoid. | 32 | reached ssthresh |
| 7 | Cong. avoid. | 33 | |
| 8 | Cong. avoid. | 34 | |
| 9 | Cong. avoid. | 35 | |
| 10 | Cong. avoid. | 36 | **3 dup ACKs** |
| 11 | Fast recovery → CA | 18 | ssthresh = 36/2 = 18 |
| 12 | Cong. avoid. | 19 | |
| 13 | Cong. avoid. | 20 | |
| 14 | Cong. avoid. | 21 | |
| 15 | Cong. avoid. | 22 | **RTO timeout** |
| 16 | Slow start | 1 | ssthresh = 22/2 = 11 |
| 17 | Slow start | 2 | |
| 18 | Slow start | 4 | |
| 19 | Slow start | 8 | |
| 20 | Cong. avoid. | 11 | capped at ssthresh |
| 21 | Cong. avoid. | 12 | |

Plotted, this gives TCP's characteristic **sawtooth**:

```
cwnd
 36 |                   *
 32 |           *  *  *
    |
 22 |                                *
 18 |                       *  *  *
 16 |        *
 11 |                                            *  *
  8 |     *                                   *
  4 |   *                                   *
  2 |  *                                  *
  1 | *                                 *
    +----------------------------------------------------> RTT
      1  2  3  4  5  6  7  8  9 10 11 12 13 14 15 16 ...
```

Look at the two loss responses. A triple duplicate ACK means data is still flowing, so the network is only mildly congested, and TCP halves its window. A timeout suggests something much worse, so TCP restarts from one segment.

### Fast recovery in detail

In the round 10 event, with cwnd = 36:

1. The third duplicate ACK arrives. ssthresh = 18. Retransmit the lost segment. cwnd = 18 + 3 = 21.
2. Each additional duplicate ACK means another segment has left the network, so cwnd += 1. This lets new segments trickle out and keeps the "ACK clock" running.
3. When the ACK covering the retransmission arrives, deflate: cwnd = ssthresh = 18. Continue in congestion avoidance.

**NewReno** (RFC 6582) improves on this when several segments in one window are lost. A *partial ACK* (one that advances but doesn't cover everything sent before the loss) triggers retransmission of the next hole without leaving fast recovery. Modern stacks replace window inflation with **Proportional Rate Reduction** (RFC 6937), which paces sending more smoothly during recovery.

### Why slow start matters: a quick calculation

Take a 100 ms RTT and a goal of 100 Mbit/s. The bandwidth-delay product is 100×10⁶ × 0.1 / 8 = **1.25 MB**, or about 856 segments of 1460 bytes. Starting at cwnd = 10, slow start needs log₂(856/10) ≈ 6.4, so about **7 RTTs (700 ms)** to fill the pipe. This is why short web transfers rarely reach full link speed, and why a larger initial window and connection reuse matter so much.

### CUBIC

Reno's linear growth of 1 MSS per RTT is painfully slow on high bandwidth-delay paths. Recovering from a halving at 856 segments would take about 428 RTTs, or 43 seconds. **CUBIC** (RFC 9438, the default on Linux, macOS, and Windows) grows cwnd as a cubic function of time since the last loss:

```
W(t) = C·(t − K)³ + W_max
```

Here `W_max` is the window at the last loss, C = 0.4, and K is the time needed to climb back to `W_max`. On a loss, CUBIC multiplies the window by β = 0.7 instead of 0.5. Growth is fast right after the loss, flattens out near `W_max` (probing carefully where loss happened before), and then accelerates again to find new bandwidth. Because growth depends on elapsed time rather than ACK arrivals, flows with different RTTs share bandwidth more fairly.

**CUBIC worked example.** Suppose W_max = 100 segments when loss occurs. The window drops to 70. Then K = ∛(W_max·(1−β)/C) = ∛(100·0.3/0.4) = ∛75 ≈ 4.2 s.

| t (s) | W(t) = 0.4·(t − 4.22)³ + 100 |
|---|---|
| 0 | 70.0 |
| 1 | 86.6 |
| 2 | 95.6 |
| 3 | 99.3 |
| 4.22 | 100.0 (plateau) |
| 5 | 100.2 |
| 6 | 102.3 |
| 7 | 108.6 |

Note the flat region around 100: CUBIC lingers near the point where the network previously dropped packets.

### BBR

Loss-based algorithms such as Reno and CUBIC fill router buffers until they overflow. That causes **bufferbloat**: high queuing delay. Google's **BBR** takes a different approach. It explicitly estimates the bottleneck bandwidth (the maximum delivery rate observed) and the minimum RTT (the propagation delay), then paces packets at the estimated bandwidth with about one BDP in flight. It periodically probes for more bandwidth and periodically drains the queue to re-measure min RTT. BBR achieves high throughput on lossy links where loss-based schemes collapse, though early versions were criticized for being unfair to CUBIC flows. BBRv2/v3 add loss and ECN sensitivity.

---

## 8. Closing a Connection

Because each direction is independent, TCP closes each one separately with **FIN**. A FIN means "I have no more data to send." The side that sent it can still receive. This is a **half-close**.

### Worked example: four-way teardown

Continue the earlier connection. The client has sent bytes 1001–1500. The server has sent bytes 5001–5300. The client's next sequence number is **1501**; the server's is **5301**.

```
Client                                              Server
(ESTABLISHED)                                       (ESTABLISHED)

  |  FIN+ACK  seq=1501  ack=5301                       |
  |--------------------------------------------------->|
(FIN-WAIT-1)                                        (CLOSE-WAIT)

  |  ACK  seq=5301  ack=1502                           |
  |<---------------------------------------------------|
(FIN-WAIT-2)
                     ... server may still send data ...
                                                    (app calls close())
  |  FIN+ACK  seq=5301  ack=1502                       |
  |<---------------------------------------------------|
(TIME-WAIT)                                         (LAST-ACK)

  |  ACK  seq=1502  ack=5302                           |
  |--------------------------------------------------->|
                                                    (CLOSED)
  ... wait 2×MSL ...
(CLOSED)
```

Walking through the numbers:

1. The client's FIN uses sequence number 1501. Like SYN, FIN consumes one sequence number.
2. The server acknowledges with `ack=1502`. The server enters CLOSE-WAIT. Its application will see end-of-file (`read()` returns 0) once it has read the buffered data.
3. When the server's application calls `close()`, it sends its own FIN with `seq=5301`.
4. The client acknowledges with `ack=5302` and enters **TIME-WAIT**.

Steps 2 and 3 are often combined into a single FIN+ACK, which gives a three-segment close.

### Why TIME-WAIT exists

The side that closes first (the "active closer") waits for **2×MSL** (Maximum Segment Lifetime; 60 s total on Linux) before releasing the 4-tuple. There are two reasons:

1. **The final ACK might be lost.** If it is, the server will retransmit its FIN, and the client must still be around to re-ACK it. Otherwise the client would answer with RST, and the server's application would see an error instead of a clean close.
2. **Old duplicates must die.** Delayed segments from this connection could otherwise show up in a new connection that reuses the same 4-tuple and be accepted as valid data.

Busy servers that close connections themselves can collect thousands of TIME-WAIT sockets. Common mitigations are letting clients close first, reusing connections (HTTP keep-alive), and Linux's `tcp_tw_reuse` for outgoing connections, which is safe when timestamps are enabled.

### Abortive close: RST

A **RST** tears down a connection immediately, with no handshake, and discards unsent data. TCP sends one when:

- a segment arrives for a port with no listener ("connection refused");
- a segment arrives for a connection that doesn't exist (for example, after a reboot);
- an application closes a socket that still has unread received data, or explicitly requests an abort (`SO_LINGER` with a zero timeout).

A RST is accepted only if its sequence number falls within the receive window. This makes blind reset attacks harder, and RFC 5961 tightens the check further.

---

## 9. The State Machine at a Glance

```
                         +---------+
                         | CLOSED  |
                         +---------+
          passive open   |       |   active open / send SYN
                         v       v
                  +--------+   +----------+
                  | LISTEN |   | SYN-SENT |
                  +--------+   +----------+
       recv SYN /     |             |  recv SYN+ACK / send ACK
       send SYN+ACK   v             |
               +-------------+      |
               | SYN-RECEIVED|      |
               +-------------+      |
        recv ACK      |             |
                      v             v
                     +---------------+
                     |  ESTABLISHED  |
                     +---------------+
       close / send FIN |        | recv FIN / send ACK
                        v        v
              +------------+   +------------+
              | FIN-WAIT-1 |   | CLOSE-WAIT |
              +------------+   +------------+
       recv ACK  |    | recv FIN      | close / send FIN
                 v    v               v
      +------------+ +---------+   +----------+
      | FIN-WAIT-2 | | CLOSING |   | LAST-ACK |
      +------------+ +---------+   +----------+
       recv FIN |        | recv ACK     | recv ACK
                v        v              v
              +-----------+          +--------+
              | TIME-WAIT |--2MSL--> | CLOSED |
              +-----------+          +--------+
```

On Linux, `ss -tan` shows these states for live sockets. A pile of CLOSE-WAIT sockets almost always means an application bug: the peer has closed, but the local code never called `close()`.

---

## 10. Keepalives, Timers, and Other Details

TCP runs several timers per connection:

| Timer | Purpose |
|---|---|
| Retransmission (RTO) | Resend unacknowledged data. |
| Persist | Probe a zero window. |
| Delayed ACK | Hold an ACK briefly to piggyback or coalesce. |
| Keepalive | Optional. After a long idle period (2 hours by default on Linux), send probes to detect dead peers. |
| TIME-WAIT | 2×MSL before releasing a closed connection. |
| SYN retransmit | Retry an unanswered SYN, with backoff (1 s, 2 s, 4 s...). |

---

## 11. Summary

| Mechanism | Problem solved | Key idea |
|---|---|---|
| Three-way handshake | Agree on ISNs and options | SYN, SYN+ACK, ACK; SYN consumes one sequence number |
| Sequence/ACK numbers | Ordering, duplicates, loss detection | Byte-level numbering; cumulative ACK = next expected byte |
| Sliding window (rwnd) | Receiver overrun | In-flight ≤ advertised window; zero-window probes |
| RTO estimation | Knowing when to give up waiting | SRTT + 4·RTTVAR, Karn's rule, exponential backoff |
| Fast retransmit / SACK / RACK | Slow loss recovery | Infer loss from duplicate ACKs or timing; repair only holes |
| Congestion control (cwnd) | Network collapse, fairness | Slow start, AIMD, CUBIC, BBR, ECN |
| FIN teardown | Graceful, per-direction close | Half-close; FIN consumes one sequence number |
| TIME-WAIT | Lost final ACK, stale duplicates | Active closer waits 2×MSL |
| RST | Abort or refuse | Immediate teardown, validated against the window |

TCP is over forty years old and has been revised continuously in that time. The handshake, sliding window, and AIMD congestion control described here are still the core of the protocol.
