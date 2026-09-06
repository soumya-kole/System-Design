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
│   ├── README.md            # entry point — always present
│   ├── notes/               # optional: deep dives, comparisons, papers
│   ├── diagrams/            # optional: .md with mermaid, or images
│   └── examples/            # optional: runnable code / configs
├── CLAUDE.md
└── README.md
```

Rules of thumb:

- One concept per top-level folder, named in `kebab-case` (`rate-limiter`, `consistent-hashing`).
- Every concept folder has a `README.md` that can be read standalone.
- Anything reused by two or more concepts gets promoted into `common/` and linked, not copied.
- Subfolders are optional — add them only when a concept outgrows a single file.

## Concept index

<!-- Keep alphabetical. Add a row when a new concept folder is created. -->

| Concept | Summary |
| ------- | ------- |
| _(none yet)_ | |

## Common building blocks

<!-- Add a row when a new topic lands in common/. -->

| Topic | Summary |
| ----- | ------- |
| [references](common/references.md) | Shared reading list — queued and read. |

## Concept README shape

Each concept `README.md` follows roughly this outline. Skip sections that don't apply
rather than padding them.

1. **Problem** — what breaks without this.
2. **Requirements** — functional, non-functional, and explicit scope cuts.
3. **Core idea** — the mechanism in a few sentences, with a diagram if it helps.
4. **Approaches** — the realistic options and what each one trades away.
5. **Deep dive** — data model, algorithms, APIs, math (capacity, QPS, storage).
6. **Failure modes & scaling** — what happens under load, partition, and restart.
7. **In the wild** — how real systems implement it.
8. **References** — papers, docs, talks.
9. **Related** — links to sibling concepts and `common/` topics.

## Conventions

- Diagrams: prefer [Mermaid](https://mermaid.js.org/) in fenced ```mermaid blocks so they
  stay diffable. Images only when Mermaid can't express it.
- Links between docs are relative (`../common/consistency/README.md`), never absolute paths.
- Numbers get units and a stated assumption (`~50k QPS peak, assuming 10x average`).
- Code examples are illustrative and short; anything runnable goes under `examples/` with
  a note on how to run it.
