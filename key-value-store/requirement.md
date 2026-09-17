# Key-Value Store — Requirements Gathering

The first ten minutes of the design. Nothing here is architecture; it is the set of
questions that decide *which* architecture is correct. The finalized requirements at the
bottom are the contract the [design](design.md) is built against, and the inputs to the
[sizing](back-of-the-envelope.md).

---

## Q1 — What is the interface?

> **Candidate:** "Key-value store" covers everything from Redis to DynamoDB to etcd. Before
> anything else: what operations, what do keys and values look like, and what *don't* we
> need? Range scans, secondary indexes, and transactions each change the design completely.

> **Interviewer:** Three operations: `get(key)`, `put(key, value)`, `delete(key)`. Keys are
> opaque strings up to **256 bytes**. Values are opaque bytes — we don't parse them — up to
> **1 MB**, though the median is around **10 KB**. A `put` overwrites; there's no append.
>
> No range scans, no secondary indexes, no multi-key transactions. Each operation touches
> exactly one key. If a team needs a scan they should be on a different system.

> **Candidate:** Two follow-ups. Do keys expire, and does a `put` need to be conditional —
> "write only if the value is still X"?

> **Interviewer:** **TTL, yes** — a lot of our use is session and cache-like data, and
> leaving expiry to clients means it never happens. Conditional put, **no**. Compare-and-set
> across replicas requires a consensus round per write, and that would eat the latency
> budget I'm about to give you. Single-key `put` semantics only; if it turns out we need
> CAS, that's a separate design.

**Pins down:** `get` / `put` / `delete` on a single opaque key, keys ≤ 256 B, values ≤ 1 MB
(~10 KB typical), per-key TTL. No scans, indexes, transactions, or conditional writes.

---

## Q2 — How much data and how much traffic?

> **Candidate:** Volume decides whether this is one big machine or a cluster. How many keys,
> how big is the dataset, and what's the read/write mix?

> **Interviewer:** About **1 billion keys** today, growing to roughly **2 billion within two
> years**. At ~10 KB a value that's **10 TB of raw data now, 20 TB planned**.
>
> Traffic: **10 billion reads and 2 billion writes a day**, so 5:1 read-heavy. Diurnal,
> with **peak around 3× the daily average**. Access is skewed — a small fraction of keys
> takes most of the reads — but don't count on that for correctness; it's a caching
> opportunity, not a design assumption.

> **Candidate:** And latency?

> **Interviewer:** Measured at the server: **reads p99 ≤ 10 ms, writes p99 ≤ 20 ms**. Ten
> milliseconds means a read cannot make more than one or two network hops inside the
> cluster, and it certainly cannot cross regions.

> **Candidate:** Let me sanity-check the load.
>
> ```
> Reads   10×10⁹ / 86,400 s   ≈ 116k/s average → ~350k/s peak
> Writes   2×10⁹ / 86,400 s   ≈  23k/s average →  ~70k/s peak
> ```
>
> Twenty terabytes and hundreds of thousands of ops per second is well past one machine,
> so this is a partitioned cluster from day one. And at 10 KB a value the network is going
> to matter: 350k reads/s is 3.5 GB/s leaving the cluster.

**Pins down:** 10 TB raw today, 20 TB planned; ~350k reads/s and ~70k writes/s at peak,
sized with headroom; p99 read ≤ 10 ms, write ≤ 20 ms. Partitioned cluster, not a single
node.

---

## Q3 — Consistency or availability?

> **Candidate:** The fork in the road. When a replica is unreachable, do we refuse the write
> to keep every replica in agreement, or accept it and reconcile later? And after a write
> is acknowledged, does the next read have to see it?

> **Interviewer:** **Availability.** This store is behind user-facing paths — sessions,
> profiles, carts. A write that fails because one replica is slow is a worse outcome than a
> read that's briefly stale. The store must stay **writable through any single node or
> availability-zone failure**.
>
> But "eventually consistent" is not a blank check. Two things I hold you to. First,
> **consistency must be tunable per request**: a caller who needs read-your-writes should be
> able to pay for it with a stronger read, and a caller who wants speed should be able to
> read from one replica. Second, **a write must never be silently lost** once we've
> acknowledged it. If two writers race, one value wins and that's fine — but it wins by a
> rule the client can understand, not by luck.

> **Candidate:** So for concurrent writes to the same key: last-writer-wins by timestamp, or
> keep both versions and let the application merge?

> **Interviewer:** Default to **last-writer-wins** — the vast majority of our keys have a
> single logical writer, and returning multiple versions to every caller is a burden they
> won't carry. But the timestamp has to be something better than each node's wall clock,
> because clock skew turns "last" into "random." If you want to offer application-level
> merge as an opt-in for the few keys that genuinely have concurrent writers, I'd like to
> see how, but LWW is the contract.

**Pins down:** AP system. Writable through single-node and single-AZ failure. Tunable
consistency per request with a strong-read option. Acknowledged writes are durable. LWW
conflict resolution with skew-resistant timestamps; sibling-return as an optional mode.

---

## Q4 — What is the failure model?

> **Candidate:** What has to survive what? Node loss, disk loss, zone loss, region loss —
> and what does an acknowledgement promise?

> **Interviewer:** **Single region, three availability zones.** Losing any one node, any
> one disk, or an entire zone must lose **no acknowledged data** and must not take the
> store below its latency targets by more than a small margin. Multi-region is **out of
> scope** — if it comes, it comes as asynchronous replication on top of this.
>
> Concretely, that means an acknowledged write has reached durable storage on **at least
> two nodes in different zones** before we say yes. A single copy on one node's disk is not
> durable in my book; a single copy in memory certainly isn't.

> **Candidate:** Node churn — how often do nodes die, and are they replaced or repaired?

> **Interviewer:** Assume **a node fails every few days** somewhere in the fleet, mostly
> disk. It gets **replaced, not repaired** — a fresh node with an empty disk joins and has to
> rebuild what the dead one held. Design for that to happen while serving traffic, and
> without an operator hand-holding it.

> **Candidate:** Replication protects against hardware. It does nothing against a bad
> deploy that deletes a million keys, because the delete replicates perfectly. Do we need
> backups?

> **Interviewer:** Yes, and thank you for separating the two. **Periodic snapshots to
> object storage**, off the serving cluster. Targets: **RPO 6 hours, RTO 4 hours** for a
> full restore. Also be able to restore a single key range without rebuilding everything —
> most incidents are one application's keys, not the whole store.

**Pins down:** replication factor 3 across 3 zones in one region. Writes acknowledged
only after 2 zone-distinct durable copies. Node replacement from an empty disk is routine
and automated. Snapshots to object storage with RPO 6 h / RTO 4 h, restorable per key
range. Multi-region out of scope.

---

## Q5 — How does it grow, and what goes wrong operationally?

> **Candidate:** Two operational questions. How do we add capacity, and how do we handle
> skew — hot keys and oversized values?

> **Interviewer:** **Adding a node must be online.** No maintenance window, no client
> reconfiguration, and moving data onto the new node shouldn't cost more than about **5%**
> of the cluster's throughput while it runs. Same for removing one. We'll grow the cluster
> roughly every quarter, so this is a routine operation, not an event.
>
> On skew: **hot keys are real** and mostly read-hot — a viral item, a shared config key
> read by every server on boot. One key must not be able to take down the nodes that hold
> it. Write-hot keys are rarer and we can push back on the application for those.
>
> Values: **reject anything over 1 MB** at the API. Don't build chunking; a client that
> needs a 50 MB blob should be using object storage and putting the pointer here.
>
> And one more: when writes spike past what the disks can absorb — a bulk import, a
> misbehaving batch job — **degrade, don't collapse**. Slow the writers down and keep
> serving reads. A store that falls over at 5× its normal write rate is worse than one
> that returns "try later."

> **Candidate:** Last one — who operates this, and what do they need to see? And is this
> one team's store or a shared platform with tenants?

> **Interviewer:** A small platform team. They need to see per-node load, ring balance,
> replication lag, and pending repair work. Every automated action — rebalance, replacement,
> repair — should be visible and pausable, but should not *require* them.
>
> Single trust domain for now: internal callers, authenticated at the network boundary,
> namespaced by key prefix. Per-tenant quotas, per-key authorization, and encryption at
> rest are real needs but **out of scope** for this round.

**Pins down:** online add/remove of nodes with ≤ 5% throughput impact. Read-hot keys
must not overload their replicas. Hard 1 MB value cap. Write overload triggers
backpressure, not failure. Operator visibility into ring balance, repair backlog, and
replication state. Multi-tenancy and per-key security deferred.

---

## Finalized requirements

### Functional

1. `get(key)`, `put(key, value, ttl?)`, `delete(key)`; single-key operations only.
2. Keys are opaque strings ≤ 256 B; values are opaque bytes ≤ 1 MB, rejected above that.
3. Optional per-key TTL; expired keys are unreadable and their space is reclaimed.
4. Per-request consistency level: at minimum *one replica* and *quorum*, with quorum
   reads seeing all quorum-acknowledged writes.
5. Concurrent writes to one key resolve deterministically (last-writer-wins on a
   skew-resistant timestamp); optional mode returns conflicting siblings to the client.
6. Nodes can be added and removed online with automatic data movement.
7. A replacement node rebuilds its data from replicas without operator intervention.
8. Periodic snapshots to object storage; restore a full cluster or a single key range.
9. Under write overload, apply backpressure to writers and keep serving reads.
10. Expose per-node load, ring ownership, replication lag, and repair backlog.

### Non-functional

| Requirement | Target |
| ----------- | ------ |
| Dataset | 10 TB raw today, 20 TB planned (1B → 2B keys, ~10 KB median value) |
| Read throughput | ~350k/s peak, sized with headroom |
| Write throughput | ~70k/s peak, sized with headroom |
| Read latency | p99 ≤ 10 ms, server-side |
| Write latency | p99 ≤ 20 ms, server-side |
| Availability | Writable through any single-node or single-AZ failure |
| Durability | No acknowledged write lost on single-node, single-disk, or single-AZ failure; ack after 2 zone-distinct durable copies |
| Backup | RPO ≤ 6 h and RTO ≤ 4 h against logical corruption (bad deploy, operator error) |
| Rebalancing cost | ≤ 5% throughput impact while adding/removing a node |
| Hot keys | One read-hot key must not saturate its replicas |

### Placement

- One region, three availability zones; replicas of every key spread across all three.
- Clients talk to the cluster through a partition-aware library or a thin proxy; no
  external coordinator or load balancer needs to know the ring.

### Explicitly out of scope

- **Range scans, secondary indexes, multi-key transactions** — a different storage
  system.
- **Conditional writes / compare-and-set** — requires per-write consensus; separate
  design if needed.
- **Multi-region replication** — layered on later as asynchronous replication.
- **Values over 1 MB** — object storage plus a pointer.
- **Multi-tenancy, per-key authorization, encryption at rest** — single internal trust
  domain for this round; these would live in an API tier in front of the ring.

---

**Next:** [back-of-the-envelope.md](back-of-the-envelope.md) — sizing from these numbers,
then [design.md](design.md) — the design built against them.
