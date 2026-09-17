# Rate Limiter — Requirements Gathering

The first ten minutes of the design. Nothing here is architecture; it is the set of
questions that decide *which* architecture is correct. The finalized requirements at the
bottom are the contract the [design](design.md) is built against, and the inputs to the
[sizing](back-of-the-envelope.md).

---

## Q1 — Where does the limiter live?

> **Candidate:** Before I sketch anything, I want to know where this thing runs. Three
> options with very different consequences: in the client SDK, as middleware inside each
> service, or as a separate tier at the edge. Which are we building?

> **Interviewer:** Think out loud about the trade-offs and tell me what you'd pick.

> **Candidate:** Client-side is the cheapest — the request never leaves the device, so we
> spend nothing. But it's unenforceable. Anyone who cares enough to abuse the API will
> strip it out, and it's exactly those callers we're limiting. So client-side is advisory
> at best.
>
> Per-service middleware puts the decision closest to the resource it protects, which is
> nice for endpoint-specific rules. The cost is that every service now carries the limiter
> library, the config, and the connection to the state store — and they drift. A limit
> change becomes a fleet-wide redeploy.
>
> I'd put enforcement in the API gateway tier: one place to configure, one place to
> upgrade, and abusive traffic gets rejected before it touches an application server.

> **Interviewer:** That's where we run it. We have a gateway tier in front of everything —
> assume it exists and that you can add a filter to it. Two refinements I'd want you to
> have caught:
>
> First, **ordering matters**. The limiter has to run after we've cheaply extracted an
> identity from the request, but before any expensive work — no database lookups, no
> downstream calls. If verifying a token requires a network hop, an attacker can make you
> do that hop for free on every rejected request.
>
> Second, **one tier isn't enough**. A volumetric attack — hundreds of thousands of
> requests a second from a botnet — shouldn't reach your gateway at all; that belongs at
> the CDN/L7 edge with much coarser rules. The gateway limiter is for per-identity
> business limits, not for absorbing floods. Two layers, different jobs.
>
> Keep the client SDK backoff too, by the way. It doesn't stop attackers, but it stops
> well-meaning customers from melting themselves, and that's most of your incidents in
> practice.

**Pins down:** enforcement at the gateway, state held externally, config hot-reloadable
without redeploy. Volumetric defense is a separate concern and out of scope here.

---

## Q2 — What is the throttling key?

> **Candidate:** What do we count against — IP, user, API key? These behave very
> differently, especially for unauthenticated traffic.

> **Interviewer:** All of them, and that's the real answer. Let me give you the cases we
> actually have:
>
> The **primary key is authenticated identity** — user ID for our first-party apps, API
> key for programmatic customers. That's the one that maps to a paid plan, so it's the one
> that can be enforced as a contractual limit.
>
> But **pre-auth endpoints have no identity**. Login, signup, password reset, OTP send.
> Those are the ones under constant attack and we have nothing but IP to key on. IP is a
> blunt instrument — an entire office behind one NAT shares a key, and IPv6 clients rotate
> addresses freely — so we accept it only where there's no alternative, and we set those
> limits generously enough that a shared NAT doesn't break a legitimate office.
>
> We also need a **tenant tier above the user tier**. We're B2B; a customer organization
> buys a plan, and their 500 employees share it. A limit that only ever counted individual
> users would let one org consume everything with no single user tripping a limit.
>
> And limits should be **endpoint-class aware**. A read costs us almost nothing. A report
> generation pins a worker for eight seconds. Those cannot share a budget.

> **Candidate:** So a single request may match several rules at once. What wins?

> **Interviewer:** Most restrictive. Evaluate them and the first denial ends it. And
> evaluate cheapest-and-most-likely-to-deny first, so an abusive caller costs you one
> lookup rather than four.

**Pins down:** composable multi-dimensional rules — `{identity | tenant | IP} × {endpoint
class}` — evaluated most-restrictive-wins, short-circuiting on first denial.

---

## Q3 — How much load should this handle?

> **Candidate:** What volume are we designing for, and what's my latency budget?

> **Interviewer:** About **2 billion API calls a day**, growing roughly 40% year over year.
> Traffic is diurnal — peak is about **5× the daily trough** — and we see occasional 10×
> spikes when large customers run overnight batch jobs.
>
> On identities: **100 million registered users**, but only about **10 million distinct
> keys are active in any 10-minute window**. That gap matters for your storage.
>
> Latency: our end-to-end p99 API SLO is **150 ms**. You may spend **5 ms** of it, and I'd
> want a hard ceiling around 10 ms with a fast path that gives up rather than blowing the
> budget.

> **Candidate:** Let me work that through.
>
> ```
> 2×10⁹ / 86,400 s          ≈ 23k QPS average
> Peak at 5×                ≈ 115k QPS
> Headroom for growth + 10× batch spikes → design for ~200k QPS
> ```
>
> Every request is at least one state read-and-write, so the store sees the full 200k
> ops/sec at peak. And since only 10M keys are active at a time, if I expire keys
> aggressively my working set is bounded by activity, not by registered users — order of
> gigabytes rather than tens.

> **Interviewer:** Right, and hold onto that TTL point. It's the difference between a
> working set that stabilizes and one that grows forever.

**Pins down:** design target ~200k QPS peak, ~10M concurrently active keys, ≤5 ms p99
added latency with a 10 ms hard ceiling. TTL on every key is a requirement, not an
optimization.

---

## Q4 — Does this need to work in a distributed environment?

> **Candidate:** How many gateway nodes, and does a limit have to hold globally across all
> of them or only per node?

> **Interviewer:** About **40 gateway nodes across 3 regions**, and this is the question I
> most want you to get right.
>
> Per-node limits are trivially easy and wrong. Divide the limit by 40 and any customer
> whose traffic isn't perfectly balanced gets throttled at a fraction of what they paid
> for. Load balancers don't distribute one customer's traffic evenly.
>
> So limits must be **shared across nodes within a region**. Across regions, no — I don't
> want a synchronous cross-region hop inside a 5 ms budget, and you shouldn't propose one.

> **Candidate:** Then a customer hitting all three regions can get up to 3× their limit.
> Is that acceptable?

> **Interviewer:** Good, that's the trade-off I wanted you to name. In practice traffic is
> geo-pinned, so it's rare — but yes, it's real, and the honest answer is we accept it.
> What I'd hold you to is a stated accuracy contract: **within a region, don't exceed the
> limit by more than about 10%.** Approximate is fine. 2× is not — that's the fixed-window
> boundary bug and it's a real outage.

> **Candidate:** And when the shared state store is unavailable — do I allow or deny?

> **Interviewer:** Depends on the endpoint, which is why I want it configurable rather than
> global. For the general API, **fail open** — a limiter outage must not become an API
> outage. For endpoints with real per-request cost, our LLM inference routes, **fail
> closed**; overspending is worse than being briefly unavailable. And I'd expect some
> degraded local enforcement rather than a binary choice.

**Pins down:** shared state within a region, no cross-region coordination, ≤10%
regional overshoot tolerated, per-rule fail-open/fail-closed policy with a degraded local
fallback.

---

## Q5 — Do we tell the user they've been throttled?

> **Candidate:** On rejection, how much do we tell the client?

> **Interviewer:** Yes, and more than most people think.
>
> Return **`429` with `Retry-After`** — that's the baseline. But also return the
> `RateLimit-*` headers on **successful** responses, not just rejections. If a client only
> learns its remaining budget by exhausting it, you've forced every well-behaved integrator
> to discover limits by tripping them. Publishing remaining budget continuously is what
> lets them self-pace.
>
> Distinguish **`429` from `503`**. 429 means *you* exceeded your limit — back off, and
> the situation is your own. 503 means *we're* shedding load — retry, it isn't about you.
> Clients should react differently, and collapsing them means they can't.

> **Candidate:** Anything about the retry timing itself?

> **Interviewer:** Jitter it. If you tell four thousand rejected clients that the window
> resets at exactly 12:01:00, they all come back in the same millisecond. You've built a
> thundering herd and scheduled it precisely.

> **Candidate:** Anything beyond the response itself?

> **Interviewer:** Two things. Customers should see current usage in the dashboard, and we
> want to **alert an account as it approaches its ceiling**, not after it's been throttled
> for an hour — most throttling incidents are accidents, and a warning fixes them before
> support gets involved. So the limiter has to emit usage telemetry, not just decisions.
>
> And be careful the headers don't leak anything about *other* tenants — remaining budget
> on a shared tenant limit tells one user something about their colleagues' activity.
> Usually fine internally, worth thinking about.

**Pins down:** `429` + jittered `Retry-After`, `RateLimit-*` headers on all responses,
429/503 kept distinct, usage telemetry emitted for dashboards and pre-throttle alerting.

---

## Finalized requirements

### Functional

1. Allow/deny each request against rules of the form *N requests per window*, keyed on
   authenticated identity, tenant, or IP.
2. Support composable multi-dimensional rules (identity × tenant × endpoint class);
   most restrictive wins, evaluation short-circuits on first denial.
3. Support per-endpoint-class weighting so expensive routes consume more budget.
4. Reject with `429`, a jittered `Retry-After`, and `RateLimit-Limit` /
   `-Remaining` / `-Reset`; emit those headers on successful responses too.
5. Keep `429` (caller exceeded quota) distinct from `503` (server shedding load).
6. Limits are reconfigurable at runtime without redeploying the gateway.
7. Emit per-key usage telemetry for customer dashboards and approaching-limit alerts.
8. Per-rule behavior when the state store is unavailable: fail open or fail closed.

### Non-functional

| Requirement | Target |
| ----------- | ------ |
| Peak throughput | ~200k QPS (115k measured peak + growth and batch-spike headroom) |
| Added latency | ≤ 5 ms p99, 10 ms hard ceiling |
| Active key cardinality | ~10M concurrent, from 100M registered |
| Accuracy | ≤ 10% overshoot within a region; 2× overshoot unacceptable |
| Availability | No single point of failure for the API; degraded local enforcement on store outage |
| Memory | Bounded by active keys, not registered users — TTL mandatory |

### Placement

- Enforcement in the **API gateway tier**, after cheap identity extraction, before any
  expensive work.
- Shared state **within a region**; no synchronous cross-region coordination.
- Client SDK backoff retained as a courtesy, never as enforcement.

### Explicitly out of scope

- **Volumetric / DDoS defense** — belongs at the CDN edge with coarser rules.
- **Cross-region global limits** — accepted overshoot for callers spanning regions.
- **Load shedding by server health** — a separate mechanism that runs alongside this one.
- **Long-window billing quotas** — a metering problem with durability requirements a rate
  limiter does not have.

---

**Next:** [back-of-the-envelope.md](back-of-the-envelope.md) — sizing from these numbers,
then [design.md](design.md) — the design built against them.
