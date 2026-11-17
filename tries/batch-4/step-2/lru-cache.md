# LRU Caching, Explained in Full

A cache is a small, fast store that sits in front of a large, slow one. The cache cannot hold everything, so whenever it is full and a new item arrives, it has to decide which item to throw away. That decision is the **eviction policy**, and it largely determines how well the cache works.

**Least Recently Used (LRU)** is the most widely used eviction policy. The rule is short: *when you must evict, evict the item that has gone unused for the longest time.* This explainer covers why that rule works, how to implement it so every operation runs in constant time, a step-by-step trace of the data structures on a concrete access sequence, and a comparison with FIFO, LFU, CLOCK, ARC and the theoretical optimum on that same sequence. It ends with complexity analysis, working code and practical notes.

---

## 1. Why "recently used" is a good predictor

Real workloads show **temporal locality**: an item that was accessed a moment ago is likely to be accessed again soon. A user who opened a document will probably scroll it. A function that was called will probably be called again from the same loop. A database row that was just read is likely part of a transaction that will touch it again.

LRU turns that observation into a policy. It treats recency as a stand-in for future demand. The item touched longest ago is the one we have the least evidence for, so it is the one to drop.

The theoretical best policy is Belady's **OPT** (also called MIN). OPT evicts the item whose *next* use is furthest in the future. OPT is impossible to run in practice because it needs to know the future, but it is a useful yardstick. LRU can be read as "OPT, run backwards": it assumes the recent past mirrors the near future and evicts the item whose *last* use is furthest in the past.

LRU also has a formal guarantee. Sleator and Tarjan showed in 1985 that LRU is *k-competitive*. With a cache of size *k*, LRU never incurs more than about *k* times as many misses as OPT would with the same cache size, and that bound is the best any deterministic online policy can achieve. LRU also has the **stack property**: the contents of an LRU cache of size *k* are always a subset of the contents of an LRU cache of size *k*+1. As a result, giving LRU more memory never makes it miss more often. FIFO lacks this property, which is why it can suffer from Bélády's anomaly.

---

## 2. The interface

An LRU cache with capacity *c* supports two operations:

- `get(key)`: if the key is present, return its value and mark it as the most recently used. Otherwise report a miss.
- `put(key, value)`: insert or update the key and mark it as the most recently used. If this pushes the size above *c*, evict the least recently used key.

Both operations need to be **O(1)**. A cache that takes O(n) time per lookup defeats its own purpose.

---

## 3. The data structures: a hash map plus a doubly linked list

Two requirements pull in different directions:

1. **Fast lookup by key.** A hash map gives O(1) average lookup, but it keeps no order.
2. **Fast reordering by recency.** We need to move any item to the "most recent" end, and remove the item at the "least recent" end, both in O(1). A doubly linked list can do both, provided we already hold a pointer to the node we want to move.

The standard design combines them:

- A **doubly linked list** of nodes, ordered by recency. The head end holds the most recently used (MRU) item and the tail end holds the least recently used (LRU) item.
- A **hash map** from key to *list node*. It does not map key to value. The map hands us the node pointer, and the list lets us unlink and relink that node in constant time.

```
 hash map                       doubly linked list (MRU → LRU)
 ┌─────┬──────┐
 │ "A" │  ●───┼──────┐      HEAD ⇄ [A] ⇄ [C] ⇄ [B] ⇄ TAIL
 │ "B" │  ●───┼──┐   │              ▲     ▲     ▲
 │ "C" │  ●───┼┐ │   └──────────────┘     │     │
 └─────┴──────┘│ └────────────────────────┼─────┘
               └──────────────────────────┘
```

Two **sentinel** nodes, `HEAD` and `TAIL`, sit at the ends permanently and hold no data. With sentinels, every real node always has a non-null `prev` and `next`, so insert and remove need no special cases for an empty list or for the first and last positions.

Each node stores its `key` as well as its `value`. When we evict the tail node, we need its key to delete the matching entry from the hash map.

### The three primitive operations

Everything is built from three pointer manipulations:

- **`unlink(node)`**: `node.prev.next = node.next; node.next.prev = node.prev`. Two pointer writes.
- **`push_front(node)`**: insert right after `HEAD`. Four pointer writes.
- **`pop_back()`**: take `TAIL.prev`, unlink it and return it.

On top of those:

- **Hit** (`get` finds the key): look up node in the map, `unlink(node)`, `push_front(node)`.
- **Miss with room**: create node, `push_front(node)`, add to map.
- **Miss when full**: `victim = pop_back()`, delete `victim.key` from map, then create and insert the new node.

Every step costs a constant amount of work, so both `get` and `put` run in O(1).

---

## 4. Worked example: tracing LRU step by step

We will use one access sequence throughout this document:

```
A  B  C  A  D  B  A  E  B  C  A  D
1  2  3  4  5  6  7  8  9  10 11 12
```

The cache capacity is **3**. Every access is a "get, and on a miss, load and put". The list is written MRU on the left, LRU on the right.

### Full hit/miss table

| Step | Access | Result | Evicted | List after (MRU → LRU) | Map keys |
|---:|:---:|:---:|:---:|:---|:---|
| 1 | A | miss | — | A | {A} |
| 2 | B | miss | — | B, A | {A, B} |
| 3 | C | miss | — | C, B, A | {A, B, C} |
| 4 | A | **hit** | — | A, C, B | {A, B, C} |
| 5 | D | miss | B | D, A, C | {A, C, D} |
| 6 | B | miss | C | B, D, A | {A, B, D} |
| 7 | A | **hit** | — | A, B, D | {A, B, D} |
| 8 | E | miss | D | E, A, B | {A, B, E} |
| 9 | B | **hit** | — | B, E, A | {A, B, E} |
| 10 | C | miss | A | C, B, E | {B, C, E} |
| 11 | A | miss | E | A, C, B | {A, B, C} |
| 12 | D | miss | B | D, A, C | {A, C, D} |

**LRU result: 3 hits, 9 misses (hit rate 25%).**

### Zooming in on the pointer work

Here is what the hash map and linked list actually do on three representative steps.

**Step 4: a hit on A.** Before the access the list is `HEAD ⇄ C ⇄ B ⇄ A ⇄ TAIL`.

1. `map["A"]` returns the node for A in O(1).
2. `unlink(A)`: `B.next = TAIL`, `TAIL.prev = B`. The list is now `HEAD ⇄ C ⇄ B ⇄ TAIL`.
3. `push_front(A)`: `A.prev = HEAD`, `A.next = C`, `C.prev = A`, `HEAD.next = A`. The list is now `HEAD ⇄ A ⇄ C ⇄ B ⇄ TAIL`.

The map does not change, because the node object for A is the same and only its neighbours moved.

**Step 5: a miss on D with a full cache.** Before the access the list is `HEAD ⇄ A ⇄ C ⇄ B ⇄ TAIL`.

1. `map["D"]` is absent, so this is a miss. The caller fetches D from the slow store.
2. The size is 3, which equals capacity, so we evict. `pop_back()` takes `TAIL.prev`, which is B, and unlinks it: `C.next = TAIL`, `TAIL.prev = C`.
3. Delete `map["B"]`. The node remembered its key, which is why this step is O(1).
4. Create node D, `push_front(D)`, and set `map["D"] = D`. The list is now `HEAD ⇄ D ⇄ A ⇄ C ⇄ TAIL`.

Note why B was the victim. A had been refreshed at step 4, so the oldest untouched item was B, last used at step 2. FIFO would have evicted A here, since A was loaded first, even though A had just been used.

**Step 9: a hit on B, deep in the list.** Before the access the list is `HEAD ⇄ E ⇄ A ⇄ B ⇄ TAIL`. B sits at the LRU position and would be the next item evicted. The hit rescues it. The node is unlinked from the tail and pushed to the front, giving `HEAD ⇄ B ⇄ E ⇄ A ⇄ TAIL`. This is the essence of LRU: any single touch resets an item's age to zero.

### What went wrong at steps 10 to 12

Steps 10, 11 and 12 are all misses, and each one evicts something that is needed again soon. At step 10, C evicts A, and A is needed at step 11. A then evicts E, which is harmless, and D evicts B. The working set across steps 8 to 12 is {A, B, C, D, E}, five keys, and a cache of three cannot hold it. Once the working set exceeds capacity, LRU starts to **thrash**: it keeps evicting items just before they come back. Section 7 returns to this.

---

## 5. Comparison on the same sequence

Same sequence, same capacity of 3. A ghost hit in ARC counts as a miss, because the data was not in the cache.

### FIFO (First In, First Out)

FIFO evicts whichever item was *loaded* earliest and ignores hits completely.

| Step | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9 | 10 | 11 | 12 |
|---|---|---|---|---|---|---|---|---|---|---|---|---|
| Access | A | B | C | A | D | B | A | E | B | C | A | D |
| Result | m | m | m | **H** | m | **H** | m | m | m | m | m | m |
| Evicted | | | | | A | | B | C | D | A | E | B |

**FIFO: 2 hits.** At step 5 it throws out A despite the step-4 hit, and that costs a miss at step 7.

### LFU (Least Frequently Used)

LFU keeps a use count per resident item and evicts the lowest count. Ties are broken by recency, evicting the least recent among the tied items. Counts are reset when an item is evicted.

| Step | Access | Result | Evicted | Contents with counts |
|---:|:---:|:---:|:---:|:---|
| 1 | A | miss | — | A:1 |
| 2 | B | miss | — | A:1, B:1 |
| 3 | C | miss | — | A:1, B:1, C:1 |
| 4 | A | **hit** | — | A:2, B:1, C:1 |
| 5 | D | miss | B (tie with C, older) | A:2, C:1, D:1 |
| 6 | B | miss | C (tie with D, older) | A:2, D:1, B:1 |
| 7 | A | **hit** | — | A:3, D:1, B:1 |
| 8 | E | miss | D (tie with B, older) | A:3, B:1, E:1 |
| 9 | B | **hit** | — | A:3, B:2, E:1 |
| 10 | C | miss | E | A:3, B:2, C:1 |
| 11 | A | **hit** | — | A:4, B:2, C:1 |
| 12 | D | miss | C | A:4, B:2, D:1 |

**LFU: 4 hits.** LFU wins over LRU at step 10. LRU evicted A because A had not been touched for three steps. LFU saw that A had been used three times and evicted the one-off E instead. In this sequence A is the "hot" key, and frequency captures that better than recency.

The flip side is that LFU struggles when popularity *changes*. A key that was hot an hour ago keeps a high count and can squat in the cache long after demand has moved on. Production LFU variants add aging or decay to counts for this reason.

### CLOCK (Second Chance)

CLOCK approximates LRU cheaply. Frames are arranged in a circle with a "hand", and each frame has one reference bit. A hit just sets the bit to 1, with no list manipulation. On a miss, the hand sweeps: a frame with bit 1 is given a second chance (bit cleared, hand moves on), and the first frame with bit 0 is evicted. In this trace a newly loaded page gets bit 1, and the hand advances past it.

Notation: `X¹` means page X with reference bit 1, and `▸` marks the hand position after the step.

| Step | Access | Result | Sweep / eviction | Frames after |
|---:|:---:|:---:|:---|:---|
| 1 | A | miss | fill f0 | A¹, ▸—, — |
| 2 | B | miss | fill f1 | A¹, B¹, ▸— |
| 3 | C | miss | fill f2 | ▸A¹, B¹, C¹ |
| 4 | A | **hit** | set A's bit (already 1) | ▸A¹, B¹, C¹ |
| 5 | D | miss | clear A, B, C; wrap; evict A | D¹, ▸B⁰, C⁰ |
| 6 | B | **hit** | set B's bit | D¹, ▸B¹, C⁰ |
| 7 | A | miss | clear B; evict C | ▸D¹, B⁰, A¹ |
| 8 | E | miss | clear D; evict B | D⁰, E¹, ▸A¹ |
| 9 | B | miss | clear A; evict D | B¹, ▸E¹, A⁰ |
| 10 | C | miss | clear E; evict A | ▸B¹, E⁰, C¹ |
| 11 | A | miss | clear B; evict E | B⁰, A¹, ▸C¹ |
| 12 | D | miss | clear C; evict B | D¹, ▸A¹, C⁰ |

**CLOCK: 2 hits.** Step 5 shows CLOCK's known weakness. When every bit is set, the hand clears them all, comes back around, and evicts whatever it started on. In that situation CLOCK behaves exactly like FIFO, which is why it evicts the freshly used A. On long real traces with many frames this case is rare and CLOCK tracks LRU closely. On a three-frame toy it shows up immediately. CLOCK's appeal is cost: a hit is a single bit write with no locking and no pointer updates, which is why operating systems use it (or variants such as CLOCK-Pro) for page replacement.

### ARC (Adaptive Replacement Cache)

ARC (Megiddo and Modha, 2003) keeps four lists:

- **T1**: resident items seen *once* recently (the recency side).
- **T2**: resident items seen *at least twice* (the frequency side).
- **B1, B2**: *ghost* lists that hold only the keys of items recently evicted from T1 and T2, with no data.

The target size of T1 is the parameter *p*. A ghost hit in B1 means "we evicted a recency item too early", so *p* grows. A ghost hit in B2 means "we evicted a frequency item too early", so *p* shrinks. ARC tunes itself between LRU-like and LFU-like behaviour as the workload changes. Lists below are written LRU → MRU.

| Step | Access | Case | Result | p | T1 | T2 | B1 | B2 |
|---:|:---:|:---|:---:|:---:|:---|:---|:---|:---|
| 1 | A | new | miss | 0 | A | | | |
| 2 | B | new | miss | 0 | A, B | | | |
| 3 | C | new | miss | 0 | A, B, C | | | |
| 4 | A | in T1 → T2 | **hit** | 0 | B, C | A | | |
| 5 | D | new; T1 > p, demote B | miss | 0 | C, D | A | B | |
| 6 | B | ghost in B1; p↑; demote C | miss | 1 | D | A, B | C | |
| 7 | A | in T2 | **hit** | 1 | D | B, A | C | |
| 8 | E | new; T1 = p, demote B from T2 | miss | 1 | D, E | A | C | B |
| 9 | B | ghost in B2; p↓; demote D | miss | 0 | E | A, B | C, D | |
| 10 | C | ghost in B1; p↑; demote A from T2 | miss | 1 | E | B, C | D | A |
| 11 | A | ghost in B2; p↓; demote E | miss | 0 | | B, C, A | D, E | |
| 12 | D | ghost in B1; p↑; demote B from T2 | miss | 1 | | C, A, D | E | B |

**ARC: 2 hits**, but look at *why* it misses. Six of its nine misses (steps 6 and 9 through 12) are ghost hits. ARC correctly recognises that it has seen these keys before, and it is adjusting *p* back and forth trying to settle on a split. Twelve accesses with a five-key working set and three slots is too short and too hostile for the adaptation to pay off: each correction is undone by the next access. On long traces ARC usually matches or beats both LRU and LFU, and it is also **scan-resistant**. A one-pass scan only ever enters T1, so it cannot flush the frequently used items in T2. The lesson is that tiny traces show *mechanics*, not steady-state hit rates.

### OPT (Belady's optimal, for reference)

OPT evicts the resident item whose next use is furthest away.

- Step 5 (D): next uses are A at step 7, B at 6, C at 10. Evict **C**.
- Step 8 (E): next uses are A at 11, B at 9, D at 12. Evict **D**.
- Step 10 (C): B and E are never used again. Evict **B**.
- Step 12 (D): nothing is used again, so any eviction will do.

Hits at steps 4, 6, 7, 9 and 11. **OPT: 5 hits.**

### Scoreboard

| Policy | Hits | Misses | Hit rate | Per-hit cost | Key idea |
|---|---:|---:|---:|---|---|
| OPT | 5 | 7 | 42% | n/a (needs the future) | furthest next use |
| LFU | 4 | 8 | 33% | O(1) with frequency buckets | most uses wins |
| **LRU** | **3** | **9** | **25%** | O(1), 6 pointer writes | most recent wins |
| FIFO | 2 | 10 | 17% | O(1), nothing | load order only |
| CLOCK | 2 | 10 | 17% | O(1), one bit write | approximate LRU |
| ARC | 2 | 10 | 17% | O(1), more bookkeeping | adaptive recency/frequency |

Do not read this table as a ranking of the policies in general. It is one short, adversarial sequence. On typical workloads the usual order is OPT > ARC ≈ tuned LFU variants > LRU > CLOCK > FIFO, with large swings depending on the access pattern. The table is useful because each row shows a different *mechanism* responding to identical input.

---

## 6. A second, smaller example: the sequential-scan trap

Capacity 3, a loop over four keys:

```
A B C D A B C D A B C D
```

| Step | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9 | 10 | 11 | 12 |
|---|---|---|---|---|---|---|---|---|---|---|---|---|
| Access | A | B | C | D | A | B | C | D | A | B | C | D |
| LRU | m | m | m | m (ev A) | m (ev B) | m (ev C) | m (ev D) | m (ev A) | m | m | m | m |

**LRU: 0 hits.** Every eviction removes exactly the key that is needed next. This is LRU's classic pathology, called *sequential flooding*. It hits LRU whenever a loop or scan is even one item larger than the cache. OPT gets 5 hits here (it keeps two of the keys pinned), and MRU, "evict the most recent", gets 5 hits too. Large database scans and full-table reads cause this in real systems. It is why PostgreSQL uses ring buffers for big sequential scans, and why Linux, InnoDB and others split their LRU into "young" and "old" regions so that one-touch scan pages never reach the protected region.

---

## 7. Complexity

Let *c* be the capacity and *n* the number of operations.

| Operation | Time | Notes |
|---|---|---|
| `get` (hit) | O(1) average | hash lookup + unlink + push_front |
| `get` (miss) | O(1) average | hash lookup only |
| `put` (new key, not full) | O(1) average | allocate node + push_front + map insert |
| `put` (new key, full) | O(1) average | pop_back + map delete + insert |
| `put` (existing key) | O(1) average | update value + move to front |
| *n* operations | O(n) | |
| Space | O(c) | one node (key, value, 2 pointers) + one map entry per item |

"Average" refers to the hash map. Under adversarial collisions a naive hash map degrades to O(c) per lookup. Randomised hashing (SipHash in Python and Rust) or tree-backed buckets (Java 8+ `HashMap`) bound that risk.

**Why not something simpler?**

- *Array plus timestamps*: O(1) hit, but finding the minimum timestamp to evict is O(c).
- *Min-heap keyed by last-use time*: O(log c) for every hit (decrease-key) and every eviction.
- *Singly linked list*: unlinking a node needs its predecessor, which costs O(c) to find. The doubly linked list stores `prev` precisely to avoid that.
- *Hash map alone*: no order, so eviction is O(c).

The hash map plus doubly linked list is the simplest structure that makes *every* operation O(1).

**Memory overhead.** Each entry carries two pointers plus the key duplicated in the node, which is roughly 16 to 40 bytes of bookkeeping per item on a 64-bit system on top of the hash table's own overhead. For caches of millions of small objects this matters, and it is one reason CLOCK (one bit per entry) remains popular.

---

## 8. Code

### Python, from scratch

```python
class _Node:
    __slots__ = ("key", "value", "prev", "next")

    def __init__(self, key=None, value=None):
        self.key = key
        self.value = value
        self.prev = None
        self.next = None


class LRUCache:
    def __init__(self, capacity: int):
        if capacity <= 0:
            raise ValueError("capacity must be positive")
        self.capacity = capacity
        self.map = {}                 # key -> _Node
        self.head = _Node()           # sentinel: MRU side
        self.tail = _Node()           # sentinel: LRU side
        self.head.next = self.tail
        self.tail.prev = self.head

    # --- list primitives -------------------------------------------------
    def _unlink(self, node):
        node.prev.next = node.next
        node.next.prev = node.prev

    def _push_front(self, node):
        node.prev = self.head
        node.next = self.head.next
        self.head.next.prev = node
        self.head.next = node

    def _pop_back(self):
        node = self.tail.prev
        self._unlink(node)
        return node

    # --- public API ------------------------------------------------------
    def get(self, key):
        node = self.map.get(key)
        if node is None:
            return None               # miss
        self._unlink(node)            # hit: refresh recency
        self._push_front(node)
        return node.value

    def put(self, key, value):
        node = self.map.get(key)
        if node is not None:          # update existing
            node.value = value
            self._unlink(node)
            self._push_front(node)
            return
        if len(self.map) == self.capacity:
            victim = self._pop_back()
            del self.map[victim.key]  # why the node stores its key
        node = _Node(key, value)
        self.map[key] = node
        self._push_front(node)

    def keys_mru_to_lru(self):
        out, cur = [], self.head.next
        while cur is not self.tail:
            out.append(cur.key)
            cur = cur.next
        return out
```

Replaying the worked example reproduces the table in Section 4:

```python
cache, hits = LRUCache(3), 0
for k in "ABCADBAEBCAD":
    if cache.get(k) is not None:
        hits += 1
    else:
        cache.put(k, k.lower())
    print(k, cache.keys_mru_to_lru())
print("hits:", hits)   # hits: 3
```

### Python, the idiomatic shortcut

`collections.OrderedDict` is itself a hash map threaded with a doubly linked list, so it already provides the structure:

```python
from collections import OrderedDict

class LRUCache:
    def __init__(self, capacity):
        self.capacity = capacity
        self.data = OrderedDict()

    def get(self, key):
        if key not in self.data:
            return None
        self.data.move_to_end(key)          # MRU = end
        return self.data[key]

    def put(self, key, value):
        self.data[key] = value
        self.data.move_to_end(key)
        if len(self.data) > self.capacity:
            self.data.popitem(last=False)   # evict LRU = front
```

For memoising pure functions, `functools.lru_cache(maxsize=N)` does all of this for you.

### Java

`LinkedHashMap` in access-order mode is an LRU cache once you override one hook:

```java
import java.util.LinkedHashMap;
import java.util.Map;

class LRUCache<K, V> extends LinkedHashMap<K, V> {
    private final int capacity;

    LRUCache(int capacity) {
        super(16, 0.75f, true);   // true = access order, not insertion order
        this.capacity = capacity;
    }

    @Override
    protected boolean removeEldestEntry(Map.Entry<K, V> eldest) {
        return size() > capacity;
    }
}
```

None of these versions are thread-safe. Wrapping every call in a single mutex is correct but serialises all readers, because in LRU even a `get` *writes* to the list. Section 9 discusses how real systems get around this.

---

## 9. LRU in practice

**Concurrency.** Because every hit mutates shared list pointers, a strict LRU under high concurrency becomes a lock-contention hotspot. Common remedies:

- *Sharding*: split the key space into N independent LRU caches, each with its own lock.
- *Buffered promotion*: record hits in a per-thread buffer and apply them to the list in batches. Caffeine (Java) works this way.
- *Approximation*: CLOCK or sampled LRU. Redis does not keep a global list at all. On eviction it samples a handful of keys (5 by default) and evicts the one with the oldest access timestamp. That gives near-LRU quality with no per-hit pointer work.

**Scan resistance.** As Section 6 showed, plain LRU is easily flushed by one-off scans. Production systems often use segmented designs:

- *SLRU / midpoint insertion*: new items enter a probationary segment and are promoted to a protected segment only on a second hit. MySQL InnoDB inserts new pages at the 3/8 point of its list.
- *2Q* and *LRU-K*: decide on the time of the K-th most recent access rather than only the last one.
- *ARC* and *CAR*: adaptive, ghost-list driven, as traced above.
- *W-TinyLFU*: a small LRU "window" in front of a frequency-filtered main segment, with a compact count-min sketch deciding admission. This is the default in Caffeine and is close to optimal on many traces.

**TTL and size.** Real caches usually combine LRU with expiry (time to live), so stale data is dropped even if it is popular. When entries vary in size, they also evict by *bytes* rather than by count: keep popping the tail until enough space is freed.

**Where you will meet LRU:** CPU caches (usually pseudo-LRU, since true LRU across 8 or 16 ways is too costly in hardware), OS page caches (CLOCK variants), database buffer pools, CDN and reverse-proxy caches, browser caches, DNS resolvers, ORM identity maps, and memoisation decorators.

---

## 10. Choosing a policy

- Start with **LRU**. It is simple, predictable, O(1), has a provable competitive bound, and is good enough for most application-level caches.
- Switch to a **frequency-aware** policy (W-TinyLFU, LFU with decay) when a stable set of hot keys is being evicted by bursts of one-off keys. The LFU row of the worked example shows this effect.
- Use **ARC or a segmented LRU** when the workload alternates between recency-heavy and frequency-heavy phases, or when large scans are common.
- Use **CLOCK or sampled LRU** when per-hit overhead or lock contention matters more than the last few percent of hit rate.
- Always **measure** on a recorded trace of your real traffic. As the scoreboard shows, a short or unusual sequence can invert the usual ranking.

---

## Summary

LRU evicts the item that has gone unused the longest, betting that the recent past predicts the near future. A hash map from key to node, combined with a doubly linked list ordered by recency, makes lookup, promotion and eviction all O(1) in O(c) space. Every hit unlinks a node and pushes it to the front, and every eviction pops the tail and deletes its key from the map. On the twelve-access example, LRU scored 3 hits against OPT's 5. LFU did better because one key was persistently hot, while FIFO, CLOCK and a not-yet-adapted ARC did worse. LRU's main weaknesses are scans larger than the cache and contention on its shared list. Most production caches are LRU at heart, with segmentation, frequency filtering, sampling or sharding added to address exactly those weaknesses.
