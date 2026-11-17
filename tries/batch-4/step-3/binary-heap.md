# Binary Heaps and Priority Queues

A **priority queue** is an abstract data type that holds items with priorities and always lets you get at the "most urgent" one next. A **binary heap** is the most common way to implement it: a complete binary tree stored in a plain array, with no pointers, that supports insertion and removal of the minimum in O(log n) time and construction from n items in O(n) time.

This explainer covers the interface, the invariant, the array layout, sift-up and sift-down traced on concrete keys, linear-time construction with a proof, heapsort, decrease-key, and complete Python code.

---

## 1. The priority queue interface

A min-priority queue supports these operations:

| Operation | Meaning | Binary heap cost |
|---|---|---|
| `push(x)` | insert an item | O(log n) |
| `peek()` | look at the smallest item | O(1) |
| `pop()` | remove and return the smallest item | O(log n) |
| `heapify(list)` | build a queue from n items at once | O(n) |
| `decrease_key(x, k)` | lower the priority of an item already inside | O(log n) |
| `len()` | number of items | O(1) |

A max-priority queue is the mirror image: flip every comparison. Everything below uses a **min-heap** unless stated otherwise. Heapsort, in section 6, switches to a max-heap because that makes the in-place ascending sort work.

A sorted array gives O(1) access to the minimum but O(n) insertion; an unsorted array is the reverse. A balanced search tree gives O(log n) for everything but needs pointers and rebalancing. The heap keeps only as much order as it needs to answer "what is the minimum?", and that partial order is cheap to maintain.

---

## 2. The heap invariant

A binary min-heap is a binary tree with two properties:

1. **Shape property.** The tree is *complete*: every level is full except possibly the last, and the last level is filled from left to right with no gaps.
2. **Heap-order property.** Every node's key is less than or equal to the keys of its children.

The heap-order property says nothing about siblings or cousins. The left child may be bigger or smaller than the right child. The only guarantee is along each root-to-leaf path, where keys never decrease. That means the minimum is always at the root, and that is all the queue needs.

The shape property keeps the tree balanced. A complete tree with n nodes has height ⌊log₂ n⌋, so any path from root to leaf has at most about log₂ n edges. Every operation that walks a single path is therefore O(log n).

---

## 3. Array layout

Because the tree is complete, there is no need for pointers. Number the nodes level by level, left to right, starting at 0, and store node *i* in `a[i]`. Then for any index *i*:

- `parent(i) = (i - 1) // 2`
- `left(i)   = 2*i + 1`
- `right(i)  = 2*i + 2`

(Some textbooks use 1-based indexing, where the formulas become `i // 2`, `2i`, and `2i + 1`. The two conventions are equivalent; this document uses 0-based indexing to match most programming languages.)

Here is a concrete min-heap with ten keys:

```
index:  0  1  2  3  4  5  6  7   8   9
a    = [2, 5, 3, 9, 6, 4, 8, 12, 10, 7]
```

Drawn as a tree:

```
                 2
             /       \
           5           3
         /   \       /   \
        9     6     4     8
       / \   /
     12  10 7
```

Check the formulas on a few nodes. Node 1 (key 5) has children at 2·1+1 = 3 (key 9) and 2·1+2 = 4 (key 6), and its parent is (1−1)//2 = 0 (key 2). Node 4 (key 6) has a left child at 9 (key 7) and a right child at 10, which is out of range because n = 10, so it has only one child. Node 9 (key 7) has parent (9−1)//2 = 4 (key 6).

Verify heap order: 2 ≤ 5, 3; 5 ≤ 9, 6; 3 ≤ 4, 8; 9 ≤ 12, 10; 6 ≤ 7. It is a valid min-heap, even though 4 (depth 2) is smaller than 5 (depth 1): order across different branches is irrelevant.

Some useful facts follow from the layout:

- The leaves are exactly the indices from `n // 2` to `n - 1`. In the example, indices 5 through 9 are leaves.
- The last internal (non-leaf) node is at `n // 2 - 1`, which is index 4 here.
- Appending to the end of the array adds a node at the next free spot in the bottom level, which keeps the tree complete. Removing the last element likewise keeps it complete. This is why every heap operation does its structural change at the end of the array and then repairs order.

---

## 4. Sift-up: restoring order after an insertion

To insert a key, append it to the end of the array. The shape is still complete, but the new key may be smaller than its parent, which breaks heap order on exactly one edge. **Sift-up** (also called *bubble-up*, *percolate-up*, or *swim*) fixes this. While the key is smaller than its parent, swap it with the parent and move up. Stop when the parent is smaller or equal, or when the key reaches the root.

### Trace: insert 1 into the example heap

Start:

```
index:  0  1  2  3  4  5  6  7   8   9
a    = [2, 5, 3, 9, 6, 4, 8, 12, 10, 7]
```

**Step 0.** Append 1 at index 10.

```
a = [2, 5, 3, 9, 6, 4, 8, 12, 10, 7, 1]
```

**Step 1.** The parent of 10 is (10−1)//2 = 4, with key 6. Since 1 < 6, swap.

```
a = [2, 5, 3, 9, 1, 4, 8, 12, 10, 7, 6]     (1 now at index 4)
```

**Step 2.** The parent of 4 is (4−1)//2 = 1, with key 5. Since 1 < 5, swap.

```
a = [2, 1, 3, 9, 5, 4, 8, 12, 10, 7, 6]     (1 now at index 1)
```

**Step 3.** The parent of 1 is 0, with key 2. Since 1 < 2, swap.

```
a = [1, 2, 3, 9, 5, 4, 8, 12, 10, 7, 6]     (1 now at index 0, the root)
```

**Step 4.** Index 0 has no parent, so stop.

The resulting tree:

```
                 1
             /       \
           2           3
         /   \       /   \
        9     5     4     8
       / \   / \
     12  10 7   6
```

Why is this correct? The demoted parent was already at most every key in its subtree, so it is still at most its new children. The only edge that can be wrong is the one between the new key and its new parent, which the next iteration checks. The loop runs at most once per level, so sift-up is O(log n).

---

## 5. Sift-down: restoring order after removing the root

To pop the minimum, take `a[0]`. That leaves a hole at the root. The standard trick is to move the **last** element into the root and shrink the array by one. The shape stays complete, but the new root is probably too large. **Sift-down** (also *bubble-down*, *percolate-down*, or *sink*) fixes this. While the key is larger than its smaller child, swap it with that smaller child and move down. Stop when the key is at most both children, or when it reaches a leaf.

The detail that matters is to swap with the **smaller** child. If you swapped with the larger child, that larger child would become the parent of the smaller one, and heap order would break immediately.

### Trace: pop from the heap produced in section 4

Start:

```
index:  0  1  2  3  4  5  6  7   8   9  10
a    = [1, 2, 3, 9, 5, 4, 8, 12, 10, 7, 6]
```

**Step 0.** Save the result, 1. Move the last element (6, at index 10) to the root and shrink to n = 10.

```
a = [6, 2, 3, 9, 5, 4, 8, 12, 10, 7]
```

**Step 1.** At index 0 (key 6), the children are index 1 (key 2) and index 2 (key 3). The smaller child is 2, at index 1. Since 6 > 2, swap.

```
a = [2, 6, 3, 9, 5, 4, 8, 12, 10, 7]     (6 now at index 1)
```

**Step 2.** At index 1 (key 6), the children are index 3 (key 9) and index 4 (key 5). The smaller child is 5, at index 4. Since 6 > 5, swap.

```
a = [2, 5, 3, 9, 6, 4, 8, 12, 10, 7]     (6 now at index 4)
```

**Step 3.** At index 4 (key 6), the left child is index 9 (key 7) and the right child is index 10, which is out of range. The smaller (only) child is 7. Since 6 ≤ 7, stop.

Return 1. The array is back to exactly the heap from section 3, which makes sense: we inserted 1 and then removed it, and the replacement element 6 found its way back to its original position.

Correctness mirrors sift-up: the promoted child is the smaller one, so it is at most its sibling and everything below. Only the edges from the sinking key to its new children can be wrong, and the next iteration checks them. Sift-down follows one path, so it is O(log n), at up to two comparisons per level.

---

## 6. Building a heap in O(n): bottom-up heapify

Given n unordered keys, the obvious approach is n pushes, which costs O(n log n) in the worst case. For example, inserting keys in decreasing order makes every new key sift all the way to the root. There is a better way, due to Floyd (1964).

**Bottom-up heapify:** treat the array as a complete tree that is already in the right shape. Every leaf is trivially a one-element heap. Then, for each internal node from the last one (`n // 2 - 1`) back to the root, run sift-down on it. When you sift down node *i*, both of its subtrees are already valid heaps, because their roots have larger indices and were processed earlier. Sift-down merges "a root plus two heaps" into one heap, so when the loop reaches index 0, the whole array is a heap.

### Trace: heapify [9, 4, 7, 1, 8, 2, 6, 3, 5]

```
index:  0  1  2  3  4  5  6  7  8
a    = [9, 4, 7, 1, 8, 2, 6, 3, 5]      n = 9, last internal node = 9//2 - 1 = 3
```

As a tree:

```
                 9
             /       \
           4           7
         /   \       /   \
        1     8     2     6
       / \
      3   5
```

**i = 3 (key 1).** The children are index 7 (key 3) and index 8 (key 5). Since 1 ≤ 3, there is no swap. Zero swaps.

**i = 2 (key 7).** The children are index 5 (key 2) and index 6 (key 6). The smaller child is 2. Since 7 > 2, swap.

```
a = [9, 4, 2, 1, 8, 7, 6, 3, 5]
```

Index 5 has children at 11 and 12, both out of range, so stop. One swap.

**i = 1 (key 4).** The children are index 3 (key 1) and index 4 (key 8). The smaller child is 1. Since 4 > 1, swap.

```
a = [9, 1, 2, 4, 8, 7, 6, 3, 5]
```

Now 4 is at index 3, with children index 7 (key 3) and index 8 (key 5). The smaller child is 3. Since 4 > 3, swap.

```
a = [9, 1, 2, 3, 8, 7, 6, 4, 5]
```

Index 7 is a leaf, so stop. Two swaps.

**i = 0 (key 9).** The children are index 1 (key 1) and index 2 (key 2). The smaller child is 1. Swap.

```
a = [1, 9, 2, 3, 8, 7, 6, 4, 5]
```

Now 9 is at index 1, with children index 3 (key 3) and index 4 (key 8). The smaller child is 3. Swap.

```
a = [1, 3, 2, 9, 8, 7, 6, 4, 5]
```

Now 9 is at index 3, with children index 7 (key 4) and index 8 (key 5). The smaller child is 4. Swap.

```
a = [1, 3, 2, 4, 8, 7, 6, 9, 5]
```

Index 7 is a leaf, so stop. Three swaps.

Final heap, with 0 + 1 + 2 + 3 = 6 swaps in total:

```
                 1
             /       \
           3           2
         /   \       /   \
        4     8     7     6
       / \
      9   5
```

Check heap order: 1 ≤ 3, 2; 3 ≤ 4, 8; 2 ≤ 7, 6; 4 ≤ 9, 5. It is valid.

### Proof that bottom-up heapify is O(n)

Define the **height** of a node as the number of edges on the longest path from it down to a leaf. Leaves have height 0. Sift-down on a node of height *h* does at most *h* swaps, at a cost of at most 2 comparisons each, so the total work is O(sum of heights of all nodes).

In a complete binary tree with n nodes, the number of nodes of height *h* is at most ⌈n / 2^(h+1)⌉. Intuitively, about half the nodes are leaves, a quarter have height 1, an eighth have height 2, and so on. So the total work is at most

```
  ⌊log n⌋
    Σ     ⌈n / 2^(h+1)⌉ · h   ≤   n · Σ (h / 2^h)  over h ≥ 0   ·  (constant)
   h=0
```

The key fact is that the series Σ h/2^h converges. Start from the geometric series Σ x^h = 1/(1−x) for |x| < 1. Differentiate and multiply by x to get Σ h·x^h = x/(1−x)². At x = 1/2 this gives (1/2)/(1/4) = 2. So the total work is at most about n · 2 / 2 · O(1) = O(n).

There is also a neat exact bound. For a *perfect* binary tree of height H, with n = 2^(H+1) − 1 nodes, the sum of heights is exactly n − (H + 1). A charging argument shows the sum of heights is at most n − 1 for any complete tree: for each node *v*, step to its right child once, then follow left children to a leaf. Those paths never share an edge, so the total is bounded by the number of edges, n − 1.

Check this against the trace. The nine-node tree has these heights: node 0 has height 3 (the path 0 → 1 → 3 → 7), node 1 has height 2, nodes 2 and 3 have height 1, and the five leaves have height 0. The sum is 3 + 2 + 1 + 1 = 7 ≤ n − 1 = 8. The trace did 6 swaps, within the bound.

**Why is this cheaper than n pushes?** Repeated insertion can pay log n for the many nodes near the *bottom*; bottom-up heapify pays a lot only for the few nodes near the *top*. Note that heapify does not sort: extracting everything in order still costs Θ(n log n), as the comparison-sorting lower bound requires.

---

## 7. Heapsort

Heapsort turns the heap into an in-place, O(n log n), comparison sort that uses O(1) extra memory. To sort ascending in place, use a **max-heap**:

1. Build a max-heap from the array using bottom-up heapify. The largest key is now at index 0.
2. Repeat for `end = n−1` down to `1`: swap `a[0]` (the current max) with `a[end]`, so the max lands in its final sorted position. Then treat the heap as having size `end` and sift the new root down within that prefix.

The array divides into a shrinking heap on the left and a growing sorted region on the right.

### Trace: heapsort [4, 10, 3, 5, 1]

**Build the max-heap** (n = 5, last internal node = 1). Max-heap sift-down swaps with the *larger* child when the parent is smaller.

- i = 1 (key 10): the children are 5 and 1. Since 10 ≥ 5, there is no swap.
- i = 0 (key 4): the children are 10 and 3. The larger child is 10, and 4 < 10, so swap, giving `[10, 4, 3, 5, 1]`. Now 4 is at index 1, with children 5 (index 3) and 1 (index 4). The larger child is 5, and 4 < 5, so swap, giving `[10, 5, 3, 4, 1]`. Index 3 is a leaf, so stop.

```
max-heap: [10, 5, 3, 4, 1]

          10
         /  \
        5    3
       / \
      4   1
```

**Extraction phase** (the `|` separates the heap from the sorted suffix):

| Step | Swap root with `a[end]` | Sift-down inside heap | Array after |
|---|---|---|---|
| end = 4 | swap 10 ↔ 1 → `[1, 5, 3, 4 \| 10]` | 1 < max(5, 3) = 5, so swap → `[5, 1, 3, 4]`; 1 < 4 (index 3, only child), so swap → `[5, 4, 3, 1]` | `[5, 4, 3, 1 \| 10]` |
| end = 3 | swap 5 ↔ 1 → `[1, 4, 3 \| 5, 10]` | 1 < max(4, 3) = 4, so swap → `[4, 1, 3]`; index 1 has no child within size 3 | `[4, 1, 3 \| 5, 10]` |
| end = 2 | swap 4 ↔ 3 → `[3, 1 \| 4, 5, 10]` | 3 ≥ 1, so stop | `[3, 1 \| 4, 5, 10]` |
| end = 1 | swap 3 ↔ 1 → `[1 \| 3, 4, 5, 10]` | heap of size 1, nothing to do | `[1, 3, 4, 5, 10]` |

The result is `[1, 3, 4, 5, 10]`, which is sorted.

**Properties of heapsort:**

- **Time.** The build phase costs O(n). Then n − 1 sift-downs each cost O(log n), so the total is O(n log n) in the worst, average, and best cases (for distinct keys).
- **Space.** O(1) extra, because the sort is fully in place.
- **Not stable.** Equal keys can be reordered by the long-distance swaps.
- **In practice** it is usually slower than quicksort because sift-down has poor cache locality. Its value is the guaranteed worst case, which is why introsort (C++ `std::sort`) falls back to heapsort when quicksort recurses too deeply.

---

## 8. Decrease-key and the position map

Many graph algorithms need to *lower the priority of an item that is already in the queue*. Dijkstra's shortest-path algorithm lowers a vertex's tentative distance when it finds a shorter route, and Prim's minimum spanning tree algorithm lowers a vertex's connection cost. This is **decrease-key**.

Changing the key is easy once you know where the item is. In a min-heap, a smaller key can only violate order with its *parent*, never with its children, since it was already at most its children and got smaller. So decrease-key is: overwrite the key, then sift-up. That is O(log n).

The hard part is **finding the item**. A heap is not searchable; looking for an arbitrary item is O(n). The fix is an **indexed priority queue**: alongside the array, keep a dictionary `pos` that maps each item to its current index in the array. Every swap must update `pos` for both items it moves. Then decrease-key is a dictionary lookup followed by sift-up.

(Increase-key is symmetric: overwrite the key, then sift-down. Arbitrary delete can be done by decreasing the key to −∞ and popping, or by swapping the item with the last element, shrinking, and then sifting the moved element up *or* down, whichever is needed.)

### Trace: decrease-key on the section 3 heap

Attach item names so we can see the position map work. Here each slot shows `item:key`:

```
index:  0    1    2    3    4    5    6    7     8     9
a    = [A:2, B:5, C:3, D:9, E:6, F:4, G:8, H:12, I:10, J:7]
pos  = {A:0, B:1, C:2, D:3, E:4, F:5, G:6, H:7, I:8, J:9}
```

Call `decrease_key(I, 4)`.

**Step 0.** Look up `pos[I] = 8`. Set the key to 4: `a[8] = I:4`.

**Step 1.** The parent of 8 is (8−1)//2 = 3, holding `D:9`. Since 4 < 9, swap. Now `a[3] = I:4` and `a[8] = D:9`. Update `pos[I] = 3` and `pos[D] = 8`.

**Step 2.** The parent of 3 is 1, holding `B:5`. Since 4 < 5, swap. Now `a[1] = I:4` and `a[3] = B:5`. Update `pos[I] = 1` and `pos[B] = 3`.

**Step 3.** The parent of 1 is 0, holding `A:2`. Since 4 ≥ 2, stop.

Result:

```
index:  0    1    2    3    4    5    6    7     8    9
a    = [A:2, I:4, C:3, B:5, E:6, F:4, G:8, H:12, D:9, J:7]
pos  = {A:0, I:1, C:2, B:3, E:4, F:5, G:6, H:7, D:8, J:9}
```

```
                   A:2
               /         \
            I:4           C:3
           /    \        /   \
         B:5    E:6    F:4   G:8
        /   \   /
     H:12  D:9 J:7
```

Heap order holds: 2 ≤ 4, 3; 4 ≤ 5, 6; 3 ≤ 4, 8; 5 ≤ 12, 9; 6 ≤ 7. The tie between `I:4` and `F:4` is harmless, because heap order uses ≤ and nodes in different branches are never compared. Every item's `pos` entry matches its array index.

**A common alternative: lazy deletion.** Python's `heapq` has no decrease-key, so Dijkstra implementations often push a fresh `(new_key, item)` and skip stale entries when popped. The heap may hold O(E) entries, but O(log E) = O(log V), so the bound is unchanged.

---

## 9. Code

The following Python implementation is an indexed min-heap with push, pop, peek, decrease-key, bottom-up construction, and membership tests. It stores `(key, item)` pairs and keeps a position map. Items must be hashable and unique.

```python
class IndexedMinHeap:
    """Binary min-heap with a position map, supporting decrease_key in O(log n)."""

    def __init__(self, pairs=()):
        # pairs: iterable of (key, item)
        self._a = [(k, x) for k, x in pairs]
        self._pos = {}
        for i, (_, x) in enumerate(self._a):
            if x in self._pos:
                raise ValueError(f"duplicate item {x!r}")
            self._pos[x] = i
        self._heapify()

    # ---- public API -------------------------------------------------------

    def __len__(self):
        return len(self._a)

    def __contains__(self, item):
        return item in self._pos

    def peek(self):
        if not self._a:
            raise IndexError("peek from empty heap")
        return self._a[0]

    def push(self, key, item):
        if item in self._pos:
            raise ValueError(f"item {item!r} already present")
        self._a.append((key, item))
        self._pos[item] = len(self._a) - 1
        self._sift_up(len(self._a) - 1)

    def pop(self):
        if not self._a:
            raise IndexError("pop from empty heap")
        top = self._a[0]
        last = self._a.pop()
        del self._pos[top[1]]
        if self._a:                      # heap was not of size 1
            self._a[0] = last
            self._pos[last[1]] = 0
            self._sift_down(0)
        return top

    def key_of(self, item):
        return self._a[self._pos[item]][0]

    def decrease_key(self, item, new_key):
        i = self._pos[item]              # KeyError if absent
        if new_key > self._a[i][0]:
            raise ValueError("new key is larger than current key")
        self._a[i] = (new_key, item)
        self._sift_up(i)

    # ---- internals --------------------------------------------------------

    def _swap(self, i, j):
        a = self._a
        a[i], a[j] = a[j], a[i]
        self._pos[a[i][1]] = i
        self._pos[a[j][1]] = j

    def _sift_up(self, i):
        a = self._a
        while i > 0:
            p = (i - 1) // 2
            if a[i][0] < a[p][0]:
                self._swap(i, p)
                i = p
            else:
                break

    def _sift_down(self, i):
        a, n = self._a, len(self._a)
        while True:
            l = 2 * i + 1
            if l >= n:
                break                    # i is a leaf
            r = l + 1
            c = r if r < n and a[r][0] < a[l][0] else l   # smaller child
            if a[c][0] < a[i][0]:
                self._swap(i, c)
                i = c
            else:
                break

    def _heapify(self):
        for i in range(len(self._a) // 2 - 1, -1, -1):
            self._sift_down(i)
```

Comparisons use only the key (`a[i][0]`), so items never need to be comparable to each other. This avoids a classic bug with tuple-based heaps, where two equal keys cause Python to compare the items and raise a `TypeError`.

### In-place heapsort

```python
def heapsort(a):
    """Sort list a ascending, in place, using a max-heap. O(n log n), O(1) extra."""
    n = len(a)

    def sift_down(i, size):
        while True:
            l = 2 * i + 1
            if l >= size:
                return
            r = l + 1
            c = r if r < size and a[r] > a[l] else l     # larger child
            if a[c] > a[i]:
                a[i], a[c] = a[c], a[i]
                i = c
            else:
                return

    for i in range(n // 2 - 1, -1, -1):   # build max-heap in O(n)
        sift_down(i, n)
    for end in range(n - 1, 0, -1):       # move max to the end, shrink heap
        a[0], a[end] = a[end], a[0]
        sift_down(0, end)
```

### Dijkstra with decrease-key

```python
def dijkstra(graph, source):
    """graph: {u: [(v, weight), ...]} with non-negative weights."""
    dist = {source: 0}
    pq = IndexedMinHeap([(0, source)])
    done = set()
    while pq:
        d, u = pq.pop()
        done.add(u)
        for v, w in graph.get(u, ()):
            if v in done:
                continue
            nd = d + w
            if v not in dist or nd < dist[v]:
                dist[v] = nd
                if v in pq:
                    pq.decrease_key(v, nd)
                else:
                    pq.push(nd, v)
    return dist
```

### Quick sanity checks

```python
h = IndexedMinHeap([(9,'a'),(4,'b'),(7,'c'),(1,'d'),(8,'e'),
                    (2,'f'),(6,'g'),(3,'h'),(5,'i')])
assert [k for k, _ in h._a] == [1, 3, 2, 4, 8, 7, 6, 9, 5]   # matches section 6
h.decrease_key('e', 0)
assert h.peek() == (0, 'e')
assert [h.pop()[0] for _ in range(len(h))] == [0, 1, 2, 3, 4, 5, 6, 7, 9]

xs = [4, 10, 3, 5, 1]
heapsort(xs)
assert xs == [1, 3, 4, 5, 10]                                # matches section 7
```

### Using the standard library

Python's `heapq` provides the same algorithms on a plain list (`heapify`, `heappush`, `heappop`, `nsmallest`, `merge`), with no decrease-key; negate keys for a max-heap. Java has `PriorityQueue`, C++ has `std::priority_queue` (a *max*-heap by default), and Go has `container/heap`.

---

## 10. Practical notes and variations

- **Sift with a hole.** Hold the moving key in a temporary and shift neighbours into the hole, writing the key once at the end. This halves memory writes.
- **d-ary heaps.** With d children per node the tree has log_d n levels: sift-up and decrease-key get cheaper, sift-down costs d−1 comparisons per level. d = 4 is a common, cache-friendly choice.
- **Top-k, k-way merge, running median.** A size-k min-heap keeps the k largest of a stream in O(n log k); a heap of k cursors merges k sorted lists in O(N log k); a max-heap plus a min-heap track a running median.
- **Other heaps.** For fast meld, use leftist, binomial, or pairing heaps. Fibonacci heaps give O(1) amortized decrease-key, but their constants rarely pay off.
- **Ties.** Heaps are not FIFO among equal keys. Store `(key, counter, item)` if you need that.

---

## Summary

A binary heap stores a complete binary tree in an array, using index arithmetic in place of pointers, and maintains a single invariant: every parent is at most its children. Each mutation makes one structural change at the end of the array and then repairs order along one root-to-leaf path. Insertion and decrease-key use sift-up; pop-min uses sift-down. Both are O(log n). Building a heap bottom-up costs O(n), because the sum of node heights in a complete tree is less than n. Heapsort reuses the same machinery to sort in place in O(n log n) with O(1) extra space. With a position map added, the heap becomes an indexed priority queue that supports decrease-key, which is what Dijkstra's and Prim's algorithms need.
