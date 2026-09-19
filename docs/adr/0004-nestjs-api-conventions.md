# ADR-0004: NestJS API conventions — shared contract types, validated DTOs, consistent errors

- **Status:** Accepted
- **Date:** 2026-09-19
- **Deciders:** Prompt Versioning Service team

## Context

The API and web app must agree on request / response shapes, every route must
validate input, and errors should be consistent. `shared/types` is a leaf lib (it
must depend on nothing), and the web app must not drag server-only packages.

## Decision

- **Contract types** (plain TypeScript interfaces) live in `libs/shared/types`,
  imported by both `api` and `web`. No runtime dependencies there.
- The **API** defines request **DTO classes** with `class-validator` decorators
  (implementing the shared interfaces) and enables a global `ValidationPipe`
  (`whitelist`, `transform`) plus a global **exception filter** for a consistent
  error shape.
- OpenAPI / Swagger is compatible and may be added later, but is not required for
  the MVP. **— Superseded by [ADR-0005](0005-openapi-swagger.md): Swagger is now
  required.**

## Example

```ts
// libs/shared/types — pure contract interface (no runtime deps)
export interface CreateVersionRequest {
  content: string;
  variables?: Record<string, string>;
  params?: { model?: string; temperature?: number; tags?: string[] };
}

// apps/api — validated DTO implements the shared interface
import { IsString, IsNotEmpty, IsOptional } from 'class-validator';
export class CreateVersionDto implements CreateVersionRequest {
  @IsString() @IsNotEmpty() content!: string;
  @IsOptional() variables?: Record<string, string>;
  @IsOptional() params?: CreateVersionRequest['params'];
}

// apps/api/src/main.ts
app.useGlobalPipes(new ValidationPipe({ whitelist: true, transform: true }));
```

`web` imports only `CreateVersionRequest` from `shared/types`; `class-validator`
never crosses into the web bundle.

## Consequences

- Validated, typed contracts on every route; shared interfaces prevent drift.
- `shared/types` stays dependency-free; `web` never imports `class-validator`.
- Consistent error responses across the API.
- A little boilerplate per endpoint (a DTO class + decorators).
