# Quicksort: A Complete Explainer

Quicksort, invented by Tony Hoare in 1959 and published in 1961, still sits at the heart of the standard sorting routines in C, C++, Java, Go, and Rust. It is fast in practice, sorts in place, and is short enough to write from memory, but it has real pitfalls worth understanding.

---

## 1. The Core Idea

Quicksort is a **divide-and-conquer** algorithm. To sort an array `A[lo..hi]`:

1. **Choose a pivot.** Pick one element of the subarray. Its value is the pivot.
2. **Partition.** Rearrange the subarray so that every element "small" relative to the pivot comes before every element "large" relative to it. After partitioning, the subarray is split into two regions that can be sorted independently.
3. **Recurse.** Sort the left region and the right region with Quicksort.
4. **Combine.** Nothing to do. Because the left region holds only small elements and the right region only large ones, once both are sorted the whole subarray is sorted.

Merge Sort does trivial work when dividing and real work when merging. Quicksort is the opposite: partitioning does the work and combining is free, which is why it can sort in place.

The base case is a subarray of zero or one element, which is already sorted.

In pseudocode:

```
quicksort(A, lo, hi):
    if lo < hi:
        p = partition(A, lo, hi)
        quicksort(A, lo, p - 1)     # exact bounds depend on the scheme
        quicksort(A, p + 1, hi)
```

Everything interesting lives inside `partition`. There are two classic ways to write it.

---

## 2. Lomuto Partition Scheme

The Lomuto scheme, popularized by Nico Lomuto and later by Jon Bentley's *Programming Pearls* and the CLRS textbook, is the easier of the two to understand and prove correct.

### How it works

- The pivot is the **last** element, `A[hi]`.
- An index `i` marks the boundary of the "small" region. Everything at positions `lo..i` is less than or equal to the pivot. It starts at `lo - 1`, meaning the region is empty.
- A second index `j` scans from `lo` to `hi - 1`. Whenever `A[j] <= pivot`, we grow the small region by one (`i += 1`) and swap `A[j]` into it.
- When the scan finishes, we swap the pivot into position `i + 1`, which lies right between the two regions.

```
lomuto_partition(A, lo, hi):
    pivot = A[hi]
    i = lo - 1
    for j from lo to hi - 1:
        if A[j] <= pivot:
            i = i + 1
            swap A[i], A[j]
    swap A[i + 1], A[hi]
    return i + 1
```

**The invariant.** At the start of each loop iteration:
- `A[lo..i]` holds elements `<= pivot`,
- `A[i+1..j-1]` holds elements `> pivot`,
- `A[j..hi-1]` has not been examined yet,
- `A[hi]` is the pivot.

When `j` reaches `hi`, the unexamined region is empty. Swapping the pivot with `A[i+1]`, the first "large" element, puts it in its **final sorted position**. That position is returned, and the pivot is excluded from both recursive calls.

### Worked trace: Lomuto on `[7, 2, 1, 6, 8, 5, 3, 4]`

Indices 0 to 7. Pivot = `A[7]` = **4**. Start with `i = -1`.

| Step | j | A[j] | A[j] <= 4? | Action | i | Array after |
|---|---|---|---|---|---|---|
| 1 | 0 | 7 | no | none | -1 | `[7, 2, 1, 6, 8, 5, 3, 4]` |
| 2 | 1 | 2 | yes | i=0, swap A[0], A[1] | 0 | `[2, 7, 1, 6, 8, 5, 3, 4]` |
| 3 | 2 | 1 | yes | i=1, swap A[1], A[2] | 1 | `[2, 1, 7, 6, 8, 5, 3, 4]` |
| 4 | 3 | 6 | no | none | 1 | `[2, 1, 7, 6, 8, 5, 3, 4]` |
| 5 | 4 | 8 | no | none | 1 | `[2, 1, 7, 6, 8, 5, 3, 4]` |
| 6 | 5 | 5 | no | none | 1 | `[2, 1, 7, 6, 8, 5, 3, 4]` |
| 7 | 6 | 3 | yes | i=2, swap A[2], A[6] | 2 | `[2, 1, 3, 6, 8, 5, 7, 4]` |
| Final | | | | swap A[3], A[7] | | `[2, 1, 3, 4, 8, 5, 7, 6]` |

Return value: **3**. The pivot 4 now sits at index 3, which is exactly where it belongs in the sorted output. Everything to its left (`2, 1, 3`) is smaller; everything to its right (`8, 5, 7, 6`) is larger.

The scheme made 7 comparisons (always `n - 1` for a subarray of size `n`) and 4 swaps.

### Weaknesses of Lomuto

It swaps far more than Hoare's scheme, and on an all-equal array every comparison succeeds, so the split is maximally unbalanced and the sort goes quadratic (Section 7 fixes this).

---

## 3. Hoare Partition Scheme

Hoare's original scheme uses two indices that start at opposite ends of the subarray and move toward each other.

### How it works

- Choose a pivot value. The original paper uses the first element; a common modern choice is the middle element, `A[(lo + hi) / 2]`.
- `i` starts just left of `lo`, and `j` starts just right of `hi`.
- Repeatedly:
  - Move `i` right until it finds an element `>= pivot`.
  - Move `j` left until it finds an element `<= pivot`.
  - If `i >= j`, the indices have met or crossed: return `j`.
  - Otherwise `A[i]` belongs on the right and `A[j]` belongs on the left, so swap them.

```
hoare_partition(A, lo, hi):
    pivot = A[(lo + hi) // 2]
    i = lo - 1
    j = hi + 1
    loop forever:
        do i = i + 1 while A[i] < pivot
        do j = j - 1 while A[j] > pivot
        if i >= j:
            return j
        swap A[i], A[j]
```

A crucial difference: Hoare's scheme does **not** guarantee that the pivot ends up in its final position. It only guarantees that every element of `A[lo..j]` is `<=` every element of `A[j+1..hi]`. So the recursive calls are:

```
quicksort(A, lo, p)       # note: p, not p - 1
quicksort(A, p + 1, hi)
```

Getting these bounds wrong is the single most common Quicksort bug. If you use `p - 1` with Hoare, you can skip elements; if you pick the pivot as the last element with these bounds, you can get infinite recursion when `j` returns `hi`. That is why the middle (or first) element is the safe choice for Hoare.

### Worked trace: Hoare on `[7, 2, 1, 6, 8, 5, 3, 4]`

Indices 0 to 7. Middle index is `(0 + 7) // 2 = 3`, so pivot = **6**. Start with `i = -1`, `j = 8`.

**Round 1.**
- `i` moves to 0. `A[0] = 7`, not `< 6`. Stop at `i = 0`.
- `j` moves to 7. `A[7] = 4`, not `> 6`. Stop at `j = 7`.
- `0 < 7`, so swap `A[0]` and `A[7]`: `[4, 2, 1, 6, 8, 5, 3, 7]`

**Round 2.**
- `i` moves to 1 (`2 < 6`, keep going), 2 (`1 < 6`, keep going), 3 (`6` is not `< 6`). Stop at `i = 3`.
- `j` moves to 6. `A[6] = 3`, not `> 6`. Stop at `j = 6`.
- `3 < 6`, so swap `A[3]` and `A[6]`: `[4, 2, 1, 3, 8, 5, 6, 7]`

**Round 3.**
- `i` moves to 4. `A[4] = 8`, not `< 6`. Stop at `i = 4`.
- `j` moves to 5. `A[5] = 5`, not `> 6`. Stop at `j = 5`.
- `4 < 5`, so swap: `[4, 2, 1, 3, 5, 8, 6, 7]`

**Round 4.**
- `i` moves to 5. `A[5] = 8`, stop at `i = 5`.
- `j` moves to 4. `A[4] = 5`, stop at `j = 4`.
- `i >= j`, so return **4**.

Result: `[4, 2, 1, 3, 5 | 8, 6, 7]`. The left part `A[0..4]` holds values all `<= 6`; the right part `A[5..7]` holds values all `>= 6`. Note that the pivot value 6 ended up at index 6, inside the right part and not yet at its final sorted position (index 5). That is fine: the recursion will place it.

Hoare used 3 swaps here, against Lomuto's 4 on the same input, and the gap widens on larger arrays.

### Why Hoare is usually faster

Elements already on the correct side are never touched, so it swaps less. Both scans stop on keys *equal* to the pivot, so duplicates are spread evenly and an all-equal array splits down the middle. The price is subtler correctness and less forgiving boundaries.

### Side-by-side summary

| Property | Lomuto | Hoare |
|---|---|---|
| Pivot location | Usually last element | Usually middle or first |
| Pivot placed in final position? | Yes | Not necessarily |
| Recursive bounds | `[lo, p-1]`, `[p+1, hi]` | `[lo, p]`, `[p+1, hi]` |
| Average swaps (random input) | about n/2 | about n/6 |
| All-equal input | Quadratic | n log n |
| Ease of reasoning | Simple | Subtle |

---

## 4. A Full Recursive Trace

Partitioning is one step. Here is the whole algorithm on `[7, 2, 1, 6, 8, 5, 3, 4]` using Lomuto, so you can see how the recursion unfolds. Brackets mark the active subarray; a pivot in bold is in its final position.

**Call 1: `quicksort(A, 0, 7)`**, pivot 4.
From the trace above: `[2, 1, 3, **4**, 8, 5, 7, 6]`, pivot index 3.
Recurse on indices 0..2 and 4..7.

**Call 2: `quicksort(A, 0, 2)`** on `[2, 1, 3]`, pivot 3.
- `j=0`: 2 <= 3, `i=0`, swap A[0] with itself.
- `j=1`: 1 <= 3, `i=1`, swap A[1] with itself.
- Swap A[2] with itself. Pivot index 2.

Array: `[2, 1, **3**, **4**, 8, 5, 7, 6]`. Recurse on 0..1 and 3..2 (empty).

**Call 3: `quicksort(A, 0, 1)`** on `[2, 1]`, pivot 1.
- `j=0`: 2 > 1, no action.
- Swap A[0] and A[1]. Pivot index 0.

Array: `[**1**, 2, **3**, **4**, 8, 5, 7, 6]`. Recurse on 0..-1 (empty) and 1..1 (single element). Both are base cases. Index 1 (value 2) is now settled.

**Call 4: `quicksort(A, 4, 7)`** on `[8, 5, 7, 6]`, pivot 6. `i = 3`.
- `j=4`: 8 > 6, no action.
- `j=5`: 5 <= 6, `i=4`, swap A[4] and A[5]: `[5, 8, 7, 6]`.
- `j=6`: 7 > 6, no action.
- Swap A[5] and A[7]: `[5, 6, 7, 8]`. Pivot index 5.

Array: `[1, 2, 3, 4, **5**, **6**, 7, 8]`. Recurse on 4..4 (single) and 6..7.

**Call 5: `quicksort(A, 6, 7)`** on `[7, 8]`, pivot 8.
- `j=6`: 7 <= 8, `i=6`, swap with itself.
- Swap A[7] with itself. Pivot index 7.

Recurse on 6..6 (single) and 8..7 (empty).

**Final:** `[1, 2, 3, 4, 5, 6, 7, 8]`.

The recursion tree looks like this:

```
                [7 2 1 6 8 5 3 4]  pivot 4
               /                  \
         [2 1 3] pivot 3        [8 5 7 6] pivot 6
          /                      /        \
     [2 1] pivot 1             [5]      [7 8] pivot 8
          \                               /
          [2]                           [7]
```

The tree has depth 3 for 8 elements, close to the ideal `log2(8) = 3`. That balance is what makes Quicksort fast.

---

## 5. Complexity Analysis

Let `T(n)` be the running time on an array of `n` elements. Partitioning a subarray of size `n` takes `Θ(n)` time: each element is compared with the pivot about once. If the pivot splits the array into parts of size `k` and `n - k - 1`:

```
T(n) = T(k) + T(n - k - 1) + Θ(n)
```

The whole story depends on `k`.

### Best case: Θ(n log n)

If every pivot is the median, each split is even:

```
T(n) = 2 T(n/2) + Θ(n)
```

By the Master Theorem (or by drawing the recursion tree: `log2 n` levels, each doing `Θ(n)` total work), `T(n) = Θ(n log n)`.

### Worst case: Θ(n²)

If every pivot is the smallest or largest element, one side is empty and the other has `n - 1` elements:

```
T(n) = T(n - 1) + Θ(n)
     = n + (n-1) + (n-2) + ... + 1
     = n(n+1)/2
     = Θ(n²)
```

With "last element as pivot", this happens on input that is **already sorted** or **reverse sorted**, which is embarrassingly common in real data. The recursion depth also becomes `n`, which can overflow the call stack for large arrays.

### Average case: Θ(n log n)

Assume all input orderings are equally likely (or, equivalently, that the pivot is chosen uniformly at random). Then each value of `k` from `0` to `n - 1` is equally likely, and the expected number of comparisons `C(n)` satisfies:

```
C(n) = (n - 1) + (1/n) * Σ_{k=0}^{n-1} [ C(k) + C(n - k - 1) ]
     = (n - 1) + (2/n) * Σ_{k=0}^{n-1} C(k)
```

Multiply both sides by `n`, write the same equation for `n - 1`, and subtract. The sums mostly cancel, leaving:

```
n C(n) = (n + 1) C(n - 1) + 2(n - 1)
```

Divide by `n(n + 1)`:

```
C(n)/(n+1) = C(n-1)/n + 2(n-1)/(n(n+1))
```

Telescoping this sum gives `C(n)/(n+1) ≈ 2 H(n)`, where `H(n)` is the n-th harmonic number, approximately `ln n`. Therefore:

```
C(n) ≈ 2 n ln n ≈ 1.39 n log2 n
```

That is only about 39% more comparisons than the best case, and in practice Quicksort often beats Merge Sort and Heapsort thanks to its tight, cache-friendly inner loop.

**An intuition for why "average" is good.** Even a fairly lopsided split keeps the depth logarithmic. If every split were 1-to-9, the recursion depth would be `log_{10/9} n`, about `6.6 log2 n`: still `O(log n)`, so total work is still `O(n log n)`. Quicksort only collapses when splits are *consistently* terrible. A random pivot lands in the middle half of the values with probability 1/2, so good splits happen constantly.

### Space and stability

Quicksort sorts in place, but recursion uses `O(log n)` stack on average and `O(n)` in the worst case, unless you recurse into the smaller side first, which guarantees `O(log n)`. It is **not stable**: long-distance swaps can reorder equal keys.

| Case | Time | Stack space |
|---|---|---|
| Best | Θ(n log n) | Θ(log n) |
| Average | Θ(n log n), about 1.39 n log2 n comparisons | Θ(log n) |
| Worst | Θ(n²) | Θ(n), or Θ(log n) with smaller-side recursion |

---

## 6. Pivot Strategies

Because the worst case comes entirely from bad pivots, choosing the pivot well is the main practical lever.

### First or last element

Simplest possible. Catastrophic on sorted, reverse-sorted, or nearly sorted input, all of which are common. Avoid it outside teaching code.

### Middle element

Using `A[(lo + hi) / 2]` fixes the sorted and reverse-sorted cases, since the middle of a sorted array is its median. It is still deterministic, so specific adversarial inputs (sometimes called "median-of-three killers") can force quadratic behavior. A good default for Hoare's scheme.

### Random pivot

Pick an index uniformly at random from `lo..hi` and swap it into the pivot position. Now no particular input is bad. The expected running time is `O(n log n)` for **every** input, and the probability of a substantially worse run is vanishingly small for large `n`. This is the version analyzed in most textbooks. The costs are a random number call per partition and non-reproducible behavior, which can matter for debugging.

### Median-of-three

Look at the first, middle, and last elements and use the median of the three. This:

- handles sorted and reverse-sorted input perfectly,
- makes a very bad pivot much less likely (the pivot can never be the minimum or maximum of the subarray),
- reduces the expected comparison count from about `1.39 n log2 n` to about `1.19 n log2 n`,
- can double as a sentinel: after sorting the three samples, `A[lo] <= pivot <= A[hi]`, which lets the scanning loops skip bounds checks.

Example: for `[9, 4, 7, 1, 8, 2, 5]`, the samples are `A[0] = 9`, `A[3] = 1`, `A[6] = 5`. Their median is 5, a far better pivot than 9 (the max) or 1 (the min).

### Ninther and true median

For large arrays, the "ninther" takes the median of three medians-of-three, giving a pivot very close to the true median (Bentley and McIlroy, 1993). Finding the exact median in `O(n)` with median-of-medians guarantees `O(n log n)` Quicksort, but its constant factor is so large that introsort (Section 8) is used instead.

---

## 7. Duplicate Keys and Three-Way Partitioning

Two-way partitioning has a blind spot: many equal keys. With Lomuto, an array of all-equal values goes quadratic. Even with Hoare, equal keys are partitioned over and over, wasting effort on elements that are already "sorted" relative to each other.

**Three-way partitioning**, also called the Dutch National Flag approach after Dijkstra's formulation, splits the subarray into three regions:

```
[ < pivot | == pivot | > pivot ]
```

The middle region is never touched again. Only the outer regions are recursed into.

### Dijkstra's three-way partition

Maintain three indices:
- `lt`: everything in `A[lo..lt-1]` is `< pivot`,
- `i`: the scanner; `A[lt..i-1]` is `== pivot`,
- `gt`: everything in `A[gt+1..hi]` is `> pivot`.
- `A[i..gt]` is not yet examined.

```
three_way_partition(A, lo, hi):
    pivot = A[lo]
    lt, i, gt = lo, lo, hi
    while i <= gt:
        if A[i] < pivot:
            swap A[lt], A[i];  lt += 1;  i += 1
        elif A[i] > pivot:
            swap A[i], A[gt];  gt -= 1      # do not advance i: the new A[i] is unexamined
        else:
            i += 1
    return lt, gt
```

### Worked trace: three-way on `[3, 1, 3, 2, 3, 5, 3]`

Pivot = **3**. Start `lt = 0`, `i = 0`, `gt = 6`.

| i | A[i] | Compare | Action | Array after | lt | gt |
|---|---|---|---|---|---|---|
| 0 | 3 | equal | i=1 | `[3, 1, 3, 2, 3, 5, 3]` | 0 | 6 |
| 1 | 1 | less | swap A[0], A[1]; lt=1, i=2 | `[1, 3, 3, 2, 3, 5, 3]` | 1 | 6 |
| 2 | 3 | equal | i=3 | `[1, 3, 3, 2, 3, 5, 3]` | 1 | 6 |
| 3 | 2 | less | swap A[1], A[3]; lt=2, i=4 | `[1, 2, 3, 3, 3, 5, 3]` | 2 | 6 |
| 4 | 3 | equal | i=5 | `[1, 2, 3, 3, 3, 5, 3]` | 2 | 6 |
| 5 | 5 | greater | swap A[5], A[6]; gt=5 | `[1, 2, 3, 3, 3, 3, 5]` | 2 | 5 |
| 5 | 3 | equal | i=6 | `[1, 2, 3, 3, 3, 3, 5]` | 2 | 5 |

Now `i = 6 > gt = 5`, so stop. Regions: `[1, 2]` less, `[3, 3, 3, 3]` equal (indices 2..5), `[5]` greater. Only `[1, 2]` and `[5]` are recursed into; all four 3s are done.

When the input has only `k` distinct values, three-way Quicksort runs in `O(n log k)` time, and in linear time when `k` is constant. Bentley and McIlroy's variant achieves the same result with fewer swaps by collecting equal elements at both ends during a Hoare-style scan and swapping them into the middle at the end.

---

## 8. Practical Optimizations

Production Quicksorts layer several refinements on top of the basic algorithm.

### Insertion sort cutoff and smaller-side recursion

Below 10 to 20 elements, recursion overhead dominates, so switch to insertion sort. Recursing only into the smaller partition and looping on the larger bounds the stack at `O(log n)`.

### Introsort, dual-pivot, and pdqsort

**Introsort** (David Musser, 1997) runs Quicksort but tracks recursion depth; past about `2 log2 n` it switches that subarray to Heapsort, guaranteeing `O(n log n)` while keeping Quicksort's average speed. It underlies most C++ `std::sort` implementations. **Dual-pivot Quicksort** (Yaroslavskiy, used in Java since version 7) partitions around two pivots into three regions and has fewer cache misses. **Pattern-defeating Quicksort** (pdqsort), used by Rust's `sort_unstable` and Go's `sort`, adds sorted-run detection, a fast path for equal keys, and branchless partitioning.

---

## 9. Code

### Python: Lomuto with random pivot

```python
import random

def quicksort_lomuto(a, lo=0, hi=None):
    if hi is None:
        hi = len(a) - 1
    if lo < hi:
        p = lomuto_partition(a, lo, hi)
        quicksort_lomuto(a, lo, p - 1)
        quicksort_lomuto(a, p + 1, hi)

def lomuto_partition(a, lo, hi):
    # Random pivot, moved to the end so the classic scheme applies.
    r = random.randint(lo, hi)
    a[r], a[hi] = a[hi], a[r]
    pivot = a[hi]
    i = lo - 1
    for j in range(lo, hi):
        if a[j] <= pivot:
            i += 1
            a[i], a[j] = a[j], a[i]
    a[i + 1], a[hi] = a[hi], a[i + 1]
    return i + 1
```

### Python: Hoare with median-of-three

```python
def quicksort_hoare(a, lo=0, hi=None):
    if hi is None:
        hi = len(a) - 1
    if lo < hi:
        p = hoare_partition(a, lo, hi)
        quicksort_hoare(a, lo, p)        # p, not p - 1
        quicksort_hoare(a, p + 1, hi)

def hoare_partition(a, lo, hi):
    mid = (lo + hi) // 2
    # Order the three samples so that a[lo] <= a[mid] <= a[hi].
    if a[mid] < a[lo]:
        a[lo], a[mid] = a[mid], a[lo]
    if a[hi] < a[lo]:
        a[lo], a[hi] = a[hi], a[lo]
    if a[hi] < a[mid]:
        a[mid], a[hi] = a[hi], a[mid]
    pivot = a[mid]
    i, j = lo - 1, hi + 1
    while True:
        i += 1
        while a[i] < pivot:
            i += 1
        j -= 1
        while a[j] > pivot:
            j -= 1
        if i >= j:
            return j
        a[i], a[j] = a[j], a[i]
```

### Python: three-way Quicksort with smaller-side recursion and cutoff

```python
import random

CUTOFF = 16

def insertion_sort(a, lo, hi):
    for k in range(lo + 1, hi + 1):
        x = a[k]
        m = k - 1
        while m >= lo and a[m] > x:
            a[m + 1] = a[m]
            m -= 1
        a[m + 1] = x

def quicksort_3way(a, lo=0, hi=None):
    if hi is None:
        hi = len(a) - 1
    while hi - lo + 1 > CUTOFF:
        r = random.randint(lo, hi)
        a[lo], a[r] = a[r], a[lo]
        pivot = a[lo]
        lt, i, gt = lo, lo, hi
        while i <= gt:
            if a[i] < pivot:
                a[lt], a[i] = a[i], a[lt]
                lt += 1
                i += 1
            elif a[i] > pivot:
                a[i], a[gt] = a[gt], a[i]
                gt -= 1
            else:
                i += 1
        # Recurse into the smaller side; loop on the larger one.
        if lt - lo < hi - gt:
            quicksort_3way(a, lo, lt - 1)
            lo = gt + 1
        else:
            quicksort_3way(a, gt + 1, hi)
            hi = lt - 1
    insertion_sort(a, lo, hi)
```

### C: Hoare partition, iterative on the larger side

```c
#include <stddef.h>

static void swap(int *x, int *y) { int t = *x; *x = *y; *y = t; }

static ptrdiff_t hoare_partition(int *a, ptrdiff_t lo, ptrdiff_t hi) {
    int pivot = a[lo + (hi - lo) / 2];   /* avoids overflow of (lo + hi) */
    ptrdiff_t i = lo - 1, j = hi + 1;
    for (;;) {
        do { i++; } while (a[i] < pivot);
        do { j--; } while (a[j] > pivot);
        if (i >= j) return j;
        swap(&a[i], &a[j]);
    }
}

void quicksort(int *a, ptrdiff_t lo, ptrdiff_t hi) {
    while (lo < hi) {
        ptrdiff_t p = hoare_partition(a, lo, hi);
        if (p - lo < hi - p) {
            quicksort(a, lo, p);
            lo = p + 1;
        } else {
            quicksort(a, p + 1, hi);
            hi = p;
        }
    }
}
```

---

## 10. Common Pitfalls

- **Wrong recursive bounds for Hoare.** Use `[lo, p]` and `[p+1, hi]`, not `[lo, p-1]`.
- **Last-element pivot with Hoare.** Can make `j` return `hi`, so the left call is the whole array again: infinite recursion.
- **Strict vs. non-strict comparisons.** Hoare's scans must use `<` and `>` (stopping on equal keys). Using `<=` lets `i` run off the end of the array.
- **Integer overflow in the midpoint.** In fixed-width languages, `(lo + hi) / 2` can overflow; use `lo + (hi - lo) / 2`.
- **Deterministic pivot on untrusted input.** An attacker who knows your pivot rule can craft quadratic inputs, a real denial-of-service vector. Randomize, or use introsort.
- **Assuming stability.** Quicksort reorders equal elements.

---

## 11. Summary

Quicksort picks a pivot, partitions around it, and sorts each side recursively. Lomuto is easy to reason about; Hoare swaps less and handles duplicates better. It averages `Θ(n log n)` time, about `1.39 n log2 n` comparisons, in `O(log n)` extra space, but can hit `Θ(n²)` with bad pivots. Randomized or median-of-three pivots, three-way partitioning, and introsort close those gaps, which is why it still powers the sort you call every day.
