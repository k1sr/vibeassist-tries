# A* Search, Explained in Full

A* (pronounced "A-star") is the standard algorithm for finding a shortest path from a start node to a goal node in a weighted graph, given a guess of how far each node is from the goal. It was published in 1968 by Peter Hart, Nils Nilsson and Bertram Raphael for the Shakey robot project. It is still the default choice for game pathfinding, robot motion planning, puzzle solving and route finding.

This explainer covers the algorithm itself, a complete hand trace on a small grid, how to pick a heuristic, proofs that it returns optimal paths, how it compares with Dijkstra's algorithm and greedy best-first search, tie-breaking, and a working implementation.

---

## 1. The problem and the core idea

We have:

- a graph of nodes connected by edges, each edge with a non-negative cost `c(u, v)`;
- a start node `s` and a goal node `t`;
- a **heuristic** function `h(n)` that estimates the cost of the cheapest path from `n` to `t`.

We want the cheapest path from `s` to `t`.

A* keeps two numbers for every node it has reached:

- `g(n)`: the cost of the best path from `s` to `n` found so far. This number is known exactly for that path.
- `h(n)`: the estimated remaining cost from `n` to the goal.

It adds them:

```
f(n) = g(n) + h(n)
```

`f(n)` estimates the total cost of the cheapest path from start to goal that passes through `n`. A* always expands the node with the smallest `f`. That one rule is the whole algorithm. Everything else is bookkeeping.

The intuition: Dijkstra's algorithm grows a circle of explored nodes outward from the start, ordered by `g`. It knows nothing about where the goal is, so it spends as much effort in the wrong direction as in the right one. A* tilts that circle toward the goal by adding `h`. Nodes that lie in the right direction have small `h`, so they come up for expansion sooner. Nodes far off course get a large `h` and wait, often forever.

## 2. The algorithm

A* maintains two sets:

- The **open set** (also called the frontier or fringe) holds nodes that have been discovered but not yet expanded. It is a priority queue ordered by `f`.
- The **closed set** holds nodes that have already been expanded. When A* uses a consistent heuristic (defined in section 5), a node's `g` value is final once it is closed.

Pseudocode:

```
A*(s, t, h):
    g[s] = 0
    parent[s] = none
    open = priority queue containing s with priority f = h(s)
    closed = empty set

    while open is not empty:
        n = pop node with smallest f from open
        if n == t:
            return reconstruct_path(parent, t)
        add n to closed
        for each neighbour m of n:
            if m in closed: continue
            tentative = g[n] + c(n, m)
            if m not seen or tentative < g[m]:
                g[m] = tentative
                parent[m] = n
                push or update m in open with f = g[m] + h(m)

    return failure
```

Two details deserve attention.

**The goal test happens when the goal is popped, not when it is pushed.** The first path to reach the goal is not necessarily the cheapest. A cheaper path might still be sitting in the open set with a lower `f`. Only when the goal has the lowest `f` of everything in the queue can we be sure no better path exists. Testing on push is a common bug that breaks optimality.

**"Push or update."** Many priority queues have no decrease-key operation. The usual workaround is to push a duplicate entry with the better priority and, when popping, skip any entry for a node that is already closed (this is called "lazy deletion"). The implementation in section 9 does this.

## 3. A worked example on a grid

### The map

A 5-row by 6-column grid. Movement is 4-connected (up, down, left, right), and every move costs 1. `#` is a wall. Coordinates are `(row, col)` with `(0, 0)` at the top left.

```
      col: 0 1 2 3 4 5
row 0:     . . . . . .
row 1:     . # # # # .
row 2:     S . . . # G
row 3:     . . . # . .
row 4:     . . . . . .
```

Start `S = (2, 0)`, goal `G = (2, 5)`. The straight route along row 2 hits a dead end at `(2, 3)`: walls lie to the right `(2, 4)`, above `(1, 3)` and below `(3, 3)`. The real shortest path has to go around, either over the top through row 0 or under the bottom through row 4. That makes the map a good test. The heuristic will first be fooled by the corridor, and we can watch A* recover.

### The heuristic

We use **Manhattan distance**: `h(r, c) = |r - 2| + |c - 5|`. Section 4 explains why it suits a 4-connected grid.

### Tie-breaking rule

Many nodes will share an `f` value. In this trace ties are broken by **smaller `h` first** (prefer nodes that look closer to the goal), and then by **insertion order** (first in, first out). Section 8 explains why this matters.

### Neighbour order

Neighbours are generated in the order up, right, down, left.

### The trace

Each row shows the node being expanded, its `g`, `h` and `f`, and then the open set after its neighbours have been added. Open-set entries are written `(r,c) g+h=f` and sorted by priority. The closed set is cumulative: it holds every node expanded in this row and in earlier rows.

| Iter | Expand (g, h, f) | Open set after expansion | Closed set |
|---|---|---|---|
| 0 | (start) | (2,0) 0+5=5 | {} |
| 1 | (2,0) 0, 5, 5 | (2,1) 1+4=5; (1,0) 1+6=7; (3,0) 1+6=7 | (2,0) |
| 2 | (2,1) 1, 4, 5 | (2,2) 2+3=5; (3,1) 2+5=7; (1,0) 1+6=7; (3,0) 1+6=7 | + (2,1) |
| 3 | (2,2) 2, 3, 5 | (2,3) 3+2=5; (3,2) 3+4=7; (3,1) 2+5=7; (1,0) 1+6=7; (3,0) 1+6=7 | + (2,2) |
| 4 | (2,3) 3, 2, 5 | (3,2) 3+4=7; (3,1) 2+5=7; (1,0) 1+6=7; (3,0) 1+6=7 | + (2,3) |
| 5 | (3,2) 3, 4, 7 | (3,1) 2+5=7; (1,0) 1+6=7; (3,0) 1+6=7; (4,2) 4+5=9 | + (3,2) |
| 6 | (3,1) 2, 5, 7 | (1,0) 1+6=7; (3,0) 1+6=7; (4,2) 4+5=9; (4,1) 3+6=9 | + (3,1) |
| 7 | (1,0) 1, 6, 7 | (3,0) 1+6=7; (4,2) 4+5=9; (4,1) 3+6=9; (0,0) 2+7=9 | + (1,0) |
| 8 | (3,0) 1, 6, 7 | (4,2) 4+5=9; (4,1) 3+6=9; (0,0) 2+7=9; (4,0) 2+7=9 | + (3,0) |
| 9 | (4,2) 4, 5, 9 | (4,3) 5+4=9; (4,1) 3+6=9; (0,0) 2+7=9; (4,0) 2+7=9 | + (4,2) |
| 10 | (4,3) 5, 4, 9 | (4,4) 6+3=9; (4,1) 3+6=9; (0,0) 2+7=9; (4,0) 2+7=9 | + (4,3) |
| 11 | (4,4) 6, 3, 9 | (3,4) 7+2=9; (4,5) 7+2=9; (4,1) 3+6=9; (0,0) 2+7=9; (4,0) 2+7=9 | + (4,4) |
| 12 | (3,4) 7, 2, 9 | (3,5) 8+1=9; (4,5) 7+2=9; (4,1) 3+6=9; (0,0) 2+7=9; (4,0) 2+7=9 | + (3,4) |
| 13 | (3,5) 8, 1, 9 | (2,5) 9+0=9; (4,5) 7+2=9; (4,1) 3+6=9; (0,0) 2+7=9; (4,0) 2+7=9 | + (3,5) |
| 14 | pop (2,5) 9, 0, 9 | **goal reached** | |

### What happened, phase by phase

**Iterations 1 to 4: the heuristic leads into the corridor.** Every node along row 2 has `f = 5`, which equals the Manhattan lower bound from the start. A* walks straight toward the goal and expands only four nodes. Then it reaches `(2, 3)`, and all three onward neighbours are walls. Nothing new is added. The corridor is a trap the heuristic cannot see, because Manhattan distance ignores walls.

**Iterations 5 to 8: the frontier rises to the next `f` level.** The smallest `f` left in the open set is now 7. A* works through every `f = 7` node. Among them, the tie-breaker picks the smallest `h` first: `(3,2)` with `h = 4`, then `(3,1)` with `h = 5`, then `(1,0)` and `(3,0)` with `h = 6`. Note that `(1,0)` is expanded but its neighbour `(1,1)` is a wall, so the only way up is `(0,0)`, which lands at `f = 9`.

**Iterations 9 to 13: the `f = 9` sweep along the bottom.** All remaining work happens at `f = 9`, the true optimal cost. With tie-breaking on smallest `h`, A* runs straight along the bottom row and up the right side: `(4,2)`, `(4,3)`, `(4,4)`, `(3,4)`, `(3,5)`. Each step lowers `h` by one, and each newly discovered node goes to the front of the tie group. The top route is never explored. `(0,0)` is in the open set with `f = 9`, but its `h = 7` keeps it at the back.

**Iteration 14: the goal is popped** with `g = 9`. Following parent pointers backward gives the path:

```
(2,0) → (2,1) → (2,2) → (3,2) → (4,2) → (4,3) → (4,4) → (3,4) → (3,5) → (2,5)
```

That is 9 moves. Drawn on the map:

```
      col: 0 1 2 3 4 5
row 0:     . . . . . .
row 1:     . # # # # .
row 2:     S * * x # G
row 3:     . . * # * *
row 4:     . . * * * .
```

(`*` marks the path, `x` marks the dead end that A* explored and abandoned.)

A* expanded 13 of the 25 open cells. Dijkstra's algorithm on the same map, with the same tie-breaking, expands 23. The exact counts appear in section 6.

## 4. Choosing a heuristic: Manhattan, Euclidean and friends

The heuristic is where you put your knowledge of the problem. It has to match the way the agent can move.

### Manhattan distance

```
h(n) = |dx| + |dy|
```

This is the exact cost of the shortest path on a 4-connected grid with no obstacles and unit move cost. With obstacles the true distance can only be longer, so Manhattan never overestimates. On 4-connected grids it is the strongest simple heuristic you can use.

On an **8-connected** grid, where diagonal moves are allowed, Manhattan is *not* safe. A diagonal step changes both `dx` and `dy` by 1 at a cost of 1 (or about 1.414), but Manhattan counts it as 2. Manhattan therefore overestimates, and A* can return a suboptimal path.

### Euclidean distance

```
h(n) = sqrt(dx² + dy²)
```

This is the straight-line distance. Every path, whatever moves it uses, is at least as long as the straight line, so Euclidean never overestimates in any grid or continuous space with unit-scaled costs. It is the natural choice when movement is any-angle (navigation meshes, continuous planning, road networks with coordinates).

On a 4-connected grid, though, Euclidean is **weaker** than Manhattan. For the goal at `dx = 3, dy = 4`, Manhattan says 7 (the true obstacle-free cost) while Euclidean says 5. A weaker heuristic means smaller `f` values for off-course nodes, so more of them get expanded before the goal. On our example map:

| Heuristic | Nodes expanded | Path cost |
|---|---|---|
| `h = 0` (Dijkstra) | 23 | 9 |
| Euclidean | 20 | 9 |
| Manhattan | 13 | 9 |

All three find the optimal path, because all three are admissible. They differ only in how much work they do.

### Octile and Chebyshev distance

For 8-connected grids, use **octile distance** when diagonals cost √2:

```
h(n) = max(dx, dy) + (√2 − 1) · min(dx, dy)
```

and **Chebyshev distance**, `max(dx, dy)`, when diagonals cost 1. Each is exact on an empty grid with its movement model, just as Manhattan is for 4-connected grids.

### The general rule

A heuristic is better the closer it gets to the true remaining cost **without exceeding it**. If `h1(n) ≤ h2(n) ≤ h*(n)` for all `n`, where `h*` is the true cost, then `h2` **dominates** `h1`, and A* with `h2` never expands a node that A* with `h1` would skip (ties aside). The perfect heuristic `h = h*` expands only nodes on an optimal path. The zero heuristic expands everything Dijkstra would.

## 5. Admissibility and consistency, with proofs

### Definitions

A heuristic is **admissible** if it never overestimates:

```
h(n) ≤ h*(n)   for every node n
```

where `h*(n)` is the true cheapest cost from `n` to the goal.

A heuristic is **consistent** (or monotone) if, for every edge `(n, m)`:

```
h(n) ≤ c(n, m) + h(m)
```

and `h(goal) = 0`. This is a triangle inequality: the estimate from `n` cannot drop by more than the cost of the step you take.

### Claim 1: Consistency implies admissibility

*Proof.* Take any node `n` and an optimal path `n = n0, n1, ..., nk = goal`. Apply consistency to each edge and sum:

```
h(n0) ≤ c(n0, n1) + h(n1)
      ≤ c(n0, n1) + c(n1, n2) + h(n2)
      ≤ ...
      ≤ c(n0, n1) + ... + c(n(k-1), nk) + h(nk)
      = h*(n) + 0
```

So `h(n) ≤ h*(n)`. ∎

The converse is false. Admissible heuristics can be inconsistent, although this rarely happens with heuristics built from geometry.

### Claim 2: Manhattan distance is consistent on a 4-connected unit grid

*Proof.* A single move changes exactly one coordinate by exactly 1. So `|dx| + |dy|` changes by exactly 1, up or down. Then `h(n) ≤ 1 + h(m) = c(n, m) + h(m)`. At the goal, `dx = dy = 0`, so `h = 0`. ∎

### Claim 3: Euclidean distance is consistent for any move set whose cost is at least its displacement

*Proof.* By the triangle inequality for Euclidean distance, `‖n − t‖ ≤ ‖n − m‖ + ‖m − t‖`. If each move from `n` to `m` costs at least its straight-line length `‖n − m‖`, then `h(n) ≤ c(n, m) + h(m)`. This covers 4-connected and 8-connected grids with unit or √2 diagonal costs. ∎

### Claim 4: With an admissible heuristic, A* returns an optimal path (tree search, or graph search with re-opening)

*Proof.* Let `C*` be the optimal cost. Suppose for contradiction that A* pops the goal `t` with `g(t) > C*`. Consider an optimal path `s = p0, p1, ..., pk = t`. At any moment before the goal is popped, some node `pi` on this path is in the open set with `g(pi) = g*(pi)`, its optimal cost. This holds because `s` starts in open with `g = 0`, and whenever the first such node is expanded, its successor on the path is pushed with its optimal `g`. For that node:

```
f(pi) = g*(pi) + h(pi) ≤ g*(pi) + h*(pi) = C* < g(t) = f(t)
```

(The goal has `h(t) = 0`, so `f(t) = g(t)`.) Since `f(pi) < f(t)`, A* would pop `pi` before `t`. Repeating the argument, every node on the optimal path is popped before `t`, including the moment `t` is reached through that path with `g = C*`. This contradicts `g(t) > C*`. ∎

### Claim 5: With a consistent heuristic, `f` is non-decreasing along any path, and a closed node never needs re-opening

*Proof (monotone f).* For an edge `(n, m)` with `g(m) = g(n) + c(n, m)`:

```
f(m) = g(n) + c(n, m) + h(m) ≥ g(n) + h(n) = f(n)
```

using consistency. So `f` never decreases along a path.

*Proof (no re-opening).* Suppose node `m` is popped with value `g(m)` while a cheaper path to `m` exists through some node `p` still in open. Along that cheaper path, `f` is non-decreasing, so `f(p) ≤ g_better(m) + h(m) < g(m) + h(m) = f(m)`. Then `p` would have been popped before `m`, a contradiction. So the first time a node is popped, its `g` is optimal. ∎

This is why the simple "skip closed neighbours" line in the pseudocode is safe with a consistent heuristic. With an admissible but inconsistent heuristic, graph search must allow closed nodes to be re-opened when a cheaper `g` turns up. Otherwise it can return a suboptimal path.

### Claim 6: A* with a consistent heuristic is optimally efficient

Among algorithms that are given the same heuristic and are guaranteed to find optimal paths, none expands fewer nodes than A* (up to tie-breaking). Every node with `f(n) < C*` must be expanded by any such algorithm. If one were skipped, a path through it might, as far as the algorithm can tell, cost less than `C*`. A* expands exactly those nodes, plus some with `f(n) = C*`, which is where tie-breaking comes in.

## 6. A* compared with Dijkstra's algorithm and greedy best-first search

All three algorithms share the same skeleton. They differ only in the priority used for the open set.

| Algorithm | Priority | Optimal? | Typical work |
|---|---|---|---|
| Dijkstra | `g(n)` | Yes (non-negative costs) | Explores in all directions |
| Greedy best-first | `h(n)` | No | Very little when the heuristic is good, a lot when it is misleading |
| A* | `g(n) + h(n)` | Yes, if `h` is admissible | Between the two |

### Dijkstra is A* with h = 0

Set `h(n) = 0` and A* becomes Dijkstra's algorithm exactly. The zero heuristic is trivially consistent, so Dijkstra is optimal, but it gets no help with direction. On our map it expanded 23 of 25 cells before popping the goal, including the whole top route and the far-left column.

Dijkstra is still the right tool when you need distances from one source to *many* targets, or when no useful heuristic exists. A* is a single-target algorithm. Its heuristic aims at one goal.

### Greedy best-first ignores g

Greedy best-first search ranks nodes by `h` alone. It rushes toward the goal and often finds *a* path quickly. On our grid it expanded only 11 nodes and happened to find the 9-step path, because every route around the wall was equally long.

It is not optimal in general. A minimal counterexample:

```
        1          10
   S ───────▶ A ────────▶ G
   │                      ▲
   │ 3                 3  │
   └────────▶ B ──────────┘

   h(S) = 5, h(A) = 1, h(B) = 3, h(G) = 0
```

The heuristic is admissible: `h(A) = 1 ≤ 10` and `h(B) = 3 ≤ 3`.

- **Greedy:** from `S`, open = {A (h=1), B (h=3)}. It pops `A`, pushes `G` with `h = 0`, and pops `G` next. It returns `S → A → G` with cost **11**.
- **A\*:** from `S`, open = {A f=1+1=2, B f=3+3=6}. It pops `A` and pushes `G` with `f = 11 + 0 = 11`. Then it pops `B` (`f = 6 < 11`), finds `G` with `g = 6`, and updates `G` to `f = 6`. It pops `G` and returns `S → B → G` with cost **6**.

The `g` term is what makes A* go back and question a path that looked promising but turned out expensive.

### The weighting view

You can write all three as `f(n) = w_g · g(n) + w_h · h(n)`:

- `w_g = 1, w_h = 0`: Dijkstra.
- `w_g = 0, w_h = 1`: greedy best-first.
- `w_g = 1, w_h = 1`: A*.
- `w_g = 1, w_h = w > 1`: **Weighted A\***. It is faster but only bounded-suboptimal: the path it returns costs at most `w` times the optimum. This trade-off is popular in games and robotics when "good enough, now" beats "perfect, later".

## 7. Complexity

In the worst case, with a useless heuristic, A* explores as much as Dijkstra does: `O((V + E) log V)` time with a binary heap, and `O(V)` memory for `g`, the parent pointers and the open and closed sets. With a good heuristic the number of expanded nodes can be dramatically smaller, but in practice **memory**, not time, is usually what runs out first, because the open set holds the whole frontier.

When memory is the bottleneck, look at the variants: **IDA\*** (iterative deepening on `f`, linear memory), **SMA\*** (memory-bounded), and **Jump Point Search**, which prunes symmetric paths on uniform-cost grids.

## 8. Tie-breaking

On grids, ties in `f` are everywhere. In our trace, iterations 9 to 13 all had `f = 9`, and so did several nodes that were never expanded. How you break ties does not affect *correctness*, because any tie-breaking returns an optimal path with an admissible heuristic. It does affect **how many nodes are expanded** and **which of several equally optimal paths you get**.

### Prefer smaller h (equivalently, larger g)

Among nodes with equal `f`, expanding the one with smaller `h` means preferring the node that is *deeper* along its path, closer to the goal. The search then behaves like depth-first search *inside* a tie group: it dives toward the goal instead of fanning out sideways. In our trace, this rule kept A* running along the bottom row in iterations 9 to 13 instead of interleaving `(0,0)`, `(4,0)` and `(4,1)`. Without it, a FIFO or arbitrary order could expand all five `f = 9` nodes at each level before making progress.

On an open grid with no obstacles this effect is large. Every node inside the rectangle spanned by the start and the goal has the same `f`, so poor tie-breaking can expand the entire rectangle, while "smaller h first" expands little more than the path itself.

### Implementation options

1. **Composite priority key:** push `(f, h, counter, node)` into the heap. Python compares tuples left to right, so `f` decides first, then `h`, then the insertion counter. The counter also stops the heap from ever comparing two node objects directly. This is the approach used in section 9.
2. **Nudge the heuristic:** multiply `h` by `(1 + ε)` for a tiny `ε`, such as `1 / (expected maximum path length)`. The slightly inflated `h` breaks ties toward the goal. Strictly speaking, this makes `h` inadmissible by a hair. The path stays optimal only while `ε · h` is smaller than the smallest cost difference between distinct paths, which is easy to guarantee with integer costs.
3. **Cross-product nudge:** add a tiny penalty based on how far a node strays from the straight line between start and goal. This gives straighter-looking paths among equal-cost options. It is popular in games for purely visual reasons.

### Last-in versus first-in

For the final tie level (equal `f` and equal `h`), LIFO tends to keep extending the most recently discovered branch, and FIFO spreads effort more evenly. Neither changes optimality. Pick one deliberately so results are reproducible. Nondeterministic tie-breaking makes debugging miserable.

## 9. Implementation

A complete Python implementation for grids. It uses `heapq`, lazy deletion and tie-breaking on `(f, h, counter)`. It supports 4-connected and 8-connected movement with matching heuristics.

```python
import heapq
import math
from typing import Callable, Dict, List, Optional, Tuple

Cell = Tuple[int, int]

def manhattan(a: Cell, b: Cell) -> float:
    return abs(a[0] - b[0]) + abs(a[1] - b[1])

def euclidean(a: Cell, b: Cell) -> float:
    return math.hypot(a[0] - b[0], a[1] - b[1])

def octile(a: Cell, b: Cell) -> float:
    dx, dy = abs(a[0] - b[0]), abs(a[1] - b[1])
    return max(dx, dy) + (math.sqrt(2) - 1) * min(dx, dy)

STEPS_4 = [(-1, 0, 1.0), (0, 1, 1.0), (1, 0, 1.0), (0, -1, 1.0)]
STEPS_8 = STEPS_4 + [(-1, -1, math.sqrt(2)), (-1, 1, math.sqrt(2)),
                     (1, -1, math.sqrt(2)), (1, 1, math.sqrt(2))]

def a_star(
    grid: List[str],
    start: Cell,
    goal: Cell,
    heuristic: Callable[[Cell, Cell], float] = manhattan,
    diagonal: bool = False,
) -> Tuple[Optional[List[Cell]], float, int]:
    """Return (path, cost, nodes_expanded). path is None if unreachable."""
    rows, cols = len(grid), len(grid[0])
    steps = STEPS_8 if diagonal else STEPS_4

    def passable(r: int, c: int) -> bool:
        return 0 <= r < rows and 0 <= c < cols and grid[r][c] != "#"

    g: Dict[Cell, float] = {start: 0.0}
    parent: Dict[Cell, Optional[Cell]] = {start: None}
    counter = 0
    h0 = heuristic(start, goal)
    open_heap = [(h0, h0, counter, start)]   # (f, h, tiebreak, node)
    closed = set()
    expanded = 0

    while open_heap:
        f, h, _, node = heapq.heappop(open_heap)
        if node in closed:
            continue                          # stale duplicate entry
        if node == goal:
            path = []
            while node is not None:
                path.append(node)
                node = parent[node]
            return path[::-1], g[goal], expanded

        closed.add(node)
        expanded += 1
        r, c = node
        for dr, dc, cost in steps:
            nr, nc = r + dr, c + dc
            if not passable(nr, nc):
                continue
            # Forbid squeezing diagonally between two walls.
            if dr and dc and not (passable(r + dr, c) and passable(r, c + dc)):
                continue
            nxt = (nr, nc)
            if nxt in closed:
                continue
            tentative = g[node] + cost
            if tentative < g.get(nxt, math.inf):
                g[nxt] = tentative
                parent[nxt] = node
                nh = heuristic(nxt, goal)
                counter += 1
                heapq.heappush(open_heap, (tentative + nh, nh, counter, nxt))

    return None, math.inf, expanded


if __name__ == "__main__":
    grid = [
        "......",
        ".####.",
        "S...#G",
        "...#..",
        "......",
    ]
    start, goal = (2, 0), (2, 5)
    for name, h in [("zero", lambda a, b: 0), ("euclidean", euclidean),
                    ("manhattan", manhattan)]:
        path, cost, n = a_star(grid, start, goal, h)
        print(f"{name:10s} cost={cost:.0f} expanded={n} path={path}")
```

Running it prints the numbers from the table in section 4: Dijkstra (the zero heuristic) expands 23 nodes, Euclidean 20, Manhattan 13, and all three return cost 9.

### Notes on the code

- **Lazy deletion.** A node may sit in the heap several times with different priorities. The `if node in closed: continue` line discards outdated entries. This is simpler and usually faster than a heap with decrease-key.
- **Closed-set skip.** Skipping closed neighbours is correct because all three heuristics are consistent for the movement model they are used with (Manhattan with `diagonal=False` only). If you plug in an inconsistent heuristic, remove that check and let improved `g` values re-open nodes.
- **Corner cutting.** The diagonal check stops paths from slipping between two diagonally touching walls. Whether that is allowed depends on your domain, but agents squeezing through solid corners usually looks wrong.
- **Generic graphs.** To use this on a non-grid graph, replace `steps` and `passable` with a `neighbours(node)` function that yields `(next_node, cost)` pairs. Nothing else changes.

## 10. Practical pitfalls

1. **Testing for the goal on push instead of pop.** This returns the first path found, not the cheapest one.
2. **A heuristic that does not match the move set.** Manhattan with diagonal moves overestimates and silently breaks optimality.
3. **Scale mismatch.** If edge costs are in seconds and the heuristic is in metres, `h` is meaningless. Convert distance to cost with the best-case rate, for example distance divided by the top speed.
4. **Negative edge costs.** A* inherits Dijkstra's requirement that costs be non-negative. The proofs above rely on it.
5. **Floating-point ties.** With Euclidean or octile distances, values that should be equal can differ in the last bit. Round, or use integer costs scaled by a constant (for example 10 and 14 for straight and diagonal moves), when reproducibility matters.
6. **Unreachable goals.** Without an early exit, A* explores the whole reachable component before reporting failure. On huge maps, precompute connected regions so impossible queries are rejected in O(1).

## 11. Summary

- A* expands nodes in order of `f = g + h`: the known cost so far plus an estimate of the cost remaining.
- If `h` never overestimates (admissible), the first time the goal is **popped** its path is optimal.
- If `h` also satisfies the triangle inequality (consistent), `f` never decreases along a path, closed nodes never need re-opening, and A* is optimally efficient for that heuristic.
- Match the heuristic to the movement model: Manhattan for 4-connected grids, octile or Chebyshev for 8-connected grids, Euclidean for any-angle movement. A tighter admissible heuristic means fewer expansions.
- Dijkstra is A* with `h = 0`: optimal but undirected. Greedy best-first uses `h` alone: fast but not optimal. A* combines the two.
- Break ties toward smaller `h` to avoid flooding plateaus of equal `f`, and break them deterministically so results are reproducible.

On the example map, the difference is concrete: the same optimal 9-step path costs 23 expansions with Dijkstra, 20 with a mismatched heuristic, and 13 with the right one.
