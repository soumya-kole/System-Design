# CLAUDE.md

Guidance for Claude Code when working in this repository.

## What this repo is

A documentation-first reference library of system design concepts. It is **notes, not a
codebase** — there is no build, no test suite, and no application to run. The deliverable
of almost every task here is Markdown that a human will read while studying or preparing
for a design discussion.

Optimize for: correctness, concrete numbers, and honest trade-offs. Avoid: marketing tone,
padded sections, and restating the same idea in three places.

## Structure

- `common/` — cross-cutting fundamentals reused by many concepts (consistency models,
  hashing, queues, caching primitives, networking, storage engines, …). May contain
  subfolders or bare `.md` files.
- `<concept-name>/` — one top-level folder per system design concept (`rate-limiter`,
  `url-shortener`, `consistent-hashing`, …). Every concept folder has exactly these three
  files, read in this order:
  1. `requirement.md` — requirements gathering. Written as the clarifying dialogue
     (candidate asks, interviewer answers, each exchange ends with what it pins down),
     closing with a **Finalized requirements** section: functional, non-functional targets
     as a table, placement, and explicit out-of-scope.
  2. `back-of-the-envelope.md` — sizing. Starts from an inputs table copied from the
     finalized requirements, derives traffic, storage, bandwidth, node counts, and latency
     budget with the arithmetic shown, and ends with a summary table naming which
     constraint actually binds.
  3. `design.md` — the design built against the other two: problem, mechanism,
     approaches, deep dive, failure modes, real-world implementations.
- `README.md` — root index; holds the concept table and the common building blocks table.

Conventional subfolders inside a concept (all optional): `notes/`, `diagrams/`,
`examples/`.

## Rules

**Naming**

- Folders and files: `kebab-case`. No spaces, no capitals, no dates in names.
- Concept folders contain no `README.md`; the entry point is `design.md`. `README.md`
  appears only at the root and under `common/<topic>/`.

**Adding a new concept**

1. Create `<concept-name>/requirement.md`, `back-of-the-envelope.md`, and `design.md`
   following the outlines in the root [README](README.md). Write all three; a concept with
   only a design is incomplete.
2. Numbers flow forward: the design quotes the sizing, the sizing quotes the requirements.
   Don't invent a figure in `design.md` that isn't derived in `back-of-the-envelope.md`.
3. Link the three files to each other (previous/next at the bottom, and from the design's
   Requirements and Capacity sections back to their sources).
4. Add a row to the concept index table in the root `README.md` (alphabetical order).
5. Link out to any `common/` topics it depends on instead of re-explaining them.

**Adding to `common/`**

1. Put it under `common/<topic>/README.md` (or `common/<topic>.md` if it stays small).
2. Add a row to the "Common building blocks" table in the root `README.md`.
3. Update concepts that were explaining it inline to link here instead.

**Content**

- `design.md` must stand on its own: a reader landing there cold should get the problem,
  the mechanism, and the trade-offs without opening another file. It restates the
  finalized requirements as a compact table and the headline sizing numbers as a short
  block — enough to follow the design, not a copy of the other two files.
- `requirement.md` is a dialogue, not a list. Every question should change the design if
  answered differently; cut the ones that don't.
- `back-of-the-envelope.md` shows every calculation as `input × input ≈ result`. A number
  with no visible derivation is a bug.
- Prefer a comparison table over prose when weighing more than two options.
- Back-of-envelope math shows its assumptions. `100M DAU → ~1.2k QPS average` is useful;
  a bare number is not.
- Diagrams as Mermaid in fenced ```mermaid blocks. Images only when Mermaid can't do it,
  and then they live in the concept's `diagrams/`.
- Cross-doc links are relative paths.
- Code snippets are illustrative pseudocode or short real code. Anything meant to run goes
  in `examples/` with run instructions.

**Don't**

- Don't duplicate a `common/` explanation inside a concept — link it.
- Don't create empty scaffolding folders "for later"; add a folder when there's content.
- Don't rename or restructure existing concept folders without being asked — inbound
  relative links break silently.
- Don't add build tooling, linters, or CI unless explicitly requested.

## Working style here

- When asked to add a concept, write all three files in full rather than stub outlines.
- When editing an existing concept, keep its existing section ordering and voice. If a
  number changes, change it at its source (`requirement.md` or `back-of-the-envelope.md`)
  and propagate forward.
- After creating or removing a concept folder, update the root `README.md` tables in the
  same change.
