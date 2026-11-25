# seed-multi-export — export several agents into one seed file

**Date:** 2026-11-25
**Status:** Approved — `/spec-review` PASS_WITH_ISSUES
**Modularity attachment:** `docs/architecture/current/modularity/open-bbcd/agent-lifecycle/` (L2), capability
*Declarative seeding* (added by arch PR #26). This extends the merged spec
[`2026-11-24-declarative-seed-design.md`](2026-11-24-declarative-seed-design.md) (§5 Export). The file format,
import and static-ID rules are unchanged.

**Grounded against:** OpenBBC `main` @ `f2905e5` (OpenBBC #72). Paths are relative to `open-bbcd/`.

## Business value / Why

A real deployment ships several agents together. For example, AI Sales Copilot needs `sales-copilot` and
`customer-sparring`. Today an export starts from **one** version (`POST /agent_versions/{version_id}/export`), so
shipping two agents means two exports merged by hand into one file. That is error-prone: duplicate backends, two
`format:` keys, and keeping static IDs consistent are all manual.

The `openbbc-seed-v1` format already holds any number of agents and versions. This change lets one export produce
that file directly:

- a **backoffice page**, reached from the main menu, where the operator picks **at most one version per agent** across
  all agents and downloads one seed file;
- an **endpoint** that does the same, for scripts and CI.

Success is judged by:

- Exporting `sales-copilot` v38 and `customer-sparring` v16 in one go gives a file that validates and imports.
- That file is equivalent to the two single exports merged by hand: backends are shared and not duplicated, and each
  version carries its own deploy marker.

## Change level

**C2.** New REST endpoint `POST /seed/export` and a new backoffice route. There is no DB change, no event change, and
no change to the seed file format or the import. Existing contracts stay unchanged, including the single-version
export `POST /agent_versions/{version_id}/export`. `/seed/export` follows the existing `…/export` route pattern on
purpose.

## Scope

### In

1. **Repository: export from several roots.**
   - `SeedRepository.Export` changes from one root version to a list of selected versions. Each selected version has
     its own deploy marker.
   - The closure (spec §5) is the union of every selected version's closure. Agents, versions and backends are
     de-duplicated, as the single-root walk already does for shared targets.
   - The whole export stays one transaction, under the same lock order (agents ascending `id`, then versions).
   - **Status precondition:** every *selected* version must be `READY`, `TRAINING` or `DEPLOYED`; otherwise 409
     (`ErrVersionNotExportable`, whose message names the version).
   - **One version per agent:** two selected versions belonging to the same agent → 400
     (`ErrDuplicateAgentSelection`). Sub-agent targets pulled in
     by the closure don't count as selections. They are exported with `deploy: none`, as today, even if another
     version of that agent was selected. A binding needs its exact target version.
   - **Deploy markers:** each selected version gets the deploy value from the request (default `if_none`). Every
     closure-only version gets `none`. This always satisfies the seed rule of at most one marker per agent.
   - **Order in the file:** versions of the same agent are emitted in source-history order (by their position in the
     agent's version chain), so the import's append-at-tip keeps history order. A closure-only version of a selected
     agent therefore never lands after the selected one if it is older. Closure order is one breadth-first walk over all
     selected versions in request order, following bindings by name; this keeps the one-element case byte-identical to
     today's single export. Agents appear in first-appearance order. Same determinism rule as today.
   - Static IDs, placeholders, escaping, write-once persistence and the `static_ids` key check (keys must belong to
     the closure) are exactly as in spec §5.
   - The single-version `Export(ctx, versionID, req)` stays and becomes a one-element call of the multi export, with
     unchanged behaviour and output.
   - `ExportPreview` gets the same multi-root form, for the page's confirmation step.
2. **New endpoint `POST /seed/export`.**
   - Body: `{"versions": [{"version_id": "<uuid|static_id>", "deploy": "if_none|force|none"}], "static_ids": {...}}`.
   - `version_id` is resolved like every other route: a UUID or a static ID.
   - `versions` must be non-empty, or the request gets 400. Duplicate entries (the same version, including once by UUID
     and once by static ID) also get 400. Unknown body fields (e.g. a top-level `deploy`) get 400, since the decoder
     disallows unknown fields.
   - **Error precedence:** resolve every `version_id` first (404), then check request shape (400), then status and
     static IDs (409).
   - Response: same as the single export. `200 application/yaml`, with
     `Content-Disposition: attachment; filename=openbbc.seed.yaml`, and `X-Seed-Env-Vars`.
   - Errors: 400 for an empty list, an invalid `deploy`, a duplicate entry, an unknown body field, two versions of
     one agent, an invalid static ID, or a `static_ids` key outside the closure. 404 for an unknown version. 409 for a non-exportable version or a static-ID
     conflict.
3. **Backoffice page "Export"**, a new main-menu item next to Agents, MCP, Datasets, Evals and Training sessions.
   - Route `GET /seed/export`.
   - One row per agent: agent name, a version dropdown, and a deploy dropdown.
     - The version dropdown offers *not exported* (the default) and the agent's exportable versions (READY, TRAINING,
       DEPLOYED), newest first, labelled `v<N>`, the status and the static ID if set.
     - The deploy dropdown shows `if_none` / `force` / `none`, defaulting to `if_none`. It is disabled while *not
       exported* is chosen.
   - **Continue** loads the confirmation step, built from `ExportPreview` over the chosen versions. Like today's modal,
     it lists every agent and version in the closure with static IDs prefilled (read-only where already set), plus the
     URL warning. **Download** posts to `POST /seed/export` and saves the file. After the download, the page shows the
     names from `X-Seed-Env-Vars`.
   - Agents with no exportable version are listed with the dropdown disabled and the note "no exportable version".
4. **Tests.**
   - Repository: a two-agent export with a shared backend (de-duplicated) and per-version deploy markers. Two versions
     of one agent → error. A selected version plus a closure-only version of the same agent → allowed, with the
     closure one `none`. A non-exportable selection → 409. The single-root export output is byte-identical to before.
   - Handler: the 200/400/404/409 cases above. The page renders agents, versions and disabled rows.
   - Round trip: a two-agent export imports into an empty DB with both versions deployed.

### Out

- Exporting more than one version of the same agent in one file. Users who need a rollback version keep the older
  version in the target DB by importing it earlier.
- Exporting whole agent histories, datasets, evals or sessions.
- Any change to the seed format, the import, or the existing `POST /agent_versions/{version_id}/export`.
- Saving export selections ("export profiles").

## Contracts

1. **REST, new: `POST /seed/export`.**
   - Request: `{"versions":[{"version_id": string, "deploy"?: "none"|"if_none"|"force"}], "static_ids"?: {uuid: static_id}}`.
   - Response: `200`, `application/yaml`, an `openbbc-seed-v1` body, header `Content-Disposition: attachment; filename=openbbc.seed.yaml`, and header `X-Seed-Env-Vars: <comma-separated names>`.
   - Errors: `400` (empty `versions`, invalid `deploy`, duplicate entry, unknown body field, two versions of one agent,
     invalid static ID, `static_ids` key outside the closure), `404` (unknown version), `409` (version not READY, TRAINING or DEPLOYED, or a static-ID
     conflict).
2. **Backoffice, new:** `GET /seed/export` (the page) and its confirmation fragment route. The fragment route and
   query shape are left to the builder.
3. **Unchanged:** `POST /agent_versions/{version_id}/export` (request, response and output bytes), the
   `openbbc-seed-v1` format, the import, and the DB schema.
4. **Events:** none.

## Acceptance criteria

1. `POST /seed/export` with `sales-copilot` v38 (`if_none`) and `customer-sparring` v16 (`force`) returns one file:
   - both agents and versions are in it;
   - the `kb` backend appears once;
   - the deploy markers are `if_none` and `force`.

   `open-bbcd seed validate --no-env` passes on it.
2. Importing that file into an empty DB creates both agents, creates both versions and deploys both.
3. Two versions of the same agent in `versions` → 400, and nothing is persisted.
   - A duplicate entry in `versions` (same version twice, including once by UUID and once by static ID) → 400.
   - An unknown body field (e.g. top-level `deploy`) → 400.
4. A selected version whose sub-agent target is another version of a different selected agent exports both. The
   target version has `deploy: none`, and the file validates.
5. A DRAFT version in `versions` → 409, naming it.
6. `POST /agent_versions/{version_id}/export` output is byte-identical to before for the existing test fixtures.
7. The backoffice main menu has **Export**. The page lets the operator pick at most one version per agent, and the
   downloaded file equals the endpoint's output for the same selection.

## Risks & assumptions

- **Closure-only versions of a selected agent.** A file can contain two versions of one agent when a binding requires
  it. Only the selected one can carry a deploy marker, so the import rule still holds. The page's confirmation step
  lists closure-only versions, so the operator sees them.
- **Static-ID persistence covers every exported row.** As with the single export, IDs are written for every row in
  the union closure. A large selection stamps many IDs at once. That is expected (write-once), but irreversible.
- **Architecture docs lag.** `docs/architecture/current/` describes export as single-version (capabilities.md,
  agent-lifecycle.md published surface, access-model.md, containers.md, data-flows.md); sync with
  `/arch-review seed-multi-export` after merge.
- **Assumption:** operators ship one version per agent. A rollback version needs a separate earlier export (see Out).
