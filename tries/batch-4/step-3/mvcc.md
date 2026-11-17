# Multi-Version Concurrency Control, Worked Through

Multi-version concurrency control (MVCC) is the technique most modern relational databases use to let many transactions read and write the same data at once without making readers and writers wait for each other. PostgreSQL, Oracle, MySQL's InnoDB, SQL Server (with snapshot isolation enabled), CockroachDB, and many others build on it. The core idea fits in one sentence: **instead of overwriting a row in place, a write creates a new version of it, and each transaction reads whichever version was current as of its own snapshot.**

This explainer works through that idea with concrete timelines. It uses PostgreSQL as the main reference because its design is unusually transparent: every version is a physical tuple in the table, and the visibility rules are visible in the tuple headers. Where other systems differ, that is noted.

---

## 1. The problem MVCC solves

Picture a table `accounts(id, balance)` and two transactions:

- **T1** runs a long report: `SELECT sum(balance) FROM accounts`.
- **T2** moves 100 from account A to account B: two `UPDATE`s, then `COMMIT`.

With classic two-phase locking (2PL), T1 takes shared locks on every row it reads, and T2 needs exclusive locks on the rows it writes. Whoever arrives second waits. If the report takes ten minutes, the transfer waits up to ten minutes, or the report waits behind every transfer. Neither is acceptable for a busy system.

Without isolation, T1 might read A after the debit and B before the credit, and the sum comes out 100 short.

MVCC removes the conflict between reading and writing. T2 writes new versions of A and B. T1 keeps reading the old versions, which still exist, because those are the versions its snapshot says are current. The report sees a consistent state from before the transfer; the transfer never waits for the report. The slogan is **readers don't block writers, and writers don't block readers.** Writers still conflict with other writers on the same row, as we'll see.

The costs are equally concrete: old versions take space, someone has to clean them up, and the isolation you get by default ("snapshot isolation") is not quite serializable. The rest of this document covers each of those.

---

## 2. Versions on disk: PostgreSQL's tuple header

In PostgreSQL, every row version (a *tuple*) carries a header with, among other fields:

| Field  | Meaning |
|--------|---------|
| `xmin` | ID of the transaction that created this version (by `INSERT` or `UPDATE`). |
| `xmax` | ID of the transaction that deleted or superseded this version, or 0 if none. Also used for row locks. |
| `cmin`/`cmax` | Command number within the creating/deleting transaction, so a transaction can tell its own earlier statements' writes from later ones. |
| `ctid` | Physical location (page, offset). For a superseded version, it points to the next newer version. |
| hint bits | Cached "xmin committed / aborted", "xmax committed / aborted" flags, so the commit log need not be consulted every time. |

Transaction IDs (XIDs) are 32-bit counters assigned when a transaction first writes. Read-only transactions get only a virtual ID and never consume an XID.

The three operations become:

- **INSERT**: write a new tuple with `xmin = my XID`, `xmax = 0`.
- **DELETE**: do not remove anything; set `xmax = my XID` on the current version.
- **UPDATE**: a DELETE plus an INSERT. Set `xmax = my XID` on the old version, write a new tuple with `xmin = my XID`, and set the old tuple's `ctid` to point at the new one.

Whether any of this "happened" depends entirely on whether the writing transaction committed, which is recorded separately in the commit log (`pg_xact`). An abort costs almost nothing: the versions it created simply never become visible, and the `xmax` it stamped is ignored.

### A version chain, step by step

Start with one row, inserted and committed by transaction 100:

```
accounts, row id=1
  (0,1)  xmin=100  xmax=0    balance=500   ctid=(0,1)
```

Transaction 105 runs `UPDATE accounts SET balance = 400 WHERE id = 1` and commits:

```
  (0,1)  xmin=100  xmax=105  balance=500   ctid=(0,2)   -- superseded
  (0,2)  xmin=105  xmax=0    balance=400   ctid=(0,2)   -- current
```

Transaction 112 sets the balance to 450 and commits:

```
  (0,1)  xmin=100  xmax=105  balance=500   ctid=(0,2)
  (0,2)  xmin=105  xmax=112  balance=400   ctid=(0,3)
  (0,3)  xmin=112  xmax=0    balance=450   ctid=(0,3)
```

That is a **version chain**: three physical tuples for one logical row, linked oldest-to-newest through `ctid`. Each version has a lifetime `[xmin, xmax)` in transaction terms. Which one a given transaction sees depends on its snapshot.

Other designs store the chain differently. Oracle and InnoDB keep only the newest version in the table and reconstruct older versions on demand from *undo* records (newest-to-oldest). PostgreSQL's append-style approach makes reads of old versions cheap and rollback trivial, at the price of table bloat and the need for vacuum (Section 7).

---

## 3. Snapshots and the visibility rule

A **snapshot** answers one question: "which transactions count as committed for me?" In PostgreSQL it has three parts:

- `xmin`: the oldest transaction ID that was still running when the snapshot was taken. Everything below it is finished, committed or aborted.
- `xmax`: one past the highest transaction ID assigned so far. Anything at or above it started after the snapshot, so it is invisible.
- `xip[]`: the list of transaction IDs between `xmin` and `xmax` that were in progress at snapshot time. They are invisible too, even if they commit a millisecond later.

You can see this yourself with `SELECT pg_current_snapshot();`, which returns something like `110:115:110,113`, meaning xmin=110, xmax=115, in-progress={110, 113}.

A transaction ID `X` is **visible to snapshot S** when:

```
X < S.xmax  AND  X not in S.xip  AND  X committed
```

(Since everything below `S.xmin` is finished, only the commit status matters there.)

A tuple is **visible** to a transaction holding snapshot S when both halves hold:

1. **Its creator counts.** `xmin` is visible to S, *or* `xmin` is my own transaction and the tuple was created by an earlier command (`cmin` < current command).
2. **Its deleter doesn't count.** `xmax` is 0, *or* `xmax` aborted, *or* `xmax` is not visible to S (still running, in `xip`, or at/above `S.xmax`), *or* `xmax` is my own transaction but from the current or a later command.

The real code (`HeapTupleSatisfiesMVCC`) adds cases for row locks and subtransactions, but this is the logic.

### Worked example: three readers, one chain

Take the three-version chain above and suppose commit order was 100, 105, 112. Consider three snapshots:

**Snapshot A = `104:106:105`** (taken while 105 was still running)

- (0,1): xmin=100 visible. xmax=105 is in `xip`, so the delete doesn't count. **Visible.**
- (0,2): xmin=105 in `xip`. Not visible.
- (0,3): xmin=112 ≥ 106. Not visible.
- Reader sees **500**.

**Snapshot B = `110:111:110`** (after 105 committed, before 112 started)

- (0,1): xmax=105 committed and < 111 and not in `xip`. Deleted. Not visible.
- (0,2): xmin=105 visible; xmax=112 ≥ 111, so not visible to S. **Visible.**
- (0,3): xmin=112 ≥ 111. Not visible.
- Reader sees **400**.

**Snapshot C = `113:114:`** (after 112 committed)

- Only (0,3) passes both checks. Reader sees **450**.

Note the invariant: for any snapshot, **at most one version of a row is visible**, because each version's `xmax` equals the next version's `xmin`. If the transaction that would end one version is visible, so is the one that starts the next.

---

## 4. A concurrent timeline

Now let the clock run. Isolation level is REPEATABLE READ (one snapshot for the whole transaction). Balances start at A=500, B=300, both created by committed transaction 90.

| Time | T200 (transfer) | T201 (report) | Tuples after this step |
|------|-----------------|---------------|------------------------|
| t1 | `BEGIN` | | |
| t2 | | `BEGIN` | |
| t3 | `UPDATE … A = A-100` (gets XID 200) | | A: (90→200, 500), (200→0, 400) |
| t4 | | `SELECT sum(balance)`; snapshot taken: `200:201:200` | |
| t5 | `UPDATE … B = B+100` | | B: (90→200, 300), (200→0, 400) |
| t6 | `COMMIT` | | |
| t7 | | `SELECT sum(balance)` again | |
| t8 | | `COMMIT` | |

At t4 the report's snapshot lists 200 as in progress. Evaluating the A versions: the old one (xmax=200, in `xip`) is visible; the new one (xmin=200, in `xip`) is not. B has only its original version. Sum = 500 + 300 = **800**.

At t7, T200 has committed, but the report is still using the snapshot from t4, where 200 is listed as in progress. The `xip` list is frozen at snapshot time; a later commit does not change it. The report again sees 500 and 300 and gets **800**. No locks were taken, nobody waited, and the report saw one consistent state.

Under READ COMMITTED, PostgreSQL takes a **fresh snapshot for every statement**. The t7 query would get something like `202:202:` and see 400 + 400 = **800**. That is also consistent, but it is a *different* state from the t4 query. That is the non-repeatable read READ COMMITTED allows, and in a multi-statement report it can mix states: table X read before a commit, table Y after.

---

## 5. When writers meet: write-write conflicts

MVCC lets readers ignore writers, but two transactions updating **the same row** must still be ordered. PostgreSQL uses the row's `xmax` as a lock: a writer that finds `xmax` set by a still-running transaction waits for that transaction to finish.

What happens next depends on the isolation level.

**READ COMMITTED.** Two clerks each try to decrement stock for item 7 (stock = 10):

| Time | T300 | T301 |
|------|------|------|
| t1 | `UPDATE items SET stock = stock - 1 WHERE id = 7` → writes version stock=9, xmax of old = 300 | |
| t2 | | same `UPDATE`; finds old version's xmax = 300 running → **waits** |
| t3 | `COMMIT` | |
| t4 | | wakes; follows `ctid` to the newest version (stock=9), **re-checks the WHERE clause** on it, applies the update → stock=8 |

This re-check is called EvalPlanQual. The result is correct here (8), but notice T301's statement now mixes two views: it chose rows using its original snapshot and updated the newest version. With a WHERE clause that depends on the changed column, READ COMMITTED can therefore skip or double-apply rows in surprising ways. It is a pragmatic default, not a strong guarantee.

**REPEATABLE READ.** At t4, T301 discovers the row it wants to update was changed by a transaction its snapshot cannot see. Updating the newest version would silently overwrite a change T301 never read (a *lost update*), so PostgreSQL instead raises:

```
ERROR:  could not serialize access due to concurrent update
```

T301 must roll back and retry. This is the "first updater wins" rule of snapshot isolation. If T300 had aborted instead, T301 would proceed normally, since an aborted `xmax` does not count.

Applications using REPEATABLE READ or SERIALIZABLE must therefore be written with a **retry loop** around each transaction. That requirement is the price of not blocking.

---

## 6. Snapshot isolation is not serializable: write skew

PostgreSQL's REPEATABLE READ is **snapshot isolation (SI)**: each transaction reads from one snapshot and the first-updater-wins rule prevents lost updates. SI prevents dirty reads, non-repeatable reads, lost updates, and (unlike the SQL standard's minimum for REPEATABLE READ) phantoms. But it allows one classic anomaly: **write skew**.

### The on-call doctors

A hospital rule: at least one doctor must be on call per shift. Table `oncall(doctor, shift, on_call)` holds Alice and Bob, both on call for shift 1. Both feel ill and try to go off call at the same time, each in their own transaction:

```sql
BEGIN ISOLATION LEVEL REPEATABLE READ;
SELECT count(*) FROM oncall WHERE shift = 1 AND on_call;   -- check the rule
-- if count >= 2, it's safe for me to leave
UPDATE oncall SET on_call = false WHERE shift = 1 AND doctor = 'me';
COMMIT;
```

| Time | T_alice | T_bob |
|------|---------|-------|
| t1 | snapshot S_a; count = 2 | |
| t2 | | snapshot S_b; count = 2 |
| t3 | update Alice's row → on_call=false | |
| t4 | | update Bob's row → on_call=false |
| t5 | `COMMIT` ✓ | |
| t6 | | `COMMIT` ✓ |

Final state: nobody is on call. Each transaction's check was correct *for its snapshot*, and the two transactions updated **different rows**, so first-updater-wins never fired. There is no serial order (Alice then Bob, or Bob then Alice) that produces this outcome, because in either serial order the second doctor would see count = 1 and stay.

Write skew generalizes: two transactions read overlapping data, make decisions based on it, and write *disjoint* data that invalidates the other's premise. Double-booking a meeting room with two overlapping inserts is the same pattern.

### Fixing it

Options, from blunt to elegant:

1. **Materialize the conflict.** Have both transactions update a shared row (for example a per-shift counter), so first-updater-wins catches it.
2. **Lock explicitly.** `SELECT … FOR UPDATE` on the rows read. This turns the read into a write-lock, so Bob blocks on Alice's row lock. It doesn't help with phantoms (rows that don't yet exist), which is why the meeting-room case needs exclusion constraints or a lock on a parent row.
3. **Use a constraint.** PostgreSQL exclusion constraints (`EXCLUDE USING gist (room WITH =, during WITH &&)`) solve the double-booking case outright.
4. **Use SERIALIZABLE.** Let the database detect it.

### Serializable Snapshot Isolation

Since version 9.1, PostgreSQL's SERIALIZABLE level is **Serializable Snapshot Isolation (SSI)**, based on Cahill, Röhm and Fekete (2008) and implemented by Ports and Grittner (2012). It keeps SI's non-blocking reads and adds detection.

The theory: every non-serializable SI execution contains a "dangerous structure" in its dependency graph. A *rw-antidependency* T1 →rw T2 means T1 read a version that T2 later superseded, so T1 must come before T2 in any equivalent serial order. Fekete et al. proved that every SI anomaly has a transaction T_pivot with **both an incoming and outgoing rw-antidependency** among concurrent transactions: T_in →rw T_pivot →rw T_out (where T_in and T_out may be the same transaction).

To spot these, PostgreSQL records **SIREAD locks**: markers of what each serializable transaction has read, at tuple, page, or relation granularity. They block nothing; they exist so a writer can notice "someone concurrent read what I'm about to supersede." In the doctors example:

- Alice read Bob's row (via the count); Bob then updated it. Alice →rw Bob.
- Bob read Alice's row; Alice updated it. Bob →rw Alice.

That's a cycle with both transactions as pivots. When Bob tries to commit at t6, PostgreSQL raises:

```
ERROR:  could not serialize access due to read/write dependencies among transactions
```

Bob retries, takes a fresh snapshot, sees count = 1, and stays on call. Because SSI tracks a sufficient condition rather than full cycles, it can produce **false positives** (aborting transactions that were actually serializable), especially when SIREAD locks are coarsened from tuples to pages or relations under memory pressure. Declaring transactions `READ ONLY` helps, and `SERIALIZABLE READ ONLY DEFERRABLE` waits for a "safe" snapshot that can never take part in an anomaly and then runs with no tracking overhead at all.

### Isolation summary in PostgreSQL

| Level | Snapshot | Dirty read | Non-repeatable read | Phantom | Lost update | Write skew |
|-------|----------|-----------|---------------------|---------|-------------|------------|
| READ COMMITTED | per statement | no | possible | possible | possible* | possible |
| REPEATABLE READ (SI) | per transaction | no | no | no | no (aborts) | possible |
| SERIALIZABLE (SSI) | per transaction | no | no | no | no | no (aborts) |

\*In READ COMMITTED, `UPDATE … SET x = x - 1` is safe because of the EvalPlanQual re-check, but a read-modify-write split across a `SELECT` and a separate `UPDATE` can lose updates.

---

## 7. Garbage collection: vacuum

Every update and delete leaves a dead version behind. In the version chain from Section 2, once no snapshot anywhere could still need balance=500 or balance=400, those tuples are pure waste: they take space on the page, they make sequential scans read more, and index entries still point at them. Removing them is **vacuum**.

### When is a version dead?

A version is **removable** when its `xmax` committed and is older than every snapshot that could still look at it. PostgreSQL computes a horizon, roughly the oldest `xmin` across all running transactions' snapshots (plus replication slots and, with `hot_standby_feedback`, standby queries). Call it `OldestXmin`.

- If `xmax` committed and `xmax < OldestXmin`, every live snapshot sees the deletion. The tuple is **dead** and can be removed.
- If `xmax` committed but `xmax ≥ OldestXmin`, some snapshot might still need it. The tuple is **recently dead** and must stay.
- If `xmin` aborted, the tuple never existed for anyone and is removable right away.

### Worked example

Return to the chain, now with some activity:

```
  (0,1)  xmin=100  xmax=105  balance=500
  (0,2)  xmin=105  xmax=112  balance=400
  (0,3)  xmin=112  xmax=0    balance=450
```

Suppose that when vacuum runs, the only other open transaction is a REPEATABLE READ report with snapshot `108:109:108`, started before 112 and still running. `OldestXmin` = 108.

- (0,1): xmax=105, committed, 105 < 108. **Dead.** Remove.
- (0,2): xmax=112, committed, but 112 ≥ 108. The report can still see this version (for its snapshot, 105 is visible and 112 is not). **Recently dead.** Keep.
- (0,3): live. Keep.

When the report commits, `OldestXmin` moves past 112, and the next vacuum removes (0,2).

This is why a **single long-running transaction** is the classic cause of PostgreSQL bloat. A session left `idle in transaction` for six hours pins `OldestXmin` six hours in the past, and no dead version created since then, in *any* table of the database, can be reclaimed. Watch `pg_stat_activity` for old `backend_xmin` values, and consider `idle_in_transaction_session_timeout`.

### What vacuum actually does

A plain `VACUUM` (or autovacuum, which runs it automatically based on dead-tuple counts) does several things per table:

1. **Scans heap pages**, skipping pages the *visibility map* marks all-visible, and collects the TIDs of dead tuples.
2. **Removes index entries** pointing to those TIDs, in each index. This is usually the most expensive part.
3. **Reclaims heap space** by marking the line pointers unused and compacting the page. The space goes into the **free space map**, so future inserts and updates reuse it. The file usually does not shrink, except when trailing pages are entirely empty.
4. **Updates the visibility map**, marking pages where every tuple is visible to everyone. Index-only scans depend on this to avoid visiting the heap.
5. **Freezes old tuples** (below).

`VACUUM FULL` is different: it rewrites the whole table into a new file, returning space to the operating system, but holds an exclusive lock for the duration. Tools like `pg_repack` do the same online.

### Cheaper cleanup: HOT and pruning

If an update changes no indexed column and the new version fits on the same page, PostgreSQL performs a **heap-only tuple (HOT)** update: the indexes keep pointing at the original line pointer, and the chain is followed within the page. Any later access to that page can **prune** the chain, removing dead HOT versions and redirecting the line pointer, without a full vacuum and without touching indexes.

### Freezing and transaction ID wraparound

XIDs are 32 bits and compared using modular arithmetic: for any XID, about two billion IDs are "in the past" and two billion "in the future." A tuple with `xmin = 100` that stays around while the counter advances by more than 2^31 would suddenly appear to be from the *future*, and become invisible. That would look like data loss.

To prevent it, vacuum **freezes** tuples old enough that every transaction regards them as committed. Since 9.4 it does this by setting a flag in the tuple's infomask rather than overwriting `xmin`. A frozen tuple is treated as older than every XID. Each table tracks `relfrozenxid`, the oldest unfrozen XID it might contain. When `age(relfrozenxid)` exceeds `autovacuum_freeze_max_age` (200 million by default), autovacuum forces an aggressive anti-wraparound vacuum, even if autovacuum is otherwise disabled. If that cannot keep up (often because of a long transaction or an abandoned replication slot or prepared transaction holding back the horizon), PostgreSQL eventually refuses new XIDs to protect the data, and an administrator must intervene.

### How other engines collect garbage

InnoDB keeps old versions in undo logs and a background **purge** thread discards them once no read view needs them; a long transaction shows up as a growing "history list length." Oracle reads old versions from undo segments and, if they have been overwritten, fails long queries with `ORA-01555: snapshot too old`. The trade-off is universal: old versions must survive as long as some reader may need them, paid for in space, cleanup work, or failed queries.

---

## 8. Putting it all together: one more timeline

To close, here is a single timeline that touches every mechanism. Table `t(id, v)`, row id=1 with v=10, created by committed XID 50. Autovacuum is about to run.

| Time | T60 (RR) | T61 (RC) | T62 (RR) | Notes |
|------|----------|----------|----------|-------|
| t1 | `BEGIN`; `SELECT v` → snapshot `60:61:60`… sees 10 | | | T60 has no XID yet; it gets one on its first write |
| t2 | | `UPDATE t SET v=20` (XID 61) | | old: xmax=61; new: (61, 20) |
| t3 | | `COMMIT` | | |
| t4 | `SELECT v` → still 10 | | | same snapshot, 61 ≥ its xmax |
| t5 | | | `BEGIN`; `SELECT v` → 20 | new snapshot sees 61 committed |
| t6 | `UPDATE t SET v=v+1` | | | target version superseded by invisible committed XID 61 → **serialization error**; T60 rolls back |
| t7 | autovacuum runs | | | OldestXmin limited by T62's snapshot (≥ 62), so v=10 tuple (xmax=61 < horizon) is **dead** and removed; v=20 is live |
| t8 | T60 retries: fresh snapshot sees 20, writes 21 (XID 63), commits | | | v=20 tuple now has xmax=63; it is recently dead while T62 lives |
| t9 | | | `SELECT v` → 20 | T62's snapshot can't see 63 |
| t10 | | | `COMMIT` | next vacuum can remove v=20 |

Each line follows from the rules in this document: one visible version per snapshot, writers waiting on or aborting against concurrent writers, retries to resolve conflicts, and vacuum reclaiming a version only after the last snapshot that could see it is gone.

---

## 9. Closing thought

MVCC is not a free lunch, but it is a very good deal: it gives you consistent, non-blocking reads in exchange for space, background cleanup, and the discipline to retry when the database says two transactions could not both have happened in order.
