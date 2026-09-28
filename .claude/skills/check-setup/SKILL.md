---
name: check-setup
description: Read-only audit that every onboarding phase completed — config, MCP connectivity, repos, tracker, rules
disable-model-invocation: true
---

# /check-setup

## Purpose

Onboarding Phase 6. Read-only completeness audit. Reports ✅ / ⚠️ / ❌ per phase artifact with the exact
command to fix each gap. Changes nothing.

## Inputs

None. Reads the config files, `workspace/`, `docs/repos/`, `ROADMAP.md`, and (read-only) the
tracker/git-host MCPs.

## Checks

1. **Config present & non-placeholder** — `.claude/tracker.json` has real `tracker` + `gitHost` values
   (no `plane|github|gitlab` / `...` placeholders); `.env` exists with the chosen platforms' keys set;
   `.mcp.json` contains only the chosen platform block(s). Fix: `/setup-project` (Phase 0).
2. **MCP connectivity** — probe each configured MCP server with one read call; report ✅ / ❌. Fix:
   check `.env` tokens and `uv` on PATH (Plane), then re-run `/setup-project`.
3. **Docs-repo identity** — `git remote -v` origin is not the template, and the docs repo exists on the
   git host. Fix: `/setup-project` (Phase 1).
4. **Architecture schema conformance** — read the schema at
   `@.claude/skills/check-setup/arch-schema.md` and enforce its four-check gate:
   - **Presence** — all 20 required files exist (17 content + 3 lens READMEs) under
     `docs/architecture/current/`, with ≥1 `ddd/contexts/<name>.md`. Missing = fail.
   - **Non-placeholder** — scan every required section (per the schema) for `ARCH_GAP` markers;
     each is a failure. The literal `N/A because <reason>` with ≥5-word reason counts as filled.
   - **Cross-references** — the 10 enforced rules in the schema (`## Cross-reference rules` →
     "Enforced"). Dangling link = fail.
   - **Estimation coverage** — only if `estimation/` exists. Every L2 capability from
     `bizbok/capabilities.md` must appear in ≥1 file inside `estimation/`.

   Also **warn (do not fail)**:
   - `_migration-quarantine/` is non-empty (invoke `/migrate-arch` review or manually empty).
   - Any `modularity/<L1>/README.md` missing the soft-convention header line.

   Fix hint: for a fresh scaffold, `/setup-project` (Phase 2). For a brownfield input, run
   `/migrate-arch`. For unfilled `ARCH_GAP` markers, fill them manually or via `/new-feature`.

5. **Initial arch-log entry present** — at least one
   `docs/architecture/logs/YYYY-MM-DD-*/README.md` exists. Fix: `/setup-project` (Phase 2) creates
   an initial-scaffold log entry.
6. **Repos cloned & indexed** — every repo has a `workspace/<repo>/` checkout, a `docs/repos/<repo>.md`
   index, and a row in `workspace/INDEX.md`. Fix: `/setup-workspace` (or `--refresh`).
7. **Tracker project** — the configured `tracker.projectId` resolves on the tracker. Fix:
   `/setup-project` (Phase 4).
8. **ROADMAP shell present** — `ROADMAP.md` exists with the header and the empty rows table using
   the columns `Feature | Priority | Level | Module | Spec | Issues | Milestone | Link`. Rows may
   be empty; this is expected before the first `/plan-sprint`. Fix: `/setup-project` (Phase 2).
9. **Rules propagated** — each service repo has agent rules referencing `docs/conventions/`. Fix:
   `/propagate-rules`.

## Output

A checklist, one line each `✅ | ⚠️ | ❌ <check> — <detail> [fix: <command>]`, then a one-line
summary `N/9 checks pass`.

Where check 4 (architecture schema conformance) fails, the report expands with one line per
failed sub-check in the machine-parseable format from the schema doc:

    docs/architecture/current/<path> § <Section>:
      ARCH_GAP — required section unfilled
      Fill with: <hint>
      See: .claude/skills/check-setup/arch-schema.md#<anchor>

or (for cross-reference failures):

    docs/architecture/current/<path> § <Section>:
      Cross-ref violation — <rule N description>
      Fix with: <concrete guidance — usually "add the target file" or "correct the link">
      See: .claude/skills/check-setup/arch-schema.md#cross-reference-rules

## Config

Reads `.claude/tracker.json` (`tracker` + `gitHost`). Read-only; writes nothing.
