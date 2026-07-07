# Lecture 2 — Linearizability & Consensus (Part 1: Foundations)

**Course:** Self-Directed CS — System Design Lab
**Week 2 · Lecture 2**
**Builds on:** Lecture 1 (Replication & Consistency)
**Status:** ✅ Closed — Part 1. Full Raft election protocol + log replication → Lecture 3.

---

## 1. Key Concepts

1. **Linearizability** = the illusion of one copy of the data, with every operation taking effect at a single, instantaneous point in time. Once any observer sees a new value, every subsequent read by anyone must see that value or newer — time never runs backward globally.
2. Linearizability is **stronger** than monotonic reads: monotonic reads promises no regression *for one user*; linearizability promises no regression *for any observer, system-wide*.
3. A history is linearizable iff there **exists a single global ordering** of all operations (consistent with real-time order for non-overlapping ops) that every observed value is consistent with. Violations are proven by showing no such ordering can exist.
4. Quorums (`w + r > n`) guarantee overlap for *completed* writes, but say nothing about **in-flight** writes — this is the seam where quorum systems fall short of linearizability (full treatment deferred to after Raft).
5. The real danger of leader failure isn't just data loss — it's **split-brain**: two leaders active simultaneously, each stamping a conflicting timeline, because nodes cannot distinguish "dead" from "slow/partitioned."
6. **Majority quorum for leader election** structurally prevents split-brain: any two majorities of the same cluster must overlap in ≥1 node (pigeonhole), and a node votes only once — so two disjoint majorities can never both exist.
7. **Terms** are a logical clock (not wall-clock) numbering elections. Higher term always wins; stale-term messages are rejected; a leader seeing a higher term steps down immediately.
8. **Committed vs. uncommitted writes:** committed = replicated to a majority, client told "success," must survive every future leader change. Uncommitted = never acknowledged, safe to discard.
9. **Election restriction:** a voter refuses a candidate whose log is less up-to-date than its own (compared via last-log-term, then last-log-index). Combined with majority overlap, this proves **Leader Completeness** — a new leader always already holds every committed write.

---

## 2. Definitions

| Term | Definition |
|---|---|
| **Linearizability** | Strongest single-object consistency guarantee: system behaves as one copy, operations take effect atomically, no observer ever sees time move backward. |
| **Split-brain** | Two nodes simultaneously believing they are leader, each accepting writes independently — produces contradictory, not just stale, data. |
| **Term** | A monotonically increasing integer numbering each election period in Raft; acts as a logical clock for leadership. |
| **Committed entry** | A log entry replicated to a majority of nodes; guaranteed durable across all future leader changes. |
| **Uncommitted entry** | A log entry not yet replicated to a majority; safe to discard on leader change since no client was told it succeeded. |
| **Election restriction** | Raft rule: a node denies its vote to any candidate whose log is less up-to-date than its own. |
| **Leader Completeness Property** | Raft's core safety guarantee: any newly elected leader already contains all previously committed entries. |
| **RequestVote RPC** | Raft's election message; carries candidate's term, `lastLogTerm`, and `lastLogIndex` so voters can apply the election restriction. |

---

## 3. Mental Models

- **Linearizability test = "can I construct one global order everyone agrees with?"** If yes → linearizable. If a value must have appeared both before and after a given read to explain all observations → contradiction → not linearizable.
- **Durability ≠ Agreement.** Synchronous replication makes a *write* safe (survives a crash). Quorum voting makes a *decision* unique (only one winner can ever exist). Don't reach for the durability tool when the problem is "prevent two simultaneous authorities."
- **Logical counters beat physical clocks whenever "who's newer" must never be gamed by clock skew** — first seen with version numbers (Lecture 1), now with terms (Lecture 2). Same principle, two applications.
- **Overlap proves existence; a rejection rule converts existence into guarantee.** Pigeonhole alone only proves *some* voter holds the committed data — it takes an explicit rule (election restriction) to force that voter to block an unqualified candidate. Overlap is necessary, not sufficient.
- **A promise ("success") is the boundary of what must survive.** Anything acknowledged to a client must survive every future failure/reconfiguration; anything not yet acknowledged owes no such guarantee.

---

## 4. Diagrams (ASCII)

### Linearizability violation — proving no valid global order exists
```
Client A:  write x=9   |———————————————|
Client B:  read            |—| → 0
Client C:  read                  |—| → 9
Client D:  read                        |—| → 0   ← VIOLATION

Once C observes 9, the write must be ordered BEFORE C.
D reads AFTER C in real time, so D must also see 9 (or newer).
D returning 0 has no valid placement in a single global order → not linearizable.
```

### Split-brain — the danger a naive "leader silent → re-elect" rule creates
```
        [ L1 leader, term 4 ]
               |
      ---- network partition ----
     /                            \
[F1, F2]  (can't reach L1)      [L1] (alive, still serving reachable clients)
     |
  timeout → elect L2, term 5
     |
[L2 leader, term 5] ...now TWO leaders accepting writes concurrently
     → contradictory timelines, not just stale data
```

### Why majority election prevents split-brain (pigeonhole)
```
5 nodes: need 3 votes to win.
Partition A: {1,2,3}  → CAN reach 3 votes → may elect a leader
Partition B: {4,5}    → CANNOT reach 3    → elects no one

Any two majorities of 5 nodes must share ≥1 node (3+3 > 5).
That shared node votes once → two disjoint majorities are impossible.
⇒ at most one leader can ever be elected.
```

### Leader Completeness derivation
```
W committed  ⇒  W is on majority {A,B,C}
Winner needs a majority of votes
Two majorities overlap in ≥1 node (pigeonhole)      → that node holds W
Election restriction: voter with more up-to-date     → refuses candidates
   log (i.e., holding W) denies vote to any candidate    missing W
   missing W
────────────────────────────────────────────────────────────────
⇒ No candidate missing W can ever assemble a majority
⇒ Every newly elected leader already holds every committed write
   (Leader Completeness Property)
```

---

## 5. Engineering Takeaways

- **Never resolve "who is the real leader" using wall-clock time or timeouts alone** — use a logical term counter with strict "higher always wins, lower always rejected" semantics.
- **A node going silent must never trigger unconditional re-election if the old leader might still be alive and reachable by others** — this is exactly the split-brain trap. Majority-based election is what makes re-election safe.
- **"Success" to a client should only be returned after majority replication**, never after a single node ack — this is the line between an entry that must survive forever and one that's safe to discard.
- **Sync replication and quorum voting solve different problems.** Use sync replication for durability of a single write; use quorum voting when you need to guarantee uniqueness of a decision (e.g., leadership, exclusive claims).
- **Don't conflate "quorum read overlaps a write" with "linearizable."** Overlap says a fresh value exists among your responses; it says nothing about in-flight writes or agreement on "has this happened yet." True linearizability needs consensus-backed ordering, not ad hoc quorum reads.

---

## 6. Complexity / Cost Summary

Not algorithmic this lecture, but the operational cost profile of the guarantees:

| Guarantee | What it costs |
|---|---|
| Linearizability | Highest latency/availability cost — every operation must be globally ordered; sacrificed first under a CAP partition. |
| Majority-quorum election | Cluster is unavailable for writes during the minority side's downtime — but this is the price of eliminating split-brain, not a flaw. |
| Committed-write durability | Requires majority replication before ack — write latency bounded by the median (not slowest) node, unlike full-sync-to-all. |

---

## 7. Interview Notes

**Likely questions**
- "What does linearizability actually guarantee, and how is it different from eventual/causal consistency?"
- "Why can't you just re-elect a leader the instant it stops responding?" → split-brain, partial failure, can't distinguish dead/slow.
- "How does Raft prevent a new leader from overwriting committed data?" → majority overlap + election restriction → Leader Completeness.
- "Why use a term counter instead of a timestamp for leadership?" → clock skew can let a stale leader appear "newer."
- Given a sequence of reads/writes with real-time overlaps, determine if it's linearizable — practice constructing (or disproving) a single global order.

**Common candidate mistakes**
- Treating "quorum read overlaps write set" as equivalent to linearizability (it isn't — overlap ≠ ordering agreement).
- Assuming discarding *any* leader's unreplicated writes is a "trade-off" — it isn't a loss if the write was never acknowledged.
- Reaching for synchronous replication to solve a leader-uniqueness problem (durability tool ≠ agreement tool).
- Using wall-clock time to compare leader legitimacy across nodes.

**High-value framing line:** *"Committed means majority-replicated and acknowledged — that's the line between what must survive every failure and what's safe to discard."*

---

## 8. Revision Sheet (≤ 5 min)

- **Linearizability:** one-copy illusion; once observed, locked in for everyone; test = can you construct one global order consistent with every read?
- **Danger of naive re-election:** split-brain (two leaders, contradictory timelines) — worse than lost data, because it's undetectable until reconciliation.
- **Fix:** majority-quorum election. Two majorities of the same cluster always overlap (pigeonhole) → only one can exist at a time.
- **Terms:** logical clock for leadership. Higher always wins; lower always rejected; leader steps down on seeing higher term.
- **Committed** (majority-replicated, acked) → must survive forever. **Uncommitted** → safe to discard, no promise broken.
- **Leader Completeness:** committed write on majority → overlap guarantees a voter has it → election restriction makes that voter block any candidate missing it → new leader always has all committed writes.
- **Durability tool ≠ Agreement tool:** sync replication protects data; quorum voting protects uniqueness of a decision.

---

## 9. Knowledge Connections

**Builds on:** Lecture 1 — quorums/pigeonhole (reused three times now: seat booking, `w+r>n`, leader election), logical version numbers (direct ancestor of terms), invariant heuristic (committed-write survival is itself an invariant).

**Within this lecture:** linearizability (the spec) → split-brain (why naive leadership fails it) → majority election (the fix) → terms (make the fix robust to time) → election restriction (make the fix preserve data) — one connected chain ending in Leader Completeness.

**Forward links:**
- **Raft full protocol** (Lecture 3) — election timeouts, heartbeats, `RequestVote`/`AppendEntries` mechanics, commit-index advancement.
- **Why leaderless quorums aren't linearizable by default** — deferred; revisit once Raft is solid, will be quick given this foundation.
- **Paxos** — alternate consensus algorithm, same guarantees, different mechanics.
- **Distributed transactions / 2PC** — the multi-object generalization of "agree before committing."

**Real systems:** etcd, Consul, ZooKeeper (Raft/ZAB-based consensus for leader election and config); CockroachDB/Spanner (linearizable transactions built atop consensus); MongoDB replica sets (term-like election epochs).

---

## 10. Flashcards

**Q:** What is the core illusion linearizability provides?
**A:** That there is one copy of the data and every operation happens atomically at a single instant — no observer ever sees values move backward in time.

**Q:** How do you prove a history is *not* linearizable?
**A:** Show that no single global ordering of operations (respecting real-time order) is consistent with all observed reads.

**Q:** Why is split-brain worse than losing unreplicated writes?
**A:** Lost writes are missing data (bounded, detectable). Split-brain produces contradictory data from two simultaneously "valid" leaders — may be unreconcilable.

**Q:** Why does majority-quorum election prevent split-brain?
**A:** Any two majorities of the same cluster must overlap in at least one node (pigeonhole); that node votes once, so two disjoint majorities — and thus two leaders — can never coexist.

**Q:** Why must Raft's term be a logical counter, not a timestamp?
**A:** Wall clocks can skew; a stale leader with a fast clock could otherwise appear "newer" than the legitimate leader, letting "higher wins" reinstate split-brain.

**Q:** What's the difference between committed and uncommitted entries?
**A:** Committed = replicated to a majority, client acknowledged, must survive all future leader changes. Uncommitted = never acknowledged, safe to discard.

**Q:** State the election restriction rule.
**A:** A node refuses its vote to any candidate whose log is less up-to-date than its own (compared by last log term, then index).

**Q:** How do overlap and the election restriction together prove Leader Completeness?
**A:** Overlap guarantees a voter holding any committed entry is among any winning majority; the election restriction guarantees that voter denies votes to candidates missing that entry — so no candidate lacking it can ever win.

**Q:** Durability tool vs. agreement tool — give one example of each and what each protects.
**A:** Synchronous replication (durability) protects a single write from being lost. Majority-quorum voting (agreement) protects against two nodes both claiming the same exclusive role/decision.

---

# Homework

### Core
1. **(Conceptual)** In your own words, explain why "the write only had one ack, so the quorum guarantee shouldn't apply yet" was the correct objection that exposed the earlier flawed example. What does this tell you about what `w + r > n` actually promises?
2. **(Construction)** Write a 4-operation history (mix of reads/writes, at least one pair overlapping in time) that IS linearizable, and explain why a valid global order exists.

### Advanced
3. **(Design)** A 7-node Raft cluster partitions into groups of 4 and 3. Which side (if either) can elect a leader? What happens to write requests sent to the 3-node side during the partition?
4. **(Trace)** Node X was leader in term 6 with one uncommitted entry (only on X). It gets partitioned out; the remaining nodes elect Y as leader in term 7 and commit new entries. X rejoins. Trace exactly what happens to X's uncommitted entry, using the rules from this lecture.

### Challenge
5. **(Synthesis)** Earlier, an attempted example tried to show quorums failing linearizability but was flawed because it ignored write-completion status. Now that you understand committed vs. uncommitted precisely: construct a correct scenario where a **completed** (majority-acked) write still allows two different linearizability-violating reads under `w + r > n`. (This previews the quorum/linearizability gap we deferred — attempt it, we'll verify next time.)

### Reflection
6. Where did your reasoning conflate two rules that sound similar but do different jobs (this lecture had at least one clear instance)? Naming the confusion precisely is what prevents it from recurring under interview pressure.

7. (Reading) Designing Data-Intensive Applications, 2nd ed. — Chapter 10, "Consistency and Consensus." Read the sections on linearizability and the leader election / consensus algorithms portion (Raft/Paxos coverage — your edition's section titles may differ slightly from the 1st edition's "Consensus Algorithms and Coordination Services," so use judgment on section names, but the chapter itself is confirmed). Editorial note for calibration: this chapter was almost completely rewritten for your edition specifically for clarity, so it should land well on top of what we just built. Skip the batch/stream processing material that follows — not relevant yet. 

**Next session opens with:** Lecture 3 — Raft's full election protocol and log replication (heartbeats, `AppendEntries`, commit-index advancement) — **starting only after your explicit go-ahead**, per the new protocol.
