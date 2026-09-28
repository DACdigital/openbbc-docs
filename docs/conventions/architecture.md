# Architecture Conventions

> Cross-repo ("fullstack") standards — apply across all service repos under `workspace/`. They can
> seed per-repo `.claude/rules` in each service repo, distinct from that repo's own single-repo,
> stack-specific rules (e.g. pytest, Playwright E2E), which stay local to it.

Keep the system decomposed into bounded contexts with explicit data ownership and a stable dependency
direction; cross-cutting concerns live in shared infrastructure, not scattered per-service.

## Bounded contexts
- One bounded context = one business capability + its own data. A context maps to one or more
  service repos; a service repo never spans two contexts.
- Each context owns its data exclusively — no other context reads or writes its store directly (see
  `persistence.md`).
- Name the context after the business capability (`billing`, `onboarding`), not the technology
  (`db-service`).

## Layering & dependency direction
- Standard layers: presentation → application/use-case → domain → infrastructure. Dependencies point
  inward; domain code has zero framework/infra imports.
- No circular dependencies between contexts. If context A needs data from B, A calls B's published
  API or subscribes to its events — never B's internals or database.
- Shared code lives in an explicit shared/common package, versioned and consumed like any other
  dependency — never via copy-paste or cross-repo symlinks.

## Cross-cutting concerns
- Logging, auth, config, and observability (metrics/tracing) live in a shared library or platform
  layer, applied consistently at the edge (gateway/middleware) — not reimplemented per feature.
- Platform-wide infra decisions (e.g. which tracing backend) are made once and recorded here, not
  decided ad hoc per service.

## Service boundaries
- Draw a service boundary at a stable business capability with its own release cadence and data
  ownership — not at a technical layer (no "database service", no "utils service").
- Communicate across boundaries via versioned APIs or events (see `api.md`, `events.md`); never via
  a shared database, shared in-memory state, or direct code imports across repos.

## When to introduce a new context
- Split when a capability needs an independent release cadence, independent scaling, or a distinct
  data-ownership/compliance boundary from its neighbor.
- Do not split for org-chart reasons alone or "just in case" — a new context is a C2 change (see
  `docs/process/README.md`) and carries real coordination cost.
