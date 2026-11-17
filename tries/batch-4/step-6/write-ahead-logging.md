# Write-Ahead Logging and ARIES Recovery

A database promises that a committed transaction's changes survive crashes (durability) and that an uncommitted transaction's changes never become visible as if they had happened (atomicity). It has to keep both promises while caching pages in memory, writing them to disk in whatever order is convenient, and handling thousands of concurrent transactions. Almost every serious relational engine does this with **write-ahead logging (WAL)**, and most of them base their recovery algorithm on **ARIES** (Algorithms for Recovery and Isolation Exploiting Semantics), published by C. Mohan and colleagues at IBM in 1992.

This explainer builds up the ideas from first principles. It then walks one concrete crash scenario through every phase of ARIES recovery, with the log, the dirty page table and the transaction table shown at each step.

---

## 1. The problem: buffers, disks and policies

A database keeps data in fixed-size **pages** on disk and caches them in a **buffer pool** in memory. Transactions read and modify the cached copies. At some point the modified (dirty) pages must be written back. Two policy questions decide how hard recovery will be:

| Policy question | Choice | Consequence |
|---|---|---|
| May a dirty page written by an *uncommitted* transaction be flushed to disk? | **Steal** (yes) / No-steal (no) | With steal, the disk can hold uncommitted data, so recovery must be able to **undo**. |
| Must all of a transaction's dirty pages be flushed *before* it commits? | **Force** (yes) / **No-force** (no) | With no-force, committed data may exist only in memory at crash time, so recovery must be able to **redo**. |

No-steal plus force is the simplest to recover from, since it needs neither undo nor redo, but it performs terribly. The buffer pool must hold every page a long transaction touched, and every commit triggers random writes scattered across the disk. Production systems choose **steal + no-force**, which gives the buffer manager full freedom and makes commits cheap. The cost is that recovery must support both undo and redo. The log is what makes that possible.

## 2. The write-ahead logging protocol

The log is an append-only sequence of records stored on durable media. Each record describes a change. Because the log is written sequentially, appending to it is far cheaper than updating data pages in place.

WAL rests on two rules:

1. **Undo rule (write-ahead).** Before a dirty page is written to disk, every log record describing changes to that page must already be on disk. If an uncommitted change reaches the data file, the information needed to reverse it is guaranteed to be durable.
2. **Redo rule (commit).** Before a transaction is reported as committed, all of its log records, including the commit record, must be on disk. If a committed change is still only in memory when the system crashes, the log holds enough to reapply it.

With these two rules, the data pages on disk can be in almost any state after a crash: some committed changes missing, some uncommitted changes present. The log is the source of truth, and recovery reconciles the pages with it.

### Log sequence numbers

Every log record gets a monotonically increasing **Log Sequence Number (LSN)**, often its byte offset in the log. LSNs show up in several places:

- **pageLSN**: stored in the header of every data page; the LSN of the most recent log record that modified that page. This is how recovery knows whether a page on disk already reflects a given change.
- **flushedLSN**: kept in memory; the highest LSN known to be durable on disk. The undo rule becomes a simple check: before writing page *P*, ensure `P.pageLSN <= flushedLSN`, forcing the log first if needed.
- **prevLSN**: stored in each log record; the LSN of the previous record written by the same transaction. Following prevLSN pointers walks one transaction's history backward without scanning the whole log.
- **recLSN**: kept per dirty page in the dirty page table; the LSN of the first record that dirtied the page since it was last flushed. Redo for that page cannot need to start any earlier.
- **undoNextLSN**: kept in compensation log records; explained below.

### Log record types

An ARIES log contains these kinds of records:

- **UPDATE**: `(LSN, prevLSN, txnID, pageID, redo info, undo info)`. Redo info says how to reapply the change and undo info says how to reverse it. ARIES uses **physiological logging**: the record names a physical page but describes a logical operation within it (for example, "insert key 42 in slot 7 of page P5"). The page can then reorganize itself internally without breaking the log.
- **COMMIT** and **ABORT**: mark a transaction's decision.
- **END**: marks that all work for a transaction, including any rollback, is finished and the transaction can be forgotten.
- **CLR (Compensation Log Record)**: written when an update is undone, whether during a normal rollback or during crash recovery. A CLR describes the undo action as a redo-only change and carries an **undoNextLSN** pointing to the next record of that transaction that still needs undoing (the prevLSN of the record just undone).
- **BEGIN_CHECKPOINT** and **END_CHECKPOINT**: bracket a fuzzy checkpoint.

CLRs are what let ARIES survive crashes during recovery. Because a CLR is redo-only, it is never undone itself, and because it points past the work already compensated, a rollback that is interrupted and restarted never repeats an undo step. The amount of log written during recovery is therefore bounded, even under repeated crashes.

## 3. The two in-memory tables

During normal operation, and again during recovery, the system maintains:

- **Transaction Table (TT)**, also called the Active Transaction Table: one entry per active transaction holding `txnID`, `status` (running, committing, or aborting), and `lastLSN` (its most recent log record).
- **Dirty Page Table (DPT)**: one entry per dirty page in the buffer pool holding `pageID` and `recLSN`. A page enters the DPT the first time it is dirtied after being clean, with `recLSN` set to that record's LSN. It leaves the DPT when it is flushed.

Neither table is written to disk on every change. They are snapshotted at checkpoints and rebuilt after a crash.

## 4. Fuzzy checkpoints

Without checkpoints, recovery would have to scan the log from the beginning of time. A naive checkpoint would stop all transactions and flush every dirty page, which is unacceptable in a live system. ARIES uses **fuzzy checkpoints**:

1. Write a `BEGIN_CHECKPOINT` record.
2. Write an `END_CHECKPOINT` record containing copies of the current TT and DPT. Transactions keep running between the two records, so the snapshot is "fuzzy": it reflects some moment between them.
3. Once the `END_CHECKPOINT` record is durable, write the LSN of `BEGIN_CHECKPOINT` to a well-known **master record** on disk.

No data pages are flushed as part of the checkpoint. The DPT snapshot tells recovery the oldest change that might be missing from disk, which is all redo needs. Many systems also run a background writer that trickles dirty pages to disk, keeping recLSNs recent and redo short.

## 5. ARIES recovery: three passes

After a crash, ARIES runs three passes:

1. **Analysis**: scan forward from the last checkpoint to rebuild the TT and DPT as they were at the crash, identify the **loser** transactions (those that must be rolled back), and find where redo must start.
2. **Redo**: scan forward from the smallest recLSN in the DPT and reapply every logged change whose effect is not already on disk, *including changes by losers and changes recorded in CLRs*. ARIES calls this **repeating history**: after redo, the database is in exactly the state it was in at the moment of the crash.
3. **Undo**: roll back all losers by walking their log chains backward in a single combined pass, writing CLRs as it goes.

Repeating history can look wasteful: why redo a loser's change just to undo it? The reason is that it makes the state before undo well defined. Physiological undo operations ("delete key 42 from page P5") assume the page looks the way it did right after the original operation. Repeating history guarantees that. It also lets ARIES support fine-grained locking and logical undo, such as undoing a B-tree insert on a different page after a split, which simpler schemes cannot.

The rest of this document traces a complete example.

---

## 6. Worked example: the scenario

Four pages each hold one value:

| Page | Item | Initial value | Initial pageLSN on disk |
|---|---|---|---|
| P1 | A | 100 | 0 |
| P2 | B | 200 | 0 |
| P3 | C | 300 | 0 |
| P4 | D | 400 | 0 |

Three transactions run concurrently. Here is the log. "Redo/undo" shows the after-value and before-value.

| LSN | prevLSN | Txn | Type | Page | Redo / Undo info | Notes |
|---|---|---|---|---|---|---|
| 10 | - | T1 | UPDATE | P1 | A: 100 -> 90 | |
| 20 | - | T2 | UPDATE | P2 | B: 200 -> 250 | |
| 30 | | | BEGIN_CHECKPOINT | | | |
| 40 | | | END_CHECKPOINT | | TT = {T1: lastLSN 10, T2: lastLSN 20}; DPT = {P1: recLSN 10} | |
| 50 | 10 | T1 | UPDATE | P3 | C: 300 -> 310 | |
| 60 | - | T3 | UPDATE | P4 | D: 400 -> 380 | |
| 70 | 20 | T2 | COMMIT | | | log forced to 70 |
| 80 | 50 | T1 | UPDATE | P1 | A: 90 -> 80 | |
| 90 | 70 | T2 | END | | | |
| 100 | 60 | T3 | UPDATE | P2 | B: 250 -> 275 | |
| 110 | 100 | T3 | ABORT | | | |
| 120 | 110 | T3 | CLR | P2 | B: 275 -> 250, undoNextLSN = 60 | rollback in progress |
| | | | **CRASH** | | | |

Background events the log does not show directly:

- **Between LSN 20 and 30**, the buffer manager flushed P2 to disk (B = 250, pageLSN 20). By the WAL rule, the log through LSN 20 was forced first. Because P2 was clean when the checkpoint ran, it is **not** in the checkpoint's DPT.
- **Between LSN 60 and 70**, P3 was flushed (C = 310, pageLSN 50). This is a *steal*: T1 has not committed, yet its change reached disk.
- The log was flushed through LSN 120 before the crash. Whatever was in the log buffer after that is gone, which is harmless because no page and no commit depended on it.

At the moment of the crash, the **disk** holds:

| Page | Value on disk | pageLSN on disk |
|---|---|---|
| P1 | A = 100 | 0 |
| P2 | B = 250 | 20 |
| P3 | C = 310 | 50 |
| P4 | D = 400 | 0 |

The correct end state is easy to state. T2 committed, so B must be 250. T1 never committed and T3 was aborting, so their changes must vanish: A = 100, C = 300, D = 400. Note what the disk gets wrong: C = 310 is an uncommitted value that has to be undone. The disk happens to show A = 100, but only because T1's changes were never flushed. Recovery cannot rely on luck like that.

## 7. Analysis pass

Recovery reads the master record, finds the BEGIN_CHECKPOINT at LSN 30, and scans forward. At END_CHECKPOINT (LSN 40) it loads the snapshot:

```
TT  = { T1: status=U, lastLSN=10 ;  T2: status=U, lastLSN=20 }
DPT = { P1: recLSN=10 }
```

(U = running or undo candidate, C = committed.) Then it processes each record:

| LSN | Record | Action | TT after | DPT after |
|---|---|---|---|---|
| 50 | T1 UPDATE P3 | T1.lastLSN = 50; P3 not in DPT, add with recLSN 50 | T1(U,50), T2(U,20) | P1:10, P3:50 |
| 60 | T3 UPDATE P4 | Add T3; add P4 | T1(U,50), T2(U,20), T3(U,60) | P1:10, P3:50, P4:60 |
| 70 | T2 COMMIT | T2.status = C, lastLSN 70 | T1(U,50), T2(C,70), T3(U,60) | unchanged |
| 80 | T1 UPDATE P1 | T1.lastLSN = 80; P1 already present, recLSN stays 10 | T1(U,80), T2(C,70), T3(U,60) | unchanged |
| 90 | T2 END | Remove T2 | T1(U,80), T3(U,60) | unchanged |
| 100 | T3 UPDATE P2 | T3.lastLSN = 100; add P2 with recLSN 100 | T1(U,80), T3(U,100) | P1:10, P3:50, P4:60, P2:100 |
| 110 | T3 ABORT | T3.lastLSN = 110, marked aborting | T1(U,80), T3(U,110) | unchanged |
| 120 | T3 CLR P2 | T3.lastLSN = 120; P2 already present | T1(U,80), T3(U,120) | unchanged |

Result of analysis:

```
TT  = { T1: U, lastLSN=80 ;  T3: U (aborting), lastLSN=120 }
DPT = { P1: 10,  P3: 50,  P4: 60,  P2: 100 }
RedoLSN = min(recLSN) = 10
Losers  = { T1, T3 }
```

Two points about this result:

- The DPT is **conservative**. It lists P3 as dirty even though P3 was flushed at pageLSN 50; analysis cannot see buffer flushes, because they are not logged. That is safe, and the redo pass sorts it out. Some systems write "page flushed" records to tighten this estimate.
- Had T2 committed but not yet written END, it would remain in the TT with status C. At the end of analysis or redo, ARIES simply writes the missing END record, since a committed transaction needs no undo.

## 8. Redo pass

Redo scans forward from RedoLSN = 10. For each UPDATE or CLR on page *P* at LSN *L*, it **skips** the record if any of the following holds:

1. *P* is not in the DPT. The page was flushed after this change and has not been dirtied since.
2. *P* is in the DPT but `L < P.recLSN`. This change happened before the page was last cleaned.
3. After fetching *P*, `P.pageLSN >= L`. The page on disk already contains the change.

Checks 1 and 2 cost no I/O. Only check 3 reads the page. If none applies, redo reapplies the change and sets `P.pageLSN = L`. Redo writes **no** log records.

| LSN | Record | Decision | Effect |
|---|---|---|---|
| 10 | T1 UPDATE P1 | P1 in DPT, 10 >= recLSN 10; fetch: pageLSN 0 < 10 | **Redo**: A = 90, P1.pageLSN = 10 |
| 20 | T2 UPDATE P2 | P2 in DPT, but 20 < recLSN 100 | **Skip** (no I/O) |
| 30, 40 | checkpoint | not a page change | skip |
| 50 | T1 UPDATE P3 | P3 in DPT, 50 >= 50; fetch: pageLSN 50 >= 50 | **Skip** (already on disk) |
| 60 | T3 UPDATE P4 | fetch: pageLSN 0 < 60 | **Redo**: D = 380, P4.pageLSN = 60 |
| 70 | T2 COMMIT | not a page change | skip |
| 80 | T1 UPDATE P1 | P1 in memory, pageLSN 10 < 80 | **Redo**: A = 80, P1.pageLSN = 80 |
| 90 | T2 END | | skip |
| 100 | T3 UPDATE P2 | 100 >= recLSN 100; fetch: pageLSN 20 < 100 | **Redo**: B = 275, P2.pageLSN = 100 |
| 110 | T3 ABORT | | skip |
| 120 | T3 CLR P2 | pageLSN 100 < 120 | **Redo**: B = 250, P2.pageLSN = 120 |

State in the buffer pool after redo, which is exactly the state at the instant of the crash:

| Page | Value | pageLSN |
|---|---|---|
| P1 | A = 80 | 80 |
| P2 | B = 250 | 120 |
| P3 | C = 310 | 50 |
| P4 | D = 380 | 60 |

Redo dutifully reapplied T1's and T3's uncommitted changes, and the CLR at 120 as well. That is repeating history. LSN 20 shows the cheap skip: the DPT proved P2's change was already on disk without reading the page. LSN 50 shows the pageLSN skip.

## 9. Undo pass

Undo rolls back every loser at once. It keeps a set **ToUndo** holding the lastLSN of each loser, and repeatedly picks the **largest** LSN in the set. Processing LSNs in descending order means a single backward sweep of the log serves all losers.

- If the record is an **UPDATE**: undo it, write a CLR whose undoNextLSN is the update's prevLSN, and apply the CLR to the page (setting pageLSN to the CLR's LSN). If prevLSN is null, the transaction is fully undone, so write END. Otherwise add prevLSN to ToUndo.
- If the record is a **CLR**: do nothing to the data. Add its undoNextLSN to ToUndo, or write END if it is null.
- If the record is an **ABORT** (or another non-update record): add its prevLSN to ToUndo.

Initial `ToUndo = {80 (T1), 120 (T3)}`.

| Step | Pick | Record | Action | New log record | ToUndo after |
|---|---|---|---|---|---|
| 1 | 120 | T3 CLR, undoNext 60 | Already compensated; jump to 60 | none | {80, 60} |
| 2 | 80 | T1 UPDATE P1 (A: 90 -> 80) | Restore A = 90 | **130**: CLR T1 P1 A: 80 -> 90, prevLSN 80, undoNext 50 | {60, 50} |
| 3 | 60 | T3 UPDATE P4 (D: 400 -> 380) | Restore D = 400 | **140**: CLR T3 P4 D: 380 -> 400, prevLSN 120, undoNext null | {50} |
| 3a | | T3 fully undone | | **150**: END T3, prevLSN 140 | {50} |
| 4 | 50 | T1 UPDATE P3 (C: 300 -> 310) | Restore C = 300 | **160**: CLR T1 P3 C: 310 -> 300, prevLSN 130, undoNext 10 | {10} |
| 5 | 10 | T1 UPDATE P1 (A: 100 -> 90) | Restore A = 100 | **170**: CLR T1 P1 A: 90 -> 100, prevLSN 160, undoNext null | {} |
| 5a | | T1 fully undone | | **180**: END T1, prevLSN 170 | {} |

Step 1 is the important one. T3 had already undone LSN 100 before the crash, and the CLR at 120 recorded that. Undo does not undo LSN 100 a second time. It follows undoNextLSN straight to 60. Without CLRs, recovery would have no reliable way to tell which parts of an in-progress rollback were complete.

Final state:

| Page | Value | Correct? |
|---|---|---|
| P1 | A = 100 | T1 rolled back |
| P2 | B = 250 | T2's committed value; T3's change undone |
| P3 | C = 300 | T1's stolen write reversed |
| P4 | D = 400 | T3 rolled back |

The database is consistent and open for business. The dirty pages can be flushed lazily like any others, because the log through LSN 180 protects them.

## 10. Crashing during recovery

Suppose the system crashes again right after LSN 160 reaches disk, before 170 and 180 are written. Restart runs the same three passes:

- **Analysis** scans to 160. T3 has an END at 150, so it is gone. T1's lastLSN is 160, a CLR with undoNextLSN 10.
- **Redo** repeats history again, *including CLRs 130, 140 and 160*, so A = 90, D = 400, C = 300 are restored from the log wherever the pages on disk lack them.
- **Undo** starts with `ToUndo = {160}`. The CLR at 160 sends it to 10, it writes one CLR for LSN 10 and then END.

Only the remaining work gets done. However many times the system crashes during recovery, each original update is compensated at most once, and the log grows by a bounded amount. This idempotence comes from three mechanisms working together: pageLSN comparisons make redo idempotent, CLRs are never undone, and undoNextLSN skips work that is already compensated.

## 11. Normal-time rollback uses the same machinery

LSNs 110 and 120 show that ARIES handles an ordinary `ROLLBACK` exactly like crash undo. The transaction writes ABORT, walks its own prevLSN chain backward, writes a CLR for each update it reverses, and finally writes END. Because one code path serves both cases, the crash path gets exercised constantly in production rather than sitting untested until disaster strikes. ARIES can also roll back partially to a **savepoint** by stopping the backward walk at the savepoint's LSN.

A related tool is the **nested top action**. Some changes, such as a B-tree page split, should persist even if the transaction that triggered them later aborts, because other transactions may already depend on the new structure. ARIES logs the split's updates normally and then writes a dummy CLR whose undoNextLSN points to the record just before the split. Any later undo jumps over the split entirely.

---

## 12. Group commit

The redo rule says a transaction cannot be acknowledged until its commit record is durable, and durability means an `fsync` or an equivalent flush. On a disk where a flush takes 5 ms, flushing once per commit caps the system near 200 commits per second, however fast the CPU is. Even on NVMe, where flushes cost tens of microseconds, per-commit flushes waste device bandwidth and burn CPU in syscalls.

**Group commit** amortizes one flush over many transactions. Commit records from different transactions are appended to the shared in-memory log buffer, and a single flush makes all of them durable together.

Example timeline (flush latency 5 ms):

| Time (ms) | Event | Log buffer tail | flushedLSN |
|---|---|---|---|
| 0.0 | T4 appends COMMIT at LSN 200; becomes flush leader, issues fsync covering up to 200 | 200 | 190 |
| 0.4 | T5 appends COMMIT at 210; a flush is in flight, so T5 waits | 210 | 190 |
| 1.1 | T6 appends COMMIT at 220; waits | 220 | 190 |
| 2.3 | T7 appends COMMIT at 230; waits | 230 | 190 |
| 5.0 | Flush completes; T4 acknowledged | 230 | 200 |
| 5.0 | T5 becomes leader; one fsync covers 210..230 | 230 | 200 |
| 10.0 | Flush completes; T5, T6, T7 all acknowledged together | 230 | 230 |

Two flushes made four transactions durable. Under heavier load, a single flush might cover dozens or hundreds of commits, so throughput scales with concurrency even though each transaction still waits roughly one or two flush latencies. Some systems add a short deliberate delay before flushing (`commit_delay` in PostgreSQL, `binlog_group_commit_sync_delay` in MySQL) to gather more transactions per flush, trading a little latency for throughput.

Group commit changes nothing about correctness. Each transaction is acknowledged only once `flushedLSN` is at least its commit LSN, so the redo rule holds exactly. It also interacts cleanly with the undo rule: a page whose pageLSN is above flushedLSN simply waits for the next group flush before the buffer manager may write it.

A related, weaker option is **asynchronous commit** (`synchronous_commit = off` in PostgreSQL, `innodb_flush_log_at_trx_commit = 2` in MySQL). Here the system acknowledges a commit before the flush. The database stays consistent after a crash, because WAL still protects page writes, but the last few hundred milliseconds of acknowledged commits may be lost. This deliberately gives up durability for latency, and it should be an explicit choice.

## 13. WAL in practice

- **Real systems.** SQL Server and Db2 follow ARIES closely. InnoDB pairs a redo log with separate undo logs and rolls back incomplete transactions after replaying redo. PostgreSQL's recovery is mostly a redo pass, because MVCC leaves aborted tuple versions invisible until VACUUM removes them.
- **Torn pages.** A crash partway through a multi-sector page write can leave a page that is half old and half new. Systems detect this with checksums and repair it from a full-page image in the log (PostgreSQL) or a doublewrite buffer (InnoDB) before applying redo.
- **Log truncation.** The log can be discarded only before both the oldest recLSN in the DPT and the oldest record an active transaction might still need to undo. One long-running transaction can therefore make the log grow without bound.
- **Beyond recovery.** Shipping the log to a standby gives physical replication. Archiving it allows point-in-time recovery, and decoding it powers change data capture.

## 14. Summary

- **Steal + no-force** gives the buffer manager freedom and makes commits cheap. In exchange, recovery must be able to both undo and redo.
- **WAL's two rules** make that possible. Log records reach disk before the pages they describe, and a transaction's records reach disk before its commit is acknowledged.
- **LSNs** tie everything together: pageLSN says what a page contains, recLSN bounds where redo must start, prevLSN chains a transaction's records, and undoNextLSN tells undo what remains.
- **Fuzzy checkpoints** bound recovery work without pausing the system or forcing pages.
- **ARIES** recovers in three passes. Analysis rebuilds the transaction and dirty page tables, redo repeats history so the database matches the moment of the crash, and undo rolls back the losers while logging CLRs so that the work survives further crashes.
- **Group commit** lets one log flush make many transactions durable, so commit throughput grows with concurrency instead of being capped by flush latency.

In the worked example, the disk held a mix of stale committed data and stolen uncommitted data. The log and the three-pass algorithm turned that state into exactly the correct database: T2's committed write preserved, T1's and T3's writes erased, and an interrupted rollback finished without repeating a step.
