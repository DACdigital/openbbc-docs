# Implement Spec-Driven Docs Template — Plan

**Date:** 2026-07-21
**Source spec:** `docs/superpowers/specs/2026-07-15-spec-driven-docs-template-design.md` (the single source of truth)
**Branch:** `feat/implement-template` (stacked on `docs/spec-driven-template-design`)
**Execution:** subagent-driven, parallel waves over disjoint file groups.

## Goal

Populate this repo with the full, clone-and-use template described by the design spec: process docs,
conventions, `.claude/` commands + agents, root entry files, and config examples.

## Global constraints (bind every task)

- **The design spec is authoritative.** Every file's content, naming, and cross-references derive from
  it. When the spec fixes an exact name/path/value (command names, `tracker.json` schema, L0/L1/L2
  definitions, roles), reproduce it verbatim.
- **Writing principle:** concise and straight to the point for a technical audience — but complete.
  Omit ceremony, not substance. Markdown only; no marketing copy.
- **Placeholders, not fabrications.** Where a value is filled at init (org ids, project name, repo
  list), use an obvious placeholder (`<project>`, `<org>`, `…`), never invented specifics.
- **Company facts** (footers/attribution only where natural): DAC.digital (DAC.Infomotion Sp. z o.o.),
  www.dac.digital. Do not brand internal docs heavily.
- **Subagents write files only — no git operations.** The controller commits. Touch only the files in
  your task's scope; do not create or edit files owned by another task.

## Open-item resolutions

1. **Conventions content** — generic, stack-neutral starter guidance with concrete headings; teams
   edit. Each file states intent + a handful of concrete rules, not exhaustive standards.
2. **Required spec sections** (codified in `docs/process/AGENTS.md`, enforced over Superpowers'
   defaults): `Business value / Why`, `Level (L0/L1/L2)`, `Scope (in / out)`, `Contracts`
   (API / data / events — required for L2), `Acceptance criteria`, `Risks & assumptions`.
3. **`decisions.md` format** — reverse-chronological entries, each: `### YYYY-MM-DD — <decision>` then
   `**Decision** / **Context** / **Rationale** / **Alternatives rejected**` lines.
4. **Tracker adapters** — document the verbs per platform in `docs/process/README.md` and the relevant
   command docs: Plane (work items, cycles=milestones, epics), GitHub (issues, milestones, sub-issues),
   GitLab (issues, milestones, epics). Commands branch on `tracker.platform`; repo provisioning on
   `gitHost.platform`.
5. **README** — one page: what this is → prerequisites (Superpowers) → quick start
   (`/setup-project` → `/setup-workspace` → `/new-feature`) → repo layout → links to process &
   conventions.

## File structure & tasks (disjoint — safe to run in parallel)

- **T1 — Process docs:** `docs/process/README.md` (SDD process, L0/L1/L2, gates, per-feature flow,
  tracker-adapter verbs), `docs/process/AGENTS.md` (hard rules for agents incl. the required spec
  sections and `decisions.md` format).
- **T2 — Conventions:** `docs/conventions/{architecture,naming,api,persistence,events,testing}.md`
  (starter guidance; note they can also seed per-repo `.claude/rules`).
- **T3 — Root entry files:** `README.md`, `AGENTS.md` (hub → process docs), `CLAUDE.md`
  (`@AGENTS.md` + project-context placeholders), `ROADMAP.md` (the living-hub table with the
  `Feature | Priority | Level | Spec | Issues | Milestone | Link` header + one example row).
- **T4 — Config & plumbing:** `.mcp.json` (tracker MCP wiring, commented per platform),
  `.env.example`, `.claude/tracker.json` (example, matching the spec schema incl. optional `comms`),
  `.gitignore` (ignores `workspace/`), `workspace/.gitkeep`.
- **T5 — Commands:** `.claude/commands/{setup-project,setup-workspace,new-feature,spec-review,`
  `arch-review,contract-data-event-review,create-issues,roadmap-sync,status}.md` — one per the
  Commands table; each states purpose, inputs, steps, and which config key it reads
  (`tracker` vs `gitHost` vs `comms`).
- **T6 — Review agents:** `.claude/agents/{spec-review,architecture-review,`
  `contract-data-event-review}.md` — each reads `docs/process/` + `docs/conventions/`, emits a
  structured verdict (PASS / PASS_WITH_ISSUES / FAIL), posts to the PR. `spec-review` explicitly
  checks the required spec sections incl. business value.

## Execution

1. Wave 1: dispatch T1–T6 implementers in parallel (write-only). Controller commits per task group.
2. Wave 2: spec-compliance + cross-file-consistency review (command/agent names, paths, config keys,
   ROADMAP columns match the spec). Dispatch fixers for Critical/Important findings; re-review.
3. Final whole-branch review on the most capable model.
4. Push; offer stacked MR targeting `docs/spec-driven-template-design`.

## Acceptance

- Every path in the design tree exists with real content (T-scope above).
- No `templates/` dir; required spec sections live in `docs/process/AGENTS.md`.
- Cross-references resolve (commands ↔ agents ↔ process docs ↔ config keys).
- `tracker.json`/`.mcp.json`/`.env.example` match the spec schema and the three supported platforms.
