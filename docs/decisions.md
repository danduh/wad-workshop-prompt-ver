# Decisions

The front door to this project's decisions of record. Every architecturally
significant choice is captured as a numbered **ADR** in [`adr/`](./adr/), using
[`adr/template.md`](./adr/template.md) (MADR / Nygard format).

ADRs are **numbered, immutable, and append-only.** To reverse a decision, add a
new ADR and mark the old one *Superseded by ADR-NNNN* — never rewrite an accepted
one. Decisions of record are binding (see [`../AGENTS.md`](../AGENTS.md)): a
proposal that contradicts one must honor it or supersede it.

Use the `adr-writer` skill to add an ADR.

## Index

| ADR | Title | Status | Date |
|-----|-------|--------|------|
| [0001](adr/0001-storage-repository-interface.md) | Storage behind a repository interface (in-memory first, SQLite swap) | Accepted | 2026-09-19 |
