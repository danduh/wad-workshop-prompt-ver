# ADR-0002: Immutable versions identified by a monotonic integer, with a content hash

- **Status:** Accepted
- **Date:** 2026-09-19
- **Deciders:** Prompt Versioning Service team

## Context

Prompts are versioned like code: history must be preserved, references must be
stable, and content integrity matters. The API refers to versions explicitly
(`/versions/:version`) and via the `production` label.

## Decision

Each **Version is immutable** once written — an "edit" creates a new version.
Versions are identified by a **monotonic integer per prompt** (1, 2, 3, …), used
directly in the API path. Each version also stores a **content hash** (SHA-256 of
its normalized content) for integrity checks and duplicate detection. No API path
mutates an existing version's content.

## Example

```jsonc
// POST /api/prompts/welcome-email/versions   { "content": "Hi {{name}}" }
{ "version": 1, "contentHash": "sha256:9f2b1c…", "content": "Hi {{name}}", "createdAt": "2026-09-19T10:00:00Z" }

// An edit is a NEW version — v1 is never touched:
// POST /api/prompts/welcome-email/versions   { "content": "Hello {{name}}!" }
{ "version": 2, "contentHash": "sha256:4c7a90…", "content": "Hello {{name}}!", "createdAt": "2026-09-19T10:05:00Z" }
```

Fetch a specific one with `GET /api/prompts/welcome-email/versions/2`. Re-posting
identical content yields the same `contentHash` — a dedup signal.

## Consequences

- Human-friendly, stable references (`v3`) instead of opaque hashes.
- The content hash enables integrity verification and optional dedup.
- Immutability simplifies caching, auditing, and reasoning about "what shipped".
- The repository owns per-prompt sequence allocation; hashing cost is negligible.
