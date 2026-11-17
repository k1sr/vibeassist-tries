# Bloom Filters: A Complete, Worked Explainer

A Bloom filter is a small, fast data structure that answers one question: **"Have I seen this item before?"** It can give two answers:

- **"Definitely not."** This answer is always correct.
- **"Probably yes."** This answer is sometimes wrong.

That asymmetry is the whole idea. You give up certainty on the "yes" side, and in exchange you store a set of a million items in about a megabyte, whatever the size of the items: URLs, user IDs, DNA k-mers or full documents. The filter never stores the items themselves. It stores only a pattern of bits that the items leave behind.

Burton Bloom described the structure in 1970 in *Space/Time Trade-offs in Hash Coding with Allowable Errors*. His example was a hyphenation program: most words followed simple rules, and a compact in-memory filter could rule out the rare hard cases that needed a slow disk lookup. The same pattern is everywhere today: put a cheap, probabilistic check in front of an expensive, exact one.

This document builds the idea up from a 16-bit example, derives the false-positive rate and checks it with real numbers, shows how to size a filter, covers the counting and scalable variants, surveys real deployments, and ends with a working implementation.

---

## 1. The Structure

A Bloom filter has two parts:

1. A **bit array** of length *m*, with every bit set to 0 at the start.
2. **k independent hash functions**, *h₁ … hₖ*, each mapping an item to a position in `[0, m)`.

It supports two operations:

- **Insert(x):** compute *h₁(x) … hₖ(x)* and set each of those *k* bits to 1.
- **Query(x):** compute the same *k* positions. If **any** of them is 0, *x* was never inserted. If **all** of them are 1, *x* was *probably* inserted.

The reasoning behind "definitely not" is simple. Inserting *x* sets all *k* of its bits, and nothing ever clears a bit. So if one of *x*'s bits is still 0, *x* cannot have been inserted. False negatives are impossible.

The reasoning behind "probably" is just as simple. A bit can be 1 because some *other* item set it. If every one of *x*'s bits happens to have been set by other items, the filter reports *x* as present when it is not. That is a **false positive**.

---

## 2. A Worked Example With a 16-Bit Filter

To keep the arithmetic visible we use *m* = 16 bits and *k* = 3 hash functions. Real hash functions produce large pseudo-random numbers. Here we simply state what each one returns after reduction mod 16, so every step can be checked by hand.

### The hash values

| Item  | h₁ | h₂ | h₃ |
|-------|----|----|----|
| `cat` | 1  | 6  | 11 |
| `dog` | 4  | 6  | 13 |
| `fox` | 9  | 11 | 15 |
| `owl` | 2  | 6  | 9  |
| `emu` | 1  | 9  | 13 |

We will insert `cat`, `dog` and `fox`, then query `dog`, `owl` and `emu`.

### Starting state

```
index:  0  1  2  3  4  5  6  7  8  9 10 11 12 13 14 15
bits:   0  0  0  0  0  0  0  0  0  0  0  0  0  0  0  0
```

### Insert `cat` → set bits 1, 6, 11

```
index:  0  1  2  3  4  5  6  7  8  9 10 11 12 13 14 15
bits:   0  1  0  0  0  0  1  0  0  0  0  1  0  0  0  0
```

### Insert `dog` → set bits 4, 6, 13

Bit 6 is already 1 because of `cat`. Setting it again changes nothing. This is a **collision**, and collisions are why the filter stays compact and also why it makes mistakes.

```
index:  0  1  2  3  4  5  6  7  8  9 10 11 12 13 14 15
bits:   0  1  0  0  1  0  1  0  0  0  0  1  0  1  0  0
```

### Insert `fox` → set bits 9, 11, 15

Bit 11 is already set by `cat`.

```
index:  0  1  2  3  4  5  6  7  8  9 10 11 12 13 14 15
bits:   0  1  0  0  1  0  1  0  0  1  0  1  0  1  0  1
```

Three items wrote nine bit positions, but only **7 distinct bits** are set: 1, 4, 6, 9, 11, 13 and 15. The two collisions absorbed the other two writes.

### Query `dog` → check bits 4, 6, 13

All three are 1, so the answer is **"probably yes."** That is correct, because `dog` was inserted. This is a **true positive**.

### Query `owl` → check bits 2, 6, 9

Bits 6 and 9 are 1, but **bit 2 is 0**. The answer is **"definitely not,"** and that is correct. The lookup could stop at the first 0 bit. That early exit is why negative lookups, the common case in most deployments, are often cheaper than positive ones.

### Query `emu` → check bits 1, 9, 13

- Bit 1 was set by `cat`.
- Bit 9 was set by `fox`.
- Bit 13 was set by `dog`.

All three are 1, so the filter says **"probably yes."** But `emu` was never inserted. This is a **false positive**: three different items together covered all of `emu`'s bits.

### How likely was that?

Seven of 16 bits are set, so the fill ratio is 7/16 = 0.4375. If a new item's three positions are effectively random, the chance that all three land on set bits is:

```
(7/16)³ = 0.4375³ ≈ 0.0837  →  about 8.4%
```

A 16-bit filter holding three items gives a false positive for roughly one query in twelve. That is far too high for real use, but the mechanism is the same at every scale. The next section derives the rate in general.

---

## 3. Deriving the False-Positive Probability

Assume the hash functions behave like independent, uniformly random choices over the *m* positions. We insert *n* items with *k* hashes each, for *kn* bit-writes in total.

**Step 1: the chance that one write misses a given bit.** One write picks one of *m* positions, so it misses a particular bit with probability:

```
1 − 1/m
```

**Step 2: the chance that a given bit is still 0 after all *kn* writes.**

```
P(bit = 0) = (1 − 1/m)^(kn)
```

**Step 3: the exponential approximation.** For large *m*, (1 − 1/m)^m ≈ e^(−1), so:

```
P(bit = 0) ≈ e^(−kn/m)
P(bit = 1) ≈ 1 − e^(−kn/m)
```

**Step 4: the false-positive rate.** A query for an item that was never inserted checks *k* positions and gets a false positive when all of them are 1. Treating those *k* checks as independent:

```
p ≈ (1 − e^(−kn/m))^k
```

This is the standard Bloom filter formula. (The independence step is an approximation. Exact analyses give slightly higher values for very small filters, but the difference is negligible at practical sizes.)

### Checking it against the toy example

With *m* = 16, *n* = 3 and *k* = 3:

```
kn/m       = 9/16        = 0.5625
e^(−0.5625)              ≈ 0.5698
1 − 0.5698               ≈ 0.4302
0.4302³                  ≈ 0.0796   →  about 8.0%
```

The formula predicts about 8.0%. The actual array, with 7 bits set, gives 8.4%. The small gap comes from randomness: on average we would expect 16 × 0.4302 ≈ 6.9 bits set, and this run happened to get 7.

### A realistic example with numbers

Suppose you have **n = 1,000,000** items and budget **m = 10,000,000 bits**, which is 1.25 MB or 10 bits per item. Here is the false-positive rate for several values of *k*:

| k  | kn/m | 1 − e^(−kn/m) | p = (…)^k | p as % |
|----|------|---------------|-----------|--------|
| 1  | 0.1  | 0.0952        | 0.0952    | 9.52%  |
| 2  | 0.2  | 0.1813        | 0.0329    | 3.29%  |
| 3  | 0.3  | 0.2592        | 0.0174    | 1.74%  |
| 4  | 0.4  | 0.3297        | 0.0118    | 1.18%  |
| 5  | 0.5  | 0.3935        | 0.00943   | 0.94%  |
| 6  | 0.6  | 0.4512        | 0.00844   | 0.84%  |
| **7** | **0.7** | **0.5034** | **0.00819** | **0.82%** |
| 8  | 0.8  | 0.5507        | 0.00846   | 0.85%  |
| 10 | 1.0  | 0.6321        | 0.01019   | 1.02%  |
| 15 | 1.5  | 0.7769        | 0.02266   | 2.27%  |

The table shows the central tension:

- **Too few hashes** (k = 1, 2) give each item a weak "fingerprint". A non-member needs to match only one or two set bits.
- **Too many hashes** (k = 15) fill the array so quickly that most bits are 1, and almost anything matches.
- The best value is in the middle, at **k = 7**, which gives about 0.82%.

The curve is also flat near its minimum. Any *k* from 5 to 8 stays within about 0.12 percentage points of the best value. So you do not need to get *k* exactly right, which matters because *k* must be a whole number.

---

## 4. Choosing m and k

### The optimal k for a given m and n

Minimizing *p* with respect to *k* (by differentiating the log of the formula) gives:

```
k_opt = (m/n) · ln 2  ≈  0.693 · (m/n)
```

For 10 bits per item, k_opt = 6.93, which rounds to 7. That matches the table.

At this optimum, **exactly half the bits are set**, since e^(−k·n/m) = e^(−ln 2) = 1/2. This is a useful rule of thumb: a well-tuned Bloom filter at full capacity is about 50% ones. A filter that is mostly ones has been overfilled. One that is mostly zeros is wasting memory.

Substituting k_opt back into the formula gives the best achievable rate:

```
p_opt = (1/2)^k_opt  ≈  0.6185^(m/n)
```

### The size m for a target false-positive rate

In practice you usually know *n* (the expected number of items) and *p* (the error rate you can tolerate), and you need *m*. Solving the equation above for *m*:

```
m = −n · ln p / (ln 2)²   ≈  1.44 · n · log₂(1/p)
k = (m/n) · ln 2          ≈  log₂(1/p)
```

**Worked example:** n = 1,000,000 and p = 1%.

```
ln(0.01)  = −4.6052
(ln 2)²   =  0.4805
m = 1,000,000 × 4.6052 / 0.4805 ≈ 9,585,059 bits ≈ 1.14 MiB
k = 9.585 × 0.6931             ≈ 6.64  →  7
```

### A quick reference

The bits needed **per item** depend only on *p*, not on *n* or the size of the items:

| Target p | Bits per item (m/n) | Optimal k |
|----------|---------------------|-----------|
| 10%      | 4.79                | 3         |
| 1%       | 9.59                | 7         |
| 0.1%     | 14.38               | 10        |
| 0.01%    | 19.17               | 13        |
| 0.0001%  | 28.76               | 20        |

Two patterns stand out:

1. **Each 10× reduction in error costs about 4.8 more bits per item.** Accuracy is cheap: going from 1% to 0.01% only doubles the memory.
2. **Item size does not matter.** One hundred million 2 KB URLs at 1% fit in about 120 MB of filter (100M × 9.59 bits). Storing the URLs themselves would take around 200 GB.

### Practical guidance

- **Size for peak load, not today's load.** The rate climbs quickly past capacity. A filter sized for 1M items at 1% has a rate of about 16% at 2M items and about 83% at 5M (with *k* fixed at 7).
- **Get k hashes from two.** You do not need *k* separate hash functions. Kirsch and Mitzenmacher (2006) showed that `gᵢ(x) = h₁(x) + i · h₂(x) mod m` for i = 0 … k−1 gives the same asymptotic false-positive rate. One 128-bit hash split into two 64-bit halves is enough.
- **Use a fast non-cryptographic hash** (xxHash, MurmurHash3, wyhash) unless an attacker can choose the inputs. If they can, use a keyed hash such as SipHash with a secret seed. Otherwise an attacker can craft items that all hit the same bits and push your false-positive rate up on purpose.

---

## 5. What You Cannot Do, and What You Get for Free

A standard Bloom filter **cannot**:

- **Delete items.** Clearing *x*'s bits may also clear bits that other items depend on, which would create false negatives. In the toy example, deleting `dog` by clearing bit 6 would make `cat` (bits 1, 6, 11) look absent.
- **List its contents.** The items are not stored.
- **Give exact counts.**

It can do several useful things cheaply:

- **Union.** Two filters with the same *m*, *k* and hash functions can be combined with a bitwise OR. The result is exactly the filter you would get by inserting both sets.
- **Approximate intersection.** A bitwise AND gives a filter that contains the intersection, though with a somewhat higher false-positive rate than one built directly.
- **Cardinality estimation.** If *X* bits are set, the number of distinct items inserted is about:

  ```
  n* ≈ −(m/k) · ln(1 − X/m)
  ```

  For the toy filter, with X = 7, m = 16 and k = 3: n* = −(16/3) · ln(9/16) ≈ 5.333 × 0.5754 ≈ **3.07**. We inserted 3.

---

## 6. Counting Bloom Filters

The **counting Bloom filter** (Fan, Cao, Almeida and Broder, 2000) adds deletion. Each bit becomes a small counter, usually 4 bits:

- **Insert:** increment the *k* counters.
- **Delete:** decrement the *k* counters.
- **Query:** check that all *k* counters are greater than zero.

### Worked example

Use the same items and hash values as before, but with counters:

```
index:   0  1  2  3  4  5  6  7  8  9 10 11 12 13 14 15
after cat, dog, fox:
counts:  0  1  0  0  1  0  2  0  0  1  0  2  0  1  0  1
```

Counter 6 is 2 (from `cat` and `dog`) and counter 11 is 2 (from `cat` and `fox`).

**Delete `dog`:** decrement counters 4, 6 and 13.

```
counts:  0  1  0  0  0  0  1  0  0  1  0  2  0  0  0  1
```

- Query `dog` (4, 6, 13): counter 4 is 0, so the answer is **definitely not**. The deletion worked.
- Query `cat` (1, 6, 11): the counters are 1, 1 and 2, so the answer is **probably yes**. `cat` survived because counter 6 still records its contribution.

### Costs and caveats

- **Memory:** 4-bit counters use 4× the space of a plain filter. Fan et al. showed that with optimal *k*, the probability of any counter reaching 16 is on the order of 1.37 × 10⁻¹⁵ × m, which is negligible. Implementations still saturate at 15 rather than wrap around, and never decrement a saturated counter.
- **Deleting something never inserted corrupts the filter.** It decrements counters that belong to other items and can create false negatives. Delete only items you know were inserted, for example ones confirmed against the real data store.
- **Alternatives:** if you need deletion, a **cuckoo filter** (Fan et al., 2014) usually supports it with less memory than a counting Bloom filter and has better lookup locality, especially at false-positive rates below about 3%.

---

## 7. Scalable Bloom Filters

A plain Bloom filter needs *n* in advance. When you cannot predict *n*, as with a crawler's "seen URLs" set or a stream of event IDs, you can use a **scalable Bloom filter** (Almeida, Baquero, Preguiça and Hutchison, 2007).

### How it works

A scalable filter is a list of ordinary Bloom filters:

1. Start with one filter of capacity *n₀* and error rate *p₀*.
2. When it reaches capacity (about half its bits set), **freeze it** and add a new filter that is *s* times larger, with error rate tightened by a ratio *r*.
3. **Insert** into the newest filter only.
4. **Query** all filters. A hit in any of them is a "probably yes".

Because a query can be fooled by any of the filters, the error rates add up. The tightening ratio keeps that sum bounded:

```
P_total ≤ p₀ + p₀·r + p₀·r² + …  =  p₀ / (1 − r)
```

### Worked example

Target an overall bound of **1%**. Choose r = 0.5, so p₀ = 0.5%, and a growth factor s = 2 with n₀ = 1,000,000.

| Slice | Capacity    | Error rate pᵢ | Bits per item | Slice size |
|-------|-------------|---------------|---------------|------------|
| 0     | 1,000,000   | 0.500%        | 11.03         | ~1.38 MB   |
| 1     | 2,000,000   | 0.250%        | 12.47         | ~3.12 MB   |
| 2     | 4,000,000   | 0.125%        | 13.91         | ~6.96 MB   |
| 3     | 8,000,000   | 0.0625%       | 15.36         | ~15.4 MB   |

After 15 million inserts, the summed error is 0.5 + 0.25 + 0.125 + 0.0625 = 0.9375%, still under the 1% bound. Memory grows roughly in line with the number of items, plus a slowly rising cost per item.

**Trade-offs:** a query may touch several filters, so lookups slow down by a factor equal to the number of slices, which grows logarithmically. The later slices also cost more bits per item than a single filter sized correctly from the start. If you can estimate *n* even roughly, a single well-sized filter is better. Use the scalable variant when you cannot.

---

## 8. Other Variants Worth Knowing

- **Blocked Bloom filters** (Putze, Sanders and Singler, 2007) keep all *k* bits for an item inside one 64-byte cache line, so a lookup costs one memory access instead of *k*. The error rate is slightly higher for the same memory. RocksDB's newer filter format uses this layout.
- **Quotient and cuckoo filters** store short fingerprints instead of bits. They support deletion and use less memory than Bloom filters at low error rates.
- **Xor and ribbon filters** are static: built once from a known set, with no later inserts. They need about 1.23 and 1.0–1.1 times log₂(1/p) bits per item respectively, against 1.44 for a Bloom filter, which suits immutable data such as SSTables.

---

## 9. Real-World Uses

The pattern is always the same: **use a cheap filter to skip expensive work for items that are definitely not there.**

- **LSM-tree databases (LevelDB, RocksDB, Cassandra, HBase).** Data lives in many immutable sorted files on disk. A read for a key that is not in a file would otherwise cost a disk seek. Each file keeps a Bloom filter in memory, and the read checks it first. LevelDB's commonly recommended setting of 10 bits per key gives about 1% false positives, so 99% of unnecessary disk reads are avoided. Cassandra exposes the rate per table as `bloom_filter_fp_chance`.
- **Content delivery networks.** Akamai found that about three-quarters of the objects requested from its caches were requested only once ("one-hit wonders"). Caching them wasted disk space. Akamai used a Bloom filter to record first requests and stored an object only when it was requested a second time. This greatly reduced disk writes and improved hit rates.
- **Web browsers.** Early versions of Chrome's Safe Browsing kept a Bloom filter of hashed malicious-URL prefixes locally. A URL that did not match was allowed immediately, and only matches were checked with Google's servers. (Chrome later moved to a different compressed structure, but the design was the same: a local "definitely safe" check.)
- **Columnar file formats and query engines.** Apache Parquet and ORC can store Bloom filters per column chunk. Engines such as Spark, Trino and Impala use them to skip whole row groups when a query filters on a value that cannot be present. Distributed joins use Bloom filters the same way: one side sends a compact filter of its keys, and the other side drops rows that cannot match before sending them over the network.
- **Bitcoin lightweight clients (BIP 37).** Thin wallets sent full nodes a Bloom filter of their addresses so the nodes could forward only relevant transactions. Researchers later showed that the filters revealed which addresses a wallet owned, so the approach was replaced by compact block filters (BIP 157/158). This is a reminder that a Bloom filter can leak information about the set it represents.
- **Databases, caches and genomics.** PostgreSQL has a `bloom` index extension, Redis offers Bloom and cuckoo filters through RedisBloom, and genome assemblers such as ABySS use Bloom filters to track billions of seen k-mers (DNA substrings) that would not fit in memory exactly.

The design question in every case is: **what does a false positive cost?** In a database, it costs one wasted disk read, so a 1% rate is fine. In a security setting, the "probably yes" must always be followed by an exact check. A Bloom filter alone should never make a decision whose error is expensive.

---

## 10. Implementation

Below is a complete Bloom filter in Python. It sizes itself from *n* and *p*, derives *k* positions from one BLAKE2b digest using double hashing, and can estimate how many items it holds.

```python
import hashlib
import math


class BloomFilter:
    """A standard Bloom filter sized from expected capacity and target error rate."""

    def __init__(self, capacity: int, fp_rate: float):
        if capacity <= 0 or not 0 < fp_rate < 1:
            raise ValueError("capacity must be > 0 and 0 < fp_rate < 1")
        ln2 = math.log(2)
        self.m = math.ceil(-capacity * math.log(fp_rate) / (ln2 ** 2))
        self.k = max(1, round(self.m / capacity * ln2))
        self.bits = bytearray((self.m + 7) // 8)

    def _positions(self, item: str):
        # One 128-bit digest, split into two 64-bit hashes (Kirsch-Mitzenmacher).
        digest = hashlib.blake2b(item.encode("utf-8"), digest_size=16).digest()
        h1 = int.from_bytes(digest[:8], "little")
        h2 = int.from_bytes(digest[8:], "little") | 1  # never zero
        for i in range(self.k):
            yield (h1 + i * h2) % self.m

    def add(self, item: str) -> None:
        for pos in self._positions(item):
            self.bits[pos >> 3] |= 1 << (pos & 7)

    def __contains__(self, item: str) -> bool:
        return all(self.bits[pos >> 3] & (1 << (pos & 7))
                   for pos in self._positions(item))

    def estimated_count(self) -> float:
        x = sum(byte.bit_count() for byte in self.bits)  # Python 3.10+
        if x >= self.m:
            return float("inf")
        return -(self.m / self.k) * math.log(1 - x / self.m)
```

Using it, and measuring the real false-positive rate:

```python
bf = BloomFilter(capacity=1_000_000, fp_rate=0.01)
print(bf.m, bf.k)                      # 9585059 7  (about 1.14 MiB)

for i in range(1_000_000):
    bf.add(f"user:{i}")

assert all(f"user:{i}" in bf for i in range(1_000_000))  # no false negatives

trials = 200_000
false_hits = sum(f"other:{i}" in bf for i in range(trials))
print(f"measured FP rate: {false_hits / trials:.4%}")    # close to 1%
print(f"estimated count:  {bf.estimated_count():,.0f}")  # close to 1,000,000
```

### A counting variant

Replacing the bit array with saturating counters is a small change. For clarity this version uses one byte per counter. A production version would pack two 4-bit counters per byte.

```python
class CountingBloomFilter(BloomFilter):
    MAX = 15  # behave like 4-bit counters

    def __init__(self, capacity: int, fp_rate: float):
        super().__init__(capacity, fp_rate)
        self.counters = bytearray(self.m)

    def add(self, item: str) -> None:
        for pos in self._positions(item):
            if self.counters[pos] < self.MAX:
                self.counters[pos] += 1

    def remove(self, item: str) -> None:
        """Only call this for items you know were inserted."""
        positions = list(self._positions(item))
        if not all(self.counters[p] for p in positions):
            raise KeyError(f"{item!r} is definitely not in the filter")
        for pos in positions:
            if self.counters[pos] < self.MAX:  # saturated counters stay put
                self.counters[pos] -= 1

    def __contains__(self, item: str) -> bool:
        return all(self.counters[p] for p in self._positions(item))
```

The `remove` method refuses items the filter can prove are absent, which catches some mistaken deletes. It cannot catch a delete of a false positive. That protection has to come from the caller.

---

## 11. Summary

| Question | Answer |
|----------|--------|
| What does it store? | A bit array of size *m*; never the items. |
| Errors? | False positives only; never false negatives. |
| False-positive rate | p ≈ (1 − e^(−kn/m))^k |
| Best number of hashes | k = (m/n) · ln 2; about half the bits end up set. |
| Size for target p | m = −n · ln p / (ln 2)², about 9.6 bits per item at 1% |
| Deletion | Not supported; use a counting Bloom filter or a cuckoo filter. |
| Unknown n | Use a scalable Bloom filter; its error is bounded by p₀ / (1 − r). |
| Best use | In front of an expensive exact lookup, where a false positive costs only a little wasted work. |

The insight from 1970 still holds: if you can tolerate a small, measurable rate of "maybe", you can answer membership questions over enormous sets with a few bits per item and a handful of hash computations. Sizing comes down to two inputs, how many items and how much error you can accept, and the formulas above turn those into *m* and *k*.
