# Lecture 6 — Transactions: What ACID Guarantees, and What Weak Isolation Misses

**Course:** Self-Directed CS — System Design Lab
**Week 4 · Lecture 6**
**Builds on:** Lecture 1 (invariants, multi-leader/leaderless conflicts), Lecture 2 (committed vs. uncommitted), Lecture 5 (hot keys, queues)
**Status:** ✅ Closed — comprehension confirmed

---

## 1. Key Concepts

1. **A transaction is a group of reads and writes the database treats as one unit.** Between `BEGIN` and `COMMIT` the writes are a draft. `COMMIT` makes all of them permanent together; `ROLLBACK` or a crash before commit discards all of them.
2. **Autocommit is the default in most databases:** each statement commits on its own. A multi-statement operation run this way can be left half-done by a crash.
3. **A transaction answers two separate questions:** what happens to partial writes if something fails (atomicity), and what concurrent operations can see of the draft (isolation).
4. **Atomicity is about failure, not concurrency.** All-or-nothing on abort; a better name is *abortability*. Its practical payoff: a failed transaction can be retried safely.
5. **Consistency is the application's invariant**, not a database mechanism. The database enforces a few constraints (uniqueness, foreign keys), but A, I, and D are the tools that protect invariants the application defines.
6. **Isolation is about concurrency.** The ideal is serializability (results as if transactions ran one at a time). Most databases default to weaker levels for performance.
7. **Durability:** after commit, data survives crashes (write-ahead log synced to disk; in replicated systems, replication to followers or a majority).
8. **Read committed** prevents two anomalies:
   - **Dirty read:** seeing another transaction's uncommitted writes. Prevented by serving the last committed value until the writer commits.
   - **Dirty write:** overwriting another transaction's uncommitted write. Prevented by row-level write locks held until commit/abort.
9. **Read committed does not prevent lost updates.** In a read-modify-write cycle, two transactions read the same committed value, compute from it, and the second write silently overwrites the first, even though no rule of read committed was broken.
10. **Fixes for lost updates** all make the read-modify-write cycle indivisible or detect when it wasn't:
    - **Serialize access** (a queue or single writer per key): the next read waits until the previous write completes.
    - **Atomic operations:** `UPDATE counters SET value = value + 1` — the database does the whole cycle as one step.
    - **Explicit locking:** `SELECT ... FOR UPDATE` locks the row at read time; a second transaction's read waits.
    - **Compare-and-set:** write only if the value is still what you read (`... WHERE value = 42`); if no row was updated, retry.
    - **Automatic detection:** some databases detect the lost update and abort one transaction (previewed for Lecture 7).
11. **Pessimistic vs. optimistic:** locking and queues *prevent* conflicts by making others wait. Compare-and-set and automatic detection *allow* the race and catch it at write time, then retry.
12. **These fixes assume a single copy to coordinate on.** With multi-leader or leaderless replication (Lecture 1), locks and compare-and-set don't work across replicas, which is why those systems lean on commutative operations or keeping conflicting versions as siblings.

---

## 2. Definitions

| Term | Definition |
|---|---|
| **Transaction** | A group of operations executed as one unit: all writes commit together or none take effect. |
| **Commit / Rollback** | Make all of a transaction's writes permanent / discard all of them. |
| **Autocommit** | Mode where each statement is its own transaction, committed immediately. |
| **Atomicity** | All-or-nothing outcome under failure (abortability). |
| **Consistency (ACID)** | The application's invariants hold; mostly the application's responsibility. |
| **Isolation** | Concurrent transactions don't observe or interfere with each other's in-progress state. |
| **Durability** | Committed data survives crashes. |
| **Serializability** | Strongest isolation: outcome equals some one-at-a-time execution order. |
| **Read committed** | Isolation level preventing dirty reads and dirty writes. |
| **Dirty read** | Reading another transaction's uncommitted write. |
| **Dirty write** | Overwriting another transaction's uncommitted write. |
| **Lost update** | Two concurrent read-modify-write cycles where one write overwrites the other's effect. |
| **Read-modify-write cycle** | Read a value, compute a new value from it in the application, write it back. |
| **Compare-and-set (CAS)** | Write only if the current value equals the value previously read. |

---

## 3. Mental Models

- **Failure vs. concurrency.** Ask which threat you're facing. Atomicity and durability answer failure; isolation answers concurrency. A transaction can be perfectly atomic and still expose a half-finished state to concurrent readers.
- **C is the goal, A/I/D are the tools.** Same move as Lecture 1: find the invariant first, then pick the mechanism that protects it.
- **Rules can all be satisfied while the outcome is wrong.** The lost update breaks no read-committed rule. Correctness depends on the whole read-modify-write cycle, not on each read and write individually.
- **Make the cycle indivisible, or detect when it wasn't.** Every lost-update fix is one of these two.
- **Prevent (pessimistic) vs. detect-and-retry (optimistic).** Prevent when conflicts are frequent; detect-and-retry when they're rare and waiting is expensive.

---

## 4. Diagrams (ASCII)

### Autocommit vs. transaction under a crash
```
AUTOCOMMIT                          TRANSACTION
UPDATE A -= 100  (committed)        BEGIN
   *** crash ***                    UPDATE A -= 100  (draft)
UPDATE B += 100  (never runs)          *** crash ***
=> A lost 100, B never got it       => no COMMIT record -> draft discarded
                                    => A and B unchanged; safe to retry
```

### Dirty read vs. what read committed shows
```
T1: BEGIN; A -= 100 (draft) ............ COMMIT
Auditor:          read A
   dirty read:     sees reduced A (uncommitted)
   read committed: sees old A (last committed value)
```

### Lost update (allowed by read committed)
```
T1: read 42 -------- write 43, commit
T2:      read 42 ---------------------- write 43, commit
Final: 43  (should be 44) — no dirty read, no dirty write
```

### Fixes, by where they act in the cycle
```
read ──► modify ──► write
 │                    │
 ├─ FOR UPDATE lock: hold the row from read until commit
 ├─ queue / single writer: next cycle starts after previous write
 ├─ atomic op: database performs read+modify+write as one step
 └─ CAS: at write time, check value unchanged; else retry
```

---

## 5. Engineering Takeaways

- **Wrap multi-step operations in a transaction.** Autocommit plus multiple statements is a half-done-write bug waiting for a crash.
- **Know your database's default isolation level.** Many default to read committed, which allows lost updates.
- **Prefer atomic operations when the change can be expressed in the database** (`x = x + 1`). They're the cheapest correct fix.
- **Use `SELECT ... FOR UPDATE` when the new value needs application logic** that can't be pushed into one SQL statement.
- **Use compare-and-set when conflicts are rare** and you'd rather retry than wait.
- **A queue per key works but limits throughput** to one cycle at a time and can itself become a hot spot (Lecture 5).
- **Retries need care:** a transaction that actually committed but whose acknowledgment was lost may be applied twice on retry. Idempotency (e.g., a request ID) prevents this.

---

## 6. Cost Summary

| Fix | Style | Cost | Best when |
|---|---|---|---|
| Atomic operation | Pessimistic (internal lock) | Very low | Change expressible in DB |
| `SELECT ... FOR UPDATE` | Pessimistic | Blocking; deadlock risk | Complex application logic |
| Queue / single writer | Pessimistic | Throughput capped; possible hot spot | Very contended single key |
| Compare-and-set | Optimistic | Retries under contention | Conflicts rare |
| Automatic detection | Optimistic | Aborts + retries | Database supports it |

---

## 7. Interview Notes

**Likely questions**
- "What does each letter of ACID mean?" → separate failure (A, D) from concurrency (I); note C is the application's invariant.
- "What's a dirty read / dirty write? How does read committed prevent them?"
- "Two users increment a counter and one increment disappears. Why, and how do you fix it?" → lost update; name at least two fixes and whether each is pessimistic or optimistic.
- "Does read committed prevent lost updates?" → no; walk through the interleaving.

**Common candidate mistakes**
- Saying atomicity handles concurrency.
- Treating "consistency" as a database guarantee rather than an application invariant.
- Assuming read committed makes read-modify-write safe.
- Proposing locks or compare-and-set in a multi-leader/leaderless system without noting they don't span replicas.

**High-value framing line:** *"I'd separate the failure question from the concurrency question: atomicity handles partial failure, isolation handles interleaving, and the invariant tells me how strong the isolation needs to be."*

---

## 8. Revision Sheet (≤ 5 min)

- Transaction = draft between BEGIN and COMMIT; all writes commit together or are discarded.
- A = all-or-nothing on failure. I = concurrency. D = survives crash after commit. C = app invariant (goal).
- Read committed prevents dirty reads (serve last committed value) and dirty writes (row locks until commit).
- Lost update = two read-modify-write cycles interleave; read committed allows it.
- Fixes: queue/single writer, atomic op, `FOR UPDATE`, compare-and-set, automatic detection.
- Pessimistic (prevent by waiting) vs. optimistic (detect and retry).
- Locks/CAS need one copy to coordinate on; replicated multi-leader systems use commutative ops or siblings instead.

---

## 9. Knowledge Connections

**Builds on:** Lecture 1 (find the invariant → C; multi-leader conflicts), Lecture 2 (uncommitted entries discarded → atomicity; majority commit → durability), Lecture 5 (queue for a hot key, hot-spot risk of a single writer).

**Within this lecture:** what a transaction is → which letter handles which threat → read committed (dirty read/write) → its gap (lost update) → fixes.

**Forward links (Lecture 7):**
- **Snapshot isolation** and read skew (a long-running read seeing different committed values mid-transaction).
- **Write skew and phantoms** — anomalies even snapshot isolation allows; lost updates generalized across multiple rows.
- **Serializability implementations:** actual serial execution (your queue idea, scaled up), two-phase locking, serializable snapshot isolation.
- **Distributed transactions / 2PC** — atomic commit across partitions, connecting back to consensus.
- **Backlog:** DDIA Ch. 10 becomes readable once Lecture 7 is done.

**Real systems:** PostgreSQL (read committed default; repeatable read detects lost updates and aborts), MySQL InnoDB (repeatable read default; does *not* detect lost updates automatically), Redis (single-threaded command execution — serial access by design).

---

## 10. Flashcards

**Q:** What is a transaction?
**A:** A group of operations treated as one unit: between BEGIN and COMMIT the writes are a draft; COMMIT makes all of them permanent, ROLLBACK or a crash discards all of them.

**Q:** Does atomicity prevent other transactions from seeing your half-finished work?
**A:** No. Atomicity only governs the final outcome under failure. Visibility of in-progress state is isolation.

**Q:** Why is "consistency" the odd letter in ACID?
**A:** It's the application's invariant, not something the database can fully guarantee; A, I, and D are the database's tools for protecting it.

**Q:** What two anomalies does read committed prevent, and how?
**A:** Dirty reads (serve last committed value until commit) and dirty writes (row-level write locks held until commit/abort).

**Q:** Why does read committed still allow a lost update?
**A:** Both transactions read a committed value and write after the other commits, so no rule is broken; the problem is the stale read inside the read-modify-write cycle.

**Q:** Name four ways to prevent lost updates.
**A:** Serialize access (queue/single writer), atomic operations, `SELECT ... FOR UPDATE`, compare-and-set (plus automatic detection in some databases).

**Q:** Pessimistic vs. optimistic concurrency control?
**A:** Pessimistic prevents conflicts by making others wait (locks, queues). Optimistic lets operations proceed and detects conflicts at write time, then retries (CAS, automatic detection).

**Q:** Why don't locks and compare-and-set solve lost updates in multi-leader replication?
**A:** They assume a single up-to-date copy to coordinate on; with multiple leaders accepting writes independently, there's no single place to lock or compare.

---

# Homework

### Core
1. **(Conceptual)** In your own words: why can a transaction be fully atomic and still let another query see a wrong intermediate state? Use the transfer example.
2. **(Trace)** Two transactions run under read committed. T1 transfers 100 from A to B. T2 runs `SELECT balance FROM accounts WHERE id IN ('A','B')` and sums the result. Show one interleaving where T2's sum is correct, and explain why read committed guarantees T2 never sees A's debit before T1 commits.
3. **(Fix selection)** For each case, pick the best lost-update fix and say whether it's pessimistic or optimistic: (a) a page-view counter; (b) a wiki page edited rarely, by two people at once maybe once a month; (c) withdrawing from an account, where the application must check `balance >= amount` before writing.
4. **(Reading, textbook)** *Designing Data-Intensive Applications*, 2nd ed., **Chapter 8 — Transactions**: the sections on the meaning of ACID, weak isolation / read committed, and preventing lost updates. Stop before snapshot isolation's write skew material if you want to keep it fresh for Lecture 7 (section names may differ slightly in your edition).
5. **(Reading, paper)** *A Critique of ANSI SQL Isolation Levels* (Berenson, Bernstein, Gray, Melton, O'Neil, O'Neil — SIGMOD 1995). PDF: https://www.cs.umb.edu/~poneil/iso.pdf (also on arXiv: https://arxiv.org/abs/cs/0701157). Read Sections 1–3 for now: how the standard defined isolation levels by the anomalies they prevent, and why that definition left gaps (including dirty writes, and the lost update you saw today). Section 4 (snapshot isolation) is Lecture 7 material — skim or save it.

### Advanced
6. **(Design)** Your queue-based fix serializes all increments on one key. For a counter receiving 50,000 increments per second, what breaks? Propose a design that combines a fix from this lecture with an idea from Lecture 1 or Lecture 5 to keep correctness without a single bottleneck.
7. **(Retry safety)** A client sends a transfer, the database commits, but the network drops the "committed" response. The client retries. What goes wrong, and how would you make the retry safe?

### Challenge
8. **(Synthesis)** Compare-and-set works on a single leader. In a leaderless system with quorums (`w + r > n`, Lecture 1), describe a scenario where a CAS-style "write only if value is still 42" check gives a wrong answer. Which Lecture 1 mechanism do leaderless systems use instead?

### Reflection
9. Your fix (a queue) was a pessimistic, serialize-everything approach. When would you prefer an optimistic approach instead? What property of the workload decides it?

**Backlog:** Bigtable paper + Lecture 5 Q7.

**Next session opens with:** Lecture 7 — Isolation levels and serializability (snapshot isolation, write skew, phantoms, and how serializability is actually implemented) — starting only after your explicit go-ahead.
