# Persistence Conventions

Each bounded context owns its data exclusively. Schema changes are deliberate, reviewed, and stay
backward-compatible for the duration of a rollout.

## Data modelling
- Model around the domain (aggregates/entities), not around UI screens or reports — reporting reads
  come from replicas/read models, not compromises on the write-side schema.
- Every table has an explicit primary key (no natural-key-only tables) and `createdAt`/`updatedAt`
  timestamps.
- Prefer normalized schemas for transactional data; denormalize deliberately, and document why, for
  read-heavy paths only.

## Migrations discipline
- Every schema change is a migration file, checked into the service repo, applied in order, never
  edited after merge — a wrong migration gets a corrective migration, not a rewrite of history.
- Migrations stay backward-compatible with the previous app version during rollout: add nullable
  first, deploy code, backfill, then remove/tighten in a later migration.
- No manual schema edits against any environment; the migration pipeline is the only path.

## Per-context data ownership
- No cross-context database access — not even read-only. A context that needs another context's
  data calls its API or subscribes to its events (see `api.md`, `events.md`).
- No shared tables between contexts. If two contexts seem to need the same table, the context
  boundary is likely wrong — revisit `architecture.md` instead of sharing the table.
- Each service repo owns exactly one context's schema; no service holds another service's database
  credentials or connection string.

## PII handling
- Explicitly classify every field that stores personal data (name, email, address, IP, device id) in
  the data model or its comments; default to *not* collecting a field unless there's a named business
  need for it.
- Encrypt PII at rest and in transit; never log raw PII — mask or redact it in logs and error
  messages.
- Design tables so a data-subject's PII can be located and purged on request without cascading damage
  to other users' data.
