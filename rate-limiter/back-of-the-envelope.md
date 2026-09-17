# Rate Limiter — Back-of-the-Envelope

Sizing the design against the numbers pinned down in [requirement.md](requirement.md).
Every figure below shows its inputs; change an input and the rest should be re-derived,
not trusted.

**Inputs (from requirements)**

| Input | Value |
| ----- | ----- |
| API calls per day | 2 × 10⁹ |
| Peak-to-trough ratio | 5× |
| Occasional batch spikes | 10× |
| Registered users | 100M |
| Distinct keys active in a 10-minute window | ~10M |
| Gateway nodes | 40, across 3 regions |
| Latency budget for the limiter | ≤ 5 ms p99, 10 ms ceiling |

---

## 1. Traffic

```
Average   2 × 10⁹ req/day ÷ 86,400 s     ≈ 23k QPS
Peak      23k × 5                        ≈ 115k QPS
Design    115k + 40%/yr growth + spikes  → round to 200k QPS
```

Every request costs at least one state read-modify-write, so **the store must absorb the
full 200k ops/s**. There is no read-only fast path: even an allowed request mutates the
counter.

**Rule fan-out.** A request may match several rules (identity, tenant, IP, endpoint
class). Evaluating them in one Lua script keeps it at one round trip, but the script does
up to 4 key operations:

```
200k req/s × ~2.5 rules avg   ≈ 500k key-ops/s inside Redis
```

This is why "one round trip per request" and "one key op per request" are different
numbers. Redis's ~100k *commands*/s per node figure is for simple commands; a Lua script
touching 2–4 keys is one command but proportionally more CPU.

## 2. Per-region and per-node load

Traffic is not evenly split across regions. Assume a 50 / 30 / 20 split:

```
Largest region at design peak   200k × 0.5   ≈ 100k QPS
Smallest region                 200k × 0.2   ≈  40k QPS
Per gateway node (40 nodes)     200k ÷ 40    ≈   5k QPS average
                                              ~ 10k QPS if a node takes 2× its share
```

5–10k QPS of extra work per gateway node is trivial for the gateway itself; the cost is
the Redis round trip on each, not the CPU.

## 3. State store throughput

Per-node Redis budget, being conservative because the workload is Lua scripts rather
than bare `INCR`:

```
Assume ~50k script-executions/s per Redis primary (half the simple-command figure)
Largest region    100k ÷ 50k   = 2 primaries at 100% → 3 primaries at ~67%
                                 add 1 for N+1        → 4 primaries
Smaller regions   3 primaries each (N+1 over a minimum of 2)
```

| Region | Peak QPS | Primaries | Replicas | Total nodes |
| ------ | -------- | --------- | -------- | ----------- |
| A (50%) | 100k | 4 | 4 | 8 |
| B (30%) | 60k | 3 | 3 | 6 |
| C (20%) | 40k | 3 | 3 | 6 |
| **Total** | 200k | 10 | 10 | **20** |

Twenty modest Redis instances for the whole fleet. Throughput, not memory, is what sets
the node count.

## 4. Storage

```
Per key      ~60 B value (tokens + timestamp as a hash)
           + ~40 B key string (rl:tenant:acme:api_write)
           + ~50 B Redis overhead (dict entry, expiry)   ≈ 150 B

Active keys  10M identities × ~2.5 rules each           ≈ 25M keys
Working set  25M × 150 B                                ≈ 3.75 GB  fleet-wide
Largest region (50%)                                    ≈ 1.9 GB
Per primary in region A (÷ 4)                           ≈ 0.5 GB
```

Fits in memory on any instance. Compare the alternative that does **not** fit:

```
Sliding window log, 10,000/hour limit
  10,000 timestamps × 8 B × 25M keys                    ≈ 2 TB
```

That gap is the entire argument for token bucket / sliding window counter over a
timestamp log at this cardinality.

**Without TTL** the key count is bounded by *registered* users × rules, not active ones:

```
100M × 2.5 × 150 B                                      ≈ 37 GB, and growing
```

TTL on every key turns a growing dataset into a stable one — the requirement, not an
optimization.

## 5. Bandwidth to the store

```
Request   script SHA + 1 key + ~4 args                   ≈ 150 B
Response  3 integers                                     ≈  50 B
Per request round trip                                   ≈ 200 B

200k QPS × 200 B                                         ≈ 40 MB/s ≈ 320 Mbit/s fleet-wide
Largest region                                           ≈ 160 Mbit/s
```

Negligible on any modern network. Bandwidth is never the constraint here.

## 6. Latency budget

Where the 5 ms goes on the hot path, assuming gateway and Redis are in the same
availability zone:

| Step | Typical | Notes |
| ---- | ------- | ----- |
| Identity extraction | ~0.1 ms | Parse header / decode JWT locally — no network |
| Rule match | ~0.01 ms | In-memory config lookup |
| Network RTT to Redis | 0.3–1 ms | Same-AZ; cross-AZ adds ~1–2 ms |
| Lua script execution | 0.05–0.2 ms | 2–4 key ops, single-threaded |
| Redis queueing under load | 0–2 ms | The variable term; grows with node utilization |
| Header assembly | ~0.01 ms | |
| **Total** | **~0.5–3.5 ms** | Inside the 5 ms p99 budget with headroom |

The queueing term is why Redis nodes are sized to ~67% utilization rather than 100%.
Latency at a saturated single-threaded server goes vertical, and this design has no
second chance — a 10 ms timeout trips the local fallback.

**Cross-region is out of budget by itself:** 60–150 ms RTT between regions exceeds the
entire 10 ms ceiling, which is the quantitative reason the requirements rule out global
coordination.

## 7. Local fallback state

Each gateway node keeps an in-memory bucket per key it has seen recently, used only when
Redis is unreachable:

```
Keys per node (10M active ÷ 40 nodes, × 2 for uneven routing)   ≈ 500k
Per key in a process hash map                                    ≈ 100 B
Per node                                                         ≈ 50 MB
```

Cheap enough to keep warm at all times rather than build on first failure.

## 8. Telemetry volume

Emitting a usage event per request is unnecessary and expensive; aggregate per key per
window on the gateway and flush periodically:

```
Per request          200k events/s × ~100 B                ≈ 20 MB/s   ← don't do this
Per key per 10 s     25M keys ÷ 10 s                       ≈ 2.5M events/s worst case
                     but only keys with traffic in the window flush,
                     and most keys are quiet → assume 10%   ≈ 250k events/s
Per key per 60 s     same, ÷ 6                             ≈ 40k events/s
```

A 60-second aggregation window gives customer dashboards minute-granularity usage at
~40k events/s fleet-wide — comfortable for any stream pipeline. Approaching-limit alerts
work on the same stream.

---

## Summary

| Quantity | Value | Set by |
| -------- | ----- | ------ |
| Design QPS | 200k | traffic + growth + spikes |
| Redis ops/s (key-level) | ~500k | rule fan-out |
| Redis nodes | 20 (10 primaries + 10 replicas) | throughput, N+1 per region |
| Working set | ~3.75 GB fleet-wide | active keys × 150 B |
| Store bandwidth | ~40 MB/s | 200 B per round trip |
| Hot-path latency | ~0.5–3.5 ms | same-AZ RTT + queueing |
| Local fallback per gateway | ~50 MB | 500k keys × 100 B |
| Telemetry | ~40k events/s | 60 s aggregation |

The constraints, in order: **Redis throughput** (sets node count), **latency headroom**
(sets utilization target), then nothing else comes close. Memory, bandwidth, and
telemetry are all an order of magnitude under any limit.

---

**Previous:** [requirement.md](requirement.md) · **Next:** [design.md](design.md)
