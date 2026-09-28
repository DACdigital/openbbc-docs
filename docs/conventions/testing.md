# Testing Conventions

Test the right thing at the right level; behavior that spans services needs cross-repo tests, not
just unit coverage inside each one.

## Test levels
- **Unit** — pure domain/business logic, no I/O, no network, no database; fast (ms), run on every
  commit. Mock/stub at the boundary (repositories, external clients).
- **Integration** — one service against its real dependencies (its own database, message broker) via
  test containers or equivalent; verifies persistence, migrations, and the API contract at that
  service's edge.
- **End-to-end (e2e)** — a real user/business flow across multiple services, run against a
  staging-like environment; verifies the contracts between services actually hold, not just each
  service in isolation.

## What to test at each level
- Unit: business rules, edge cases, error handling, pure functions — the bulk of the suite.
- Integration: repository/DB queries, API request/response shape, event publish/consume against a
  real test broker, auth/permission checks.
- E2E: the critical business journeys only (e.g. sign-up → first purchase) — a handful of paths, not
  every permutation; these are the most expensive and slowest to maintain.

## Coverage expectations
- No fixed percentage target; the bar is "every domain rule and every public contract has a test that
  fails if the rule or contract breaks." Untested domain logic paths block a code-PR merge.
- New API endpoints and events ship with at least one integration test asserting the contract shape
  (see `api.md`, `events.md`).

## Cross-repo end-to-end testing
- E2E tests that span services live in one dedicated location (a shared e2e repo, or the consuming
  frontend repo) — never duplicated inside every service repo.
- Run cross-repo e2e against contract-mocked dependent services in CI for speed; run the full
  real-service e2e suite on a schedule or pre-release, not on every commit.
