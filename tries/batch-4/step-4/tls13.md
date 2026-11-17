# The TLS 1.3 Handshake, Worked Through

TLS 1.3 (RFC 8446, published 2018) is the protocol that secures most of the web today. It replaced TLS 1.2 with a handshake that is faster (one round trip instead of two), simpler (dozens of legacy options removed), and safer (forward secrecy is mandatory, and most of the handshake is encrypted). This explainer walks through the full handshake one message at a time. It works the key exchange with toy numbers small enough to check by hand, traces the key schedule step by step, explains how the client verifies the server's certificate, and covers session resumption and 0-RTT. It ends with a side-by-side comparison against TLS 1.2.

The examples use the most common configuration: the cipher suite `TLS_AES_128_GCM_SHA256`, the key exchange group X25519 or P-256, and an ECDSA or RSA-PSS server certificate.

---

## 1. What the handshake has to accomplish

Before a single byte of application data moves, client and server need to:

1. **Agree on parameters**: the protocol version, the AEAD cipher and hash, and the key exchange group.
2. **Establish a shared secret** that a passive eavesdropper cannot compute, and that stays secret even if the server's long-term private key later leaks (*forward secrecy*).
3. **Authenticate the server** (and optionally the client), so the client knows it is talking to the real `example.com` and not an attacker in the middle.
4. **Confirm that both sides saw the same handshake**, so an attacker cannot quietly downgrade the version or cipher.
5. **Derive separate traffic keys** for each direction and each phase.

TLS 1.3 does all five in one round trip. It manages that because the client *guesses* which key exchange group the server will accept and sends its key share in the very first message.

---

## 2. The 1-RTT handshake, message by message

Here is the full flow. An asterisk (`*`) marks optional messages, and braces (`{}`) mark messages encrypted with handshake keys. Square brackets (`[]`) mark messages encrypted with application keys.

```
Client                                              Server

ClientHello
  + key_share
  + supported_versions
  + signature_algorithms
  + server_name (SNI)          -------->
                                                ServerHello
                                                  + key_share
                                                  + supported_versions
                                        {EncryptedExtensions}
                                        {CertificateRequest*}
                                        {Certificate}
                                        {CertificateVerify}
                               <--------          {Finished}
                                        [Application Data*]
{Certificate*}
{CertificateVerify*}
{Finished}                     -------->
[Application Data]             <------->  [Application Data]
```

### 2.1 ClientHello

The client opens with a single, unencrypted message containing:

- **`legacy_version` = 0x0303** (TLS 1.2). The real version goes in an extension, because many middleboxes break when they see an unfamiliar version number in this field.
- **`random`**: 32 bytes of fresh randomness.
- **`legacy_session_id`**: up to 32 bytes. In "middlebox compatibility mode" the client fills it with random bytes so that the exchange looks like TLS 1.2 session resumption.
- **`cipher_suites`**: for example `TLS_AES_128_GCM_SHA256`, `TLS_AES_256_GCM_SHA384`, `TLS_CHACHA20_POLY1305_SHA256`. In TLS 1.3 a suite names only the AEAD and the hash. Key exchange and signature algorithms are negotiated separately.
- **Extensions**, of which the important ones are:
  - `supported_versions`: `[0x0304 (TLS 1.3), 0x0303 (TLS 1.2)]`.
  - `supported_groups`: `[x25519, secp256r1, secp384r1]`.
  - `key_share`: one or more actual public keys, for example an X25519 public key of 32 bytes. This is the crucial change from TLS 1.2.
  - `signature_algorithms`: `[ecdsa_secp256r1_sha256, rsa_pss_rsae_sha256, ed25519, ...]`.
  - `server_name`: `example.com`, so a server hosting many sites knows which certificate to present.
  - `psk_key_exchange_modes` and `pre_shared_key`, if the client is attempting resumption (see Section 6).
  - `application_layer_protocol_negotiation` (ALPN): `["h2", "http/1.1"]`.

### 2.2 ServerHello

The server picks a cipher suite and a group from the client's offers. If the client sent a key share for that group, the server replies with:

- `legacy_version` = 0x0303, its own 32-byte `random`, and an echo of `legacy_session_id`.
- The chosen `cipher_suite`, for example `TLS_AES_128_GCM_SHA256`.
- `supported_versions` = `0x0304`. This is how the client learns TLS 1.3 was chosen.
- `key_share`: the server's own ephemeral public key for the chosen group.

At this point **both sides can compute the shared ECDHE secret**. Everything the server sends after ServerHello is encrypted.

> **Downgrade protection.** If a TLS 1.3–capable server ends up negotiating TLS 1.2, it sets the last 8 bytes of its `random` to `44 4F 57 4E 47 52 44 01` ("DOWNGRD\x01"). A TLS 1.3 client that sees this sentinel aborts, because it means an attacker stripped the newer version from the ClientHello.

**HelloRetryRequest.** If the client guessed wrong and sent no key share the server accepts (say it sent only X25519 and the server insists on P-384), the server sends a special ServerHello called HelloRetryRequest, which names the group it wants. The client sends a second ClientHello with the right share. This costs an extra round trip, which is why clients try to guess well.

### 2.3 EncryptedExtensions

This is the first encrypted message. It carries the extensions that don't affect key derivation, such as the selected ALPN protocol (`h2`), the maximum fragment length, and the server's acknowledgement of SNI. In TLS 1.2 these travelled in the clear inside ServerHello.

### 2.4 CertificateRequest (optional)

The server sends this only if it wants client authentication, as in mutual TLS. It lists acceptable signature algorithms and, optionally, acceptable certificate authorities.

### 2.5 Certificate

This message carries the server's certificate chain: the leaf certificate for `example.com`, then any intermediates. The root is normally omitted, since the client already has it. Each entry can carry its own extensions, most usefully a stapled OCSP response and Signed Certificate Timestamps for Certificate Transparency.

### 2.6 CertificateVerify

The server proves that it holds the private key for the leaf certificate by signing the handshake transcript so far. The signed content is:

```
64 bytes of 0x20 (spaces)
|| "TLS 1.3, server CertificateVerify"
|| 0x00
|| Transcript-Hash(ClientHello ... Certificate)
```

The 64 spaces and the context string stop a signature produced here from being reused in any other protocol or context. Because the transcript includes both randoms and both key shares, this signature binds the server's identity to *this* key exchange. In TLS 1.2 the signature covered only the ServerKeyExchange parameters.

### 2.7 Server Finished

The server's Finished message is an HMAC over the whole transcript, computed with a key derived from the server's handshake traffic secret:

```
finished_key = HKDF-Expand-Label(server_handshake_traffic_secret, "finished", "", 32)
verify_data  = HMAC-SHA256(finished_key, Transcript-Hash(ClientHello ... CertificateVerify))
```

This confirms that the server saw exactly the same handshake messages as the client and that it derived the same keys.

Right after sending Finished, the server **may already send application data** (for example HTTP/2 SETTINGS) under the server application traffic key. This is sometimes called "0.5-RTT data". It is sent before the client has been authenticated, which is acceptable for server-originated data in most protocols.

### 2.8 Client Finished

The client verifies the certificate chain, the CertificateVerify signature and the server Finished. If client authentication was requested, it sends its own Certificate and CertificateVerify, and then its own Finished computed with its handshake secret. After this, both sides switch to application traffic keys.

**Total cost:** the client sends ClientHello, gets everything back in one flight, and can send its HTTP request alongside its Finished. That is **one round trip** before the first request. TLS 1.2 needed two.

---

## 3. ECDHE with toy numbers

Real X25519 uses a 255-bit prime. To see the mechanics, use a tiny elliptic curve that you can check with pencil and paper:

```
E: y² = x³ + 2x + 2   over the field F₁₇ (integers mod 17)
Generator G = (5, 1)
Order of G = 19   (19·G = point at infinity)
```

Multiples of G, which you can verify with the standard point-addition formulas:

| k | k·G | k | k·G |
|---|-----|---|-----|
| 1 | (5, 1) | 10 | (7, 11) |
| 2 | (6, 3) | 11 | (13, 10) |
| 3 | (10, 6) | 12 | (0, 11) |
| 4 | (3, 1) | 13 | (16, 4) |
| 5 | (9, 16) | 14 | (9, 1) |
| 6 | (16, 13) | 15 | (3, 16) |
| 7 | (0, 6) | 16 | (10, 11) |
| 8 | (13, 7) | 17 | (6, 14) |
| 9 | (7, 6) | 18 | (5, 16) |

**Checking one entry (2G).** Doubling uses the slope λ = (3x² + a) / (2y), where a = 2 here.

- λ = (3·25 + 2) / (2·1) = 77 / 2 mod 17. Since 77 mod 17 = 9, and the inverse of 2 mod 17 is 9 (because 2·9 = 18 ≡ 1), λ = 9·9 = 81 ≡ 13.
- x₃ = λ² − 2x = 169 − 10 = 159 ≡ 6 (mod 17).
- y₃ = λ(x − x₃) − y = 13·(5 − 6) − 1 = −14 ≡ 3 (mod 17).

So 2G = (6, 3), as the table says.

### The exchange

1. **Client** picks a private scalar `a = 3`. Its public key is `A = 3·G = (10, 6)`, which goes in the ClientHello `key_share`.
2. **Server** picks a private scalar `b = 7`. Its public key is `B = 7·G = (0, 6)`, which goes in the ServerHello `key_share`.
3. **Client** computes `a·B = 3·(7G) = 21G`. Since the order is 19, 21G = 2G = **(6, 3)**.
4. **Server** computes `b·A = 7·(3G) = 21G = 2G` = **(6, 3)**.

Both sides arrive at the same point, and the **shared secret is its x-coordinate, 6**. (X25519 likewise outputs a u-coordinate. For the NIST curves, TLS uses the x-coordinate.)

An eavesdropper sees G, A = (10, 6) and B = (0, 6). To get the secret, it has to recover a or b from a public point: the elliptic-curve discrete logarithm problem. On this toy curve you can read the answer off the table. On Curve25519, with a group of about 2²⁵² elements, the best known attacks need around 2¹²⁶ operations.

**Why "ephemeral" matters.** Both a and b are fresh for each handshake and are deleted afterwards. If the server's certificate key is stolen next year, the attacker still cannot compute 21G for today's session, because the certificate key was only ever used to *sign*. It never encrypted anything. That property is forward secrecy, and TLS 1.3 makes it mandatory: static RSA key transport and static Diffie-Hellman were removed.

---

## 4. The key schedule, step by step

TLS 1.3 derives every key from a chain of HKDF operations (RFC 5869). There are two primitives:

- **`HKDF-Extract(salt, IKM)`** = `HMAC-Hash(salt, IKM)`. It concentrates input keying material into a uniform pseudorandom key.
- **`HKDF-Expand-Label(Secret, Label, Context, Length)`** = `HKDF-Expand(Secret, HkdfLabel, Length)`, where `HkdfLabel` is the structure:

```
struct {
    uint16 length;                // output length
    opaque label<7..255>;         // "tls13 " + Label
    opaque context<0..255>;       // usually a transcript hash
} HkdfLabel;
```

A convenience wrapper is defined as well:

```
Derive-Secret(Secret, Label, Messages) =
    HKDF-Expand-Label(Secret, Label, Transcript-Hash(Messages), Hash.length)
```

### 4.1 The three stages

With SHA-256 (Hash.length = 32), the schedule runs as follows. `0` means a string of 32 zero bytes.

```
                 0
                 |
                 v
   PSK ->  HKDF-Extract = Early Secret
                 |
                 +--> Derive-Secret(., "ext binder" | "res binder", "")
                 |                     = binder_key
                 +--> Derive-Secret(., "c e traffic", ClientHello)
                 |                     = client_early_traffic_secret
                 v
           Derive-Secret(., "derived", "")
                 |
                 v
(EC)DHE -> HKDF-Extract = Handshake Secret
                 |
                 +--> Derive-Secret(., "c hs traffic", ClientHello...ServerHello)
                 |                     = client_handshake_traffic_secret
                 +--> Derive-Secret(., "s hs traffic", ClientHello...ServerHello)
                 |                     = server_handshake_traffic_secret
                 v
           Derive-Secret(., "derived", "")
                 |
                 v
      0 -> HKDF-Extract = Master Secret
                 |
                 +--> Derive-Secret(., "c ap traffic", ClientHello...server Finished)
                 |                     = client_application_traffic_secret_0
                 +--> Derive-Secret(., "s ap traffic", ClientHello...server Finished)
                 |                     = server_application_traffic_secret_0
                 +--> Derive-Secret(., "exp master", ClientHello...server Finished)
                 |                     = exporter_master_secret
                 +--> Derive-Secret(., "res master", ClientHello...client Finished)
                                       = resumption_master_secret
```

### 4.2 Walking through a full (non-PSK) handshake

**Step 1: Early Secret.** With no PSK, the input keying material is 32 zero bytes:

```
early_secret = HKDF-Extract(salt = 0x00 (one zero byte), IKM = 00…00 (32 bytes))
```

This value is the same for every non-PSK handshake using SHA-256. That is fine, because it adds no secrecy on its own. It is just the starting point of the chain.

**Step 2: Salt for the next stage.**

```
derived_1 = Derive-Secret(early_secret, "derived", "")
```

The context here is SHA-256 of the empty string, `e3b0c442 98fc1c14 9afbf4c8 996fb924 27ae41e4 649b934c a495991b 7852b855`. The encoded `HkdfLabel` is:

```
00 20                     length = 32
0d                        label length = 13
74 6c 73 31 33 20 64 65   "tls13 de"
72 69 76 65 64            "rived"
20                        context length = 32
e3 b0 c4 42 … 78 52 b8 55 SHA-256("")
```

**Step 3: Handshake Secret.** Feed in the ECDHE output. In our toy example that was the value 6. In real X25519 it is 32 bytes.

```
handshake_secret = HKDF-Extract(salt = derived_1, IKM = ECDHE_shared_secret)
```

**Step 4: Handshake traffic secrets.** Let H₁ = SHA-256(ClientHello ‖ ServerHello).

```
c_hs = HKDF-Expand-Label(handshake_secret, "c hs traffic", H₁, 32)
s_hs = HKDF-Expand-Label(handshake_secret, "s hs traffic", H₁, 32)
```

**Step 5: Traffic keys and IVs.** Each traffic secret is expanded into an AES-128-GCM key and a 12-byte IV:

```
server_hs_key = HKDF-Expand-Label(s_hs, "key", "", 16)
server_hs_iv  = HKDF-Expand-Label(s_hs, "iv",  "", 12)
client_hs_key = HKDF-Expand-Label(c_hs, "key", "", 16)
client_hs_iv  = HKDF-Expand-Label(c_hs, "iv",  "", 12)
```

The server encrypts EncryptedExtensions through Finished with `server_hs_key`. For each record, the nonce is the IV XORed with the 64-bit record sequence number (left-padded to 12 bytes). The sequence number starts at 0 and resets whenever the key changes.

**Step 6: Master Secret.**

```
derived_2     = Derive-Secret(handshake_secret, "derived", "")
master_secret = HKDF-Extract(salt = derived_2, IKM = 00…00 (32 bytes))
```

**Step 7: Application traffic secrets.** Let H₂ = SHA-256(ClientHello … server Finished).

```
c_ap_0 = HKDF-Expand-Label(master_secret, "c ap traffic", H₂, 32)
s_ap_0 = HKDF-Expand-Label(master_secret, "s ap traffic", H₂, 32)
```

These are expanded with "key" and "iv" exactly as in Step 5.

**Step 8: Resumption secret.** Let H₃ = SHA-256(ClientHello … client Finished).

```
resumption_master_secret = Derive-Secret(master_secret, "res master", H₃)
```

**Key updates.** Long-lived connections can rotate keys without a new handshake by sending a KeyUpdate message:

```
application_traffic_secret_N+1 =
    HKDF-Expand-Label(application_traffic_secret_N, "traffic upd", "", 32)
```

### 4.3 Why it is built this way

- **Every secret is bound to the transcript.** Changing even one bit of ClientHello changes H₁, and with it every key. An attacker who tampers with negotiation produces mismatched keys and a failed Finished check.
- **Phases are separated.** Handshake keys and application keys are independent. Compromising one does not reveal the other.
- **Inputs are slotted in.** PSK enters at the top and ECDHE in the middle. When one is absent, zeros take its place, so the same schedule serves full handshakes, PSK-only resumption and PSK+ECDHE resumption.
- **Directions are separated.** Client and server traffic use different keys, which rules out reflection attacks.

---

## 5. Certificate verification

When the client receives Certificate and CertificateVerify, it performs a series of checks. If any of them fails, it aborts with an alert such as `bad_certificate` or `decrypt_error`.

### 5.1 Build and validate the chain

Suppose the server sends:

```
[0] CN=example.com          issued by  "Example Issuing CA R3"
[1] CN=Example Issuing CA R3 issued by  "Example Root CA X1"
```

The client:

1. **Finds a trust anchor.** It looks up "Example Root CA X1" in its root store (the OS or browser trust store).
2. **Checks each signature.** It verifies that the root's public key validates the signature on [1], and that [1]'s public key validates the signature on [0].
3. **Checks validity periods.** Every certificate must satisfy `notBefore ≤ now ≤ notAfter`.
4. **Checks constraints.** The intermediate must have `basicConstraints: CA=TRUE` and `keyUsage: keyCertSign`. Path-length and name constraints must hold.
5. **Checks the leaf's purpose.** `extendedKeyUsage` must include `serverAuth`.

### 5.2 Check the name

The requested hostname (from SNI) must match a `subjectAltName` DNS entry in the leaf. Modern clients ignore the Common Name. A wildcard `*.example.com` matches `www.example.com` but not `example.com` or `a.b.example.com`.

### 5.3 Check revocation and transparency

- **OCSP stapling.** The server can attach a signed, recent OCSP response in the Certificate entry's `status_request` extension, which saves the client a separate lookup.
- **Certificate Transparency.** Browsers such as Chrome and Safari require Signed Certificate Timestamps proving the certificate was logged publicly. These arrive embedded in the certificate, in a TLS extension, or in the OCSP response.
- Many browsers also use pushed revocation lists (CRLSets, CRLite) rather than live checks.

### 5.4 Verify possession of the key

The leaf certificate holds the server's long-term public key, for example an ECDSA P-256 key. The client rebuilds the signed content (64 spaces, the context string, a zero byte, and its own transcript hash through Certificate) and verifies the CertificateVerify signature with that key, using the algorithm the server named. That algorithm must be one the client offered in `signature_algorithms`.

A worked logic check: suppose an attacker relays the real server's Certificate but substitutes its own key share. The real server never signed a transcript containing the attacker's key share, and the attacker cannot produce a valid signature without the server's private key. So CertificateVerify fails. **The signature is what ties "who you are" to "what key we agreed".**

### 5.5 Verify Finished

Finally the client computes its own expected `verify_data` and compares it with the server's Finished in constant time. A match proves the server derived the same handshake secret, which requires knowing b, the server's ephemeral private key.

---

## 6. Session resumption and 0-RTT

### 6.1 NewSessionTicket

After the handshake completes, the server can send one or more NewSessionTicket messages, encrypted under the application keys:

```
struct {
    uint32 ticket_lifetime;     // seconds, max 7 days
    uint32 ticket_age_add;      // random value to obscure ticket age
    opaque ticket_nonce<0..255>;
    opaque ticket<1..2^16-1>;   // opaque blob, often the PSK encrypted under a server key
    Extension extensions<0..2^16-2>;  // e.g. early_data(max_early_data_size)
} NewSessionTicket;
```

Both sides compute the PSK for that ticket:

```
PSK = HKDF-Expand-Label(resumption_master_secret, "resumption", ticket_nonce, 32)
```

Because each ticket has a distinct nonce, each ticket yields a distinct PSK. Clients are encouraged to use each ticket only once, which keeps resumed connections unlinkable.

### 6.2 Resumed handshake

On the next connection, the client includes:

- `pre_shared_key` with the ticket identity and an **obfuscated ticket age**, which is `(ms since ticket received + ticket_age_add) mod 2³²`.
- A **PSK binder**: an HMAC over the partial ClientHello, keyed from `binder_key = Derive-Secret(early_secret, "res binder", "")`. It proves that the client actually holds the PSK, not just the ticket.
- `psk_key_exchange_modes`, usually `psk_dhe_ke`, meaning "PSK plus a fresh ECDHE". The alternative, `psk_ke`, means PSK alone with no forward secrecy for this session.
- A normal `key_share`.

If the server accepts, it returns `pre_shared_key` with the selected identity index and **omits Certificate and CertificateVerify**, since authentication is inherited from the original session. In the key schedule, the PSK now goes into the Early Secret instead of zeros, and the new ECDHE output still enters at the Handshake Secret. With `psk_dhe_ke`, the resumed session has its own forward secrecy.

The resumed handshake is still 1-RTT, but cheaper: no certificate transfer, no signature to verify, and smaller messages.

### 6.3 0-RTT early data

With a ticket that permits early data, the client can send application data **in its first flight**, alongside ClientHello:

```
Client                                               Server

ClientHello
  + early_data
  + key_share
  + psk_key_exchange_modes
  + pre_shared_key
(Application Data*)  -------->
                                                ServerHello
                                           + pre_shared_key
                                               + key_share
                                      {EncryptedExtensions}
                                              + early_data
                                                 {Finished}
                     <--------          [Application Data*]
(EndOfEarlyData)
{Finished}           -------->
[Application Data]   <------->           [Application Data]
```

Parentheses mark data encrypted with `client_early_traffic_secret`, which is derived only from the PSK and ClientHello. The server can process an HTTP GET before any round trip has finished.

**The catch: 0-RTT data is replayable.** Nothing from the server contributes to the early keys, so an attacker who records ClientHello plus early data can resend it, and the server will decrypt it again. Early data also lacks forward secrecy with respect to the ticket-encryption key. The mitigations are:

- **Application rules.** Only idempotent, safe requests (such as GET without side effects) should be sent as early data. HTTP signals this with the `Early-Data: 1` header and the `425 Too Early` status code (RFC 8470).
- **Single-use tickets** or a server-side ClientHello cache within a freshness window.
- **Ticket age checks.** The server compares the client's reported ticket age with its own expectation, and rejects early data that falls outside a small tolerance.

If the server rejects early data, it simply omits `early_data` from EncryptedExtensions, skips the early records, and the connection continues as a normal 1-RTT resumption. The client then resends the data under the proper keys.

---

## 7. TLS 1.3 compared with TLS 1.2

### 7.1 The TLS 1.2 ECDHE handshake, for reference

```
Client                                    Server
ClientHello              -------->
                                     ServerHello
                                     Certificate
                                     ServerKeyExchange   (ECDHE params, signed)
                         <--------   ServerHelloDone
ClientKeyExchange
ChangeCipherSpec
Finished                 -------->
                                     ChangeCipherSpec
                         <--------   Finished
Application Data         <------->   Application Data
```

The client cannot send its key share until it has seen the server's parameters. That makes the handshake two round trips before the first request (or one with abbreviated resumption). The certificate travels in plaintext.

### 7.2 Side by side

| Aspect | TLS 1.2 | TLS 1.3 |
|---|---|---|
| Full handshake latency | 2-RTT | 1-RTT |
| Resumption latency | 1-RTT | 1-RTT, or 0-RTT with early data |
| Key exchange | RSA key transport, static DH, DHE, ECDHE | (EC)DHE only, or PSK with (EC)DHE, or PSK only |
| Forward secrecy | Optional (absent with RSA key transport) | Mandatory for full handshakes |
| Cipher suites | ~300+ defined, combining kex + auth + cipher + MAC | 5 defined, naming only AEAD + hash |
| Bulk ciphers | CBC, RC4, 3DES, GCM, ChaCha20… | AEAD only: AES-GCM, ChaCha20-Poly1305, AES-CCM |
| Key derivation | Custom PRF over master secret | HKDF with transcript-bound labels |
| What the signature covers | ServerKeyExchange params plus randoms | Entire transcript through Certificate |
| Certificate privacy | Sent in plaintext | Encrypted under handshake keys |
| Renegotiation | Supported (and a source of attacks) | Removed; KeyUpdate and post-handshake auth instead |
| Compression | Supported (CRIME attack) | Removed |
| ChangeCipherSpec | Real protocol message | Dummy only, for middlebox compatibility |
| Resumption mechanism | Session IDs, session tickets | PSK tickets with binders |
| Downgrade protection | Weak (Finished only) | Signed transcript plus ServerHello random sentinel |
| Custom DH groups | Server-chosen, often weak (Logjam) | Fixed named groups only (RFC 7919 FFDHE, curves) |

### 7.3 Attacks the redesign closes

- **POODLE and Lucky Thirteen** (CBC padding oracles): gone, because only AEAD ciphers remain.
- **RC4 biases**: RC4 is gone.
- **CRIME**: compression is gone.
- **Triple Handshake and renegotiation injection**: renegotiation is gone, and the transcript binds everything.
- **FREAK and Logjam** (export-grade downgrades): export suites and custom DH groups are gone, and the signed transcript plus the random sentinel catch downgrades.
- **ROBOT and Bleichenbacher** (RSA key transport oracles): RSA key transport is gone. RSA survives only as a signature scheme, and in TLS 1.3 handshake messages only as RSA-PSS.

### 7.4 What TLS 1.3 does not fix by itself

- **SNI is still visible** in ClientHello. Encrypted Client Hello (ECH) addresses this separately by encrypting the inner ClientHello to a key the server publishes in DNS.
- **0-RTT replay** is a deliberate trade-off that applications must handle.
- **Certificate trust** still depends on the CA ecosystem, Certificate Transparency and revocation.

---

## 8. Putting it all together

Here is one full connection to `example.com`, compressed into a timeline:

1. **t = 0.** The client generates an X25519 key pair (in our toy curve: a = 3, A = (10, 6)) and sends ClientHello with A, SNI `example.com`, and the TLS 1.3 offer.
2. **t = ½ RTT.** The server generates b = 7, B = (0, 6), and computes the shared secret 2G = (6, 3). It runs Extract to get the Handshake Secret and derives `s_hs` and `c_hs`. It sends ServerHello with B, then, under `server_hs_key`, EncryptedExtensions (ALPN h2), its certificate chain, a CertificateVerify signature over the transcript, and Finished. It derives the application secrets and optionally sends 0.5-RTT data.
3. **t = 1 RTT.** The client computes 3·B = (6, 3), derives the same handshake keys, and decrypts the server flight. It validates the chain to a trusted root, checks that `example.com` is in the subjectAltName, verifies the CertificateVerify signature and the Finished HMAC. It sends its Finished and, in the same flight, its HTTP request under `client_application_traffic_secret_0`.
4. **t = 1½ RTT.** The server verifies the client Finished, serves the response, and sends a NewSessionTicket.
5. **Next visit.** The client presents the ticket with a PSK binder and a fresh key share, and optionally sends a safe GET as 0-RTT early data.

Every step from the toy arithmetic to the HKDF labels serves the same goals: an ephemeral shared secret, a transcript that both sides commit to, a signature that binds the server's identity to that transcript, and independent keys for every phase and direction. TLS 1.3 achieves all of this in fewer round trips and with far fewer moving parts than its predecessor, which is why it became the default almost everywhere within a few years of standardisation.
