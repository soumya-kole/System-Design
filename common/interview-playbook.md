# Interview Playbook

How to run a 45-minute system design discussion. The concept folders in this repo are
the worked examples: each `requirement.md` is minutes 0–10 written out, each
`back-of-the-envelope.md` is minutes 10–15, and each `design.md` is the rest.

## The clock

| Minutes | Do | Output |
| ------- | -- | ------ |
| 0–10 | Clarify requirements | 5–8 numbered functional requirements, a table of non-functional targets with numbers, and an explicit out-of-scope list |
| 10–15 | Back-of-the-envelope | QPS, storage, bandwidth, node count. Each with the arithmetic said aloud |
| 15–25 | High-level design | One diagram with every box justified by a requirement. Name the 2–3 decisions everything else follows from |
| 25–38 | Deep dive | Data model, the hot-path algorithm, the one hard problem (usually consistency or the failure case) |
| 38–43 | Failure modes | Node down, partition, overload, hot key, bad deploy. What breaks, what the design does about it |
| 43–45 | Trade-offs | What was given up and why. Where the design would change if one requirement changed |

The most common failure is spending 20 minutes on the high-level diagram and never
reaching the deep dive. The diagram is the cheap part; the deep dive is what is being
graded.

## Minutes 0–10: questions that change the design

Ask only questions whose answer would change what gets built. Each one below has a
different design on each side of the answer.

| Question | Why it matters |
| -------- | -------------- |
| What are the operations, exactly? Single-key or multi-key? Scans? | Multi-key or range access rules out plain hash partitioning and most leaderless designs |
| How much data, how many entities, how big is each? | Decides one machine vs. a cluster, and whether the working set fits in memory |
| Reads vs. writes, and peak vs. average? | Read-heavy gets caches and replicas; write-heavy gets LSM and partitioning |
| What is the latency budget, measured where? | Sets how many network hops the hot path can afford. 10 ms allows one; 100 ms allows a few |
| When two things disagree, who wins? When a node is down, do we refuse or accept? | CP vs. AP, and it must be asked as a concrete scenario, not "strong or eventual?" |
| What has to survive what: node, disk, zone, region? | Sets replication factor and placement. Region survival is a different design from zone survival |
| Where does it sit relative to what already exists? | Gateway vs. library vs. sidecar vs. separate tier. Changes who pays the latency |
| What does the client see on failure or rejection? | Headers, error codes, retry contract. Cheap to specify, expensive to retrofit |
| How does it grow, and how often? | Online rebalancing is a requirement if growth is routine and a project if it is rare |
| What is explicitly not needed? | The out-of-scope list is the most useful thing produced in this phase |

End the phase by reading the finalized list back. An interviewer who hears "so the
contract is: these eight things, these numbers, and not those four" knows the rest of the
hour is anchored.

## Minutes 10–15: estimation

Show inputs, show arithmetic, round hard, name the constraint that binds. See
[rate-limiter/back-of-the-envelope.md](../rate-limiter/back-of-the-envelope.md) and
[key-value-store/back-of-the-envelope.md](../key-value-store/back-of-the-envelope.md)
for the full shape; the order that works:

1. **Traffic** — per day ÷ 86,400 ≈ per second. Then × peak factor, then round up for
   headroom. 1M/day ≈ 12/s; 100M/day ≈ 1.2k/s; 10B/day ≈ 116k/s.
2. **Fan-out** — what the cluster does per client request: replicas written, rules
   evaluated, shards queried. This is usually 2–5× the client number.
3. **Storage** — entities × bytes per entity × replication × engine overhead × growth.
4. **Node count** — from throughput, from storage, and from anything else physical
   (SSD endurance, NIC). The largest wins; say which.
5. **Bandwidth** — QPS × payload. Almost never binds, but say so with the number.
6. **Memory** — indexes, filters, hot working set. Decides the instance size.
7. **Latency budget** — a table of hops on the hot path summing to the target. This is
   the estimate that most often kills a design: if the sum exceeds the budget, an
   extra tier or a cross-region call has to go.

Numbers worth knowing cold:

| Thing | Number |
| ----- | ------ |
| Same-AZ round trip | ~0.5 ms |
| Cross-AZ round trip | ~1–2 ms |
| Cross-region round trip | 60–150 ms |
| NVMe random read | ~100 µs, ~500k IOPS |
| NVMe sequential write | ~2 GB/s |
| SSD endurance, enterprise | ~1–3 drive-writes per day |
| Redis, simple command | ~100k ops/s per core-ish node |
| LSM node, mixed workload | ~50k ops/s comfortably |
| Bloom filter | ~10 bits/key for ~1% false positives |
| Seconds per day | 86,400 ≈ 10⁵ |
| 1 M/day, 1 M/hour | ≈ 12/s, ≈ 280/s |

## Minutes 15–25: high-level design

One diagram, drawn left to right along the request path. Then say the sentence: "This
design reduces to N decisions: ___, ___, and ___." Everything after should be a
consequence of those. For a rate limiter: what state per key, how to mutate it
atomically, what to do when the store is down. For a KV store: how keys map to nodes,
how replicas agree, what engine sits on each node.

Prefer a comparison table over prose whenever there are three or more options, with a
**Pick** line under it that names the choice and the one reason.

## Minutes 25–38: deep dive

Pick the hardest part, not the largest. Signs of the right choice: it is where the
non-functional targets are at risk, it is where a naive implementation is subtly wrong
(a read-modify-write race, a tombstone that gets dropped early, a quorum that stops
overlapping), and it produces something concrete (a key schema, a script, a state
machine, a sequence diagram).

Questions interviewers reliably ask, and the shape of a good answer:

| Question | Answer shape |
| -------- | ------------ |
| How do you find where a key lives? | hash → owner → replica list, and who holds the map (client, gateway, or a service) |
| What happens when a node dies? | detect → keep serving from replicas → route around (hints / failover) → rebuild → catch up. Say which steps are automatic |
| How do you avoid losing data? | durable log before ack, N copies in different failure domains, quorum on ack, and backups for the case replication cannot cover |
| How do you handle deletes? | a tombstone that replicates, a grace period longer than the repair cycle, and what resurrects data if that invariant breaks |
| How do you handle a hot key? | cheapest first: coalesce → cache → serve locally → replicate wider. Name the one you would ship first |
| How do you add capacity? | what moves, from where, how fast, and what the client sees while it happens |
| What happens under a partition? | which side keeps accepting writes, what diverges, how it reconciles, and what the client is promised meanwhile |
| Why not X? | one sentence, tied to a requirement. "Leveled compaction would need twice the nodes for SSD endurance" beats "LSM is standard" |

## Minutes 38–45: failure modes and trade-offs

Walk the list: one node, one zone, network partition, store unavailable, write overload,
hot key, clock skew, bad deploy. For each: what breaks, what the design does, what the
client sees. Then the trade-offs, framed as what changes if a requirement changes:
"if we needed compare-and-set, the leaderless replication goes and Raft per shard comes
in, and write latency roughly doubles."

## Habits that separate a strong round from a mediocre one

- **Every number has its arithmetic.** "About 115k QPS" is fine only after "2 billion a
  day over 86,400 seconds is 23k, times a 5× peak."
- **Every box on the diagram is there because of a requirement.** If it is not, remove
  it. A load balancer in front of a system with client-side routing is a tell.
- **Name what was given up.** Sloppy quorum gives up the overlap guarantee. LWW gives up
  concurrent writes. A local fallback bucket gives up accuracy. Say it before being asked.
- **Prefer the boring choice and know why it is boring.** Consistent hashing, LSM, quorum
  replication, token bucket. Novelty is not graded; understanding is.
- **Tie percentiles to mechanisms.** p99 is the slower of two replicas; p99.9 is
  compaction and GC. Averages hide both.
- **Say "out of scope" out loud, with a reason.** Cutting scope with a stated reason is
  design; cutting it silently is a gap.

## Related

- [rate-limiter](../rate-limiter/design.md) and
  [key-value-store](../key-value-store/design.md) — full worked rounds in the three-file
  format.
- [references.md](references.md) — shared reading list.
