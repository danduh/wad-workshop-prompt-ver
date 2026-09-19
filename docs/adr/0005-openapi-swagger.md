# ADR-0005: Expose the API via OpenAPI / Swagger

- **Status:** Accepted
- **Date:** 2026-09-20
- **Deciders:** Prompt Versioning Service team

## Context

ADR-0004 set the API conventions and left OpenAPI / Swagger *optional*. In
practice, developers integrating the service need a live, browsable contract, and
the team requires it for the MVP. NestJS supports this first-class via
`@nestjs/swagger`.

## Decision

The API **must** expose an OpenAPI document and a Swagger UI at **`/api/docs`**,
built with `@nestjs/swagger`. Request / response DTOs are annotated
(`@ApiProperty`) so the published schema is accurate, and the served spec is the
source of truth for the HTTP contract. This **supersedes the "Swagger optional"
clause of ADR-0004**; the rest of ADR-0004 stands.

## Example

```ts
// apps/api/src/main.ts
import { SwaggerModule, DocumentBuilder } from '@nestjs/swagger';

const config = new DocumentBuilder()
  .setTitle('Prompt Versioning Service')
  .setVersion('1.0')
  .build();
SwaggerModule.setup('api/docs', app, SwaggerModule.createDocument(app, config));

// apps/api — a DTO field carries both validation and schema annotations
import { ApiProperty } from '@nestjs/swagger';
export class CreatePromptDto {
  @ApiProperty({ example: 'welcome-email' })
  @Matches(/^[a-z0-9]([a-z0-9-]*[a-z0-9])?$/) prompt_key!: string;

  @ApiProperty() @IsString() @IsNotEmpty() name!: string;
}
```

## Consequences

- Adds the `@nestjs/swagger` dependency; DTOs carry `@ApiProperty` alongside their
  `class-validator` decorators.
- A browsable `/api/docs` plus machine-readable OpenAPI JSON (can later generate a
  typed client for `web`).
- Minor per-field decorator overhead; annotations must be kept in sync with validation.
