# Merge Sort, Explained in Full

Merge sort is the standard example of divide and conquer. It sorts a list by splitting it in half, sorting each half, and merging the two sorted halves into one sorted whole. It runs in O(n log n) time in the best, average and worst case. It is **stable**. It suits linked lists and data on disk well. Variants of it sit inside Python's `sorted`, Java's `Arrays.sort` for objects, and most database engines that sort more data than fits in memory.

This explainer covers the idea, a top-down trace, the merge step one comparison at a time, a bottom-up trace, the recurrence and its solution, a stability demo, external merge sort on a concrete file, working code, and the optimizations used in practice.

---

## 1. The core idea

Merge sort rests on two facts:

1. **A list of zero or one element is already sorted.**
2. **Two sorted lists can be merged into one sorted list in linear time.** Look at the front of each list, take the smaller element, and repeat.

The algorithm follows from these:

```
merge_sort(A):
    if length(A) <= 1: return A
    split A into left half L and right half R
    L' = merge_sort(L)
    R' = merge_sort(R)
    return merge(L', R')
```

All the real work happens in `merge`. Splitting is trivial: you compute a midpoint. That is the opposite of quicksort, where partitioning does the work and combining is trivial. Merge sort is "easy split, hard join". Quicksort is "hard split, easy join".

---

## 2. The merge step

`merge` takes two sorted sequences, `L` and `R`, and produces one sorted sequence containing every element of both. It keeps one pointer `i` into `L` and one pointer `j` into `R`:

```
merge(L, R):
    i = j = 0
    out = []
    while i < len(L) and j < len(R):
        if L[i] <= R[j]:          # "<=" matters for stability (section 6)
            out.append(L[i]); i += 1
        else:
            out.append(R[j]); j += 1
    append the rest of L (from i) to out
    append the rest of R (from j) to out
    return out
```

Every comparison outputs exactly one element. Once one side runs out, the rest of the other side is copied over with no further comparisons. So merging lists of lengths `p` and `q`:

- outputs exactly `p + q` elements,
- makes **at most** `p + q − 1` comparisons, which happens when the two lists interleave right to the end,
- makes **at least** `min(p, q)` comparisons, which happens when every element of one list is smaller than every element of the other.

That linear bound is what makes the whole algorithm O(n log n).

---

## 3. Worked example: top-down merge sort

We sort this array:

```
[38, 27, 43, 3, 9, 82, 10]
```

The split uses `mid = len // 2`, the same rule as the code in section 8. A list of 7 splits into 3 on the left and 4 on the right.

### 3.1 The split phase (going down)

```
                    [38, 27, 43, 3, 9, 82, 10]
                   /                          \
           [38, 27, 43]                  [3, 9, 82, 10]
           /         \                   /            \
        [38]      [27, 43]          [3, 9]          [82, 10]
                  /     \           /    \          /      \
               [27]    [43]       [3]    [9]      [82]     [10]
```

The leaves are single elements, which are already sorted. The recursion depth is ⌈log₂ 7⌉ = 3.

### 3.2 The merge phase (coming back up)

Merges finish from the bottom up, in the order a depth-first recursion completes them:

| # | Merge | Result | Comparisons |
|---|-------|--------|-------------|
| 1 | `[27]` + `[43]` | `[27, 43]` | 1 |
| 2 | `[38]` + `[27, 43]` | `[27, 38, 43]` | 2 |
| 3 | `[3]` + `[9]` | `[3, 9]` | 1 |
| 4 | `[82]` + `[10]` | `[10, 82]` | 1 |
| 5 | `[3, 9]` + `[10, 82]` | `[3, 9, 10, 82]` | 2 |
| 6 | `[27, 38, 43]` + `[3, 9, 10, 82]` | `[3, 9, 10, 27, 38, 43, 82]` | 6 |
| | | **Total** | **13** |

Merge 2 in detail: compare 38 with 27 and take 27. Compare 38 with 43 and take 38. The left list is now empty, so 43 is copied with no comparison. Two comparisons for three elements.

### 3.3 The final merge, step by step

This is merge 6. `L = [27, 38, 43]` and `R = [3, 9, 10, 82]`. The arrow marks the element each pointer is on.

| Step | L (pointer `i`) | R (pointer `j`) | Compare | Take | Output so far |
|------|-----------------|-----------------|---------|------|---------------|
| 1 | **27**, 38, 43 | **3**, 9, 10, 82 | 27 vs 3 | 3 (R) | 3 |
| 2 | **27**, 38, 43 | 3, **9**, 10, 82 | 27 vs 9 | 9 (R) | 3, 9 |
| 3 | **27**, 38, 43 | 3, 9, **10**, 82 | 27 vs 10 | 10 (R) | 3, 9, 10 |
| 4 | **27**, 38, 43 | 3, 9, 10, **82** | 27 vs 82 | 27 (L) | 3, 9, 10, 27 |
| 5 | 27, **38**, 43 | 3, 9, 10, **82** | 38 vs 82 | 38 (L) | 3, 9, 10, 27, 38 |
| 6 | 27, 38, **43** | 3, 9, 10, **82** | 43 vs 82 | 43 (L) | 3, 9, 10, 27, 38, 43 |
| 7 | *(exhausted)* | 3, 9, 10, **82** | none | 82 (copy) | 3, 9, 10, 27, 38, 43, 82 |

Six comparisons for seven elements, which is the `p + q − 1` maximum. The last element always comes free.

The whole sort used 13 comparisons. The worst-case formula in section 5.3 gives 14 for n = 7, so this input was close to the worst case.

---

## 4. Worked example: bottom-up merge sort

Top-down merge sort uses recursion to find its subproblems. **Bottom-up** merge sort skips the recursion. It treats the array as n runs of length 1, merges neighbouring pairs into runs of length 2, then 4, then 8, doubling the width until a single run covers the array.

### 4.1 A power-of-two example

We sort:

```
[5, 2, 4, 6, 1, 3, 2, 6]
```

| Pass | Width | Merges performed | Array after pass |
|------|-------|------------------|------------------|
| 0 | n/a | (start) | `[5, 2, 4, 6, 1, 3, 2, 6]` |
| 1 | 1 | `[5]+[2]`, `[4]+[6]`, `[1]+[3]`, `[2]+[6]` | `[2, 5, 4, 6, 1, 3, 2, 6]` |
| 2 | 2 | `[2,5]+[4,6]`, `[1,3]+[2,6]` | `[2, 4, 5, 6, 1, 2, 3, 6]` |
| 3 | 4 | `[2,4,5,6]+[1,2,3,6]` | `[1, 2, 2, 3, 4, 5, 6, 6]` |

Three passes, because log₂ 8 = 3. Each pass reads and writes every element once.

Here is the last merge in detail. It contains two ties, which matter for stability:

`L = [2, 4, 5, 6]` and `R = [1, 2, 3, 6]`

| Step | Compare | Rule | Take | Output |
|------|---------|------|------|--------|
| 1 | 2 vs 1 | 1 < 2 | 1 (R) | 1 |
| 2 | 2 vs 2 | tie, so take L | 2 (L) | 1, 2 |
| 3 | 4 vs 2 | 2 < 4 | 2 (R) | 1, 2, 2 |
| 4 | 4 vs 3 | 3 < 4 | 3 (R) | 1, 2, 2, 3 |
| 5 | 4 vs 6 | 4 ≤ 6 | 4 (L) | 1, 2, 2, 3, 4 |
| 6 | 5 vs 6 | 5 ≤ 6 | 5 (L) | 1, 2, 2, 3, 4, 5 |
| 7 | 6 vs 6 | tie, so take L | 6 (L) | 1, 2, 2, 3, 4, 5, 6 |
| 8 | none | L exhausted | 6 (R, copy) | 1, 2, 2, 3, 4, 5, 6, 6 |

Seven comparisons. On a tie we always take from the left, so the 2 that started earlier in the array stays ahead of the 2 that started later. Section 6 builds on this.

### 4.2 A non-power-of-two example

Bottom-up merging handles leftover runs without special code. When a pass has an odd run at the end, that run has no partner and is carried into the next pass unchanged. Here is the array from section 3 again:

| Pass | Width | Runs after pass |
|------|-------|-----------------|
| 0 | n/a | `[38] [27] [43] [3] [9] [82] [10]` |
| 1 | 1 | `[27, 38] [3, 43] [9, 82] [10]` (the run `[10]` has no partner) |
| 2 | 2 | `[3, 27, 38, 43] [9, 10, 82]` |
| 3 | 4 | `[3, 9, 10, 27, 38, 43, 82]` |

The final answer matches section 3, but the subproblems are different. Top-down split 7 into 3 + 4. Bottom-up split it into 4 + 3. Both are correct. Bottom-up's boundaries depend only on powers of two, which makes it easy to write iteratively and well suited to hardware and parallel implementations.

---

## 5. Running time: the recurrence, solved

### 5.1 Setting it up

Let T(n) be the time to merge sort n elements. The algorithm:

- splits in constant time,
- makes two recursive calls on about n/2 elements each,
- merges in time proportional to n.

So

```
T(1) = c₀
T(n) = 2·T(n/2) + c·n        for n > 1
```

Assume for now that n is a power of two, n = 2ᵏ.

### 5.2 Solving by the recursion tree

Expand the recurrence level by level:

| Level | Subproblems | Size of each | Merge cost per subproblem | Total cost at level |
|-------|-------------|--------------|---------------------------|---------------------|
| 0 | 1 | n | c·n | c·n |
| 1 | 2 | n/2 | c·n/2 | c·n |
| 2 | 4 | n/4 | c·n/4 | c·n |
| ... | ... | ... | ... | c·n |
| k−1 | 2ᵏ⁻¹ | 2 | 2c | c·n |
| k (leaves) | n | 1 | c₀ | c₀·n |

Every internal level costs exactly c·n. The number of subproblems doubles at each level and their size halves, so the product stays the same. There are k = log₂ n internal levels, plus the leaves:

```
T(n) = c·n·log₂ n + c₀·n  =  Θ(n log n)
```

### 5.3 Checking by substitution

Unroll the recurrence directly:

```
T(n) = 2T(n/2) + cn
     = 2[2T(n/4) + cn/2] + cn        = 4T(n/4) + 2cn
     = 4[2T(n/8) + cn/4] + 2cn       = 8T(n/8) + 3cn
     ...
     = 2ᵏ·T(n/2ᵏ) + k·cn
```

Set k = log₂ n, so n/2ᵏ = 1:

```
T(n) = n·T(1) + cn·log₂ n = c₀n + cn·log₂ n
```

The **Master Theorem** agrees. With a = 2, b = 2 and f(n) = Θ(n), we get n^(log_b a) = n¹, which matches f(n). That is case 2, giving Θ(n log n).

**Exact comparison count.** The worst-case number of comparisons for merge sort on n elements is

```
W(n) = n·⌈log₂ n⌉ − 2^⌈log₂ n⌉ + 1
```

For n = 8: 8·3 − 8 + 1 = **17**. For n = 7: 7·3 − 8 + 1 = **14**. The 13 comparisons in section 3 are just under that ceiling. The lower bound for any comparison sort is log₂(n!) ≈ n log₂ n − 1.44n, so merge sort is close to optimal in comparisons, not only in asymptotic terms.

### 5.4 Best, average, worst

The split never depends on the data, so the recursion tree has the same shape for every input. The algorithm always does Θ(n log n) work. Sorted input saves some comparisons, because each merge stops comparing as soon as one side empties, but it still copies every element at every level. (Section 9 shows a one-line check that makes sorted input run in linear time.)

### 5.5 Space

The array version needs an auxiliary buffer of n elements for merging. Allocate it once and reuse it, rather than allocating a new one for each merge. Recursion adds O(log n) stack frames. So the extra space is **Θ(n)**. This is merge sort's main cost compared with heapsort (O(1) extra) and quicksort (O(log n) stack). In-place merge algorithms exist, but they are complicated and slower in practice. On linked lists, merging only relinks nodes, so merge sort needs only O(1) extra space beyond the recursion stack.

---

## 6. Stability demo

A sort is **stable** if elements with equal keys keep their original relative order. This matters when you sort by several keys in turn. For example, sort employees by name and then stably by department, and each department stays sorted by name.

Take five records, each a pair of (key, label), and sort them by key only:

```
[(3,a), (1,b), (3,c), (2,d), (1,e)]
```

The label shows each record's original position. Key 1 appears as `b` before `e`. Key 3 appears as `a` before `c`.

**Split** (mid = 2):

```
[(3,a), (1,b)]  |  [(3,c), (2,d), (1,e)]
```

**Left half:** `[(3,a)]` + `[(1,b)]` gives `[(1,b), (3,a)]`

**Right half:** `[(3,c)]` | `[(2,d), (1,e)]`. The inner merge gives `[(1,e), (2,d)]`. Merging that with `[(3,c)]` gives `[(1,e), (2,d), (3,c)]`.

**Final merge** using `<=`, so ties go to the left:

| Step | L head | R head | Compare keys | Take |
|------|--------|--------|--------------|------|
| 1 | (1,b) | (1,e) | 1 vs 1, tie | **(1,b)** from L |
| 2 | (3,a) | (1,e) | 3 vs 1 | (1,e) from R |
| 3 | (3,a) | (2,d) | 3 vs 2 | (2,d) from R |
| 4 | (3,a) | (3,c) | 3 vs 3, tie | **(3,a)** from L |
| 5 | (empty) | (3,c) | none | (3,c) copy |

Result:

```
[(1,b), (1,e), (2,d), (3,a), (3,c)]
```

`b` still comes before `e`, and `a` still comes before `c`. **The sort is stable.**

**Why it works:** every element in L came from earlier in the original array than every element in R. Taking from L on a tie therefore keeps the original order. The subcalls preserve this by induction.

**What happens with the wrong comparison.** Change `L[i] <= R[j]` to `L[i] < R[j]`. At step 1 the test `1 < 1` is false, so the merge takes `(1,e)` first:

```
[(1,e), (1,b), (2,d), (3,c), (3,a)]     # unstable!
```

Both pairs of equal keys are now reversed. One character in the merge decides whether the sort is stable.

---

## 7. External merge sort: sorting more data than fits in memory

Merge sort reads its inputs from front to back and writes its output from front to back. That suits disks, tapes, SSDs and network streams, where sequential access is cheap and random access is expensive. **External merge sort** works in two phases:

1. **Run generation:** read as much data as fits in memory, sort it in memory, and write it out as a sorted **run**. Repeat until all the input has been read.
2. **k-way merging:** merge k runs at a time into longer runs, using one input buffer per run and one output buffer. Repeat until one run remains.

### 7.1 A concrete example

Suppose the file holds 20 records and memory holds only **4 records**. In the merge phase those 4 slots become 3 input buffers and 1 output buffer, so each merge combines **k = 3** runs.

Input file:

```
41  7 93 18 | 66  2 55 30 | 81 14 49 72 |  9 60 25 88 | 37  4 77 51
```

**Pass 0: run generation.** Load 4 records, sort them in memory, write them out:

| Run | Loaded | Written (sorted) |
|-----|--------|------------------|
| R1 | 41 7 93 18 | 7 18 41 93 |
| R2 | 66 2 55 30 | 2 30 55 66 |
| R3 | 81 14 49 72 | 14 49 72 81 |
| R4 | 9 60 25 88 | 9 25 60 88 |
| R5 | 37 4 77 51 | 4 37 51 77 |

**Pass 1: 3-way merge.** Merge R1, R2 and R3 into run A, then R4 and R5 into run B.

A 3-way merge needs the smallest of three heads at each step. A min-heap of size k does this in O(log k) per element. Here is the merge into A, one step at a time. The heap holds the current head of each run.

| Step | Heap (head of R1, R2, R3) | Pop | Refill from | Output A so far |
|------|---------------------------|-----|-------------|-----------------|
| 1 | 7, 2, 14 | 2 | R2 gives 30 | 2 |
| 2 | 7, 30, 14 | 7 | R1 gives 18 | 2 7 |
| 3 | 18, 30, 14 | 14 | R3 gives 49 | 2 7 14 |
| 4 | 18, 30, 49 | 18 | R1 gives 41 | ... 18 |
| 5 | 41, 30, 49 | 30 | R2 gives 55 | ... 30 |
| 6 | 41, 55, 49 | 41 | R1 gives 93 | ... 41 |
| 7 | 93, 55, 49 | 49 | R3 gives 72 | ... 49 |
| 8 | 93, 55, 72 | 55 | R2 gives 66 | ... 55 |
| 9 | 93, 66, 72 | 66 | R2 empty | ... 66 |
| 10 | 93, (none), 72 | 72 | R3 gives 81 | ... 72 |
| 11 | 93, (none), 81 | 81 | R3 empty | ... 81 |
| 12 | 93 | 93 | R1 empty | ... 93 |

```
A = 2 7 14 18 30 41 49 55 66 72 81 93        (12 records)
B = 4 9 25 37 51 60 77 88                    (8 records, merging R4 + R5)
```

**Pass 2: final merge.** Merge A and B:

```
2 4 7 9 14 18 25 30 37 41 49 51 55 60 66 72 77 81 88 93
```

### 7.2 Counting passes

With N records, memory for M records, and k-way merging:

```
initial runs  = ⌈N / M⌉
merge passes  = ⌈log_k ⌈N / M⌉⌉
total passes  = 1 + ⌈log_k ⌈N / M⌉⌉
```

For the example: ⌈20/4⌉ = 5 runs, and ⌈log₃ 5⌉ = 2 merge passes, so 3 passes in total. That matches the trace. Each pass reads and writes the whole file once.

At realistic scale, take 1 TB of data, 10 GB of RAM, and 1 MB buffers. That gives 100 initial runs. With 10 GB of RAM you can hold far more than 100 buffers of 1 MB, so a single 100-way merge pass finishes the job. The whole terabyte is sorted in **2 passes**, which is two sequential reads and two sequential writes. That is why external merge sort is still the default for database `ORDER BY`, index builds, sort-merge joins, and MapReduce shuffles.

Two common refinements:

- **Replacement selection** uses a heap during run generation to produce runs averaging **2M** in length on random input, which halves the number of runs.
- **Double buffering** fills one buffer per run in the background while the merge consumes the other, so the CPU does not wait on I/O.

---

## 8. Code

### 8.1 Top-down (Python, returns a new list)

```python
def merge(left, right):
    out = []
    i = j = 0
    while i < len(left) and j < len(right):
        if left[i] <= right[j]:        # <= keeps it stable
            out.append(left[i]); i += 1
        else:
            out.append(right[j]); j += 1
    out.extend(left[i:])
    out.extend(right[j:])
    return out

def merge_sort(a):
    if len(a) <= 1:
        return a
    mid = len(a) // 2
    return merge(merge_sort(a[:mid]), merge_sort(a[mid:]))

print(merge_sort([38, 27, 43, 3, 9, 82, 10]))
# [3, 9, 10, 27, 38, 43, 82]
```

This version is clear but allocates new lists for every slice. The next version uses one buffer for the whole sort.

### 8.2 Top-down with a single auxiliary buffer (Python)

```python
def merge_sort_inplace(a, key=lambda x: x):
    aux = a[:]                          # one allocation for the whole sort

    def merge(lo, mid, hi):             # merge a[lo:mid] and a[mid:hi]
        aux[lo:hi] = a[lo:hi]
        i, j = lo, mid
        for k in range(lo, hi):
            if i >= mid:
                a[k] = aux[j]; j += 1
            elif j >= hi:
                a[k] = aux[i]; i += 1
            elif key(aux[i]) <= key(aux[j]):
                a[k] = aux[i]; i += 1
            else:
                a[k] = aux[j]; j += 1

    def sort(lo, hi):
        if hi - lo <= 1:
            return
        mid = (lo + hi) // 2
        sort(lo, mid)
        sort(mid, hi)
        if key(a[mid - 1]) <= key(a[mid]):   # already in order: skip merge
            return
        merge(lo, mid, hi)

    sort(0, len(a))
    return a

records = [(3, 'a'), (1, 'b'), (3, 'c'), (2, 'd'), (1, 'e')]
print(merge_sort_inplace(records, key=lambda r: r[0]))
# [(1, 'b'), (1, 'e'), (2, 'd'), (3, 'a'), (3, 'c')]   stable
```

### 8.3 Bottom-up (Python, iterative)

```python
def merge_sort_bottom_up(a):
    n = len(a)
    src, dst = a[:], [None] * n
    width = 1
    while width < n:
        for lo in range(0, n, 2 * width):
            mid = min(lo + width, n)
            hi  = min(lo + 2 * width, n)
            i, j = lo, mid
            for k in range(lo, hi):
                if i < mid and (j >= hi or src[i] <= src[j]):
                    dst[k] = src[i]; i += 1
                else:
                    dst[k] = src[j]; j += 1
        src, dst = dst, src             # swap roles instead of copying back
        width *= 2
    return src

print(merge_sort_bottom_up([5, 2, 4, 6, 1, 3, 2, 6]))
# [1, 2, 2, 3, 4, 5, 6, 6]
```

Swapping `src` and `dst` after each pass avoids copying the data back after every merge. A leftover run with no partner (`mid == hi`) is copied across unchanged, which is the "carried forward" behaviour from section 4.2.

### 8.4 k-way merge for external sorting (Python)

```python
import heapq

def k_way_merge(runs):
    """runs: list of iterators, each yielding values in sorted order."""
    heap = []
    for idx, it in enumerate(runs):
        first = next(it, None)
        if first is not None:
            heap.append((first, idx, it))   # idx breaks ties and keeps stability
    heapq.heapify(heap)
    while heap:
        value, idx, it = heapq.heappop(heap)
        yield value
        nxt = next(it, None)
        if nxt is not None:
            heapq.heappush(heap, (nxt, idx, it))

runs = [iter([7, 18, 41, 93]), iter([2, 30, 55, 66]), iter([14, 49, 72, 81])]
print(list(k_way_merge(runs)))
# [2, 7, 14, 18, 30, 41, 49, 55, 66, 72, 81, 93]
```

In a real external sort each iterator reads its run file through a buffer, and the output goes to a buffered writer. The standard library already provides this as `heapq.merge`.

### 8.5 Linked-list merge sort (C)

```c
typedef struct Node { int val; struct Node *next; } Node;

static Node *merge(Node *a, Node *b) {
    Node head = {0, NULL}, *tail = &head;
    while (a && b) {
        if (a->val <= b->val) { tail->next = a; a = a->next; }
        else                  { tail->next = b; b = b->next; }
        tail = tail->next;
    }
    tail->next = a ? a : b;               /* append the remainder, O(1) */
    return head.next;
}

Node *merge_sort(Node *h) {
    if (!h || !h->next) return h;
    Node *slow = h, *fast = h->next;      /* find the middle */
    while (fast && fast->next) { slow = slow->next; fast = fast->next->next; }
    Node *right = slow->next;
    slow->next = NULL;
    return merge(merge_sort(h), merge_sort(right));
}
```

No auxiliary array is needed: merging relinks nodes, and the remainder is attached with one pointer assignment. For linked lists, merge sort is usually the best general-purpose sort.

---

## 9. Optimizations used in practice

1. **Insertion sort for small subarrays.** Below about 16 to 32 elements, insertion sort's low overhead beats recursion. Stop recursing at that size and insertion-sort the piece. This often cuts running time by 10 to 20 percent.
2. **Skip merges that are already in order.** If `a[mid-1] <= a[mid]`, the two halves are already in order, so return without merging. With this check, sorted input runs in O(n).
3. **Allocate the buffer once** and pass it down, as in 8.2. Allocating inside every merge is a common performance mistake.
4. **Alternate source and destination arrays** at each level, as in 8.3, to avoid copying back after every merge.
5. **Natural merge sort.** Instead of starting from runs of length 1, scan for runs that are already ascending (and reverse strictly descending ones), then merge those. Partly sorted data then sorts close to linear time.
6. **Galloping.** When one side keeps winning, switch to exponential search to find how far it keeps winning, and copy that whole block at once.

Points 1, 5 and 6 together, plus a careful policy for which runs to merge, make **Timsort**. Timsort is the default sort in Python, and Java uses it for object arrays. Java uses quicksort for primitives, where stability does not matter and the lower memory use pays off.

7. **Parallelism.** The two recursive calls are independent, so they can run on different cores. The merge can also be parallelised: binary-search the median of one half in the other to split the merge into independent pieces. That gives O(n log n) work with polylogarithmic depth.

---

## 10. Merge sort compared with other sorts

| Property | Merge sort | Quicksort | Heapsort | Insertion sort |
|----------|-----------|-----------|----------|----------------|
| Worst-case time | Θ(n log n) | Θ(n²) | Θ(n log n) | Θ(n²) |
| Average time | Θ(n log n) | Θ(n log n) | Θ(n log n) | Θ(n²) |
| Best time | Θ(n log n), or Θ(n) with the skip check | Θ(n log n) | Θ(n log n) | Θ(n) |
| Extra space (arrays) | Θ(n) | Θ(log n) | Θ(1) | Θ(1) |
| Stable | **Yes** | No | No | Yes |
| Sequential access only | **Yes** | No | No | No |
| Good for linked lists | **Excellent** | Poor | Poor | OK |
| Good for external data | **Standard choice** | Poor | Poor | No |
| Cache behaviour | Good (linear scans) | Very good | Poor | Very good (small n) |

**Choose merge sort when** you need guaranteed O(n log n), you need stability, you are sorting linked lists, or the data does not fit in memory. **Choose something else when** memory is very tight and stability does not matter (heapsort or introsort), or when you are sorting primitives in memory and want the best constant factors (a well-tuned quicksort).

---

## 11. Summary

- Merge sort **splits** the input in half, **recurses** on each half, and **merges** the results. The merge is a linear-time two-pointer walk that makes at most p + q − 1 comparisons.
- The recurrence **T(n) = 2T(n/2) + Θ(n)** solves to **Θ(n log n)** in every case. The recursion tree has log₂ n levels, and each level costs Θ(n).
- It needs **Θ(n) extra space** for arrays and **O(1)** extra for linked lists.
- It is **stable** as long as ties are taken from the left half. The `<=` in the merge is what guarantees this.
- **Bottom-up** merge sort gives the same result without recursion, by doubling run widths 1, 2, 4 and so on.
- **External merge sort** generates sorted runs that fit in memory and then k-way merges them with a heap. Terabytes can be sorted in two or three sequential passes.
- Production sorts such as Timsort add natural runs, insertion-sort cutoffs and galloping on top of merge sort.
