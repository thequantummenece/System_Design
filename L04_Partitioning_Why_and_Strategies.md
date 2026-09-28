# Lecture 4 — Partitioning: Why, and the Two Core Strategies

**Course:** Self-Directed CS — System Design Lab
**Week 3 · Lecture 4**
**Builds on:** Lecture 1 (Replication & Consistency)
**Status:** ✅ Closed — comprehension confirmed, exceeded

---

## 1. Key Concepts

1. **Replication and partitioning solve orthogonal problems.** Replication multiplies copies of the *whole* dataset (reliability, read-scaling, sometimes write-entry-point distribution). Partitioning divides the dataset *itself* — it's the only technique that reduces per-node storage and processing load. Production systems run both, layered: partition for capacity/write-throughput, replicate each partition for fault tolerance.
2. **Multi-leader/leaderless replication does not substitute for partitioning.** Every node in those topologies remains a full replica — it must eventually apply *every* write system-wide, not just the ones it locally accepted. Total system-wide write-processing work stays ~N× the write volume (N = replica count), not divided by N. Only partitioning divides the actual work.
3. **Key-range partitioning:** sort keys, assign contiguous ranges to partitions. Cheap range queries (touch a small, contiguous set of partitions). Risk: skewed key distributions (e.g., timestamp keys where all new writes land in "today's" range) create a hot partition — the celebrity problem recurring at the partition level.
4. **Hash-based partitioning:** hash(key) decides the partition. Scatters writes near-uniformly, eliminating hot spots from skewed key distributions. Cost: destroys key locality — a range query must scatter-gather across every partition instead of touching a contiguous few.
5. **Composite/compound partition keys** can combine both: e.g., `hash(sensor_id)` picks the partition (spreads writes across sensors), data *within* that partition sorted by timestamp (keeps range queries for one entity efficient). Resolves the specific "one entity over a time range" query pattern.
6. **Some query-pattern combinations are structurally irreconcilable by any single partition key.** "Spread one entity's writes evenly across nodes" and "co-locate all entities' data for one instant on one node" organize data along two different axes — satisfying one destroys the other. No cleverer hash fixes this; it requires a second structure (secondary index / materialized rollup) alongside the primary layout.
7. **Partition count is decoupled from entity count.** `hash(key) mod N` uses a fixed N chosen for capacity, not one partition per key/entity. Growing entity count doesn't grow partition count; growing partition count is a deliberate resharding decision.
8. **Master heuristic, generalized from Lecture 1:** find the actual access pattern (query shape, write distribution, growth pattern) *first* — let it decide the partitioning strategy, not the reverse. Same "requirements before design" move as "find the invariant first," now applied to a new axis.

---

## 2. Definitions

| Term | Definition |
|---|---|
| **Partitioning / Sharding** | Splitting a dataset across nodes so each holds a distinct slice, dividing storage and processing load. |
| **Key-range partitioning** | Assigning contiguous key ranges to partitions; efficient for range queries, vulnerable to skewed-key hot spots. |
| **Hash-based partitioning** | Assigning partitions via a hash of the key; even write distribution, destroys range-query locality. |
| **Hot partition** | A single partition receiving disproportionate load due to key-range clustering (e.g., all "current timestamp" writes) — the celebrity problem at partition granularity. |
| **Scatter-gather query** | A query that must be sent to and aggregated across every partition because no single/few partitions hold all relevant data. |
| **Composite partition key** | A key built from multiple fields (e.g., `hash(entity_id)` + sorted timestamp) to satisfy more than one access pattern simultaneously. |
| **Consistent hashing** | (Named, mechanics deferred to Lecture 5) A hashing scheme that allows partition count to grow without mass-reshuffling existing key assignments. |

---

## 3. Mental Models

- **Multiplies copies vs. divides the dataset.** The one-line test for "does this technique actually reduce per-node load": does each node still need the whole dataset (replication) or only a slice (partitioning)? Multi-leader/leaderless still fails this test — every node is still a full copy.
- **Find the access pattern first, then pick the partition key** — the direct generalization of Lecture 1's "find the invariant first." Design follows requirements; requirements don't follow design.
- **Some query/write pattern pairs are structurally opposed, not just hard.** When two goals organize the same data along different axes (by entity vs. by moment, here), no single physical layout serves both — the fix is a second structure, not a smarter key.
- **Partition count answers "how much capacity," not "how many entities."** Don't conflate the two — a fixed N handles arbitrarily many keys; N only grows when load/capacity genuinely requires it.

---

## 4. Diagrams (ASCII)

### Replication vs. partitioning — orthogonal axes
```
REPLICATION (multiplies copies)         PARTITIONING (divides dataset)
Node A: [ALL data]                      Node A: [slice 1]
Node B: [ALL data]  (same data,         Node B: [slice 2]  (different data,
Node C: [ALL data]   N copies)          Node C: [slice 3]   1/N each)

→ reliability, read-scaling            → capacity, write-throughput
→ does NOT reduce per-node storage     → does NOT alone provide fault tolerance
  or total write-processing work         (still need to replicate each slice)

PRODUCTION SYSTEMS: partition for capacity, THEN replicate each partition.
```

### Multi-leader ≠ partitioning (the write-volume math)
```
1M writes, 5 nodes, evenly distributed:

PARTITIONING (5 shards):
  each node processes 200K writes, ONCE.  Total system work = 1M.  True 5x parallelism.

MULTI-LEADER (5 leaders, full replicas):
  each leader accepts 200K locally, but must ALSO apply the other 800K
  from peers to stay a full copy.
  Total system work ≈ 5M write-applications.  No division of labor.
```

### Key-range vs. hash — the tradeoff
```
KEY-RANGE                              HASH-BASED
[A-F][G-M][N-S][T-Z]                   hash(key) → scattered across partitions

Range query "M to P":                  Range query "M to P":
  touches 2 contiguous partitions ✓      must scatter-gather ALL partitions ✗

Skewed writes (e.g. all "today"):      Skewed writes:
  ALL land on one partition ✗            spread evenly ✓ (hash scatters uniformly)
```

### Composite key resolves ONE tension, not all
```
Key = hash(sensor_id), sorted by timestamp within partition

Query "sensor #482, T1→T2":  → 1 partition, range-scan within it ✓ (fixed)
Query "all sensors at T1":   → still touches EVERY partition ✗ (unresolved —
                                 structurally opposed to write-scatter goal;
                                 needs a secondary structure, not a better key)
```

---

## 5. Engineering Takeaways

- **Always ask "does this reduce per-node load, or just add copies?"** before treating a replication topology as a scaling solution for write throughput or storage.
- **Layer partitioning (capacity) with replication (fault tolerance) — they're not substitutes.**
- **Audit your dominant query patterns *before* choosing key-range vs. hash** — and check both the read pattern AND the write-skew pattern; a scheme that's great for reads can be a hot-spot disaster for writes on the same key.
- **When you find a query pattern that's structurally opposed to your write-distribution goal, don't keep tuning the partition key.** Recognize the conflict and reach for a second structure (secondary index, materialized rollup) instead.
- **Don't conflate "number of entities" with "number of partitions."** Capacity planning for partition count is a separate decision from how many keys/entities the system holds.

---

## 6. Complexity / Cost Summary

| Scheme | Range query cost | Write-skew resistance | Elastic growth cost |
|---|---|---|---|
| Key-range | O(few contiguous partitions) | Poor — vulnerable to clustered keys (e.g. timestamps) | Adding a new range = no reshuffle of old data |
| Hash (naive mod N) | O(N) — scatter-gather | Strong — near-uniform spread | Changing N reshuffles nearly all keys |
| Hash (consistent hashing) | O(N) — scatter-gather | Strong | Minimal reshuffle on growth (mechanics: Lecture 5) |
| Composite (hash + sorted sub-key) | O(1 partition) for the matching entity/range | Strong for the hashed dimension | Same as underlying hash scheme |

---

## 7. Interview Notes

**Likely questions**
- "Why not just add more replicas to handle write load?" → replication doesn't divide work; every replica still needs every write.
- "Key-range vs. hash partitioning — when would you use each?" → audit dominant query pattern (range scans vs. point lookups) AND write-skew risk, not just one or the other.
- "How would you partition a time-series/IoT dataset?" → composite key (hash entity ID + sort by time) to resolve the naive timestamp-only hot-spot problem; then name the remaining unresolved query pattern (aggregate-across-entities) as a secondary-index problem.
- "What happens to a range query under hash partitioning?" → scatter-gather across all partitions — name the cost explicitly, don't just say "it still works."

**Common candidate mistakes**
- Treating multi-leader/leaderless replication as a write-throughput scaling solution — it isn't; only partitioning divides the actual work.
- Picking key-range or hash based only on the read pattern, ignoring write-skew risk (or vice versa).
- Assuming a cleverer partition key can satisfy every query shape — some access-pattern combinations are structurally opposed and need a second structure.
- Conflating partition count with entity/key count.

**High-value framing line:** *"I'd start from the actual access pattern — read shape and write-skew risk both — before picking a partitioning strategy, the same way I'd start from the invariant before picking a consistency model."*

---

## 8. Revision Sheet (≤ 5 min)

- **Replication multiplies copies (reliability, read-scaling). Partitioning divides the dataset (capacity, write-throughput).** Orthogonal — production systems layer both.
- **Multi-leader/leaderless ≠ partitioning:** every node stays a full replica; total system work scales with replica count, not divided by it.
- **Key-range:** cheap range queries, vulnerable to skewed-key hot spots (e.g. timestamp writes).
- **Hash-based:** even write distribution, destroys range-query locality (scatter-gather).
- **Composite keys** (hash + sorted sub-key) resolve "one entity over a range" — but some patterns (aggregate-across-entities-at-one-moment) are structurally opposed to write-scatter and need a secondary structure, not a better key.
- **Partition count ≠ entity count** — fixed N handles arbitrary key volume; N changes only for genuine capacity reasons.
- **Master heuristic:** find the access pattern first, then pick the strategy — the partitioning-level version of "find the invariant first."

---

## 9. Knowledge Connections

**Builds on:** Lecture 1 — the celebrity/hot-key problem (recurs here at partition granularity), the "find the invariant first" heuristic (generalized here to "find the access pattern first"), replication topologies (multi-leader/leaderless, now contrasted against partitioning).

**Within this lecture:** why-partition → key-range/hash tradeoff → composite keys → the structural-conflict case (independently surfaced) — one continuous chain from "why divide data at all" to "why one scheme can't serve every query."

**Forward links (Lecture 5):**
- **Secondary indexes under partitioning** — the actual resolution technique for the "aggregate across all entities" conflict surfaced this lecture.
- **Rebalancing** — the mass-reshuffle cost of naive `mod N` hashing vs. the append-friendly growth of time-bucketed partitioning, both raised this lecture.
- **Consistent hashing** — named here, mechanics deferred; the fix that makes hash-based partitioning elastic without mass reshuffle.
- **Request routing** — how a client/query finds the right partition once data is split.

**Real systems:** Cassandra/DynamoDB (hash + composite/clustering keys, exactly the sensor-ID + timestamp pattern derived this lecture); HBase/BigTable (key-range, deliberately choosing range-query efficiency); time-series DBs (InfluxDB, TimescaleDB — time-bucketed top-level partitioning for append-friendly growth).

---

## 10. Flashcards

**Q:** Why doesn't adding more replicas solve a write-throughput scaling problem?
**A:** Every replica remains a full copy of the dataset — it must eventually apply every write, so total system-wide write-processing work scales with replica count rather than being divided among nodes.

**Q:** What's the core difference between what replication and partitioning each solve?
**A:** Replication multiplies copies of the whole dataset (reliability, read-scaling). Partitioning divides the dataset itself — the only technique that reduces per-node storage and processing load.

**Q:** Key-range partitioning's main strength and main risk?
**A:** Strength: cheap contiguous range queries. Risk: skewed key distributions (e.g., timestamp keys) create a hot partition absorbing all current writes.

**Q:** Hash-based partitioning's main strength and main cost?
**A:** Strength: near-uniform write distribution, no hot spots. Cost: destroys key locality — range queries must scatter-gather across every partition.

**Q:** Give an example of a composite partition key and what it resolves.
**A:** `hash(sensor_id)` for partition assignment + timestamp-sorted within partition — resolves "one sensor's readings over a time range" while still avoiding write hot spots.

**Q:** Why can't a smarter partition key resolve every query pattern?
**A:** Some access patterns organize data along fundamentally different axes (e.g., "spread by entity" vs. "co-locate by moment") — satisfying one structurally destroys the other. Requires a second structure (secondary index), not a better key.

**Q:** Does adding entities increase partition count?
**A:** No — partition count (N) is fixed for a given capacity plan; more keys just hash into the same N partitions. Partition count grows only via deliberate resharding for capacity reasons.

**Q:** State the master heuristic for choosing a partitioning strategy.
**A:** Find the actual access pattern (query shape and write-skew risk) first, then let it determine the strategy — the same "requirements before design" move as finding the invariant first for consistency models.

---

# Homework

### Core
1. **(Conceptual)** In your own words: why does multi-leader replication fail to divide write-processing work the way partitioning does, even though it spreads write *acceptance* across multiple nodes?
2. **(Design)** A ride-sharing app partitions trip records by `hash(driver_id)`. Name one query pattern this serves well and one it serves poorly. For the poorly-served one, propose a fix (composite key, secondary structure, or otherwise) and justify it.
3. **(Reading)** *Designing Data-Intensive Applications*, 2nd ed., **Chapter 7 — Partitioning**: read the sections on partitioning strategies (key-range and hash-based) and partitioning combined with replication. Note one detail from the book that sharpens or corrects something from this lecture.

### Advanced
4. **(Trace)** Using the IoT sensor example from this lecture: sketch what a query for "average temperature across all 10,000 sensors at T1" would actually require at the system level, given `hash(sensor_id)` partitioning. Be concrete about what "touch every partition" costs at scale (network round trips, aggregation step).
5. **(Design)** Propose a partitioning scheme for a system with two genuinely conflicting dominant queries: "get all orders for customer X" (point lookup, needs to be fast) and "get all orders placed in the last hour across all customers" (needs to be fast). Name the tension explicitly and propose your resolution.

### Challenge
6. **(Synthesis)** You raised the "10,000 new shards" concern and I corrected the mechanics but validated the underlying instinct. Now go deeper: research **consistent hashing** (not yet taught) and explain in 4-5 sentences, in your own words, how it avoids the mass-reshuffle problem that naive `hash(key) mod N` has when N changes. This previews Lecture 5.

### Reflection
7. You explicitly named the course's throughline this lecture ("we predict scenarios and design systems, not design scenarios and predict systems"). Where else, looking back across Lectures 1-4, can you see this same move — find the real constraint before picking the tool — playing out, even where it wasn't named at the time?

**Next session opens with:** Lecture 5 — Skew, Hot Spots, Secondary Indexes, and Rebalancing — **starting only after your explicit go-ahead.**
