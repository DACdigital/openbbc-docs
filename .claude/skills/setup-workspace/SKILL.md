---
name: setup-workspace
description: Provision service repos into workspace/ (create from template/exemplar or clone existing), index each, and regenerate workspace/INDEX.md
disable-model-invocation: true
---

# /setup-workspace

## Purpose

Onboarding Phase 3. Provisions service repos one at a time, for greenfield (create) and brownfield
(adopt) projects alike. New repos are created on the git host and seeded with **high-level config
only**; existing repos are cloned and **indexed** so Claude can navigate them quickly. All repos land
under `workspace/<repo>/`; `workspace/INDEX.md` and the per-repo `docs/repos/<repo>.md` files are
regenerated.

## Inputs

None required — interactive, repeated per repo. Reads `.claude/tracker.json` → `gitHost`. Run
`/setup-project` first if `gitHost` is missing or a placeholder. Optional `--refresh` argument: skip
provisioning; only re-index repos already cloned under `workspace/`.

A configured **GitLab MCP** (any org) is a **soft dependency** for template discovery against
`gitlab.dac.digital/dacdigital/starter-packs`. If it is missing or unreachable, the skill warns once
and falls back to today's manual URL flow. GitLab discovery runs regardless of `gitHost`.

## Steps

1. **Read `.claude/tracker.json` → `gitHost`** (platform, org). If missing or a placeholder, stop and
   tell the user to run `/setup-project` first.
2. **Build the template catalog (once per run).** If a GitLab MCP is configured and reachable, list
   projects in the `dacdigital/starter-packs` group. For each project, fetch: name, description, and
   `README.md`. Cache all three in memory as the run's **template catalog**. If GitLab MCP is missing,
   the group is unreachable, or the group is empty, warn once —
   `starter-packs unavailable — falling back to manual URLs` — and continue with an empty catalog.
   Discovery runs regardless of the project's target `gitHost`; the starter-packs group is the org's
   cross-host template library. Skip this step when `--refresh` was passed.
3. **Loop over repos**, one at a time, until the user says they're done. For each, ask its **name**,
   **type** (frontend / backend / ML / infra / …), and **language/stack**. Then present the route
   picker:
   - **If the catalog is non-empty**, rank templates against `(type, language)` — strongest to
     weakest: (a) explicit type token in the repo name (e.g. `frontend-*`, `backend-*`, `ml-*`);
     (b) language/stack token in the repo name (e.g. `*-next-app`, `*-spring`, `*-fastapi`);
     (c) keyword hits in the GitLab project description; (d) keyword hits in `README.md` (headings
     + first paragraph). Break ties in that order. Print all templates in ranked order, one per
     line, with the top match starred (★) and the description as a one-line hint. If nothing scores
     above zero, print the list unranked with no star.
   - **Prompt:** `[Enter] accept ★ | <n> pick # | t <URL> template | x <URL> exemplar | e <URL>
     existing | s skip`. If the catalog is empty, the prompt collapses to `t | x | e | s`.
   - **Routes:**
     - **New from template** (`Enter`, `<n>`, or `t <URL>`) — instantiate a new repo from the
       chosen template. Same-host with native "use as template" / fork support: prefer the platform
       API. Cross-host or no native support: shallow-clone the template locally, create the target
       repo on `gitHost`, repoint `origin`, squash-push a single `Init from template <template-name>`
       commit (template history is dropped intentionally).
     - **New from exemplar** (`x <URL>`) — ask for an existing repo to learn from; create an empty
       target repo, then copy across only high-level config extracted from the exemplar (see next
       step).
     - **Existing** (`e <URL>`) — the repo already exists on the git host; clone it as-is; do not
       create a new repo.
     - **Skip** (`s`) — create an empty target repo on `gitHost` with no high-level config seeded.
       The user seeds config later or in a follow-up `/setup-workspace` run.
4. **High-level config (exemplar route only).** Templates ship intentional content — no filtering.
   This step applies only to the **New from exemplar** route. Keep only toolchain and quality
   config — never application code, dependency lockfiles, secrets, or content. Copy without asking:
   `.editorconfig`, `.gitattributes`, `.gitignore`, tool/language version pins (`.tool-versions`,
   `.nvmrc`, `.node-version`, `.sdkmanrc`), CI config (`.github/workflows/*`, `.gitlab-ci.yml`),
   linter/formatter config (eslint / prettier / ruff / checkstyle / pmd / spotless), `Dockerfile` /
   `.dockerignore`, `renovate.json`, `lombok.config`. For anything **not** on this list, ask the user
   whether it belongs before copying. Agent rules (`AGENTS.md` / `CLAUDE.md` / `.claude`) are **not**
   copied here — they come from `/propagate-rules`.
5. **Clone every repo** (created or existing) into `workspace/<repo>/`.
6. **Index each repo** → `docs/repos/<repo>.md` using the template below, by inspecting the checkout
   (stack, layout, entry points, build/test/run commands, detected conventions). `--refresh` runs only
   this step and step 7 over the already-cloned repos.
7. **Regenerate `workspace/INDEX.md`** — the overview table
   `Repo | Type | Language | Branch | Last sync | Index`, where `Index` links to
   `docs/repos/<repo>.md`, `Branch` is the checked-out branch, and `Last sync` is this run's date.
8. Remind the user: `workspace/*` is gitignored except `INDEX.md`; `docs/repos/*` **is** committed.

## Per-repo index template (`docs/repos/<repo>.md`)

    # <repo> — index

    **Type:** <frontend | backend | ML | infra | …>
    **Language / stack:** <e.g. Java 21 / Spring Boot>
    **Git host:** <platform>/<org>/<repo>
    **Last indexed:** <YYYY-MM-DD>

    ## Layout
    - <top-level dir> — <one-line purpose>

    ## Entry points
    - <bootstrap / main files or service entry modules>

    ## Commands
    - Build: <cmd>
    - Test: <cmd>
    - Run (local): <cmd>

    ## Conventions detected
    - <build tool, test framework, lint / format, CI, notable patterns>

## Config

Reads `.claude/tracker.json` → `gitHost` (platform + org). Does not touch `tracker` or `comms`.
