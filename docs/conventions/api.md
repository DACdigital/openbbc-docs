# API Conventions

APIs are contracts between services and their consumers. Changing one is a C2 change (see
`docs/process/README.md`); contracts must stay backward-compatible within a major version.

## Versioning
- Version in the URL path: `/v1/...`. Bump the major version only for breaking changes; additive
  changes (new optional field, new endpoint) ship without a version bump.
- Support the previous major version for a documented deprecation window (default: 6 months) before
  removal; surface the deprecation via a `Deprecation` response header and record it as an entry
  under `docs/architecture/logs/` (API deprecations are contract changes and trigger
  `architecture-log-review`).

## Resource naming
- Plural nouns, nested only for ownership: `/users/{id}/orders/{orderId}`. Max nesting depth: 2.
- Filtering, sorting, and search go through query params (`?status=active&sort=-createdAt`), not a
  new endpoint per filter combination.

## Error response shape
- One envelope for every error: `{ "error": { "code": "string", "message": "string", "details": [...] } }`.
- `code` is a stable, machine-readable string (`RESOURCE_NOT_FOUND`), not the HTTP status repeated;
  `message` is human-readable and safe to show; never leak stack traces or internal identifiers.

## Pagination
- Cursor-based pagination by default (`?cursor=...&limit=50`); offset-based only for small, bounded
  collections.
- Every paginated response includes `nextCursor` (nullable) and, where cheap to compute, `total`.

## Idempotency
- Every non-GET mutating endpoint that can be safely retried accepts an `Idempotency-Key` header;
  the same key + body within a retention window (default 24h) returns the original response instead
  of repeating the side effect.
- Idempotency keys are client-generated (UUID), scoped per endpoint, and stored with the result they
  produced.

## Backward compatibility
- Never remove or repurpose a field; add new fields as optional with sane defaults.
- Never change a field's type or semantic meaning in place — add a new field and deprecate the old
  one instead.
- Any contract change needs a `Contracts` section in the feature spec and a contract/data/event
  review before implementation (the C2 gate).
