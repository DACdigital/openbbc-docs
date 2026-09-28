# setup-workspace — Starter-Packs Discovery & Suggestion — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Update `.claude/skills/setup-workspace/SKILL.md` so the skill discovers templates in
`gitlab.dac.digital/dacdigital/starter-packs`, suggests the best match per repo (by type +
language, using name + README), and lets the user confirm with Enter — while preserving today's
manual URL / exemplar / existing routes as fallbacks.

**Architecture:** Docs-only change to a single skill file. No code, no tests. The skill is prose
that guides the model at runtime; verification is a review reading (does the flow make sense, are
all today's routes still reachable, are fallbacks explicit). Spec: `docs/superpowers/specs/2026-08-11-setup-workspace-starter-packs-design.md`.

**Tech Stack:** Markdown. Nothing else.

**Commit policy for this plan:** Per user instruction (2026-08-11), do **not** commit the spec or
plan. The `SKILL.md` edits themselves may still be committed as normal skill changes if the user
opts in later — but skip commit steps unless asked.

---

## File Structure

Only one file is touched:

- Modify: `.claude/skills/setup-workspace/SKILL.md`
  - **Inputs section** — mention GitLab MCP as a soft dependency for template discovery.
  - **Steps section** — insert new step 2 (pre-loop discovery), renumber existing steps, rewrite
    the per-repo loop (previous step 2 → new step 3), amend the high-level config step (previous
    step 3 → new step 4) with a one-sentence clarification.

No other files change. `setup-project/SKILL.md` is not touched; Phase 3 continues to delegate to
`/setup-workspace` unchanged.

---

## Task 1 — Amend Inputs section (soft GitLab MCP dependency)

**Files:**
- Modify: `.claude/skills/setup-workspace/SKILL.md` — Inputs section (lines 17–21)

**Why:** Template discovery calls the GitLab MCP against `dacdigital/starter-packs`. It's a soft
dependency (skill falls back gracefully if unavailable), but the Inputs section should mention it
so a reader knows why GitLab access matters even when the project targets GitHub.

- [ ] **Step 1: Replace the Inputs paragraph**

Find:

    None required — interactive, repeated per repo. Reads `.claude/tracker.json` → `gitHost`. Run
    `/setup-project` first if `gitHost` is missing or a placeholder. Optional `--refresh` argument: skip
    provisioning; only re-index repos already cloned under `workspace/`.

Replace with:

    None required — interactive, repeated per repo. Reads `.claude/tracker.json` → `gitHost`. Run
    `/setup-project` first if `gitHost` is missing or a placeholder. Optional `--refresh` argument: skip
    provisioning; only re-index repos already cloned under `workspace/`.

    A configured **GitLab MCP** (any org) is a **soft dependency** for template discovery against
    `gitlab.dac.digital/dacdigital/starter-packs`. If it is missing or unreachable, the skill warns
    once and falls back to today's manual URL flow. GitLab discovery runs regardless of `gitHost`.

- [ ] **Step 2: Read back the updated section**

Read lines 17–28 of `.claude/skills/setup-workspace/SKILL.md`. Confirm:
- The original paragraph is intact.
- The new soft-dependency paragraph follows immediately after.
- No stray blank lines or accidental heading breaks.

- [ ] **Step 3: Commit (SKIP per user instruction 2026-08-11)**

Skip. Do not commit unless the user explicitly asks.

---

## Task 2 — Insert new pre-loop discovery step (new step 2)

**Files:**
- Modify: `.claude/skills/setup-workspace/SKILL.md` — Steps section, between current steps 1 and 2

**Why:** The catalog must be built **once per run**, before the per-repo loop, so ranking + suggestion
are ready by the time the first repo is picked. Placing it as new step 2 keeps the flow linear:
read config → build catalog → loop.

- [ ] **Step 1: Insert the new step 2 immediately after current step 1**

Find (the two lines that end current step 1 and start current step 2):

    1. **Read `.claude/tracker.json` → `gitHost`** (platform, org). If missing or a placeholder, stop and
       tell the user to run `/setup-project` first.
    2. **Loop over repos**, one at a time, until the user says they're done. For each, ask its **name**,

Replace with:

    1. **Read `.claude/tracker.json` → `gitHost`** (platform, org). If missing or a placeholder, stop and
       tell the user to run `/setup-project` first.
    2. **Build the template catalog (once per run).** If a GitLab MCP is configured and reachable, list
       projects in the `dacdigital/starter-packs` group. For each project, fetch: name, description, and
       `README.md`. Cache all three in memory as the run's **template catalog**. If GitLab MCP is
       missing, the group is unreachable, or the group is empty, warn once —
       `starter-packs unavailable — falling back to manual URLs` — and continue with an empty catalog.
       Discovery runs regardless of the project's target `gitHost`; the starter-packs group is the
       org's cross-host template library. Skip this step when `--refresh` was passed.
    3. **Loop over repos**, one at a time, until the user says they're done. For each, ask its **name**,

Note: the rest of current step 2 (starting with `**type** (frontend / backend / ML / infra / …)`)
is left in place — Task 3 rewrites it.

- [ ] **Step 2: Read back the updated Steps section header + first 3 numbered items**

Confirm:
- New step 2 begins with `**Build the template catalog (once per run).**`.
- New step 3 begins with `**Loop over repos**` and the rest of the old step 2 body follows.
- Old step 3 (`**High-level config (new-repo routes only).**`) is now unnumbered-broken — it
  still reads as `3.` in the source and will be re-numbered by Task 3.

- [ ] **Step 3: Commit (SKIP per user instruction)**

Skip.

---

## Task 3 — Rewrite the per-repo loop (new step 3 body)

**Files:**
- Modify: `.claude/skills/setup-workspace/SKILL.md` — the body of new step 3 (was old step 2)

**Why:** The per-repo loop is where the catalog is used: rank against `(type, language)`, print
the list with the best match starred, prompt with the new one-keystroke options, and route into
today's existing three routes plus a new "skip / empty" route.

- [ ] **Step 1: Replace the body of the per-repo loop step**

Find (the current step 2 body — now step 3 after Task 2):

    3. **Loop over repos**, one at a time, until the user says they're done. For each, ask its **name**,
       **type** (frontend / backend / ML / infra / …), and **language/stack**, then pick a route:
       - **New from template** — ask for a git-host template repo; instantiate a new repo from it.
       - **New from exemplar** — ask for an existing repo to learn from; create an empty repo, then copy
         across only high-level config extracted from the exemplar (see below).
       - **Existing** — the repo already exists on the git host; clone it as-is.

Replace with:

    3. **Loop over repos**, one at a time, until the user says they're done. For each, ask its **name**,
       **type** (frontend / backend / ML / infra / …), and **language/stack**. Then present the route
       picker:
       - **If the catalog is non-empty:** rank templates against `(type, language)` using — strongest
         to weakest — (a) explicit type token in the repo name (e.g. `frontend-*`, `backend-*`,
         `ml-*`), (b) language/stack token in the repo name (e.g. `*-next-app`, `*-spring`,
         `*-fastapi`), (c) keyword hits in the GitLab project description, (d) keyword hits in
         `README.md` (headings + first paragraph). Break ties in that order. Print all templates in
         ranked order, one per line, with the top match starred and the description as a one-line
         hint. If nothing scores above zero, print the list unranked with no star.
       - **Prompt:** `[Enter] accept ★ | <n> pick # | t <URL> template | x <URL> exemplar | e <URL>
         existing | s skip`. If the catalog is empty, the prompt collapses to `t | x | e | s`.
       - **Route the choice** into one of four routes:
         - **New from template** (`Enter`, `<n>`, or `t <URL>`) — instantiate a new repo from the
           chosen template. Same-host with native "use as template" / fork support: prefer the
           platform API. Cross-host or no native support: shallow-clone the template locally,
           create the target repo on `gitHost`, repoint `origin`, squash-push a single
           `Init from template <template-name>` commit (drop template history).
         - **New from exemplar** (`x <URL>`) — ask for an existing repo to learn from; create an
           empty target repo, then copy across only high-level config extracted from the exemplar
           (see next step).
         - **Existing** (`e <URL>`) — the repo already exists on the git host; clone it as-is; do
           not create a new repo.
         - **Skip** (`s`) — create an empty target repo on `gitHost` with no high-level config
           seeded. The user seeds config later or in a follow-up `/setup-workspace` run.

- [ ] **Step 2: Read back new step 3 in full**

Confirm:
- The ranking rules are present in the stated priority order.
- The prompt line matches the design exactly: `[Enter] accept ★ | <n> pick # | t <URL> template | x <URL> exemplar | e <URL> existing | s skip`.
- All four routes (Template / Exemplar / Existing / Skip) are named, with their trigger keys.
- Cross-host template mechanics (shallow-clone + squash-push) are stated.

- [ ] **Step 3: Commit (SKIP per user instruction)**

Skip.

---

## Task 4 — Clarify high-level config filter scope (amend new step 4)

**Files:**
- Modify: `.claude/skills/setup-workspace/SKILL.md` — new step 4 (was old step 3), first sentence

**Why:** The high-level config filter today applies to "new-repo routes only" (template +
exemplar). The design (Part 3) narrows this: templates ship whatever they ship — the filter
applies only to **exemplar**. This prevents accidentally stripping intentional template content.

- [ ] **Step 1: Amend the opening sentence of the high-level config step**

Find:

    3. **High-level config (new-repo routes only).** Keep only toolchain and quality config — never

(Note: this line still reads `3.` in the source after Task 2, since the renumbering by markdown
lists is positional. If Task 2 already renumbered it to `4.`, adjust accordingly.)

Replace the opening `3.` line with:

    4. **High-level config (exemplar route only).** Templates ship intentional content — no filtering.
       This step applies only to the **New from exemplar** route. Keep only toolchain and quality
       config — never

Then verify the rest of the paragraph (the file-list starting with `.editorconfig`, ...) is
untouched.

- [ ] **Step 2: Read back new step 4 in full**

Confirm:
- New step 4 starts with `**High-level config (exemplar route only).**`.
- The template exemption sentence (`Templates ship intentional content — no filtering.`) is
  present.
- The concrete file list (`.editorconfig`, `.gitattributes`, …, `lombok.config`) is intact.

- [ ] **Step 3: Commit (SKIP per user instruction)**

Skip.

---

## Task 5 — Full-file re-numbering + read-through

**Files:**
- Modify: `.claude/skills/setup-workspace/SKILL.md` — Steps section numbering only

**Why:** After Tasks 2–4, the ordinal numbers of the remaining steps (clone → index → regenerate
INDEX → gitignore reminder) are one higher than in the original. Markdown auto-renumbers on
render, but the source numbers should still be corrected for readability.

- [ ] **Step 1: Renumber the Steps list source**

Target final numbering:

    1. Read `.claude/tracker.json` → `gitHost`
    2. Build the template catalog
    3. Loop over repos (route pick)
    4. High-level config (exemplar route only)
    5. Clone every repo
    6. Index each repo
    7. Regenerate `workspace/INDEX.md`
    8. Remind the user (`workspace/*` gitignored, `docs/repos/*` committed)

Adjust the leading digit on each line so they match the sequence above. The bodies of steps 5–8
are unchanged from today.

- [ ] **Step 2: Read the full Steps section end-to-end**

Confirm:
- Numbers run 1 → 8 with no gaps or duplicates.
- Every step's body is intact — no accidental deletions during renumbering.
- Cross-references inside the file (if any) still point at the right step.

- [ ] **Step 3: Sanity-check `--refresh` semantics**

Search the file for `--refresh`. It appears in the Inputs section and in the current step 5
("`--refresh` runs only this step and step 6 over the already-cloned repos"). Confirm that after
renumbering, this reference points at the correct new numbers (should be steps 6 + 7). Fix
inline if wrong.

- [ ] **Step 4: Commit (SKIP per user instruction)**

Skip.

---

## Task 6 — Manual review pass (does the flow make sense?)

**Files:**
- Read only: `.claude/skills/setup-workspace/SKILL.md`

**Why:** A skill is prose that steers a model at runtime. There are no unit tests. The only
verification is a careful human/model read: does someone reading this from top to bottom know
exactly what to do?

- [ ] **Step 1: Read the entire SKILL.md top-to-bottom**

Not just the changed sections — the whole file. Check that:
- The Purpose paragraph still describes what the skill does (may need no changes, but confirm).
- Inputs mentions GitLab MCP soft dependency.
- Steps flow linearly (read config → build catalog → loop → …).
- The per-repo prompt is stated verbatim.
- All four routes are unambiguous about what they do and what URL (if any) they expect.
- Cross-host template mechanics are stated where they apply.
- Fallback behavior (empty catalog) is stated where it applies.
- The per-repo index template (unchanged from today) still renders correctly under its heading.

- [ ] **Step 2: Backward-compat check**

Confirm that a user pasting a URL today still gets today's behavior:
- `t <URL>` reaches New-from-template.
- `x <URL>` reaches New-from-exemplar.
- `e <URL>` reaches Existing.
- The high-level config filter still runs for the exemplar route.

- [ ] **Step 3: List anything ambiguous**

If any step reads ambiguously (e.g. "which repo gets the squash-push?" or "who runs the platform
API call?"), flag it. Fix inline or note as a follow-up.

- [ ] **Step 4: Commit (SKIP per user instruction)**

Skip.

---

## Self-review checklist (author runs this after writing the plan)

- **Spec coverage** — every Part of the spec has a task:
  - Part 1 (pre-loop discovery) → Task 2.
  - Part 2 (per-repo interaction: ranking + prompt + routing) → Task 3.
  - Part 3 (cross-host template mechanics) → Task 3 (Template route bullet).
  - Part 4 (backward compat) → Task 6, Step 2.
  - Part 5 (SKILL.md edits: step 1a + rewrite step 2 + amend step 3) → Tasks 2, 3, 4, plus Task 5
    for renumbering and Task 1 for the Inputs soft-dep note. Covered.
- **Placeholder scan** — no "TBD" / "add appropriate error handling" / "similar to Task N".
  Every step names the exact text to change and the exact replacement.
- **Type consistency** — the prompt string is identical across the spec, Task 3 Step 1, and Task
  3 Step 2's verification. Route names (Template / Exemplar / Existing / Skip) are consistent
  across Tasks 3 and 6.

No gaps found. Plan is ready.
