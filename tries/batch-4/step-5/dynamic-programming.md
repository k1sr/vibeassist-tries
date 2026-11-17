# Dynamic Programming, Explained

Dynamic programming (DP) is a way to solve problems by breaking them into smaller subproblems, solving each subproblem **once**, and reusing those answers. The name, coined by Richard Bellman in the 1950s, is famously unhelpful: "programming" here means planning or tabulating, not writing code, and "dynamic" was chosen partly because it sounded impressive. A better mental label is **"recursion plus a cache"** or **"filling in a table in the right order."**

This explainer covers the idea from the ground up: when DP applies, the two ways to implement it (memoization and tabulation), and three classic problems worked through with fully filled tables: 0/1 knapsack, longest common subsequence, and edit distance. Each example ends with how to recover the actual solution, not just its value, from the table.

---

## 1. When does dynamic programming apply?

A problem is a good DP candidate when it has two properties.

**Optimal substructure.** An optimal solution to the whole problem can be built from optimal solutions to subproblems. For shortest paths: if the shortest route from A to C passes through B, then the A-to-B part must itself be a shortest route from A to B. Otherwise you could swap in a shorter A-to-B route and improve the whole thing.

**Overlapping subproblems.** A naive recursive solution solves the same subproblems again and again. This is what separates DP from plain divide and conquer. Merge sort splits an array into halves that never overlap, so caching buys nothing. Computing Fibonacci numbers recursively asks for `fib(3)` many times over, so caching saves a great deal.

If a problem has optimal substructure but no overlap, use divide and conquer. If it has overlap but the locally best choice is always globally best, a greedy algorithm may be simpler. DP is for the case where you need to **consider several choices at each step** and **those choices lead to shared subproblems**.

### The recipe

Almost every DP solution follows the same five steps:

1. **Define the state.** Decide what a subproblem is, described by a small set of parameters, such as "the best value using the first `i` items with capacity `c`."
2. **Write the recurrence.** Express the answer for a state in terms of answers for "smaller" states.
3. **Identify base cases.** Find the states whose answers are known directly.
4. **Choose an evaluation order.** Make sure every state is computed after the states it depends on (for tabulation), or let recursion handle it (for memoization).
5. **Extract the answer**, and if needed **reconstruct the solution** by retracing the choices that produced it.

Step 1 is where most of the difficulty lives. A good state definition makes the recurrence almost write itself.

---

## 2. A warm-up: Fibonacci

The Fibonacci sequence is defined by `F(0) = 0`, `F(1) = 1`, and `F(n) = F(n-1) + F(n-2)`. The direct recursive translation looks like this:

```python
def fib(n):
    if n < 2:
        return n
    return fib(n - 1) + fib(n - 2)
```

This is correct but exponentially slow. The call tree for `fib(5)` looks like this:

```
fib(5)
├── fib(4)
│   ├── fib(3)
│   │   ├── fib(2)
│   │   │   ├── fib(1)
│   │   │   └── fib(0)
│   │   └── fib(1)
│   └── fib(2)
│       ├── fib(1)
│       └── fib(0)
└── fib(3)
    ├── fib(2)
    │   ├── fib(1)
    │   └── fib(0)
    └── fib(1)
```

`fib(3)` is computed twice and `fib(2)` three times. For `fib(50)` the number of calls exceeds 40 billion. The number of *distinct* subproblems, though, is only 51. That gap between total calls and distinct subproblems is the signature of overlapping subproblems, and it is exactly what DP removes.

---

## 3. Memoization vs. tabulation

There are two standard ways to implement a DP solution. They compute the same values; they differ in direction and in the mechanics.

### Memoization (top-down)

Keep the natural recursive structure, but store each result the first time it is computed and return the stored value on later calls.

```python
from functools import lru_cache

@lru_cache(maxsize=None)
def fib_memo(n):
    if n < 2:
        return n
    return fib_memo(n - 1) + fib_memo(n - 2)
```

Or with an explicit dictionary, which makes the mechanism visible:

```python
def fib_memo(n, memo=None):
    if memo is None:
        memo = {}
    if n < 2:
        return n
    if n not in memo:
        memo[n] = fib_memo(n - 1, memo) + fib_memo(n - 2, memo)
    return memo[n]
```

Each value of `n` is now computed once, so the running time drops from exponential to O(n).

### Tabulation (bottom-up)

Start from the base cases and fill a table in an order that guarantees each entry's dependencies are already filled.

```python
def fib_tab(n):
    if n < 2:
        return n
    table = [0] * (n + 1)
    table[1] = 1
    for i in range(2, n + 1):
        table[i] = table[i - 1] + table[i - 2]
    return table[n]
```

The table for `n = 8`:

| i      | 0 | 1 | 2 | 3 | 4 | 5 | 6 | 7  | 8  |
|--------|---|---|---|---|---|---|---|----|----|
| F(i)   | 0 | 1 | 1 | 2 | 3 | 5 | 8 | 13 | 21 |

Because each entry depends only on the previous two, the table can be shrunk to two variables, giving O(1) space:

```python
def fib_two_vars(n):
    a, b = 0, 1
    for _ in range(n):
        a, b = b, a + b
    return a
```

This **space optimization** is a common final step for tabulated solutions: once you know which earlier rows or cells a cell depends on, you only need to keep those.

### Choosing between them

| Aspect | Memoization (top-down) | Tabulation (bottom-up) |
|---|---|---|
| Code shape | Mirrors the recurrence; often easiest to write first | Requires working out an explicit fill order |
| Subproblems solved | Only those actually reachable from the original problem | Usually every state in the table |
| Overhead | Function calls and hash lookups | Tight loops over arrays; usually faster in practice |
| Recursion depth | Can overflow the stack for deep problems (Python's default limit is about 1,000) | No recursion |
| Space optimization | Hard, since the cache holds everything | Easy to keep just the last row or two |
| Reconstruction | Re-run choices against the cache | Trace back through the table |

A practical approach is to write the memoized version first to confirm the recurrence is right, then convert to tabulation if performance, stack depth or memory matters.

---

## 4. Worked example: 0/1 knapsack

### The problem

You have a knapsack that holds at most `W` units of weight, and `n` items, each with a weight and a value. Each item is either taken whole or left behind (hence "0/1"; you cannot take fractions). Maximize the total value without exceeding the weight limit.

Concrete input, capacity **W = 7**:

| Item | Weight | Value |
|------|--------|-------|
| A    | 1      | 1     |
| B    | 3      | 4     |
| C    | 4      | 5     |
| D    | 5      | 7     |

A greedy strategy fails here. Picking by best value-to-weight ratio takes D first (7/5 = 1.4), then A, leaving capacity 1 with nothing else that fits: total value 8. The optimum, as we will see, is 9.

### State and recurrence

Let `dp[i][c]` be the best value achievable using only the first `i` items with capacity `c`.

For item `i` (weight `w`, value `v`) there are two choices:

- **Skip it:** the value is `dp[i-1][c]`.
- **Take it** (only if `w <= c`): the value is `dp[i-1][c - w] + v`.

```
dp[i][c] = dp[i-1][c]                              if w_i > c
dp[i][c] = max(dp[i-1][c], dp[i-1][c - w_i] + v_i) otherwise
```

Base cases: `dp[0][c] = 0` for every `c` (no items means no value), and `dp[i][0] = 0`.

### The filled table

Rows are items considered so far; columns are capacities 0 through 7.

| Items considered     | c=0 | c=1 | c=2 | c=3 | c=4 | c=5 | c=6 | c=7 |
|----------------------|-----|-----|-----|-----|-----|-----|-----|-----|
| none (i=0)           | 0   | 0   | 0   | 0   | 0   | 0   | 0   | 0   |
| + A (w=1, v=1)       | 0   | 1   | 1   | 1   | 1   | 1   | 1   | 1   |
| + B (w=3, v=4)       | 0   | 1   | 1   | 4   | 5   | 5   | 5   | 5   |
| + C (w=4, v=5)       | 0   | 1   | 1   | 4   | 5   | 6   | 6   | **9** |
| + D (w=5, v=7)       | 0   | 1   | 1   | 4   | 5   | 7   | 8   | **9** |

A few cells worked out by hand:

- **Row B, c=4:** skip gives `dp[A][4] = 1`; take gives `dp[A][4-3] + 4 = 1 + 4 = 5`. Max is 5 (items A and B).
- **Row C, c=7:** skip gives `dp[B][7] = 5`; take gives `dp[B][7-4] + 5 = 4 + 5 = 9`. Max is 9.
- **Row D, c=6:** skip gives `dp[C][6] = 6`; take gives `dp[C][1] + 7 = 1 + 7 = 8`. Max is 8 (items A and D).
- **Row D, c=7:** skip gives `dp[C][7] = 9`; take gives `dp[C][2] + 7 = 1 + 7 = 8`. Max is 9, so D is *not* taken.

The answer is the bottom-right cell: **9**.

### Reconstructing the chosen items

The table stores values, not decisions, but the decisions can be recovered by walking backwards. Starting at `(i = n, c = W)`: if `dp[i][c] == dp[i-1][c]`, item `i` was skipped; otherwise it was taken, and capacity drops by its weight.

| Step | Cell        | Compare with row above | Decision | Remaining capacity |
|------|-------------|------------------------|----------|--------------------|
| 1    | D, c=7 → 9  | C, c=7 → 9 (equal)     | skip D   | 7                  |
| 2    | C, c=7 → 9  | B, c=7 → 5 (differs)   | take C   | 7 − 4 = 3          |
| 3    | B, c=3 → 4  | A, c=3 → 1 (differs)   | take B   | 3 − 3 = 0          |
| 4    | A, c=0 → 0  | none, c=0 → 0 (equal)  | skip A   | 0                  |

Chosen items: **B and C**, total weight 7, total value 9.

### Code

```python
def knapsack(weights, values, capacity):
    n = len(weights)
    dp = [[0] * (capacity + 1) for _ in range(n + 1)]

    for i in range(1, n + 1):
        w, v = weights[i - 1], values[i - 1]
        for c in range(capacity + 1):
            dp[i][c] = dp[i - 1][c]                      # skip item i
            if w <= c:
                dp[i][c] = max(dp[i][c], dp[i - 1][c - w] + v)  # take item i

    # Reconstruct
    chosen, c = [], capacity
    for i in range(n, 0, -1):
        if dp[i][c] != dp[i - 1][c]:
            chosen.append(i - 1)
            c -= weights[i - 1]
    chosen.reverse()
    return dp[n][capacity], chosen


best, items = knapsack([1, 3, 4, 5], [1, 4, 5, 7], 7)
print(best, items)   # 9 [1, 2]  -> items B and C
```

Time and space are both O(n · W). Note that this is **pseudo-polynomial**: it is polynomial in the numeric value of `W`, not in the number of bits needed to write `W` down. Knapsack is NP-hard in general; DP makes it fast only when capacities are reasonably small integers.

**Space optimization.** Each row depends only on the row above, so a single 1-D array suffices, provided you iterate capacity **downwards** so that `dp[c - w]` still holds the previous row's value:

```python
def knapsack_1d(weights, values, capacity):
    dp = [0] * (capacity + 1)
    for w, v in zip(weights, values):
        for c in range(capacity, w - 1, -1):
            dp[c] = max(dp[c], dp[c - w] + v)
    return dp[capacity]
```

Iterating upwards instead would let an item be used more than once, which turns this into the *unbounded* knapsack problem. The loop direction alone changes which problem you solve. The trade-off of the 1-D version is that you lose the full table, so reconstruction needs extra bookkeeping (for example, a separate boolean `took[i][c]` table).

---

## 5. Worked example: longest common subsequence

### The problem

A **subsequence** is what remains after deleting zero or more characters from a string without reordering the rest. "ACE" is a subsequence of "ABCDE"; "AEC" is not. Given two strings, find the longest sequence that is a subsequence of both. LCS underlies `diff` tools, DNA sequence comparison and version-control merges.

Concrete input:

- X = `ABCBDAB` (length 7)
- Y = `BDCABA` (length 6)

### State and recurrence

Let `L[i][j]` be the length of the LCS of the first `i` characters of X and the first `j` characters of Y.

```
L[i][j] = 0                              if i == 0 or j == 0
L[i][j] = L[i-1][j-1] + 1                if X[i] == Y[j]
L[i][j] = max(L[i-1][j], L[i][j-1])      otherwise
```

The reasoning: if the last characters match, that character can end the common subsequence, so extend the best answer for both shorter prefixes. If they do not match, at least one of the two last characters is not in the LCS, so drop one or the other and keep whichever is better.

### The filled table

Rows are prefixes of X, columns are prefixes of Y. Cells where the characters match (and the diagonal +1 rule fired) are marked with `↖`.

|       | ∅ | B   | D   | C   | A   | B   | A   |
|-------|---|-----|-----|-----|-----|-----|-----|
| **∅** | 0 | 0   | 0   | 0   | 0   | 0   | 0   |
| **A** | 0 | 0   | 0   | 0   | ↖1  | 1   | ↖1  |
| **B** | 0 | ↖1  | 1   | 1   | 1   | ↖2  | 2   |
| **C** | 0 | 1   | 1   | ↖2  | 2   | 2   | 2   |
| **B** | 0 | ↖1  | 1   | 2   | 2   | ↖3  | 3   |
| **D** | 0 | 1   | ↖2  | 2   | 2   | 3   | 3   |
| **A** | 0 | 1   | 2   | 2   | ↖3  | 3   | ↖4  |
| **B** | 0 | ↖1  | 2   | 2   | 3   | ↖4  | **4** |

Some cells worked out:

- **Row 2 (B), column 5 (B):** match, so `L[1][4] + 1 = 1 + 1 = 2`.
- **Row 3 (C), column 3 (C):** match, so `L[2][2] + 1 = 1 + 1 = 2`.
- **Row 5 (D), column 5 (B):** no match, so `max(L[4][5], L[5][4]) = max(3, 2) = 3`.
- **Row 6 (A), column 6 (A):** match, so `L[5][5] + 1 = 3 + 1 = 4`.

The LCS length is the bottom-right cell: **4**.

### Reconstructing the subsequence

Start at `(7, 6)` and walk back:

- If the characters match, that character is in the LCS: record it and move diagonally up-left.
- Otherwise move toward the larger neighbour: up if `L[i-1][j] >= L[i][j-1]`, else left.

| Position (i, j) | X[i] | Y[j] | Action                          | Collected |
|-----------------|------|------|---------------------------------|-----------|
| (7, 6)          | B    | A    | up (4) ≥ left (4) → go up       |           |
| (6, 6)          | A    | A    | match → take A, go diagonal     | A         |
| (5, 5)          | D    | B    | up (3) > left (2) → go up       | A         |
| (4, 5)          | B    | B    | match → take B, go diagonal     | B A       |
| (3, 4)          | C    | A    | up (1) < left (2) → go left     | B A       |
| (3, 3)          | C    | C    | match → take C, go diagonal     | C B A     |
| (2, 2)          | B    | D    | up (0) < left (1) → go left     | C B A     |
| (2, 1)          | B    | B    | match → take B, go diagonal     | B C B A   |
| (1, 0)          | —    | —    | reached edge, stop              |           |

The characters were collected back to front, so reading them forward gives **`BCBA`**. Check: B-C-B-A appears in order in `ABCBDAB` (positions 2, 3, 4, 6) and in `BDCABA` (positions 1, 3, 5, 6).

LCS is not always unique. Here `BDAB` and `BCAB` are also valid length-4 answers. Which one the traceback finds depends on how ties are broken (preferring "left" over "up" on ties would produce a different string). If you need *all* LCSs, you branch on ties, which can be exponential in the worst case.

### Code

```python
def lcs(x, y):
    m, n = len(x), len(y)
    L = [[0] * (n + 1) for _ in range(m + 1)]

    for i in range(1, m + 1):
        for j in range(1, n + 1):
            if x[i - 1] == y[j - 1]:
                L[i][j] = L[i - 1][j - 1] + 1
            else:
                L[i][j] = max(L[i - 1][j], L[i][j - 1])

    # Reconstruct
    out, i, j = [], m, n
    while i > 0 and j > 0:
        if x[i - 1] == y[j - 1]:
            out.append(x[i - 1])
            i, j = i - 1, j - 1
        elif L[i - 1][j] >= L[i][j - 1]:
            i -= 1
        else:
            j -= 1
    return L[m][n], "".join(reversed(out))


print(lcs("ABCBDAB", "BDCABA"))   # (4, 'BCBA')
```

Time and space are O(m · n). If you only need the length, two rows suffice, giving O(min(m, n)) space. Recovering the actual subsequence in linear space is possible with Hirschberg's algorithm, a divide-and-conquer refinement that keeps O(m · n) time.

---

## 6. Worked example: edit distance

### The problem

The **edit distance** (Levenshtein distance) between two strings is the minimum number of single-character operations needed to turn one into the other, where the allowed operations are:

- **insert** a character,
- **delete** a character,
- **substitute** one character for another.

Each operation costs 1. Edit distance powers spell checkers ("did you mean...?"), fuzzy search and DNA alignment.

Concrete input: transform **`kitten`** into **`sitting`**.

### State and recurrence

Let `D[i][j]` be the edit distance between the first `i` characters of the source `a` and the first `j` characters of the target `b`.

```
D[i][0] = i          (delete all i characters)
D[0][j] = j          (insert all j characters)

D[i][j] = D[i-1][j-1]                       if a[i] == b[j]   (no cost)
D[i][j] = 1 + min( D[i-1][j],               delete a[i]
                   D[i][j-1],               insert b[j]
                   D[i-1][j-1] )            substitute a[i] -> b[j]
                                            otherwise
```

Each neighbour corresponds to an operation: moving **down** (from `i-1` to `i`) consumes a source character without producing a target character, which is a deletion. Moving **right** produces a target character without consuming a source character, which is an insertion. Moving **diagonally** consumes one of each: free if they match, a substitution if not.

### The filled table

Rows are prefixes of `kitten`, columns are prefixes of `sitting`.

|       | ∅ | s | i | t | t | i | n | g     |
|-------|---|---|---|---|---|---|---|-------|
| **∅** | 0 | 1 | 2 | 3 | 4 | 5 | 6 | 7     |
| **k** | 1 | 1 | 2 | 3 | 4 | 5 | 6 | 7     |
| **i** | 2 | 2 | 1 | 2 | 3 | 4 | 5 | 6     |
| **t** | 3 | 3 | 2 | 1 | 2 | 3 | 4 | 5     |
| **t** | 4 | 4 | 3 | 2 | 1 | 2 | 3 | 4     |
| **e** | 5 | 5 | 4 | 3 | 2 | 2 | 3 | 4     |
| **n** | 6 | 6 | 5 | 4 | 3 | 3 | 2 | **3** |

Some cells worked out:

- **(k, s):** no match. `1 + min(D[0][1]=1, D[1][0]=1, D[0][0]=0) = 1`, via substitution k → s.
- **(i, i):** match, so copy the diagonal: `D[1][1] = 1`.
- **(e, i)** at row 5, column 5: no match. `1 + min(up D[4][5]=2, left D[5][4]=2, diagonal D[4][4]=1) = 2`, via substitution e → i.
- **(n, n)** at row 6, column 6: match, so copy the diagonal `D[5][5] = 2`.
- **(n, g)** at row 6, column 7: no match. `1 + min(up D[5][7]=4, left D[6][6]=2, diagonal D[5][6]=3) = 3`, via insertion of g.

The edit distance is the bottom-right cell: **3**.

### Reconstructing the edit script

Walk back from `(6, 7)`, at each step choosing a neighbour that could have produced the current value:

| Position (i, j) | a[i] | b[j] | Value | Came from              | Operation          |
|-----------------|------|------|-------|------------------------|--------------------|
| (6, 7)          | n    | g    | 3     | left (6, 6) = 2        | insert `g`         |
| (6, 6)          | n    | n    | 2     | diagonal (5, 5) = 2    | keep `n`           |
| (5, 5)          | e    | i    | 2     | diagonal (4, 4) = 1    | substitute `e`→`i` |
| (4, 4)          | t    | t    | 1     | diagonal (3, 3) = 1    | keep `t`           |
| (3, 3)          | t    | t    | 1     | diagonal (2, 2) = 1    | keep `t`           |
| (2, 2)          | i    | i    | 1     | diagonal (1, 1) = 1    | keep `i`           |
| (1, 1)          | k    | s    | 1     | diagonal (0, 0) = 0    | substitute `k`→`s` |

Read forward, the edit script is:

1. **k**itten → **s**itten (substitute k → s)
2. sitt**e**n → sitt**i**n (substitute e → i)
3. sittin → sittin**g** (insert g)

Three operations, matching the table.

### Code

```python
def edit_distance(a, b):
    m, n = len(a), len(b)
    D = [[0] * (n + 1) for _ in range(m + 1)]
    for i in range(m + 1):
        D[i][0] = i
    for j in range(n + 1):
        D[0][j] = j

    for i in range(1, m + 1):
        for j in range(1, n + 1):
            if a[i - 1] == b[j - 1]:
                D[i][j] = D[i - 1][j - 1]
            else:
                D[i][j] = 1 + min(D[i - 1][j],      # delete
                                  D[i][j - 1],      # insert
                                  D[i - 1][j - 1])  # substitute

    # Reconstruct the edit script
    ops, i, j = [], m, n
    while i > 0 or j > 0:
        if i > 0 and j > 0 and a[i - 1] == b[j - 1] and D[i][j] == D[i - 1][j - 1]:
            ops.append(f"keep {a[i - 1]}")
            i, j = i - 1, j - 1
        elif i > 0 and j > 0 and D[i][j] == D[i - 1][j - 1] + 1:
            ops.append(f"substitute {a[i - 1]} -> {b[j - 1]}")
            i, j = i - 1, j - 1
        elif j > 0 and D[i][j] == D[i][j - 1] + 1:
            ops.append(f"insert {b[j - 1]}")
            j -= 1
        else:
            ops.append(f"delete {a[i - 1]}")
            i -= 1
    ops.reverse()
    return D[m][n], ops


dist, script = edit_distance("kitten", "sitting")
print(dist)            # 3
for op in script:
    print(op)
# substitute k -> s
# keep i
# keep t
# keep t
# substitute e -> i
# keep n
# insert g
```

Time and space are O(m · n); as with LCS, the distance alone needs only two rows.

### The connection to LCS

If substitutions are disallowed (only insertions and deletions), the edit distance equals `len(a) + len(b) − 2 · LCS(a, b)`. Every character not in the common subsequence must be deleted from one side or inserted on the other. The two problems are close relatives, and their tables have the same shape and fill order.

---

## 7. Reconstructing solutions: the general pattern

All three examples follow the same pattern for recovering the actual answer:

1. **Fill the table of optimal values.**
2. **Start at the cell holding the final answer.**
3. **Ask which choice produced this value.** Recompute each option of the recurrence and see which one equals the stored value.
4. **Record that choice and move to the cell it came from.**
5. **Repeat until you reach a base case**, then reverse the recorded choices if you collected them back to front.

Two implementation styles are common:

- **Recompute on the way back**, as in all three examples above. This needs no extra memory, but the traceback must mirror the recurrence exactly, including the tie-breaking rule.
- **Store a parent pointer** for each cell during the fill, recording which option won (for example `'D'`, `'L'`, `'U'` for diagonal, left, up). The traceback then just follows pointers. This costs extra memory but is simpler and less error-prone for complicated recurrences.

```python
# Parent-pointer variant for LCS
def lcs_with_parents(x, y):
    m, n = len(x), len(y)
    L = [[0] * (n + 1) for _ in range(m + 1)]
    P = [[None] * (n + 1) for _ in range(m + 1)]
    for i in range(1, m + 1):
        for j in range(1, n + 1):
            if x[i - 1] == y[j - 1]:
                L[i][j], P[i][j] = L[i - 1][j - 1] + 1, "D"
            elif L[i - 1][j] >= L[i][j - 1]:
                L[i][j], P[i][j] = L[i - 1][j], "U"
            else:
                L[i][j], P[i][j] = L[i][j - 1], "L"

    out, i, j = [], m, n
    while i > 0 and j > 0:
        move = P[i][j]
        if move == "D":
            out.append(x[i - 1]); i -= 1; j -= 1
        elif move == "U":
            i -= 1
        else:
            j -= 1
    return "".join(reversed(out))
```

The key point: **space optimizations that discard old rows also discard the information needed for reconstruction.** If you need the solution and not just its value, either keep the full table, keep a compact decision table, or use a divide-and-conquer technique such as Hirschberg's.

---

## 8. Memoization on a 2-D problem

Memoization is not limited to one-parameter problems. Here is edit distance written top-down. It is often the quickest way to get a correct solution before worrying about fill order:

```python
from functools import lru_cache

def edit_distance_memo(a, b):
    @lru_cache(maxsize=None)
    def solve(i, j):
        if i == 0:
            return j
        if j == 0:
            return i
        if a[i - 1] == b[j - 1]:
            return solve(i - 1, j - 1)
        return 1 + min(solve(i - 1, j),
                       solve(i, j - 1),
                       solve(i - 1, j - 1))
    return solve(len(a), len(b))


print(edit_distance_memo("kitten", "sitting"))   # 3
```

The cache plays the role of the table. The recursion visits cells in whatever order the dependencies demand, so you never have to think about loop order. The costs are recursion depth (up to `m + n` frames here) and per-call overhead.

---

## 9. Common pitfalls

- **A state that is missing information.** If the best decision at a state depends on something the state does not record, the recurrence is wrong. In knapsack, "best value with the first `i` items" alone is not enough; remaining capacity must be part of the state.
- **Wrong fill order.** In tabulation, reading a cell before it is written silently yields zeros. Draw arrows from each cell to its dependencies and make sure your loops respect them.
- **Off-by-one errors between table indices and string indices.** The tables above have an extra row and column for the empty prefix, so `dp[i]` corresponds to `s[i-1]`. Be consistent.
- **Loop direction in 1-D optimizations.** As shown for knapsack, iterating capacity upwards versus downwards changes the problem being solved.
- **Mutable default arguments in memoized Python functions.** `def f(n, memo={})` shares the cache across unrelated calls. Use `None` and create the dictionary inside, or use `functools.lru_cache`.
- **Unhashable arguments with `lru_cache`.** Lists cannot be cached; pass tuples or indices instead.
- **Assuming DP makes everything polynomial.** The table size is the product of the state dimensions. If one dimension is a large number (like knapsack capacity) or a subset (like the travelling salesman's visited set, with 2ⁿ states), the "efficient" DP can still be enormous.

---

## 10. Where to go next

Once the pattern is familiar, the same recipe covers a large family of problems:

- **Coin change:** fewest coins to make an amount, or the number of ways to make it.
- **Longest increasing subsequence:** O(n²) DP, or O(n log n) with patience sorting.
- **Matrix chain multiplication** and other **interval DP** problems, where the state is a range `[i, j]`.
- **Shortest paths:** Bellman-Ford and Floyd-Warshall are both dynamic programs.
- **DP on trees:** maximum independent set, tree diameter.
- **Bitmask DP:** travelling salesman over small sets, with states such as `(visited_set, current_city)`.
- **Sequence alignment** in bioinformatics (Needleman-Wunsch, Smith-Waterman), which generalizes edit distance with weighted scores.

In every case the work is the same: define a state that captures everything a future decision needs, write the recurrence, get the base cases right, fill the table in dependency order (or let memoization handle it), and trace back through the table when you need the solution itself rather than only its value.
