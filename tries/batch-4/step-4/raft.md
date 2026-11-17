# Raft, Explained by Working Through It

Raft is a consensus algorithm. It lets a group of servers agree on a single, ordered sequence of commands, and it keeps working as long as a majority of them are up and can talk to each other. Each server feeds that sequence, called the **replicated log**, into a deterministic state machine such as a key-value store. If every server applies the same commands in the same order, every server ends up in the same state. Raft's job is to make sure the logs agree.

Diego Ongaro and John Ousterhout designed Raft in 2014 to be easier to understand than Paxos. Paxos and Raft are about equally fault tolerant and fast. The difference is that Raft breaks the problem into parts you can reason about one at a time:

1. **Leader election**: exactly one server at a time takes client requests.
2. **Log replication**: the leader copies its log to the others and forces them to match it.
3. **Safety**: a committed entry is never lost or overwritten, even when leaders crash.
4. **Membership changes**: servers can be added or removed without stopping the cluster.

This document explains each part and then walks through it with concrete traces on a five-server cluster, S1 to S5.

---

## 1. The basic model

### Server roles

A Raft server is always in one of three roles:

- **Follower**: passive. It answers requests from leaders and candidates and does nothing on its own.
- **Candidate**: a server trying to become leader.
- **Leader**: handles all client requests and sends the log to the followers.

In normal operation there is one leader and every other server is a follower.

### Terms

Raft splits time into **terms**, numbered 1, 2, 3, and so on. Each term starts with an election. If a candidate wins, it leads for the rest of that term. If the vote splits, the term ends with no leader and a new term begins. A term has at most one leader.

Terms work as a logical clock. Every server stores a `currentTerm`, and every message carries the sender's term. Three rules follow:

- If a server sees a term higher than its own, it updates its term and steps down to follower immediately.
- If a server receives a request carrying an older term, it rejects the request.
- A stale leader, for example one that was cut off by a network partition, finds out it was replaced the first time it talks to anyone.

### State on each server

Each server writes three things to stable storage before it answers any RPC:

| Field | Meaning |
|---|---|
| `currentTerm` | latest term the server has seen |
| `votedFor` | candidate it voted for in `currentTerm` (or null) |
| `log[]` | entries, each holding a command and the term in which the leader received it |

Each server also keeps two values in memory. `commitIndex` is the highest log index known to be committed. `lastApplied` is the highest index already applied to the state machine. A leader additionally tracks two values for each follower. `nextIndex[f]` is the next index to send to follower f. `matchIndex[f]` is the highest index known to be replicated on f.

Raft log indexes start at 1. Every entry is identified by its **(index, term)** pair, which is the key to everything that follows.

### Two RPCs

All of core Raft uses two remote procedure calls:

- **RequestVote(term, candidateId, lastLogIndex, lastLogTerm)**: sent by candidates.
- **AppendEntries(term, leaderId, prevLogIndex, prevLogTerm, entries[], leaderCommit)**: sent by leaders to replicate entries. With an empty `entries[]`, it serves as a heartbeat.

A third RPC, InstallSnapshot, handles log compaction (Section 7).

---

## 2. Leader election

### The mechanism

Followers expect to hear from a leader regularly. Each follower runs an **election timer**, set to a random value in a range such as 150 to 300 ms. Any valid AppendEntries from the current leader resets the timer. If the timer fires, the follower assumes the leader is gone and starts an election:

1. Increment `currentTerm`.
2. Become a candidate and vote for itself.
3. Reset its own election timer.
4. Send RequestVote to every other server.

A server grants its vote only if all of these are true:

- the candidate's term is at least the server's `currentTerm` (a higher term first updates the voter's term and clears its `votedFor`);
- the server has not already voted for someone else in this term;
- the candidate's log is **at least as up-to-date** as the voter's own log.

"Up-to-date" has a precise meaning. Compare the terms of the two last entries, and the log with the higher last term wins. If the last terms are equal, the longer log wins. This check is the **election restriction**. Section 5 explains why it matters.

A candidate that collects votes from a majority (3 of 5) becomes leader and immediately sends heartbeats to assert its authority. If a candidate receives an AppendEntries from a leader whose term is at least its own, it steps down. If its timer fires again before it wins, it starts a new election in a new term.

Randomized timeouts make split votes rare and short. Usually one server times out first, wins, and sends heartbeats before anyone else's timer fires.

### Worked example: a 5-node election timeline

All five servers boot with `currentTerm = 0`, empty logs, and no leader. Each picks a random election timeout:

| Server | Timeout |
|---|---|
| S1 | 260 ms |
| S2 | 210 ms |
| S3 | 170 ms |
| S4 | 290 ms |
| S5 | 225 ms |

**t = 170 ms.** S3's timer fires first. S3 sets `currentTerm = 1`, `votedFor = S3`, and broadcasts:

```
RequestVote(term=1, candidateId=S3, lastLogIndex=0, lastLogTerm=0)
```

**t ≈ 175 ms.** Each of the other servers sees term 1 > 0, so it updates `currentTerm = 1`. None has voted in term 1, and S3's empty log is as up-to-date as their own empty logs. All four grant the vote and record `votedFor = S3`, which resets their timers.

| Voter | Term after | Vote |
|---|---|---|
| S1 | 1 | S3 |
| S2 | 1 | S3 |
| S4 | 1 | S3 |
| S5 | 1 | S3 |

**t ≈ 176 ms.** S3 has 2 votes (its own and the first reply), then 3. Three is a majority of five, so S3 becomes **leader for term 1**. It does not need to wait for the remaining replies. It sends empty AppendEntries heartbeats every 50 ms, which keeps everyone's timer from firing.

**A split vote.** Later, at t = 5,000 ms, S3 crashes. The followers stop getting heartbeats. Suppose S2 and S5 happen to time out within a millisecond of each other:

- S2 → term 2, votes for itself, sends RequestVote(term 2).
- S5 → term 2, votes for itself, sends RequestVote(term 2).
- S1 receives S2's request first and votes S2. Then S5's request arrives, and S1 refuses because it already voted in term 2.
- S4 receives S5's request first and votes S5. It refuses S2.
- S3 is down.

Result for term 2: S2 has 2 votes (S2, S1) and S5 has 2 votes (S5, S4). Neither has 3, so **term 2 has no leader**.

Both candidates pick fresh random timeouts. S2 draws 160 ms and S5 draws 240 ms. S2 fires first:

- S2 → term 3, RequestVote(term 3).
- S1, S4 and S5 each see term 3 > 2. They update their term and clear `votedFor`. S5 also drops from candidate back to follower, because a higher term always wins.
- All logs are equal, so all three grant. S2 has 4 votes and becomes **leader for term 3**.

When S3 eventually restarts, it still believes it is in term 1. The first heartbeat it receives carries term 3, so it updates its term and stays a follower. The two lessons: a split vote only costs one timeout period, and terms move forward monotonically for everyone.

---

## 3. Log replication

### The mechanism

When a client sends a command, the leader:

1. Appends `(index = lastIndex+1, term = currentTerm, command)` to its own log.
2. Sends AppendEntries to each follower in parallel, containing every entry from `nextIndex[f]` onward.
3. Once the entry is stored on a majority of servers (leader included), marks it **committed**, applies it to the state machine, and returns the result to the client.
4. Includes `leaderCommit` in later AppendEntries, so followers learn which entries are committed and apply them too.

The AppendEntries **consistency check** carries the correctness argument. The leader includes `prevLogIndex` and `prevLogTerm`, which identify the entry just before the new ones. A follower accepts the RPC only if its log has an entry at `prevLogIndex` whose term is `prevLogTerm`. If an existing entry conflicts with a new one (same index, different term), the follower deletes that entry and everything after it, then appends the new entries.

This produces the **Log Matching Property**:

- If two logs contain an entry with the same index and term, the entries hold the same command.
- If two logs contain an entry with the same index and term, the logs are identical in all preceding entries.

The first part holds because a leader creates at most one entry per index in its term and never moves it. The second follows by induction from the consistency check. Every successful AppendEntries confirms that the follower matches the leader up to `prevLogIndex`.

### Worked example: replicating three commands

Go back to term 1 with S3 as leader and every log empty. A client sends `x=1`, then `y=2`, then `x=3`.

S3 appends all three to its log:

```
index:  1      2      3
term:   1      1      1
cmd:    x=1    y=2    x=3
```

S3 has `nextIndex = 1` for every follower (a new leader initializes it to its own last index + 1, which was 1). It sends:

```
AppendEntries(term=1, leaderId=S3, prevLogIndex=0, prevLogTerm=0,
              entries=[(1,1,x=1),(2,1,y=2),(3,1,x=3)], leaderCommit=0)
```

Index 0 is the empty base of every log and always matches. Replies arrive:

| Follower | Result | matchIndex | nextIndex |
|---|---|---|---|
| S1 | success | 3 | 4 |
| S2 | success | 3 | 4 |
| S4 | (message lost) | 0 | 1 |
| S5 | (slow, no reply yet) | 0 | 1 |

S3 now counts copies of each index. Index 3 is on S3, S1 and S2, which is 3 of 5. Index 3 is a term-1 entry, the leader's current term, so S3 sets **`commitIndex = 3`**. It applies `x=1`, `y=2`, `x=3` to its state machine and replies to the client.

The next heartbeat carries `leaderCommit = 3`. S1 and S2 set their own `commitIndex = min(leaderCommit, index of last new entry) = 3` and apply the three commands. The leader retries S4 indefinitely, and the retry succeeds. S5 eventually answers too. Neither needs to be present for the commit, but both catch up.

Now the client sends `z=9`. S3 appends `(4, 1, z=9)` and sends S1 and S2 `AppendEntries(prevLogIndex=3, prevLogTerm=1, entries=[(4,1,z=9)], leaderCommit=3)`. Each follower checks that it has index 3 with term 1, appends, and replies. With S3, S1 and S2 holding index 4, the leader commits it.

---

## 4. Commit rules, precisely

The leader advances `commitIndex` to N when **all** of these hold:

1. N > `commitIndex`,
2. a majority of `matchIndex[i]` values (counting the leader itself) are ≥ N,
3. `log[N].term == currentTerm`.

Rule 3 is the subtle one. **A leader commits entries from earlier terms only indirectly.** Once an entry from the current term is committed, all preceding entries are committed with it, by the Log Matching Property. The leader never commits an old-term entry by counting its replicas.

Followers use a simpler rule. They set `commitIndex = min(leaderCommit, index of last new entry)`. Any server applies entries `lastApplied+1 .. commitIndex` in order.

### Why rule 3 exists: a trace

This is the classic counterexample, Figure 8 in the Raft paper, retold on our five servers. Each log below shows terms by index.

**Term 2.** S1 is leader. It appends index 2 with term 2 and replicates it only to S2, then crashes.

```
S1: [1, 2]      S2: [1, 2]      S3: [1]     S4: [1]     S5: [1]
```

**Term 3.** S5 runs for election. Its last entry has term 1, the same as the last entries of S3 and S4, and its log is just as long. They vote for it (S3, S4, S5 = majority). S5 receives a client command, appends index 2 with **term 3**, and crashes before replicating it.

```
S1: [1, 2]      S2: [1, 2]      S3: [1]     S4: [1]     S5: [1, 3]
```

**Term 4.** S1 restarts and wins with votes from S1, S2 and S3. It continues replicating, and S3 receives index 2 with term 2. Index 2 (term 2) now sits on S1, S2 and S3, which is a majority.

```
S1: [1, 2]      S2: [1, 2]      S3: [1, 2]  S4: [1]     S5: [1, 3]
```

Suppose S1 committed index 2 at this point because "it is on a majority". Then S1 crashes again.

**Term 5.** S5 runs. Its last term is 3, which beats the last term 2 of S2 and S3 and the last term 1 of S4. So S2, S3 and S4 all vote for it. S5 becomes leader and replicates its log, **overwriting index 2 on everyone with its term-3 entry**. An entry that S1 considered committed has disappeared. That violates safety.

**With rule 3**, S1 in term 4 does not commit index 2 by counting replicas. It first appends a term-4 entry at index 3, perhaps a no-op, and replicates it to S2 and S3:

```
S1: [1, 2, 4]   S2: [1, 2, 4]   S3: [1, 2, 4]   S4: [1]   S5: [1, 3]
```

Now index 3 (term 4) is on a majority and is in the current term, so S1 commits index 3, and index 2 is committed with it. If S1 crashes now, S5 cannot win. A majority of servers have last term 4 > 3, so it would need a vote from at least one of them, and none will vote for it. This is also why practical Raft implementations have a new leader **commit a no-op entry at the start of its term**. Doing so commits everything from earlier terms promptly.

---

## 5. Safety: why committed entries survive

The core guarantee is **Leader Completeness**. If an entry is committed in some term, it appears in the log of every leader of every later term. The argument is short:

- A committed entry is stored on a majority of servers.
- A new leader needs votes from a majority.
- Any two majorities of the same cluster share at least one server.
- That shared server holds the committed entry and votes only for a candidate whose log is at least as up-to-date as its own. Combined with the commit rule, this means the candidate's log must also contain the entry.

From Leader Completeness comes **State Machine Safety**. If a server has applied the entry at index i, no other server ever applies a different entry at index i. Raft also guarantees:

- **Election Safety**: at most one leader per term (one vote per server per term, and two majorities intersect).
- **Leader Append-Only**: a leader never deletes or overwrites entries in its own log. It only appends.

---

## 6. Leader crash and log repair: a full trace

When leaders crash, follower logs can diverge from the leader's in three ways. They can be **missing** entries, they can hold **extra uncommitted** entries from old terms, or both. Raft repairs this from the leader's side. The leader's log is authoritative, and conflicting follower entries are overwritten.

### Before the crash

S1 is leader in term 2. Logs (term of each entry by index):

```
index:   1  2  3  4  5
S1 (L):  1  1  2  2  2
S2:      1  1  2  2
S3:      1  1  2  2
S4:      1  1  2
S5:      1  1
```

Index 4 is on S1, S2 and S3. It is a term-2 entry in term 2, so **commitIndex = 4**. Index 5 is only on S1, so it is uncommitted. S4 is one entry behind and S5 is two behind, perhaps because of a slow link.

### The crash and the election

S1 crashes.

**Term 3.** S4 times out first. It sends `RequestVote(term=3, lastLogIndex=3, lastLogTerm=2)`.

- S5 (last entry index 2, term 1): S4's last term 2 > 1, so S4 is more up-to-date. **Grant.**
- S2 (last entry index 4, term 2): the last terms are equal, but S4's log is shorter. **Reject.**
- S3: same as S2. **Reject.**

S4 has 2 votes and loses. The election restriction just stopped a server that lacks committed entry 4 from becoming leader.

**Term 4.** S2 times out next. It sends `RequestVote(term=4, lastLogIndex=4, lastLogTerm=2)`. S3 (equal log), S4 and S5 (shorter logs) all grant. S2 is **leader for term 4**.

S2 initializes `nextIndex[f] = 5` for every follower and `matchIndex[f] = 0`. It appends a no-op at index 5 with term 4:

```
S2 (L):  1  1  2  2  4
```

### Repairing each follower

**S3 (missing one entry):**
```
→ AppendEntries(term=4, prevLogIndex=4, prevLogTerm=2, entries=[(5,4)], leaderCommit=4)
```
S3 has index 4 with term 2, so the check passes. It appends and replies success. `matchIndex[S3] = 5`, `nextIndex[S3] = 6`.

Index 5 is now on S2 and S3, which is two copies. That is not a majority yet.

**S4 (missing two entries):**
```
→ AppendEntries(prevLogIndex=4, prevLogTerm=2, entries=[(5,4)])
← reject: S4 has no index 4
```
S2 decrements `nextIndex[S4]` to 4 and retries:
```
→ AppendEntries(prevLogIndex=3, prevLogTerm=2, entries=[(4,2),(5,4)])
← success
```
`matchIndex[S4] = 5`. Index 5 is now on S2, S3 and S4, a majority, and it has the current term 4. **S2 commits index 5**, which also confirms the earlier entries.

**S5 (missing three entries):**
```
→ prevLogIndex=4 → reject (no index 4)
→ prevLogIndex=3 → reject (no index 3)
→ prevLogIndex=2, prevLogTerm=1, entries=[(3,2),(4,2),(5,4)] → success
```

**S1 restarts (has an extra, conflicting entry).** S1's log is `1 1 2 2 2`, and its index 5 is the uncommitted term-2 entry. It wakes in term 2. The first message it receives is from S2 with term 4, so S1 updates its term and becomes a follower.

```
→ AppendEntries(term=4, prevLogIndex=4, prevLogTerm=2, entries=[(5,4)])
```
S1's index 4 has term 2, so the check passes. The new entry at index 5 has term 4, but S1's index 5 has term 2. That is a **conflict**: S1 deletes index 5 and everything after it, then appends `(5,4)`.

Final state, all identical:

```
index:   1  1  2  2  4   (terms)
S1..S5:  1  1  2  2  4
```

The client command that S1 had placed at index 5 in term 2 was never committed, so the client never got a success reply. A client that times out retries the command, ideally with a unique request ID so the state machine can detect duplicates.

### Faster backup

Decrementing `nextIndex` one step per round trip can be slow for a follower that is far behind. A common optimization has the follower return a hint with its rejection, such as `conflictTerm` (the term of its entry at `prevLogIndex`) and `conflictIndex` (the first index it holds for that term, or its log length if it has nothing at that index). The leader can then skip back a whole term at a time. In the trace above, S5 would reply "my log length is 2" and S2 would jump straight to `nextIndex = 3`.

---

## 7. Log compaction and snapshots

Logs cannot grow without bound. Each server independently takes a **snapshot** of its state machine up to some applied index. It records `lastIncludedIndex` and `lastIncludedTerm`, then discards the log prefix. Sometimes a follower needs entries that the leader has already discarded, such as a new server or one that has been down for a long time. In that case the leader sends an **InstallSnapshot** RPC. The follower replaces its state with the snapshot. It keeps any log entries that follow the snapshot, as long as they are consistent with it, and discards the rest. The consistency check works the same way as before, with `(lastIncludedIndex, lastIncludedTerm)` serving as the base of the log.

---

## 8. Client interaction

- **Finding the leader.** Clients may contact any server. Followers reject the request and point to the leader they know about.
- **Exactly-once semantics.** Clients tag each command with a client ID and a sequence number. The state machine stores the latest result for each client and returns it instead of re-executing a duplicate.
- **Linearizable reads.** A leader cannot just answer from local state, because it might have been replaced without knowing it. The standard approach is **ReadIndex**. The leader makes sure it has committed an entry in its current term (the no-op), records `commitIndex` as the read index, and confirms it is still leader by getting heartbeat acknowledgments from a majority. It waits until `lastApplied ≥ readIndex` and then serves the read. Some systems use **leases** instead, which depend on bounded clock drift.

---

## 9. Membership changes

Production clusters need to replace failed machines and change size without downtime. The danger is that the cluster cannot switch every server from the old configuration C_old to the new one C_new at the same instant. While servers disagree, a majority under C_old and a separate majority under C_new could each elect a leader in the same term.

For example, growing from 3 servers {S1, S2, S3} to 5 servers {S1 … S5}: S1 and S2 still using C_old form a majority of 3, while S3, S4 and S5 using C_new form a majority of 5. Those two groups share no server, so two leaders could exist at once.

Raft avoids this by storing configurations **as special log entries**. A server always uses the latest configuration entry in its log, **whether or not that entry is committed**. There are two approaches.

### Approach A: Joint consensus

The change passes through a transitional configuration, C_old,new:

1. The leader appends a C_old,new entry and replicates it. While this configuration is in effect, every decision, both elections and commits, needs **separate majorities of both C_old and C_new**.
2. Once C_old,new is committed, neither C_old nor C_new can make decisions alone. The leader then appends a C_new entry.
3. Once C_new is committed, the old configuration is irrelevant. Servers that are not in C_new can shut down.

**Worked example: replace S5 with S6 (5 → 5).**
C_old = {S1, S2, S3, S4, S5}, C_new = {S1, S2, S3, S4, S6}. Leader is S1, term 7.

| Step | Log entry | Commit needs |
|---|---|---|
| 0 | (S6 joins as a non-voting learner and catches up) | n/a |
| 1 | index 40: C_old,new | 3 of {S1..S5} **and** 3 of {S1..S4,S6} |
| 2 | index 41: C_new | 3 of {S1..S4,S6} |
| 3 | S5 shuts down | n/a |

If S1 crashes between steps 1 and 2, any new leader must win majorities in both configurations. Either it holds C_old,new and continues the change, or it never received it and the change is effectively abandoned. In neither case can the two configurations elect separate leaders.

### Approach B: Single-server changes

Ongaro's dissertation describes a simpler approach that most implementations use, including etcd and Consul. **Add or remove only one server at a time.** When one server is added or removed, every majority of C_old overlaps every majority of C_new. Any two such majorities therefore share a voter, and no intermediate configuration is needed.

Rules:
- The leader appends the new configuration and starts using it immediately.
- It does not start another change until the current one is committed.
- **Safety fix:** a new leader must commit an entry in its own term, the no-op, before it begins a configuration change. Without this rule, a configuration entry left over from a previous leader's term can combine with a new change in a way that lets two disjoint majorities form. This bug was found after the dissertation was published.

**Worked example: replace S5 with S6, one server at a time.**

1. S6 starts as a **learner**, also called a non-voting member. The leader replicates the log to it (probably via InstallSnapshot) until it is nearly caught up. If S6 were added as a voter while empty, a 6-node cluster would need 4 of 6 for a commit. With S6 lagging, losing one more server could stall commits.
2. Leader appends `C1 = {S1..S6}` at index 50. Majority is 4 of 6. When it commits, S6 is a voter.
3. Leader appends `C2 = {S1, S2, S3, S4, S6}` at index 51. Majority is 3 of 5. When it commits, S5 is out and can be turned off.

Check the overlap at step 2: a majority of the old 5 (any 3) and a majority of the new 6 (any 4) together count 7 servers out of 6 distinct ones, so at least one server is in both.

### Two practical wrinkles

- **Removing the leader.** If C_new does not include the current leader, the leader keeps managing the cluster until C_new is committed, without counting itself toward majorities. It then steps down, and a server from C_new is elected.
- **Disruptive removed servers.** A removed server stops receiving heartbeats. It times out, increments its term, and sends RequestVote, which can force a working leader to step down. Raft prevents this two ways. First, servers ignore RequestVote if they have heard from a current leader within the minimum election timeout. Second, many implementations add **PreVote**, a dry-run election that does not increment anyone's term. A candidate only runs a real election if it would actually win, which also stops partitioned servers from inflating terms.

---

## 10. Putting it together

Here is Raft compressed into a checklist:

- **All servers:** if any RPC carries a term > `currentTerm`, adopt that term and become a follower. Apply committed entries in order.
- **Followers:** respond to RPCs. If the election timer expires, become a candidate.
- **Candidates:** increment the term, vote for yourself, and request votes. With a majority, become leader. On hearing from a valid leader, become a follower. On timeout, start over.
- **Leaders:** send heartbeats. Append client commands and replicate them. On a rejection, decrement `nextIndex` and retry. Commit index N when a majority have it **and** it belongs to the current term.
- **Voters:** one vote per term, only for candidates whose log is at least as up-to-date as their own.
- **Followers on AppendEntries:** reject if the term is stale or `prevLogIndex/prevLogTerm` does not match. Delete conflicting entries, append new ones, and advance `commitIndex` to `min(leaderCommit, last new index)`.

Raft is fast in the common case: a commit takes one round trip from the leader to a majority. During failures it gives up availability rather than correctness. With 5 servers it tolerates 2 failures, and with 2f+1 servers it tolerates f. Every rule above serves one invariant: **once an entry is committed, every future leader has it, so every state machine applies the same commands in the same order.**
