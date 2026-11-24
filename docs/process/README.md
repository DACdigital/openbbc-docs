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
| **C1 — normal** | typical functional change; no impact on architecture/DB/API/events | spec → spec-review → (optional local plan) → issues → implement → code PR review |
| **C2 — architectural/data/contract** | new bounded context; DB/API/event/contract change; tenant isolation | same as C1 — the unified `/spec-review` covers architecture and contracts inline |

Rules:
- Ambiguous → pick the **higher** level.
- A reviewer can **raise** the level, never lower it.
- The fast lane is **C0-only** — never for changes touching DB, migrations, API, events, security,
  tenant isolation, or upgrades.
- A new implementation plan is a local working note. It is never committed, in any repo.

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
- **Architecture-log gate (any change touching `docs/architecture/current/`)** — happens
  **here**, gating `/arch-change`'s push and (if invoked by hand) an already-open arch PR.
  A passing verdict ends up as a PR comment (posted by `/arch-change` when gated pre-push, by
  `/arch-log-review` itself on-PR); a `FAIL` is reported to the author in-session and never
  posted. Independent of C1/C2 — a pure refactor still triggers it.
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

**`/arch-log-review` also runs before the push.** `/arch-change` commits locally and runs it in
pre-push mode against the branch diff; on `FAIL` nothing leaves the machine, on `PASS` /
`PASS_WITH_ISSUES` the arch PR opens carrying that verdict as its first comment. Branches touching
`docs/architecture/current/` that were not opened by `/arch-change` (a standalone `/modularize`
run, an `/arch-review` sync PR, a manual edit) run it on-PR by hand.

Throughout, **a `FAIL` verdict is never posted**. Only `PASS` and `PASS_WITH_ISSUES` reach the
PR, so a spec PR carries the verdict a human is asked to act on rather than the whole history of
drafts that preceded it. A gate invoked by hand on an already-open PR follows the same rule: a
`FAIL` is reported to the author in the session and kept off the PR.

## Architecture log entries

Every change to `docs/architecture/current/` adds exactly one log entry in the same PR, at
`docs/architecture/logs/YYYY-MM-DD-<codename>/README.md`. This section is the normative format;
`architecture-log-review` checks entries against it.

    # <codename> — <one-line title>

    **Date**: YYYY-MM-DD
    **Codename**: <slug, matches dir name>

    **Driver**: <what prompted this change — `new: …` or `change: …` when written by /arch-change;
    otherwise the external event, refactor need, feature spec, incident or /modularize run>

    **Decision**: <what was decided, one or two sentences>

    **Rationale**: <why this choice over alternatives>

    **Alternatives rejected**: <what else was considered and why it lost — or "None — this change
    aligns the doc with reality.">

    **Impact**: <bulleted list of the specific files in `current/` this change touches, and any
    downstream service repos affected>

    **Links**:
    - <spec PR / tracker item / external doc that motivated or documents this>

Rules:

- **Codename:** short human handle in the dir name, lowercase kebab-case `[a-z0-9-]+`, max ~40
  chars, e.g. `2026-08-12-split-billing-context/`. The date prefix keeps ordering; the codename need
  not be globally unique.
- **Append-only:** log directories, once merged, are never edited or deleted. Corrections come as
  new entries that supersede.
- **Impact is specific:** it lists the `current/` files the change touched; "everything under
  `current/`" without a real reason is not acceptable.
- Additional artifacts (diagrams, exports, screenshots) live next to the entry's `README.md` as
  loose files.
- Enforcement: `/arch-log-review` invokes the `architecture-log-review` agent on any change touching
  `current/` — pre-push from `/arch-change`, or on-PR by hand. A `FAIL` is never posted.

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
| 0 Access | choose tracker / git-host / comms and each instance URL; write `tracker.json` + `.mcp.json`; resolve a client per role (see [Client resolution](#client-resolution)) — every role must resolve to proceed |
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
`docs/architecture/`, `docs/superpowers/`, `ROADMAP.md`), and opens a sync PR
against the project default branch. The PR is the review step — the human decides what lands.

**Template design specs vs. project specs.** Both live in `docs/superpowers/specs/` (plans in
`docs/superpowers/plans/`), where `superpowers:brainstorming` and `/new-spec` write them — template
maintainers use the same tools and paths as project teams. In the template repo those files are the
template's own design history; they are not shipped to projects. `/setup-project` removes every
tracked file under `docs/superpowers/specs/` and `docs/superpowers/plans/` (except `.gitkeep`) in
Phase 1, before the first push, and `/sdd-rebase` never syncs `docs/superpowers/**`. Skills and
agents never read design specs as rules; anything normative lives in `docs/process/`,
`docs/conventions/` or `.claude/skills/check-setup/arch-schema.md`.

Projects set up before Phase 1 did this still carry the template's dated design specs and plans
(`2026-07-15-…` to `2026-10-20-…`). `/sdd-rebase` removes them in the sync PR: it finds
files under `docs/superpowers/` that are byte-identical to the template's copy, so the project's own
specs are never touched. Existing arch-log entries that link one of them keep the link as history
(append-only).

## Sprint-driven ROADMAP growth

`ROADMAP.md` starts empty after setup and grows one sprint at a time. At sprint boundary the DM
runs `/plan-sprint`, which reads the modularity tree, shows candidate scope, asks the DM to pick
the sprint's contents (from tree L2 nodes and/or ad-hoc items), creates the tracker milestone, and
appends rows to `ROADMAP.md`. No feature rows are seeded at project setup and no tracker milestones
are pre-created.

See `.claude/skills/plan-sprint/SKILL.md` for the flow. ROADMAP rows may originate from the tree
(with a `Module` slug) or from planning discussion (blank `Module`) — the tree is a helper, not the
only source of ROADMAP entries.

### ROADMAP schema

One row per feature or task in a planned sprint, columns in this order:

    | Feature | Priority | Level | Module | Spec | Issues | Milestone | Link |

- `Module` — the modularity-tree slug the row derives from; blank for ad-hoc items (bug, tech debt,
  ops task, client ask that doesn't fit the tree).
- `Spec`, `Issues` and `Link` start empty: `/new-spec` sets `Spec`, `/create-issues` fills `Issues`,
  `/roadmap-sync` reconciles `Link` and `Milestone` during planning.

## Per-feature flow

```
select feature (from ROADMAP or ad-hoc during /plan-sprint)
  │
  ├─ /create-issues <name>    → one tracker issue per feature (planning-time; called by
  │                            /plan-sprint's Y/n prompt, or standalone per row)
  │
  ├─ if the solution architecture must change first (new or change):
  │    /arch-change   → docs/architecture/current/** + docs/architecture/logs/…
  │                   → review → /modularize --refine (optional handoff, same log entry)
  │                   → /arch-log-review (pre-push) → arch PR → human merge  ← GATE
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

- **`/arch-change <name>`** — change `docs/architecture/current/`. Asks first: **new** (add
  something not yet there) or **change** (modify, correct, move, retire). Hands tree-shape ops to
  `/modularize --refine` as a default-yes next step. Opens an **arch PR** only after a pre-push
  `/arch-log-review` passes. Never writes a feature spec under `docs/superpowers/specs/`.
- **`/new-spec <name>`** — initialize a feature spec from an existing arch node. Opens a
  **spec PR** gated by the unified `/spec-review` (structure + naming + architecture + contracts).
  Never touches `docs/architecture/`. Post-approval, `/arch-review <name>` can be run to sync
  `docs/architecture/current/` with the spec on its own PR.

When a feature needs both, the arch PR merges first, then `/new-spec` runs and its spec PR
references the newly-merged arch. `/new-spec` blocks with an instruction to run `/arch-change`
(type: new) first when the target capability is a novelty (not in `current/`).

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

### Client resolution

Skills never assume **how** this machine reaches a platform. Each skill that touches the tracker,
the git host or the comms channel obtains a client for that **role** (`tracker`, `gitHost`, `comms`)
through this procedure, and describes its operations neutrally ("list issues in project X", "open a
PR"). Credentials are never written to the repo, never stored by a skill, and never asked for in chat:
they stay where each tool already keeps them — the `gh` / `glab` keyring or config, the shell
environment, or the user's own MCP entries. `.env` is not read by anything.

**Instance identity.** `.claude/tracker.json` carries `instanceUrl` — the web root, no API path, no
trailing slash — for `tracker` and `gitHost`. A client bound to any other host is rejected. The docs
repo itself lives on `gitHost` (`/setup-project` Phase 1 puts it there), so skills acting on the docs
repo's own PRs use the `gitHost` client. Hosts and API URLs are derived, never stored:

| Platform | CLI hostname | API base |
|---|---|---|
| GitHub (`https://github.com`) | `github.com` | `https://api.github.com` |
| GitHub Enterprise (`https://<ghe>`) | `<ghe>` | `https://<ghe>/api/v3` |
| GitLab (any) | hostname of `instanceUrl` | `<instanceUrl>/api/v4` |
| Plane | — | `<instanceUrl>` (the MCP server's `PLANE_BASE_URL`) |

**Adapter table** — the only platform-specific part. Adding a platform adds a row.

| Field | Plane | GitHub | GitLab | Slack | Discord |
|---|---|---|---|---|---|
| CLI + host check | — | `gh auth status --hostname <host>` | `glab auth status --hostname <host>` | — | — |
| MCP identity (command or URL match) | command contains `plane-mcp-server` | command contains `server-github` or `github-mcp-server`, or GitHub MCP URL | command contains `server-gitlab`, or GitLab MCP URL | Slack MCP server or claude.ai Slack connector | Discord MCP server or connector |
| Instance check | probe lists projects of workspace `org` on `instanceUrl` | probe's API base matches the derived API base | probe's API base matches the derived API base | probe can read `channel` | probe can read `channel` |
| Project `.mcp.json` secret | `PLANE_API_KEY` | — | — | — | — |
| Read probe | list workspace projects | get authenticated user | get current user | read channel info | read channel info |

Env-var users of GitHub / GitLab are covered by tier 1: `gh` honours `GH_TOKEN` / `GITHUB_TOKEN`, and
`glab` honours `GITLAB_TOKEN` / `GITLAB_HOST`.

**`resolve(role)`** — platform and `instanceUrl` from `tracker.json`, everything else from the
adapter row. The first tier that passes wins.

1. **CLI** — the adapter has one, it is on PATH, and its host check succeeds for the derived
   hostname. → `cli`.
2. **Your own MCP server or connector** — a server in `claude mcp list` at any non-project scope
   (local, user, plugin, connector, managed) whose command or URL matches the adapter's MCP identity,
   and whose read probe passes the instance check. Ask once: "use your existing `<name>`?" On yes →
   `mcp` with that name and scope, `approved: true`. If the project's `.mcp.json` also defines this
   role, add its name to `disabledMcpjsonServers` in `.claude/settings.local.json` and tell the user
   to restart Claude Code for it to take effect.
3. **Project MCP** — the role's server from `.mcp.json`. `claude mcp list` shows no missing-variable
   warning for it, and its read probe passes. → `mcp`, scope `project`.
4. **None** — stop. List every candidate found with its bound host or workspace and why it was
   rejected (wrong host, `PLANE_API_KEY` unset, probe returned 401, not on PATH). Then give the fix
   for the cheapest tier: `glab auth login --hostname <host>`, `export PLANE_API_KEY=…` in the shell
   or app that launches Claude Code, or `claude mcp add --scope local …`.

Do not judge a project server by the text of `claude mcp get`: it prints `${VAR}` unexpanded even
when expansion works. The missing-variable warning in `claude mcp list` and the read probe are the
signals.

**Modes.** *Interactive* (`/setup-project` and every operating skill) runs the full procedure, may
prompt at tier 2, and writes `clients.local.json` and `disabledMcpjsonServers`. *Read-only*
(`/check-setup`) runs the same tiers with no prompt and no writes, and reports per role the client it
would pick, any tier-2 candidate it would offer, and any difference from `clients.local.json`.

**Saved result — `.claude/clients.local.json`** (gitignored, per machine, never holds a credential):

```json
{
  "tracker": { "tier": "mcp", "client": "plane",           "scope": "project",   "instanceUrl": "https://plane.dac.digital" },
  "gitHost": { "tier": "cli", "client": "glab",                                  "instanceUrl": "https://gitlab.dac.digital" },
  "comms":   { "tier": "mcp", "client": "claude.ai Slack", "scope": "connector", "instanceUrl": null, "approved": true }
}
```

`tier` is `cli` | `mcp`; `client` is the CLI binary or MCP server name; `scope` (MCP only) is `local` |
`project` | `user` | `plugin` | `connector` | `managed`; `instanceUrl` is the `tracker.json` value
the client was checked against (`null` for comms); `approved` marks a tier-2 server the user accepted.
The file lives next to the `tracker.json` actually used. When a skill falls back to a service repo's
own `tracker.json`, it resolves in memory and writes nothing there.

**Using the saved result.**

- At start, read the role's entry. If the file or entry is missing, run `resolve(role)` and write it.
- **Instance drift** — the entry's `instanceUrl` differs from `tracker.json`: re-resolve before use.
- **Self-heal** — the saved client's first call fails with an auth, connection or not-found error:
  run `resolve(role)` once, rewrite the entry, retry once. A second failure is a real error; report
  it. There is no time-based expiry.
- During self-heal, an `approved` tier-2 server that still matches is reused without asking again.
- One client may serve two roles (GitLab as tracker and git host); each role keeps its own entry.
