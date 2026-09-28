---
name: split-pr
description: Split an oversized PR/MR into a stack of small, reviewable ones, then cut its ticket into sub-issues that match one-to-one. Two approval gates; --help for usage.
disable-model-invocation: true
---

# /split-pr `[<pr-url>] [--repo <name>] [--issue <id>] [--onto <branch>] [--local] [--dry-run] [--help]`

## Usage

Step 0 prints this block verbatim. Keep it in sync with `## Inputs` below; it is the text a user
sees, and improvising a different wording each run defeats the point of having it.

```
/split-pr [<pr-url>] [--repo <name>] [--issue <id>] [--onto <branch>] [--local] [--dry-run] [--help]

Split an oversized PR into a stack of small ones, then cut its ticket to match.

  <pr-url>          PR/MR URL or #NN. Omitted -> the open PR/MR for the
                    current branch is looked up when a git host is configured;
                    failing that, split against the branch's merge-base.
  --repo <name>     Split workspace/<name>/ instead of the current repo.
  --issue <id>      Parent ticket to split. Omitted -> the PR's linked issue.
                    Neither -> pure code split, tracker untouched.
  --onto <branch>   Base for the first PR. Default: the PR's target, or the
                    repo's default branch.
  --local           Build the branch stack but touch no git host: open no
                    PRs, push nothing, just print the push commands.
  --dry-run         Print the division table and stop. Creates nothing.
  --help            This text.

Two gates: the division table (editable) before anything is built, and a
confirmation before anything reaches the tracker or the git host.

  /split-pr --dry-run                        see the split without doing it
  /split-pr --repo ai-sales-indexer --issue AISC-1
  /split-pr https://gitlab.example.com/g/p/-/merge_requests/42
```

## Purpose

Split one oversized PR/MR into a stack of small, individually reviewable ones, **and split the
ticket behind it to match**. Triggered by anyone (dev who has outgrown their branch, reviewer who
will not review a 2000-line diff, lead). It creates git branches and change requests, delegates
sub-issue creation to `/split-issue`, and edits no tracked documentation of any kind.

The diff leads. The partition comes from the changed paths, the human approves it as a table
(**gate 1**), and only then is the ticket cut to fit — one sub-issue per group, created by
`/split-issue`, each MR naming the one it implements (and closing it only where the host can, see
step 11).

Where the parent ticket **already has** sub-issues, that is not an error: step 5 shows them and asks
whether to cut the stack along them instead. Aligning to what already exists writes nothing to the
tracker at all.

**Path-level only.** Groups are partitions of the changed *file paths*. Splitting two unrelated
changes that live in the same file is out of scope — see step 7. Expect this limit to bite on files
two sub-issues both legitimately claim (a `DOMAIN.md` the template ships and the feature extends, a
package `__init__.py` that imports the new modules). Name the file, say which group you gave it and
what the other branch therefore lacks; never split its hunks.

## Inputs

- `<pr-url>` (optional) — PR/MR URL or `#NN`. Omitted → the skill **looks one up**: with a git
  host configured it asks for an open PR/MR whose source branch is the current branch, and treats
  a match as the original exactly as if you had passed it. No match → the branch is split against
  its merge-base with **no original PR**, which does not stop the stack being published. See
  step 2.
- `--repo <name>` (optional) — split a service-repo checkout at `workspace/<name>/` instead of the
  current repo. Lets the skill run from the umbrella repo that holds `.claude/tracker.json`,
  against a service repo that has no config of its own.
- `--issue <id>` (optional) — tracker-native id of the **parent ticket to split**, in whatever form
  the configured tracker uses (`PROJ-N`, a work-item UUID, `#NN`, or a URL). Omitted → the issue
  linked from the PR, when there is one. Neither → the run is a pure code split and every tracker
  step is skipped.
- `--onto <branch>` (optional) — target branch for the first PR in the stack. Defaults to the PR's
  own target when there is one, otherwise the repo's default branch.
- `--local` (optional flag) — build the branch stack but touch no git host: open no PRs, push
  nothing, print the push commands instead. The explicit opt-out for when a host **is** configured
  and you want the stack only. Implied when no `gitHost` is configured.
- `--dry-run` (optional flag) — propose and print the plan, then **stop at gate 1**. Creates no
  branches, no sub-issues, no MRs; pushes nothing. See step 7.
- `--help` (optional flag) — print the usage block from `## Usage` and exit. See step 0.

## Steps

0. **Parse the arguments — before anything else.** This step touches no repo, reads no config and
   reaches no network. It runs first precisely so a mistyped flag costs nothing.
   - **`--help` or `-h`** → print the `## Usage` block verbatim and **exit**. Nothing below runs.
   - **Any other token starting with `-`**, i.e. not one of `--repo`, `--issue`, `--onto`,
     `--local`, `--dry-run`, `--help` → print `unknown flag <token>`, then the usage block, then **exit**.
     Never guess what was meant and never carry on with the flag ignored: `--dryrun` silently
     dropped is a run that creates branches, sub-issues and MRs the user believed they were only
     previewing.
   - **A flag that needs a value but has none** (`--repo`, `--issue`, `--onto`) → print
     `<flag> needs a value`, then the usage block, then exit.
   - Anything left that is not a flag is the positional `<pr-url>`. More than one → print
     `only one <pr-url> may be given` and exit.

   No arguments at all is **valid, not a help request**: it means split the current branch, with
   step 2 deciding whether a PR is involved. Print usage only when it is actually asked for.
1. **Resolve the repo and the host.** Repo root is `workspace/<name>/` when `--repo` was passed
   (resolve the symlink), otherwise `git rev-parse --show-toplevel`. If `--repo` names a checkout
   that is not there, **refuse**: print `workspace/<name> is not set up — run /setup-workspace
   first` and exit. **Never provision the checkout** — this skill does not clone, fetch or invoke
   `/setup-workspace`; setting the workspace up is the user's call.

   Read `gitHost.platform` and `gitHost.apiUrl` from the **invoking** directory's
   `.claude/tracker.json`, falling back to the resolved repo's own. **If neither has one, do not
   fail** — the run proceeds without a host, exactly as `--local` does, and `<pr-url>` is rejected
   with `no gitHost configured`.

   **Then resolve a client for that host, and check it points at the right instance.** Every host
   operation below — steps 2, 11 and 12 — goes through whatever client this environment provides
   for `gitHost.platform`: an MCP server, a CLI (`glab`, `gh`, …), or the platform's REST API
   directly. Take whichever is available and authenticated; the skill does not care which, and no
   step below should name one.

   **`gitHost.apiUrl` is the test, not the platform name.** A client authenticated against a
   different instance of the same platform is not a usable adapter — a GitLab MCP pointed at
   `gitlab.com` cannot see a project on a self-hosted GitLab, and will fail in ways that read like
   a missing project rather than a wrong endpoint. Confirm the client targets `gitHost.apiUrl`
   before relying on it, and if none does, say which clients you found and what host each is bound
   to rather than reporting the project as unreachable.

   **Publishing is decided here and only here:** a host is in play when one is configured and
   `--local` was not passed. That decision is independent of whether an original PR exists —
   conflating the two is how a deliberate "I never opened a PR, split it into seven" turns into a
   stack nobody can review.
2. **Resolve the change set.** Whether a host is involved is decided by the `gitHost` config, not
   by whether `<pr-url>` was typed.
   - **With `<pr-url>`** — fetch source branch, target branch, title, description, linked issue and
     the changed-file list via the host client from step 1. On failure, warn and resolve the same
     branches locally instead.
   - **Without it, but with a host configured — look one up.** Ask the host for an open PR/MR whose
     source branch is the resolved repo's current branch. Exactly one → treat it as the original
     and continue as though it had been passed, **naming the PR you found and saying you inferred
     it**; the user typed no URL, so an inferred one must never be silent. More than one → list
     them and ask which. None → say so and carry on: there is simply no original to
     reference or settle, which does **not** stop the stack being published. Opening a PR first is
     not a precondition for splitting into a stack of them.

     This is what someone means when they run `/split-pr` on a branch that already has an MR open.
     Specifying it also keeps runs reproducible: left unwritten, the behaviour depends on whether
     the model happens to think of looking, and the same command does different things on
     different days.
   - **No host, or `--local`** — source is the resolved repo's current branch, target is `--onto`
     or that repo's default; changed files are `git diff --name-only <target>...<source>`. The same
     branch/target resolution is used when the lookup above found nothing.

   Every git command in this skill runs against the **resolved** repo, never the invoking one.
3. **Refuse-with-hint guards.** Print the hint and exit on any of:
   - dirty working tree → `commit or stash before splitting`;
   - fewer than two changed paths → `nothing to split — <n> changed path(s)`;
   - source branch not resolvable locally after a `git fetch` → `cannot reach <branch> locally`;
   - **target branch not resolvable** — whether it came from `--onto`, from the PR, or from the
     repo's default. Try `<branch>`, then `<remote>/<branch>` after a fetch. Resolves → use it, and
     say so when the match came from the remote, since targeting a branch you have never checked
     out is legitimate and silently substituting a different ref is not. Neither → refuse with
     `--onto <branch> does not exist locally or on <remote>`.

     Guard this here rather than letting step 2 run into it. Git's own message is
     `fatal: ambiguous argument '<branch>...<source>': unknown revision or path not in the working
     tree`, which reads like a syntax error and sends people looking at their quoting instead of
     their branch name. The default target needs the same check: a fresh clone has only the
     checked-out branch as a local head, so the repo's default branch can be missing even though
     `git clone` just produced it;
   - **the source has been split before** — any local ref under `refs/heads/split/<source-leaf>/`
     already exists → list them and refuse with `<source> already has a split stack — delete those
     branches or split a different branch`.

   **Resolve both branch refs before any diff runs.** The two branch guards above are ordering
   constraints as much as checks: step 2's changed-file list and the path-count guard above both
   need the refs to resolve, so an unresolvable one has to be caught before either. Resolve source
   and target first, refuse if either fails, and only then compute the diff.

   **Check the stack prefix here, not at step 8.** Step 8 also refuses on an existing branch name,
   but it cannot fire until the groups are named, which is after the human has done all of gate 1's
   work. A branch prefix is knowable from the source branch alone, so it belongs with the other
   cheap guards; discovering a re-split at step 8 wastes the whole review.

   **Under `--dry-run` this one warns instead of refusing.** Dry-running a branch that was split
   before is a legitimate thing to do — re-deriving the partition to compare it against the stack
   that already exists is exactly the kind of check a dry run is for.
4. **Resolve the parent ticket.** `--issue <id>` if given, else the issue linked from the PR fetched
   in step 2. If there is neither — or `.claude/tracker.json` has no `tracker.platform` — say so
   once (`no parent ticket — splitting code only`) and skip steps 5, 10 and every tracker mention in
   step 11. A missing ticket is not an error; it makes this a pure code split.

   Otherwise fetch the ticket and its **existing** sub-issues via the tracker adapter:
   - **Plane** — resolve `<id>` to a work-item UUID, then `workitem list` the project and keep the
     items whose `parent` equals it. **Filter client-side**: `pql='childOf("PROJ-N")'` and
     `retrieve_by_identifier` both fail on some editions (`PQL and structured filters are not
     supported on this Plane edition`, HTTP 403) — do not depend on either. Paginate with
     `per_page` + `cursor`; sub-issues can sort anywhere in the list.

     Two more Plane quirks, both of which silently return nothing useful rather than erroring:
     **never pass `fields` to `workitem retrieve`** — a field mask nulls the entire response,
     including the very fields it names — and **`description_stripped` is null** even unmasked, so
     the body has to be read out of `description_html`. Retrieve unmasked and parse the HTML.
   - **GitHub** — the parent issue's `/sub_issues` collection.
   - **GitLab** — the parent's linked issues with relation `relates_to` (what `/split-issue`
     creates), plus epic children when the parent is an epic.

   **This step reads the tracker and nothing else** — never create, rename, reparent or close a
   tracker item here.
5. **Choose the partition basis** — only when step 4 found existing sub-issues. Never decide this
   silently: print them as a numbered list (id, title) and ask which the split should follow.
   - **align** — cut the stack along the existing sub-issues, one group per sub-issue (step 6a).
     **Writes nothing to the tracker**: step 10 is skipped entirely, because the sub-issues the
     stack maps onto already exist.
   - **new** — ignore them and partition from the diff (step 6b), then create a fresh set in
     step 10. Carry a warning into gate 2 naming the existing sub-issues that will be **left open
     and untouched** alongside the new ones. That is a duplicate-ticket risk and the user has to
     see it before confirming, not after.

   With no existing sub-issues there is nothing to ask: go to step 6b.
6. **Seed the groups** — one of two ways.

   **6a. Align to the existing sub-issues.** Propose **one group per sub-issue**, in the sub-issues'
   own order (Plane `sequence_id`, GitHub/GitLab issue number), each group titled from its sub-issue
   and carrying that id through to step 11. Assign each changed path to the sub-issue whose title
   and summary name it — the paths, directories, modules and filenames written into the sub-issue
   description are the matching signal.

   Where a description is thin, **the branch's own commit history is the better signal**: which
   commit introduced a path says more about which unit of work owns it than a keyword match does.
   Say which signal you used when the two disagree.

   Three divergences are expected. Surface all of them; resolve none silently:
   - **paths matching no sub-issue** → collect into one trailing `unmapped` group with no ticket
     and say so. This is usually real work nobody ticketed, and hiding it is how it ships
     unreviewed.
   - **sub-issues matching no path** → list them and open no group; never create an MR with an
     empty diff. When a sub-issue names a deliverable that has no file behind it at all, that is
     worth stating plainly — the ticket text and the branch disagree, and only one of them is real.
   - **tracker order ≠ dependency order** → reorder for dependency and name which groups moved and
     why. The stack must build in order even when the tracker was written in another one.

   **The mapping is strictly one-to-one.** One sub-issue, one group, one MR — a sub-issue never
   becomes two groups, and a group never carries two sub-issue ids. The whole value of aligning is
   that the tracker and the branch stack can be read against each other line for line, and a
   sub-issue spread over two MRs destroys that for everyone downstream who has to work out which
   half they are looking at. A big group is not a reason to break it.

   The group set under 6a is therefore fixed: one group per sub-issue, plus at most one trailing
   `unmapped` group carrying no id. That group is the single exception, and it exists because paths
   nobody ticketed have to surface somewhere.

   **6b. Derive from the diff.** Partition the changed paths into an ordered chain, each group a
   reviewable unit. Order by dependency, not by size — scaffolding before what sits on it, schema
   and migrations before the services that query them, production code before the docs describing
   them. Give each group a `title` and a one-line `summary` written to stand as a sub-issue: step 10
   hands both to `/split-issue` verbatim, so they are the ticket text, not internal labels.

   **Test files ride with the code they cover.** Put each test in the same group as its subject, so
   every MR carries its own evidence and a reviewer can judge one group without reading the next.
   Do not collect the tests into a trailing group of their own: that produces a stack of untested
   MRs followed by one enormous unreviewable one, and it defers exactly the part of the diff that
   tells a reviewer whether the rest is right. The exception is **shared test infrastructure** —
   fixtures, factories, markers, containers, a root `conftest`: that belongs with the toolchain at
   the base of the stack, because every later group's tests import it. Where a test covers two
   groups, it goes with the later one, whose branch is the first where it can pass.

   **Either way:** every changed path lands in exactly **one** group; assert the partition is total
   and disjoint before showing it.
7. **Gate 1 — approve the division.** Nothing has happened yet: no branch exists, no ticket has
   been touched, nothing has been pushed. This step is the whole refinement loop — there is no
   separate keep/edit/drop pass before it, because the commands below already cover every edit and
   asking the same questions twice only wastes the reviewer's attention.

   Print the plan as a table, one row per group:

   | # | Branch | Sub-issue | Paths | Builds onto |
   |---|--------|-----------|-------|-------------|
   | 1 | `split/<leaf>/01-<slug>` | *new* — "Extract the config loader" | 4 | `main` |
   | 2 | `split/<leaf>/02-<slug>` | *new* — "Port the indexer to it" | 11 | `split/<leaf>/01-<slug>` |

   The `Sub-issue` column says `*new* — "<title>"` for a ticket step 10 will create, or the existing
   `<id> — <title>` under 6a. List each group's paths under the table when a group holds few enough
   to read; otherwise print the count and the common prefixes. State plainly which intermediate
   branches are not expected to build or pass tests on their own, so nobody discovers it in CI.

   Then loop on an edit command, reprinting the table after each one:
   - `move <path> to <n>` — rehome a single path.
   - `merge <a> <b>` — fold two groups into one, prompting for the surviving title and summary.
     **Refused under 6a**: the survivor would carry two sub-issue ids.
   - `split <n>` — break one group into two, prompting for the path partition between them.
     **Refused under 6a**: both halves would carry the same sub-issue id.
   - `add` — a new group, taking its paths from the groups that currently hold them. **Refused
     under 6a**: the group set is the sub-issue list, and a new group would have no sub-issue to
     carry.
   - `rename <n>` — new title and summary; under 6a, also the existing sub-issue id it maps to.
   - `reorder <n> <m>` — move a group in the chain; re-derive every `Builds onto` afterwards.
   - `drop <n>` — **re-prompt for which group inherits its paths**; a dropped group's paths must
     never silently vanish from the stack. Under 6a this does **not** touch the sub-issue itself;
     closing that is `/split-issue --refine`'s job, not this skill's. **Dropping is permitted under
     6a**, unlike the three above: it leaves a sub-issue with no MR, which is exactly what 6a
     already prescribes for a sub-issue matching no path. Refusing it would strand a group whose
     paths all moved elsewhere and ship it as an empty MR, which 6a forbids outright.
   - `approve` — the only exit.

   Refuse the three with a hint, not a bare error:
   `<cmd> would break the one-to-one mapping with <id> — re-run without aligning, or split the
   parent with /split-issue first`.

   **Re-assert total-and-disjoint after every command.** If an edit left a path in two groups, print
   `path <p> claimed by groups <a> and <b>`; if in none, print `path <p> unassigned`. Either way
   re-prompt, and never resolve it by splitting a file's hunks — that is out of scope and produces
   branches that do not build. An edit loop that leaves the partition broken builds a stack that
   does not match the source.

   **Under 6a, re-assert the one-to-one mapping too:** every group carries exactly one sub-issue id
   except the single `unmapped` group, and no id appears in two groups. Assert it after every
   command rather than trusting the refusals above — a rename that retargets a group at an id
   another group already holds gets past them.

   Nothing below this step runs without an explicit `approve`.

   **`--dry-run` stops here** — print the table and exit, having created no branch, no sub-issue
   and no MR. The dry run must be safe to hand to someone who has not read this file: it names
   exactly what a real run would create and then does none of it.
8. **Build the stack** — plain git, identical on every host. Local and reversible: this creates
   branches in the resolved repo and pushes nothing, which is why it sits between the two gates.
   Refuse if any target branch name already exists; never clobber. (Step 3's prefix guard will
   normally have caught this already — this is the backstop for a name that collides some other
   way.) Then for each group `i` in order:
   `git checkout -b split/<source-leaf>/<nn>-<slug> <target-of-i>`, then
   `git checkout <source> -- <paths>`, then commit; `<target-of-(i+1)>` becomes this branch.
   Return to `<source>` at the end.

   **Do not name branches `<source>/<nn>-<slug>`.** Refs are files: `refs/heads/<source>` already
   exists, so git refuses any ref beneath it with `cannot lock ref … exists`. Note the failure mode
   if it happens anyway — git may still have switched the index and worktree to the target's tree
   before failing to write the ref, leaving every other path staged as a deletion on the unchanged
   HEAD. The tree is recoverable (`git checkout -f <source>`), but verify before continuing.

   Never force-push, never rewrite or delete the source branch — it stays intact as the fallback if
   the split is wrong.

   **Verify before reporting:** `git diff <last-branch> <source>` must be empty. If it is not, the
   partition and the built stack disagree — say so rather than reporting success, and do not
   proceed to gate 2 with a stack that does not reconstruct the source.
9. **Gate 2 — confirm the irreversible half.** Everything above this line can be undone by deleting
   local branches. Everything below writes to the tracker and the git host. State the whole cost in
   one prompt and count it out:

   ```
   create <n> sub-issues under <parent-id>, push <n> branches, open <n> draft MRs?
   ```

   Under 6a, the sub-issue line reads `map onto <n> existing sub-issues (no tracker writes)`
   instead. Under the **new**-with-existing case from step 5, repeat the warning here — name the
   existing sub-issues that stay open and untouched. A single explicit yes covers steps 10, 11 and
   12; anything else stops the run with the local stack left in place, and say that it is left in
   place and how to delete it.

   **Count the cost before prompting, and skip the gate when it is zero.** With no parent ticket and
   no git host there is nothing irreversible left: no sub-issue to create, no MR to open, and no
   remote to push to. Asking someone to confirm `0 sub-issues, 0 MRs` and a push that cannot happen
   is how people learn to approve without reading, which costs you the gate on the run where it
   matters. Print `nothing to confirm — no tracker, no host` and go straight on.
10. **Split the ticket — delegate to `/split-issue`.** Skipped when step 4 found no parent ticket,
    and skipped under 6a where the sub-issues already exist. **This skill never calls a tracker
    write itself**; `/split-issue` owns every sub-issue creation, so the tracker has exactly one
    writer.

    Write the approved groups to a JSON handoff file — a temp path, never inside the repo — as an
    array in stack order:

    ```json
    [{"title": "Extract the config loader",
      "summary": "…one paragraph, the sub-issue body…",
      "paths": ["src/config/loader.py", "src/config/schema.py"]}]
    ```

    Then invoke `/split-issue <parent-id> --groups <path>`, forwarding `--dry-run` when it was
    passed (it cannot be, since step 7 already exited, but forward it anyway so the two skills stay
    safe under any future reordering). `--groups` makes `/split-issue` create exactly these
    sub-issues, in this order, with no proposal or Q&A of its own — gate 1 was that approval.

    Map each returned sub-issue id back onto its group by position and **verify the count matches**.
    If `/split-issue` returns fewer ids than there are groups, stop before step 11: opening MRs that
    reference tickets which do not exist is worse than stopping with the branches built. Say which
    groups have no ticket and let the user re-run.
11. **Open one change request per group** via the adapter, using the sub-issue ids from step 10 (or
    the existing ones under 6a). Never reached under `--dry-run`:
    - **GitHub** — one PR per group, `head` = the stack branch, `base` = the previous stack branch
      (the original target for the first). Draft by default; inherit milestone and assignees from
      the parent PR when set.
    - **GitLab** — one MR per group, `source_branch` = the stack branch, `target_branch` = the
      previous stack branch. `Draft:` title prefix; inherit milestone and assignees from the parent
      MR when set.
    - **No host, or `--local`** — open nothing. Print the `git push` command and the intended base
      for each branch, in order.

    **Whether to publish depends on step 1's decision, never on whether an original PR exists.**
    With a host and no `--local`, open the MRs even when there is no original — that is the whole
    point of running this on a branch you deliberately never opened a PR for.

    Each body lists the group's paths and states its position in the chain (`2 of 5`), and links
    the parent PR/MR **when there is one**. With no original, say in the summary that the chain has
    no parent PR, so nobody hunts for a link that was never going to exist.

    **Reference the tracker id; only use a closing keyword when the host can honour it.** A
    closing keyword works only when the ticket lives in the same host that owns the PR — GitHub
    issues closed by a GitHub PR, GitLab issues by a GitLab MR, same project. Decide from the
    config, not from habit:
    - `tracker.platform == gitHost.platform` → `Closes #NN` in the body, and say it will close on
      merge.
    - `tracker.platform != gitHost.platform` — a standalone tracker (Plane, Jira, Linear, …)
      against repos on GitHub or GitLab → **no keyword.** The host cannot close a ticket it does
      not own, and `Closes PROJ-42` in the body silently does nothing while reading to every
      reviewer as though it does. Link the ticket by URL, write `Implements <ID>`, and state in the
      run summary that these tickets must be closed by hand.

    Never emit a keyword that resolves to nothing. A ticket that looks auto-closed and isn't is
    worse than one everybody knows is manual.
12. **Settle the original.** Prompt: **close** it with a comment pointing at the chain, or **leave
    it open** while the stack is reviewed. Never delete it. The parent **ticket** is settled by
    neither option — it stays open above its new sub-issues, which is where it belongs.

    **Do not offer to retarget it onto the last stack branch**, however tidy that sounds. Step 8
    builds each branch with `git checkout -b` from the target plus `git checkout <source> -- <paths>`
    — fresh commits that share no history with the source. The content matches exactly, which is
    why step 8's own verification passes, but a PR is diffed from the **merge-base**, and the
    merge-base of the stack and the source stays at the original target. Retargeting therefore
    shows the entire diff again, against a branch that already contains every one of those changes:
    eight files byte-identical on both sides, presented as outstanding work. Verified on GitLab —
    the retargeted MR reported all 8 of 8 files.

    Making retarget work would mean the stack sharing history with the source, i.e. rewriting the
    source branch, which this skill forbids. Closing is not a workaround here; it is the correct
    settle.

    **With no original there is nothing to settle.** No `<pr-url>` was given and step 2's lookup
    found none, or no host is in play — nothing was ever opened, so neither option exists. Do
    not offer them. Say that the source branch is intact and still checked out, and move on — a
    prompt whose only honest answer is "neither" teaches the reader to distrust the other prompts
    in this run. An **inferred** PR is a real original: settle it like any other.
13. **Print summary** — the ordered chain of branches, sub-issue ids and PR/MR URLs as the same
    table from gate 1 with the placeholder columns filled in, plus any path the user moved between
    groups during the edit loop, and any ticket that must be closed by hand (step 11).

## Config

Reads `.claude/tracker.json` → `gitHost.platform` and `gitHost.apiUrl` (which together pick and
validate the host client used in steps 2, 11 and 12 — see step 1; `apiUrl` is what distinguishes a
self-hosted instance from the platform's public one, so it is load-bearing rather than
informational) and `tracker.platform` (to read the parent ticket and its sub-issues in step 4,
and — compared against `gitHost.platform` — to decide in step 11 whether a closing keyword is
possible at all). The target branch is **not** configured — it comes from the PR itself or from the
repo's default branch, so no new config key is introduced.

The two platforms are configured independently and frequently differ — a standalone tracker against
repos on a separate host is a common setup, and no PR/MR can auto-close a ticket under it. Read both
keys and compare them; treat matching platforms as the special case, not the default.

Without `tracker.platform` there is no parent ticket to split: say so once and run as a pure code
split (step 4). `--issue` under that config is refused with `no tracker configured — run /split-pr
without --issue`.

The config is **optional**. With no `gitHost`, the skill still runs in local mode — it builds the
branch stack and prints what to push. That is what lets `--repo` work from an umbrella repo against
a service checkout carrying no config of its own, and what lets this skill be copied into a service
repo that has none.

**Writes to git branches, to the git host, and to the tracker only through `/split-issue`.** It
never calls a tracker write itself, and it never edits tracked documentation — not specs, not plans,
not an architecture tree, not a roadmap. Reactive splitting reacts to work already in flight; the
documents that work was planned from are not its to rewrite. Where a project defines its own one-way
direction rule between docs and tracker, this respects it by leaving docs alone.

**The `/split-issue` contract.** Step 10 depends on `/split-issue <parent> --groups <file>` creating
exactly the given sub-issues in the given order, skipping its own proposal and Q&A, and honouring
`--dry-run`. Both skills carry that contract; change one and change the other.

**Portability.** Nothing above is specific to one project: the platforms come from config, the
branch work is plain git, and the only paths the skill knows are the ones in the diff. To use it
elsewhere, drop the directory in alongside `/split-issue` and provide `.claude/tracker.json` — or
provide nothing and run in local mode. Adding a platform means adding one bullet to step 4 and one
to step 11; the flow does not change.
