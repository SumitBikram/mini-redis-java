# Architecture Decision Records

An ADR is a short, dated note that records **one** significant design decision: the context,
what was chosen, what was rejected, and the consequences. ADRs are never edited after
they're accepted. If a decision changes, write a new ADR that supersedes the old one.

## When to write one

Write an ADR when a choice:

- is hard or expensive to reverse (I/O model, persistence format, threading model)
- had real alternatives that were weighed (e.g. `synchronized` vs `ConcurrentHashMap`)
- will make a future reader ask "why on earth is it done like this?"

Skip it for things that are easy to reverse or purely local (naming, small refactors).

## How

1. Copy [`TEMPLATE.md`](TEMPLATE.md) to `NNNN-short-title.md` (next number, zero-padded).
2. Start with status `Proposed`; set it to `Accepted` when the PR merges.
3. Add a row to the index below.

## Index

| # | Title | Status | Date |
|---|---|---|---|
| [0001](0001-use-pure-java-21-and-maven.md) | Use pure Java 21 and Maven, no frameworks | Accepted | 2026-10-25 |
