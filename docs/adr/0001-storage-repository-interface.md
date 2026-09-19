# ADR-0001: Storage behind a repository interface (in-memory first, SQLite swap)

- **Status:** Accepted
- **Date:** 2026-09-19
- **Deciders:** Prompt Versioning Service team

## Context

The service must persist prompts, versions, and labels. Constraints:

- It is demoed on a mix of macOS and Windows machines, with **no external database**.
- `better-sqlite3` (the intended SQLite driver) is a native addon: it ships
  prebuilt binaries for macOS (Intel/ARM) and Windows x64 on Node 20, but an
  Alpine/musl target (the later Docker image) needs a compile step.
- The domain must stay **pure** — it must not depend on any storage engine.

## Decision

Define a **repository interface** (a port) in `libs/prompts/data-access` for
prompts, versions, and labels. Ship an **in-memory adapter first** — zero setup,
identical on every OS — and add a **`better-sqlite3` adapter** later as a drop-in
implementation of the same interface. The domain never imports `data-access`
(enforced by `@nx/enforce-module-boundaries`).

## Example

```ts
// libs/prompts/data-access — the repository port (illustrative)
export interface PromptRepository {
  createPrompt(input: { prompt_key: string; name: string }): Promise<Prompt>;
  addVersion(prompt_key: string, content: string /* + variables/params */): Promise<Version>;
  getVersion(prompt_key: string, version: number): Promise<Version | null>;
  setLabel(prompt_key: string, label: string, version: number): Promise<void>;
  getByLabel(prompt_key: string, label: string): Promise<Version | null>;
}

// Two interchangeable adapters implement the same port:
class InMemoryPromptRepository implements PromptRepository { /* Maps — now */ }
class SqlitePromptRepository  implements PromptRepository { /* better-sqlite3 — later */ }
```

The `api` depends on `PromptRepository` (via DI), never on a concrete adapter — so
swapping in-memory → SQLite changes one provider binding and nothing else.

## Consequences

- Zero-setup start; early sessions focus on domain / versioning logic, not DB setup.
- The port/adapter split is demonstrated live when SQLite is swapped in.
- In-memory data does not survive a restart (acceptable for early steps); tests
  run fast against it.
- The SQLite adapter's native-module and Alpine/musl caveats are deferred to when
  that adapter lands (the ship / Docker session).
