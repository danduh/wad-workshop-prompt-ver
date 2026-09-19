# Implementation Plan — Prompt Versioning Service MVP

- **Status:** Draft
- **Date:** 2026-09-20
- **Based on:** [`docs/adr/`](../docs/adr/) (0001–0005), [`docs/hld.md`](../docs/hld.md),
  scope from [`docs/prd.md`](../docs/prd.md)

## How to work

- Each task below is an **independent deliverable** with its own **test** — build
  it, test it, ship it, move on.
- **Delivery order (real-life):** the **full API — including its Swagger docs — is
  built, tested, and green before any UI work begins.**
- **Test-first**, one commit per task, strict TypeScript (no `any`). Keep
  `nx run-many -t lint test build` green.
- Order is **bottom-up** so every task only depends on ones already done and never
  crosses a module boundary: `shared/types → prompts/domain → prompts/data-access
  → apps/api → web/ui → apps/web`.
- Storage is the **in-memory adapter** behind the repository interface (ADR-0001).
  **Version diff is out of scope.**

## Tasks

### Contracts & domain — `@pvs/shared-types`, `@pvs/prompts-domain`

**T1 — Shared contract types.** (ADR-0002, ADR-0004; FR1/2/6)
- Deliver: `Prompt`, `Version`, `Label`, `PromptParams` entities + the request/
  response contracts (`CreatePromptRequest`, `CreateVersionRequest`,
  `SetProductionLabelRequest`, `ApiErrorResponse`) — pure interfaces, no runtime deps.
- Files: `libs/shared/types/src/lib/entities.ts`, `.../contracts.ts`, `index.ts`
  (replace `types.ts` placeholder).
- Test: type-satisfaction spec; `nx test types`.

**T2 — Prompt-key validation.** (FR1, PRD slug rule)
- Deliver: `isValidPromptKey` / `assertValidPromptKey` (`^[a-z0-9]([a-z0-9-]*[a-z0-9])?$`).
- Files: `libs/prompts/domain/src/lib/prompt-key.ts`. Test: accept/reject; `nx test domain`.

**T3 — Content hashing.** (ADR-0002)
- Deliver: `computeContentHash` → `sha256:…` over normalized content (`node:crypto`).
- Files: `.../content-hash.ts`. Test: determinism + identical-content dedup.

**T4 — Version factory + domain errors.** (ADR-0002)
- Deliver: `makeVersion(n, input, createdAt)` (immutable) + typed errors
  (`PromptKeyInvalid`, `DuplicatePromptKey`, `PromptNotFound`, `VersionNotFound`,
  `NoProductionVersion`, `LabelTargetNotFound`).
- Files: `.../version-factory.ts`, `.../errors.ts`, `index.ts` (remove `domain.ts`).
- Test: version shape/immutability; error identities.

### Storage — `@pvs/prompts-data-access`

**T5 — Repository port + DI token.** (ADR-0001)
- Deliver: `PromptRepository` interface (`createPrompt`, `listPrompts`, `getPrompt`,
  `addVersion`, `listVersions`, `getVersion`, `setLabel`, `getByLabel`) + injection token.
- Files: `.../prompt-repository.ts`, `.../prompt-repository.token.ts`. Test: compiles against contracts.

**T6 — In-memory repository.** (ADR-0001/0002/0003; FR1–FR6)
- Deliver: `InMemoryPromptRepository` — per-prompt monotonic sequence, first version
  auto-promotes `production`, unique `prompt_key`, label targets an existing version.
- Files: `.../in-memory-prompt-repository.ts` (remove `data-access.ts`).
- Test: dup-key→error, monotonic versions, first-auto-production, promote/rollback,
  `getByLabel`, no-production error, immutability + hash dedup; `nx test data-access`.

### API — `apps/api`  *(built and green before any UI)*

**T7 — Add API dependencies.** (ADR-0004, ADR-0005) — `class-validator`,
`class-transformer`, `@nestjs/swagger`. Test: installs; `nx build api`.

**T8 — Request DTO classes.** (ADR-0004, ADR-0005) — implement the shared contracts
with `class-validator` **and** `@ApiProperty` decorators (`@Matches` slug, `@IsInt @Min(1)` version…).
- Files: `apps/api/src/app/prompts/dto/*.dto.ts`. Test: validation accept/reject specs.

**T9 — Global `ValidationPipe` + exception filter.** (ADR-0003/0004) — `whitelist` +
`transform`; map domain errors → `ApiErrorResponse`.
- Files: `apps/api/src/main.ts`, `.../common/domain-exception.filter.ts`. Test: filter maps each error → status/shape.

**T10 — Prompts service + controller + module.** (FR1–FR6; PRD §5) — 7 routes; bind
`PROMPT_REPOSITORY → InMemoryPromptRepository`; wire into `AppModule`; remove default `App*`.
- Files: `apps/api/src/app/prompts/*.ts`, `app.module.ts`. Test: controller + service specs; `nx test api`.

**T11 — OpenAPI / Swagger.** (ADR-0005; FR — API contract) — `SwaggerModule` served at
`/api/docs`; DTOs/controllers annotated so the schema matches the routes.
- Files: `apps/api/src/main.ts` (+ `@ApiTags`/`@ApiResponse` on the controller).
- Test: `/api/docs` and the OpenAPI JSON serve and include all 7 routes (e2e assertion).

**T12 — API end-to-end flow.** (PRD §7) — create → auto-production → v2 → promote →
`GET` returns v2 → rollback → `GET` returns v1; 404s; immutability; `/api/docs` reachable.
- Files: `apps/api-e2e/src/api/prompts.e2e.spec.ts` (replace "Hello API"). Test: `nx e2e api-e2e`.

### Web UI — `@pvs/web-ui`  *(starts after the API is complete)*

**T13 — Presentational components.** (FR7, FR6, FR3) — `PromptList`, `VersionList`,
`VersionDetail` (read-only content/params/hash), `ProductionLabelControl` (pick a version + `onSetProduction`).
- Files: `libs/web/ui/src/lib/**/*.tsx`, `index.ts`. Test: component specs; `nx test ui`.

### Web app — `apps/web`

**T14 — Typed API client.** (FR3/4/5) — axios; types from `@pvs/shared-types` only.
- Files: `apps/web/src/app/api/prompts-api.ts`. Test: client spec (axios mocked).

**T15 — Compose the app.** (FR7) — browse prompts/versions, view content, set
production; remove `NxWelcome`.
- Files: `apps/web/src/app/app.tsx`. Test: `nx test web`.

## Whole-MVP acceptance

`nx run-many -t lint test build` green **and** `nx e2e api-e2e` green; `/api/docs`
serves the OpenAPI spec; and the success-criteria flow (create → promote →
rollback) works end to end.

## Out of scope

Version diff · the SQLite adapter · auth / multi-tenancy / network DB · labels
beyond `production` (modelled, not built).
