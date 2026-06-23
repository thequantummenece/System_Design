# Lecture 1 — Replication & Consistency Foundations

**Course:** Self-Directed CS — System Design Lab
**Week 1 · Lecture 1**
**Primary text:** *Designing Data-Intensive Applications*, 2nd ed. (Kleppmann & Riccomini) — Replication chapter
**Prerequisites:** Basic client–server model
**Status:** ✅ Closed — comprehension confirmed

---

## 1. Key Concepts

1. **Why distribution exists.** You leave a single machine for exactly two reasons: **scale** (you outgrow one machine; vertical scaling has a ceiling and superlinear cost) and **reliability** (one machine is a single point of failure). The moment a second machine appears, you inherit the four tensions below.

2. **The four tensions of multi-machine systems.**
   - The network is **unreliable** (loss, delay, duplication, reordering).
   - There is **no shared clock** (machines disagree on time; timestamps across nodes are untrustworthy).
   - **Partial failure** (some nodes fail while others run; survivors can't tell *dead* from *slow* from *unreachable*).
   - **Consistency** (multiple copies of the same data may disagree — for how long, and who is "the truth"?).

3. **Replication vs. Partitioning** — the two pillars of distributed data.
   - **Replication:** *same* data on multiple nodes. Buys reliability + read scaling. Core problem: keeping copies in agreement.
   - **Partitioning (sharding):** *different* data on different nodes. Buys capacity beyond one machine.

4. **Synchronous vs. asynchronous replication.** Sync waits for replicas to confirm before ack'ing the write (safe, but slow and fragile to replica downtime). Async acks immediately and propagates in the background (fast, available, but risks losing an acknowledged write if the leader dies mid-gap). **In practice: semi-synchronous** — one synchronous follower, the rest async.

5. **Replication lag and its three read anomalies** (async-induced), each with a standard fix.

6. **Leaderless replication & quorums.** `w + r > n` forces read and write sets to overlap, guaranteeing a read sees at least one node with the latest write.

7. **Conflict resolution.** Wall-clock **LWW** (lossy) vs. **logical version numbers / version vectors** (causality-aware, lossless-but-complex).

8. **CAP framing.** Under a network **partition**, choose **Consistency** or **Availability** — not both. Driven by whether a correctness **invariant** is at stake.

9. **The master heuristic: find the invariant.** A rule that must *never* be violated forces coordination (CP path); its absence frees you to optimize for availability/UX (AP path).

---

## 2. Definitions

| Term | Definition |
|---|---|
| **Vertical scaling** | Making one machine bigger. Simple, but ceilinged and superlinear in cost. |
| **Horizontal scaling** | Adding more machines. Unlocks scale/reliability but creates a distributed system. |
| **Replication** | Keeping copies of the same data on multiple nodes. |
| **Partitioning / Sharding** | Splitting the dataset so each node owns a distinct slice. |
| **Leader / Follower** | Single-leader replication: all writes go to the leader, which streams a replication log to followers (read replicas). |
| **Synchronous replication** | Write is acknowledged only after replica(s) confirm. Strong, slow, fragile to replica downtime. |
| **Asynchronous replication** | Write acknowledged immediately; propagation happens later. Fast, available, can lose acked writes. |
| **Semi-synchronous** | One sync follower + the rest async — the common production compromise. |
| **Replication lag** | The time window during which a follower is behind the leader. |
| **Read-after-write (read-your-writes)** | Guarantee that a user always sees *their own* most recent write. |
| **Monotonic reads** | Guarantee that a user's successive reads never move *backward* in time. |
| **Consistent prefix reads** | Guarantee that writes are seen in causal order (cause before effect). |
| **Quorum** | The threshold of nodes that must respond for an operation to count (`w` for writes, `r` for reads). |
| **`w + r > n`** | Quorum condition guaranteeing read/write set overlap. |
| **LWW (Last-Write-Wins)** | Conflict resolution by keeping the value with the latest *wall-clock* timestamp. Silently drops writes. |
| **Logical version number** | A monotonically increasing counter that encodes order directly (no clock). |
| **Version vector** | A set of per-replica counters; detects whether two writes are causally ordered or **concurrent**. |
| **Sibling** | One of multiple concurrent values preserved when a conflict is detected (must be merged). |
| **Hot key / Celebrity problem** | A single item receiving a disproportionate share of traffic, defeating naive load spreading. |
| **Sharded counter** | Splitting a high-contention counter into per-node tallies, summed on read (exploits commutativity). |
| **Optimistic UI update** | Showing an action's result on the client before the server confirms, reconciling on failure. |
| **CAP partition (P)** | A period during which some nodes cannot communicate with others. |
| **Invariant** | A correctness rule that must always hold (e.g. `balance ≥ 0`, "a seat has ≤ 1 owner"). |

---

## 3. Mental Models

- **Match the guarantee to the cost of being wrong.** Not "how important is the data" but "what happens if two copies briefly disagree?" Irreversible correctness violation → strong consistency. Cosmetic/annoying → eventual consistency.

- **Find the invariant first.** Before drawing any boxes, ask: *what rule must never break?*
  - Invariant present → coordination required → **CP**, consensus, the expensive path.
  - Invariant absent → freedom → **AP**, eventual consistency, optimize for UX.

- **Don't fight the tension — design so it can't hurt you.** Instead of forcing nodes to stay in sync, ask whether the operation is **commutative** (order-independent, like addition). If so, nodes can drift freely and still converge on merge (sharded counters, CRDTs).

- **Conditional-on-current-state is the danger signal.** An operation that must validate against shared state *before* committing (withdraw only if `balance ≥ 100`, claim the seat only if unclaimed) cannot be decided safely by one node alone — it needs coordination first.

- **Existence vs. identity.** A quorum overlap proves a fresh value *exists* in your response set; it does not tell you *which* value it is. Version metadata bridges existence → identity.

- **Logical order, never physical time.** Reliable ordering schemes encode order in the data path (counters/vectors). Wall clocks are an external, unsynchronized device and will lie about order.

---

## 4. Diagrams (ASCII)

### Sync vs. Async replication (the core trade)
```
SYNCHRONOUS                          ASYNCHRONOUS
client → leader → follower           client → leader ──ack──→ client
              ↘ (wait confirm)                     ↘ (background copy)
        ack only after follower OK            follower catches up later

safe, slow, fragile if follower down  fast, available, can LOSE acked write
        [favors Consistency]                  [favors Availability]
```

### The three replication-lag anomalies
```
READ-AFTER-WRITE        MONOTONIC READS          CONSISTENT PREFIX
you write, then read    read 1 sees new data,    you see an effect
a lagging follower —    read 2 (more-lagged      before its cause
your own write gone     replica) sees it vanish  (answer before question)
                        — time runs backward
FIX: read from leader   FIX: pin user to one     FIX: ensure causal
 / track your own write    replica (hash userID)    write ordering
```

### Quorum overlap (why `w + r > n`)
```
n = 5 nodes:   [A][B][C][D][E]
write quorum w=3 →  A B C   written
read  quorum r=3 →      C D E   read
                        ^ overlap = w + r - n = 1 node guaranteed fresh

If w + r ≤ n, read and write sets can be DISJOINT → stale read possible.
```

### LWW vs. Version Vectors (what each surrenders)
```
                 LWW (wall clock)        VERSION VECTORS
concurrent       picks 1 by timestamp    detects concurrency,
writes           → SILENTLY DROPS other  keeps both as siblings
resolution       automatic               app must merge / CRDT
cost paid        DURABILITY (data loss)  SIMPLICITY (merge burden)
clock skew?      vulnerable (wrong winner) immune (no clock used)
```

---

## 5. Engineering Takeaways

- **Default to semi-synchronous** for the leader path: one sync replica bounds data-loss risk without paying full-sync latency on every follower.
- **Async replicas are read scaling, not durability.** Treat an acked-but-unreplicated write as at risk until it propagates.
- **For user-editable data, read-your-writes from the leader** (or via a write-position token), not from an arbitrary follower.
- **Pin users to a replica** (hash of user ID) to get monotonic reads cheaply.
- **Never resolve conflicts by wall-clock time if losing a write is unacceptable.** LWW is fine for caches/cosmetic state; dangerous for anything a user was told "succeeded."
- **Quorums (`w + r > n`) are not linearizability.** Concurrent writes, partial write failures, and sloppy quorums all leave holes. Read DDIA's "Limitations of Quorum Consistency."
- **Exploit commutativity for high-contention counts** (sharded counters) instead of serializing every increment.
- **`fsync` on append = durability + crash recovery**, and append-only writes are sequential (fast). Same mechanism underlies WALs and log-structured storage.
- **Logs are transport, not archives.** Bound their size via truncation (once all followers consume) or compaction (keep only latest value per key → size ≈ #distinct keys).

---

## 6. Complexity / Cost Summary

Not an algorithmic lecture, but the operational cost profile:

| Operation | Latency driver | Failure exposure |
|---|---|---|
| Sync write | slowest replica in quorum | blocks if replica down |
| Async write | local only | loses acked write if leader dies in gap |
| Quorum read/write | `w`th / `r`th fastest response | tolerates loss of a *minority* of nodes |
| Quorum overlap | `w + r − n` guaranteed-fresh nodes | none, if `w + r > n` |

**Fault-tolerance rule of thumb:** majority quorum on `n = 2f + 1` nodes survives `f` failures. Use **majority, never unanimity** — unanimity makes the system *less* reliable than one machine.

---

## 7. Interview Notes

**Likely questions**
- "Design a system that scales reads" → read replicas + async replication + name the lag anomalies and fixes.
- "How do you keep a like/view counter at scale?" → hot key → sharded counters / commutativity → eventual consistency.
- "Two clients write the same key concurrently — what happens?" → LWW vs. version vectors; explain silent data loss.
- "Explain `w + r > n`" → pigeonhole overlap; then immediately note it's *not* linearizability.
- "Is this CP or AP?" → find the invariant first; state what the system does *during a partition*.

**Common candidate mistakes**
- Saying "use a cache" / "add a replica" without naming the cost.
- Requiring *all* replicas to ack (unanimity) — kills availability.
- Resolving conflicts/ordering by wall-clock timestamp (clock skew).
- Confusing read-after-write (your own writes) with monotonic reads (reads not regressing).
- Treating quorum reads as fully linearizable.
- Justifying CP/AP by "importance" instead of the invariant + cost-of-being-wrong.

**High-value framing line:** *"First I'd identify the invariant — that tells me whether I'm forced onto the consistency path or free to optimize for availability."*

---

## 8. Revision Sheet (≤ 5 min)

- **Distribution exists for:** scale + reliability → inherits 4 tensions (unreliable net, no shared clock, partial failure, consistency).
- **Replication** = same data, many nodes (reliability/read scaling). **Partitioning** = different data per node (capacity).
- **Sync** = safe/slow/fragile; **Async** = fast/available/can-lose-acked-write; **prod = semi-sync**.
- **Lag anomalies → fix:** read-after-write → read from leader; monotonic reads → pin to one replica; consistent prefix → causal ordering.
- **Quorum:** `w + r > n` ⇒ overlap ⇒ ≥1 fresh node. Overlap proves *existence*, not *identity* → need versions to pick.
- **Conflict:** LWW loses **durability** (silent drop, clock-skew-prone); version vectors lose **simplicity** (siblings/merge, but no data loss + concurrency detection).
- **CAP:** under partition pick C or A. Decide via **invariant**: present → CP; absent → AP.
- **Master move:** *find the invariant first;* match guarantee to cost of being wrong; use commutativity so drift self-heals.
- **Fault tolerance:** majority on `2f+1` survives `f`; never unanimity.

---

## 9. Knowledge Connections

**Builds on:** client–server model; the four tensions (foundation for everything distributed).

**Within this lecture:** invariant heuristic → CAP → sync/async choice → conflict resolution are one connected chain.

**Forward links (flagged for later):**
- **Consensus (Raft/Paxos)** — the formal "how an odd number of nodes agree on one outcome"; the CP machinery behind seat booking. (DDIA: Consistency & Consensus.)
- **Linearizability / causal consistency** — the rigorous replacements for the crude CAP model.
- **Partitioning/sharding** — the second data pillar; hot keys reappear as skewed workloads + hot-spot relief.
- **CRDTs / version vectors** — automatic conflict-free merge; deeper sibling resolution.
- **TrueTime (Spanner)** — bounding clock uncertainty to make physical time trustworthy.
- **Encoding & evolution (fwd/backward compatibility)** — needed for evolving services; pairs with Newman's *Building Microservices*.
- **WAL / log-structured storage** — same append-first durability mechanism seen here.

**Real systems:** PostgreSQL/MySQL replication (single-leader), Cassandra/DynamoDB (leaderless quorums, LWW/vector clocks), Kafka (log as source of truth, compaction), Twitter/Instagram (hot-key fan-out).

**OS-lab analogs:** page cache ↔ buffer pools; journaling ↔ WAL; locks ↔ distributed locks.

---

## 10. Flashcards

**Q:** Why does adding a second machine immediately create hard problems?
**A:** It creates a distributed system, inheriting four tensions: unreliable network, no shared clock, partial failure, and multi-copy consistency.

**Q:** Replication vs. partitioning — one line each.
**A:** Replication = copies of the *same* data (reliability/read scaling). Partitioning = *different* data per node (capacity).

**Q:** What does asynchronous replication risk that synchronous doesn't?
**A:** Losing an already-acknowledged write if the leader dies before propagation.

**Q:** Read-after-write vs. monotonic reads?
**A:** Read-after-write = you always see *your own* latest write. Monotonic reads = your successive reads never go *backward* in time.

**Q:** Fix for monotonic reads?
**A:** Pin each user to a single replica (e.g., hash of user ID) so they don't bounce between replicas at different lag.

**Q:** Why does `w + r > n` work?
**A:** Pigeonhole — read and write sets can't fit disjointly in `n` nodes, so they share ≥ `w + r − n` nodes, one of which holds the latest write.

**Q:** Does a quorum overlap tell you which value is freshest?
**A:** No — it guarantees a fresh value *exists* in the response set; you need version metadata to identify it.

**Q:** What does wall-clock LWW sacrifice, and why is it dangerous?
**A:** Durability — it silently discards the losing write, and clock skew can make it discard the genuinely newer one.

**Q:** What do version vectors give you that LWW can't?
**A:** They detect *concurrency* (neither vector dominates) and preserve both writes as siblings — no silent loss — at the cost of merge complexity.

**Q:** The master heuristic for picking a consistency model?
**A:** Find the invariant. Present → coordination/CP. Absent → freedom/AP. Then match the guarantee to the cost of being wrong.

**Q:** Why majority quorum, never unanimity?
**A:** Unanimity lets a single down node block all operations — less reliable than one machine. Majority on `2f+1` survives `f` failures.

**Q:** How do you scale a high-contention counter (likes)?
**A:** Sharded counters — per-node tallies summed on read; works because increment is commutative, so drift self-heals.

---

# Homework

### Core
1. **(Conceptual)** In your own words, explain why semi-synchronous replication is the common production default. What specific failure does the single synchronous follower protect against, and what does keeping the rest asynchronous preserve?
2. **(Conceptual)** Give a fresh real-world operation (not bank/likes/seat/bio) and classify it CP or AP. State the invariant (or its absence) and what the system should do during a partition.
3. 
### Advanced
4. **(Design)** Sketch read-your-writes for a profile page where a user edits their own bio but also views others' bios. Which reads hit the leader, which can hit followers, and how does a write-position token let you serve *some* of the user's own reads from a follower?
5. **(Trade-off)** DynamoDB-style systems often default to LWW. Given everything about silent data loss, why is that a *defensible* default for many workloads? When is it indefensible? Be precise about the workload property that flips the answer.

### Challenge
6. **(Synthesis)** Construct a concrete interleaving where a leaderless quorum with `w + r > n` *still* returns a stale or surprising result despite the overlap. (Hint: think concurrent writes, or a write that succeeded on some nodes but failed the quorum.) This previews "Limitations of Quorum Consistency."

### Reflection
7. Where in this lecture did your first instinct turn out backwards, and what was the corrected principle? Writing this down cements the *reasoning move*, not just the fact.
