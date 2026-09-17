# System Design

A personal reference library for system design concepts. Each concept lives in its own
top-level folder; shared fundamentals that many concepts build on live in [common/](common/).

## Layout

```
.
├── common/                  # cross-cutting fundamentals reused by many concepts
│   ├── consistency/
│   ├── networking/
│   ├── storage/
│   └── ...
├── rate-limiter/            # one folder per concept
│   ├── requirement.md       # 1. requirements gathering — always present
│   ├── back-of-the-envelope.md  # 2. sizing math — always present
│   ├── design.md            # 3. the design — always present
│   ├── notes/               # optional: deep dives, comparisons, papers
│   ├── diagrams/            # optional: .md with mermaid, or images
│   └── examples/            # optional: runnable code / configs
├── CLAUDE.md
└── README.md
```

Rules of thumb:

- One concept per top-level folder, named in `kebab-case` (`rate-limiter`, `consistent-hashing`).
- Every concept folder has the same three files, read in order: `requirement.md` →
  `back-of-the-envelope.md` → `design.md`. The design can be read standalone; the other
  two explain where its numbers and constraints came from.
- Anything reused by two or more concepts gets promoted into `common/` and linked, not copied.
- Subfolders are optional — add them only when a concept outgrows the three files.

## Concept index

<!-- Keep alphabetical. Add a row when a new concept folder is created. -->

| Concept | Summary | Files |
| ------- | ------- | ----- |
| [key-value-store](key-value-store/design.md) | Dynamo-style store — consistent hashing with virtual nodes, leaderless quorum replication, LSM storage, and why SSD endurance sets the node count. | [requirement](key-value-store/requirement.md) · [estimate](key-value-store/back-of-the-envelope.md) · [design](key-value-store/design.md) |
| [rate-limiter](rate-limiter/design.md) | Admission control — counting algorithms, atomic evaluation in Redis, and what to do when the store is down. | [requirement](rate-limiter/requirement.md) · [estimate](rate-limiter/back-of-the-envelope.md) · [design](rate-limiter/design.md) |

## Common building blocks

<!-- Add a row when a new topic lands in common/. -->

| Topic | Summary |
| ----- | ------- |
| [gossip-protocol](common/gossip-protocol.md) | Masterless membership and failure detection — epidemic spread in O(log N) rounds, generation/version merging, phi accrual vs SWIM, and what a "down" verdict may not trigger. |
| [interview-playbook](common/interview-playbook.md) | Running a 45-minute design round — the clock, questions that change the design, estimation order, and the answers interviewers expect. |
| [lsm-tree](common/lsm-tree.md) | Log-structured storage — WAL, memtable, SSTables, bloom filters, compaction strategies and their read/write/space amplification, tombstones, write stalls. |
| [references](common/references.md) | Shared reading list — queued and read. |

## Concept folder shape

Each concept is three files. Skip sections that don't apply rather than padding them.

### 1. `requirement.md` — requirements gathering

The first ten minutes of a design discussion, written as a dialogue: the candidate asks a
clarifying question, the interviewer answers with the concrete constraints, and each
exchange ends with a one-line **Pins down:** summary. Typical questions: where does it
run, what is the key, how much load, is it distributed, what does the client see. Ends
with a **Finalized requirements** section:

- **Functional** — numbered list of what the system must do.
- **Non-functional** — a table of targets with numbers (QPS, latency, accuracy,
  availability, memory).
- **Placement** — where it sits relative to the rest of the system.
- **Explicitly out of scope** — what was cut and why.

### 2. `back-of-the-envelope.md` — sizing

Starts from an **Inputs** table copied from the finalized requirements, then derives, with
the arithmetic visible: traffic (average, peak, design target), per-node load, store
throughput and node count, storage, bandwidth, latency budget, and anything else the
design will quote. Ends with a **Summary** table and one sentence naming which constraint
actually binds.

### 3. `design.md` — the design

1. **Problem** — what breaks without this.
2. **Requirements** — a compact table restating the finalized requirements, linking to
   `requirement.md` for the reasoning.
3. **Core idea** — the mechanism in a few sentences, with a diagram if it helps.
4. **Approaches** — the realistic options and what each one trades away.
5. **Deep dive** — data model, algorithms, APIs, and a **Capacity** subsection quoting the
   headline numbers from `back-of-the-envelope.md`.
6. **Failure modes & scaling** — what happens under load, partition, and restart.
7. **In the wild** — how real systems implement it.
8. **References** — papers, docs, talks.
9. **Related** — the sibling files, sibling concepts, and `common/` topics.

## Conventions

- Diagrams: prefer [Mermaid](https://mermaid.js.org/) in fenced ```mermaid blocks so they
  stay diffable. Images only when Mermaid can't express it.
- Links between docs are relative (`../common/consistency/README.md`), never absolute paths.
- Numbers get units and a stated assumption (`~50k QPS peak, assuming 10x average`).
- Code examples are illustrative and short; anything runnable goes under `examples/` with
  a note on how to run it.
