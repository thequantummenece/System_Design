# Lecture 7 — Isolation Levels and Serializability

**Course:** Self-Directed CS — System Design Lab
**Week 5 · Lecture 7**
**Builds on:** Lecture 6 (transactions, read committed, lost updates, pessimistic vs. optimistic), Lecture 1 (find the invariant), Lecture 4 (partitioning)
**Status:** ✅ Closed — comprehension confirmed (re-teaches needed on 2PL lock behavior and the stale-premise idea; both resolved)

---

## 1. Key Concepts

1. **Read skew:** a single transaction reads different parts of the database at different points in time, so it sees a combination of values that never existed together. Every individual read is committed, so read committed allows it. Example: read A before a transfer commits and B after, and the total is off by the transfer amount.
2. **Read skew is tolerable for a single interactive read** (refresh and it fixes itself) **but damaging for long reads:** backups (the inconsistency becomes permanent on restore), analytic queries, and integrity checks.
3. **Snapshot isolation (SI):** each transaction reads from a consistent snapshot of the database as of the moment it started. Writes committed after it started are invisible to it, for its whole lifetime.
4. **Why read committed's mechanism isn't enough:** read committed keeps the old value only *until the writer commits*. SI must keep old versions *after* commit, for as long as any older transaction might read them, and must pin each transaction to its start time.
5. **MVCC (multi-version concurrency control)** implements SI:
   - Every transaction gets an increasing transaction ID at start.
   - Each row version records `created_by` and `deleted_by`. An `UPDATE` marks the old version deleted and inserts a new version; it never overwrites.
   - At start, a transaction records which transactions are in progress. When reading, it **ignores** writes by: transactions in progress at its start (even if they commit later), transactions with a higher ID, and aborted transactions.
   - A version is visible if its creator is not ignored and its deleter is ignored or absent.
6. **MVCC consequences:** readers never block writers and writers never block readers. Read committed and SI share the machinery: read committed takes a new snapshot per *statement*, SI one per *transaction*. Old versions are garbage-collected (PostgreSQL `VACUUM`); a long-running transaction keeps everything since its start alive.
7. **SI and lost updates:** SI alone doesn't prevent lost updates. Some databases detect same-row write conflicts and abort one transaction (PostgreSQL's repeatable read does); others don't (MySQL InnoDB).
8. **Write skew:** two transactions each read to check a condition, decide it's safe, and write **different** rows. Each write invalidates the other's check, so together they break an invariant. No dirty read, no dirty write, no lost update, no read skew — every rule holds. It's a lost update generalized across rows linked by a multi-row invariant.
9. **Write-skew pattern:** (1) read to check a condition, (2) decide, (3) write in a way that changes the result of the check. Examples: on-call doctors, meeting-room booking, unique usernames, double spending, two game players moving to the same square.
10. **`SELECT ... FOR UPDATE` fixes write skew only when the rows the decision depends on exist** (the doctors case). It fails when the check is for the **absence** of rows: zero rows returned means zero rows locked.
11. **Phantom:** a write in one transaction that changes the result of a search query in another. Fixes short of serializability: database constraints (unique constraints; PostgreSQL exclusion constraints for overlapping ranges) when they fit, or **materializing conflicts** (precreate rows such as room/time-slot rows so there's something to lock) as a last resort.
12. **Serializability:** the result equals some one-at-a-time execution order. It prevents every anomaly above. Three implementations:
    - **Actual serial execution:** one thread, one transaction at a time. Practical because data fits in RAM and OLTP transactions are short. Requires stored procedures (no interactive transactions: a network round trip would idle the single thread and stall everyone). Throughput is one core per partition; cross-partition transactions need coordination and are much slower. VoltDB, Redis, Datomic.
    - **Two-phase locking (2PL):** shared locks for reads, exclusive locks for writes, all held until commit/abort. Readers block writers and writers block readers. Phantoms are handled with predicate locks or, in practice, index-range (next-key) locks, falling back to table locks. Costs: lower throughput, unpredictable latency, deadlocks (detected; one transaction aborted and retried). MySQL InnoDB, SQL Server at serializable.
    - **Serializable snapshot isolation (SSI):** optimistic. Transactions run under SI with no blocking; the database tracks when a transaction's premise may have gone stale (it ignored an uncommitted write that later committed, or someone wrote into a range it read). At commit, if a committed transaction changed what it read, it aborts and retries. PostgreSQL serializable, CockroachDB, FoundationDB.
13. **2PL never takes away another transaction's lock.** In the doctors example both transactions hold shared locks on both rows; each upgrade to exclusive waits on the other → deadlock → one is aborted → the other commits → the aborted one retries, re-reads, and refuses.
14. **Stale premise:** a read that was valid when made but invalidated by a later commit. A transaction that made decisions from it can't be patched midway (the database can't know which later steps depended on it), so it must be aborted and rerun from the start. This is distinct from a dirty read, which reads uncommitted data.
15. **SSI aborts at commit time, not at the conflicting write,** because the writer might still abort (then the premise was fine) and the reader might be read-only (then no write skew results). Aborting early would waste work for nothing.

---

## 2. Definitions

| Term | Definition |
|---|---|
| **Read skew** | One transaction observes different parts of the database at different points in time. |
| **Snapshot isolation** | Each transaction reads a consistent snapshot as of its start. |
| **MVCC** | Keeping multiple committed versions of each row, tagged with creating/deleting transaction IDs, to serve snapshots. |
| **Visibility rules** | The rules deciding which row version a transaction sees (ignore in-progress-at-start, later, and aborted transactions). |
| **Vacuum / garbage collection** | Removing row versions no running transaction can still see. |
| **Write skew** | Concurrent transactions each check a condition and write different rows, jointly violating an invariant. |
| **Phantom** | A write that changes the result of another transaction's search query. |
| **Materializing conflicts** | Creating placeholder rows so a phantom becomes a lockable row conflict. |
| **Serializability** | Outcome equals some serial execution of the transactions. |
| **Stored procedure** | A transaction's full logic submitted to and executed inside the database in one go. |
| **Two-phase locking (2PL)** | Acquire shared/exclusive locks as you go; release all only at commit/abort. Not related to two-phase commit. |
| **Predicate lock** | A lock on all rows matching a condition, including rows not yet inserted. |
| **Index-range (next-key) lock** | A coarser lock on an index range covering a predicate; the practical form of predicate locking. |
| **Deadlock** | Two transactions each waiting for a lock the other holds. |
| **SSI** | Optimistic serializability on top of SI: detect stale premises and abort at commit. |
| **Stale premise** | A committed value read correctly but invalidated by a later commit before the reader commits. |

---

## 3. Mental Models

- **Which point in time is this transaction looking at?** Read committed: "now," re-evaluated per statement. SI: "when I started," fixed. Read skew is the gap between the two.
- **Versions plus a visibility rule, not a copy of the database.** A snapshot is just a rule for choosing among row versions.
- **Single-row invariant → single-row tools. Multi-row invariant → multi-row protection.** Lecture 1's "find the invariant" decides which isolation level you actually need.
- **Absence can't be locked.** Any check of the form "no row like this exists" is a phantom risk.
- **Pessimistic vs. optimistic, again.** 2PL prevents by blocking; SSI detects and retries. Same choice as Lecture 6 (locks vs. compare-and-set), now at full serializability.
- **Serial execution trades concurrency for simplicity** and is correct only under specific conditions: short transactions, data in memory, a workload that partitions cleanly.

---

## 4. Diagrams (ASCII)

### Read skew vs. snapshot isolation
```
                 READ COMMITTED              SNAPSHOT ISOLATION
T2 read A        500 (T1 not committed)      500 (snapshot at T2 start)
T1 commits       A=400, B=600                A=400, B=600
T2 read B        600 (latest committed)      500 (still the snapshot)
T2 sum           1,100  wrong                1,000  correct
```

### MVCC versions for one row
```
B = 500   created_by 3    deleted_by 12
B = 600   created_by 12
T13 (started while T12 in progress) -> ignores T12 -> sees B = 500
T14 (started after T12 committed)   -> sees B = 600
```

### Write skew (doctors)
```
Alice: count on-call -> 2 ok        Bob: count on-call -> 2 ok
Alice: set Alice off                 Bob: set Bob off      (different rows)
COMMIT                               COMMIT
=> 0 on call; invariant broken, no single-row rule violated
```

### Why FOR UPDATE fails for phantoms
```
SELECT ... overlapping bookings FOR UPDATE  -> 0 rows -> 0 locks
both transactions pass the check -> both INSERT -> double booking
```

### Doctors under 2PL
```
Alice: S-lock(Alice row), S-lock(Bob row)
Bob:   S-lock(Alice row), S-lock(Bob row)        (shared locks coexist)
Alice: X-lock(Alice row)? waits for Bob's S-lock
Bob:   X-lock(Bob row)?   waits for Alice's S-lock
=> deadlock -> abort Bob -> Alice commits -> Bob retries -> count 1 -> refuse
```

### Doctors under SSI
```
Alice, Bob: read count=2 from snapshot, update own rows   (no blocking)
Alice commits
Bob commits? -> DB: a range Bob read was written by a now-committed txn -> ABORT
Bob retries -> count 1 -> refuse
```

---

## 5. Engineering Takeaways

- **Run backups and long analytical scans under snapshot isolation**, never read committed.
- **Keep transactions short in MVCC databases;** a forgotten open transaction blocks vacuum and bloats storage.
- **Know what "repeatable read" means in your database.** In PostgreSQL it's snapshot isolation with lost-update detection; in MySQL it's snapshot isolation without it. Names don't match behavior across vendors.
- **For check-then-write logic, first look for a constraint** (unique, exclusion). It's the cheapest correct fix.
- **Use `SELECT ... FOR UPDATE` when the decision depends on rows that exist;** don't rely on it when checking for absence.
- **Choosing a serializable implementation:** serial execution for short in-memory partitionable workloads; 2PL when conflicts are frequent and blocking is acceptable; SSI when conflicts are rare and you want readers never to block.
- **Serializable isolation means the application must retry aborted transactions**, safely (idempotency, Lecture 6 Q7).

---

## 6. Cost Summary

| Approach | Blocking | Main cost | Phantom handling | Best when |
|---|---|---|---|---|
| Snapshot isolation | None (readers/writers) | Version storage, vacuum; allows write skew | None | Read-heavy, write skew impossible or guarded |
| Serial execution | Everything is serial | One core per partition; no interactive txns; slow cross-partition | Not needed | Short txns, in-memory, partitionable |
| 2PL | Readers ↔ writers block | Throughput, unpredictable latency, deadlocks | Index-range / predicate locks | High contention, blocking acceptable |
| SSI | None | Aborts and retries under contention | Read-range tracking | Low–moderate contention |

---

## 7. Interview Notes

**Likely questions**
- "What's the difference between read committed and snapshot isolation?" → statement vs. transaction snapshot; read skew; MVCC.
- "How does MVCC work?" → transaction IDs, created_by/deleted_by, visibility rules, vacuum.
- "What is write skew? Give an example and a fix." → doctors or bookings; FOR UPDATE, constraints, serializable.
- "Why can't you just lock the rows for a uniqueness/booking check?" → phantoms; zero rows to lock.
- "Compare 2PL and SSI." → pessimistic blocking + deadlocks vs. optimistic aborts at commit.
- "Is two-phase locking the same as two-phase commit?" → no.

**Common candidate mistakes**
- Thinking snapshot isolation is serializable (it allows write skew).
- Believing `FOR UPDATE` prevents double bookings.
- Saying 2PL "releases" or "steals" locks before commit.
- Calling a stale read a dirty read.
- Proposing single-threaded execution without noting the stored-procedure and partitioning constraints.

**High-value framing line:** *"I'd look at whether the invariant spans multiple rows. If it does, snapshot isolation isn't enough, and I'd choose between a constraint, explicit locking, or serializable isolation based on contention."*

---

## 8. Revision Sheet (≤ 5 min)

- Read skew = reads at different moments; fix = snapshot isolation (snapshot at transaction start).
- MVCC: many versions per row; created_by/deleted_by; ignore in-progress-at-start, later, aborted txns.
- Read committed = snapshot per statement; SI = snapshot per transaction; readers never block writers.
- Write skew = check, decide, write *different* rows; invariant spans rows; SI allows it.
- FOR UPDATE works when rows exist; fails for absence checks (phantoms).
- Phantom fixes: constraints, materializing conflicts, serializability.
- Serializable via: serial execution (stored procs, partitioned, in-memory), 2PL (locks held to commit; deadlocks), SSI (detect stale premise, abort at commit).
- 2PL ≠ 2PC. Stale premise ≠ dirty read.

---

## 9. Knowledge Connections

**Builds on:** Lecture 1 (invariant-first; multi-row invariant needs multi-row protection), Lecture 2 (an aborted write is discarded), Lecture 4 (partitioning to scale serial execution), Lecture 6 (lost update → write skew; locks vs. compare-and-set → 2PL vs. SSI).

**Your own homework connections:** Lecture 6 Q2 found read skew before it was taught; your queue answers in Lectures 5–6 are actual serial execution, valid under the conditions above.

**Forward links:**
- **Lecture 8 — Distributed transactions:** atomic commit across partitions and nodes (two-phase commit), why it's hard, and how it connects to consensus. Serial execution's slow cross-partition transactions are the motivation.
- **DDIA Ch 10 (Consistency and Consensus)** becomes readable after Lecture 8.
- **Storage-engines block (Database Internals Part I):** how version chains and index-range locks are stored in B-trees and LSM-trees.

**Real systems:** PostgreSQL (MVCC; SSI at serializable), MySQL InnoDB (MVCC; 2PL with next-key locks at serializable), Oracle (SI called "serializable"), SQL Server, VoltDB, Redis, CockroachDB, FoundationDB.

---

## 10. Flashcards

**Q:** What is read skew?
**A:** One transaction reads different parts of the database at different points in time, seeing a combination of values that never existed together.

**Q:** Why doesn't read committed's "keep the old value" mechanism prevent read skew?
**A:** It keeps the old value only until the writer commits; afterward every read gets the new value. SI needs old versions kept after commit and each transaction pinned to its start time.

**Q:** Under MVCC, which writes does a transaction ignore?
**A:** Writes by transactions in progress when it started (even if they commit later), by transactions with higher IDs, and by aborted transactions.

**Q:** How do read committed and snapshot isolation differ under MVCC?
**A:** Read committed takes a new snapshot per statement; snapshot isolation uses one snapshot for the whole transaction.

**Q:** What is write skew?
**A:** Two transactions each check a condition, decide it's safe, and write different rows; together they violate an invariant that spans those rows.

**Q:** Why doesn't `SELECT ... FOR UPDATE` prevent double-booking a room?
**A:** The check looks for absence; the query returns zero rows, so nothing is locked. This is a phantom.

**Q:** Why can't single-threaded serial execution run interactive transactions?
**A:** The single thread would sit idle during each network round trip, stalling every other transaction behind it.

**Q:** What happens in the doctors example under 2PL?
**A:** Both hold shared locks on both rows; each upgrade to exclusive waits on the other → deadlock → one aborts and retries, sees one doctor on call, and refuses.

**Q:** Why must a transaction with a stale premise be restarted rather than continued?
**A:** Its decisions were computed from a read that is no longer true, and the database can't know which later steps depended on it.

**Q:** Why does SSI abort at commit rather than at the moment of the conflicting write?
**A:** The writer might still abort, and the reader might be read-only; in either case no anomaly results, so aborting early would waste work.

**Q:** Two-phase locking vs. two-phase commit?
**A:** 2PL is a single-database concurrency control method for serializability. 2PC is a protocol for atomic commit across multiple nodes. Unrelated.

---

# Homework

### Core
1. **(Conceptual)** In your own words: why is snapshot isolation enough to prevent read skew but not write skew? Name the specific thing a snapshot cannot tell a transaction.
2. **(MVCC trace)** Row X = 10 (created by T5). T20 starts. T21 starts, updates X to 20, commits. T22 starts. T20 and T22 each read X. What does each see, by which rule? When can the version X = 10 be garbage-collected?
3. **(Classify)** For each, name the anomaly (lost update, read skew, write skew, phantom, or none) under snapshot isolation, and the cheapest correct fix:
   (a) Two users register the username "aadesh" at the same time.
   (b) A nightly report sums all account balances while transfers run.
   (c) A purchase checks `SUM(spend) + amount <= budget`, then inserts a spend row; two purchases run concurrently.
   (d) Two concurrent `UPDATE counter SET value = value + 1`.

### Hands-on (recommended)
4. **(Reproduce write skew in PostgreSQL)** Install PostgreSQL (or use Docker). Create a `doctors` table with two on-call doctors. Open two `psql` sessions and run the doctors transactions interleaved step by step:
   - under `REPEATABLE READ` (snapshot isolation): confirm both commit and nobody is on call;
   - under `SERIALIZABLE` (SSI): confirm one gets `could not serialize access`.
   Write down the exact error message and which session got it.

### Reading
5. **(Textbook)** *DDIA* 2nd ed., **Chapter 8 — Transactions**: the sections on snapshot isolation / repeatable read, write skew and phantoms, and serializability (serial execution, two-phase locking, serializable snapshot isolation). Section names may differ slightly in your edition.
6. **(Internals)** *Database Internals*, **Ch 5 — Transaction Processing and Recovery**: the concurrency-control sections (concurrency control overview, multiversion concurrency control, pessimistic and optimistic schemes, lock-based concurrency control including deadlocks). This is the "how it's built" view of today's lecture.
7. **(Paper)** *Serializable Snapshot Isolation in PostgreSQL* — Dan R. K. Ports and Kevin Grittner, VLDB 2012. PDF: https://arxiv.org/pdf/1208.4179 (also https://drkp.net/papers/ssi-vldb12.pdf). Read Sections 1–3 and 5: why PostgreSQL chose SSI, the write-skew examples, and how SSI tracks read/write dependencies. It's written by the engineers who built it, so it's more practical than the original 2008 SSI paper. Also finish **Berenson et al. Section 4** (snapshot isolation) from last week.

### Advanced
8. **(Design)** A ticketing system sells seats for a concert with 50,000 seats and huge bursts at sale start. Compare three designs: (a) PostgreSQL serializable (SSI); (b) 2PL; (c) single-threaded serial execution partitioned by section. For each, predict what happens under the burst (aborts, blocking, throughput) and pick one.
9. **(Phantoms)** For the meeting-room problem, design the materialized-conflicts fix concretely (table schema and the transaction). Then name two ways it breaks as requirements change.

### Challenge
10. **(Synthesis)** Serial execution partitions data and gives each partition one thread. A transaction must move money between accounts on two different partitions. Sketch what the two partitions must do so the transfer is atomic (both or neither) even if one partition crashes midway. Don't look anything up; this is a preview of Lecture 8. What problem do you run into if the coordinator crashes?

### Reflection
11. In this lecture you needed a correction on 2PL ("the other lock is removed") and on "dirty read" vs. "stale premise." What did each misconception assume that turned out false?

### Mini-project (portfolio)
**Build a tiny MVCC key-value store** (Python or C++): transaction IDs, versioned values with `created_by`/`deleted_by`, snapshot reads following the visibility rules from this lecture, a `vacuum()` that deletes unreachable versions, and a test that reproduces read skew under a "read latest committed" mode and shows it disappearing under snapshot mode. Stretch: add write-skew detection by tracking read keys and aborting at commit (a simplified SSI). Small enough to finish in a few evenings, and a strong interview talking point.

---

## Backlog
- Bigtable paper + Lecture 5 Q7
- Lecture 6: Q4 (DDIA reading), Q5 (Berenson Sections 1–3), Q9 (reflection)
- *Database Internals* Ch 5 (WAL/recovery sections) and Ch 9 (Failure Detection)

---

## End-of-Week Summary (Week 5)

- **Covered:** read skew, snapshot isolation, MVCC internals; write skew and phantoms; three implementations of serializability.
- **Strong:** MVCC visibility trace (clean first try); finding the phantom in the booking example; the two-case analysis of interactive transactions under serial execution; SSI commit-time reasoning.
- **Needed reinforcement:** 2PL lock semantics (locks are never taken away; the conflict surfaces as a deadlock); terminology (dirty read vs. stale premise; availability vs. throughput).
- **Recurring pattern:** reaching for serialization first. This lecture showed when it's genuinely the right design and what conditions it needs.
- **Revise:** the doctors example under all four regimes (SI, FOR UPDATE, 2PL, SSI). If you can trace all four, the lecture is solid.
- **Next week:** Lecture 8 — distributed transactions (two-phase commit), which completes the transactions block. Then the two recap lectures and the month-end assessment you planned.
