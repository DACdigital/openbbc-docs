# Unified Arch Log + Layer Discipline (drop `decisions.md`)

**Date:** 2026-08-12
**Status:** Approved design, pre-implementation
**Builds on:** sub-projects (1)–(4) of the initiative (all merged). This is sub-project (5) — a
follow-up to unify the two coexisting decision-log patterns into one.

## Purpose

After sub-project (1) shipped `docs/architecture/logs/YYYY-MM-DD-<codename>/` as an
append-only, PR-gated log for architecture changes, the older per-feature
`docs/<feature>/decisions.md` pattern kept coexisting. The two logs cover the same conceptual
object ("decisions worth remembering") at different scopes and with different formats and
enforcement.

This sub-project unifies them under the arch-log pattern. Rationale:

- **`docs/architecture/current/` is the shape of the product** — not infra, not devops. Every L1/L2
  feature adds or changes product shape; feature decisions therefore belong in the arch log, not a
  parallel per-feature file.
- **Two decision-log formats in one repo is cognitive overhead** for anyone reading the process.
- **Enforcement asymmetry**: the arch log is PR-gated; `decisions.md` is trust-based and
  frequently skipped. One rule for all decisions is stronger.

## Design principles (from the brainstorm)

1. **Layer discipline.** `docs/architecture/current/` describes business capabilities, services and
   their responsibilities, contracts (APIs/events/data ownership), and deployment topology at the
   boxes-and-arrows level. It does **not** describe dep-version bumps, CI configuration,
   docker-compose image bumps, lint rules, framework upgrades that don't change any contract,
   code-level refactors, or small bug fixes — those are per-spec or per-repo concerns.
   Heuristic: **"if the change would still matter to someone reading the product plan a year from
   now, it's architecture; if it's noise a year from now, it's implementation."**
2. **One log system.** `docs/<feature>/decisions.md` goes away. All decisions worth persisting
   land in `docs/architecture/logs/YYYY-MM-DD-<codename>/README.md`.
3. **`/new-feature` always proposes a `current/` update (Shape A).** The skill asks whether the
   feature adds or changes anything in `current/`. If yes → writes spec + `current/` diff + paired
   arch-log entry in the same PR. If no → writes only the spec (no `current/` change, no
   arch-log entry). Auto-detection of matching content in `current/` informs the Q&A but doesn't
   short-circuit it.
4. **Direction of influence rule remains intact.** The docs → tracker / service-repos rule from
   sub-project (3) applies unchanged.

## Scope

- **In:** dropping `decisions.md` from the repo layout, process docs, and `/new-feature` skill;
  extending `/new-feature` to Shape A (propose `current/` update + draft paired arch-log entry);
  adding a layer-discipline hard rule to `AGENTS.md`; extending the `architecture-log-review` agent
  with a layer-discipline check; adding the layer-discipline heuristic to the scaffolded
  `current/README.md` (via `/setup-project` Phase 2).
- **Out (YAGNI):** any migration tool for existing `docs/<feature>/decisions.md` files in
  downstream projects (users do that themselves if they want); any deprecation of
  `docs/architecture/current/` in favor of a different name; changing the arch-log format;
  changing the modularity tree.

## Part 1 — Drop `decisions.md`

Remove all references to `docs/<feature>/decisions.md`:

- **`README.md`** — repo-layout line `docs/<feature>/  spec.md, decisions.md, optional plan.md`
  becomes `docs/<feature>/  spec.md, optional plan.md`.
- **`docs/process/README.md`** — the sentence `The one exception is docs/<feature>/decisions.md:
  it is a decision log, not a review verdict, and it is committed` is removed. In its place, a
  short pointer: architecture decisions live in `docs/architecture/logs/`, no other in-repo
  decision-log format exists.
- **`docs/process/AGENTS.md`** — the entire `## decisions.md entry format` section (Decision /
  Context / Rationale / Alternatives rejected schema) is removed. The `decisions.md is the one
  exception, and it IS committed` hard rule is removed.
- **`.claude/skills/new-feature/SKILL.md`** — the `Seed docs/<name>/decisions.md` step is removed;
  the `commit docs/<name>/{spec.md,decisions.md[,plan.md]}` list becomes
  `commit docs/<name>/{spec.md[,plan.md]}` and adds `docs/architecture/**` when the run touched
  `current/`.

## Part 2 — Layer discipline rule and heuristic

### AGENTS.md hard rule (new)

Add a hard rule under the arch-log-related rules:

> **Architecture layer discipline.** `docs/architecture/current/` describes the product at the
> business / service / contract level: business capabilities, services and their responsibilities,
> contracts between them (APIs, events, data ownership), and deployment topology at the
> boxes-and-arrows level. It does **not** describe implementation choices such as dependency
> versions, docker-compose image bumps, CI configuration, lint rules, framework upgrades that
> don't change any contract, code-level refactors, or small bug fixes. Rule of thumb: if the
> change would still matter to someone reading the product plan a year from now, it belongs in
> `current/`; if it's noise a year from now, it belongs in a spec, a PR description, or a code
> comment. `architecture-log-review` enforces this.

### `current/README.md` scaffold — new callout

`/setup-project` Phase 2 scaffold for `docs/architecture/current/README.md` gains a short callout
at the top (before the `## Context` section):

    > **Scope of this vault.** This describes the *shape of the product* — business
    > capabilities, services and their responsibilities, contracts (APIs / events / data
    > ownership), and deployment topology at the boxes-and-arrows level. It does not describe
    > implementation choices (dep versions, CI, lint, docker-compose image bumps, code-level
    > refactors). Rule of thumb: if a change matters a year from now to someone reading the
    > product plan, it belongs here; otherwise it belongs in a spec or a PR description.

Adopt route (import): the callout is not injected on adopt runs (user's existing vault is
respected as-is). Scaffold route (starter): callout is included.

### `architecture-log-review` agent — new check

Append a check to the agent's Checks list:

> **Layer discipline.** The change described by `Impact` is at business / service / contract /
> deployment-topology level. Changes limited to dependency versions, docker-compose or infra
> image bumps, CI/lint/test tooling, or code-level refactors are a fail — those are per-spec or
> per-repo concerns, not architecture.

Verdict severity: FAIL when the change is clearly implementation-detail; PASS_WITH_ISSUES when
mixed (some arch, some implementation — user should split into arch entry + separate change).

## Part 3 — `/new-feature` Shape A rework

`.claude/skills/new-feature/SKILL.md` gets a new step and drops the decisions.md seed. Approximate
step-list after the rework:

1. **Validate `<name>`** and check `ROADMAP.md` for a matching row (unchanged).
2. **Load context files** (unchanged): read `nfrs.md`, `assumptions.md`, and if `Module` matches
   the modularity node's `README.md`. Also grep `current/` (excluding `modularity/`) for
   references to `<name>` and load matching files.
3. **Run `superpowers:brainstorming`** (unchanged).
4. **Write `docs/<name>/spec.md`** (unchanged).
5. **Propose `current/` update.** Ask the user: "Does this feature add or change anything in
   `docs/architecture/current/`?" Present any matched content from step 2 as context ("here's
   what's already described related to this feature").
   - **If yes** — mini Q&A: which files/pages to add or modify, what content. Write the diff to
     `current/` files. Draft the paired arch-log entry at
     `docs/architecture/logs/YYYY-MM-DD-<name>/README.md` with Driver = "new feature: <name>",
     Decision = "introduce <name> into product shape", Impact = list of `current/` files touched,
     Links = `docs/<name>/spec.md`.
   - **If no** — skip. No `current/` write, no arch-log entry. Spec references existing `current/`
     content.
6. **Propose the level** (was step 5).
7. **Optionally run `superpowers:writing-plans`** (was step 6).
8. **Open the spec PR** (was step 7): commit `docs/<name>/{spec.md[,plan.md]}` plus, if step 5
   touched them, the relevant `docs/architecture/current/**` files and the new
   `docs/architecture/logs/YYYY-MM-DD-<name>/README.md`.
9. **Post the notification** (was step 8; unchanged).

Removed: former step 4 that seeded `docs/<name>/decisions.md`. Renumbered accordingly.

## Part 4 — `/check-setup` — no change

No new check needed. Layer discipline is enforced at PR time by `architecture-log-review`.
`/check-setup` already validates the arch-vault, initial log entry, and NFRs+Assumptions files.

## Part 5 — Ripples (file list)

- **Modify** `.claude/skills/new-feature/SKILL.md` — remove decisions.md seed, add
  current/-update step, restructure commit step.
- **Modify** `.claude/skills/setup-project/SKILL.md` — add layer-discipline callout to the
  scaffold in Phase 2.
- **Modify** `.claude/agents/architecture-log-review.md` — append the layer-discipline check.
- **Modify** `docs/process/AGENTS.md` — remove the `decisions.md entry format` section; remove
  the `decisions.md is the one exception` hard rule; add the layer-discipline hard rule.
- **Modify** `docs/process/README.md` — remove the `decisions.md` sentence in "Where each gate
  happens".
- **Modify** `README.md` — layout line drops `decisions.md`.

Six files. No new skill files, no new agents.

## Part 6 — Migration guidance (informative, not enforced)

Downstream projects that already adopted the template and have existing `decisions.md` files
face a one-time cleanup:

- Keep existing `docs/<feature>/decisions.md` files as historical artifacts (no new entries added
  under this process).
- For future feature work, the arch log carries the record.
- Optionally, migrate the highest-value entries into arch-log entries — but this is voluntary.

The template doesn't ship any migration tool.

## Open questions

None blocking implementation.
