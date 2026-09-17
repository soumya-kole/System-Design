# Key-Value Store — Back-of-the-Envelope

Sizing the design against the numbers pinned down in [requirement.md](requirement.md).
Every figure shows its inputs; change an input and re-derive the rest.

**Inputs (from requirements)**

| Input | Value |
| ----- | ----- |
| Keys | 1B today, 2B planned |
| Median value | 10 KB (key ≤ 256 B, value ≤ 1 MB) |
| Raw dataset | 10 TB today, 20 TB planned |
| Reads per day | 10 × 10⁹ |
| Writes per day | 2 × 10⁹ |
| Peak-to-average | 3× |
| Replication factor | N = 3, one replica per AZ, 3 AZs |
| Latency | read p99 ≤ 10 ms, write p99 ≤ 20 ms |

**Design choices the math depends on** (justified in [design.md](design.md)): quorum
reads and writes with R = W = 2; a log-structured merge (LSM) storage engine with
size-tiered compaction; partition-aware clients that send requests straight to a replica.

---

## 1. Traffic

```
Reads    10×10⁹ ÷ 86,400 s   ≈ 116k/s avg   × 3 peak ≈ 350k/s   → design 500k/s
Writes    2×10⁹ ÷ 86,400 s   ≈  23k/s avg   × 3 peak ≈  70k/s   → design 100k/s
```

Design targets round up ~40% over measured peak to cover growth and the occasional
batch job.

## 2. Replica-level load

A client request fans out to replicas, so the cluster does more work than the client
sees.

```
Writes   each put goes to all N = 3 replicas          100k × 3   = 300k replica writes/s
Reads    coordinator is itself a replica; reads its own copy and asks
         R − 1 = 1 other replica for a digest (hash)  500k × 2   = 1M replica lookups/s
```

A digest read still does the full local lookup — it just returns a hash instead of
10 KB — so it costs the same CPU and disk, only less network.

## 3. Storage

```
Per key on disk    10 KB value + 256 B key + ~50 B metadata (timestamp, TTL, flags)  ≈ 10.3 KB
Raw planned        2B keys × 10.3 KB                                                 ≈ 20 TB
Replicated         20 TB × 3                                                         = 60 TB
```

Size-tiered compaction can transiently hold two copies of a range while merging, so
provision **2× the replicated size**:

```
Provisioned        60 TB × 2                                                         = 120 TB
```

## 4. Node count

Three candidate constraints; the largest wins.

**By storage.** With 2 × 3.84 TB NVMe per node (~7.5 TB usable):

```
120 TB ÷ 7.5 TB   ≈ 16 nodes
```

**By throughput.** A well-tuned LSM node serves ~50k mixed ops/s comfortably:

```
(1M lookups + 300k writes) ÷ 50k   ≈ 26 nodes
```

**By SSD write endurance.** Compaction rewrites every byte several times. This is the
one people miss:

```
Ingest per day        2×10⁹ writes × 3 replicas × 10.3 KB               ≈ 62 TB/day fleet-wide
Write amplification   size-tiered ≈ 4×  (leveled would be ≈ 10×)
Disk writes per day   62 TB × 4                                          ≈ 250 TB/day fleet-wide
Endurance budget      7.68 TB per node × 1 DWPD (drive-writes-per-day)   = 7.68 TB/day/node

Nodes needed          250 ÷ 7.68                                         ≈ 33 nodes
```

Leveled compaction would need ~80 nodes just to stay inside drive endurance, which is
why this design uses size-tiered despite its worse space amplification.

**Pick: 36 nodes, 12 per AZ.** Multiple of 3 for AZ symmetry, ~10% over the endurance
floor. Each node: 2 × 3.84 TB NVMe, 64 GB RAM, 25 GbE.

## 5. Per-node load at design peak

```
Lookups     1M ÷ 36              ≈ 28k/s
Writes      300k ÷ 36            ≈ 8.3k/s   ≈ 85 MB/s ingest
Data held   60 TB ÷ 36           ≈ 1.7 TB   (3.3 TB with compaction headroom, of 7.5 TB)
Keys held   2B × 3 ÷ 36          ≈ 170M
```

Storage per node is under half the disk. **The node count is set by write endurance and
throughput, not by capacity** — the disks are mostly empty.

## 6. Disk I/O per node

```
Reads   worst case, every lookup misses cache:  28k/s × 10 KB   ≈ 280 MB/s, 28k IOPS
        NVMe random read: ~500k IOPS, ~3 GB/s              → fine even with 0% cache hit

Writes  WAL append       85 MB/s sequential
        SSTable flush    85 MB/s sequential
        compaction       85 MB/s × (4 − 1) rewrites        ≈ 255 MB/s
        total            ≈ 425 MB/s sequential             → fine for NVMe (~2 GB/s)
```

Peak write bandwidth is fine. The daily *total* is what threatens the drive, per §4.

## 7. Network bandwidth

Fleet-wide at design peak, assuming partition-aware clients hit a replica directly 80%
of the time:

```
Client read egress     500k × 10 KB                              ≈ 5.0 GB/s
Client write ingress   100k × 10 KB                              ≈ 1.0 GB/s
Write replication      100k × 10 KB × 2 extra replicas           ≈ 2.0 GB/s
Read digests           500k × ~64 B                              ≈ negligible
Mis-routed reads       20% × 5 GB/s forwarded to a replica       ≈ 1.0 GB/s
Total                                                            ≈ 9 GB/s ≈ 72 Gbit/s

Per node               72 ÷ 36                                   ≈ 2 Gbit/s
```

Two gigabits per node against a 25 GbE NIC. Bandwidth is not a constraint, and there is
room for repair streaming on top.

## 8. Memory per node

```
Memtables       2 active + 2 flushing × 256 MB                     ≈ 1 GB
Bloom filters   170M keys × 10 bits                                ≈ 210 MB
Index summary   sampled 1 per 128 keys × 170M × ~40 B              ≈ 55 MB
Block cache     hot 1% of data: 1% × 60 TB ÷ 36                    ≈ 17 GB
OS / heap / page cache slack                                       ≈ 10 GB
Total                                                              ≈ 28 GB   → 64 GB node is comfortable
```

If 1% of keys serve ~80% of reads, a 17 GB block cache turns 28k lookups/s into ~6k disk
reads/s. Disk could take all 28k (§6), so the cache buys latency, not survival.

## 9. Latency budget

**Read at QUORUM (R = 2), client is partition-aware:**

| Step | Typical |
| ---- | ------- |
| Client → replica-coordinator (may cross AZ) | 0.3–1 ms |
| Local lookup: memtable, bloom filters, block cache hit | ~0.1 ms |
| Local lookup: cache miss, 1–2 SSTable reads on NVMe | 0.2–0.5 ms |
| Parallel digest from second replica (cross-AZ RTT + its lookup) | 1–1.5 ms |
| Wait for slowest of the two | dominated by the above |
| Response to client | 0.3–1 ms |
| **Total** | **~2 ms p50, ~6–8 ms p99** |

Tail comes from the slower of two replicas plus compaction and GC interference. Inside
the 10 ms budget with ~2 ms to spare, which is not a lot: **no room for an extra hop**,
which is why the coordinator must be a replica.

**Write at QUORUM (W = 2):**

| Step | Typical |
| ---- | ------- |
| Client → coordinator | 0.3–1 ms |
| Coordinator → 3 replicas in parallel (cross-AZ) | 1 ms RTT |
| Each replica: WAL append + group-commit fsync + memtable insert | 0.5–2 ms |
| Wait for 2 of 3 acknowledgements | second-fastest, not slowest |
| Response | 0.3–1 ms |
| **Total** | **~3 ms p50, ~10–15 ms p99** |

Waiting for 2 of 3 rather than all 3 is what keeps the write p99 inside 20 ms: one
slow disk does not show up in the tail.

## 10. Rebalancing and recovery

**Adding node 37:**

```
Data to move       60 TB ÷ 37                                     ≈ 1.6 TB
Throttle           5% of cluster throughput ≈ 5% × 9 GB/s         ≈ 450 MB/s fleet-wide
                   but the receiving node caps at ~200 MB/s to keep serving
Time               1.6 TB ÷ 200 MB/s                              ≈ 8,000 s ≈ 2.2 h
Per source node    with virtual nodes the 1.6 TB comes from all 36 → ~6 MB/s each
```

**Replacing a dead node** is the same volume but streams from 3 different replica sets
in parallel, so ~1 hour is realistic. During that hour the affected ranges have 2 live
replicas, and R = W = 2 still works.

**Hinted handoff while a node is down for 1 hour** (using average, not peak):

```
Writes it missed   23k/s × 3 ÷ 36 = 1.9k/s × 10.3 KB × 3,600 s      ≈ 70 GB
Spread over 35 nodes                                                ≈ 2 GB each
```

Cheap to hold for a few hours; cap hints at 3 hours and fall back to anti-entropy repair
beyond that.

**Merkle tree repair** of one node's ranges: hashing 1.7 TB at ~500 MB/s ≈ 1 hour of
background I/O, once per repair cycle — schedule it inside the tombstone grace period.

## 11. Backups

SSTables are immutable, so a snapshot is a set of hard links and an incremental upload
is just the files created since the last one.

```
Full snapshot      one replica of everything: 60 TB ÷ 3                ≈ 20 TB
Incremental        new SSTable bytes ≈ daily ingest per replica         ≈ 20 TB/day
                   (every write lands in a new file before compaction merges it)
Upload rate        20 TB ÷ 86,400 s ÷ 36 nodes                          ≈ 6.5 MB/s per node
Retention          1 full + 7 days incremental                          ≈ 160 TB in object storage

Restore (RTO)      20 TB pulled by 36 nodes at 200 MB/s each = 7.2 GB/s → ~50 min
                   + replay to 3 replicas via streaming                 → ~2–3 h, inside 4 h
```

Six-hourly incrementals meet the 6 h RPO at negligible per-node bandwidth. Restoring
one key range is the same procedure on the SSTables covering that range only.

---

## Summary

| Quantity | Value | Set by |
| -------- | ----- | ------ |
| Design throughput | 500k reads/s, 100k writes/s | traffic × 3 peak + headroom |
| Replica-level | 1M lookups/s, 300k writes/s | R = W = 2, N = 3 |
| Nodes | 36 (12 per AZ), 2 × 3.84 TB NVMe, 64 GB, 25 GbE | SSD endurance under 4× write amplification |
| Provisioned storage | 120 TB (43% used at plan) | 60 TB replicated × 2 compaction headroom |
| Per node | 28k lookups/s, 8.3k writes/s, 1.7 TB, 170M keys | ÷ 36 |
| Disk writes | ~425 MB/s peak, ~7 TB/day per node | compaction |
| Network | ~2 Gbit/s per node | 10 KB values |
| Memory | ~28 GB of 64 GB | 17 GB block cache for the hot 1% |
| Read latency | ~2 ms p50, 6–8 ms p99 | one cross-AZ RTT + slowest of 2 |
| Write latency | ~3 ms p50, 10–15 ms p99 | fsync + second-fastest of 3 |
| Add a node | ~2 h at ≤ 5% impact | 1.6 TB at 200 MB/s |
| Backups | ~6.5 MB/s per node upload, ~160 TB retained, ~2–3 h full restore | immutable SSTables, 6 h incrementals |

The binding constraint is **SSD write endurance under compaction**, which sets both the
node count and the choice of size-tiered over leveled compaction. Throughput is close
behind. Raw capacity, network, and memory are all comfortably under limit.

---

**Previous:** [requirement.md](requirement.md) · **Next:** [design.md](design.md)
