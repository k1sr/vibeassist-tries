# B-tree and B+tree Indexes: How Databases Find Rows Fast

PostgreSQL, MySQL/InnoDB, SQL Server, Oracle and SQLite all keep most of their indexes in B-trees. Knowing how the structure works lets you predict query plans and design indexes deliberately. This explainer covers the structure, works through inserts, deletes and range scans on concrete keys, shows how a node sits on a disk page, and then explains how real databases use the result.

---

## 1. The problem B-trees solve

A table is a pile of rows stored on fixed-size **pages**, usually 4 KB, 8 KB or 16 KB. Without an index, `SELECT * FROM users WHERE id = 48213` has to read every page. That is a full scan, O(N).

A balanced binary search tree would cut that to about log₂(N) comparisons. For a billion rows that is roughly 30 levels. If each level is a separate page read from disk, that means 30 random I/Os per lookup, which is far too many. The cost that matters on disk is **page reads**, not comparisons.

A B-tree's answer is to make every node a whole page and fill it with hundreds of keys. A node with 400 children cuts the search space by 400× at each step instead of 2×, so a billion keys fit in a tree only **four levels** deep. The top two levels are small enough to stay in memory all the time, so a point lookup usually costs one or two real disk reads.

---

## 2. B-tree vs. B+tree

The name "B-tree" covers a family of structures. Two variants matter here.

**Classic B-tree (Bayer and McCreight, 1972).** Every node, internal or leaf, holds keys *and* their associated values (or row pointers). A search can stop at an internal node if the key is found there.

**B+tree.** Only the **leaves** hold values. Internal nodes hold only *separator keys* and child pointers, and act purely as a routing directory. The leaves are linked into a **doubly linked list** in key order.

Real databases almost always use the B+tree. When a database's documentation says "B-tree index", it nearly always means a B+tree (or a close relative such as PostgreSQL's B-link tree). There are three reasons:

1. **Higher fanout.** Internal nodes carry no values, so more separators fit per page. That makes the tree shallower.
2. **Cheap range scans.** Once you reach the first matching leaf, you follow sibling pointers sideways. You never climb back up the tree.
3. **Uniform cost.** Every lookup goes all the way to a leaf, so performance is predictable.

The rest of this document uses the B+tree.

### Invariants

For a B+tree of order *m* (maximum children per internal node):

- Every leaf is at the same depth. The tree is always perfectly height-balanced.
- An internal node with *k* keys has *k + 1* children. Keys in child *i* fall between separator *i − 1* (inclusive) and separator *i* (exclusive).
- Every node except the root is at least about half full. This guarantees O(log N) height and ≥ 50% space use.
- Keys within a node are sorted.
- Leaves are chained left-to-right in key order.

---

## 3. A small tree to work with

To keep the examples readable, we use a tiny capacity:

- **Leaves** hold at most **3 keys** and at least **2** (except when the leaf is the root).
- **Internal nodes** hold at most **3 keys** (4 children) and at least **1 key** (2 children), except the root.

Real pages hold hundreds of keys, but the mechanics are the same. In the diagrams, `[ ]` is a node and `→` is the leaf sibling link. When a node splits, the split is "left gets the smaller half".

---

## 4. Worked example: inserts and splits

### The insertion algorithm

1. Descend from the root to the leaf where the key belongs. At each internal node, choose the child whose range contains the key.
2. Insert the key into the leaf in sorted position.
3. If the leaf now has too many keys, **split** it into two leaves. **Copy** the first key of the new right leaf up into the parent as a separator.
4. If the parent now has too many keys, split it as well. For an internal split, the middle key is **moved** up (not copied), because internal keys are only routing information.
5. If the root splits, create a new root above it. This is the **only** way the tree gets taller, and it is why all leaves always stay at the same depth.

### Insert 10, 20, 30

The tree is one leaf, which is also the root:

```
[10 20 30]
```

### Insert 40: first leaf split

The leaf would become `[10 20 30 40]`, four keys, which is over capacity. Split it into `[10 20]` and `[30 40]`. Copy the right leaf's first key, **30**, into a new root:

```
        [30]
       /    \
[10 20] → [30 40]
```

Searching for 30 now means: at the root, 30 ≥ 30, so go right. The value lives in the leaf. The 30 in the root is only a signpost.

### Insert 50, then 60

50 goes into the right leaf: `[30 40 50]`. Then 60 overflows it to `[30 40 50 60]`. Split it into `[30 40]` and `[50 60]`, and copy **50** up:

```
           [30 50]
          /   |   \
[10 20] → [30 40] → [50 60]
```

### Insert 70, then 80

70 fills `[50 60 70]`. 80 overflows it, so split into `[50 60]` and `[70 80]` and copy **70** up. The root now has three keys, which is still legal:

```
              [30 50 70]
          /    |     |     \
[10 20] → [30 40] → [50 60] → [70 80]
```

### Insert 90, then 100: cascading split and a taller tree

90 fills `[70 80 90]`. 100 overflows it, so split into `[70 80]` and `[90 100]` and copy **90** up. The root would become `[30 50 70 90]`, four keys, which is too many for an internal node.

Split the root. Keep `[30 50]` on the left and `[90]` on the right, and **move** the middle key **70** up into a brand-new root. Note that 70 is not duplicated at internal level. It now lives only in the new root, and the leaf `[70 80]` still holds the real entry.

```
                         [70]
                 /                 \
           [30 50]                 [90]
         /    |    \              /    \
[10 20]→[30 40]→[50 60]  →  [70 80]→[90 100]
```

The tree grew from the top. Height went from 2 to 3, and every leaf is still at the same depth.

### Insert 25, then 27: a split that stops early

25 belongs in `[10 20]` (25 < 30), which becomes `[10 20 25]`. Then 27 overflows it to `[10 20 25 27]`. Split it into `[10 20]` and `[25 27]` and copy **25** into the parent `[30 50]`, which becomes `[25 30 50]`. That still fits, so the split stops there:

```
                              [70]
                    /                       \
           [25 30 50]                       [90]
        /    |    |    \                   /    \
[10 20]→[25 27]→[30 40]→[50 60]  →  [70 80]→[90 100]
```

### Insert 45

45 goes into `[30 40]`, which becomes `[30 40 45]`. No split. This is the common case: on a real page with hundreds of slots, the vast majority of inserts just slide a key into a leaf.

Current tree, used for the delete examples:

```
                                  [70]
                       /                         \
             [25 30 50]                          [90]
          /    |     |      \                   /    \
[10 20]→[25 27]→[30 40 45]→[50 60]  →  [70 80]→[90 100]
   A       B        C         D           E        F
```

---

## 5. Worked example: deletes

Deletion is insertion in reverse, with one extra choice. When a node falls below its minimum, it can either **borrow** from a sibling that has spare keys (a redistribution), or **merge** with a sibling when neither has keys to spare.

### Delete 27: borrow from a sibling

Leaf B `[25 27]` becomes `[25]`, one key, which is below the minimum of 2. Look at the siblings:

- Left sibling A `[10 20]` is exactly at the minimum and cannot lend.
- Right sibling C `[30 40 45]` has 3 keys and can lend one.

Move C's smallest key, **30**, into B. B becomes `[25 30]` and C becomes `[40 45]`. The separator between B and C in the parent was 30. It must now be C's new first key, **40**. The parent becomes `[25 40 50]`:

```
             [25 40 50]
          /    |    |     \
[10 20]→[25 30]→[40 45]→[50 60] → ...
```

Only the parent separator changed. No structure was added or removed.

### Delete 20: merge two leaves

Leaf A `[10 20]` becomes `[10]`, which is under the minimum. A has no left sibling, and its right sibling B `[25 30]` is at the minimum and cannot lend. So **merge** them: A + B = `[10 25 30]`, three keys, which fits. Remove the separator between them, **25**, from the parent, and remove B's page (it goes back on the free list). The parent becomes `[40 50]`:

```
                          [70]
                /                     \
          [40 50]                     [90]
        /    |    \                  /    \
[10 25 30]→[40 45]→[50 60]  →  [70 80]→[90 100]
     A'        C       D           E        F
```

Note that 25 is no longer an internal key, but it is still a value in leaf A'.

### Delete 100: a merge that cascades upward

Leaf F `[90 100]` becomes `[90]`, which underflows. Its only sibling, E `[70 80]`, is at the minimum. Merge: E + F = `[70 80 90]`. Remove separator 90 from the parent. That parent was `[90]`, so it now has **zero keys and one child**, which is an internal underflow.

Fix the internal node the same way: look at its sibling. The left internal sibling `[40 50]` has 2 keys, one more than the minimum, so it can lend. For internal nodes, borrowing is a **rotation through the parent**:

1. The root separator **70** comes down into the underflowing node, which becomes `[70]`.
2. The left sibling's last key, **50**, goes up into the root, replacing 70.
3. The left sibling's last child, leaf D `[50 60]`, moves over to become the first child of the underflowing node.

```
                       [50]
                /                \
          [40]                   [70]
        /      \               /      \
[10 25 30]→[40 45]  →  [50 60]→[70 80 90]
```

Check the invariants. Everything left of root key 50 (10, 25, 30, 40, 45) is less than 50, and everything right of it (50, 60, 70, 80, 90) is at least 50. Every node meets its minimum, and every leaf is at depth 3.

If the left internal sibling had also been at the minimum, the two internal nodes would have **merged**, pulling the root's separator down between them. That would leave the root with zero keys and a single child. The root is then discarded and the merged node becomes the new root. This is the only way a B+tree gets **shorter**, again from the top, so balance is never broken.

### Deletes in practice

Many production systems are lazier than the textbook. PostgreSQL, for example, does not merge underfull leaves on delete. It removes only pages that become completely empty, and it does that during `VACUUM`. InnoDB merges pages when they fall below a threshold (`MERGE_THRESHOLD`, 50% by default). Systems tolerate sparse pages because aggressive merging causes "thrashing", where a page splits and merges repeatedly as a workload hovers around the boundary.

---

## 6. Worked example: a range scan

Query: `SELECT key FROM t WHERE key BETWEEN 28 AND 75`, using the tree from the end of Section 5.

**Step 1: descend to the lower bound.**
- Root `[50]`: 28 < 50, so take the left child. *(page read 1)*
- Internal `[40]`: 28 < 40, so take the left child. *(page read 2)*
- Leaf `[10 25 30]`: binary-search for the first key ≥ 28, which is **30**. *(page read 3)*

**Step 2: walk the leaf chain.**
- Leaf `[10 25 30]`: emit **30**. End of leaf, so follow the sibling link.
- Leaf `[40 45]`: emit **40, 45**. *(page read 4)*
- Leaf `[50 60]`: emit **50, 60**. *(page read 5)* Note that the scan moves between subtrees without going back through the root.
- Leaf `[70 80 90]`: emit **70**. Next is 80 > 75, so **stop**. *(page read 6)*

Result: `30, 40, 45, 50, 60, 70`, read in sorted order with 6 page reads: 3 for the descent and 3 more leaves. That is the B+tree range cost: **O(log N) to find the start, then one page per leaf of results**. The output is already sorted, which is why an index can satisfy `ORDER BY` with no sort step.

The same mechanism handles `key > 28` (scan to the end of the chain), `key < 75` (start at the leftmost leaf), `ORDER BY key DESC` (walk the chain backward), and `LIKE 'abc%'` on strings (a range from `'abc'` up to the next prefix).

---

## 7. Page layout: what a node looks like on disk

A node is a page, typically **8 KB** in PostgreSQL and SQL Server and **16 KB** in InnoDB. Most engines use some form of the **slotted page** layout:

```
+--------------------------------------------------------------+
| Page header                                                  |
|  - page LSN (last WAL record that changed this page)         |
|  - checksum, flags, page type (leaf / internal)              |
|  - level in the tree (0 = leaf)                              |
|  - left-sibling and right-sibling page numbers               |
|  - number of slots, free-space start/end offsets             |
+--------------------------------------------------------------+
| Slot array:  [off 1][off 2][off 3] ... →        (grows down) |
|                                                              |
|                    ... free space ...                        |
|                                                              |
|           (grows up) ← [cell 3][cell 2][cell 1]              |
+--------------------------------------------------------------+
| Special area (e.g. high key in PostgreSQL B-link pages)      |
+--------------------------------------------------------------+
```

- The **slot array** is a sorted array of small fixed-size offsets, each pointing to a **cell**. Binary search runs over the slots, so inserting a key in the middle shifts only 2-byte offsets, not the variable-length cells themselves.
- A **leaf cell** holds `(key, value)`. The value is either a row pointer (PostgreSQL's 6-byte TID: block number plus line offset) or, in a clustered index, the whole row.
- An **internal cell** holds `(separator key, child page number)`.
- Cells are appended from the end of the page toward the slot array. When the two meet, the page is full and must split. Deleted cells leave holes that are reclaimed by compacting the page.

### Fanout arithmetic

Take an index on a `BIGINT` (8 bytes) in an 8 KB page. An internal entry is roughly 8 bytes of key, a 4–6 byte child pointer, a few bytes of tuple header, and a 2–4 byte slot: about **20 bytes**. After the page header, that gives roughly **400 entries per internal page**.

| Levels | Approx. leaf entries reachable |
|-------:|-------------------------------:|
| 1      | 400 |
| 2      | 160,000 |
| 3      | 64,000,000 |
| 4      | 25,600,000,000 |

A table with **a billion rows needs a 4-level index**. The root and level 2 together are about 400 pages (around 3 MB) and are always cached. A cold point lookup therefore costs about **one or two physical reads**.

### Techniques that raise fanout

- **Suffix truncation:** a separator only has to distinguish two children, so `"Jon"` can separate `"Johnson"` from `"Jones"`.
- **Prefix compression:** a shared prefix on a page is stored once.
- **Deduplication:** PostgreSQL 13+ stores a repeated key once with a list of TIDs.
- **Fill factor:** leaving free space on purpose (`FILLFACTOR = 90`) delays splits from later mid-range inserts.

---

## 8. Complexity

Let *N* be the number of keys, *B* the number of entries per page (the fanout), and *K* the number of results.

| Operation | Page I/Os | Notes |
|---|---|---|
| Point search | O(log_B N) | Height is ⌈log_B N⌉, which is 3–4 in practice |
| Insert | O(log_B N) | Descent plus, rarely, splits along one root-to-leaf path |
| Delete | O(log_B N) | Descent plus, rarely, merges or borrows along one path |
| Range scan | O(log_B N + K/B) | Descent plus one page per *B* results |
| Full ordered scan | O(N/B) | Walk the leaf chain |
| Space | O(N/B) pages | Each node ≥ 50% full; random inserts settle near ~69% |
| Bulk load (sorted) | O(N/B) | Build leaves left to right, then build upper levels |

Inside a node, binary search costs O(log B) CPU comparisons, so the total CPU cost of a lookup is O(log B · log_B N) = O(log N). In the end, comparisons are still logarithmic in N. The B-tree's advantage is that the **I/O** count is log *base B*, and B is in the hundreds.

**Splits are rare and amortized O(1).** A leaf with capacity B splits only once every ~B/2 inserts into it, and internal splits are rarer by another factor of B. Most inserts touch exactly one leaf page.

The ~69% figure (ln 2) is the expected steady-state occupancy under random inserts with 50/50 splits. Sequential inserts would leave every page half full, so engines detect a split at the rightmost leaf and split *unevenly*, leaving the old page nearly full. PostgreSQL and InnoDB both do this.

---

## 9. How databases use B+trees

### Clustered vs. secondary indexes

**Clustered (index-organized) tables.** In InnoDB and in SQL Server tables with a clustered index, the table *is* a B+tree keyed by the primary key, and the leaves hold the full rows. A primary-key lookup ends at the row itself. A **secondary index** stores `(indexed columns, primary key)` in its leaves, so finding a row by a secondary key means two tree descents: first the secondary index to get the primary key, then the clustered index to get the row. This is why a wide primary key in InnoDB inflates every secondary index.

**Heap tables.** In PostgreSQL (and in SQL Server heaps), rows sit unordered in a heap file, and *every* index, including the primary key, is a secondary B+tree whose leaves store TIDs. A lookup is one descent plus one heap-page fetch. Updates that move a row can mean updating every index, which PostgreSQL softens with HOT (heap-only tuple) updates.

### Composite indexes and the leftmost-prefix rule

An index on `(last_name, first_name, birth_date)` sorts entries by `last_name`, then by `first_name` within each last name, and so on. Because of that ordering, it can serve:

- `WHERE last_name = 'Ng'`, which is one contiguous range.
- `WHERE last_name = 'Ng' AND first_name = 'Ada'`, a narrower range.
- `WHERE last_name = 'Ng' AND first_name > 'M'`, a range inside a range.
- `WHERE last_name = 'Ng' ORDER BY first_name`, with no sort needed.

It **cannot** efficiently serve `WHERE first_name = 'Ada'` alone, because matching entries are scattered across every last name. (Some engines can do a "skip scan" that jumps between distinct leading values, which helps only when the leading column has few distinct values.) As a rule of thumb, put equality columns first and the range or sort column last.

### Covering and index-only scans

If every column a query needs is in the index, the engine can answer from the leaves alone, without visiting the table. In SQL Server and PostgreSQL 11+ you can add non-key payload columns with `INCLUDE (...)` for exactly this purpose. PostgreSQL also consults a **visibility map** to confirm that a heap page has no rows that are invisible to some transactions. Only then can it skip the heap fetch.

### Other uses

`UNIQUE` and `PRIMARY KEY` checks, `MIN`/`MAX` (the leftmost and rightmost leaf entries), and merge joins all rely on the same sorted structure. Indexing foreign-key columns turns "does any child row reference this parent?" from a table scan into a lookup.

### Concurrency and crash safety

Engines protect pages with short-lived **latches**. In **latch crabbing**, a writer holds a parent's latch until the child is latched and known to be "safe" (it cannot split or merge). PostgreSQL uses the **B-link tree** (Lehman and Yao, 1981): every page has a **high key** and a **right-link**, so a reader that lands on a page split under it just follows the right-link instead of blocking. Because a split touches several pages, it is written to the **write-ahead log** as one atomic action, and recovery completes or replays it after a crash.

### Key choice and insert patterns

- **Monotonic keys** (auto-increment IDs, timestamps) always insert at the rightmost leaf. Locality and page fill are excellent, but that one page can become a contention hot spot.
- **Random keys** (UUIDv4, hashes) scatter inserts across every leaf, causing splits everywhere, ~69% fill, and a working set equal to the whole index. Time-ordered IDs (UUIDv7, ULID) avoid this while staying globally unique.
- **Bulk loading.** `CREATE INDEX` sorts first and builds leaves left to right at the target fill factor, which is much faster than N inserts and produces a compact tree.

### Maintenance

Deletes and random-order splits leave pages sparse and scattered (**bloat** and **fragmentation**). `REINDEX`, `ALTER INDEX ... REBUILD` and `OPTIMIZE TABLE` rebuild the index compactly. Run them when monitoring shows real bloat, not as a ritual.

---

## 10. When a B+tree is not the right tool

- **Equality-only lookups:** a **hash index** is O(1) but cannot do ranges or ordering. At height 3–4 the difference is usually small.
- **Write-heavy ingestion:** **LSM-trees** (RocksDB, Cassandra) turn random writes into sequential flushes, at the cost of read amplification and compaction.
- **Analytics, geometry, full text:** bitmap and columnar indexes, GiST/R-trees, GIN/inverted indexes, and BRIN each beat a B-tree in their own niche.
- **Low selectivity:** when a predicate matches a large share of the table, a sequential scan is cheaper than many random lookups. That is why optimizers sometimes ignore your index.

---

## 11. Summary

A B+tree is a sorted, height-balanced, very wide tree of pages. Inserts split full nodes and grow the tree at the root. Deletes borrow or merge and shrink it at the root. Range scans descend once and walk the leaf chain. With fanout in the hundreds, billions of rows fit in 3–4 levels, so lookups cost O(log_B N) page reads and range scans O(log_B N + K/B). Primary keys, secondary and covering indexes, uniqueness checks and sort avoidance all rest on this one structure, and your choices of clustering, column order, key pattern and fill factor decide how well it performs.
