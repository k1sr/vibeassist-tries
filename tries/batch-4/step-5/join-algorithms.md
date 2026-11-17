# Database Join Algorithms, Worked Through

A join looks like one operation in SQL, but a database engine can run it in very different ways. The same `JOIN ... ON` clause might finish in two milliseconds or two hours depending on which algorithm the query planner picks. This explainer covers the four families every relational engine uses: nested loop, index nested loop, sort-merge and hash join. It traces each one on the same two small tables, works out their I/O costs on larger tables with real page counts, covers the Grace hash join for inputs that do not fit in memory, and ends with how a planner chooses one and how to read the result in an `EXPLAIN` plan.

---

## 1. The running example

Two tables, joined on `dept_id`:

**`employees`** (6 rows)

| emp_id | name | dept_id |
|-------:|------|--------:|
| 1 | Ana | 10 |
| 2 | Ben | 20 |
| 3 | Cai | 10 |
| 4 | Dee | 30 |
| 5 | Eli | 20 |
| 6 | Fay | 40 |

**`departments`** (4 rows)

| dept_id | dept_name |
|--------:|-----------|
| 10 | Sales |
| 20 | Eng |
| 30 | Ops |
| 50 | Legal |

The query:

```sql
SELECT e.name, d.dept_name
FROM employees e
JOIN departments d ON e.dept_id = d.dept_id;
```

The correct answer has five rows:

| name | dept_name |
|------|-----------|
| Ana | Sales |
| Cai | Sales |
| Ben | Eng |
| Eli | Eng |
| Dee | Ops |

Fay (department 40) has no matching department, and Legal (50) has no employees, so neither appears in an inner join. Each algorithm below must produce these five rows. They differ only in how much work they do to find them, and in what order the rows come out.

Some vocabulary. The **outer** (or *driving*) input is the one the algorithm loops over. The **inner** input is the one it searches for matches. For hash joins, the **build** side is loaded into a hash table and the **probe** side is streamed against it.

---

## 2. Nested loop join

The simplest possible join compares every pair of rows.

```
for each row d in departments:          -- outer
    for each row e in employees:        -- inner
        if e.dept_id == d.dept_id:
            emit (e.name, d.dept_name)
```

### Trace

| Outer row | Inner rows scanned | Matches emitted |
|-----------|--------------------|-----------------|
| d=10 Sales | Ana, Ben, Cai, Dee, Eli, Fay | Ana–Sales, Cai–Sales |
| d=20 Eng | Ana, Ben, Cai, Dee, Eli, Fay | Ben–Eng, Eli–Eng |
| d=30 Ops | Ana, Ben, Cai, Dee, Eli, Fay | Dee–Ops |
| d=50 Legal | Ana, Ben, Cai, Dee, Eli, Fay | (none) |

That is 4 × 6 = **24 comparisons** to find 5 matches. The inner table was read in full four times.

The output order follows the outer table: all Sales rows, then Eng, then Ops. That ordering is sometimes useful to a later operator.

### Why it still matters

Nested loop is O(|R| × |S|), but it has three properties no other algorithm has:

1. **It works for any join condition.** `e.salary BETWEEN d.min_pay AND d.max_pay`, `ST_Contains(a.geom, b.geom)`, `a.ts > b.ts`: sort-merge and hash join need an equality condition, and nested loop does not.
2. **It needs no memory and no setup.** It produces its first row immediately, which suits `LIMIT 1` and `EXISTS` queries.
3. **When the outer side is tiny, it is the cheapest option.** If the outer input has one row, a nested loop scans the inner side once, and nothing else can beat that.

### Block nested loop

On disk, the naive version is even worse than it looks, because it rereads the inner table for every outer *row*. The practical variant is **block nested loop**: load as many outer pages as memory allows (a "block"), then scan the inner table once per block, checking each inner row against every outer row in memory. Section 7 shows that this one change cuts the cost from about 50 million page reads to about 6,000.

---

## 3. Index nested loop join

If the inner table has an index on the join column, the inner loop can become a lookup instead of a scan.

```
for each row d in departments:                         -- outer
    for each e in index_lookup(employees_dept_idx, d.dept_id):
        emit (e.name, d.dept_name)
```

Assume a B-tree index on `employees(dept_id)`. Its leaf level, in key order, looks like this:

```
10 → [row 1 Ana, row 3 Cai]
20 → [row 2 Ben, row 5 Eli]
30 → [row 4 Dee]
40 → [row 6 Fay]
```

### Trace

| Outer row | Index probe | Rows fetched | Matches emitted |
|-----------|-------------|--------------|-----------------|
| d=10 | descend to key 10 | Ana, Cai | Ana–Sales, Cai–Sales |
| d=20 | descend to key 20 | Ben, Eli | Ben–Eng, Eli–Eng |
| d=30 | descend to key 30 | Dee | Dee–Ops |
| d=50 | descend, key absent | none | (none) |

There are four index descents and five heap fetches, with no wasted comparisons. Fay is never touched, because no outer row asks for key 40.

### The catch

Each probe costs a tree descent (typically 2–4 page reads, though upper levels are usually cached) plus one heap page read per matching row if the index is **unclustered**, meaning the rows it points to are scattered across the table. Index nested loop is excellent when the outer side is small or heavily filtered. When the outer side is large, it is often the worst choice, because it turns a sequential scan into millions of random reads. Section 7 puts numbers on this.

Index nested loop is the workhorse of OLTP systems: "fetch this customer's 12 orders" is a one-row outer side with an index on `orders.customer_id`.

---

## 4. Sort-merge join

If both inputs are sorted on the join key, a single coordinated pass over both finds every match, like merging two sorted decks of cards.

### Step 1: sort both inputs

`employees` sorted by `dept_id`:

```
Ana 10, Cai 10, Ben 20, Eli 20, Dee 30, Fay 40
```

`departments` sorted by `dept_id` (already in order):

```
10 Sales, 20 Eng, 30 Ops, 50 Legal
```

### Step 2: merge

Keep a pointer into each list. Compare the current keys. If they are equal, emit matches. If they differ, advance the pointer with the smaller key.

```
E ptr        D ptr        Comparison   Action
Ana 10       10 Sales     10 = 10      emit Ana–Sales; advance E
Cai 10       10 Sales     10 = 10      emit Cai–Sales; advance E
Ben 20       10 Sales     20 > 10      advance D
Ben 20       20 Eng       20 = 20      emit Ben–Eng;   advance E
Eli 20       20 Eng       20 = 20      emit Eli–Eng;   advance E
Dee 30       20 Eng       30 > 20      advance D
Dee 30       30 Ops       30 = 30      emit Dee–Ops;   advance E
Fay 40       30 Ops       40 > 30      advance D
Fay 40       50 Legal     40 < 50      advance E
(end)                                  stop
```

That is **10 comparison steps** for the merge, against 24 for the plain nested loop, and the gap widens quickly as the tables grow. The merge is O(|R| + |S|) once the inputs are sorted.

### Duplicates on both sides

The trace above is simple because `departments.dept_id` is unique. If both sides had duplicates, for example two rows with key 20 on each side, the merge has to emit all 2 × 2 = 4 combinations. Implementations do this by **marking** the start of the group on the inner side and rewinding to that mark for each outer row in the group. A heavily duplicated key degrades this part toward a nested loop, but only inside that group.

### When it wins

- The inputs are **already sorted**: an index scan returns rows in key order, or an earlier merge join produced sorted output. Then the sort step is free.
- The query also needs `ORDER BY dept_id`, so the sorted output serves double duty. Planners call this an **interesting order**.
- The inputs are too big for a hash table, and the sort and the merge can be combined (see Section 7).
- Range joins such as `a.x BETWEEN b.lo AND b.hi` can use a variant of the merge, while hash join cannot handle them at all.

---

## 5. Hash join

Hash join spends memory instead of sort work. It builds a hash table on the smaller input, then streams the larger input past it.

### Build phase

Use `departments` (4 rows) as the build side. For a readable example, take the hash function h(k) = (k / 10) mod 3:

| dept_id | k / 10 | h(k) | Bucket |
|--------:|-------:|-----:|-------:|
| 10 | 1 | 1 | 1 |
| 20 | 2 | 2 | 2 |
| 30 | 3 | 0 | 0 |
| 50 | 5 | 2 | 2 |

The hash table:

```
bucket 0: [30 Ops]
bucket 1: [10 Sales]
bucket 2: [20 Eng] → [50 Legal]
```

Bucket 2 holds a **collision**: two different keys hash to the same bucket. That is normal. The probe always compares the actual keys, never just the hash.

### Probe phase

Stream `employees` through it:

| Probe row | h(dept_id) | Bucket contents | Key comparisons | Emitted |
|-----------|-----------:|-----------------|-----------------|---------|
| Ana 10 | 1 | 10 Sales | 10=10 ✓ | Ana–Sales |
| Ben 20 | 2 | 20 Eng, 50 Legal | 20=20 ✓, 20≠50 | Ben–Eng |
| Cai 10 | 1 | 10 Sales | 10=10 ✓ | Cai–Sales |
| Dee 30 | 0 | 30 Ops | 30=30 ✓ | Dee–Ops |
| Eli 20 | 2 | 20 Eng, 50 Legal | 20=20 ✓, 20≠50 | Eli–Eng |
| Fay 40 | 1 (4 mod 3) | 10 Sales | 40≠10 | (none) |

That is **8 key comparisons**, plus one hash computation per row. The output comes out in probe-side order (Ana, Ben, Cai, Dee, Eli), with no useful sort order.

### Properties

- Cost is O(|R| + |S|): each row is touched once on each side.
- It handles **equality conditions only**, because hashing destroys ordering.
- The build side must fit in memory, or the engine has to partition (next section). This is why planners always try to build on the **smaller** input after filtering.
- The build phase **blocks**: no output row appears until the whole build input has been read. That matters for queries with `LIMIT`.

---

## 6. Grace hash join: when the build side does not fit

Suppose memory holds only a fraction of the build side. A naive hash join would spill the hash table to disk and thrash. **Grace hash join** (named for the GRACE database machine project of the 1980s) avoids this by partitioning both inputs with the *same* hash function first, so that matching rows always land in matching partitions.

### Phase 1: partition

Pick a partitioning function h₁ and a number of partitions k. Read each input once, and write each row to partition file h₁(key). Here, pretend memory is so small that we need k = 2, using h₁(k) = (k / 10) mod 2:

| Row | k / 10 | h₁ | Partition |
|-----|-------:|---:|----------:|
| Dept 10 Sales | 1 | 1 | D₁ |
| Dept 20 Eng | 2 | 0 | D₀ |
| Dept 30 Ops | 3 | 1 | D₁ |
| Dept 50 Legal | 5 | 1 | D₁ |
| Emp Ana 10 | 1 | 1 | E₁ |
| Emp Ben 20 | 2 | 0 | E₀ |
| Emp Cai 10 | 1 | 1 | E₁ |
| Emp Dee 30 | 3 | 1 | E₁ |
| Emp Eli 20 | 2 | 0 | E₀ |
| Emp Fay 40 | 4 | 0 | E₀ |

Result:

```
Partition 0:  D₀ = {20 Eng}                        E₀ = {Ben 20, Eli 20, Fay 40}
Partition 1:  D₁ = {10 Sales, 30 Ops, 50 Legal}    E₁ = {Ana 10, Cai 10, Dee 30}
```

The key invariant: **a row in E₀ can only match a row in D₀**, because equal keys produce equal hashes. No row in partition 0 ever needs to be compared with partition 1.

### Phase 2: join each partition pair

For each i, load Dᵢ into an in-memory hash table using a *different* hash function h₂, then probe with Eᵢ. Using the same function again would put every row in a partition into the same few buckets.

- **Partition 0**: build {20 Eng}. Probe Ben (match), Eli (match), Fay (no match). Emits Ben–Eng and Eli–Eng.
- **Partition 1**: build {10 Sales, 30 Ops, 50 Legal}. Probe Ana (match), Cai (match), Dee (match). Emits Ana–Sales, Cai–Sales and Dee–Ops.

Five rows total, the same as every other algorithm.

### Skew and recursion

If one partition is still too large for memory, usually because one key value is extremely common (**skew**), the engine re-partitions that partition recursively with yet another hash function. When a single key is too frequent to split, no hash function can separate it, and engines fall back to a nested loop over chunks of that partition.

### Hybrid hash join

A refinement used by most modern engines: during Phase 1, keep partition 0 of the build side *in memory* as a hash table instead of writing it out. Probe-side rows that hash to partition 0 are joined immediately and never written to disk. When memory is only slightly smaller than the build side, this saves most of the partitioning I/O.

---

## 7. I/O cost, with page counts

On disk, the cost that matters is page I/O, not comparisons. Use the textbook setup:

- **R** (`orders`): **M = 1,000 pages**, 100 rows per page, so **100,000 rows**
- **S** (`customers`): **N = 500 pages**, 80 rows per page, so **40,000 rows**
- **B = 102 buffer pages** of memory available to the join
- Join on `orders.customer_id = customers.id`. Each order matches exactly one customer.
- The cost of writing the final output is ignored, because it is the same for every algorithm.

### Simple (tuple-at-a-time) nested loop

Scan R once. For each **row** of R, scan all of S:

```
M + (rows in R) × N = 1,000 + 100,000 × 500 = 50,001,000 I/Os
```

At 10 ms per random I/O, that is about **5.8 days**.

### Page nested loop

For each **page** of R, scan S once and compare all row pairs in the two pages:

```
M + M × N = 1,000 + 1,000 × 500 = 501,000 I/Os
```

With the smaller table outer instead: 500 + 500 × 1,000 = **500,500**. The outer choice barely matters here. What matters is how often the inner side gets rescanned.

### Block nested loop

Use B − 2 = 100 pages for a block of the outer table, 1 page for scanning the inner table and 1 for output:

```
R outer: M + ⌈M / (B−2)⌉ × N = 1,000 + ⌈1,000/100⌉ × 500 = 1,000 + 10 × 500 =  6,000 I/Os
S outer: N + ⌈N / (B−2)⌉ × M =   500 + ⌈500/100⌉   × 1,000 =  500 + 5 × 1,000 = 5,500 I/Os
```

Putting the **smaller relation on the outside** wins, because it needs fewer blocks and so fewer rescans of the larger relation. The jump from 50 million to 5,500 I/Os comes entirely from using memory well.

### Index nested loop

Suppose S has a B-tree index on `id`. Assume a probe costs about 2 I/Os to reach the leaf (the root is cached), plus 1 I/O to fetch the customer row.

```
M + (rows in R) × (2 + 1) = 1,000 + 100,000 × 3 = 301,000 I/Os
```

That is **55 times worse** than block nested loop, because every order triggers random reads. Now add a filter, `WHERE orders.order_date = '2026-11-01'`, that leaves 200 orders, and assume an index on `order_date` finds them in about 5 pages:

```
≈ 5 + 200 × 3 = 605 I/Os
```

Now index nested loop is the clear winner. **Selectivity of the outer side decides everything for index nested loop.**

### Sort-merge join

External sort cost: each pass reads and writes every page, so 2 × pages × passes. With B = 102:

- Sorting R: pass 0 creates ⌈1,000 / 102⌉ = 10 sorted runs. One merge pass (fan-in 101 ≥ 10) finishes it, so **2 passes**, costing 2 × 1,000 × 2 = **4,000**.
- Sorting S: ⌈500 / 102⌉ = 5 runs, then one merge pass, so **2 passes**, costing 2 × 500 × 2 = **2,000**.
- Merge-join the sorted files: M + N = **1,500**.

```
Total = 4,000 + 2,000 + 1,500 = 7,500 I/Os
```

**Refinement:** don't write the fully sorted files. After pass 0 there are 10 + 5 = 15 runs, which is fewer than B, so the final merge of both sorts *and* the join can happen in one combined pass that reads all 15 runs at once:

```
Pass 0 for both (read + write):   2 × (1,000 + 500) = 3,000
Combined merge + join (read):          1,000 + 500  = 1,500
Total                                               = 4,500 I/Os
```

This works whenever B is greater than about √(larger table), here √1,000 ≈ 32, which leaves plenty of headroom.

If R already had a clustered index on `customer_id`, its sort would be free, and the cost drops further.

### Hash join

**In-memory hash join** needs the build side plus a small overhead factor to fit in memory. S is 500 pages and B is 102, so it does not fit here. If B were 600, the cost would be one read of each input:

```
M + N = 1,500 I/Os
```

**Grace hash join** with B = 102:

- Partition phase: read R and S, write them back out as partitions: 2 × (1,000 + 500) = **3,000**.
- Join phase: read every partition pair once: 1,000 + 500 = **1,500**.

```
Total = 3 × (M + N) = 4,500 I/Os
```

The requirement is that each build partition fits in memory. With B − 1 = 101 partitions, each S partition is about 500 / 101 ≈ 5 pages, comfortably under 100. The general rule is B > √(f × N), where f is a fudge factor (about 1.2) for hash-table overhead: √(1.2 × 500) ≈ 25, so 102 is more than enough.

**Hybrid hash** with B = 102: we only need about 5 partitions to make each one fit (500/5 = 100 pages, about 120 with overhead, so call it 6 partitions of ~84 pages). Keep one partition resident. About one sixth of both inputs skips the write-and-reread round trip, saving roughly 2 × 1,500 / 6 = 500 I/Os, for **≈ 4,000 I/Os**.

### Summary

| Algorithm | I/Os (M=1,000, N=500, B=102) | Approx. time at 10 ms/I/O |
|-----------|-----------------------------:|---------------------------:|
| Tuple nested loop | 50,001,000 | 5.8 days |
| Page nested loop | 500,500 | 83 min |
| Index NL (all 100k orders) | 301,000 | 50 min |
| Block NL (S outer) | 5,500 | 55 s |
| Sort-merge (naive) | 7,500 | 75 s |
| Sort-merge (combined final pass) | 4,500 | 45 s |
| Grace hash | 4,500 | 45 s |
| Hybrid hash | ≈ 4,000 | 40 s |
| In-memory hash (if B ≥ ~600) | 1,500 | 15 s |
| Index NL (200 filtered orders) | ≈ 605 | 6 s |

These are rough estimates: sequential I/O is much cheaper than random I/O, and caching changes everything in practice. But the ranking is real, and it is exactly the arithmetic a cost-based planner does.

---

## 8. How a planner picks

A cost-based optimizer such as PostgreSQL's, SQL Server's or Oracle's does not have a fixed rule like "always hash." It enumerates candidate plans, estimates each one's cost, and keeps the cheapest. For joins, it considers the following.

### 1. Cardinality estimates

Everything starts here. The planner estimates how many rows each input produces after filters, using table statistics: row counts, distinct values per column, most-common-value lists and histograms. For an equi-join, a standard estimate is:

```
rows(R ⋈ S) ≈ rows(R) × rows(S) / max(ndistinct(R.key), ndistinct(S.key))
```

For our small tables: 6 × 4 / max(4, 4) = **6** estimated rows, against 5 actual. That is close enough.

**Bad estimates are the main cause of bad join plans.** If the planner thinks a filter returns 10 rows when it really returns 500,000, it will pick index nested loop and then perform half a million random lookups.

### 2. The join condition

- Equality: all algorithms are possible.
- Inequality or range (`<`, `BETWEEN`): nested loop or index nested loop, and some engines support merge-based range joins.
- Arbitrary expression or function: nested loop only.

### 3. Available indexes

An index on the inner side's join key makes index nested loop possible. An index that returns rows in join-key order makes sort-merge cheaper, because the sort is free.

### 4. Memory budget

PostgreSQL's `work_mem`, SQL Server's memory grants and Oracle's PGA settings determine whether a hash table fits in memory or must spill to partitions, and how many passes a sort needs.

### 5. Required output order

If the query has `ORDER BY` on the join key, or feeds a `GROUP BY` or another merge join on the same key, sort-merge's sorted output can eliminate a later sort. The planner credits it for that.

### 6. Join type

`LEFT`, `RIGHT`, `FULL`, `SEMI` (`EXISTS`, `IN`) and `ANTI` (`NOT EXISTS`) joins constrain which side can be outer or build. Many engines support "right" variants of hash join to recover flexibility. Full outer joins are typically limited to merge and hash join.

### 7. Join order

With more than two tables, the planner must also choose the *order*, and the number of possible orders grows factorially. PostgreSQL uses dynamic programming up to a threshold (`geqo_threshold`, 12 tables by default), then switches to a genetic search. Most planners favor **left-deep trees**, so each join's inner input is a base table or index, which suits pipelining.

### Rules of thumb that fall out of the cost model

| Situation | Usual winner |
|-----------|--------------|
| Small (or highly filtered) outer side, index on inner join key | Index nested loop |
| Both sides large, equality join, no useful order | Hash join |
| Both inputs already sorted (indexes), or output needs that order | Sort-merge |
| Non-equality join condition | Nested loop (block or indexed) |
| Tiny tables (a few pages) | Anything, and usually nested loop or hash |
| `LIMIT 1` / `EXISTS` with an index | Index nested loop (it returns first rows fast) |

---

## 9. Reading EXPLAIN plans

Plans are trees. Read them from the innermost, most indented node outward. For two-child join nodes in PostgreSQL, the **first child is the outer** (or probe) side and the **second is the inner** (or build) side. The examples below use PostgreSQL syntax. The costs are illustrative but internally consistent, and the format is `cost=startup..total rows=estimated`.

### Hash join

```sql
EXPLAIN ANALYZE
SELECT e.name, d.dept_name
FROM employees e JOIN departments d ON e.dept_id = d.dept_id;
```

```
Hash Join  (cost=1.09..2.20 rows=6 width=64) (actual time=0.031..0.040 rows=5 loops=1)
  Hash Cond: (e.dept_id = d.dept_id)
  ->  Seq Scan on employees e  (cost=0.00..1.06 rows=6 width=36) (actual rows=6 loops=1)
  ->  Hash  (cost=1.04..1.04 rows=4 width=36) (actual rows=4 loops=1)
        Buckets: 1024  Batches: 1  Memory Usage: 9kB
        ->  Seq Scan on departments d  (cost=0.00..1.04 rows=4 width=36) (actual rows=4 loops=1)
```

How to read it:
- `departments` feeds the `Hash` node, so it is the **build** side, matching our trace in Section 5.
- `employees` is the **probe** side.
- `Batches: 1` means the hash table fit in memory. `Batches: 8` would mean a Grace-style partitioned hash join that spilled to disk, which is a hint to raise `work_mem` or improve the estimate.
- `rows=6` estimated against `rows=5` actual: the estimate from Section 8 at work.
- The startup cost of 1.09 reflects the blocking build phase. No row comes out until the hash table is done.

### Index nested loop

```sql
EXPLAIN ANALYZE
SELECT e.name, d.dept_name
FROM departments d JOIN employees e ON e.dept_id = d.dept_id
WHERE d.dept_name = 'Eng';
```

```
Nested Loop  (cost=0.15..9.42 rows=2 width=64) (actual time=0.018..0.024 rows=2 loops=1)
  ->  Seq Scan on departments d  (cost=0.00..1.05 rows=1 width=36) (actual rows=1 loops=1)
        Filter: (dept_name = 'Eng'::text)
        Rows Removed by Filter: 3
  ->  Index Scan using employees_dept_id_idx on employees e
        (cost=0.15..8.35 rows=2 width=36) (actual rows=2 loops=1)
        Index Cond: (dept_id = d.dept_id)
```

- The outer side is filtered to **one row**, so the planner probes the index once.
- `loops=1` on the inner index scan means it ran once per outer row. On a big outer input, look for something like `loops=100000`: multiply the per-loop cost by the loop count to see the real price.
- `Index Cond: (dept_id = d.dept_id)` is the parameterized lookup, with the outer row's value pushed into the index search.

### Merge join

```sql
EXPLAIN
SELECT e.name, d.dept_name
FROM employees e JOIN departments d ON e.dept_id = d.dept_id
ORDER BY e.dept_id;
```

```
Merge Join  (cost=2.27..2.42 rows=6 width=68)
  Merge Cond: (e.dept_id = d.dept_id)
  ->  Sort  (cost=1.14..1.15 rows=6 width=40)
        Sort Key: e.dept_id
        ->  Seq Scan on employees e  (cost=0.00..1.06 rows=6 width=40)
  ->  Index Scan using departments_pkey on departments d  (cost=0.13..12.19 rows=4 width=36)
```

- `departments` arrives already sorted via its primary-key index, so only `employees` needs a `Sort` node.
- There is **no sort above the Merge Join**, even though the query says `ORDER BY e.dept_id`. The merge output is already in that order, so the planner chose this plan partly to avoid a final sort. That is the "interesting order" effect.
- On a real table, a `Sort` node with `Sort Method: external merge  Disk: 51200kB` under `EXPLAIN ANALYZE` means the sort spilled. That is the multi-pass external sort from Section 7.

### Nudging the planner (for diagnosis only)

PostgreSQL lets you disable methods per session to compare plans:

```sql
SET enable_hashjoin = off;
SET enable_mergejoin = off;
EXPLAIN ANALYZE ...;   -- now forced toward nested loop
RESET ALL;
```

This is a diagnostic tool, not a fix. If a forced plan is faster, the real problem is usually stale statistics (`ANALYZE`), correlated columns the planner treats as independent (`CREATE STATISTICS`), or a memory setting, not the planner's algorithm choice.

---

## 10. What to remember

1. **Nested loop** compares every pair. It is quadratic but universal, needs no memory, and is fastest when the outer side is tiny. The block variant makes it usable on disk.
2. **Index nested loop** turns the inner scan into lookups. It is unbeatable for selective queries and disastrous for large unfiltered ones, because of random I/O.
3. **Sort-merge** sorts both sides, then makes one linear pass. It is strong when inputs are already ordered or the output must be, and it can merge its final sort pass with the join.
4. **Hash join** builds on the smaller side and probes with the larger, for linear cost on equality joins. It is the default choice for big analytical joins.
5. **Grace and hybrid hash** partition both inputs so each piece fits in memory, at about 3 × (M + N) I/Os, provided memory exceeds roughly the square root of the build side.
6. **The planner chooses** from cardinality estimates, join conditions, indexes, memory and required order. When it chooses badly, check the estimates first: compare `rows=` estimated against actual in `EXPLAIN ANALYZE`, and the reason is usually there.
