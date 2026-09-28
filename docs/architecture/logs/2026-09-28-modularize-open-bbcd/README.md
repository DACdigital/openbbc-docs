# modularize-open-bbcd — decompose open-bbcd into 7 L2 contract modules

**Date**: 2026-09-28
**Codename**: modularize-open-bbcd

**Driver**: /modularize run on open-bbcd

**Decision**: decomposed L1 `open-bbcd` into 7 L2 children: `agent-lifecycle`,
`feedback-datasets`, `evaluation`, `training`, `deployed-runtime`, `artifacts`, and
`tool-runtime`. The first six mirror the DDD contexts open-bbcd hosts (one L2 per hosted
context); the seventh (`tool-runtime`) is the cross-cutting LLM tool machinery both BO
chat and deployed runtime depend on.

**Rationale**: The natural axis for L2 decomposition of a Go daemon that hosts multiple
DDD contexts is one module per hosted context, because each context publishes its own
REST surface, owns its own tables, and has its own state machine — a change to one L2
should not cascade into another at compile time. The exception is the shared tool-dispatch
runtime (`tools.Builder`, `toolBackendStoreAdapter`, MCP client, MCP-over-REST bridge),
which sits under both `feedback-datasets` (BO chat orchestrator) and `deployed-runtime`
(production orchestrator) — modelling it as its own L2 keeps the shared-substrate contract
visible and prevents "does this belong to BO or Deployed?" from turning into a
copy-paste-then-diverge trap. All 17 L2 capabilities in `bizbok/capabilities.md` that open-
bbcd owns bundle cleanly into these 7 modules (roughly 2–3 capabilities per module).

Language note in the `agent-lifecycle` L2 purpose: the legacy noun "MCP attachment"
(`agent_version_mcp_backend`) was deliberately avoided in the module's prose in favour of
functional language ("per-version MCP visibility, plus editable per-version prompt notes")
— the name is embedded across `glossary.md`, `bizbok/information-map.md`,
`bizbok/capabilities.md`, `ddd/contexts/agent-lifecycle.md`, `ddd/access-model.md`, and
`assumptions.md`, and a repo-wide rename is a separate arch-change out of scope for
`/modularize`.

**Alternatives rejected**:
- **`backoffice-ui` as its own L2** — rejected: htmx templates + static assets + handler
  wiring are cross-cutting UI infra folded into each domain L2's handler code, not a
  contract module with its own boundary. If the UI stack ever gets split (SPA rewrite,
  separate frontend service, template engine change), this decision can be revisited.
- **`service-lifecycle` as its own L2** — rejected: too small to be a peer to the domain
  L2s. Subcommands (`serve` / `migrate` / `healthcheck`), boot, embedded goose migrations,
  and the health probe together are ≤500 LOC and change rarely; documented in
  `c4/containers.md § open-bbcd` and `nfrs.md § Observability`.
- **One L2 per L2 capability** (17 modules) — rejected: too fine-grained; capabilities
  cluster naturally by DDD context, and 17 L2s would fragment the tree without payoff.
- **Fold `tool-runtime` into `deployed-runtime`** — rejected: the tool machinery is
  genuinely shared with `feedback-datasets` (BO chat uses the same `tools.Builder` and
  MCP client). Folding it under either surface would misrepresent the dependency
  direction.

**Impact**:
- + docs/architecture/current/modularity/open-bbcd/agent-lifecycle/README.md
- + docs/architecture/current/modularity/open-bbcd/feedback-datasets/README.md
- + docs/architecture/current/modularity/open-bbcd/evaluation/README.md
- + docs/architecture/current/modularity/open-bbcd/training/README.md
- + docs/architecture/current/modularity/open-bbcd/deployed-runtime/README.md
- + docs/architecture/current/modularity/open-bbcd/artifacts/README.md
- + docs/architecture/current/modularity/open-bbcd/tool-runtime/README.md

**Links**:
- (user may add related spec PRs, feature docs, or tracker items before commit)
