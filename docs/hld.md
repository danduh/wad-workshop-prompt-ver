# HLD — Prompt Versioning Service

High-level design for the MVP in [`prd.md`](prd.md). Diagrams are Mermaid (they
render on GitHub and in any Mermaid viewer). The design choices sketched here are
formalised as ADRs — see [`decisions.md`](decisions.md).

## Domain model

A **Prompt** owns many immutable **Versions**. A **Label** is a named pointer to a
single version; `production` is the retrieval default, and moving it is
promote / rollback.

```mermaid
classDiagram
  class Prompt {
    +string prompt_key
    +string name
    +datetime createdAt
  }
  class Version {
    +int version
    +string content
    +json variables
    +json params
    +string contentHash
    +datetime createdAt
  }
  class Label {
    +string name
    +int version
  }
  Prompt "1" --> "*" Version : has
  Prompt "1" --> "*" Label : has
  Label "1" --> "1" Version : points to
  note for Version "immutable once created; edits create a new version"
  note for Label "production = retrieval default; moving it = promote / rollback"
```

## Architecture & module boundaries

Nx layers in `apps/` and `libs/`. Every arrow below is **enforced** by
`@nx/enforce-module-boundaries` — the build fails if anyone crosses a line.
`domain` depends only on `types` (never on `data-access`), and `web` can only
reach `web` libs + `shared` (never server internals). Storage sits behind the
data-access repository interface: an in-memory adapter now, a SQLite adapter as a
documented swap-in.

```mermaid
flowchart TD
  web["apps/web · React<br/>type:app · scope:web"]
  api["apps/api · NestJS<br/>type:app · scope:api"]
  webui["web/ui<br/>type:ui"]
  domain["prompts/domain<br/>type:domain"]
  da["prompts/data-access<br/>type:data-access"]
  types["shared/types<br/>type:types"]
  inmem["InMemory repo<br/>(now)"]
  sqlite["SQLite repo<br/>(swap-in)"]

  web --> webui
  web --> types
  api --> domain
  api --> da
  api --> types
  da --> domain
  da --> types
  webui --> types
  domain --> types
  da --> inmem
  da -.-> sqlite
```

## Sequence — create prompt + first version (auto-promote)

The first version of a prompt is automatically labelled `production`, so a
default fetch works immediately.

```mermaid
sequenceDiagram
  actor Dev
  participant API as api (NestJS)
  participant Domain as domain
  participant Repo as data-access (repository)
  Dev->>API: POST /api/prompts (prompt_key, name)
  API->>Repo: createPrompt(prompt_key, name)
  Repo-->>API: prompt
  API-->>Dev: 201 prompt
  Dev->>API: POST /api/prompts/:key/versions (content)
  API->>Domain: build version v1 + contentHash
  API->>Repo: saveVersion(v1)
  API->>Repo: setLabel(production, v1)
  Repo-->>API: version v1 (production)
  API-->>Dev: 201 version v1 (production)
```

## Sequence — retrieve default & rollback

Rollback is just moving the `production` label to an earlier version — no
redeploy.

```mermaid
sequenceDiagram
  actor App as Consuming app
  actor Dev
  participant API as api
  participant Repo as data-access
  App->>API: GET /api/prompts/:key
  API->>Repo: getByLabel(key, production)
  Repo-->>API: version (production)
  API-->>App: 200 content
  Note over Dev,API: rollback = move the label, no redeploy
  Dev->>API: PUT /api/prompts/:key/labels/production (version 1)
  API->>Repo: setLabel(production, v1)
  Repo-->>API: ok
  App->>API: GET /api/prompts/:key
  API-->>App: 200 content (now v1)
```
