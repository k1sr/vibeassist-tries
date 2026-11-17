# Skip Lists: A Complete Explainer

A skip list is a sorted linked list with extra "express lanes" stacked on top of it. Each express lane skips over a random subset of the elements below it, so a search can move fast across the top lanes and then drop down to slower lanes as it gets close to its target. William Pugh introduced the structure in 1990 as "a probabilistic alternative to balanced trees". It gives the same expected `O(log n)` search, insert and delete as a red-black tree or AVL tree, but it has no rotations, no recoloring and no rebalancing rules. Randomness does the balancing.

This document covers the structure, then walks through a concrete list with search traces, an insertion driven by coin flips, a deletion, a derivation of the expected cost, concurrent variants, and how Redis uses skip lists for sorted sets. A full working implementation comes at the end.

---

## 1. The Core Idea

Start with a sorted singly linked list:

```
3 -> 6 -> 7 -> 9 -> 12 -> 17 -> 19 -> 21 -> 25 -> 26
```

Searching it takes `O(n)` steps, because you can only move forward one node at a time. Now suppose every second node also has a pointer to the node two places ahead. That second lane has `n/2` nodes, so a search can cover the list in about `n/2` steps and then take at most one step on the bottom lane. Add a third lane linking every fourth node, then a fourth linking every eighth node, and so on. With `log2 n` lanes, a search looks at a constant number of nodes per lane, which gives `O(log n)` in total.

This perfectly regular layout is fragile. Inserting one element shifts the position of every element after it, and keeping "every 2^i-th node" exact would mean rebuilding large parts of the structure. Pugh's insight was to drop the exact pattern and keep only the proportions. **Each node picks its own height at random**: it is on level 1 always, on level 2 with probability `p`, on level 3 with probability `p^2`, and so on. With `p = 1/2`, about half the nodes reach level 2, a quarter reach level 3, and so on. That is the same shape as the regular layout in expectation, and a new node never forces any other node to change.

### Terminology

- **Level** (or lane): a sorted linked list. Level 1 contains every element, and each higher level contains a subset of the level below it.
- **Height** of a node: how many levels the node appears on. A node of height 3 has three forward pointers, `next[1]`, `next[2]` and `next[3]`.
- **Head**: a sentinel node with the maximum height. It holds no key and acts as the start of every level.
- **NIL**: the end of every level. You can treat it as a key of `+infinity`.
- **p**: the promotion probability, usually 1/2 or 1/4.
- **MaxLevel**: a cap on heights, normally about `log_{1/p}(N)` for the largest `N` you expect.

---

## 2. A Concrete Multi-Level List

Here is the list used throughout this document. It holds ten keys, and each key's height came from coin flips:

| Key    | 3 | 6 | 7 | 9 | 12 | 17 | 19 | 21 | 25 | 26 |
|--------|---|---|---|---|----|----|----|----|----|----|
| Height | 1 | 3 | 1 | 2 | 1  | 4  | 1  | 2  | 3  | 1  |

Drawn level by level:

```
Level 4:  H ---------------------------------> 17 ---------------------------------> NIL
Level 3:  H ------> 6 -----------------------> 17 ---------------------> 25 -------> NIL
Level 2:  H ------> 6 ---------> 9 ----------> 17 ---------> 21 -------> 25 -------> NIL
Level 1:  H -> 3 -> 6 -> 7 ----> 9 ---> 12 --> 17 --> 19 --> 21 -------> 25 -> 26 -> NIL
```

Each column is one node. Node 17, for example, is a single node object with four forward pointers: on level 4 it points to NIL, on level 3 to 25, on level 2 to 21 and on level 1 to 19. The same key does not appear in four separate nodes.

The level sizes are 10, 5, 3 and 1. With `p = 1/2` we would expect 10, 5, 2.5 and 1.25, so this list is a typical draw.

---

## 3. Searching

### The algorithm

```
x = head
for level from top down to 1:
    while x.next[level] != NIL and x.next[level].key < target:
        x = x.next[level]          # move right
    # otherwise drop down one level
x = x.next[1]
return x if x != NIL and x.key == target else NOT FOUND
```

The rule is simple. On each level, move right as long as the next key is smaller than the target. When the next key is too big, or is NIL, drop down a level. When you fall off the bottom, the node just to your right is the target if the target exists.

### Trace 1: search for 19

| Step | Level | At    | Next key | Decision                  |
|------|-------|-------|----------|---------------------------|
| 1    | 4     | H     | 17       | 17 < 19, move to 17       |
| 2    | 4     | 17    | NIL      | drop to level 3           |
| 3    | 3     | 17    | 25       | 25 > 19, drop to level 2  |
| 4    | 2     | 17    | 21       | 21 > 19, drop to level 1  |
| 5    | 1     | 17    | 19       | 19 is not < 19, stop      |

After the loop, `x = 17` and `x.next[1] = 19`, which matches. **Found, with 5 comparisons.** A plain linked list would have needed 7, since 19 is the seventh key. That is a small saving, but the gap grows quickly with `n`.

The path, with `*` marking nodes visited:

```
Level 4:  H* ---------------------------------> 17* -> NIL
Level 3:                                        17* --------------> 25 (too big)
Level 2:                                        17* ------> 21 (too big)
Level 1:                                        17* -> 19  FOUND
```

### Trace 2: search for 12

| Step | Level | At | Next key | Decision                 |
|------|-------|----|----------|--------------------------|
| 1    | 4     | H  | 17       | 17 > 12, drop            |
| 2    | 3     | H  | 6        | 6 < 12, move to 6        |
| 3    | 3     | 6  | 17       | 17 > 12, drop            |
| 4    | 2     | 6  | 9        | 9 < 12, move to 9        |
| 5    | 2     | 9  | 17       | 17 > 12, drop            |
| 6    | 1     | 9  | 12       | stop; 12 == 12, found    |

The search takes 6 comparisons. It moved right twice and dropped down three times. Every search is a staircase of this shape, made of horizontal moves and vertical drops.

### Trace 3: search for 20 (absent)

| Step | Level | At | Next key | Decision               |
|------|-------|----|----------|------------------------|
| 1    | 4     | H  | 17       | move to 17             |
| 2    | 4     | 17 | NIL      | drop                   |
| 3    | 3     | 17 | 25       | drop                   |
| 4    | 2     | 17 | 21       | drop                   |
| 5    | 1     | 17 | 19       | 19 < 20, move to 19    |
| 6    | 1     | 19 | 21       | 21 > 20, stop          |

`x.next[1] = 21`, which is not 20, so the key is **not found**. The search still produced useful information: `x = 19` is the predecessor of 20 and `21` is its successor. Range queries such as "all keys from 20 to 26" are built on this. You find the start point in `O(log n)` and then walk level 1, which costs `O(log n + k)` for `k` results.

---

## 4. Insertion with Coin Flips

Insertion is a search that remembers where it dropped down. On every level, the last node visited before dropping is the node whose forward pointer must change if the new node reaches that level. These nodes are stored in an array called `update`.

### Step 1: choose a height

```
height = 1
while height < MaxLevel and random() < p:
    height += 1
```

With `p = 1/2`, this means flipping a coin until it lands tails, and counting the heads plus one.

Let us insert **15**. The flips come up **Heads, Heads, Tails**, so 15 gets **height 3**.

### Step 2: search and record `update[]`

| Level | Walk                          | `update[level]` |
|-------|-------------------------------|-----------------|
| 4     | H, next 17 > 15, drop         | H               |
| 3     | H -> 6, next 17 > 15, drop    | 6               |
| 2     | 6 -> 9, next 17 > 15, drop    | 9               |
| 1     | 9 -> 12, next 17 > 15, drop   | 12              |

### Step 3: splice in at levels 1..height

For each level `i` from 1 to 3:

```
new.next[i]       = update[i].next[i]
update[i].next[i] = new
```

- Level 1: `15.next = 17` and `12.next = 15`
- Level 2: `15.next = 17` and `9.next = 15`
- Level 3: `15.next = 17` and `6.next = 15`
- Level 4 does not change, because 15 has height 3.

The result:

```
Level 4:  H ---------------------------------------------> 17 ----------------------------> NIL
Level 3:  H ------> 6 ------------------------> 15 ------> 17 ----------------> 25 -------> NIL
Level 2:  H ------> 6 ---------> 9 -----------> 15 ------> 17 -------> 21 ----> 25 -------> NIL
Level 1:  H -> 3 -> 6 -> 7 ----> 9 ---> 12 ---> 15 ------> 17 -> 19 -> 21 ----> 25 -> 26 -> NIL
```

No other node was moved or resized. The insert changed exactly `height` pointers, plus `height` pointers in the new node.

### When the new node is taller than the list

If the flips had produced height 6 while the list's current top level is 4, levels 5 and 6 would have no nodes yet. In that case, set `update[5] = update[6] = head` before splicing, and raise the list's recorded level to 6. The head always has `MaxLevel` pointers, so it is always ready.

### Why the cap matters

Without `MaxLevel`, a long run of heads could produce an absurdly tall node. The probability of height above `k` is `p^k`, so a run of 40 heads at `p = 1/2` almost never happens. The cap still gives a hard bound on the head's size and on the `update` array. A common choice is `MaxLevel = ceil(log_{1/p} N)` for the largest expected `N`. Pugh also suggested capping a new node's height at one above the list's current top level, which stops a single unlucky draw from creating a tall, nearly empty set of levels.

---

## 5. Deletion

Deletion mirrors insertion: search, record `update[]`, then unlink.

Let us delete **17** from the list that now includes 15.

### Search with `update[]`

| Level | Walk                                   | `update[level]` |
|-------|----------------------------------------|-----------------|
| 4     | H, next 17 is not < 17, drop           | H               |
| 3     | H -> 6 -> 15, next 17 not < 17, drop   | 15              |
| 2     | at 15, next 17, drop                   | 15              |
| 1     | at 15, next 17, drop                   | 15              |

Then `x = update[1].next[1] = 17`, which matches.

### Unlink

For each level `i` where `update[i].next[i] == x`:

```
update[i].next[i] = x.next[i]
```

- Level 1: `15.next = 19`
- Level 2: `15.next = 21`
- Level 3: `15.next = 25`
- Level 4: `H.next = NIL`

### Shrink the list level

Level 4 is now empty because head points straight to NIL, so the list's top level drops from 4 to 3:

```
Level 3:  H ------> 6 ------------------------> 15 ---------------------> 25 -------> NIL
Level 2:  H ------> 6 ---------> 9 -----------> 15 ---------> 21 -------> 25 -------> NIL
Level 1:  H -> 3 -> 6 -> 7 ----> 9 ---> 12 ---> 15 -> 19 ---> 21 -------> 25 -> 26 -> NIL
```

After the delete, 15 has taken over much of the role 17 used to play. Nothing was rebalanced. The structure is still a valid skip list, and its expected shape is unchanged because every remaining node's height was chosen independently of the deleted node.

---

## 6. Expected Complexity: The Derivation

The analysis below follows Pugh's "backwards" argument. Fix a target and follow the search path **in reverse**, starting from the bottom-level node where the search ends and working back up and to the left to the head.

### Climbing analysis

At any node on the reversed path, one of two things happened in the forward search:

1. **We arrived from the left**, meaning the forward search moved right into this node at the current level. In reverse, we step left.
2. **We arrived from above**, meaning the forward search dropped down into this node. In reverse, we step up.

When we are at a node on level `i`, the question is whether this node also reaches level `i+1`. The node's height was chosen by independent coin flips, and being on level `i` already tells us its height is at least `i`. The chance that it goes higher is exactly `p`.

- With probability `p` the node is taller, and the path came from above, so in reverse we **climb**.
- With probability `1 - p` the node stops at level `i`, so the path came from the left, and in reverse we move **left** while staying on level `i`.

Let `C(k)` be the expected number of steps needed to climb `k` levels in an infinite list:

```
C(0) = 0
C(k) = (1 - p) * (1 + C(k))  +  p * (1 + C(k - 1))
```

Solving for `C(k)`:

```
C(k) = (1 - p) + (1 - p) C(k) + p + p C(k - 1)
p C(k) = 1 + p C(k - 1)
C(k) = 1/p + C(k - 1)
C(k) = k / p
```

So climbing `k` levels costs `k/p` steps on average. With `p = 1/2` that is 2 steps per level, which matches intuition: on average you go left once and then up once.

### Bounding the number of levels

Define `L(n) = log_{1/p} n`. This is the level where we expect about `1/p` nodes, since `n * p^(L(n)-1) = 1/p`. Split the reversed path into two parts.

**Part A: climbing from level 1 to level L(n).** By the climbing analysis, this costs at most `(L(n) - 1) / p` expected steps. In a finite list the climb can only reach the head sooner, so this is an upper bound.

**Part B: everything at or above level L(n).** The horizontal steps in this part are bounded by the number of nodes at level `L(n)` or higher. That count is a binomial random variable with mean `n * p^(L(n)-1) = 1/p`. The vertical steps are bounded by how far the maximum height exceeds `L(n)`. The chance that any node exceeds `L(n) + j` is at most `n * p^(L(n)+j-1) = p^j / p`, which falls off geometrically, so the expected overshoot is at most `1/(1-p)`.

Adding the two parts:

```
E[search cost] <= (L(n) - 1)/p  +  1/p  +  1/(1 - p)
               =  L(n)/p + 1/(1 - p)
               =  (log_{1/p} n) / p  +  O(1)
```

**With p = 1/2:** `2 log2 n + 2`. For a million keys that is about 42 comparisons.
**With p = 1/4:** `4 log4 n + 4/3 = 2 log2 n + 1.33`. The search cost is about the same, but each node carries fewer pointers.

### Space

The expected height of a node is a geometric random variable with mean `1/(1-p)`:

- `p = 1/2`: 2 pointers per node on average
- `p = 1/4`: about 1.33 pointers per node on average

A balanced binary tree always uses exactly 2 child pointers per node, plus balance bits or color bits. With `p = 1/4`, a skip list is leaner.

### High-probability bounds

The expectations above are not the whole story. Standard Chernoff arguments show that the search cost is `O(log n)` **with high probability**, meaning the chance of exceeding `c log n` steps falls as `1/n^(c')` for constants depending on `c`. The maximum height is `O(log n)` with high probability as well. Bad cases exist, but they are exponentially unlikely, and they do not depend on the input order. An adversary who cannot see the random bits cannot force the worst case. This is the key difference from an unbalanced BST, where sorted input alone triggers the worst case.

### Summary table

| Operation          | Expected     | Worst case (theoretical) |
|--------------------|--------------|--------------------------|
| Search             | O(log n)     | O(n)                     |
| Insert             | O(log n)     | O(n)                     |
| Delete             | O(log n)     | O(n)                     |
| Range query (k)    | O(log n + k) | O(n)                     |
| Space              | O(n)         | O(n log n)               |

---

## 7. Indexable Skip Lists: Rank in O(log n)

If every forward pointer also stores a **span**, meaning how many level-1 nodes it skips over, you can compute an element's rank (its 0-based position) during the search: add the span of every pointer you follow. The reverse also works. To find the element at rank `r`, follow pointers while the accumulated span stays at or below `r`.

On insert, the spans of the `update[]` pointers are split between the old pointer and the new one, and the pointers above the new node's height grow by 1. On delete, the spans are merged back and decremented. Section 9 shows that Redis uses exactly this scheme for `ZRANK` and `ZRANGE`.

---

## 8. Concurrent Skip Lists

Skip lists are one of the most widely used concurrent ordered data structures, and the reason is structural. An insert or delete changes only a few local pointers, and there is no rotation that restructures a whole subtree. Concurrent balanced trees have to coordinate rebalancing across a path from the root, while a skip list update touches a handful of neighbors.

### The key invariant: level 1 is the truth

Every concurrent design treats **level 1 as the source of truth for membership**. Higher levels are only hints that speed up searches. If a higher-level link is briefly missing or stale, searches still return correct answers, just a little more slowly.

### Lock-based: the lazy skip list

Herlihy, Lev, Luchangco and Shavit (2007) described a simple, provably correct lock-based design.

- Each node has a lock, a `marked` flag (logically deleted) and a `fullyLinked` flag (the insert has finished).
- **Search is wait-free.** It takes no locks and treats a key as present if the node is found, `fullyLinked` and not `marked`.
- **Insert:** find the `preds` and `succs` arrays (the same idea as `update`), lock the predecessors bottom-up, and **validate** that each predecessor is unmarked and still points to its successor. If validation fails, unlock and retry. If it succeeds, link the new node at every level and then set `fullyLinked`.
- **Delete:** lock the victim and set `marked`. That is the linearization point, the moment the key stops existing. Then lock the predecessors, validate, unlink at every level from top to bottom, and unlock.

### Lock-free: CAS and marked pointers

Lock-free designs come from Harris's lock-free linked list, extended by Fraser (2003) and by Herlihy and Shavit. Doug Lea's `java.util.concurrent.ConcurrentSkipListMap` is the best-known production implementation.

- **Insert, bottom-up:** CAS the new node into level 1 first. **That CAS is the linearization point.** Then link the higher levels one at a time with further CAS operations, retrying on contention. A search that runs before the upper links exist will still find the node through level 1.
- **Delete, mark then unlink:** a deleting thread first **marks** the node's next pointers, usually with a spare low bit in the pointer or, in Java, by setting the value to null and adding a *marker node*. Marking prevents any thread from inserting after the doomed node. The node is then physically unlinked with CAS. Other threads that run into a marked node help unlink it before they continue, which keeps the structure lock-free.
- **Memory reclamation** is the hard part in non-GC languages. A thread may still hold a pointer to an unlinked node, so freeing it right away is unsafe. Implementations use epoch-based reclamation, hazard pointers or reference counting. Java and Go avoid the issue through garbage collection.

### Single writer, many readers

A common middle ground appears in LevelDB and RocksDB memtables. Writers are serialized externally, while readers run concurrently with no locks at all. A new node is fully initialized, then published with **release** stores, from bottom to top, and readers load pointers with **acquire** semantics. Because nodes are never removed while the memtable is live (deletes are written as tombstones), reclamation is not a problem. The arena frees everything at once when the memtable is flushed.

### Trade-offs

Concurrent skip lists perform well under mixed read and write loads and scale across cores. Their weaknesses are pointer chasing, which is unfriendly to caches compared with B-tree-style nodes, and somewhat higher memory use. For read-mostly ordered data, a B+-tree or a sorted array with copy-on-write can be faster. For balanced concurrent workloads, the skip list is a common default.

---

## 9. Skip Lists in Redis Sorted Sets

Redis sorted sets (`ZADD`, `ZRANGE`, `ZRANK` and related commands) are the best-known production skip list. Each member has a score, and the set is ordered by `(score, member)`, with ties on score broken by comparing the member strings.

### Two encodings

- **Small sets** use a compact **listpack**, a flat, sorted byte array that replaced the older ziplist. This applies while the set has at most `zset-max-listpack-entries` members (default 128) and every member is at most `zset-max-listpack-value` bytes (default 64). Linear scans over a small contiguous buffer are fast and use little memory.
- **Larger sets** switch to the `zset` structure, which pairs a **skip list** with a **hash table**:
  - The **dict** maps member to score, so `ZSCORE` is `O(1)` and duplicate members can be detected.
  - The **skip list** keeps members in order for range queries and rank.

### The node layout

Simplified from Redis's `server.h`:

```c
typedef struct zskiplistNode {
    sds ele;                          /* member string */
    double score;
    struct zskiplistNode *backward;   /* level-1 back pointer, for ZREVRANGE */
    struct zskiplistLevel {
        struct zskiplistNode *forward;
        unsigned long span;           /* nodes skipped by this forward link */
    } level[];                        /* flexible array: one entry per level */
} zskiplistNode;

typedef struct zskiplist {
    struct zskiplistNode *header, *tail;
    unsigned long length;
    int level;
} zskiplist;
```

Three choices stand out:

1. **p = 1/4** (`ZSKIPLIST_P 0.25`) and a maximum level of 32 (`ZSKIPLIST_MAXLEVEL`). This gives about 1.33 forward pointers per node while keeping search cost close to that of `p = 1/2`.
2. **Spans** on every forward link make the list indexable (Section 7), so `ZRANK` and `ZRANGE key start stop` by index run in `O(log n)` plus output size.
3. A **backward pointer** on level 1 turns the bottom level into a doubly linked list, which allows reverse iteration for `ZREVRANGE` and finding the tail quickly.

### Mapping commands to skip-list operations

| Command                              | Skip-list operation                               | Cost               |
|--------------------------------------|---------------------------------------------------|--------------------|
| `ZADD key score member` (new)        | dict insert and skip-list insert                  | O(log n)           |
| `ZADD` (existing, score changes)     | delete and reinsert, or update in place if order holds | O(log n)      |
| `ZSCORE key member`                  | dict lookup only                                  | O(1)               |
| `ZRANK key member`                   | dict lookup for score, then search summing spans  | O(log n)           |
| `ZRANGEBYSCORE key min max`          | search for first `>= min`, then walk level 1      | O(log n + k)       |
| `ZRANGE key i j`                     | find rank `i` using spans, then walk              | O(log n + k)       |
| `ZREM key member`                    | dict delete and skip-list delete                  | O(log n)           |

A leaderboard is the classic example. `ZADD board 4200 alice` records a score, `ZREVRANGE board 0 9 WITHSCORES` returns the top 10, and `ZREVRANK board alice` returns Alice's position. Each command costs logarithmic time even with millions of players.

### Why not a balanced tree?

Salvatore Sanfilippo (antirez) gave several reasons for choosing a skip list. They are not much more memory-hungry than trees and can use less with a small `p`. Range operations amount to walking a linked list, which is cache-friendly enough in practice. And they are much simpler to implement, debug and extend. Adding spans for rank, for example, took only a few lines in the insert and delete code. Redis runs commands on a single thread, so concurrency was not the deciding factor. Simplicity was.

---

## 10. Full Implementation (Python)

The code below is a complete, indexable skip list. It supports search, insert (as an upsert), delete, rank, lookup by index and range iteration. It is written for clarity rather than raw speed.

```python
import random
from typing import Any, Iterator, Optional, Tuple


class _Node:
    __slots__ = ("key", "value", "next", "span")

    def __init__(self, key: Any, value: Any, height: int) -> None:
        self.key = key
        self.value = value
        self.next: list[Optional["_Node"]] = [None] * height
        # span[i] = number of level-1 steps that next[i] jumps over
        self.span: list[int] = [0] * height


class SkipList:
    def __init__(self, p: float = 0.5, max_level: int = 32, seed: Optional[int] = None):
        self.p = p
        self.max_level = max_level
        self.head = _Node(None, None, max_level)
        self.level = 1          # number of levels currently in use
        self.length = 0
        self._rng = random.Random(seed)

    # ---- helpers -------------------------------------------------------

    def _random_height(self) -> int:
        h = 1
        while h < self.max_level and self._rng.random() < self.p:
            h += 1
        return h

    def _find_update(self, key: Any) -> Tuple[list, list]:
        """Return (update, rank): update[i] is the last node on level i
        with key < `key`; rank[i] is that node's 1-based position (head = 0)."""
        update: list = [None] * self.max_level
        rank = [0] * self.max_level
        x = self.head
        for i in range(self.level - 1, -1, -1):
            rank[i] = rank[i + 1] if i + 1 < self.level else 0
            while x.next[i] is not None and x.next[i].key < key:
                rank[i] += x.span[i]
                x = x.next[i]
            update[i] = x
        return update, rank

    # ---- queries -------------------------------------------------------

    def search(self, key: Any) -> Optional[Any]:
        x = self.head
        for i in range(self.level - 1, -1, -1):
            while x.next[i] is not None and x.next[i].key < key:
                x = x.next[i]
        x = x.next[0]
        if x is not None and x.key == key:
            return x.value
        return None

    def __contains__(self, key: Any) -> bool:
        x = self.head
        for i in range(self.level - 1, -1, -1):
            while x.next[i] is not None and x.next[i].key < key:
                x = x.next[i]
        x = x.next[0]
        return x is not None and x.key == key

    def __len__(self) -> int:
        return self.length

    def rank(self, key: Any) -> Optional[int]:
        """0-based position of key, or None if absent."""
        update, rank = self._find_update(key)
        x = update[0].next[0]
        if x is not None and x.key == key:
            return rank[0]  # position of predecessor (1-based) == 0-based index of x
        return None

    def at(self, index: int) -> Tuple[Any, Any]:
        """Return (key, value) at 0-based index, in O(log n)."""
        if not 0 <= index < self.length:
            raise IndexError(index)
        target = index + 1          # 1-based position
        traversed = 0
        x = self.head
        for i in range(self.level - 1, -1, -1):
            while x.next[i] is not None and traversed + x.span[i] <= target:
                traversed += x.span[i]
                x = x.next[i]
            if traversed == target:
                return x.key, x.value
        raise AssertionError("unreachable")

    def range(self, lo: Any, hi: Any) -> Iterator[Tuple[Any, Any]]:
        """Yield (key, value) for lo <= key <= hi, in order."""
        x = self.head
        for i in range(self.level - 1, -1, -1):
            while x.next[i] is not None and x.next[i].key < lo:
                x = x.next[i]
        x = x.next[0]
        while x is not None and x.key <= hi:
            yield x.key, x.value
            x = x.next[0]

    # ---- updates -------------------------------------------------------

    def insert(self, key: Any, value: Any = None) -> None:
        update, rank = self._find_update(key)

        existing = update[0].next[0]
        if existing is not None and existing.key == key:
            existing.value = value          # upsert
            return

        h = self._random_height()
        if h > self.level:
            for i in range(self.level, h):
                rank[i] = 0
                update[i] = self.head
                update[i].span[i] = self.length   # head spans the whole list
            self.level = h

        node = _Node(key, value, h)
        for i in range(h):
            node.next[i] = update[i].next[i]
            update[i].next[i] = node
            # split the old span between update[i] -> node -> old successor
            node.span[i] = update[i].span[i] - (rank[0] - rank[i])
            update[i].span[i] = (rank[0] - rank[i]) + 1

        # levels above h: the pointer now jumps over one more node
        for i in range(h, self.level):
            update[i].span[i] += 1

        self.length += 1

    def delete(self, key: Any) -> bool:
        update, _ = self._find_update(key)
        x = update[0].next[0]
        if x is None or x.key != key:
            return False

        for i in range(self.level):
            if update[i].next[i] is x:
                update[i].span[i] += x.span[i] - 1
                update[i].next[i] = x.next[i]
            else:
                update[i].span[i] -= 1

        while self.level > 1 and self.head.next[self.level - 1] is None:
            self.level -= 1
        self.length -= 1
        return True

    # ---- debugging -----------------------------------------------------

    def __iter__(self) -> Iterator[Tuple[Any, Any]]:
        x = self.head.next[0]
        while x is not None:
            yield x.key, x.value
            x = x.next[0]

    def dump(self) -> str:
        lines = []
        for i in range(self.level - 1, -1, -1):
            parts, x = ["H"], self.head.next[i]
            while x is not None:
                parts.append(str(x.key))
                x = x.next[i]
            parts.append("NIL")
            lines.append(f"L{i + 1}: " + " -> ".join(parts))
        return "\n".join(lines)
```

### Using it

```python
sl = SkipList(seed=42)
for k in [3, 6, 7, 9, 12, 17, 19, 21, 25, 26]:
    sl.insert(k, f"v{k}")

print(sl.dump())
print(sl.search(19))            # 'v19'
print(20 in sl)                 # False
sl.insert(15, "v15")
print(sl.rank(15))              # 5  (0-based: 3, 6, 7, 9, 12, 15)
print(sl.at(5))                 # (15, 'v15')
print(list(sl.range(10, 21)))   # [(12,'v12'), (15,'v15'), (17,'v17'), (19,'v19'), (21,'v21')]
sl.delete(17)
print([k for k, _ in sl])       # [3, 6, 7, 9, 12, 15, 19, 21, 25, 26]
```

### Notes on the span bookkeeping

`rank[i]` is the 1-based position of `update[i]`, with the head at 0. The new node will sit at position `rank[0] + 1`. On level `i`, the pointer from `update[i]` to the new node therefore spans `(rank[0] + 1) - rank[i]` positions. The new node's own pointer takes whatever is left of the old span. That remainder is `old_span - (rank[0] - rank[i])`, because the old span already counted the new position once the insert has shifted everything after it by one. On delete, the two spans join back together, minus one for the removed node. Levels the node never reached only need a `+1` or `-1`. This is the same arithmetic Redis uses in `zslInsert` and `zslDeleteNode`.

---

## 11. Practical Considerations

- **Choosing p.** `1/2` gives the shortest search paths and `1/4` gives the smallest memory footprint with almost the same speed. Values below about `1/8` make searches noticeably longer.
- **Cache behavior.** Each step dereferences a pointer to a node that may be anywhere in memory. Allocating nodes from an arena, as LevelDB does, or using "fat" nodes that hold several keys, improves locality.
- **Duplicates.** To allow duplicate keys, either compare on a tiebreaker as Redis does with `(score, member)`, or change the search to use `<=` and insert after equal keys.
- **Determinism.** For reproducible tests, seed the random number generator. In adversarial settings, do not let users see or predict it.
- **Deterministic variants.** The 1-2-3 skip list of Munro, Papadakis and Sedgewick guarantees `O(log n)` in the worst case by enforcing gap rules. It is essentially a B-tree in disguise, and in practice it is rarely worth the added complexity.

---

## 12. Summary

A skip list is a stack of sorted linked lists where each level holds a random subset of the one below it. Searches walk a staircase from the top-left down to the target. Inserts flip coins to pick a height and splice the new node into that many levels. Deletes unlink the node from every level it occupies. A short backwards analysis shows that the expected path length is `(log_{1/p} n)/p + O(1)`, and the space cost is `n/(1-p)` pointers. Because every update touches only local pointers, skip lists adapt naturally to concurrency, which is why Java's `ConcurrentSkipListMap` and the LSM memtables in LevelDB and RocksDB use them. Spans make the list indexable, and that small extension is what lets Redis sorted sets answer rank and range queries in logarithmic time. Skip lists match balanced trees on asymptotic performance with much simpler code, and in practice that simplicity is their biggest advantage.
