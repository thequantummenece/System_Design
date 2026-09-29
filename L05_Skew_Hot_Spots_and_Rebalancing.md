# Lecture 5 — Skew, Hot Spots, and Rebalancing

**Course:** Self-Directed CS — System Design Lab
**Week 3 · Lecture 5**
**Builds on:** Lecture 1 (hot keys, commutativity), Lecture 4 (key-range vs. hash partitioning)
**Status:** ✅ Closed. Secondary indexes were covered through Lecture 4's homework; request routing is deferred.

---

## 1. Key Concepts

1. **Hash partitioning fixes clustered keys, not hot keys.** A hash spreads *distinct* keys evenly, but one key always hashes to one place. If a single key receives a large share of traffic, one partition saturates while the others idle. A better hash function cannot change this, because the hash is working correctly and the load is concentrated in one key.
2. **Key splitting is the standard application-level fix.** Append a small random suffix to a hot key (`post_42` becomes `post_42_00` … `post_42_99`) so its writes scatter across partitions.
3. **Costs of key splitting.** The application must know which keys are hot (databases generally don't detect this). Reads of the logical item become a fan-out over all sub-keys plus a merge. For keys that aren't hot, this is pure overhead.
4. **Key splitting only works for commutative, mergeable operations.** Like counters (sum) recombine cleanly. An "overwrite" has no merge function: 100 sub-keys each holding a "latest" value give no way to pick the winner without ordering information, which brings back the Lecture 1 problems (unreliable timestamps, version vectors).
5. **`hash(key) mod N` is a bad rebalancing scheme.** Changing N from 10 to 11 moves about 90% of keys, because a key keeps its node only when `hash mod 10 == hash mod 11` (roughly 1 in 11). Adding one node triggers a near-total reshuffle.
6. **The fix is decoupling two mappings that `mod N` fuses together:** key → partition, and partition → node. The key → partition mapping is fixed. Only partition → node changes when the cluster changes.
7. **Fixed partitions:** create many more partitions than nodes up front (e.g. 1,000 partitions on 10 nodes). Adding an 11th node moves whole partitions from existing nodes until load evens out (about 91 partitions, roughly 9 from each of 10 nodes). Everything else stays put.
8. **Partition count and node count are independent.** A partition is a cheap logical unit, not a machine.
9. **Rebalancing goals:** balanced load afterward, continued service while data moves, and minimal data movement.

---

## 2. Definitions

| Term | Definition |
|---|---|
| **Skew** | Uneven load across partitions or nodes. |
| **Hot key** | A single key receiving a disproportionate share of traffic; hashing cannot spread it because it always maps to one place. |
| **Key splitting** | Appending a random suffix to a hot key so its writes scatter across partitions; reads must query all sub-keys and merge. |
| **Commutative operation** | An operation whose result does not depend on order (e.g. increment), so partial results merge by simple combination. |
| **Rebalancing** | Moving data between nodes when the cluster changes (nodes added/removed, load shifts). |
| **Fixed partitions** | A scheme with a fixed, large number of partitions assigned to nodes; rebalancing moves whole partitions and never changes the key → partition mapping. |
| **Consistent hashing** | A scheme (ring, clockwise ownership) where adding a node moves only the keys in the arc between the new node and its predecessor. Same design goal as fixed partitions: minimal movement. |

---

## 3. Mental Models

- **Hashing balances *keys*, not *traffic*.** A uniform hash gives uniform key placement. Load is a property of traffic per key, which the hash can't see.
- **You can only split what you can merge.** Key splitting is safe exactly when the operation has a cheap, order-independent merge (sum, max, set union). If merging needs ordering or agreement, splitting reintroduces the coordination problem.
- **Decouple what changes from what doesn't.** `mod N` couples key placement to cluster size, so any cluster change moves almost everything. Inserting an indirection layer (key → partition → node) means cluster changes only touch the second mapping. The same pattern shows up elsewhere: an indirection layer absorbs churn.
- **Minimal movement is a requirement, not an optimization.** Rebalancing runs while the system is already under load, so moving less data directly reduces risk.

---

## 4. Diagrams (ASCII)

### Hash fixes clustering, not hot keys
```
Many distinct keys:   k1 k2 k3 k4 ...  --hash-->  spread evenly across P1..P4   OK
One hot key "celeb":  celeb celeb celeb celeb --hash--> P3 P3 P3 P3            P3 saturated
```

### Key splitting (and its read cost)
```
WRITE: celeb_post -> celeb_post_{random 00..99} -> scattered across partitions
READ:  query celeb_post_00 ... celeb_post_99 (fan-out) -> merge (sum) -> result
```

### Why mod N reshuffles almost everything (N=10 -> 11)
```
key stays only if hash mod 10 == hash mod 11  (~1 in 11)
=> ~90% of keys change nodes when you add one node
```

### Fixed partitions (1,000 partitions, 10 nodes -> 11 nodes)
```
key --hash--> partition (fixed, never changes) --> node (changes)

Before: 10 nodes x ~100 partitions
After:  11 nodes x ~91 partitions
        new node takes ~9 whole partitions from each old node (~91 total)
        all other partitions stay where they are
```

---

## 5. Engineering Takeaways

- **Detect hot keys before optimizing.** Don't split keys speculatively; apply splitting only to the specific keys that measurement shows are hot.
- **Check the operation before splitting.** Confirm the write is commutative/mergeable. For overwrites, consider other approaches (batching or coalescing writes at the application layer, funneling through a single writer, caching reads).
- **Provision many more partitions than nodes.** Choose a partition count high enough to keep future growth cheap, but not so high that per-partition overhead dominates.
- **Never use `hash(key) mod N` where N changes.** It is fine for a fixed N, but a poor foundation for a cluster you expect to resize.
- **Automatic rebalancing has risk.** Moving data adds load to a cluster that may already be struggling. Many systems keep a human in the loop for rebalancing decisions (see homework challenge).

---

## 6. Cost Summary

| Scheme | Data moved when adding a node | Hot-key resistance | Notes |
|---|---|---|---|
| `hash mod N` | ~ (1 − 1/(N+1)) of all keys (~90% at N=10) | None on its own | Fused key→node mapping |
| Fixed partitions | ~ 1/(N+1) of partitions (whole partitions only) | None on its own | Requires choosing a partition count up front |
| Consistent hashing | Keys in one arc only | None on its own | Same minimal-movement goal |
| Key splitting | n/a (a hot-key fix, not a rebalancing scheme) | Strong for mergeable ops | Read fan-out of k sub-keys |

---

## 7. Interview Notes

**Likely questions**
- "Your data is hash-partitioned but one partition is overloaded. Why, and what do you do?" → hot key; hashing can't split a single key; apply key splitting for mergeable operations.
- "Why not `hash(key) mod N` for a cluster that grows?" → adding a node moves ~90% of keys; decouple key → partition from partition → node.
- "How would you add a node to a running partitioned cluster?" → fixed partitions or consistent hashing; move whole partitions from existing nodes; serve traffic during the move.
- "Can you always split a hot key?" → only for commutative/mergeable operations; explain why overwrites fail.

**Common candidate mistakes**
- Claiming a better hash function solves hot keys.
- Splitting every key "just in case" (needless read overhead).
- Proposing key splitting for an overwrite/"latest value wins" workload without addressing ordering.
- Conflating partition count with node count.

**High-value framing line:** *"Hashing spreads keys, not traffic, so I'd check whether the imbalance is key-clustering or a hot key, then match the fix to the cause."*

---

## 8. Revision Sheet (≤ 5 min)

- Hash partitioning spreads **keys**, not **traffic**. A hot key still lands on one partition.
- Fix: **key splitting** (random suffix). Cost: reads fan out to all sub-keys; wasteful for non-hot keys.
- Splitting requires a **commutative/mergeable** operation (likes: yes; overwrite: no, no merge function).
- `hash mod N` moves ~90% of keys when N changes (10 → 11).
- Fix: **decouple** key → partition (fixed) from partition → node (changes). Many partitions, few nodes.
- Adding a node under fixed partitions: ~1/(N+1) of partitions move, taken evenly from existing nodes.
- Consistent hashing has the same goal: only one arc's keys move.
- Same heuristic as always: find the actual cause (access pattern) before choosing a fix.

---

## 9. Knowledge Connections

**Builds on:** Lecture 1 (celebrity problem, sharded counters, commutativity, last-write-wins limits); Lecture 4 (key-range vs. hash partitioning, secondary indexes as the second structure).

**Within this lecture:** hot keys (imbalance from traffic) → key splitting and its limit (mergeability) → rebalancing (imbalance from cluster change) → decoupling as the general fix.

**Forward links:**
- **Request routing:** once partitions move between nodes, how does a client find the right one? (Assigned as reading; revisited later.)
- **Dynamic partitioning:** splitting partitions automatically as they grow (Bigtable tablets, HBase regions), as opposed to fixing the count up front.
- **Transactions (Lecture 6+):** partitioning makes multi-partition operations hard; that's where distributed transactions and 2PC enter.
- **Cascading failures under automatic rebalancing:** ties to Lecture 2's "can't tell dead from slow."

**Real systems:** Cassandra/DynamoDB (hash partitioning with hot-partition handling), Elasticsearch and Kafka (fixed partition counts), Bigtable/HBase (dynamic key-range tablets/regions).

---

## 10. Flashcards

**Q:** Why doesn't a good hash function prevent a hot partition?
**A:** A hash spreads distinct keys evenly, but a single key always maps to one partition, so one very popular key concentrates its traffic there regardless of hash quality.

**Q:** How does key splitting work, and what does it cost?
**A:** Append a random suffix to a hot key so writes scatter across partitions. Reads must query all sub-keys and merge, and it is wasteful for keys that aren't hot.

**Q:** Why does key splitting work for a like counter but not for "set status"?
**A:** Increments are commutative and merge by summing. Overwrites have no merge function, so sub-keys holding different "latest" values can't be reconciled without ordering information.

**Q:** Why is `hash(key) mod N` bad when N changes?
**A:** A key keeps its node only if `hash mod N_old == hash mod N_new` (about 1 in N+1 for N → N+1), so ~90% of keys move when adding one node to a 10-node cluster.

**Q:** What's the core idea behind fixed-partition rebalancing?
**A:** Decouple key → partition (fixed) from partition → node (changeable). Adding a node only reassigns whole partitions.

**Q:** 1,000 partitions on 10 nodes, add an 11th. What moves?
**A:** About 91 partitions (roughly 9 from each existing node); key → partition mapping is unchanged.

**Q:** Does adding entities require adding partitions?
**A:** No. Partition count is fixed for a capacity plan; more keys hash into the same partitions.

---

# Homework

### Core
1. **(Conceptual)** In your own words: why can't a better hash function fix a hot key? Then explain how key splitting works around it and name both costs.
2. **(Computation)** A cluster has 2,000 fixed partitions on 20 nodes. You add 5 nodes. Roughly how many partitions move in total, how many does each new node receive, and roughly how many does each old node give up? Then state, for the same change under `hash mod N` (20 → 25), what fraction of keys keep their node and why.
3. **(Reading, textbook)** *Designing Data-Intensive Applications*, 2nd ed., **Chapter 7 — Partitioning**: the sections on skewed workloads/hot spots, rebalancing, and request routing (routing is new; note the main approaches). This chapter has no Transactions dependency.
4. **(Reading, paper)** *Bigtable: A Distributed Storage System for Structured Data* (Chang et al., OSDI 2006): https://research.google.com/archive/bigtable-osdi06.pdf. Focus on the data model and tablet sections (how tables split into row-range tablets, how tablets are assigned to servers). It is a key-range system, a useful counterpart to Dynamo's hash-based design from Lecture 4's paper.

### Advanced
5. **(Design)** A hot key is written with "set status = X" (overwrite), so key splitting doesn't apply. Propose two alternative ways to relieve the hot partition and state the trade-off of each.
6. **(Verification)** Secondary indexes: for a document-partitioned (local) index and a term-partitioned (global) index, which has cheaper *writes* and which has cheaper *reads*? One sentence of reasoning each.
7. **(Reading synthesis)** After reading Bigtable, explain in 4-5 sentences how automatic tablet splitting differs from the fixed-partitions scheme from this lecture, and what problem of key-range partitioning (Lecture 4) it addresses.

### Challenge
8. **(Failure analysis)** A node becomes slow (not dead). The cluster's automatic rebalancer decides it's failed and starts moving its partitions to other nodes. Trace how this can make things worse, and explain why many systems require a human to approve rebalancing. Connect this to the "can't tell dead from slow" tension from Lecture 2.

### Reflection
9. Lecture 4 taught "find the access pattern first." In this lecture, where did that same move apply, and where did a cause you might have assumed (uneven keys) turn out to be a different cause (uneven traffic)?

**Next session opens with:** Lecture 6 — Transactions (what ACID actually guarantees, and the race conditions transactions exist to prevent). It begins **only after your explicit go-ahead.**
