# Delivery process

Tool-neutral source of truth for how this team ships software. `.claude/skills` and `.claude/agents`
are the execution adapter for this process — if they ever disagree with this document, this document
wins.

## Purpose

Spec-first delivery: every non-trivial feature is **specified and reviewed before code is written**.
A feature spec passes a review gate in this repo (the spec PR); the resulting code passes a separate
review gate in the service repo (the code PR). `ROADMAP.md` is the living hub linking features to
specs, issues, and tracker items.

## Change classification

Every change is classified C0/C1/C2 before work starts. The change level decides how much process applies.

| Change level | When | Process |
|--------------|------|---------|
| **C0 — trivial** | typo, comment, one-liner, rename with no contract impact, safe patch bump | fast lane — no spec, done directly in the service repo |
| **C1 — normal** | typical functional change; no impact on architecture/DB/API/events | spec → spec-review → (optional plan) → issues → implement → code PR review |
| **C2 — architectural/data/contract** | new bounded context; DB/API/event/contract change; tenant isolation | same as C1 — the unified `/spec-review` covers architecture and contracts inline |

Rules:
- Ambiguous → pick the **higher** level.
- A reviewer can **raise** the level, never lower it.
- The fast lane is **C0-only** — never for changes touching DB, migrations, API, events, security,
  tenant isolation, or upgrades.

See `docs/process/AGENTS.md` for the full hard-rules statement agents must follow.

## Where each gate happens

- **Spec gate (C1/C2)** — happens **here**, gating `/new-spec`'s push and (if invoked by hand) an
  already-open spec PR. `/spec-review` posts a passing unified verdict (four sub-verdicts:
  Structure, Naming, Architecture, Contracts) as a PR comment; a `FAIL` is reported to the author
  in-session and never posted. A human (PL or delegate) approves and merges the PR. **The merge
  of the spec PR is the gate.** No separate architecture or contract gates — those rubrics are
  inside `/spec-review`.
- **Architecture sync (optional, post-approval)** — after a spec is approved (merged or not),
  `/arch-review <feature>` reconciles `docs/architecture/current/` with the approved spec on its
  own `arch/<codename>` branch (default codename `sync-<spec-slug>`) and its own PR. Not part of
  the pre-merge gate sequence.
- **Architecture-log gate (any PR touching `docs/architecture/current/`)** — happens **here**, on
  the PR that changes the arch. `/arch-log-review` posts a passing verdict as a PR comment; a
  `FAIL` is reported to the author in-session and never posted. Independent of C1/C2 — a pure
  refactor still triggers it.
- **Code gate (all implemented work)** — happens in the **service repos**, via normal human PR review.
  Not this repo's concern.

Nothing review-related is committed to the repo — PR comments and approvals are the audit trail, and
they live in the PR history, not in a file. Architectural decisions that shape the product are the
exception and DO live in the repo, under `docs/architecture/logs/YYYY-MM-DD-<codename>/README.md`,
written alongside the `docs/architecture/current/` changes they document. See AGENTS.md for the
log format and the `architecture-log-review` gate.

## Required gates, and when they run

A branch's **required gates** are decided by the change level:

| Change level | Required gates |
|--------------|----------------|
| **C0 — trivial** | none — no spec |
| **C1 / C2 — normal or architectural** | `/spec-review` |

Any branch that also touches `docs/architecture/current/` requires `/arch-log-review` on top,
regardless of level.

`/spec-review` is a single unified gate — one verdict with four sub-verdicts (Structure, Naming,
Architecture, Contracts). There are no separate architecture or contract gates on the spec PR;
those rubrics are folded into `/spec-review`.

**`/spec-review` runs before the push.**

1. `/new-spec` commits the spec locally and runs `/spec-review` against it. There is no PR yet,
   so the verdict comes back into the session. On `FAIL` nothing leaves the machine — no push, no
   PR, no comment, no notification — and the author fixes the spec and retries. Teams that want
   the same guarantee for hand-made pushes wire this gate into a git `pre-push` hook; the
   mechanism differs, the rule does not.
2. On `PASS` / `PASS_WITH_ISSUES` the branch is pushed and the PR opens **ready for review**
   carrying that verdict as its first comment.
3. `/arch-review` (the post-approval `current/` sync command) may then be run manually by the
   author or a reviewer after the spec is approved; it is not part of the pre-merge gate
   sequence.

**`/arch-log-review` runs on-PR** — it reviews a PR diff, so it cannot run before the PR exists.
Applies to any branch touching `docs/architecture/current/` (typically opened by `/new-feature`,
not `/new-spec`).

Throughout, **a `FAIL` verdict is never posted**. Only `PASS` and `PASS_WITH_ISSUES` reach the
PR, so a spec PR carries the verdict a human is asked to act on rather than the whole history of
drafts that preceded it. A gate invoked by hand on an already-open PR follows the same rule: a
`FAIL` is reported to the author in the session and kept off the PR.

## Modularity (planning-time)

Between spec-approval and issue-creation, the team can decompose a feature's L2 module into L3
implementation shards using `/modularize <l2-slug>`. This produces tree files under
`docs/architecture/current/modularity/` with a paired arch-log entry per run — enforced by
`architecture-log-review`. Modularity is **optional per feature** and never runs automatically;
`/create-issues` always creates one tracker issue per feature (planning-time), and L3 nodes when
they exist can be handed in as scope-grounding context to `/split-issue` if a feature ticket
later needs breaking down.

See `.claude/skills/modularize/SKILL.md` for the full flow. Three levels only: Architecture (L1) →
Service (L2) → Implementation (L3).

## Onboarding (one-time)

Before feature work, `/setup-project` runs a seven-phase init that works the same for greenfield
(create everything) and brownfield (adopt what exists), branching create-vs-fetch per resource. Every
phase is idempotent; a re-run resumes at the first incomplete phase.

| Phase | What it does |
|-------|--------------|
| 0 Access | choose tracker / git-host / comms; write `.env` + `.mcp.json`; probe each MCP server (must be green to proceed) |
| 1 Identity | pick a project name; rewrite placeholders; detach the template; create/adopt + push the docs repo |
| 2 Context | fill `CLAUDE.md`; import solution architecture (adopt-or-scaffold); ingest estimate (optional; NFRs+Assumptions only); seed `ROADMAP.md` shell |
| 3 Repos | `/setup-workspace` — create (template/exemplar, high-level config only) or clone + index existing |
| 4 Tracker | create or fetch the tracker project |
| 5 Rules | `/propagate-rules` — project-wide conventions stay here; repo/language-specific rules → each repo via PR |
| 6 Check | `/check-setup` — audit every phase artifact |

`docs/conventions/` are project-wide (this repo). Repo/language-specific rules live in each service
repo, delivered by `/propagate-rules`. Per-repo navigation indexes live in `docs/repos/<repo>.md`.

## Template maintenance

Downstream project docs repos are clones of `dac-docs-template` and drift as they add project
context, specs, and arch content. To pull template-generic improvements (new skills, updated
process rules, fixed conventions) back into a project docs repo, run `/sdd-rebase` — it fetches
the upstream template via a `template` git remote, copies files that belong to the template
(skipping a short exclude list of project-owned paths like `tracker.json`, `.mcp.json`,
`docs/architecture/`, `docs/superpowers/`, `ROADMAP.md`), and opens a sync PR against the project
default branch. The PR is the review step — the human decides what lands.

## Sprint-driven ROADMAP growth

`ROADMAP.md` starts empty after setup and grows one sprint at a time. At sprint boundary the DM
runs `/plan-sprint`, which reads the modularity tree, shows candidate scope, asks the DM to pick
the sprint's contents (from tree L2 nodes and/or ad-hoc items), creates the tracker milestone, and
appends rows to `ROADMAP.md`. No feature rows are seeded at project setup and no tracker milestones
are pre-created.

See `.claude/skills/plan-sprint/SKILL.md` for the flow. ROADMAP rows may originate from the tree
(with a `Module` slug) or from planning discussion (blank `Module`) — the tree is a helper, not the
only source of ROADMAP entries.

## Per-feature flow

```
select feature (from ROADMAP or ad-hoc during /plan-sprint)
  │
  ├─ /create-issues <name>    → one tracker issue per feature (planning-time; called by
  │                            /plan-sprint's Y/n prompt, or standalone per row)
  │
  ├─ if capability not yet in solution architecture:
  │    /new-feature   → docs/architecture/current/** + docs/architecture/logs/…
  │                   → arch PR → /arch-log-review → human merge  ← GATE
  │
  ├─ /new-spec        → docs/superpowers/specs/YYYY-MM-DD-<name>-design.md
  │                   → spec PR → /spec-review (unified: structure + naming + arch + contracts)
  │                   → human merge  ← GATE
  │                   → /arch-review <feature>  (optional, post-approval — sync current/)
  │
  ├─ Devs implement in workspace/<repo>, code PRs reviewed & merged
  │  ↳ /split-issue <id> if a dev/reviewer wants smaller PRs mid-flight (tracker-only)
  │
  └─ all child issues close → feature done → ROADMAP updated
```

### Two entry points

Feature work has two entry commands. Use whichever matches your intent:

- **`/new-feature <name>`** — evolve `docs/architecture/current/`: add a new capability, edit
  NFRs/assumptions, or introduce an L1/L2 tree node. Opens an **arch PR** gated by
  `/arch-log-review`. Never writes a feature spec under `docs/superpowers/specs/`.
- **`/new-spec <name>`** — initialize a feature spec from an existing arch node. Opens a
  **spec PR** gated by the unified `/spec-review` (structure + naming + architecture + contracts).
  Never touches `docs/architecture/`. Post-approval, `/arch-review <name>` can be run to sync
  `docs/architecture/current/` with the spec on its own PR.

When a feature needs both, the arch PR merges first, then `/new-spec` runs and its spec PR
references the newly-merged arch. `/new-spec` blocks with an instruction to run `/new-feature`
first when the target capability is a novelty (not in `current/`).

Task specs / sub-tasks below the feature spec are optional and **not reviewed**. Do not run the full
flow for every ticket — it is overkill for simple work and wastes tokens. Simple follow-up tickets are
usually C0 or fold into the feature's existing C1/C2 spec.

## Roles

| Role | Responsibility |
|------|-----------------|
| **DM** (Delivery Manager) | Owns `ROADMAP.md`, client communication, delivery sign-off. |
| **PL** (Project Lead) | Sets up repos and CI/CD, reviews and merges spec PRs (may delegate). |
| **Dev** | Implements child issues in service repos, opens code PRs. |

## Tracker adapters

The issue tracker and the git host are independently configured (see `.claude/tracker.json`) — Plane,
for instance, is issues-only, so repos still live on GitHub or GitLab. Tracker-touching commands
(`/create-issues`, `/roadmap-sync`, `/status`, `/setup-project`, `/check-setup`) branch on
`tracker.platform`; repo/rule provisioning (`/setup-workspace`, `/propagate-rules`, `/check-setup`)
branches on `gitHost.platform`. Each platform maps to the same concepts through different verbs:

| Concept | Plane | GitHub | GitLab |
|---------|-------|--------|--------|
| Ticket | work item | issue | issue |
| Grouping by timebox | cycle (≈ milestone) | milestone | milestone |
| Grouping by theme | epic | milestone + sub-issues, or a tracking issue | epic |
| Sub-ticket | work item with `parent` | sub-issue attached via `/sub_issues` API | issue linked via `relates_to` |

Adding a platform later means adding one new adapter section per command — no rewrites to the
command's flow or to this document.
