# Dijkstra's Shortest-Path Algorithm

Dijkstra's algorithm answers one of the most common questions in computing: *given a weighted graph and a starting vertex, what is the cheapest way to reach every other vertex?* Edsger W. Dijkstra designed it in 1956, reportedly in about twenty minutes while sitting at a café in Amsterdam, and published it in 1959. More than sixty years later it still sits underneath route planners, network routing protocols such as OSPF and IS-IS, game pathfinding (as the base of A\*), and countless scheduling and optimization tools.

This explainer covers the algorithm in full: the problem it solves, the core idea, a complete step-by-step trace with the priority queue shown at every iteration, a proof sketch of correctness, running time under different priority-queue implementations, a concrete example of how it fails with negative edges, and working code.

---

## 1. The Problem

We are given:

- A graph `G = (V, E)`, directed or undirected, with `n = |V|` vertices and `m = |E|` edges.
- A weight function `w(u, v)` on each edge, which must be **non-negative**: `w(u, v) ≥ 0`.
- A source vertex `s`.

The *length* (or cost) of a path is the sum of its edge weights. The **shortest-path distance** `δ(s, v)` is the minimum length over all paths from `s` to `v`, or `∞` if no path exists.

The single-source shortest-path (SSSP) problem asks for `δ(s, v)` for every vertex `v`, and usually also for the actual paths, which we represent compactly as a *shortest-path tree*: each vertex stores a `prev` pointer to its predecessor on a shortest path from `s`.

An undirected graph is handled by treating each undirected edge `{u, v}` as two directed edges `u → v` and `v → u` with the same weight. Everything below works for both.

---

## 2. The Core Idea

Dijkstra's algorithm is a *greedy* algorithm built on two ingredients.

**Ingredient 1: tentative distances and relaxation.** Each vertex `v` keeps a number `dist[v]`, an upper bound on its true distance. Initially `dist[s] = 0` and `dist[v] = ∞` for everyone else. The basic operation is **relaxing** an edge `u → v`:

```
relax(u, v):
    if dist[u] + w(u, v) < dist[v]:
        dist[v] = dist[u] + w(u, v)
        prev[v] = u
```

Relaxation asks: "Is going through `u` a better way to reach `v` than anything I've found so far?" It never makes `dist[v]` smaller than the true distance, because every value it writes is the length of an actual path.

**Ingredient 2: settle the closest unsettled vertex.** The algorithm maintains a set `S` of *settled* vertices whose distances are final. At each step it picks the unsettled vertex `u` with the smallest tentative distance, declares it settled, and relaxes all of `u`'s outgoing edges.

Why is this greedy choice safe? Intuitively, if `u` is the closest vertex not yet settled, any alternative route to `u` must leave the settled region at some vertex that is *already at least as far* as `u`, and since weights are non-negative, continuing along that route can only add cost. So nothing can ever beat `dist[u]`. Section 5 makes this rigorous.

A helpful mental picture is a wavefront: imagine pouring water in at `s`, with water flowing along edges at unit speed. Vertices get wet in order of their distance from `s`. Dijkstra's algorithm simulates this flood, skipping the boring time in between and jumping straight to the next vertex to get wet.

### Pseudocode

```
Dijkstra(G, w, s):
    for each vertex v in V:
        dist[v] = ∞
        prev[v] = nil
    dist[s] = 0
    Q = priority queue containing all vertices, keyed by dist
    while Q is not empty:
        u = Q.extract_min()            # u is now settled
        for each edge (u, v) with weight w:
            if dist[u] + w < dist[v]:
                dist[v] = dist[u] + w
                prev[v] = u
                Q.decrease_key(v, dist[v])
    return dist, prev
```

The priority queue is what makes "pick the closest unsettled vertex" fast. Many practical implementations do not use `decrease_key` at all; instead they push a fresh `(distance, vertex)` entry whenever a distance improves and simply ignore *stale* entries when they are popped. This is called **lazy deletion**, and it is what our trace and code use, because it works with simple binary heaps such as Python's `heapq` or C++'s `std::priority_queue`.

```
Dijkstra-Lazy(G, w, s):
    dist[v] = ∞ for all v;  dist[s] = 0
    PQ = [(0, s)]
    while PQ not empty:
        (d, u) = PQ.pop_min()
        if d > dist[u]: continue        # stale entry, skip
        for each edge (u, v, w):
            if d + w < dist[v]:
                dist[v] = d + w
                prev[v] = u
                PQ.push((dist[v], v))
```

---

## 3. A Worked Example

Consider this directed graph with six vertices `A`–`F`. The source is `A`.

| Edge  | Weight |
|-------|-------:|
| A → B | 4 |
| A → C | 2 |
| C → B | 1 |
| B → D | 5 |
| C → D | 8 |
| C → E | 10 |
| D → E | 2 |
| D → F | 6 |
| E → F | 2 |

A rough picture:

```
            4
     A ----------> B
     |           ^ |
   2 |        1 /  | 5
     v         /   v
     C -------+    D ------6------> F
     |  \     8    ^ \              ^
     |   +---------+  \ 2           |
  10 |                 v            | 2
     +---------------> E -----------+
```

Notice the traps: the direct edge `A → B` (cost 4) is beaten by `A → C → B` (cost 3), and the direct edge `C → D` (cost 8 from C, total 10) is beaten by going through `B`. A correct algorithm must discover these detours.

We use the lazy-deletion version. When two queue entries tie on distance, we break ties alphabetically by vertex name; any tie-breaking rule gives the same final distances.

### Initial state

| Vertex | A | B | C | D | E | F |
|--------|---|---|---|---|---|---|
| dist   | 0 | ∞ | ∞ | ∞ | ∞ | ∞ |
| prev   | – | – | – | – | – | – |

**Priority queue:** `[(0, A)]`
**Settled:** {}

### Iteration 1 — pop `(0, A)`

`A` is not stale (`0 = dist[A]`), so settle it and relax its edges:

- `A → B`: `0 + 4 = 4 < ∞` → `dist[B] = 4`, `prev[B] = A`, push `(4, B)`.
- `A → C`: `0 + 2 = 2 < ∞` → `dist[C] = 2`, `prev[C] = A`, push `(2, C)`.

| Vertex | A | B | C | D | E | F |
|--------|---|---|---|---|---|---|
| dist   | **0** | 4 | 2 | ∞ | ∞ | ∞ |
| prev   | – | A | A | – | – | – |

| Priority queue (min first) |
|---|
| (2, C) |
| (4, B) |

**Settled:** {A}

### Iteration 2 — pop `(2, C)`

Valid entry. Settle `C`, relax its three edges:

- `C → B`: `2 + 1 = 3 < 4` → **improvement**: `dist[B] = 3`, `prev[B] = C`, push `(3, B)`. The old `(4, B)` entry is now stale but stays in the heap.
- `C → D`: `2 + 8 = 10 < ∞` → `dist[D] = 10`, `prev[D] = C`, push `(10, D)`.
- `C → E`: `2 + 10 = 12 < ∞` → `dist[E] = 12`, `prev[E] = C`, push `(12, E)`.

| Vertex | A | B | C | D | E | F |
|--------|---|---|---|---|---|---|
| dist   | **0** | 3 | **2** | 10 | 12 | ∞ |
| prev   | – | C | A | C | C | – |

| Priority queue (min first) |
|---|
| (3, B) |
| (4, B) *stale* |
| (10, D) |
| (12, E) |

**Settled:** {A, C}

### Iteration 3 — pop `(3, B)`

Valid (`3 = dist[B]`). Settle `B`:

- `B → D`: `3 + 5 = 8 < 10` → **improvement**: `dist[D] = 8`, `prev[D] = B`, push `(8, D)`.

| Vertex | A | B | C | D | E | F |
|--------|---|---|---|---|---|---|
| dist   | **0** | **3** | **2** | 8 | 12 | ∞ |
| prev   | – | C | A | B | C | – |

| Priority queue (min first) |
|---|
| (4, B) *stale* |
| (8, D) |
| (10, D) *stale* |
| (12, E) |

**Settled:** {A, C, B}

### Iteration 4 — pop `(4, B)`

`4 > dist[B] = 3`, so this entry is **stale**. Discard it and do nothing else.

| Priority queue (min first) |
|---|
| (8, D) |
| (10, D) *stale* |
| (12, E) |

**Settled:** {A, C, B} (unchanged)

### Iteration 5 — pop `(8, D)`

Valid. Settle `D`:

- `D → E`: `8 + 2 = 10 < 12` → **improvement**: `dist[E] = 10`, `prev[E] = D`, push `(10, E)`.
- `D → F`: `8 + 6 = 14 < ∞` → `dist[F] = 14`, `prev[F] = D`, push `(14, F)`.

| Vertex | A | B | C | D | E | F |
|--------|---|---|---|---|---|---|
| dist   | **0** | **3** | **2** | **8** | 10 | 14 |
| prev   | – | C | A | B | D | D |

| Priority queue (min first) |
|---|
| (10, D) *stale* |
| (10, E) |
| (12, E) *stale* |
| (14, F) |

**Settled:** {A, C, B, D}

### Iteration 6 — pop `(10, D)`

Tie at distance 10; `D` sorts before `E`. But `10 > dist[D] = 8`: **stale**, discard.

| Priority queue (min first) |
|---|
| (10, E) |
| (12, E) *stale* |
| (14, F) |

### Iteration 7 — pop `(10, E)`

Valid. Settle `E`:

- `E → F`: `10 + 2 = 12 < 14` → **improvement**: `dist[F] = 12`, `prev[F] = E`, push `(12, F)`.

| Vertex | A | B | C | D | E | F |
|--------|---|---|---|---|---|---|
| dist   | **0** | **3** | **2** | **8** | **10** | 12 |
| prev   | – | C | A | B | D | E |

| Priority queue (min first) |
|---|
| (12, E) *stale* |
| (12, F) |
| (14, F) *stale* |

**Settled:** {A, C, B, D, E}

### Iteration 8 — pop `(12, E)`

`12 > dist[E] = 10`: **stale**, discard.

| Priority queue (min first) |
|---|
| (12, F) |
| (14, F) *stale* |

### Iteration 9 — pop `(12, F)`

Valid. Settle `F`. It has no outgoing edges, so nothing to relax.

| Priority queue (min first) |
|---|
| (14, F) *stale* |

**Settled:** {A, C, B, D, E, F}

### Iteration 10 — pop `(14, F)`

Stale (`14 > 12`), discard. The queue is now empty and the algorithm terminates.

### Final result

| Vertex | Distance | Predecessor | Shortest path |
|--------|---------:|-------------|---------------|
| A | 0  | –  | A |
| B | 3  | C  | A → C → B |
| C | 2  | A  | A → C |
| D | 8  | B  | A → C → B → D |
| E | 10 | D  | A → C → B → D → E |
| F | 12 | E  | A → C → B → D → E → F |

To reconstruct a path, follow `prev` pointers backward from the target and reverse: `F ← E ← D ← B ← C ← A`.

Some observations from the trace:

1. **Vertices were settled in non-decreasing order of distance**: A (0), C (2), B (3), D (8), E (10), F (12). This is no accident; it is the heart of the correctness proof.
2. **Every vertex's distance improved at most a few times**, and each improvement corresponded to discovering a cheaper detour (`A→C→B` instead of `A→B`, `…→B→D` instead of `C→D`, and so on).
3. **Ten pops happened for six vertices.** The four extra pops were stale entries left over from lazy deletion. With a `decrease_key` heap there would be exactly six extractions, but the bookkeeping per operation is heavier.
4. Once a vertex was settled, its distance never changed again.

---

## 4. Why We Need the Priority Queue

What makes Dijkstra's algorithm different from a breadth-first search? BFS processes vertices in order of *number of edges* from the source; it finds shortest paths only when every edge has the same weight. Dijkstra generalizes BFS by replacing the FIFO queue with a priority queue keyed by accumulated distance. In fact, if every edge weight is 1, Dijkstra's algorithm visits vertices in exactly the same order as BFS (modulo ties).

Conversely, if you replaced the priority queue with a plain FIFO queue but kept re-adding vertices whenever their distance improved, you would get a variant of Bellman–Ford (the "SPFA" heuristic). It is still correct for non-negative weights, but it may relax each edge many times, losing Dijkstra's efficiency guarantee.

---

## 5. Proof Sketch of Correctness

We prove the following invariant:

> **Claim.** When a vertex `u` is extracted from the priority queue (settled), `dist[u] = δ(s, u)`.

Two simple facts are useful first.

**Fact 1 (upper bound).** At all times, `dist[v] ≥ δ(s, v)`. Each finite value of `dist[v]` is the length of a real path from `s` to `v` (built by relaxation), and no real path is shorter than the shortest one.

**Fact 2 (convergence).** If `x → y` is the last edge of some shortest path to `y`, and `dist[x] = δ(s, x)` at the moment we relax `x → y`, then afterward `dist[y] = δ(s, y)`. This is because relaxation sets `dist[y] ≤ δ(s, x) + w(x, y) = δ(s, y)`, and Fact 1 prevents it from going lower.

**Proof of the claim (by contradiction).** Suppose the claim fails, and let `u` be the *first* vertex settled with `dist[u] > δ(s, u)`. Clearly `u ≠ s`, since `dist[s] = 0 = δ(s, s)`. Also `δ(s, u) < ∞` (otherwise `dist[u] = ∞ = δ(s, u)`), so a shortest path `P` from `s` to `u` exists.

At the moment just before `u` is settled, let `S` be the set of already-settled vertices. The path `P` starts in `S` (at `s`) and ends outside `S` (at `u`). So there is a first edge `x → y` on `P` with `x ∈ S` and `y ∉ S`.

- Since `x` was settled before `u`, and `u` is the first failure, `dist[x] = δ(s, x)`. When `x` was settled, edge `x → y` was relaxed, and by Fact 2 (subpaths of shortest paths are shortest paths), `dist[y] = δ(s, y)`.
- Because `y` lies on the shortest path to `u` *before* `u`, and **all edge weights are non-negative**, `δ(s, y) ≤ δ(s, u)`.
- Putting it together: `dist[y] = δ(s, y) ≤ δ(s, u) < dist[u]`.

But `y` is unsettled and has a strictly smaller tentative distance than `u`, so the priority queue should have returned `y` (or something with distance at most `dist[y]`) instead of `u`. Contradiction. (If `y = u`, then `dist[u] = δ(s, u)` directly, also a contradiction.) ∎

The single place where non-negative weights entered was the inequality `δ(s, y) ≤ δ(s, u)`: walking further along a path can never make it cheaper. Section 7 shows exactly what goes wrong when this fails.

**Termination and the lazy variant.** Each vertex is settled at most once, and each settled vertex relaxes each outgoing edge once, so there are at most `m` successful relaxations and at most `m + 1` pushes. In the lazy variant, a popped entry `(d, u)` with `d > dist[u]` is skipped; the first non-stale pop of `u` has `d = dist[u]`, and the claim above applies to it unchanged.

---

## 6. Running Time with Different Priority Queues

The algorithm performs:

- `n` **insert** operations (or one bulk build),
- `n` **extract-min** operations,
- up to `m` **decrease-key** operations (one per successful relaxation).

Its total running time is therefore

```
T = n·T_insert + n·T_extract_min + m·T_decrease_key
```

and the choice of priority queue decides everything.

| Priority queue | insert | extract-min | decrease-key | Total for Dijkstra | Best when |
|---|---|---|---|---|---|
| Unsorted array / plain `dist` array | O(1) | O(n) | O(1) | **O(n²)** | Dense graphs, `m ≈ n²` |
| Binary heap | O(log n) | O(log n) | O(log n) | **O((n + m) log n)** | Sparse graphs; the practical default |
| Binary heap, lazy deletion (no decrease-key) | O(log m) | O(log m) | — (push instead) | **O(m log m) = O(m log n)** | Simple code, standard libraries |
| d-ary heap, `d = max(2, m/n)` | O(log_d n) | O(d log_d n) | O(log_d n) | **O(m log_{m/n} n)** | Tunes between array and binary heap |
| Fibonacci heap | O(1) amortized | O(log n) amortized | O(1) amortized | **O(m + n log n)** | Theoretical optimum for comparison-based PQs |
| Pairing heap | O(1) | O(log n) amortized | o(log n) amortized | Near Fibonacci in practice | Fast in practice, simpler than Fibonacci |
| Dial's buckets (integer weights ≤ C) | O(1) | amortized O(1) + scan | O(1) | **O(m + n·C)** | Small integer weights |
| Radix heap (integer weights ≤ C) | O(1) | O(log C) amortized | O(1) | **O(m + n log C)** | Integer weights, large range |

A few notes:

- **The array version** is the one Dijkstra originally described. Each extract-min scans all `n` vertices, giving `O(n²)` total regardless of `m`. For a complete graph, where `m = Θ(n²)`, this is optimal: you must look at every edge anyway, and a heap only adds a log factor.
- **The binary heap** wins on sparse graphs (road networks, where `m` is a small multiple of `n`). For `n = 10⁶` and `m = 4·10⁶`, `O(m log n)` is around 80 million basic steps, versus `10¹²` for the array.
- **`O(m log m)` vs `O(m log n)`.** Lazy deletion lets the heap hold up to `m` entries. Since `m ≤ n²`, `log m ≤ 2 log n`, so the bound is still `O(m log n)`. The heap uses `O(m)` memory instead of `O(n)`.
- **Fibonacci heaps** achieve `O(m + n log n)`, which is optimal among comparison-based approaches, since Dijkstra can be used to sort `n` numbers (put them on edges from a single source). In practice, their constant factors and pointer-heavy structure usually make them slower than a binary or 4-ary heap on real inputs.
- **Integer-weight structures** escape the comparison lower bound. Dial's algorithm uses an array of `C·n + 1` buckets indexed by distance (or a circular array of `C + 1` buckets). Radix heaps and more advanced structures (Thorup's algorithm achieves linear time for undirected graphs with integer weights) push further.

Space usage is `O(n + m)` for the graph in adjacency-list form plus `O(n)` for `dist`, `prev`, and a `decrease_key` heap, or `O(m)` for a lazy heap.

**Early exit.** If you only need the distance to one target `t`, you can stop as soon as `t` is settled. This does not change the worst-case bound, but in practice it can cut work dramatically. Bidirectional search (running Dijkstra from both `s` and `t` and meeting in the middle) and A\* (adding an admissible heuristic to the key) refine this idea further.

---

## 7. Why Negative Edges Break Dijkstra

Consider this small directed graph, source `A`:

| Edge | Weight |
|------|-------:|
| A → B | 1 |
| A → C | 3 |
| C → B | −3 |
| B → D | 1 |

The true shortest distances are easy to compute by hand:

- `δ(A, C) = 3`
- `δ(A, B) = min(1, 3 + (−3)) = 0`, via `A → C → B`
- `δ(A, D) = δ(A, B) + 1 = 1`, via `A → C → B → D`

There is no negative *cycle*, so shortest paths are perfectly well defined. Now run classic Dijkstra (the version that never revisits a settled vertex):

| Step | Pop | Settled | Relaxations | dist A / B / C / D |
|---|---|---|---|---|
| 0 | — | {} | init | 0 / ∞ / ∞ / ∞ |
| 1 | A (0) | {A} | B ← 1, C ← 3 | 0 / 1 / 3 / ∞ |
| 2 | B (1) | {A, B} | D ← 2 | 0 / **1** / 3 / 2 |
| 3 | D (2) | {A, B, D} | — | 0 / 1 / 3 / **2** |
| 4 | C (3) | {A, B, D, C} | C → B: 3 − 3 = 0 < 1, but B is already settled | 0 / 1 / 3 / 2 |

The algorithm reports `dist[B] = 1` and `dist[D] = 2`. Both are wrong; the correct answers are 0 and 1.

What happened? At step 2, `B` was the closest unsettled vertex, so the greedy rule declared its distance final. The proof's key inequality, "any detour through an unsettled vertex `y` costs at least `dist[y]`", is false here: the detour through `C` starts more expensive (3 > 1) but the negative edge pulls it back down to 0. The wavefront picture breaks too: water cannot flow backward in time, but a negative edge would require exactly that.

**What about the lazy implementation?** If your code has no "settled" check and simply re-pushes any vertex whose distance improves, then at step 4 it would set `dist[B] = 0`, push `(0, B)`, pop it, and re-relax `B → D` to give `dist[D] = 1`. The final answer is correct in this case. But this is no longer Dijkstra's algorithm in any meaningful sense: vertices can be re-processed many times, and there are known graph families with negative edges on which this "label-correcting" approach takes **exponential** time. And if the graph contains a negative cycle, it never terminates. Do not rely on it.

**What to use instead:**

- **Bellman–Ford**, `O(n·m)`: relaxes every edge `n − 1` times; handles negative edges and detects negative cycles.
- **Johnson's algorithm** for all-pairs shortest paths: run Bellman–Ford once from a new virtual source to compute a potential `h(v)`, reweight each edge as `w'(u, v) = w(u, v) + h(u) − h(v) ≥ 0`, then run Dijkstra from every vertex. Reweighting preserves which paths are shortest because the potentials telescope along any path.
- **Adding a constant to every edge does *not* work.** It penalizes paths with more edges, changing which path is shortest. In the example above, adding 3 to every edge gives `A → B = 4` but `A → C → B = 6 + 0 = 6`, so the wrong path now looks best.
- Recent theoretical work (Bernstein, Nanongkai, and Wulff-Nilsen, 2022) gives near-linear time SSSP with negative integer weights, but Bellman–Ford and Johnson remain the practical standards.

---

## 8. Code

### Python (binary heap with lazy deletion)

```python
import heapq
from math import inf
from typing import Dict, Hashable, List, Optional, Tuple

Graph = Dict[Hashable, List[Tuple[Hashable, float]]]


def dijkstra(graph: Graph, source: Hashable):
    """Single-source shortest paths for non-negative edge weights.

    graph maps each vertex to a list of (neighbor, weight) pairs.
    Returns (dist, prev): dist[v] is the shortest distance from source
    (inf if unreachable) and prev[v] is v's predecessor on that path.
    """
    dist: Dict[Hashable, float] = {v: inf for v in graph}
    prev: Dict[Hashable, Optional[Hashable]] = {v: None for v in graph}
    dist[source] = 0
    heap = [(0, source)]

    while heap:
        d, u = heapq.heappop(heap)
        if d > dist[u]:
            continue  # stale entry left behind by an earlier improvement
        for v, w in graph.get(u, ()):
            if w < 0:
                raise ValueError(f"negative edge {u}->{v} ({w}); use Bellman-Ford")
            nd = d + w
            if nd < dist.get(v, inf):
                dist[v] = nd
                prev[v] = u
                heapq.heappush(heap, (nd, v))
    return dist, prev


def shortest_path(prev, source, target):
    """Rebuild the path source -> target from predecessor pointers."""
    if target != source and prev.get(target) is None:
        return None  # unreachable
    path = []
    node = target
    while node is not None:
        path.append(node)
        node = prev[node]
    path.reverse()
    return path if path[0] == source else None


if __name__ == "__main__":
    g = {
        "A": [("B", 4), ("C", 2)],
        "B": [("D", 5)],
        "C": [("B", 1), ("D", 8), ("E", 10)],
        "D": [("E", 2), ("F", 6)],
        "E": [("F", 2)],
        "F": [],
    }
    dist, prev = dijkstra(g, "A")
    for v in sorted(g):
        print(v, dist[v], " -> ".join(shortest_path(prev, "A", v)))
```

Running this prints exactly the table from Section 3:

```
A 0 A
B 3 A -> C -> B
C 2 A -> C
D 8 A -> C -> B -> D
E 10 A -> C -> B -> D -> E
F 12 A -> C -> B -> D -> E -> F
```

Implementation notes:

- **Ties and unorderable vertices.** Tuples compare element by element, so if two entries have equal distance, Python compares the vertices. That is fine for strings and integers. For arbitrary objects, push `(dist, counter, vertex)` with a monotonically increasing counter to avoid comparison errors.
- **Vertices that only appear as targets** are handled by `dist.get(v, inf)`; they are added to `dist` on first discovery.
- **The negative-weight check** is cheap and turns a silent wrong answer into a loud error.

### C++ (O(n²) array version for dense graphs)

For dense graphs given as an adjacency matrix, the original array-based version is both simplest and asymptotically best:

```cpp
#include <vector>
#include <limits>

// adj[u][v] = weight of edge u->v, or INF if absent. All weights >= 0.
std::vector<long long> dijkstraDense(const std::vector<std::vector<long long>>& adj,
                                     int source,
                                     std::vector<int>& prev) {
    const long long INF = std::numeric_limits<long long>::max() / 4;
    int n = adj.size();
    std::vector<long long> dist(n, INF);
    std::vector<bool> settled(n, false);
    prev.assign(n, -1);
    dist[source] = 0;

    for (int iter = 0; iter < n; ++iter) {
        int u = -1;
        for (int v = 0; v < n; ++v)              // O(n) extract-min
            if (!settled[v] && (u == -1 || dist[v] < dist[u]))
                u = v;
        if (dist[u] == INF) break;                // remaining vertices unreachable
        settled[u] = true;
        for (int v = 0; v < n; ++v)               // relax all edges out of u
            if (adj[u][v] < INF && dist[u] + adj[u][v] < dist[v]) {
                dist[v] = dist[u] + adj[u][v];
                prev[v] = u;
            }
    }
    return dist;
}
```

Both loops are `O(n)` and run `n` times, giving `O(n²)` with tiny constants and excellent cache behavior.

---

## 9. Variations and Practical Tips

- **Multiple sources.** To find the distance from the *nearest* of several sources, initialize all of them with distance 0 and push them all at once. One run handles all sources together.
- **Shortest path to a set of targets.** Stop when the first target is settled.
- **Single-pair queries on huge graphs.** Use bidirectional Dijkstra, A\* with a consistent heuristic (straight-line distance on maps), or preprocessing-based techniques such as contraction hierarchies, which are what make continent-scale route planning answer in milliseconds.
- **Counting shortest paths or breaking ties.** Track an extra count array updated when `dist[u] + w == dist[v]`, or use a lexicographic key such as `(distance, hops)`.
- **Floating-point weights.** Comparisons such as `nd < dist[v]` can flip because of rounding. Usually harmless, but for exactness consider integer weights (e.g., centimeters instead of meters).
- **Overflow.** In fixed-width languages, choose `INF` well below the type's maximum so that `INF + w` cannot wrap around, as in the C++ snippet above.
- **Maximum-probability or bottleneck paths.** Dijkstra's greedy structure works for any path-cost function that is monotone non-decreasing along a path. Products of probabilities in `[0, 1]` (maximize) or path bottlenecks (max-min capacity) can be handled with a modified relaxation step; equivalently, take `−log p` to turn probabilities into non-negative additive weights.

---

## 10. Summary

- Dijkstra's algorithm solves single-source shortest paths on graphs with **non-negative** edge weights.
- It repeatedly settles the unsettled vertex with the smallest tentative distance and relaxes its outgoing edges; vertices are settled in non-decreasing order of true distance.
- Correctness rests on one inequality: with non-negative weights, extending a path never makes it shorter. The proof picks the first vertex settled incorrectly and finds an unsettled vertex that should have been chosen instead.
- Running time depends on the priority queue: `O(n²)` with an array (best for dense graphs), `O((n + m) log n)` with a binary heap (the practical default), `O(m + n log n)` with a Fibonacci heap, and better still for small integer weights.
- With negative edges it can return wrong answers, as the four-vertex example shows. Use Bellman–Ford, or Johnson's reweighting followed by Dijkstra, instead.
- A clean implementation is about twenty lines: a distance map, a predecessor map, a heap, and a stale-entry check.
