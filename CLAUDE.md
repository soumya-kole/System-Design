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
  `url-shortener`, `consistent-hashing`, …). May contain subfolders and several files.
- `README.md` — root index; holds the concept table and the common building blocks table.

Conventional subfolders inside a concept (all optional): `notes/`, `diagrams/`,
`examples/`.

## Rules

**Naming**

- Folders and files: `kebab-case`. No spaces, no capitals, no dates in names.
- `README.md` is the only uppercase filename inside a concept folder.

**Adding a new concept**

1. Create `<concept-name>/README.md` following the outline in the root [README](README.md).
2. Add a row to the concept index table in the root `README.md` (alphabetical order).
3. Link out to any `common/` topics it depends on instead of re-explaining them.

**Adding to `common/`**

1. Put it under `common/<topic>/README.md` (or `common/<topic>.md` if it stays small).
2. Add a row to the "Common building blocks" table in the root `README.md`.
3. Update concepts that were explaining it inline to link here instead.

**Content**

- Every concept README must stand on its own: a reader landing there cold should get the
  problem, the mechanism, and the trade-offs without opening another file.
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

- When asked to add a concept, write the full README rather than a stub outline.
- When editing an existing concept, keep its existing section ordering and voice.
- After creating or removing a concept folder, update the root `README.md` tables in the
  same change.
