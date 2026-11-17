# Consistent Hashing, Explained

Almost every distributed storage or caching system eventually has to answer one question: **given a key, which machine holds it?** The obvious answer, "hash the key and take it modulo the number of machines," works perfectly right up until the number of machines changes. Then it falls apart. Consistent hashing is the family of techniques that fixes this. When the cluster grows or shrinks, only the keys that *have* to move get moved.

This explainer builds the idea from first principles. It works through a concrete ring by hand, counts how many keys move under each scheme, measures how virtual nodes even out the load, and then covers two alternatives, rendezvous hashing and jump consistent hash, with working code for each.

---

## 1. The problem with `hash(key) % N`

Suppose you run a cache tier with `N` servers. To place a key, you compute:

```python
server = hash(key) % N
```

This is fast, stateless, and spreads keys evenly when the hash is good. Every client computes the same answer without coordinating with anyone.

Now one server dies, or you add one to handle more traffic. `N` changes, and with it the result of `% N` for nearly every key. A key whose hash is 41 lives on server `41 % 3 = 2` in a three-server cluster, but on server `41 % 4 = 1` in a four-server cluster. Nothing about the key changed. Only the divisor did.

How many keys move? Going from `N` to `N + 1` servers, a key stays put only if `h % N == h % (N + 1)`. For a uniformly random hash that happens with probability about `1 / (N + 1)`. So **roughly `N / (N + 1)` of all keys move**: 75% going from 3 to 4 servers, about 91% going from 10 to 11, and about 99% going from 100 to 101.

In a cache, every moved key is a miss. Adding capacity to relieve load briefly sends almost the whole working set back to the database at once, which is exactly the moment the database can least afford it. In a storage system, every moved key is data that has to be copied across the network. Either way, the cost of a membership change is proportional to the *entire dataset*, not to the size of the change.

The goal of consistent hashing is to make that cost proportional to the change. When you add one server to a cluster of `N`, ideally only about `1 / (N + 1)` of the keys move, and all of them move *to the new server*. Nothing gets shuffled between servers that were already there.

---

## 2. The hash ring

The classic construction, from Karger et al. (1997), goes like this:

1. Take the output space of a hash function, say all 64-bit integers, and bend it into a circle, so the largest value wraps around to 0.
2. Hash each **node** (by its name or address) to a point on that circle.
3. Hash each **key** to a point on the same circle.
4. A key belongs to the **first node you reach walking clockwise** from the key's position. If you pass the top of the circle, wrap around to the start.

Each node therefore owns the arc of the circle between its predecessor and itself. When a node joins, it lands somewhere on the circle and takes over part of exactly one neighbor's arc. When a node leaves, its arc goes to its clockwise successor. No other node is involved.

### 2.1 A concrete ring

To keep the arithmetic readable, use a tiny ring with positions `0` to `99`. In a real system these would be 64-bit hash values, but the mechanics are identical.

Three nodes hash to these positions:

| Node | Position |
|------|----------|
| A    | 10       |
| B    | 40       |
| C    | 75       |

Ten keys hash to these positions:

| Key | k1 | k2 | k3 | k4 | k5 | k6 | k7 | k8 | k9 | k10 |
|-----|----|----|----|----|----|----|----|----|----|-----|
| Pos | 5  | 18 | 27 | 41 | 52 | 63 | 70 | 80 | 91 | 99  |

Drawn as a line that wraps around (read the right edge as connecting back to the left):

```
0         10                  40                            75             99
|----k1---A----k2----k3-------B--k4------k5------k6----k7---C---k8---k9---k10-|
     5         18    27          41      52      63    70       80   91   99
```

Apply the rule, "walk clockwise to the first node at or after the key":

- **k1 (5)** → the next node is A at 10 → **A**
- **k2 (18), k3 (27)** → the next node is B at 40 → **B**
- **k4 (41), k5 (52), k6 (63), k7 (70)** → the next node is C at 75 → **C**
- **k8 (80), k9 (91), k10 (99)** → no node after 75 before the end, so wrap around to A at 10 → **A**

| Node | Arc owned            | Keys                 | Count |
|------|----------------------|----------------------|-------|
| A    | (75, 99] ∪ [0, 10]   | k1, k8, k9, k10      | 4     |
| B    | (10, 40]             | k2, k3               | 2     |
| C    | (40, 75]             | k4, k5, k6, k7       | 4     |

Note the arc sizes: A owns 35 positions, B owns 30, and C owns 35. Those happen to be fairly even. With random node placement they usually aren't, and section 3 deals with that.

### 2.2 Adding a node

Add node **D**, which hashes to position **60**. It lands inside C's arc `(40, 75]` and splits it in two. D takes `(40, 60]`, and C keeps `(60, 75]`.

Which keys change owner? Only those in `(40, 60]`: **k4 (41)** and **k5 (52)**, both moving from C to D. Everything else stays where it was.

| Key  | Before | After |
|------|--------|-------|
| k1   | A      | A     |
| k2   | B      | B     |
| k3   | B      | B     |
| k4   | C      | **D** |
| k5   | C      | **D** |
| k6   | C      | C     |
| k7   | C      | C     |
| k8   | A      | A     |
| k9   | A      | A     |
| k10  | A      | A     |

**2 of 10 keys moved**, and both moved to the new node. A and B weren't affected at all.

### 2.3 The same change under modulo hashing

For comparison, use each key's position as its hash value and go from `% 3` to `% 4`:

| Key | Hash | `h % 3` | `h % 4` | Moved? |
|-----|------|---------|---------|--------|
| k1  | 5    | 2       | 1       | yes    |
| k2  | 18   | 0       | 2       | yes    |
| k3  | 27   | 0       | 3       | yes    |
| k4  | 41   | 2       | 1       | yes    |
| k5  | 52   | 1       | 0       | yes    |
| k6  | 63   | 0       | 3       | yes    |
| k7  | 70   | 1       | 2       | yes    |
| k8  | 80   | 2       | 0       | yes    |
| k9  | 91   | 1       | 3       | yes    |
| k10 | 99   | 0       | 3       | yes    |

**10 of 10 keys moved.** The expected number for 3 → 4 is 75%. This sample happens to be unlucky, but it makes the point: under modulo hashing, keys get reshuffled between servers that didn't change at all. Server 0 gave away k2 and also received k5 and k8, even though nothing happened to server 0.

### 2.4 Removing a node

Start again from the original three-node ring and remove **B** (position 40). B's arc `(10, 40]` goes to its clockwise successor, C, which now owns `(10, 75]`.

Keys that move: **k2 (18)** and **k3 (27)**, both from B to C. **2 of 10 keys moved**, exactly the keys B was holding. That is the minimum possible, because those keys had to go *somewhere*.

The general rule is that a membership change on a ring moves only the keys in the affected arc. On average that is `K / N` keys for `K` keys and `N` nodes. Modulo hashing moves about `K × N / (N + 1)`.

---

## 3. Load imbalance and virtual nodes

The ring fixes the movement problem, but the basic version has a serious flaw. **Nodes placed at random positions own arcs of very different sizes.**

With `N` random points on a circle, arc lengths follow roughly an exponential distribution. The largest arc is around `ln N` times the average, and the smallest can be close to zero. The example above was evenly spaced by design. A real hash function won't be so kind.

Worse, when a node fails, its *entire* load goes to a single neighbor. That neighbor's load doubles while everyone else stays the same, which is a recipe for cascading failure.

### 3.1 The fix: many points per node

The standard fix is **virtual nodes** (or "vnodes", or "replicas" in some libraries). Instead of hashing each physical node once, hash it `V` times, for example `hash("A#0")`, `hash("A#1")`, …, `hash("A#V-1")`, and place all `V` points on the ring. A key still walks clockwise to the first point it meets, and that point maps back to its physical node.

Each physical node now owns `V` small, scattered arcs instead of one big one. Its total share is a sum of `V` roughly independent arc lengths, so by the law of large numbers it concentrates around `1 / N`. The relative standard deviation shrinks roughly as `1 / √V`.

Virtual nodes also fix the failure problem. When a node leaves, its `V` arcs are spread around the ring, so each one goes to a *different* successor. The departed node's load gets spread over many survivors instead of landing on one.

### 3.2 Measured numbers

The following figures come from a simulation with **10 physical nodes**, MD5-based 64-bit positions, and exact arc lengths. They are averaged over 30 different sets of node names, so they reflect typical behavior rather than one lucky layout. "Load" is each node's fraction of the ring divided by the ideal `1/10`, so 1.0 is perfect.

| Vnodes per node (V) | Std. dev. of load | Max load / mean |
|---------------------|-------------------|-----------------|
| 1                   | 0.88              | 2.93×           |
| 10                  | 0.28              | 1.51×           |
| 100                 | 0.09              | 1.16×           |
| 1,000               | 0.03              | 1.05×           |

Read the first row carefully. With one point per node, **the busiest node typically carries almost three times its fair share**. If you size servers for average load, that node falls over first. At 100 vnodes the hottest node is about 16% over the mean, and at 1,000 it is about 5% over.

The `1/√V` pattern shows up clearly. Each tenfold increase in `V` cuts the standard deviation by about √10 ≈ 3.2 (0.88 → 0.28 → 0.09 → 0.03).

### 3.3 The cost of virtual nodes

Virtual nodes aren't free:

- **Memory.** The ring holds `N × V` entries. With 1,000 nodes × 1,000 vnodes that is a million sorted entries. That's fine for a server, but it adds up if every client keeps its own copy.
- **Lookup time.** Lookup is a binary search, `O(log(N × V))`. That's fast, but slower than an O(1) array index.
- **Rebuild time.** Changing membership means inserting or removing `V` points and re-sorting.

Typical production values are 100–256 vnodes per node. That's enough to get imbalance down to the 5–15% range without the ring getting unwieldy.

### 3.4 Weighted nodes

Virtual nodes also give you **weighting** for free. A machine with twice the RAM gets twice the vnodes and so receives about twice the keys. This is how heterogeneous clusters are usually handled on a ring.

---

## 4. A ring implementation

Here is a complete, minimal implementation in Python, with virtual nodes, weights, and lookups by binary search.

```python
import bisect
import hashlib


def _hash64(s: str) -> int:
    """Stable 64-bit hash. Never use Python's built-in hash(): it is salted per process."""
    return int.from_bytes(hashlib.md5(s.encode()).digest()[:8], "big")


class HashRing:
    def __init__(self, vnodes: int = 160):
        self.vnodes = vnodes
        self._points: list[int] = []     # sorted ring positions
        self._owners: list[str] = []     # owner of each position, parallel to _points
        self._weights: dict[str, int] = {}

    def add(self, node: str, weight: int = 1) -> None:
        if node in self._weights:
            raise ValueError(f"{node} already on ring")
        self._weights[node] = weight
        for i in range(self.vnodes * weight):
            pos = _hash64(f"{node}#{i}")
            idx = bisect.bisect_left(self._points, pos)
            self._points.insert(idx, pos)
            self._owners.insert(idx, node)

    def remove(self, node: str) -> None:
        self._weights.pop(node)
        keep = [(p, o) for p, o in zip(self._points, self._owners) if o != node]
        self._points = [p for p, _ in keep]
        self._owners = [o for _, o in keep]

    def lookup(self, key: str) -> str:
        if not self._points:
            raise LookupError("empty ring")
        pos = _hash64(key)
        idx = bisect.bisect_left(self._points, pos)
        if idx == len(self._points):      # past the last point: wrap to the start
            idx = 0
        return self._owners[idx]

    def lookup_n(self, key: str, n: int) -> list[str]:
        """First n *distinct* physical nodes clockwise: the replica set."""
        if n > len(self._weights):
            raise ValueError("not enough nodes")
        pos = _hash64(key)
        idx = bisect.bisect_left(self._points, pos)
        found: list[str] = []
        while len(found) < n:
            owner = self._owners[idx % len(self._points)]
            if owner not in found:
                found.append(owner)
            idx += 1
        return found
```

A few things worth pointing out:

- **Use a stable hash.** Python's `hash()` on strings is randomized per process, so two clients would build different rings. MD5, xxHash, MurmurHash3, or SipHash with a fixed key are all fine. Cryptographic strength doesn't matter here, but a good spread and stability across processes do.
- **`insert` into a list is O(n).** That's fine for rings that change rarely. If membership changes often, use a sorted container or rebuild the arrays in bulk.
- **`lookup_n` skips vnodes owned by a node already chosen.** This is how Dynamo-style systems pick a key's *preference list* of `n` replicas. Without the skip, two consecutive vnodes from the same machine would put two "replicas" on one box.

Usage:

```python
ring = HashRing(vnodes=160)
for name in ["cache-a", "cache-b", "cache-c"]:
    ring.add(name)

print(ring.lookup("user:42"))          # e.g. 'cache-b'
print(ring.lookup_n("user:42", 2))     # e.g. ['cache-b', 'cache-a']

ring.add("cache-d")                    # roughly 1/4 of keys move, all to cache-d
```

### 4.1 Measured movement at scale

The same simulation, with 100,000 keys and 10 → 11 nodes, gives:

| Scheme                          | Keys moved | Fraction | Ideal (1/11) |
|---------------------------------|-----------:|---------:|-------------:|
| Modulo (`% 10` → `% 11`)        | 90,949     | 90.9%    | 9.1%         |
| Ring, 100 vnodes per node       | 7,831      | 7.8%     | 9.1%         |

The ring moves about a tenth as many keys. It also moves *slightly fewer* than the ideal 9.1%, because the new node's 100 vnodes happened to claim a bit less than its fair share of the ring. That is the leftover imbalance from section 3, which shows up as the newcomer's load.

---

## 5. Rendezvous hashing (highest random weight)

**Rendezvous hashing**, from Thaler and Ravishankar (1996), predates the ring and gets the same guarantees with no ring at all.

For each key, compute a score for *every* node by hashing the key and the node together. The node with the highest score wins:

```python
def rendezvous(key: str, nodes: list[str]) -> str:
    return max(nodes, key=lambda node: _hash64(f"{key}|{node}"))
```

That's the whole algorithm.

### 5.1 Why it is consistent

Think of each key as running a private lottery among the nodes. Each node's ticket for that key is `hash(key, node)`, which is fixed and independent of the other nodes.

- **Adding a node** adds one new ticket to every key's lottery. A key moves only if the new node's ticket beats the current winner. With `N + 1` uniform tickets, the newcomer wins with probability `1 / (N + 1)`, which is exactly the ideal fraction. And moved keys always go *to the new node*, because the existing tickets didn't change.
- **Removing a node** only affects keys where that node was the winner. Those keys fall to their second-highest ticket, which is spread evenly over the survivors. That gives the same "many successors" benefit that virtual nodes provide, with no extra configuration.

In the simulation (20,000 keys, 10 → 11 nodes), rendezvous hashing moved **1,786 keys, or 8.9%**, against an ideal of 9.1%.

### 5.2 Balance with no tuning

Because every key runs an independent lottery, each node's share is a binomial random variable. Its imbalance is the same as perfectly random assignment, with no arc-length variance at all. You don't need virtual nodes to get even load.

### 5.3 Replicas and weights

For `n` replicas, take the top `n` scores:

```python
import heapq

def rendezvous_n(key: str, nodes: list[str], n: int) -> list[str]:
    return heapq.nlargest(n, nodes, key=lambda node: _hash64(f"{key}|{node}"))
```

For weighted nodes, the standard trick (the "logarithmic method") turns the hash into a uniform value `u` in (0, 1) and scores each node as `-weight / ln(u)`. A node with weight `w` then wins in proportion to `w`, and adding or removing a node still moves the minimum number of keys.

```python
import math

def weighted_rendezvous(key: str, weights: dict[str, float]) -> str:
    def score(node: str) -> float:
        u = (_hash64(f"{key}|{node}") + 1) / (2**64 + 1)   # strictly in (0, 1)
        return -weights[node] / math.log(u)
    return max(weights, key=score)
```

### 5.4 The trade-off

Lookup is **O(N)**, because every node has to be hashed for every key. With 10 or 50 nodes that costs nothing and is often faster than a binary search over thousands of vnodes. With 10,000 nodes it starts to hurt. Hierarchical variants (rendezvous over racks, then over nodes within the rack) or "skeleton-based" rendezvous bring it down to O(log N) when needed.

Rendezvous hashing is used in systems such as Apache Ignite, GitHub's load balancer, and many CDN request routers. In practice it is often the better default for small to medium clusters: it is simpler than a ring, needs no state, and is balanced without tuning.

---

## 6. Jump consistent hash

**Jump consistent hash**, from Lamping and Veach at Google (2014), solves a narrower problem with impressive efficiency. It maps a 64-bit key to a bucket number in `[0, N)`. It uses **no memory**, runs in **O(log N)** time, and moves the theoretical minimum of keys when `N` grows.

```python
def jump_hash(key: int, num_buckets: int) -> int:
    """Lamping & Veach, 'A Fast, Minimal Memory, Consistent Hash Algorithm' (2014)."""
    b, j = -1, 0
    while j < num_buckets:
        b = j
        key = (key * 2862933555777941757 + 1) & 0xFFFFFFFFFFFFFFFF   # 64-bit LCG step
        j = int((b + 1) * ((1 << 31) / ((key >> 33) + 1)))
    return b
```

The same thing in C, where it is usually deployed:

```c
int32_t jump_consistent_hash(uint64_t key, int32_t num_buckets) {
    int64_t b = -1, j = 0;
    while (j < num_buckets) {
        b = j;
        key = key * 2862933555777941757ULL + 1;
        j = (b + 1) * ((double)(1LL << 31) / (double)((key >> 33) + 1));
    }
    return b;
}
```

### 6.1 How it works

Imagine growing the cluster one bucket at a time, from 1 to `N`. When bucket `b` is added (going from `b` to `b + 1` buckets), each key should jump into it with probability `1 / (b + 1)`, and otherwise stay where it is. That gives perfectly even distribution with minimal movement at every step.

Simulating every step would take O(N). The trick is that, using a pseudo-random sequence seeded by the key, you can directly compute *the next bucket at which this key will jump*, skipping all the steps where it stays. The expected number of jumps is about `ln N`, which is where the O(log N) comes from. The linear congruential generator makes the sequence deterministic per key, so every client computes the same answer.

### 6.2 Measured movement

With 100,000 keys going from 10 to 11 buckets:

| Scheme       | Keys moved | Moved to new bucket 10 |
|--------------|-----------:|-----------------------:|
| Modulo       | 90,949     | —                      |
| Jump hash    | 9,004      | 9,004 (100%)           |

That's 9.0% moved against an ideal of 9.1%, and **every** moved key went to the new bucket. Balance is also excellent: the per-bucket standard deviation is about what you'd get from perfectly random assignment, with no vnodes involved.

### 6.3 The catch

Jump hash gives you buckets numbered `0` to `N − 1`, and you can only **add or remove at the end**. It can't express "server 3 out of 0–9 died" without renumbering, and renumbering defeats the purpose.

This makes it a perfect fit for **data sharding**, where shards are logical, numbered, and replicated. If a shard's primary dies, a replica takes over the *same shard number*. It is a poor fit for **caches fronting arbitrary servers**, where any individual server can disappear. Use a ring or rendezvous hashing there.

Jump hash also takes an integer key, so hash string keys to 64 bits first.

---

## 7. Choosing between them

| Property                        | Modulo   | Ring + vnodes         | Rendezvous           | Jump hash            |
|---------------------------------|----------|-----------------------|----------------------|----------------------|
| Keys moved on +1 node           | ~N/(N+1) | ~1/(N+1)              | 1/(N+1)              | 1/(N+1)              |
| Lookup time                     | O(1)     | O(log NV)             | O(N)                 | O(log N)             |
| Memory                          | O(1)     | O(NV)                 | O(N) node list       | O(1)                 |
| Balance without tuning          | Perfect  | Poor (needs vnodes)   | Good                 | Excellent            |
| Remove an *arbitrary* node      | —        | Yes                   | Yes                  | No (tail only)       |
| Weights                         | No       | Via vnode count       | Via log method       | No (not natively)    |
| Replica sets                    | Awkward  | Walk clockwise        | Top-n scores         | Combine with replicas|

Rough guidance:

- **Fewer than a few hundred nodes, nodes come and go arbitrarily:** use rendezvous hashing. It's the least code, it's balanced by default, and it handles weights cleanly.
- **Large clusters, or you need range ownership:** use a ring with 100–256 vnodes per node. "Which keys does node X own?" is a natural question on a ring (it's a set of arcs) and is useful for streaming data during rebalancing. This is why Dynamo, Cassandra, and Riak use rings.
- **Numbered shards that only grow:** use jump hash. It needs zero memory, is very fast, and is close to perfectly balanced.
- **Never** use plain modulo for anything where `N` changes and moving keys has a cost.

---

## 8. Practical pitfalls

**Use the same hash and the same node names everywhere.** Consistency only holds if every client builds the same ring. If one client calls a node `10.0.0.5:11211` and another calls it `cache-a`, they disagree on every key that node owns. Hash a stable identifier, not something that changes on restart.

**Membership is the hard part.** Consistent hashing assumes everyone agrees on the node list. In practice that agreement comes from configuration management, a coordination service (ZooKeeper, etcd), or a gossip protocol. While membership is changing, different clients can briefly route the same key differently. Design for that: caches tolerate it, but stores need versioning or read repair.

**Hot keys aren't solved.** Consistent hashing balances *key counts*, not *traffic*. One celebrity key at a million requests per second still lands on one node. Fixes for that live at a different layer: replicating hot keys, adding random suffixes to spread them, or caching them on the client.

**Bounded loads.** Mirrokni, Thorup, and Zadimoghaddam (2016) proposed *consistent hashing with bounded loads*. Each node gets a capacity of `(1 + ε)` times the average, and a key whose ring successor is full moves on to the next node. This caps worst-case load while still keeping movement low, and it is used in Google Cloud Pub/Sub and HAProxy. It's worth knowing if the leftover vnode imbalance from section 3 isn't good enough.

**Rebalancing is still I/O.** Moving `1/N` of the data is the minimum, but it isn't zero. A storage system adding a node still has to stream that data. Ring-based systems usually do it arc by arc, with the new node serving reads only once each arc has finished copying.

---

## 9. Summary

Modulo hashing ties a key's location to the total number of nodes, so changing that number moves almost every key. Consistent hashing ties a key's location to a property of the key and the individual nodes, so changing the node set moves only the keys that were, or now are, on the nodes that changed.

- On the **hash ring**, nodes and keys share one circle, and a key belongs to the next node clockwise. Adding D to our three-node ring moved 2 of 10 keys, while modulo moved all 10. **Virtual nodes** are what make the ring usable in practice: they cut the busiest node's overload from about 2.9× to about 1.16× at 100 vnodes per node.
- **Rendezvous hashing** has each key pick the node with the highest `hash(key, node)`. It's balanced by default, simple, and O(N) per lookup.
- **Jump consistent hash** maps keys to numbered buckets with no memory and near-perfect balance, but only grows or shrinks at the end.

All three move about `1 / (N + 1)` of keys when a node is added, against about `N / (N + 1)` for modulo. At the scale of a real cache or database, that is the difference between a routine capacity change and an outage.
