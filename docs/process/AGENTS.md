# Hard rules for agents

This document binds every Claude command and subagent operating in this repo. It **overrides**
Superpowers' looser defaults (`brainstorming`, `writing-plans`) wherever the two disagree — per
Superpowers' own precedence, project `AGENTS.md`/`CLAUDE.md` wins over skill defaults. Read
`docs/process/README.md` first for the process this document enforces.

## Required spec sections

Every feature spec (`docs/superpowers/specs/YYYY-MM-DD-<name>-design.md`) MUST contain
these sections, in this order. `/new-spec` (via `superpowers:brainstorming`) must produce
them; `/spec-review` must treat a missing or vague section as a finding, not a nit.

1. **Business value / Why** — the user or business outcome this feature delivers and how success is
   judged. Never skip this: it is how a developer implements against intent, not just mechanics.
2. **Change level (C0/C1/C2)** — the proposed classification per `docs/process/README.md`, with a one-line
   justification.
3. **Scope (in / out)** — what this feature does and explicitly does not do.
4. **Contracts** — API / data / events touched or introduced. **Required for C2**; may be stated as
   "none" for C1/C0-adjacent specs, but the section itself must still be present.
5. **Acceptance criteria** — concrete, checkable conditions for "done."
6. **Risks & assumptions** — what could go wrong, what is being assumed and not yet verified.

A spec missing any of these sections is incomplete and must not pass `/spec-review`.

## Hard rules

- **Ambiguous change level → pick the higher one.** If it is unclear whether a change is C0/C1 or C1/C2,
  classify it at the higher level.
- **A reviewer can raise the level, never lower it.** `/spec-review` (the single unified gate) may
  escalate a C1 spec to C2 via any of its sub-verdicts (Architecture or Contracts) if it finds
  architecture/data/contract impact the spec didn't declare. It may never downgrade a level a prior
  step assigned. `/arch-review` (the post-approval sync command) does not judge change level.
- **The fast lane is C0-only.** Never route a change through the fast lane — no spec, done directly in
  the service repo — if it touches: the database or migrations, an API, events, security, tenant
  isolation, or a framework/dependency upgrade. Any of those makes the change at least C1.
- **Nothing review-related is committed.** Verdicts, approvals, and review discussion live in PR
  comments/reviews — never as files in this repo. The PR history is the audit trail.
- **A FAIL verdict is never posted.** Only `PASS` and `PASS_WITH_ISSUES` verdicts reach a PR
  comment (and the comms channel). A `FAIL` verdict and its findings go back to the author in the
  invoking session and nowhere else — no PR comment, no comms notification. Getting a spec from
  first draft to its first passing verdict can take many `FAIL` rounds; commenting each one buries
  the PR's review space under superseded findings. Consequence: the first gate comment a spec PR
  ever receives is always `PASS` or `PASS_WITH_ISSUES`.
- **The spec gate runs before the push.** A spec reaches the remote only once `/spec-review`
  returns `PASS` or `PASS_WITH_ISSUES` — `/new-spec` runs it locally as a pre-push gate, and on
  `FAIL` the push does not happen, no PR is opened, nothing is posted anywhere, and the author
  gets the findings in the session to fix and retry. Once it passes, the PR opens **ready for
  review** carrying that verdict as its first comment. `/arch-log-review` (for branches touching
  `docs/architecture/current/`) still runs on the PR because it reviews a PR diff — a `FAIL`
  there follows the same rule and is not posted.
- **Architecture changes require a log entry.** Any PR that modifies `docs/architecture/current/`
  must add a new `docs/architecture/logs/YYYY-MM-DD-<codename>/README.md` in the same PR, with all
  required fields (Driver / Decision / Rationale / Alternatives rejected / Impact / Links) present.
  `architecture-log-review` enforces this.
- **The solution-architecture home is mandatory.** From Phase 2 onward, all 20 required files
  under `docs/architecture/current/` per the schema must exist. Each required section is
  either filled with real content or filled with `N/A because <reason>` (≥5-word reason). An
  `ARCH_GAP` marker signals an unfilled section and fails `/check-setup`. At least one entry
  must exist in `docs/architecture/logs/`. `/check-setup` reports the absence of any required
  file, any unfilled `ARCH_GAP` marker, or any dangling cross-reference as a fail.
- **Docs → tracker / service repos is the only allowed direction of influence.** Skills that
  operate in the docs repo (`/modularize`, `/plan-sprint`, `/new-feature`, `/new-spec`, `/create-issues`) may
  read docs state and write to `docs/` and/or the tracker. Skills that respond to *implementation*
  concerns (`/split-issue`, service-repo automation) may read tracker/PR/service-repo state but
  MUST NOT write to `docs/`. The master plan (`docs/`) shapes execution; execution never mutates
  the master plan.
- **Do not suggest `/roadmap-sync` outside planning.** `/roadmap-sync` is a planning-time
  reconciliation command — invoke it inside `/plan-sprint`, at sprint open/close, or when the
  user explicitly asks. Never suggest it as a follow-up to `/new-spec`, `/create-issues`,
  `/new-feature`, `/spec-review`, `/arch-review`, or any other feature-scoped work. Those flows
  already write the ROADMAP columns they own; a proactive `/roadmap-sync` suggestion pushes the
  user toward a planning operation from a non-planning context. `/status` may point at it as a
  hint when the user is actively looking at stale numbers — that is a diagnostic, not a
  workflow suggestion.
- **Architecture layer discipline.** `docs/architecture/current/` describes the product at three
  lens levels:
  - **Business** (`bizbok/`) — capabilities, stakeholders, value streams, information concepts.
  - **Domain** (`ddd/`) — bounded contexts, aggregates, domain events, access model.
  - **System** (`c4/`) — context, containers, data flows, deployment, integrations.

  Plus cross-cutting at the top level: `nfrs.md`, `assumptions.md`, `constraints.md`,
  `glossary.md`.

  It does NOT describe implementation choices — dependency versions, docker-compose or infra
  image bumps, CI configuration, lint rules, framework upgrades that don't change any contract,
  code-level refactors, small bug fixes. It also does NOT describe delivery scheduling — sprint
  numbers, calendar dates, quarters, tracker cycle or milestone references, deadlines, or
  "deferred to the next sprint." That information lives in `ROADMAP.md` and the tracker and
  decays too fast to be architecture-of-record.

  **Product-phase language is allowed:** MVP, M1, M2, Beta, GA, v1, "phase 1" describe the shape
  of the product at a delivery slice, not when that slice ships. Rule of thumb: if the change
  would still matter to someone reading the product plan a year from now, it belongs in
  `current/`; otherwise it belongs in a spec, a PR description, or a code comment.
  `architecture-log-review` enforces this.

  Reviewer soft signal: a single PR touching all three lens dirs (`bizbok/` + `ddd/` + `c4/`)
  at once is unusual and gets extra scrutiny — usually the change wasn't sliced cleanly.
- **Architecture files must conform to the schema.** `docs/architecture/current/` follows the
  schema at `.claude/skills/check-setup/arch-schema.md`: 20 required files, each with defined
  required sections and non-placeholder rules, enforced by `/check-setup`. Deviations require
  a schema change (edit `arch-schema.md`) under C2 discipline. Projects do not customize the
  schema per-project — template maintainers update it and `/sdd-rebase` propagates. New
  required files, sections, or cross-reference rules are breaking changes; removals are safe.
- **Reshape-from-freestyle is one-way via `/migrate-arch`.** Never manually reshape freestyle
  or pre-schema content in `docs/architecture/current/` into schema shape — run
  `/migrate-arch` and review its PR. The skill uses LLM interpretation, preserves gaps as
  `ARCH_GAP` markers, quarantines unmatched content in
  `docs/architecture/current/_migration-quarantine/`, and never invents content the source
  did not have. (Ordinary feature-driven edits to already-schema-shaped files via
  `/new-feature` are the normal path and are unaffected.)
