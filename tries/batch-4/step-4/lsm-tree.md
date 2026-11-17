# Log-Structured Merge Trees, Worked Through

A log-structured merge tree (LSM tree) is a storage engine design built around one idea: **never update data in place on disk**. Every write goes to memory first and is later written out to disk in large sorted batches. Over time, background work merges those batches together. The result is an engine that absorbs heavy write loads well. The price is paid on reads, which may have to check several places, and in background merge work that rewrites the same bytes more than once.

LevelDB, RocksDB, Cassandra, ScyllaDB, HBase, Bigtable, Pebble (CockroachDB) and the storage layers of many time-series and streaming systems use LSM trees. This explainer follows concrete data through every part of the structure, then puts numbers on the trade-offs.

---

## 1. Why not a B-tree?

A B-tree keeps one sorted copy of the data in fixed-size pages, such as 4 KB or 16 KB. To update a 100-byte row, the engine reads the page that holds it, changes it in memory and eventually writes the whole page back. With random keys, almost every write touches a different page, so:

- each logical write of 100 bytes can cost a 4–16 KB page write, an amplification of 40–160x before counting the journal;
- the writes land at random places on the device, which is slow on hard disks and wears flash unevenly.

An LSM tree turns many small random writes into a few large sequential ones. It buffers updates in memory, sorts them there and writes them out as one immutable file. Nothing on disk is ever modified, only created and later deleted. That one change shapes everything else in the design.

---

## 2. The moving parts

| Component | Where it lives | Mutable? | Purpose |
|---|---|---|---|
| Write-ahead log (WAL) | Disk, append-only | Append only | Durability for data still in memory |
| Memtable | RAM | Yes | Sorted buffer of recent writes |
| Immutable memtable | RAM | No | A full memtable waiting to be flushed |
| SSTable (sorted string table) | Disk | No | Sorted, indexed, immutable run of key-value pairs |
| Levels / tiers | Disk | Files are replaced | Groups of SSTables arranged by age and size |
| Bloom filters, block index | RAM (cached) and inside each SSTable | No | Let reads skip files and blocks |
| Manifest | Disk | Append only | Records which SSTables make up the current tree |

Every entry also carries a **sequence number**, a counter that increases with every write. When two entries share a key, the one with the higher sequence number wins. That rule is what lets the engine keep several versions of a key in different files without any of them being wrong.

---

## 3. A write, step by step

Suppose a fresh database and a client issuing these operations:

```
seq 101  PUT  user:42  -> {"name":"Ada"}
seq 102  PUT  user:07  -> {"name":"Lin"}
seq 103  PUT  user:42  -> {"name":"Ada L."}
seq 104  DELETE user:07
seq 105  PUT  user:99  -> {"name":"Kai"}
```

### 3.1 Write-ahead log

Each operation is first appended to the WAL. A WAL record is roughly:

```
[crc32][length][type=PUT][seq=101][key="user:42"][value=...]
```

The append is sequential. Whether the call returns before the bytes reach stable storage depends on the sync policy:

- **sync on every write**: the safest and slowest choice, bounded by `fsync` latency, which is often 0.1–2 ms on SSDs;
- **group commit**: concurrent writers share one `fsync`, so 50 writers waiting together pay for a single flush;
- **no sync**: the OS flushes when it chooses. A crash can lose the last few seconds of writes but will not corrupt the files.

After all five operations the WAL holds five records in arrival order. It is **not** sorted, and it does not need to be. It is only read during crash recovery.

### 3.2 Memtable

The same operations are inserted into the memtable, usually a skip list (RocksDB, LevelDB) or a balanced tree. It is kept sorted by key, then by sequence number descending:

```
user:07  seq=104  DELETE
user:07  seq=102  PUT {"name":"Lin"}
user:42  seq=103  PUT {"name":"Ada L."}
user:42  seq=101  PUT {"name":"Ada"}
user:99  seq=105  PUT {"name":"Kai"}
```

Note two things:

1. The overwrite of `user:42` did not replace the old entry. Both versions sit side by side. Some engines collapse them in the memtable, but any engine that supports snapshots must keep older versions while a snapshot can still see them.
2. The delete is not a removal. It is a new entry, a **tombstone**, saying "`user:07` is deleted as of sequence 104." Section 6 explains why.

A write is acknowledged once it is in the WAL (subject to the sync policy) and in the memtable. **No SSTable is touched on the write path.** That is why LSM writes are fast and why their latency is steady.

### 3.3 Crash recovery

If the process dies now, the memtable is lost but the WAL survives. On restart the engine replays WAL records 101–105 into a new memtable and rebuilds exactly the state above. Once a memtable has been flushed to an SSTable and the manifest records that SSTable, the matching WAL segment is no longer needed and is deleted.

---

## 4. Flushing a memtable to an SSTable

Memtables have a size limit, commonly 64 MB. When the active memtable fills:

1. It is marked **immutable**, and a new empty memtable plus a new WAL segment take over incoming writes. Writers do not wait.
2. A background thread iterates the immutable memtable in sorted order and writes an SSTable.
3. The new file is recorded in the manifest, and the old WAL segment is deleted.

### 4.1 SSTable layout

Pretend our five entries are the whole memtable. The flush writes them in key order:

```
SSTable 000001.sst
+--------------------------------------------------------------+
| Data block 0 (target ~4 KB, compressed)                      |
|   user:07 #104 DEL                                           |
|   user:07 #102 PUT {"name":"Lin"}                            |
|   user:42 #103 PUT {"name":"Ada L."}                         |
|   user:42 #101 PUT {"name":"Ada"}    <- may be dropped, see below |
|   user:99 #105 PUT {"name":"Kai"}                            |
+--------------------------------------------------------------+
| Filter block: bloom filter over {user:07, user:42, user:99}  |
| Index block: last key of each data block -> offset           |
|   "user:99" -> block 0 @ offset 0, size 212                  |
| Properties: num entries, min/max key, min/max seq, ...       |
| Footer: offsets of index + filter blocks, magic number       |
+--------------------------------------------------------------+
```

Whether the flush drops the shadowed `user:42 #101` depends on whether any open snapshot still needs it. With no snapshots, the flush keeps only the newest version of each key: `user:07 #104 DEL`, `user:42 #103` and `user:99 #105`. The tombstone for `user:07` must be kept, because older SSTables might still hold a live `user:07`.

Keys within a block are usually **prefix-compressed**: `user:42` after `user:07` is stored as "shared 5 bytes, then `42`". A restart point every 16 keys lets lookups binary-search inside a block.

### 4.2 What a real flush looks like

At realistic sizes: a 64 MB memtable of 100-byte values with 16-byte keys holds about 500,000 entries, counting per-entry overhead. With LZ4 or Snappy compression at a typical 2:1 ratio, the SSTable is roughly 30–35 MB, split into about 8,000 data blocks of 4 KB each. Its bloom filter at 10 bits per key is 500,000 × 10 / 8 ≈ 625 KB. The index block, with one entry per data block, is around 200 KB.

The flush is a single sequential write of about 33 MB. At 500 MB/s that takes about 70 ms of device time.

---

## 5. Reading: point lookups and bloom filters

A read must find the **newest** version of a key. It checks sources from newest to oldest and stops at the first hit:

1. active memtable
2. immutable memtables, newest first
3. Level 0 SSTables, newest first, since L0 files can overlap each other
4. Level 1, then Level 2, and so on, where each level has at most one candidate file

### 5.1 Worked lookup

Suppose the tree looks like this:

```
memtable:   user:15 #510
L0:         f9 [user:00 .. user:80]   f8 [user:10 .. user:95]
L1:         f5 [user:00 .. user:33]  f6 [user:34 .. user:66]  f7 [user:67 .. user:99]
L2:         ... 10 files ...
```

**GET user:42**

- Memtable: no `user:42`. A skip-list search costs nanoseconds.
- L0 f9: the range covers 42, so check f9's bloom filter. It says *definitely not present*. Skip, with no disk I/O.
- L0 f8: the range covers 42. The bloom filter says *maybe*. Binary-search the index block (cached), read one 4 KB data block, find `user:42 #103 PUT {"name":"Ada L."}`. Return it.

Total: one data-block read. L1 and L2 were never consulted, because the first hit is the newest version by construction.

**GET user:50**, a key that does not exist anywhere:

- Memtable: miss.
- f9 bloom: no. f8 bloom: no.
- L1: binary-search the file ranges, so only f6 can hold 50. Bloom: no.
- L2: one candidate file. Bloom: no.
- … and so on for each level.

Without bloom filters, this miss would cost one block read per L0 file plus one per level: with 2 L0 files and 5 levels, about 7 random reads. With bloom filters it usually costs **zero** reads.

### 5.2 Bloom filter numbers

A bloom filter with *m* bits, *n* keys and *k* hash functions has a false-positive rate of about

```
FPR ≈ (1 - e^(-k·n/m))^k
```

With the optimal *k* = (m/n)·ln 2:

| Bits per key | Optimal k | False-positive rate |
|---|---|---|
| 5 | 3 | ~9.2% |
| 8 | 6 | ~2.2% |
| 10 | 7 | ~0.82% |
| 16 | 11 | ~0.046% |

At the common setting of 10 bits per key, a lookup that misses has to probe 7 sources (2 L0 files and 5 levels). The expected number of wasted data-block reads is 7 × 0.0082 ≈ **0.057**. One read in about 17 pays a single unnecessary I/O.

The memory cost is real. A 1 TB database of 100-byte values holds about 10 billion keys. At 10 bits per key, that is 100 Gbit, or **12.5 GB** of filters. Engines respond in two ways. They cache filters per block, so cold ones can be evicted. And some, following the Monkey paper, give fewer bits per key to the biggest, deepest level, since it holds most of the keys but answers a small share of the lookups that reach it.

### 5.3 Range scans

Bloom filters are no help for `SCAN user:40 .. user:60`, because they only answer "is this exact key present?" A scan opens an iterator on every source that overlaps the range (memtables, overlapping L0 files and one file per level) and runs a **k-way merge** using a min-heap keyed on (key, seq desc). For each key, it emits only the newest version and skips the rest, including keys whose newest version is a tombstone.

With 2 memtables, 4 L0 files and 5 levels, a scan merges 11 iterators. Each needs at least one block read to start. That is why range-heavy workloads care most about keeping L0 small and the level count low. Prefix bloom filters, built over key prefixes such as `user:`, can help bounded prefix scans.

---

## 6. Deletes and tombstones

Deleting a key in place is impossible because SSTables are immutable. Simply leaving the key out of the next flush does not work either. Suppose `user:07 → Lin` already lives in an L3 file. If the memtable dropped the key on delete, a later GET would miss the memtable, find nothing in the newer files, reach L3 and return `Lin`. The deleted value would come back.

So a delete writes a **tombstone**, a marker with a sequence number that shadows every older version. Readers treat "the newest version is a tombstone" as "not found."

### 6.1 Tombstone lifecycle, traced

```
t0  L3 contains: user:07 #102 PUT "Lin"
t1  DELETE user:07            -> memtable: user:07 #104 DEL
t2  flush                     -> L0 file: user:07 #104 DEL
t3  compaction L0 -> L1       -> L1 file: user:07 #104 DEL      (still needed: L3 has older data)
t4  compaction L1 -> L2       -> L2 file: user:07 #104 DEL      (still needed)
t5  compaction L2 -> L3       merges #104 DEL with #102 PUT
                              -> both discarded, provided L3 is the bottom level,
                                 or no deeper level has the key,
                                 and no snapshot older than #104 is open
```

The tombstone can only disappear once it meets every older version of its key, or once the engine can prove no older versions exist below. Until then it costs space and slows reads.

### 6.2 Why tombstones hurt

- **Scan slowdown.** A queue-like pattern ("insert at tail, delete at head") can leave millions of tombstones at the front of the key range. Every scan from the head must step over them. Cassandra's warning threshold (`tombstone_warn_threshold`, 1,000 per query) exists because of this.
- **Space that is not reclaimed.** Deleting 100 GB of data first *adds* tombstones. The space comes back only after compaction carries them down to the old data.
- **Range tombstones.** `DELETE RANGE user:00 .. user:50` writes one range marker instead of 50 point tombstones. Readers must then check every key against a set of active range tombstones, which makes them more complex but saves a large amount of write volume.

RocksDB addresses the first two with options that pick files with many tombstones for compaction early (for example, `CompactOnDeletionCollector`).

---

## 7. Compaction: leveled vs tiered

Without compaction, every flush adds another SSTable. Reads would grow slower, and old versions and tombstones would never be removed. Compaction merges SSTables, keeps only the newest live version of each key (respecting snapshots) and writes new files. There are two main strategies.

### 7.1 Leveled compaction (LevelDB, RocksDB default)

Rules:

- **L0** holds flushed files, which may overlap. When it reaches a threshold (say 4 files), they are compacted into L1.
- **L1 and below**: within a level, files have **non-overlapping** key ranges, so the level as a whole is one sorted run.
- Each level has a size target that grows by a **fanout** T, typically 10: L1 = 256 MB, L2 = 2.56 GB, L3 = 25.6 GB, and so on.
- When level *i* exceeds its target, the engine picks a file there and merges it with **every overlapping file in level i+1**.

**Trace.** Use small numbers: 4 MB files, L1 target 40 MB (10 files), L2 target 400 MB.

```
State A
  L1: [a-c] [d-f] [g-i] [j-l] [m-o] [p-r] [s-u] [v-w] [x-y] [z]   = 40 MB (full)
  L2: 100 files, each covering ~1/100 of the keyspace          = 400 MB

L0 -> L1 compaction adds 4 MB to L1. L1 is now 44 MB > 40 MB target.

Pick L1 file [g-i] (round-robin cursor).
Overlapping L2 files: those covering g..i -> about 10 files (40 MB, since fanout = 10)
Read:  4 MB (L1) + 40 MB (L2) = 44 MB
Write: ~44 MB, minus anything removed (overwritten versions, tombstones)
Result:
  L1: 9 files (36 MB)  -- back under target
  L2: 100 - 10 + 11 = 101 files
```

That pass wrote 44 MB to move 4 MB down one level, **11x for this step**. On average, pushing a byte from level *i* to *i+1* rewrites about T bytes of the next level. Engines quote the per-level cost as roughly T in the worst case and closer to T/2 on average, depending on overlap and file boundaries.

### 7.2 Tiered (size-tiered) compaction (Cassandra STCS, ScyllaDB, RocksDB "universal")

Rules:

- Files of similar size are grouped into a **tier**.
- When a tier holds T files (say 4), they are merged into **one** file that joins the next tier up.
- Files within a tier overlap. Each tier holds up to T sorted runs.

**Trace** with T = 4 and 32 MB flushes:

```
flush 1..4    -> tier0: [32][32][32][32]           -> merge -> tier1: [128]
flush 5..8    -> tier0: 4×32                       -> merge -> tier1: [128][128]
flush 9..12   ->                                    -> tier1: [128][128][128]
flush 13..16  ->                                    -> tier1: [128][128][128][128]
                                                    -> merge -> tier2: [512]
...
flush 64      -> tier3: [2048]
```

Each byte is rewritten **once per tier**. With 4 tiers, a byte written at flush time is rewritten about 4 times over its life, whatever T is. Compare leveled, which rewrites about T bytes per level.

The costs:

- **Reads:** up to T overlapping runs per tier must be checked. With 4 tiers × 4 runs, that is up to 16 sorted runs, against about one per level in leveled.
- **Space:** before a merge finishes, both the inputs and the output exist. Merging the top tier can briefly need **2x** the live data size. Old versions of a key also survive in separate runs until they happen to be merged together.

### 7.3 Side-by-side on the same workload

Workload: 100 GB of unique logical data with frequent overwrites, written over time. Leveled uses T = 10 with L1 = 256 MB. Tiered uses T = 4 with 64 MB flushes.

| | Leveled (T=10) | Tiered (T=4) |
|---|---|---|
| Levels / tiers | L1–L4 (256 MB → 256 GB targets; L4 holds ~90 GB) | ~5 tiers (64 MB × 4^k up to ~64 GB) |
| Sorted runs a point read may touch | L0 (≤4) + 4 = ~8 | ~4 runs × 5 tiers = up to ~20 |
| Write amplification | ~1 (WAL) + 1 (flush) + ~1 (L0→L1) + ~5–10 per level × 3 ≈ **18–33** | ~1 (WAL) + 1 (flush) + 5 tier merges ≈ **7** |
| Space amplification (steady state) | ~**1.11x** | ~**1.5–2x** (up to ~2x+ during big merges) |
| Best for | Read-heavy, space-sensitive | Write-heavy, ingest bursts |

Many engines offer hybrids. RocksDB's leveled mode can run L0 as a tiered stage. ScyllaDB's incremental compaction splits large tier merges into fragments to cut the 2x space spike. Dostoevsky's "lazy leveling" uses tiering on all levels except the last.

---

## 8. Putting numbers on amplification

Three ratios describe any storage engine. An LSM tree lets you move cost between them, but it cannot shrink all three at once. This is the RUM conjecture: you can optimise for any two of Read, Update and Memory/space overhead, at the expense of the third.

### 8.1 Write amplification (WA)

```
WA = bytes written to the device / bytes written by the application
```

**Leveled, worked out.** Take a 1 TB database, 64 MB memtables, L1 = 256 MB, fanout 10:

```
L1 = 256 MB
L2 = 2.56 GB
L3 = 25.6 GB
L4 = 256 GB
L5 = 2.56 TB target (holds ~0.72 TB of the 1 TB)
```

That is 5 levels below L0. Per application byte:

- WAL: 1x
- flush to L0: 1x
- L0 → L1: about 1x (L0 and L1 are about the same size)
- L1→L2, L2→L3, L3→L4, L4→L5: about T/2 to T each, so 4 × (5–10) = 20–40x

Total: **about 23–43x**. RocksDB production reports often show 10–30x, because real workloads have skew and overwrites that compaction removes early. At 43x, an application writing 50 MB/s pushes about 2.1 GB/s at the SSD. That decides both whether the device can keep up and how fast it wears out. A 3.84 TB drive rated at 1 drive write per day for 5 years is good for about 7 PB, which at 2.1 GB/s lasts around 40 days.

**Tiered, same database:** with T = 4 and 64 MB flushes, tiers go 64 MB, 256 MB, 1 GB, 4 GB, 16 GB, 64 GB, 256 GB, 1 TB, so about 8 merges. WA ≈ 1 + 1 + 8 ≈ **10x**. Lowering the number of tiers with a larger T reduces WA further and makes reads worse.

**B-tree comparison:** 100-byte random updates into 16 KB pages, with the page written back about once per update batch, can easily see 50–150x, plus a journal. LSM trees win clearly on random small writes.

### 8.2 Read amplification (RA)

```
RA = I/Os (or bytes read) per logical read
```

For **point reads that hit**, in leveled, with filters and index blocks cached:

- Each probed source costs a bloom check in RAM. Only true positives and false positives cost a block read.
- Expected I/O ≈ 1 (the real hit) + (probed sources before the hit) × FPR ≈ 1 + 8 × 0.0082 ≈ **1.07 block reads**.

For **point reads that miss**: ≈ 8 × 0.0082 ≈ **0.07 block reads**.

If index and filter blocks are **not** cached, each probed source costs up to 2–3 extra I/Os (filter, index, data). RA then rises to 10–25. This is why sizing the block cache, and pinning L0 filters and indexes in it, matters so much.

For **range scans** of *r* entries: about one seek per sorted run, plus *r* × entry-size / block-size sequential block reads. Leveled with about 8 runs needs 8 seeks. Tiered with about 20 runs needs 20.

### 8.3 Space amplification (SA)

```
SA = bytes on disk / bytes of live logical data
```

**Leveled:** in steady state, the last level holds about T/(T+1) of all data. The upper levels together hold about 1/T as much again, and in the worst case all of it is out-of-date versions of last-level keys. So SA ≤ 1 + 1/T = **1.11x** at T = 10. RocksDB's `level_compaction_dynamic_level_bytes` sizes levels from the bottom up to keep this true even when the database is far smaller than the last level's target.

**Tiered:** if the largest tier holds the live data and the tiers above hold a full rewrite's worth of updates, SA approaches 2x. During the final merge, inputs and outputs exist side by side. In the worst case (every key overwritten and all tiers due to merge at once), SA can briefly exceed 2x. Planning for 50% free disk is the old rule of thumb for Cassandra STCS.

**Worked example.** 500 GB of live data:

- Leveled, T=10: about 555 GB on disk. A 1 TB volume is comfortable.
- Tiered, T=4: about 750 GB steady and up to about 1 TB during a top-tier merge. A 1 TB volume is at risk.

### 8.4 One table to remember

| Metric | Leveled (T=10, 5 levels) | Tiered (T=4, 8 tiers) | B-tree (16 KB pages) |
|---|---|---|---|
| Write amp | 15–40x | 6–12x | 50–150x random small writes |
| Point-read I/O (cached filters) | ~1.07 | ~1.1–1.3 | ~1 (inner pages cached) |
| Sorted runs per scan | ~8 | ~20–30 | 1 |
| Space amp | ~1.1x | 1.5–2x+ | ~1.3–1.5x (page fill factor) |

---

## 9. Full trace: one key's life

Follow one key from write to final merge, with T = 10, 4 MB L0 files and L1 = 40 MB.

```
10:00:00.000  PUT order:5512 -> {"status":"new"}       seq 9001
              WAL append (group commit with 31 other writers, one fsync, 0.4 ms)
              memtable insert (skip list, ~1 µs)
              ACK to client

10:00:00.250  PUT order:5512 -> {"status":"paid"}      seq 9044
              memtable now has #9044 and #9001

10:00:12.000  memtable reaches 64 MB -> becomes immutable, new memtable + WAL segment
10:00:12.080  flush writes 000214.sst to L0 (#9001 dropped: shadowed, no snapshot)
              manifest += 000214.sst ; WAL segment 37 deleted

10:00:40      L0 has 4 files -> compaction L0 -> L1
              merges 000211–000214 with overlapping L1 files -> new L1 files
              order:5512 #9044 now lives in L1

10:07:00      DELETE order:5512                          seq 51230 (tombstone)
10:07:20      flush -> L0 ; 10:07:55 L0 -> L1
              L1 now has the tombstone ... and #9044 was already merged into L2 at 10:05

10:30         L1 -> L2 compaction picks the file holding the tombstone
              merges with the L2 file holding #9044
              L2 is not the bottom level, but no file in L3+ overlaps order:5512
              and no snapshot older than 51230 is open
              -> both entries dropped. The key is gone and so is its space.

GET order:5512 at 10:08: memtable miss -> L0 bloom miss -> L1 hit: tombstone -> NOT FOUND
GET order:5512 at 10:31: every bloom check misses -> NOT FOUND, 0 data-block reads
```

The client wrote three times. On disk, the key's bytes were written to the WAL (3 records), L0 (2 files), L1 (2 rewrites) and L2 (1 rewrite plus the final merge). That is roughly 8–9 physical writes for 3 logical ones. A long-lived key that reached L5 would see the larger 20–40x figures from Section 8.

---

## 10. Practical tuning levers

- **Memtable size.** A larger memtable means fewer, bigger L0 files and fewer overwrites reaching disk. The costs are more RAM and longer WAL replay after a crash.
- **L0 triggers.** `level0_file_num_compaction_trigger` (for example 4) starts compaction. `slowdown` (for example 20) and `stop` (for example 36) throttle writers when compaction falls behind. A stall is the LSM version of "the bill came due."
- **Fanout T.** A higher T in leveled means fewer levels, so fewer read probes and lower space amplification, but higher write amplification per level. A higher T in tiered means fewer merges but more runs per tier for reads to check.
- **Bloom bits per key.** Going from 10 to 16 bits cuts the FPR roughly 18x for 60% more filter memory. That pays off when misses are common.
- **Compression per level.** Use none or LZ4 on L0–L1, where data is hot and short-lived, and ZSTD on the last level, where about 90% of the bytes live.
- **Rate-limit compaction** so its I/O does not crowd out foreground reads, and keep enough headroom that it never falls permanently behind.
- **Key-value separation** (WiscKey, RocksDB BlobDB, Badger): store large values in a separate log and keep only pointers in the LSM tree. With 4 KB values and 16-byte keys, compaction rewrites roughly 1% of the bytes it would otherwise rewrite. The trade-off is that values now need their own garbage collection.

---

## 11. Summary

An LSM tree accepts writes into a WAL and a sorted memtable, flushes them as immutable sorted files and keeps merging those files in the background. Every design choice is a choice of *when* to pay for sorting and cleaning up:

- **Writes** are cheap at the moment they happen: a sequential log append plus an in-memory insert.
- **Reads** pay for checking several sorted runs. Bloom filters cut point lookups to about one I/O, but range scans still merge every run.
- **Deletes** are writes too. Tombstones live until compaction carries them down to the data they shadow.
- **Compaction** is the bill. Leveled pays more in write amplification (about 10–40x) for tight space (about 1.1x) and fast reads. Tiered pays in space (up to 2x) and read fan-out for much lower write amplification (about 5–12x).

Choose leveled when reads and disk space matter most. Choose tiered when ingest throughput or SSD wear matters most. Choose a hybrid when you need some of both, which describes most production deployments.
