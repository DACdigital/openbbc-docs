---
name: check-setup
description: Read-only audit that every onboarding phase completed — config, client resolution, repos, tracker, rules
disable-model-invocation: true
---

# /check-setup

## Purpose

Onboarding Phase 6. Read-only completeness audit. Reports ✅ / ⚠️ / ❌ per phase artifact with the exact
command to fix each gap. Changes nothing.

## Inputs

None. Reads the config files, `workspace/`, `docs/repos/`, `ROADMAP.md`, and (read-only) the
tracker / git-host / comms clients per
[Client resolution](../../../docs/process/README.md#client-resolution).

## Checks

1. **Config present & non-placeholder** — `.claude/tracker.json` has real `tracker` + `gitHost` values
   (no `plane|github|gitlab` / `...` / `https://...` placeholders), each with an `instanceUrl` (a file
   without it is from before client resolution — fix: re-run `/setup-project`); `.mcp.json` contains
   only blocks for no-CLI tracker platforms, with secrets only as `${VAR}` (a literal-looking token
   is ❌ — it is committed; rotate it); `.claude/clients.local.json` is gitignored
   (`git check-ignore` succeeds). A leftover `.env` is ⚠️, not ❌: "`.env` is not read by anything —
   move its values into your shell environment". Fix: `/setup-project` (Phase 0).
2. **Client resolution** — for each configured role (`tracker`, `gitHost`, `comms`), run
   [Client resolution](../../../docs/process/README.md#client-resolution) in **read-only mode**: no
   prompt, no writes. Report per role ✅ tier / client / scope / host, or ❌. Then compare with
   `.claude/clients.local.json`: a missing entry, a different client, or an `instanceUrl` that differs
   from `tracker.json` is ⚠️ drift — fix: re-run `/setup-project` (Phase 0). A tier-2 candidate the
   user has not been offered yet is reported as ⚠️ "your own `<name>` could serve this role".

   On ❌, name the real cause before blaming the token — the server's error text often reads the same
   for all of them:

   | Condition | Report | Fix |
   |---|---|---|
   | CLI found but logged in to a different host | client bound to `<host>`, project uses `<instanceUrl>` | `glab auth login --hostname <host>` / `gh auth login --hostname <host>` |
   | MCP server answers, but from a different workspace or host | **server answers from the wrong instance** — green for you, broken for the team | point that server at `<instanceUrl>`, or remove it so the next tier applies |
   | `claude mcp list` shows a missing-variable warning for the project server | **variable not set in Claude Code's environment — the credential never reached the server** | `export PLANE_API_KEY=…` in the shell or app that launches Claude Code; Claude Code does **not** read `.env` |
   | variable set, call still fails | genuine auth failure | check the token's validity and scopes, `uv` on PATH (Plane) |

   Do **not** infer failure from `${…}` text in `claude mcp get` output: it prints values unexpanded
   even when expansion works. The missing-variable warning and the read probe are the signals.
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
   `/migrate-arch`. For unfilled `ARCH_GAP` markers, fill them manually or via `/arch-change`.

5. **Initial arch-log entry present** — at least one
   `docs/architecture/logs/YYYY-MM-DD-*/README.md` exists. Fix: `/setup-project` (Phase 2) creates
   an initial-scaffold log entry.
6. **Repos cloned & indexed** — every repo has a `workspace/<repo>/` checkout, a `docs/repos/<repo>.md`
   index, and a row in `workspace/INDEX.md`. Fix: `/setup-workspace` (or `--refresh`).
7. **Tracker project** — the configured `tracker.projectId` resolves on the tracker, read through the
   `tracker` client from check 2 (skipped with ⚠️ if that role did not resolve). Fix:
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
