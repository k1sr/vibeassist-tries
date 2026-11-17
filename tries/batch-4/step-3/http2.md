# HTTP/2, Explained with Worked Examples

HTTP/2 is the second major version of the Hypertext Transfer Protocol. It was first published as RFC 7540 in 2015 and revised as RFC 9113 in 2022. Its header compression scheme, HPACK, is specified separately in RFC 7541. HTTP/2 keeps the *semantics* of HTTP exactly as they were: methods, status codes, header fields, URIs and caching rules all work as before. What it changes is the *wire format*: how those semantics are turned into bytes on a connection.

This explainer covers the whole protocol: the connection preface, the binary framing layer, every frame type, streams and their lifecycle, multiplexing, HPACK, flow control, prioritization, server push and error handling. It ends with a comparison against HTTP/1.1 and HTTP/3. Wherever possible it shows real bytes.

---

## 1. Why HTTP/2 exists

HTTP/1.1 is a text protocol with one outstanding request per connection at a time. A client that wants 80 resources from one origin has three options, all poor:

1. **Serialize** the requests on one connection and wait a round trip for each.
2. **Pipeline** the requests by sending several before any response arrives. Responses must still come back *in order*, so one slow response blocks every response behind it (head-of-line blocking). Pipelining was so fragile in practice that browsers shipped it disabled.
3. **Open many connections**. Browsers settled on about six per origin. Each one pays for its own TCP and TLS handshakes and its own slow start, and competes with the others for congestion control.

On top of that, every HTTP/1.1 request repeats its headers in full. A cookie of a couple of kilobytes and a long `user-agent` string are resent on every request, often making the headers larger than the request itself.

HTTP/2 grew out of Google's SPDY experiment and fixes these problems with four ideas:

- **Binary framing.** Messages are cut into length-prefixed frames that are cheap and unambiguous to parse.
- **Multiplexing.** Many independent *streams* share one TCP connection, and their frames interleave freely.
- **Header compression (HPACK).** Repeated headers shrink to one or two bytes.
- **Flow control and prioritization** per stream, so one large download cannot starve everything else.

---

## 2. Starting a connection

### 2.1 Negotiation

Over TLS, which is how browsers always use HTTP/2, the protocol is chosen during the TLS handshake through **ALPN** (Application-Layer Protocol Negotiation). The client offers `h2` and `http/1.1`, and the server picks one. No extra round trip is needed. RFC 9113 requires TLS 1.2 or later, bans renegotiation and compression, and blacklists a set of weak cipher suites.

Cleartext HTTP/2 (`h2c`) exists. It can start with "prior knowledge" (the client simply knows the server speaks h2c), or with the HTTP/1.1 `Upgrade: h2c` mechanism, which RFC 9113 deprecated. Browsers never implemented h2c, but it is common between back-end services, gRPC being the best-known case.

### 2.2 The connection preface

The first bytes the client sends are a fixed 24-byte magic string:

```
PRI * HTTP/2.0\r\n\r\nSM\r\n\r\n
```

In hex:

```
50 52 49 20 2a 20 48 54 54 50 2f 32 2e 30 0d 0a
0d 0a 53 4d 0d 0a 0d 0a
```

It is chosen so that an HTTP/1.1 server reading it sees a nonsense request (method `PRI`, version `HTTP/2.0`) and fails cleanly instead of misreading what follows. A SETTINGS frame must come straight after the magic string. The server's preface is just a SETTINGS frame. Each side must acknowledge the other's SETTINGS.

---

## 3. The frame

Every HTTP/2 frame starts with a fixed 9-byte header:

```
+-----------------------------------------------+
|                 Length (24)                   |
+---------------+---------------+---------------+
|   Type (8)    |   Flags (8)   |
+-+-------------+---------------+-------------------------------+
|R|                 Stream Identifier (31)                      |
+=+=============================================================+
|                   Frame Payload (0...)                      ...
+---------------------------------------------------------------+
```

- **Length**: the payload size in bytes, not counting the 9-byte header. It may not exceed `SETTINGS_MAX_FRAME_SIZE`, which defaults to 16,384 (2^14) and can be raised to 16,777,215 (2^24 − 1).
- **Type**: what kind of frame this is (see below).
- **Flags**: booleans whose meaning depends on the type.
- **R**: a reserved bit, always 0.
- **Stream Identifier**: the stream the frame belongs to. Stream 0 is the connection itself.

### 3.1 Frame types

| Type | Code | Stream | Purpose |
|---|---|---|---|
| DATA | 0x0 | ≠0 | Message body bytes |
| HEADERS | 0x1 | ≠0 | Opens a stream and carries a header block |
| PRIORITY | 0x2 | ≠0 | Priority signal (deprecated in RFC 9113) |
| RST_STREAM | 0x3 | ≠0 | Terminates one stream immediately |
| SETTINGS | 0x4 | 0 | Connection configuration |
| PUSH_PROMISE | 0x5 | ≠0 | Announces a server-pushed stream |
| PING | 0x6 | 0 | Liveness check and RTT measurement |
| GOAWAY | 0x7 | 0 | Graceful or error shutdown of the connection |
| WINDOW_UPDATE | 0x8 | any | Flow-control credit |
| CONTINUATION | 0x9 | ≠0 | Continues a header block too large for one frame |

The important flags are `END_STREAM` (0x1), `ACK` (0x1 on SETTINGS and PING), `END_HEADERS` (0x4), `PADDED` (0x8) and `PRIORITY` (0x20).

Receivers must ignore frame types they do not recognize. That rule is what makes extensions possible: ALTSVC, ORIGIN and PRIORITY_UPDATE were all added later as new frame types.

### 3.2 Worked example: a SETTINGS frame

Say a client wants to turn off server push and set a 6 MiB initial stream window. Each setting is a 16-bit identifier followed by a 32-bit value, so 6 bytes per setting.

```
00 00 0c        Length = 12 (two settings × 6 bytes)
04              Type = SETTINGS
00              Flags = none
00 00 00 00     Stream = 0 (connection)
00 02 00 00 00 00   SETTINGS_ENABLE_PUSH (0x2) = 0
00 04 00 60 00 00   SETTINGS_INITIAL_WINDOW_SIZE (0x4) = 0x600000 = 6,291,456
```

The server acknowledges with an empty SETTINGS frame that has the ACK flag set:

```
00 00 00 04 01 00 00 00 00
```

The defined settings and their defaults:

| ID | Name | Default |
|---|---|---|
| 0x1 | HEADER_TABLE_SIZE | 4,096 |
| 0x2 | ENABLE_PUSH | 1 |
| 0x3 | MAX_CONCURRENT_STREAMS | unlimited (100 or more recommended) |
| 0x4 | INITIAL_WINDOW_SIZE | 65,535 |
| 0x5 | MAX_FRAME_SIZE | 16,384 |
| 0x6 | MAX_HEADER_LIST_SIZE | unlimited (advisory) |

Settings are not negotiated. Each side announces its own limits, and the peer must respect them.

### 3.3 Worked example: a DATA frame

The body `hello` on stream 1, as the last frame of that message:

```
00 00 05        Length = 5
00              Type = DATA
01              Flags = END_STREAM
00 00 00 01     Stream = 1
68 65 6c 6c 6f  "hello"
```

That is 14 bytes in total, 9 of them framing overhead. If the PADDED flag is set, the payload starts with a one-byte pad length and ends with that many zero bytes. Padding hides the real size of the content from anyone observing the traffic.

### 3.4 Other control frames

**PING** always carries 8 opaque bytes. The receiver echoes them back with ACK set:

```
00 00 08 06 00 00 00 00 00  de ad be ef 00 00 00 01   (ping)
00 00 08 06 01 00 00 00 00  de ad be ef 00 00 00 01   (pong)
```

**RST_STREAM** with error code CANCEL (0x8) on stream 3, which is what a browser sends when the user navigates away mid-download:

```
00 00 04 03 00 00 00 00 03  00 00 00 08
```

**GOAWAY** carries the highest stream ID the sender processed or might process, plus an error code. NO_ERROR, last stream 5:

```
00 00 08 07 00 00 00 00 00  00 00 00 05  00 00 00 00
```

Streams above 5 that the client has already sent were never processed, so the client can safely retry them on a new connection. This is what makes graceful server restarts possible. A well-behaved server first sends GOAWAY with last stream ID 2^31−1, waits about one RTT, then sends a second GOAWAY with the real value. This avoids a race with requests that are already in flight.

---

## 4. Streams

A **stream** is an independent, bidirectional sequence of frames within the connection, carrying one request/response exchange. Its properties:

- **IDs**: client-initiated streams use odd numbers (1, 3, 5, …) and server-initiated (pushed) streams use even numbers. A new stream must have an ID higher than every earlier stream the same endpoint opened. IDs are never reused, so a long-lived connection that runs out of IDs (2^31−1) must be replaced.
- **Concurrency** is capped by the peer's `SETTINGS_MAX_CONCURRENT_STREAMS`.
- **Cheap creation**: a stream is opened by sending a HEADERS frame on a new ID. No handshake is needed.

### 4.1 Stream states

```
                         +--------+
                 send PP |        | recv PP
                ,--------+  idle  +--------.
               /         |        |         \
              v          +--------+          v
       +----------+          |           +----------+
       | reserved |          | send H /  | reserved |
       | (local)  |          | recv H    | (remote) |
       +----------+          v           +----------+
           |             +--------+             |
           |     recv ES |        | send ES     |
   send H  |     ,-------+  open  +-------.     | recv H
           |    /        |        |        \    |
           v   v         +--------+         v   v
       +----------+          |           +----------+
       |   half   |          |           |   half   |
       |  closed  |          | send R /  |  closed  |
       | (remote) |          | recv R    | (local)  |
       +----------+          |           +----------+
            |                |                 |
            | send ES /      |       recv ES / |
            | send R /       v        send R / |
            | recv R     +--------+   recv R   |
            `----------->|        |<-----------'
                         | closed |
                         +--------+
   H = HEADERS, PP = PUSH_PROMISE, ES = END_STREAM, R = RST_STREAM
```

A typical GET goes like this. The client sends HEADERS with END_STREAM, so the stream is *half-closed (local)* from the client's side. The server sends HEADERS and then DATA, and the last DATA frame carries END_STREAM, which closes the stream.

### 4.2 HTTP messages as frames

An HTTP/2 message is:

1. One HEADERS frame, plus any CONTINUATION frames, with the header block.
2. Zero or more DATA frames with the body.
3. Optionally, a trailing HEADERS frame (trailers, which gRPC uses for its status).

The request line and status line become **pseudo-headers**, whose names start with a colon and which must come before regular headers: `:method`, `:scheme`, `:authority`, `:path` for requests, and `:status` for responses. All header names must be lowercase. Connection-specific headers (`connection`, `keep-alive`, `transfer-encoding`, `upgrade`) are forbidden, because framing already does their job. `te: trailers` is the one value of `te` that is allowed.

---

## 5. Multiplexing: a timeline

Here a browser fetches `index.html`, then finds `style.css` (small), `app.js` (medium) and `hero.jpg` (large) while parsing it. All requests share one TCP connection. Time runs downward. C→S is client to server and S→C is server to client.

```
 t   dir   frame                         stream  notes
 --  ---   ---------------------------   ------  -------------------------------
  0  C→S   preface + SETTINGS               0
  0  C→S   WINDOW_UPDATE (+15 MB)           0    widen connection window
  0  C→S   HEADERS GET /index.html  [ES]    1
  1  S→C   SETTINGS, SETTINGS ACK           0
  1  S→C   HEADERS :status 200              1
  1  S→C   DATA (16 KB)                     1
  1  C→S   SETTINGS ACK                     0
  2  S→C   DATA (9 KB) [ES]                 1    index.html complete
  2  C→S   HEADERS GET /style.css   [ES]    3    discovered while parsing
  2  C→S   HEADERS GET /app.js      [ES]    5
  2  C→S   HEADERS GET /hero.jpg    [ES]    7
  3  S→C   HEADERS 200                      7    image headers ready first
  3  S→C   DATA (16 KB)                     7
  3  S→C   HEADERS 200                      3
  3  S→C   DATA (4 KB) [ES]                 3    style.css complete
  3  S→C   HEADERS 200                      5
  3  S→C   DATA (16 KB)                     5
  3  S→C   DATA (16 KB)                     7
  4  S→C   DATA (12 KB) [ES]                5    app.js complete
  4  S→C   DATA (16 KB)                     7
  …  S→C   DATA …                           7
  9  S→C   DATA (3 KB) [ES]                 7    hero.jpg complete
```

Some things to notice:

- Three requests went out back-to-back at t=2 with no waiting.
- The server answered **out of order**. Stream 7's headers went first, and stream 3 finished before stream 5, even though 5 was requested first. In HTTP/1.1 that would require three connections.
- Frames of different streams interleave at frame granularity. A 4 KB stylesheet does not wait behind a 2 MB image.
- The server decides the interleaving order. That is the job of prioritization (section 8).

Multiplexing does *not* remove TCP-level head-of-line blocking. If one TCP segment holding part of stream 7 is lost, the kernel holds back every later byte, including frames for streams 3 and 5, until the retransmission arrives. HTTP/3 was created to fix this (section 10).

---

## 6. HPACK header compression

SPDY first compressed headers with DEFLATE. The CRIME attack (2012) showed that compressing secret data together with attacker-controlled data leaks the secret through the compressed length. HPACK was designed to resist this. It does not compress across arbitrary byte strings. Instead it uses:

1. A **static table** of 61 common header fields, fixed by the spec.
2. A **dynamic table**, a FIFO of recently sent fields, kept in sync on both sides of each direction of the connection.
3. **Huffman coding** of literal strings, using a static code tuned for HTTP headers.

The two tables share one index space. Indices 1 to 61 are the static table, and 62 and up are the dynamic table, with 62 being the most recently inserted entry.

Some static table entries:

| Index | Name | Value |
|---|---|---|
| 1 | :authority | |
| 2 | :method | GET |
| 3 | :method | POST |
| 4 | :path | / |
| 5 | :path | /index.html |
| 6 | :scheme | http |
| 7 | :scheme | https |
| 8 | :status | 200 |
| 13 | :status | 404 |
| 24 | cache-control | |
| 31 | content-type | |
| 58 | user-agent | |

### 6.1 Representations

Each header field is encoded in one of these forms, identified by its leading bits:

| Leading bits | Representation | Effect on dynamic table |
|---|---|---|
| `1xxxxxxx` | Indexed header field (7-bit index) | none |
| `01xxxxxx` | Literal **with** incremental indexing (6-bit name index, 0 = literal name) | value is added |
| `0000xxxx` | Literal **without** indexing (4-bit name index) | none |
| `0001xxxx` | Literal **never indexed** (4-bit name index) | none, and intermediaries must not index it either |
| `001xxxxx` | Dynamic table size update (5-bit) | resizes the table |

String literals have a 1-bit `H` flag (Huffman or not) followed by a 7-bit-prefix length, then the bytes.

**Never indexed** exists for security. Low-entropy secrets such as short `authorization` tokens or cookies should not enter the dynamic table, because an attacker who can inject headers could otherwise probe for them by watching how large the encoded output is.

### 6.2 Integer encoding

Integers use an N-bit prefix. If the value fits in the prefix (it is less than 2^N − 1), it is stored there directly. If not, the prefix is filled with ones and the remainder follows in 7-bit groups, least significant first, with the high bit as a continuation flag.

Encoding 1337 with a 5-bit prefix:

```
prefix max = 31          → first byte low 5 bits = 11111      → 0x1f (if other bits 0)
1337 − 31 = 1306
1306 mod 128 = 26, continue → 1 0011010                        → 0x9a
1306 / 128 = 10,  stop      → 0 0001010                        → 0x0a
result: 1f 9a 0a
```

### 6.3 Worked example: first request

The request to encode:

```
:method: GET
:scheme: https
:path: /
:authority: www.example.com
```

The encoding, without Huffman so the bytes are easy to read:

```
82                         indexed, index 2  → :method: GET
87                         indexed, index 7  → :scheme: https
84                         indexed, index 4  → :path: /
41                         01 000001: literal w/ indexing, name = index 1 (:authority)
0f                         H=0, length 15
77 77 77 2e 65 78 61 6d    "www.exam"
70 6c 65 2e 63 6f 6d       "ple.com"
```

That is 20 bytes. With Huffman coding the 15-byte hostname shrinks to 12 (`H=1`, length 12 → `8c`):

```
82 87 84 41 8c f1 e3 c2 e5 f2 3a 6b a0 ab 90 f4 ff
```

That is 17 bytes for the whole request header block. The equivalent HTTP/1.1 request line plus Host header (`GET / HTTP/1.1\r\nHost: www.example.com\r\n`) is 39 bytes.

After this block, both sides' dynamic tables hold one entry:

```
[62] :authority: www.example.com     size = 10 + 15 + 32 = 57 bytes
```

An entry's size is defined as name length + value length + 32. The 32 approximates per-entry overhead. The default table holds 4,096 bytes, and older entries are evicted from the end when space is needed.

### 6.4 Worked example: second request on the same connection

```
:method: GET
:scheme: https
:path: /
:authority: www.example.com
cache-control: no-cache
```

```
82                         :method: GET
87                         :scheme: https
84                         :path: /
be                         1 0111110: indexed, index 62 → :authority: www.example.com
58                         01 011000: literal w/ indexing, name = index 24 (cache-control)
08                         H=0, length 8
6e 6f 2d 63 61 63 68 65    "no-cache"
```

That is 14 bytes, or 12 with Huffman (`58 86 a8 eb 10 64 9c bf` for the last field). The whole `:authority` field is now **one byte**. The dynamic table now holds:

```
[62] cache-control: no-cache          53 bytes
[63] :authority: www.example.com      57 bytes
```

A third identical request costs exactly **five bytes**: `82 87 84 bf be`. Here `bf` is index 63 (`:authority`), which moved down when the new entry took index 62. This is where HPACK's real savings come from. Large, repetitive `cookie` and `user-agent` headers drop to a byte or two on every request after the first.

### 6.5 Adding a custom value

To send `user-agent: curl/8` and also store it in the table:

```
7a                         01 111010: literal w/ indexing, name index 58 (user-agent)
06                         H=0, length 6
63 75 72 6c 2f 38          "curl/8"
```

### 6.6 Putting the block in a frame

The 17-byte Huffman block from 6.3, sent as stream 1 with END_STREAM and END_HEADERS:

```
00 00 11        Length = 17
01              Type = HEADERS
05              Flags = END_STREAM (0x1) | END_HEADERS (0x4)
00 00 00 01     Stream = 1
82 87 84 41 8c f1 e3 c2 e5 f2 3a 6b a0 ab 90 f4 ff
```

A response of `:status: 200` is a single byte, `88`. A 404 is `8d`.

If a header block is larger than the maximum frame size, it is split over a HEADERS frame without END_HEADERS followed by CONTINUATION frames. The last CONTINUATION carries END_HEADERS. No other frame, for any stream, may appear in between. That rule keeps HPACK decoding stateful and ordered, and it also opened the door to the 2024 "CONTINUATION flood" denial-of-service bugs, where servers buffered endless CONTINUATION frames.

The fact that the dynamic table changes on every header block means header blocks *must* be decoded in the exact order they were sent. Over TCP that is guaranteed. It is also why HPACK cannot be reused as-is over QUIC.

---

## 7. Flow control

TCP flow control protects the receiver's *connection* buffer. HTTP/2 needs more than that: a proxy multiplexing many clients onto one back-end connection must stop one slow consumer from filling the shared pipe. So HTTP/2 adds **credit-based flow control at two levels**, per stream and for the connection as a whole.

The rules:

- It applies **only to DATA frames**. Headers and control frames are never flow-controlled, so a SETTINGS or PING can always get through.
- Every stream, and the connection, starts with a send window of 65,535 bytes.
- `SETTINGS_INITIAL_WINDOW_SIZE` changes the starting window of *streams* only. The connection window can only grow through WINDOW_UPDATE on stream 0.
- Sending a DATA frame of N bytes uses up N from **both** the stream window and the connection window. The frame's whole payload counts, padding included.
- The receiver grants more credit with WINDOW_UPDATE, which carries a 31-bit increment from 1 to 2^31−1.
- A window may never exceed 2^31−1. Overflowing it is a FLOW_CONTROL_ERROR.

### 7.1 Worked example: a 100 KB response with default windows

The server sends a 102,400-byte body on stream 1. Both windows start at 65,535. The maximum frame size is 16,384.

```
action                              stream-1 window   connection window
----------------------------------  ---------------   -----------------
start                                     65,535            65,535
S→C DATA 16,384                           49,151            49,151
S→C DATA 16,384                           32,767            32,767
S→C DATA 16,384                           16,383            16,383
S→C DATA 16,383                                0                 0
   -- server blocked; 37,249 bytes still to send --
C→S WINDOW_UPDATE stream 0, +32,768            0            32,768
C→S WINDOW_UPDATE stream 1, +32,768       32,768            32,768
S→C DATA 16,384                           16,384            16,384
S→C DATA 16,384                                0                 0
   -- blocked again; 4,481 bytes left --
C→S WINDOW_UPDATE stream 0, +32,768            0            32,768
C→S WINDOW_UPDATE stream 1, +32,768       32,768            32,768
S→C DATA 4,481 [END_STREAM]               28,287            28,287
```

The WINDOW_UPDATE for stream 1 with +32,768 (0x8000) looks like this on the wire:

```
00 00 04 08 00 00 00 00 01  00 00 80 00
```

With default windows, throughput is capped at roughly 64 KB per round trip. On a 100 ms path that is about 5 Mbit/s no matter how fast the link is. That is why real clients raise their windows at once. Chrome, for example, advertises a 6 MiB stream window and sends a connection-level WINDOW_UPDATE of about 15 MB right after the preface:

```
00 00 04 08 00 00 00 00 00  00 ef 00 01      (+15,663,105 on stream 0)
```

### 7.2 Changing INITIAL_WINDOW_SIZE mid-connection

If a receiver changes `SETTINGS_INITIAL_WINDOW_SIZE` from 65,535 to 16,384, the sender adjusts **every open stream's window by the difference** (−49,151). A stream that had 20,000 remaining drops to −29,151. The negative value is legal, and the sender simply cannot send on that stream until WINDOW_UPDATEs bring it back above zero.

### 7.3 Strategy is left to implementations

The spec defines the mechanism, not the policy. Receivers decide when to send updates. A common approach is to replenish once half the window has been consumed, and to grow windows automatically from measured bandwidth-delay product (BDP), as gRPC does with PING-based BDP estimation. A receiver that wants to apply backpressure, for example a proxy whose downstream client is slow, simply stops sending updates.

---

## 8. Prioritization

With dozens of streams in flight, the server has to choose which frames to send next. RFC 7540 defined a **dependency tree**: each stream could depend on a parent stream with a weight from 1 to 256, carried in HEADERS (PRIORITY flag) or in standalone PRIORITY frames. Siblings shared bandwidth in proportion to their weights, and children waited for their parents.

In practice the scheme failed. Browsers built very different trees (Chrome a linear chain, Firefox phantom "group" nodes), many servers ignored the tree or implemented it wrongly, and the reprioritization messages raced against one another. RFC 9113 **deprecated** it. Its replacement is **RFC 9218, Extensible Priorities**, which works in both HTTP/2 and HTTP/3. It uses a header plus an optional PRIORITY_UPDATE frame:

```
priority: u=1, i
```

`u` is urgency, from 0 (highest) to 7 (lowest), default 3. `i` marks a resource as incremental, meaning it can be used as bytes arrive (progressive images, for example), so it should be interleaved round-robin with its peers rather than sent one after another.

---

## 9. Server push, errors, and security

### 9.1 Server push

A server could anticipate requests by sending PUSH_PROMISE on an existing client stream. The promise carried a synthetic request header block and reserved a new even-numbered stream, and the response followed on that stream. The idea was to save the round trip in which the browser discovers `style.css`.

Push almost never helped. Servers could not tell what the browser already had in cache, pushed bytes competed with more important responses, and measured gains were negligible or negative. Chrome removed support in 2022, and most other clients never enabled it or have also dropped it. The `103 Early Hints` informational response, which lets the server suggest preloads that the client fetches itself, has largely taken its place. Clients should send `SETTINGS_ENABLE_PUSH = 0`.

### 9.2 Errors

There are two scopes:

- **Stream errors** end one stream with RST_STREAM. The connection survives.
- **Connection errors** send GOAWAY and close the connection. Examples are an HPACK decoding failure (COMPRESSION_ERROR), because the shared table state is now unreliable, or a frame on stream 0 that is not allowed there.

Error codes: NO_ERROR (0x0), PROTOCOL_ERROR (0x1), INTERNAL_ERROR (0x2), FLOW_CONTROL_ERROR (0x3), SETTINGS_TIMEOUT (0x4), STREAM_CLOSED (0x5), FRAME_SIZE_ERROR (0x6), REFUSED_STREAM (0x7), CANCEL (0x8), COMPRESSION_ERROR (0x9), CONNECT_ERROR (0xa), ENHANCE_YOUR_CALM (0xb), INADEQUATE_SECURITY (0xc), HTTP_1_1_REQUIRED (0xd).

REFUSED_STREAM is special: it promises that the stream was not processed at all, so the client can safely retry even a non-idempotent request.

### 9.3 Denial-of-service lessons

Multiplexing lets a single connection carry a lot of work, and attackers have taken advantage of that:

- **Rapid Reset (CVE-2023-44487)**: the client opens a stream with HEADERS and immediately sends RST_STREAM. A reset stream no longer counts against MAX_CONCURRENT_STREAMS, so one connection could make a server start millions of requests per second. It drove record-size DDoS attacks in 2023. The fix is to track and limit resets per connection.
- **CONTINUATION flood (2024)**: endless CONTINUATION frames without END_HEADERS exhausted server memory or CPU.
- Earlier variants include **HPACK bomb** (a small header block that expands to a huge one), **ping flood**, **settings flood** and **zero-window** attacks that pin server buffers.

A robust implementation limits decoded header size, control-frame rates, reset rates and idle zero-window streams.

---

## 10. Comparison with HTTP/1.1 and HTTP/3

### 10.1 The same request three ways

**HTTP/1.1** (text, roughly 80 bytes before cookies, repeated in full every time):

```
GET / HTTP/1.1\r\n
Host: www.example.com\r\n
Cache-Control: no-cache\r\n
\r\n
```

**HTTP/2** (binary HEADERS frame, 9 + 12 = 21 bytes on a warm connection, 9 + 5 = 14 bytes once `cache-control` is indexed).

**HTTP/3** (HEADERS frame inside a QUIC stream, QPACK-encoded, similarly compact).

### 10.2 Feature table

| | HTTP/1.1 | HTTP/2 | HTTP/3 |
|---|---|---|---|
| Spec | RFC 9112 | RFC 9113 | RFC 9114 |
| Transport | TCP | TCP | QUIC over UDP |
| Encryption | optional TLS | TLS in practice (h2c rare) | always TLS 1.3, built into QUIC |
| Wire format | text | binary frames | binary frames on QUIC streams |
| Concurrency | 1 in-flight per connection, ~6 connections | many streams per connection | many streams per connection |
| HOL blocking | HTTP and TCP level | TCP level only | none across streams |
| Header compression | none | HPACK | QPACK |
| Flow control | TCP only | per stream + connection (HTTP layer) | per stream + connection (QUIC layer) |
| Setup latency (new) | TCP 1 RTT + TLS 1–2 RTT | same as HTTP/1.1 | 1 RTT combined; 0-RTT on resumption |
| Connection migration | no | no | yes (connection IDs survive IP changes) |
| Prioritization | n/a | tree (deprecated) → RFC 9218 | RFC 9218 |
| Server push | no | yes (now effectively dead) | specified, rarely used |
| Discovery | default | ALPN `h2` | `Alt-Svc` header or HTTPS DNS record, ALPN `h3` |

### 10.3 What HTTP/3 changes

HTTP/3 keeps HTTP/2's model of streams and frames but moves it onto **QUIC**, a transport that runs over UDP and has independent streams built in. The effects:

- **No cross-stream head-of-line blocking.** QUIC numbers and retransmits each stream on its own, so a lost packet carrying part of `hero.jpg` delays only `hero.jpg`. On lossy mobile networks this is HTTP/3's main advantage over HTTP/2. On clean networks, HTTP/2 over one TCP connection can match or beat it.
- **Some framing moves into the transport.** HTTP/3 has no stream IDs in its frame headers, no WINDOW_UPDATE, no RST_STREAM and no PING. QUIC provides all of them. HTTP/3 frames use variable-length integers in place of HTTP/2's fixed 9-byte header.
- **QPACK replaces HPACK.** HPACK assumes header blocks arrive in order, which independent QUIC streams no longer guarantee. QPACK sends dynamic table insertions on a dedicated unidirectional encoder stream. Each header block states the table state it needs, and the encoder can choose to reference only acknowledged entries, trading some compression for zero blocking.
- **Faster setup and mobility.** The transport and TLS 1.3 handshakes are combined, and connection IDs let a phone move from Wi-Fi to cellular without dropping the connection.

HTTP/2 is far from obsolete. It is the default for TLS traffic everywhere, it is the transport under gRPC, and many networks rate-limit or block UDP. Clients usually connect over HTTP/2 first and upgrade to HTTP/3 once `Alt-Svc` tells them it is available. Many also race the two and use whichever connects first.

---

## 11. Summary

HTTP/2 keeps HTTP's meaning and replaces its encoding:

- A **9-byte frame header** (length, type, flags, stream ID) wraps everything, with ten frame types covering data, headers, settings, liveness, flow control and teardown.
- **Streams** carry independent request/response exchanges over one connection. Client streams use odd IDs and server streams even IDs, and frames of different streams interleave freely.
- **HPACK** turns repeated headers into one-byte indexes through a 61-entry static table and a synchronized dynamic table. Literals are Huffman-coded, and secrets can be marked never-indexed.
- **Flow control** gives every stream and the connection a credit window, 65,535 bytes by default, which real clients enlarge right away.
- **Prioritization** and **server push** were the least successful parts of the design. The first has been replaced by RFC 9218, and the second is effectively dead.
- The one problem HTTP/2 could not solve, TCP head-of-line blocking, is why **HTTP/3** exists.

To read real HTTP/2 traffic, run `nghttp -nv https://example.com` or open a TLS key log in Wireshark. Each frame shown above appears exactly as described: nine header bytes, then a payload.
