# Hash Tables, From First Principles

A hash table stores key–value pairs and finds any of them in constant time on average. Given a key such as `"cat"` or `42`, it jumps almost directly to where that key's value lives, without scanning a list or walking a tree. Python's `dict`, Java's `HashMap`, Go's `map`, caches, compiler symbol tables and database hash joins are all built on this idea.

This explainer works through every moving part with concrete numbers: how keys become array indexes, how collisions are handled, why the table grows, and what it all costs. It ends with working code.

---

## 1. The core idea: turn a key into an array index

Arrays are fast because, given an index, the machine computes an element's location in one step. Real keys, though, are strings, tuples and objects, not small integers. A hash table bridges that gap in two stages:

1. **Hash:** a *hash function* converts the key into an integer, the *hash code*.
2. **Compress:** that integer is reduced to a valid index in an array of size `m`, usually with `hash % m` or, when `m` is a power of two, `hash & (m - 1)`.

Each position in the array is called a **bucket** or **slot**. To insert, look up or delete a key, you compute its bucket and work there.

```
key ──hash()──▶ big integer ──mod m──▶ bucket index ──▶ array[index]
```

There are almost always more possible keys than buckets, so two different keys will sometimes map to the same bucket. That is a **collision**, and handling collisions is most of what hash table design is about.

---

## 2. Hash functions

### 2.1 What a good hash function needs

A hash function for a table must be:

- **Deterministic:** the same key always produces the same hash. Without this you could never find anything again.
- **Consistent with equality:** if `a == b`, then `hash(a) == hash(b)`. The reverse is not required, because unequal keys may share a hash. That is just a collision.
- **Fast:** it runs on every operation.
- **Well-distributed:** small differences in the input should produce large, unpredictable differences in the output.

They do *not* need to be cryptographic like SHA-256, which would be far too slow, though they should resist adversarial inputs (section 8).

### 2.2 Worked example: hashing strings

Consider two candidate hash functions for short lowercase words and a table of size `m = 11`.

**Candidate A: sum of character codes.**

| Key   | Character codes  | Sum | Sum mod 11 |
|-------|------------------|-----|------------|
| `cat` | 99 + 97 + 116    | 312 | **4**      |
| `act` | 97 + 99 + 116    | 312 | **4**      |
| `tac` | 116 + 97 + 99    | 312 | **4**      |

Every anagram collides because addition ignores order. Sums also stay small: three-letter lowercase words sum to between 291 and 366, so a table with 10,000 buckets would use fewer than 80 of them.

**Candidate B: polynomial rolling hash.** Treat the string as digits in base 31:

```
h(s) = s[0]·31^(n-1) + s[1]·31^(n-2) + … + s[n-1]
```

This is computed efficiently with Horner's rule: start at 0, and for each character do `h = h·31 + code`.

For `"cat"` (c = 99, a = 97, t = 116):

```
h = 0
h = 0·31  + 99  = 99
h = 99·31 + 97  = 3069 + 97   = 3166
h = 3166·31 + 116 = 98146 + 116 = 98262
98262 mod 11 = 98262 − 11·8932 = 98262 − 98252 = 10   → bucket 10
```

For `"act"` (a = 97, c = 99, t = 116):

```
h = 97
h = 97·31 + 99 = 3007 + 99 = 3106
h = 3106·31 + 116 = 96286 + 116 = 96402
96402 mod 11 = 96402 − 96393 = 9   → bucket 9
```

For `"dog"` (d = 100, o = 111, g = 103):

```
h = 100
h = 100·31 + 111 = 3211
h = 3211·31 + 103 = 99541 + 103 = 99644
99644 mod 11 = 99644 − 99638 = 6   → bucket 6
```

Position now matters, so the anagrams separate. Java's `String.hashCode()` uses exactly this scheme with base 31, wrapped to 32-bit integers.

### 2.3 Hashing integers and the choice of `m`

For integer keys, the simplest hash is the key itself, compressed with `k mod m`. The quality of that compression depends on `m`.

Suppose all keys are multiples of 4 (0, 4, 8, 12, …) and `m = 8`. Then `k mod 8` is always 0 or 4, and six of the eight buckets are never used. With a prime `m = 7` the same keys give 0, 4, 1, 5, 2, 6, 3, … and every bucket is reached. A prime shares no factors with regular strides in the data.

Production tables often use powers of two instead, because `hash & (m − 1)` is cheaper than division, and compensate by **mixing** the hash first so its low bits depend on all of the input. Java's `HashMap`, for example, XORs the high 16 bits into the low 16.

---

## 3. Collision strategy 1: separate chaining

In **separate chaining**, each bucket holds a small collection, classically a linked list and in practice often a dynamic array, containing every entry that hashed there.

- **Insert:** hash to bucket `i`, scan its chain. If the key is present, update its value. Otherwise append it.
- **Lookup:** hash to bucket `i`, scan its chain for the key.
- **Delete:** hash to bucket `i`, remove the key from its chain.

### 3.1 Worked trace

Table size `m = 7`, hash `h(k) = k mod 7`. Insert the keys 10, 22, 31, 4, 15, 28 and 17 in that order.

| Step | Key | k mod 7 | Action                          |
|------|-----|---------|---------------------------------|
| 1    | 10  | 3       | bucket 3 empty, start chain     |
| 2    | 22  | 1       | bucket 1 empty, start chain     |
| 3    | 31  | 3       | **collision**, append to 3      |
| 4    | 4   | 4       | bucket 4 empty, start chain     |
| 5    | 15  | 1       | **collision**, append to 1      |
| 6    | 28  | 0       | bucket 0 empty, start chain     |
| 7    | 17  | 3       | **collision**, append to 3      |

Final state:

```
[0] → 28
[1] → 22 → 15
[2] → ∅
[3] → 10 → 31 → 17
[4] → 4
[5] → ∅
[6] → ∅
```

**Lookup 17:** `17 mod 7 = 3`. Scan bucket 3 and compare against 10, then 31, then 17. Found after 3 comparisons.

**Lookup 24:** `24 mod 7 = 3`. Compare against 10, 31 and 17, then reach the end of the chain. Not found after 3 comparisons.

**Lookup 5:** `5 mod 7 = 5`. The bucket is empty. Not found after 0 comparisons.

**Delete 31:** unlink it from bucket 3, leaving `10 → 17`. No other bucket is affected.

Seven keys in seven buckets is a **load factor** of 1.0, yet three buckets are empty and one holds three keys. Even a perfectly random hash produces clumps like this. Section 6 shows why the expected chain length still stays small.

### 3.2 Strengths and weaknesses

Chaining is easy to get right, handles deletion cleanly and degrades gracefully. Its weakness is memory and cache behavior: each node is a separate heap allocation, and following pointers jumps around memory. Java's `HashMap` adds one defense: a chain longer than 8 entries becomes a red-black tree, so a pathological bucket costs O(log n) instead of O(n).

---

## 4. Collision strategy 2: open addressing

In **open addressing**, all entries live directly in the array, one per slot. There are no chains. When a key's home slot is taken, the table follows a deterministic **probe sequence** to other slots until it finds the key or an empty slot.

The probe sequence for key `k` is `h(k, 0), h(k, 1), h(k, 2), …`. Three classic schemes:

| Scheme            | Probe `i` lands at                     |
|-------------------|----------------------------------------|
| Linear probing    | `(h(k) + i) mod m`                     |
| Quadratic probing | `(h(k) + c₁·i + c₂·i²) mod m`          |
| Double hashing    | `(h₁(k) + i·h₂(k)) mod m`              |

Because every entry occupies a slot, the load factor must stay strictly below 1, and in practice well below it.

### 4.1 Linear probing, traced

Table size `m = 11`, `h(k) = k mod 11`, same keys as before: 10, 22, 31, 4, 15, 28, 17.

| Key | Home (k mod 11) | Probes                       | Placed at |
|-----|-----------------|------------------------------|-----------|
| 10  | 10              | 10 empty                     | 10        |
| 22  | 0               | 0 empty                      | 0         |
| 31  | 9               | 9 empty                      | 9         |
| 4   | 4               | 4 empty                      | 4         |
| 15  | 4               | 4 taken → 5 empty            | 5         |
| 28  | 6               | 6 empty                      | 6         |
| 17  | 6               | 6 taken → 7 empty            | 7         |

Resulting array:

```
index:  0   1   2   3   4   5   6   7   8   9   10
value: 22   ·   ·   ·   4  15  28  17   ·  31  10
```

Load factor `α = 7/11 ≈ 0.64`.

**Lookup 17:** home 6 holds 28, which doesn't match. Slot 7 holds 17. Found in 2 probes.

**Lookup 26 (absent):** `26 mod 11 = 4`. Slot 4 holds 4, slot 5 holds 15, slot 6 holds 28, slot 7 holds 17, and slot 8 is empty. Not found after **5 probes**, even though the table is only about two-thirds full.

That shows **primary clustering**, linear probing's main weakness. Slots 4 to 7 have merged into one run. Any key hashing into it must walk to its end, and landing there makes the run longer still. Clusters feed themselves.

**Wraparound:** insert 21. `21 mod 11 = 10` is taken, so the probe wraps to slot 0 (taken by 22), then slot 1, which is empty. 21 goes in slot 1.

### 4.2 Deletion and tombstones

Deletion is where open addressing gets subtle. Go back to the table before 21 was inserted and delete 28 from slot 6 by simply emptying it:

```
index:  0   1   2   3   4   5   6   7   8   9   10
value: 22   ·   ·   ·   4  15   ·  17   ·  31  10
```

Now look up 17. Its home is 6, which is empty, so the search stops and reports *not found*. But 17 is sitting in slot 7. Emptying slot 6 broke the probe chain that led to it.

The standard fix is a **tombstone**, a special marker meaning "deleted, keep probing":

```
index:  0   1   2   3   4   5   6   7   8   9   10
value: 22   ·   ·   ·   4  15  ✝   17   ·  31  10
```

- **Lookups** treat a tombstone as occupied and continue past it. Looking up 17 now probes 6 (tombstone, continue) and then 7 (found).
- **Inserts** may reuse a tombstone slot. They should keep scanning to confirm the key isn't already present further along, then place it in the first tombstone they saw.

Tombstones lengthen probes just as live entries do, so implementations count them toward the resize threshold and clear them out when rebuilding. Linear probing can instead use **backward-shift deletion**, moving later entries in the cluster back to fill the gap.

### 4.3 Quadratic probing and double hashing

**Quadratic probing** visits offsets +0, +1, +4, +9, …, so keys with neighboring home slots no longer share a path. Keys with the *same* home still do (*secondary clustering*, which is much milder). It may not visit every slot, but with a prime `m` and `α ≤ 0.5` it always finds an empty one.

**Double hashing** uses a second hash to choose the step size, so keys with the same home walk different strides. The step must be nonzero and coprime with `m`, which is automatic when `m` is prime.

Worked example: rebuild the table above, but insert 17 using double hashing with `h₁(k) = k mod 11` and `h₂(k) = 7 − (k mod 7)`. Before 17 arrives, slots 0, 4, 5, 6, 9 and 10 are full.

```
h₁(17) = 17 mod 11 = 6
h₂(17) = 7 − (17 mod 7) = 7 − 3 = 4

i = 0:  6               → taken (28)
i = 1: (6 + 4)  mod 11 = 10 → taken (10)
i = 2: (6 + 8)  mod 11 = 3  → empty, place 17 at slot 3
```

Key 17 jumped past the cluster at 4 to 6 instead of extending it.

### 4.4 Why open addressing is popular anyway

The fastest modern tables (Google's SwissTable, Rust's `HashMap`, Python's `dict`) use open addressing. Entries sit contiguously, so probing scans cache lines already loaded, with no pointer-chasing or per-entry allocation. SwissTable also compares 16 one-byte hash tags at once with SIMD, rejecting most non-matching slots without touching the keys.

---

## 5. Load factor

The **load factor** is

```
α = n / m      (n = number of entries, m = number of buckets)
```

It is the most important number for performance, because nearly every cost grows with it.

| Strategy           | Typical max α | Notes                                              |
|--------------------|---------------|----------------------------------------------------|
| Separate chaining  | 0.75 to 1.0+  | Java `HashMap` default is 0.75                     |
| Linear probing     | 0.5 to 0.7    | Performance falls apart near 1                     |
| Python `dict`      | ~0.67 (2/3)   | Open addressing with perturbed probing             |
| SwissTable         | 0.875 (7/8)   | SIMD group probing tolerates high load             |

Lower α means faster operations and more wasted memory. The resize threshold picks the point on that trade-off.

---

## 6. How the costs depend on α

Assume a hash function that behaves like a random assignment of keys to buckets, the *simple uniform hashing* assumption. Then the expected costs, counted in key comparisons or probes, are as follows.

**Separate chaining**

- Unsuccessful search: `1 + α`, meaning the hash computation plus a full scan of an average-length chain.
- Successful search: `1 + α/2`, meaning on average you find the key halfway along its chain.

Both are O(1) as long as α is bounded by a constant.

**Linear probing** (Knuth's classic analysis)

- Successful search: about `½ · (1 + 1/(1 − α))`
- Unsuccessful search or insert: about `½ · (1 + 1/(1 − α)²)`

Plugging in numbers shows why open addressing tables resize early:

| α    | Chaining miss | Linear probe hit | Linear probe miss |
|------|---------------|------------------|-------------------|
| 0.25 | 1.25          | 1.17             | 1.39              |
| 0.50 | 1.50          | 1.50             | 2.50              |
| 0.75 | 1.75          | 2.50             | 8.50              |
| 0.90 | 1.90          | 5.50             | 50.50             |
| 0.99 | 1.99          | 50.50            | 5000.50           |

Chaining degrades linearly, while linear probing's miss cost grows with the *square* of `1/(1 − α)`: at 90% full, a failed lookup averages fifty probes. At moderate load, though, linear probing often wins in wall-clock time, because its probes hit adjacent memory.

---

## 7. Resizing and rehashing

A table that never grows gets slower as α rises. The fix is to **resize**: when α would exceed the threshold, allocate a bigger array, typically double the size, and **rehash** every entry into it. You cannot just copy the old array, because each bucket is `hash mod m` and `m` has changed.

### 7.1 Worked example

Separate chaining, initial `m = 4`, maximum load factor 0.75, hash `h(k) = k mod m`. Policy: before inserting, if `(n + 1) / m > 0.75`, double `m` and rehash.

Insert 5, 12, 9:

| Key | n before | (n+1)/m | k mod 4 | Result               |
|-----|----------|---------|---------|----------------------|
| 5   | 0        | 0.25    | 1       | bucket 1: [5]        |
| 12  | 1        | 0.50    | 0       | bucket 0: [12]       |
| 9   | 2        | 0.75    | 1       | bucket 1: [5, 9]     |

```
m = 4, n = 3, α = 0.75
[0] → 12
[1] → 5 → 9
[2] → ∅
[3] → ∅
```

Now insert 6. `(3 + 1) / 4 = 1.0 > 0.75`, so the table resizes first. Allocate `m = 8` and reinsert every existing key with the new modulus:

| Key | Old bucket (mod 4) | New bucket (mod 8) |
|-----|--------------------|--------------------|
| 12  | 0                  | 4                  |
| 5   | 1                  | 5                  |
| 9   | 1                  | 1                  |

Then insert 6: `6 mod 8 = 6`.

```
m = 8, n = 4, α = 0.5
[0] → ∅
[1] → 9
[2] → ∅
[3] → ∅
[4] → 12
[5] → 5
[6] → 6
[7] → ∅
```

The collision between 5 and 9 disappeared. With power-of-two sizes, each key either stays at index `i` or moves to `i + 4`, depending on one extra hash bit: 9 is `1001` (bit 2 is 0, stays at 1) and 5 is `0101` (bit 2 is 1, moves to 5).

### 7.2 Why doubling makes resizing cheap on average

A single resize costs O(n), but resizes become rarer as the table grows.

Suppose you insert `n` items into a table that starts at size 1 and doubles whenever it fills. Resizes happen at sizes 1, 2, 4, 8, …, up to about `n`. The total number of entries moved across all resizes is

```
1 + 2 + 4 + … + n/2 + n  <  2n
```

So `n` inserts cost at most `n + 2n`, which is **O(1) amortized per insert**. This depends on growing *geometrically*. Growing by a fixed 100 slots would mean `n/100` resizes of up to `n` each: O(n²) total.

Any single insert can still pay the full O(n). Latency-sensitive systems such as Redis resize **incrementally**, moving a few buckets per operation.

If a table also shrinks, its shrink threshold (say 0.25) must sit well below the grow threshold (0.75). Otherwise alternating inserts and deletes at the boundary would rehash on every operation.

---

## 8. Worst cases and hash flooding

Everything above is an *average*. If every key hashes to the same bucket, each operation becomes O(n) and inserting n keys costs O(n²). With a fixed, public hash function, an attacker can force this. In 2011, researchers tied up web servers for minutes with single POST requests whose parameter names all collided. This is **hash flooding**.

The defenses are **keyed hashing** (a random per-process seed mixed into a function such as SipHash, as Python and Rust do), **treeified buckets** (Java's O(log n) cap), and in theory **universal hashing**, which picks the function at random from a family that guarantees expected O(1) for any input.

---

## 9. Complexity summary

| Operation           | Average     | Worst case | Notes                                  |
|---------------------|-------------|------------|----------------------------------------|
| Insert              | O(1)*       | O(n)       | *amortized, including resizes          |
| Lookup              | O(1)        | O(n)       | O(log n) worst with treeified buckets  |
| Delete              | O(1)        | O(n)       | needs tombstones or backward shift with open addressing |
| Resize              | O(n)        | O(n)       | rare; amortizes to O(1) per insert     |
| Iterate all entries | O(n + m)    | O(n + m)   | must visit empty buckets too           |
| Space               | O(n)        | O(n)       | as long as m stays proportional to n   |

Hash tables do **not** offer ordering: minimums, sorted iteration and range queries need a full scan. Use a balanced tree for those.

---

## 10. Code

### 10.1 Separate chaining with resizing

```python
class ChainedHashMap:
    """Hash map using separate chaining with automatic doubling."""

    def __init__(self, capacity=8, max_load=0.75):
        self._buckets = [[] for _ in range(capacity)]
        self._size = 0
        self._max_load = max_load

    def _index(self, key, capacity=None):
        return hash(key) % (capacity or len(self._buckets))

    def put(self, key, value):
        bucket = self._buckets[self._index(key)]
        for i, (k, _) in enumerate(bucket):
            if k == key:                     # existing key: update in place
                bucket[i] = (key, value)
                return
        if (self._size + 1) / len(self._buckets) > self._max_load:
            self._resize(2 * len(self._buckets))
            bucket = self._buckets[self._index(key)]
        bucket.append((key, value))
        self._size += 1

    def get(self, key, default=None):
        for k, v in self._buckets[self._index(key)]:
            if k == key:
                return v
        return default

    def remove(self, key):
        bucket = self._buckets[self._index(key)]
        for i, (k, _) in enumerate(bucket):
            if k == key:
                bucket.pop(i)
                self._size -= 1
                return True
        return False

    def _resize(self, new_capacity):
        new_buckets = [[] for _ in range(new_capacity)]
        for bucket in self._buckets:
            for k, v in bucket:              # rehash every entry
                new_buckets[self._index(k, new_capacity)].append((k, v))
        self._buckets = new_buckets

    def __len__(self):
        return self._size
```

The equality check inside the chain is how colliding keys are told apart, and `_resize` recomputes every index with the *new* capacity.

### 10.2 Linear probing with tombstones

```python
_EMPTY = object()
_TOMBSTONE = object()


class LinearProbingHashMap:
    """Open-addressing hash map with linear probing and tombstone deletion."""

    def __init__(self, capacity=8, max_load=0.5):
        self._keys = [_EMPTY] * capacity
        self._vals = [None] * capacity
        self._size = 0          # live entries
        self._used = 0          # live entries + tombstones
        self._max_load = max_load

    def _probe(self, key):
        """Yield slot indexes in probe order, starting at the key's home."""
        m = len(self._keys)
        start = hash(key) % m
        for i in range(m):
            yield (start + i) % m

    def get(self, key, default=None):
        for idx in self._probe(key):
            k = self._keys[idx]
            if k is _EMPTY:
                return default               # a true gap ends the search
            if k is not _TOMBSTONE and k == key:
                return self._vals[idx]
        return default

    def put(self, key, value):
        if (self._used + 1) / len(self._keys) > self._max_load:
            self._resize(2 * len(self._keys))
        first_tombstone = None
        for idx in self._probe(key):
            k = self._keys[idx]
            if k is _EMPTY:
                if first_tombstone is not None:
                    idx = first_tombstone    # reuse a deleted slot
                else:
                    self._used += 1
                self._keys[idx] = key
                self._vals[idx] = value
                self._size += 1
                return
            if k is _TOMBSTONE:
                if first_tombstone is None:
                    first_tombstone = idx
            elif k == key:
                self._vals[idx] = value      # update existing
                return
        # No empty slot found; fall back to a tombstone if we saw one.
        if first_tombstone is None:
            raise RuntimeError("table full")  # unreachable while max_load < 1
        self._keys[first_tombstone] = key
        self._vals[first_tombstone] = value
        self._size += 1

    def remove(self, key):
        for idx in self._probe(key):
            k = self._keys[idx]
            if k is _EMPTY:
                return False
            if k is not _TOMBSTONE and k == key:
                self._keys[idx] = _TOMBSTONE
                self._vals[idx] = None
                self._size -= 1              # _used stays: tombstone still occupies
                return True
        return False

    def _resize(self, new_capacity):
        old = [(k, v) for k, v in zip(self._keys, self._vals)
               if k is not _EMPTY and k is not _TOMBSTONE]
        # If most occupancy is tombstones, rebuilding at the same size suffices.
        if self._size < len(self._keys) * self._max_load / 2:
            new_capacity = len(self._keys)
        self._keys = [_EMPTY] * new_capacity
        self._vals = [None] * new_capacity
        self._size = self._used = 0
        for k, v in old:                     # rehash; tombstones are dropped
            self.put(k, v)

    def __len__(self):
        return self._size
```

`get` stops only at a truly **empty** slot, never a tombstone, which keeps the section 4.2 trace correct. The resize trigger counts tombstones (`_used`), and rebuilding drops them.

---

## 11. Recap

A hash function turns a key into an array index, and a collision strategy handles the rest. **Chaining** is simple and degrades gently. **Open addressing** is faster thanks to cache locality but needs careful deletion and a lower load factor. Keeping **α = n/m** bounded by doubling and rehashing gives expected O(1) operations at O(1) amortized resize cost. One practical rule follows from all of this: never mutate a key while it is in a table, because its entry stays in the old bucket and can't be found again.
