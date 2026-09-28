# setup-workspace — Starter-Packs Discovery & Suggestion

**Date:** 2026-08-11
**Status:** Approved design, pre-implementation
**Builds on:** `docs/superpowers/specs/2026-07-22-greenfield-brownfield-onboarding-design.md` (Phase 3 —
repos, delegated by `/setup-project` to `/setup-workspace`).

## Purpose

Today `/setup-workspace` asks the user, per repo, to paste a URL for one of three routes (new from
template / new from exemplar / existing). The user always has to know which URL to paste. This
design adds an org-wide template catalog: before the loop starts, the skill queries
`gitlab.dac.digital/dacdigital/starter-packs`, and for each repo it suggests the best-matching
template — a Y/Enter confirms; anything else (pick from list, custom URL, existing repo) is one
keystroke away.

## Scope

- **In:** changes to `.claude/skills/setup-workspace/SKILL.md` — a pre-loop discovery step, a
  modified per-repo prompt, ranking heuristic, cross-host template mechanics, fallbacks.
- **Out (YAGNI):** any new tag/manifest convention on starter-packs (matching uses repo name +
  `README.md` only); cross-run caching of the catalog; publishing/curating new starter-packs;
  changes to `/setup-project` (still delegates to `/setup-workspace` as today).

## Writing principle

Same as the base designs: concise, technical, complete. The SKILL.md file is the execution
adapter; this design is the source of truth for the change.

## Part 1 — Pre-loop discovery

Before the per-repo loop starts, `/setup-workspace` builds an in-memory **template catalog**:

1. If a GitLab MCP is configured and reachable, list projects in the `dacdigital/starter-packs`
   group. For each project, fetch: name, description, `README.md`. Cache all three in memory for
   the run.
2. If the GitLab MCP is missing or the group is unreachable/empty, warn once —
   `starter-packs unavailable — falling back to manual URLs` — and continue with an empty catalog.

Discovery runs regardless of the project's target `gitHost` (from `.claude/tracker.json`). The
starter-packs group is the org's template library and is used cross-host: a GitLab template can
seed a GitHub target repo (see Part 3).

Cost is bounded — one small group, a handful of templates — so eager fetching is fine. No
persistent cache; every `/setup-workspace` run re-fetches.

## Part 2 — Modified per-repo interaction

Per-repo prompts are unchanged for `name`, `type` (frontend / backend / ML / infra / …), and
`language/stack`. What changes is the route pick that follows.

### Ranking

If the catalog is non-empty, rank templates against the user-supplied `(type, language)` using
these signals, strongest → weakest:

1. **Explicit type token in the repo name** (e.g. `frontend-*`, `backend-*`, `ml-*`).
2. **Language/stack token in the repo name** (e.g. `*-next-app`, `*-spring`, `*-fastapi`).
3. **Keyword hits in the GitLab project description**.
4. **Keyword hits in the `README.md`** — headings + first paragraph.

Break ties by signal strength in that order. The top-scoring template is the **starred
suggestion**. If no template scores above zero (e.g. `type = infra` with no matching starter), no
star; show the list unranked.

### Prompt

Print all templates in ranked order, mark the top with `★`, include the GitLab description as a
one-line hint:

    Templates from dacdigital/starter-packs (best match ★):
      ★ 1. frontend-next-app     Next.js 15 + TS + Tailwind starter
        2. frontend-react-vite   React 19 + Vite SPA starter
        3. backend-spring        Spring Boot 3 REST service
        4. ml-python-fastapi     Python 3.12 + FastAPI + Poetry

    [Enter] accept ★ | <n> pick # | t <URL> template | x <URL> exemplar | e <URL> existing | s skip

Routing the choice into the existing three routes:

| Input | Route |
|-------|-------|
| `Enter` (or `<n>`) | New-from-template using the picked starter-pack |
| `t <URL>` | New-from-template using an arbitrary URL |
| `x <URL>` | New-from-exemplar (today's high-level-config extraction) |
| `e <URL>` | Existing (clone as-is; no new-repo creation) |
| `s` | Skip templates: create an empty repo with no high-level config seeded (user seeds later or in a follow-up run) |

If the catalog is empty, the prompt collapses to today's `t | x | e | s` ask — no list, no
suggestion — preserving current behavior.

## Part 3 — Cross-host template mechanics

Because starter-packs is always on GitLab but the target `gitHost` may be GitHub:

1. **Same host, native support** — prefer the platform's "use as template" / fork API (GitLab
   fork, GitHub template repo).
2. **Cross host, or no native support** — shallow-clone the template locally, create the target
   repo on `gitHost`, repoint `origin`, squash-push a single
   `Init from template <template-name>` commit. Template history is dropped intentionally — a
   starter is a starting point, not a fork.

The **high-level config filter** (see today's step 3 in `SKILL.md`) applies only to the
new-from-exemplar route. Templates ship whatever they ship — no filtering.

## Part 4 — Backward compatibility

- Any user pasting a URL still bypasses discovery entirely — today's flow, unchanged.
- Catalog failures never block the per-repo loop; they degrade to today's prompt.
- No changes to route mechanics after a URL is chosen — same clone, push, index, and
  `workspace/INDEX.md` regeneration.

## Part 5 — SKILL.md edits (concrete)

Replace step 2 of `.claude/skills/setup-workspace/SKILL.md` with:

- A new **step 1a (pre-loop)** covering Part 1 above (discovery + catalog).
- A rewritten **step 2 (loop)** covering Part 2 above (ranking + prompt + routing).

Retain steps 3–7 as-is (high-level config filter, cloning, per-repo indexing, `INDEX.md`
regeneration, gitignore reminder). Add one sentence to step 3 clarifying that the filter runs
only on the exemplar route (see Part 3).

## Open questions

None blocking implementation. Two low-priority follow-ups worth noting:

- Should the ranker preview the top match's README excerpt inline before the prompt (a few lines
  under the ★ entry)? Punt to post-implementation if users still find it hard to choose.
- Should a `?` command open the top match's README in a pager for preview? Same — punt.
