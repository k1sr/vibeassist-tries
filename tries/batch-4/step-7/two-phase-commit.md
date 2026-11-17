# Two-Phase Commit: A Complete Explainer

## 1. The problem

A single database makes a transaction atomic by writing to one log. Either the commit record reaches stable storage or it does not, and recovery replays or discards work based on that one fact. Once a transaction touches several independent systems (two database shards, a message broker and a database, three microservices with their own storage), there is no single log. Each system can decide on its own to commit or abort, and a crash at the wrong moment can leave the money debited in one place and never credited in another.

The **two-phase commit protocol (2PC)** is the classic answer. It gives a group of independent resource managers a way to reach one atomic decision: every participant commits, or every participant aborts. It does not make failures go away. It makes sure that no failure can produce a *mixed* outcome, at the cost of sometimes having to wait.

## 2. Roles and vocabulary

- **Coordinator (C)**: the process that drives the protocol. Often this is the transaction manager embedded in the application server, or one of the participants elected for the job.
- **Participants (P1, P2, P3)**: the resource managers holding the data: databases, queues, shards. Each can do local work, hold locks, and write a durable local log.
- **Vote**: a participant's answer to "can you commit?" A YES vote is a promise. NO is a veto.
- **Prepared state**: the state a participant enters after voting YES. It has made its changes durable *and* given up the right to abort on its own.
- **Decision**: COMMIT or ABORT, chosen only by the coordinator.

The protocol rests on two rules:

1. **Unanimity.** The coordinator may decide COMMIT only if every participant voted YES. One NO, or one missing vote, means ABORT.
2. **Irrevocability.** Once a participant votes YES, it must be able to commit even if it crashes and restarts. Once the coordinator logs a decision, it can never change it.

## 3. The protocol, step by step

### Phase 1: Prepare (voting)

1. The coordinator writes a `BEGIN` or `PREPARING` record listing the participants. (Some variants skip this; see the presumed-abort optimisation below.)
2. The coordinator sends `PREPARE(T)` to every participant.
3. Each participant checks whether it can commit: constraints satisfied, no deadlock victim, disk available. If yes, it force-writes its redo and undo information plus a `PREPARED(T)` record to its log, keeps its locks, and replies `VOTE-YES`. If no, it writes `ABORT(T)` locally, rolls back, releases locks, and replies `VOTE-NO`.

### Phase 2: Commit (decision)

4. If every vote is YES, the coordinator force-writes `COMMIT(T)` to its log. **This write is the commit point of the whole distributed transaction.** If any vote is NO or a timeout fires, it writes `ABORT(T)` instead.
5. The coordinator sends the decision to every participant.
6. Each participant writes the decision to its log, applies it (makes changes visible or rolls them back), releases its locks, and replies `ACK`.
7. When every ACK has arrived, the coordinator writes `END(T)` and may forget the transaction.

The "force-write" is essential. A log record only counts once it is on stable storage, because the whole point is surviving a crash.

## 4. Worked example: the happy path

Scenario: a bank transfer of $100. P1 holds Alice's account, P2 holds Bob's account, P3 is an audit ledger that must record every transfer. Time runs downward.

```
 t   Coordinator C              P1 (Alice)          P2 (Bob)            P3 (Audit)
 --  -------------------------  ------------------  ------------------  ------------------
 0   log BEGIN(T1,{P1,P2,P3})
 1   --- PREPARE(T1) ---------> recv                recv                recv
 2                              lock row, -100      lock row, +100      append entry
 3                              force PREPARED      force PREPARED      force PREPARED
 4   <-- VOTE-YES ------------- send                send                send
 5   all 3 YES
 6   force COMMIT(T1)           <== commit point ==>
 7   --- COMMIT(T1) ----------> recv                recv                recv
 8                              force COMMIT        force COMMIT        force COMMIT
 9                              release locks       release locks       release locks
10   <-- ACK ------------------ send                send                send
11   log END(T1), forget
```

Message count: 3 PREPARE + 3 VOTE + 3 COMMIT + 3 ACK = 12 messages, i.e. 4n for n participants. Forced log writes: 1 at the coordinator (COMMIT) plus 2 per participant (PREPARED, COMMIT) = 7. The latency on the critical path is two round trips plus the forced writes.

## 5. Worked example: a NO vote

Same transaction, but Alice has only $60.

```
 t   Coordinator C              P1 (Alice)          P2 (Bob)            P3 (Audit)
 1   --- PREPARE(T2) --------->
 2                              check fails         lock, +100          append
 3                              log ABORT, undo     force PREPARED      force PREPARED
 4   <-- VOTE-NO -------------- send                VOTE-YES            VOTE-YES
 5   force ABORT(T2)
 6   --- ABORT(T2) -----------> (already aborted)   recv                recv
 7                                                  undo, release       undo, release
 8   <-- ACK ------------------ ACK                 ACK                 ACK
 9   log END(T2)
```

A participant that votes NO can abort unilaterally right away. It already knows the outcome, since one NO forces ABORT. The coordinator still sends ABORT to the participants that voted YES, because they are prepared and cannot act on their own. In practice the coordinator does not have to send ABORT to P1 at all, and need not wait for P2 and P3's votes before aborting: the first NO decides the outcome.

## 6. Failure cases and recovery traces

What makes 2PC worth studying is how it behaves when things break. The cases below go through each point where a crash or lost message can happen. The governing principle never changes: **a participant's log tells it what it promised; the coordinator's log tells everyone what was decided.**

### Case A: A participant crashes before voting

P3 crashes at t=2, before writing PREPARED.

```
 t   C                          P1           P2           P3
 1   --- PREPARE(T3) -------->
 3                              PREPARED     PREPARED     ** CRASH **
 4   <-- YES, YES; P3 silent
 ... timeout waiting for P3
 9   force ABORT(T3)
10   --- ABORT --------------> undo         undo         (down)
 ...
20                                                        restart: log has no
                                                          PREPARED(T3) -> abort
                                                          locally, discard work
```

**Recovery rule:** a participant that finds no PREPARED record for a transaction may abort it unilaterally. It never promised anything. The coordinator's timeout on votes is safe for the same reason: nobody can have committed yet.

### Case B: A participant crashes after voting YES

P2 crashes at t=5, after sending YES but before receiving the decision.

```
 t   C                          P1           P2              P3
 4   <-- YES, YES, YES
 6   force COMMIT(T4)
 7   --- COMMIT -------------> commit       ** CRASH **     commit
 8   <-- ACK (P1, P3)
 ... C keeps retrying COMMIT to P2
15                                           restart: finds PREPARED(T4),
                                             no decision -> state UNCERTAIN
16                                           re-acquire locks from log
17                                           --- QUERY(T4) --> C
18   <-- COMMIT(T4) ---------------------------
19                                           redo, force COMMIT, release
20   <-- ACK
21   log END(T4)
```

**Recovery rule:** a participant that finds PREPARED but no decision is *in doubt*. It must not decide on its own. It reacquires the locks recorded in its log, so no other transaction sees half-applied state, and asks the coordinator (or waits to be told). This is why PREPARED must be force-written along with enough redo/undo information to finish either way.

### Case C: A participant crashes after receiving the decision

If P2 had force-written COMMIT(T4) before crashing, on restart it simply redoes the commit from its log and re-sends ACK if asked. No coordination needed. Decisions are idempotent: receiving COMMIT twice is harmless.

### Case D: A message is lost

- *Lost PREPARE or lost vote:* the coordinator times out and aborts. Safe, same reasoning as Case A.
- *Lost decision:* the participant stays prepared and in doubt. Either the coordinator retransmits (it has not received an ACK) or the participant times out and sends QUERY. Either way it learns the logged decision.
- *Lost ACK:* the coordinator retransmits the decision; the participant replies ACK again. The coordinator cannot write END and forget until all ACKs arrive, so it never loses information a participant might still need.

### Case E: The coordinator crashes before deciding

C crashes after sending PREPARE but before writing any decision.

```
 t   C                          P1           P2           P3
 1   --- PREPARE(T5) -------->
 3                              PREPARED     PREPARED     PREPARED
 4   ** CRASH **                YES -->x     YES -->x     YES -->x
 ...
12   restart: log shows BEGIN(T5), no decision
13   decide ABORT, force ABORT(T5)
14   --- ABORT --------------> undo         undo         undo
15   <-- ACK x3; log END(T5)
```

**Recovery rule:** a coordinator that recovers with no decision for a transaction may abort it. No COMMIT was logged, so no participant can have been told to commit. Note what happened during t=4 to t=12, though. All three participants sat prepared, holding locks, unable to do anything. That gap is the blocking problem, covered in section 7.

### Case F: The coordinator crashes after deciding, before (or while) sending

```
 t   C                          P1           P2           P3
 4   <-- YES x3
 6   force COMMIT(T6)
 7   --- COMMIT -------------> recv, commit
     ** CRASH **                             (no msg)     (no msg)
 ...
20   restart: log shows COMMIT(T6), no END
21   --- COMMIT(T6) ---------> dup, ACK     commit       commit
22   <-- ACK x3; log END(T6)
```

**Recovery rule:** a coordinator that recovers with a decision but no END re-sends that decision to every participant until all ACK. Here P1 has already committed and P2 and P3 have not. That divergence is temporary and harmless, because the outcome is fixed in C's log and everyone eventually converges on it.

### Case G: Both the coordinator and a participant fail

Suppose C crashes after sending COMMIT only to P1, and P1 then also crashes. P2 and P3 are prepared and in doubt. They can ask each other, but neither knows the decision. The only parties that know it are C (down) and P1 (down). P2 and P3 cannot abort, because P1 might have committed. They cannot commit, because C might have decided ABORT (for instance, if P1 had voted NO). **They must wait.** This is the clearest demonstration of why 2PC is called a blocking protocol.

### Recovery summary table

| Who recovers  | What the log says        | Action                                      |
|---------------|--------------------------|---------------------------------------------|
| Participant   | nothing / active         | Abort unilaterally                          |
| Participant   | PREPARED, no decision    | In doubt: hold locks, ask coordinator       |
| Participant   | COMMIT or ABORT          | Redo/undo accordingly, ACK                  |
| Coordinator   | BEGIN, no decision       | Abort, notify participants                  |
| Coordinator   | COMMIT/ABORT, no END     | Re-send decision until all ACK              |
| Coordinator   | END                      | Nothing; transaction forgotten              |

## 7. The blocking problem

2PC guarantees **safety**: no participant ever commits while another aborts. It does not guarantee **liveness** when the coordinator fails. Once a participant votes YES it enters an *uncertainty window* that lasts until it learns the decision. Inside that window:

- It cannot commit, because some other participant may have voted NO.
- It cannot abort, because the coordinator may have already logged COMMIT and told someone.
- It must keep its locks, so every other transaction that touches those rows also stalls.

If the coordinator is down for minutes or hours, the stall lasts that long. In real systems this shows up as "in-doubt transactions" in XA-capable databases. An operator may eventually resolve them by hand ("heuristic commit" or "heuristic abort"). That breaks atomicity if they guess wrong, which is why XA reports heuristic outcomes as a distinct error condition.

### The cooperative termination protocol

An in-doubt participant can reduce blocking by asking its peers (this requires each participant to know the full participant list, sent along with PREPARE):

- If any peer has COMMIT or ABORT logged, adopt that decision.
- If any peer has not yet voted (it has no PREPARED record), that peer can abort unilaterally and tell the others, so ABORT is safe.
- If **every reachable peer is also prepared and in doubt**, nobody can decide. Block.

Cooperative termination resolves many cases but not Case G. That limit is fundamental: the impossibility results of Skeen (1981) and the FLP result show that in an asynchronous system where crashes are indistinguishable from slowness, no atomic commit protocol can be non-blocking under every failure pattern.

### Why not just add timeouts?

A timeout lets a participant *suspect* the coordinator has failed, but it cannot tell whether the coordinator is dead or merely slow, or what it decided before going quiet. Acting on the timeout by committing or aborting risks contradicting a decision that already exists. In a prepared participant, timeouts can only trigger *queries*, never decisions.

## 8. Common optimisations

- **Presumed abort.** The coordinator writes nothing before the decision and does not force-write ABORT. If anyone asks about a transaction the coordinator has no record of, the answer is ABORT. Participants need not ACK an abort. This cuts log writes and messages on the failure path and is the default in most commercial systems (it is the model behind X/Open XA).
- **Presumed commit.** The coordinator force-writes a participant list up front. Then COMMIT needs no ACKs, and a missing record means COMMIT. This is cheaper when commits dominate.
- **Read-only participants.** A participant that only read data votes READ-ONLY, releases its locks immediately, and drops out of phase 2.
- **One-phase commit.** If only one resource manager is involved, skip PREPARE and just tell it to commit.
- **Last-agent / last-resource optimisation.** Prepare everyone except one participant, then hand that last participant the authority to decide. Its local commit becomes the global commit point. This lets one non-XA resource join a distributed transaction.
- **Replicated coordinator.** Run the coordinator's decision log on a consensus group (Paxos or Raft) so a single machine failure does not block. Google Spanner and CockroachDB do roughly this: each participant is itself a replicated group, and the decision is a replicated write. This attacks blocking where it starts, the single point of failure, rather than changing the protocol's message pattern.

## 9. Three-phase commit (3PC)

Three-phase commit, proposed by Skeen, tries to remove blocking by inserting a step between voting and committing so that **no participant can be uncertain while another has already committed**.

### Phases

1. **CanCommit (voting).** As in 2PC: `CAN-COMMIT?` and YES or NO.
2. **PreCommit.** If all said YES, the coordinator sends `PRE-COMMIT`. Participants log it and ACK. A participant in pre-committed state knows that *everyone* voted YES.
3. **DoCommit.** After all PRE-COMMIT ACKs, the coordinator sends `DO-COMMIT`. Participants commit.

### Why it helps

In 2PC, a participant that is prepared might be next to one that has already committed (it got COMMIT) or one that has aborted (it voted NO). Its state does not let it tell which. 3PC guarantees two properties:

- No participant can be in the *uncertain* (prepared) state while any other participant is *committed*, because commit requires everyone to have passed through PRE-COMMIT first.
- No participant can be *pre-committed* while any other has *aborted*.

So a recovery coordinator, elected from the surviving participants, can poll the survivors and decide:

- If any survivor is pre-committed or committed, decide COMMIT (everyone voted YES).
- If all survivors are merely prepared or uncertain, decide ABORT. No one can have committed yet.

### Worked example: coordinator dies during PreCommit

```
 t   C                          P1              P2              P3
 4   <-- YES x3
 5   --- PRE-COMMIT ----------> pre-committed
     ** CRASH **                                (prepared)      (prepared)
 ...
12   P1, P2, P3 time out; elect P2 as new coordinator
13   P2 polls: P1=pre-committed, P2=prepared, P3=prepared
14   P1 is pre-committed -> everyone voted YES -> COMMIT
15   P2 sends PRE-COMMIT to P3, then DO-COMMIT to all
16   all commit; no blocking
```

If instead C had crashed before sending any PRE-COMMIT, the poll would find everyone merely prepared, and P2 would safely decide ABORT.

### The catch

3PC is non-blocking only under the assumptions of a **synchronous network with fail-stop crashes and no partitions**: timeouts must reliably mean "crashed." With a network partition, the two sides can each elect a recovery coordinator. One side contains a pre-committed node and commits; the other side contains only prepared nodes and aborts. Atomicity is violated. Since real networks do partition, 3PC is rarely used in production. It also costs an extra round trip on every transaction. The practical lesson is that the real fix for blocking is a fault-tolerant coordinator built on consensus (Paxos Commit, by Gray and Lamport, is the formal version), not an extra phase.

## 10. Sagas: giving up atomic isolation

2PC and 3PC both hold locks across the network for the duration of the protocol. That is acceptable inside a single datacentre between databases that speak XA. It is often unacceptable between microservices owned by different teams, between companies, or over long-running business processes that last minutes or days. Many modern systems (message brokers, HTTP APIs, managed cloud stores) also do not support a prepare phase at all.

A **saga** (Garcia-Molina and Salem, 1987) replaces one distributed transaction with a sequence of **local** transactions T1, T2, ..., Tn, each of which commits immediately in its own service. For each step Ti there is a **compensating transaction** Ci that semantically undoes it. If step Tk fails, the saga runs Ck-1, ..., C1 in reverse order.

### Worked example: travel booking

Three services: Flights (P1), Hotels (P2), Payments (P3).

**Happy path:**

```
 t   Orchestrator             Flights           Hotels            Payments
 1   --- BookFlight -------->  commit seat
 2   <-- ok
 3   --- BookHotel ---------------------------> commit room
 4   <-- ok
 5   --- Charge ---------------------------------------------->  commit charge
 6   <-- ok
 7   saga COMPLETE
```

**Failure at step 3 (card declined):**

```
 t   Orchestrator             Flights           Hotels            Payments
 1   BookFlight ------------>  commit seat
 3   BookHotel ---------------------------->    commit room
 5   Charge ------------------------------------------------->   DECLINED
 6   compensate in reverse:
 7   --- CancelHotel -------------------------> commit release
 8   --- CancelFlight ------> commit release
 9   saga ABORTED (compensated)
```

Every arrow is a short local transaction. No locks are held across services, and no service ever waits on another's decision.

### Orchestration vs. choreography

- **Orchestration:** a central saga orchestrator (a workflow engine, a state machine with a durable log) issues each step and each compensation. It is easy to reason about and easy to observe, as in the example above.
- **Choreography:** each service listens for events ("FlightBooked") and emits the next ("HotelBooked" or "HotelFailed"). Nothing is centralised, but the overall flow is harder to see and debug.

### What sagas give up, and how to cope

- **Isolation.** Between T1 and the final step, other transactions can see the intermediate state: the seat is gone even though the trip later fails. Countermeasures include *semantic locks* (mark records "PENDING"), *commutative updates*, re-reading values before acting, and ordering steps so the riskiest or least compensable ones run last ("pivot transaction" design).
- **Perfect undo.** Some actions cannot be undone: an email sent, a package shipped. Compensation is semantic ("send an apology", "issue a refund"), not a rollback. Steps after the pivot should be *retriable* rather than compensable: designed to succeed eventually.
- **Exactly-once execution.** Messages may be redelivered, so every step and every compensation must be **idempotent**, keyed by a saga ID plus a step ID. The orchestrator persists its own progress so that it can resume after a crash. That log plays the same role as the 2PC coordinator's log, but it never blocks other work.

### Choosing between them

| Concern                     | 2PC                                  | Saga                                      |
|-----------------------------|--------------------------------------|-------------------------------------------|
| Atomicity                   | Strict, all-or-nothing               | Eventual, via compensation                |
| Isolation                   | Full (locks held to the end)         | None across steps; must be designed in    |
| Blocking on failure         | Yes (coordinator failure)            | No; steps commit locally                  |
| Latency                     | 2 round trips, locks held throughout | Each step short; total may be long        |
| Participant requirements    | Must support prepare (XA etc.)       | Must expose compensating actions          |
| Good fit                    | Few databases, one trust domain, short transactions | Microservices, cross-org, long-running workflows |

## 11. Putting it together

Two-phase commit solves atomic commitment with a small, precise idea: separate the *promise* (PREPARED, durably logged) from the *decision* (COMMIT or ABORT, durably logged in one place). That separation makes every failure recoverable without inconsistency, as the traces above show. Participants that have not promised can abort, participants that have promised ask, and the coordinator's log is the single source of truth.

The price is the uncertainty window. A prepared participant whose coordinator has vanished can only wait while holding its locks. Three-phase commit narrows the window under synchronous assumptions, but real networks partition and break it. Production systems that need strict atomicity therefore keep 2PC and make the coordinator itself highly available through consensus replication. Systems that cannot afford cross-service locks, or whose participants cannot prepare, give up isolation and use sagas: a chain of local commits with compensations, where the coordinator's log drives forward progress and never blocks anyone else.

A short checklist for any implementation:

1. Force-write PREPARED before voting YES, and the decision before sending it.
2. Never let a timeout turn a prepared participant's uncertainty into a decision.
3. Make every decision and every saga step idempotent.
4. Know your presumption (abort or commit) and keep the logging consistent with it.
5. Treat a single-node coordinator as the availability bottleneck it is: replicate it or design around it.
