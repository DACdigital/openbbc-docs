# openbbc-docs

A clone-and-use starter repository for **spec-driven software delivery**. Every non-trivial feature
is specified and reviewed before code is written: feature specs pass a **spec-PR gate** in this repo;
code passes a **code-PR gate** in the service repos. `ROADMAP.md` is the living hub that ties specs,
issues, and tracker state together.

Service repos live inside this repo as `workspace/` checkouts so Claude can reason across specs and
code from one place.

## Prerequisites

- [Superpowers](https://github.com/obra/superpowers) — **required**. `/new-spec` orchestrates
  `superpowers:brainstorming` (idea → spec) and `superpowers:writing-plans` (spec → plan). Install it
  before running any commands here.
- An issue tracker (Plane, GitHub, or GitLab) and, if different, a git host (GitHub or GitLab) for the
  service repos.
- MCP runtimes: the Plane MCP server runs via `uvx` — install [uv](https://docs.astral.sh/uv/) if you
  use Plane. The GitHub/GitLab MCP servers run via `npx` (Node). Missing runtimes are caught by the
  Phase 0 connectivity probe in `/setup-project`.

## Quick start

`/setup-project` is the **orchestrator** — it runs the full seven-phase onboarding (0–6) and works the
same whether you're starting fresh (greenfield) or adopting existing repos/tracker (brownfield):

1. `/setup-project` — Phase 0–6: configure + probe access, rename/detach this repo, fill context,
   import solution architecture, seed the `ROADMAP.md` shell, provision repos, create/fetch the
   tracker project, propagate rules, and audit the result.

The phase commands are also runnable on their own, and are re-runnable/idempotent:

- `/setup-workspace` — provision service repos (create from template/exemplar or clone) and index them.
- `/propagate-rules` — push repo/language-specific rules into the service repos via PR.
- `/check-setup` — read-only audit of onboarding completeness.
- `/modularize [<slug>]` — decompose the solution architecture into a three-level tree of module
  nodes; runs on demand per node.
- `/plan-sprint` — DM sprint-planning ceremony: pick items from the modularity tree + ad-hoc
  discussion, create the tracker milestone, append `ROADMAP.md` rows.
- `/split-issue <issue-id> [--pr <pr-url>]` — reactive tracker-only decomposition of a large issue
  into sub-issues; triggered by dev or reviewer when a PR proves too big.

Then `/new-feature <name>` to evolve `docs/architecture/current/` when the capability isn't there yet, or `/new-spec <name>` to brainstorm and write `docs/superpowers/specs/YYYY-MM-DD-<name>-design.md` for a capability already in `current/`.

## Levels

This project uses two independent "level" axes. Do not confuse them.

**Architecture tree levels (L1 / L2 / L3)** — depth in the modularity tree under
`docs/architecture/current/modularity/`. Managed by `/modularize`.

| Level | Name | What it holds |
|-------|------|---------------|
| **L1** | Architecture | services / infra entities (children of the tree root) |
| **L2** | Service | contract modules within a service |
| **L3** | Implementation | issue-sized shards; optional scope-grounding for `/split-issue` when a feature ticket needs breaking down |

Only L1 and L2 are architecture-of-record; L3 is planning-time decomposition stored in the same
tree for convenience.

**Change classification (C0 / C1 / C2)** — how much process a change goes through.

| Level | Name | When | Process |
|-------|------|------|---------|
| **C0** | trivial | typo, comment, one-liner, safe patch bump | fast lane — no spec |
| **C1** | normal | typical functional change; no arch/DB/API/event impact | spec → `/spec-review` → issues → code PR |
| **C2** | architectural | new bounded context; DB/API/event/contract change; tenant isolation | same as C1 — `/spec-review` covers architecture and contracts in one pass; optional post-approval `/arch-review <feature>` syncs `docs/architecture/current/` |

**The two axes are independent.** An L2 module (Service level) may host either a C1 or a C2
change; an L1 addition is usually a C2 change but not by definition.

## Repo layout

```
docs/process/         the process — C0/C1/C2 change class, gates, per-feature flow (tool-neutral source of truth)
docs/conventions/      cross-repo standards (architecture, naming, api, persistence, events, testing)
docs/architecture/     this project's target solution architecture (current/) + change log (logs/)
docs/repos/            per-repo navigation indexes (committed): stack, layout, entry points, commands
docs/superpowers/      specs/YYYY-MM-DD-<name>-design.md + plans/YYYY-MM-DD-<name>.md — one file per feature
.claude/skills/        /setup-project (orchestrator), /setup-workspace, /propagate-rules, /check-setup, /new-feature, /new-spec, /arch-log-review, ...
.claude/agents/        spec-review (unified), architecture-review (current-sync delta detector), architecture-log-review
workspace/             gitignored service-repo checkouts
ROADMAP.md             living hub: feature → priority/level/spec/issues/milestone/tracker link
```

### Schema for `docs/architecture/current/`

Every project's `docs/architecture/current/` follows a fixed 20-file schema across three
architectural lenses plus cross-cutting top-level files. The schema lives at
`.claude/skills/check-setup/arch-schema.md` and is enforced by `/check-setup`, scaffolded by
`/setup-project`, and migrated into via `/migrate-arch`.

Layout at a glance:

    docs/architecture/current/
    ├── README.md, nfrs.md, assumptions.md, constraints.md, glossary.md    ← top-level
    ├── bizbok/  — capabilities, stakeholders, value-streams, information-map    ← business layer
    ├── ddd/     — context-map, contexts/<name>.md, access-model                  ← domain layer
    ├── c4/      — context, containers, data-flows, deployment, integrations    ← system layer
    ├── estimation/  (optional; coverage-gated if present)
    └── modularity/  (existing; planning-time decomposition tree)

Missing files, unfilled `ARCH_GAP` markers, or dangling cross-references fail `/check-setup`.
See `.claude/skills/check-setup/arch-schema.md` for the full rules.

## Links

- [docs/process/README.md](docs/process/README.md) — the process in full.
- [docs/conventions/](docs/conventions/) — starter standards; edit for your stack.
