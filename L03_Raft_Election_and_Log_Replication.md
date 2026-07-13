# Lecture 3 — Raft: Election Protocol & Log Replication

**Course:** Self-Directed CS — System Design Lab
**Week 2 · Lecture 3**
**Builds on:** Lecture 1 (Replication & Consistency), Lecture 2 (Linearizability & Consensus Foundations)
**Status:** ✅ Closed — comprehension confirmed

---

## 1. Key Concepts

1. **Election timeouts** are randomized (not fixed) per node, reset on every leader contact. Randomization makes simultaneous candidacy rare, so one candidate almost always requests votes before any competitor's timer fires.
2. **Split votes** occur when two+ candidates time out near-simultaneously and votes divide, so nobody reaches majority. Randomized timeouts make this rare, not impossible; a failed election just retries with a fresh random timeout.
3. Majority-quorum election is **not primarily a tie-breaker** for split votes — its core job (from Lecture 2) is preventing split-brain across a *partition*, which can occur with zero ties, zero split votes (two disjoint groups, one candidate each, no competition, "most votes wins" would let both win).
4. **`AppendEntries`** is the single RPC for both real log replication and heartbeats (empty `AppendEntries` sent periodically, ~50–100ms) — heartbeats are not a separate mechanism.
5. **Log consistency check (`prevLogIndex`/`prevLogTerm`):** before appending, the leader requires the follower to confirm it has a matching entry immediately prior to the new one(s). Mismatch → reject → leader decrements and retries until it finds the last point of agreement → overwrites everything after with its own log. This is the **Log Matching Property**.
6. The consistency check exists to prevent the log **forking** — not merely to avoid a stale read. Without it, two different entries could each become **committed** at the same index on different servers, an unrecoverable state since "committed" must mean permanent.
7. **`commitIndex`:** the leader advances its own `commitIndex` once an entry is acked by a majority (leader counts itself). It propagates this via `leaderCommit` on every `AppendEntries`. A follower sets `commitIndex = min(leaderCommit, index of its own last appended entry)` — never committing entries it does not yet physically possess.
8. **Committing entries from previous terms — the closing subtlety.** "Replicated to a majority" is *not* sufficient on its own for permanent commitment. A leader may only *directly* commit an entry created in its **own current term**; entries from earlier terms are committed only *indirectly*, as a side effect of a later current-term entry being committed. Without this rule, an entry could sit on a majority and still be legitimately overwritten by a future leader — breaking the "committed = forever" guarantee.

---

## 2. Definitions

| Term | Definition |
|---|---|
| **Election timeout** | Randomized per-node timer; reset on any leader contact; expiry triggers candidacy. |
| **Candidate** | State a node enters on timeout: increments term, votes for itself, sends `RequestVote` to all peers. |
| **Split vote** | Two+ simultaneous candidates divide the electorate so neither reaches majority; election fails and retries. |
| **Livelock** | System makes active but non-converging progress (e.g., repeated failed elections) — distinct from deadlock, where nothing happens at all. |
| **Heartbeat** | An `AppendEntries` RPC with zero entries, sent periodically to signal leader liveness and reset followers' timeouts. |
| **`prevLogIndex` / `prevLogTerm`** | Fields in `AppendEntries` identifying the entry immediately preceding the new one(s); used by the follower to verify log continuity before accepting. |
| **Log Matching Property** | Guarantee that if two logs contain an entry with the same index and term, all preceding entries in both logs are identical. |
| **`commitIndex`** | Highest log index known to be committed (majority-replicated); leader advances it, followers derive it via `min(leaderCommit, own last index)`. |
| **State Machine Safety** | Raft's core guarantee: once an entry is applied at a given log index, no other entry is ever applied at that same index — logs never silently fork at a committed position. |
| **Committing entries from previous terms** | Rule that a leader may only directly commit entries from its own current term; older-term entries commit only indirectly via a later current-term entry. Prevents legitimately-overwritable "committed" entries. |

---

## 3. Mental Models

- **Majority election solves partition split-brain, not vote-splitting.** Don't reach for it as a "tie-breaker" reflex — construct the no-tie, no-split-vote partition scenario to see why it's still required.
- **Heartbeats aren't a separate system from replication — they're the empty case of the same RPC.** One mechanism, two payload shapes.
- **Check before you write, not after — because "after" may be unrecoverable.** The `prevLogIndex` check isn't cleanup-avoidance; committed entries are permanent by definition, so conflicts that reach commitment can't be "sorted out later." Prevention has to happen before acceptance.
- **A follower can only commit what it physically has.** `commitIndex` is capped by the follower's own log length — never trust the leader's number past what you've actually received.
- **"Majority replicated" is necessary but not sufficient for permanent commitment.** The current-term-only direct-commit rule is the missing piece that makes the guarantee airtight — a genuinely subtle corner most explanations skip.

---

## 4. Diagrams (ASCII)

### AppendEntries as the unifying mechanism
```
Leader ──heartbeat (empty AppendEntries)──→ Follower   (keeps timeout reset)
Leader ──AppendEntries[entries + prevLogIndex/Term]──→ Follower
                                          │
                              matches? ───┼─── yes → append, ack
                                          └─── no  → reject
                                                      ↓
                              leader decrements probe index, retries
                              until match found → overwrite from there
```

### Why majority-election isn't just about split votes (partition case)
```
5 nodes split: {A,B,C} | {D,E}   — NO split vote in either group

If rule were "most votes wins" (no majority floor):
  {A,B,C}: candidate gets 3 votes → wins
  {D,E}:   candidate gets 2 votes, no competitor → ALSO wins
  → two leaders, split-brain, despite zero ties anywhere

Majority-of-whole-cluster floor (⌊n/2⌋+1 = 3) caps {D,E} at "cannot win"
regardless of ties — this is the real job of majority quorum.
```

### commitIndex propagation
```
Leader commitIndex = 12 (majority acked through 12)
Follower log only reaches index 9 (still catching up)

AppendEntries → leaderCommit = 12
Follower sets: commitIndex = min(12, 9) = 9
  (never commits entries it doesn't yet possess)
```

### Committing entries from previous terms (the closing gap)
```
Term 2 leader replicates entry E to a majority — but leader loses
power BEFORE declaring E committed itself.

A later leader (term 4), doing legitimate log matching, could
overwrite E — even though a majority once held it — because E
was never directly committed in ITS OWN term.

FIX: leader only directly commits entries from its current term.
Older entries (like E) become committed only indirectly, as a
side effect of a later current-term entry being committed.
```

---

## 5. Engineering Takeaways

- **Randomize timeouts whenever multiple nodes might race for the same resource on a shared trigger** — this pattern (randomized backoff/timeout to avoid thundering-herd collisions) generalizes far beyond Raft (e.g., exponential backoff in retry logic, DHCP lease renewal jitter).
- **Heartbeats double as liveness proof and replication carrier** — a useful design economy: don't build a separate liveness-check protocol when your replication RPC can carry an empty payload.
- **Validate state before mutating it, especially when the mutation is meant to be irreversible.** The `prevLogIndex` check is a general pattern: cheap upfront verification prevents expensive, potentially unrecoverable conflicts downstream.
- **"Replicated to majority" is a necessary but incomplete definition of durability/commitment** in systems with leader changes — watch for this exact gap in other consensus-adjacent systems (this is a classic source of subtle bugs in home-grown "quorum-based" systems that don't implement the full Raft safety rules).

---

## 6. Complexity / Cost Summary

| Mechanism | Cost | What it buys |
|---|---|---|
| Randomized election timeout | Slightly longer average time-to-elect vs. optimal fixed timing | Avoids livelock from repeated split votes |
| Heartbeats | Constant background traffic (~10–20/sec typical) | Leader liveness detection, timeout reset |
| `prevLogIndex` check + backward probing | O(divergence length) extra round trips on a mismatch | Guarantees Log Matching Property, prevents forks |
| Current-term-only direct commit | Slight commit latency in edge cases (waiting for a current-term entry) | Closes the previous-term overwrite gap — true permanence |

---

## 7. Interview Notes

**Likely questions**
- "Why randomize election timeouts instead of using a fixed value?" → prevents livelock from repeated split votes.
- "Walk through what happens when a follower's log has diverged from the leader's." → `prevLogIndex`/`prevLogTerm` mismatch → reject → backward probe → overwrite.
- "Is 'replicated to a majority' the same as 'committed'?" → No — leader must own the term the entry was created in to commit it directly; otherwise it's a trap question testing Figure-8-style edge cases.
- "Why does a follower cap its `commitIndex` at `min(leaderCommit, own last index)`?" → cannot apply entries it doesn't have.

**Common candidate mistakes**
- Treating majority-quorum election as solely a split-vote tie-breaker (misses the partition/split-brain motivation).
- Assuming "majority replicated" alone means "safely committed forever" — misses the previous-term commit subtlety.
- Describing heartbeats as a separate protocol from log replication.
- Believing followers independently determine commitment — they don't; only the leader decides, followers just receive `leaderCommit`.

**High-value framing line:** *"Majority replication is necessary but not sufficient for permanent commitment — a leader can only directly commit entries from its own term, precisely to avoid a later leader legitimately overwriting something that looked durable."*

---

## 8. Revision Sheet (≤ 5 min)

- **Election timeout:** randomized, resets on leader contact; expiry → candidacy (term++, self-vote, RequestVote to all).
- **Split vote:** rare due to randomization; not impossible; resolves via retry with fresh randomized timeout.
- **Majority quorum's real job:** prevent split-brain across a partition — not just break ties (partition case has zero ties, still needs the majority floor).
- **`AppendEntries`** = heartbeat (empty) or replication (with entries) — one RPC, two uses.
- **`prevLogIndex`/`prevLogTerm`** check before append → Log Matching Property → prevents committed-entry forks, not just stale reads.
- **`commitIndex`:** leader advances on majority ack; follower sets `min(leaderCommit, own last index)` — never commits what it doesn't have.
- **The closing gap:** majority-replicated ≠ automatically committed-forever. Leader only *directly* commits current-term entries; older entries commit *indirectly* via a later current-term entry.

---

## 9. Knowledge Connections

**Builds on:** Lecture 2 — terms, majority election, Leader Completeness (this lecture supplies the mechanics that *implement* what Lecture 2 proved must be true).

**Within this lecture:** election timeouts → majority's real purpose (partition, not just ties) → AppendEntries/log matching → commit propagation → previous-term commit subtlety — one continuous chain from "how a leader is chosen" to "what commitment truly guarantees."

**Forward links:**
- **Leaderless quorums vs. linearizability** — still deferred from Lecture 2; now that Raft's full commit semantics are understood, this comparison will be sharp and fast.
- **Original Raft paper (Ongaro & Ousterhout), §5.4.2 "Committing entries from previous terms," Figure 8** — the definitive worked example of the gap closed this lecture; worth reading directly, it's clarifying beyond any summary.
- **Multi-Paxos** — alternate consensus algorithm solving the same problem with a different (less prescriptive) mechanism.
- **Distributed transactions (2PC)** — the multi-object generalization; consensus is the single-object/single-log foundation transactions build on.

**Real systems:** etcd, Consul (Raft directly); Kafka's ISR/controller election is Raft-like in spirit; CockroachDB/TiDB (Raft per data range).

---

## 10. Flashcards

**Q:** Why are election timeouts randomized rather than fixed?
**A:** Fixed timeouts cause every node to become a candidate simultaneously on leader failure, guaranteeing repeated split votes (livelock in the idealized case). Randomization makes one candidate almost always act first.

**Q:** Does majority-quorum election exist mainly to break split votes?
**A:** No — its core purpose is preventing split-brain across a network partition, a scenario with zero split votes (each side has exactly one candidate, no competition).

**Q:** What is a heartbeat, mechanically?
**A:** An `AppendEntries` RPC with zero entries — the same replication mechanism, used for liveness signaling.

**Q:** What does the `prevLogIndex`/`prevLogTerm` check prevent, precisely?
**A:** The log itself forking — two different entries becoming committed at the same index on different servers, which would be a permanent, unrecoverable inconsistency.

**Q:** How does a follower compute its own `commitIndex`?
**A:** `min(leaderCommit, index of its own last appended entry)` — it can never commit entries it hasn't actually received.

**Q:** Is "replicated to a majority" sufficient for permanent commitment?
**A:** Not on its own. A leader may only directly commit entries from its own current term; entries from earlier terms are committed only indirectly, via a later current-term entry. Without this, a majority-replicated entry could still be legitimately overwritten by a future leader.

---

# Homework

### Core
1. **(Conceptual)** In your own words: why is "majority replicated" necessary but not sufficient for an entry to be permanently committed? Use the previous-term commit rule in your answer.
2. **(Trace)** A 5-node cluster's leader replicates entry #20 to itself and 2 followers (3 of 5 — a majority) and advances its `commitIndex`. Before the remaining 2 followers catch up, the leader crashes. Walk through what must be true about the next elected leader's log for entry #20 to survive, using the election restriction from Lecture 2.
3. **(Reading)** *Designing Data-Intensive Applications*, 2nd ed., **Chapter 10 — Consistency and Consensus**: read the consensus-algorithms/coordination-services portion if not already covered, focusing on how the book frames "safety vs. liveness" properties of consensus — write one line distinguishing the two.

### Advanced
4. **(Design)** Sketch what changes (if anything) in the election-timeout and heartbeat mechanics if cluster size grows from 5 to 51 nodes. What operational concern becomes significant at that scale that wasn't at 5?
5. **(Reading, optional but recommended)** Ongaro & Ousterhout, *"In Search of an Understandable Consensus Algorithm"* (the original Raft paper) — read §5.4.2 and Figure 8 directly. Summarize the exact scenario in 3–4 sentences, in your own words, without re-reading this lecture's notes.

### Challenge
6. **(Synthesis)** Construct a concrete scenario (nodes, terms, log entries) where an entry is replicated to a majority of servers, is *not* yet committed by the current-term rule, and a new leader legitimately overwrites it — without violating anything else covered in Lectures 2–3. This is the Figure-8 scenario; build it yourself before or after reading the paper.

### Reflection
7. You caught a real inconsistency in a live explanation this lecture ("sort out later" vs. a scenario where it was never sorted). What made you notice it — and where else in this course might the same kind of internal-consistency check be worth applying, even to material you're inclined to accept?

**Next session opens with:** Lecture 4 (topic TBD — likely partitioning/sharding, or the deferred leaderless-quorum-vs-linearizability comparison) — **starting only after your explicit go-ahead.**
