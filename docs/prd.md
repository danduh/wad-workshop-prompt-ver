# PRD — Prompt Versioning Service

- **Status:** Accepted
- **Date:** 2026-09-19

## 1. Problem & goal

Teams increasingly depend on LLM prompts as production assets, but prompts are
usually edited in place — no history, no way to pin what's live, no safe rollback.
The Prompt Versioning Service versions prompts the way git versions code: every
change is a new, **immutable** version, and a movable `production` label decides
which version callers get by default.

**Goal:** let a developer create a prompt, evolve it through immutable versions,
mark one version as production, and fetch the right version over a simple API —
with rollback being nothing more than moving the label.

## 2. Users & use cases

- **Prompt author (developer):** creates prompts, adds new versions, promotes a
  version to production, rolls back by re-promoting an older one.
- **Consuming application / service:** at runtime, fetches a prompt's production
  version (or a specific version) via the API by `prompt_key`.

Primary use cases:
1. Create a new prompt.
2. Publish a change as a new immutable version.
3. Promote a version to `production`.
4. Fetch the production version by `prompt_key` (default), or a specific version.
5. Roll back by promoting a previous version to `production`.

## 3. Domain model

- **Prompt** — a logical container. `prompt_key` (unique, stable, lowercase slug
  matching `^[a-z0-9]([a-z0-9-]*[a-z0-9])?$`, e.g. `welcome-email`), `name`,
  timestamps.
- **Version** — an **immutable** snapshot of a prompt's content. A **monotonic
  `version` integer** per prompt (referenced as version 1, 2, …), `content`
  (template text), optional `variables` and `params` (model, temperature, tags),
  a **content hash** (integrity / dedup), and a created timestamp. Once created,
  never changed.
- **Label** — a named pointer to one version of a prompt. The MVP requires
  `production`; the model is general (a label is just a name → version), so
  `dev` / `staging` are a trivial later addition. Moving `production` = promote or
  roll back.

Invariants:
- `prompt_key` is unique across prompts.
- A version, once written, is immutable — edits create a new version.
- `production` points to at most one version per prompt.
- Creating a prompt's **first** version sets `production` to it automatically.

## 4. MVP functional requirements

- **FR1 — Create prompt:** create a prompt with a unique `prompt_key` (slug) and
  name.
- **FR2 — Create version:** add a new immutable version to a prompt (content +
  optional variables / params); versions are numbered in order. A prompt's first
  version is auto-promoted to `production`.
- **FR3 — Set production label:** point `production` at a specific version
  (promote or roll back).
- **FR4 — Retrieve by key + version:** fetch a specific version of a prompt.
- **FR5 — Retrieve default:** fetch a prompt by `prompt_key` with no version →
  return the `production`-labelled version. If the prompt has no versions, return
  a clear not-found / no-production error.
- **FR6 — Immutability:** no API path may mutate an existing version's content.
- **FR7 — Minimal web UI:** browse prompts and their versions, view a version's
  content, and set the production label. Creating prompts / versions is API-first
  for the MVP (UI authoring is a later slice).

## 5. API surface (sketch — finalised in the ADRs / plan)

```
POST   /api/prompts                          create a prompt {prompt_key, name}
GET    /api/prompts                          list prompts
POST   /api/prompts/:key/versions            create a version {content, variables?, params?}
GET    /api/prompts/:key/versions            list versions
GET    /api/prompts/:key/versions/:version   get a specific version
PUT    /api/prompts/:key/labels/production    set production -> {version}
GET    /api/prompts/:key                      get the production version (default)
```

Request / response shapes are DTOs shared between API and web, validated on every
route. Exact contracts are pinned in the ADRs and the plan.

## 6. Out of scope (MVP)

- **Version diff** — planned as the next slice after the MVP.
- Authentication / authorization; multi-tenancy.
- Networked / managed database — storage stays local.
- A/B testing, evaluation linkage, analytics.
- Arbitrary labels beyond `production` (the model supports them; not required now).

## 7. Success criteria

- End to end: create a prompt (first version auto-production) → add a second
  version → promote v2 → `GET /api/prompts/:key` returns v2 → promote v1 → the
  same call returns v1 (rollback).
- Immutability holds: no endpoint changes stored version content.
- The web UI lists prompts / versions and can set the production label.
- Covered by tests; API contracts validated; lint + module boundaries green.

## 8. Resolved decisions

- **`prompt_key`:** lowercase slug, unique across prompts.
- **Version identifier:** monotonic integer per prompt; a content hash is stored
  for integrity / dedup.
- **First version** auto-promotes to `production`.
- **Web UI:** browse + view + set production for the MVP; authoring is API-first.

Deeper architecture decisions (storage engine, hashing, DTO / validation, API
framework specifics) are recorded as **ADRs** — see `docs/decisions.md` (next
step: `04-adrs`).
