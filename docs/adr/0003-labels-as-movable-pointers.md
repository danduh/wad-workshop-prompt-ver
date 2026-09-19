# ADR-0003: Labels are movable pointers; `production` is the default; rollback re-points

- **Status:** Accepted
- **Date:** 2026-09-19
- **Deciders:** Prompt Versioning Service team

## Context

Callers need a stable "what's live" without editing version content, and rollback
must be instant with no redeploy. Versions are immutable (ADR-0002), so "live"
cannot be a mutable flag on a version.

## Decision

A **Label** is a named pointer from a prompt to exactly one of its versions.
`production` is required and is what `GET /api/prompts/:key` returns by default.
**Promotion and rollback are the same operation:** `PUT` the label to a target
version. A prompt's **first version auto-sets `production`**. The model is general
(a label is just a name → version), so `dev` / `staging` are a trivial later
addition.

## Example

```
create prompt "welcome-email"                          production → (none)
add version 1                                          production → v1   (auto)
add version 2                                          production → v1   (unchanged)
PUT /api/prompts/welcome-email/labels/production {2}   production → v2   (promote)
GET /api/prompts/welcome-email                         → v2
PUT /api/prompts/welcome-email/labels/production {1}   production → v1   (rollback)
GET /api/prompts/welcome-email                         → v1
```

Promotion and rollback are the same call — only the pointer moves; the content of
v1 and v2 is never touched.

## Consequences

- Instant rollback, with no redeploy and no version mutation.
- One clear source of "what's live" per prompt.
- A label must always point to an existing version (a domain / repository invariant).
- Fetching the default for a prompt with no versions returns a clear
  "no production version" error.
