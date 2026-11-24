# declarative-seed — boot-time YAML import + full export of agents, versions and MCP backends

**Date:** 2026-11-24
**Status:** Approved — `/spec-review` PASS_WITH_ISSUES
**Modularity attachment:** `docs/architecture/current/modularity/open-bbcd/agent-lifecycle/` (L2),
which owns agents, versions, tool backends and deploy. Also touches
[`deployed-runtime`](../../architecture/current/modularity/open-bbcd/deployed-runtime/README.md),
whose routes now also accept a static agent ID, and the Helm chart (`deploy/helm/openbbc`, the
`c4/deployment.md` view).
**Architecture:** merged in arch PR #26 (log
`docs/architecture/logs/2026-11-24-declarative-seed/`).

**Grounded against:** OpenBBC `origin/main` @ `7d7d198`. Paths are relative to `open-bbcd/` unless
stated.

## Business value / Why

OpenBBC is about to run on clusters we don't operate. The usual case: we train an agent version
locally, then need that exact version on a client's deployment. Today that's impossible without
clicking through the backoffice on their cluster.

- The Helm chart (`deploy/helm/openbbc`) installs only the platform (open-bbcd, the aikdm cron
  drainers, Postgres). Agents, versions, MCP backends and wiring exist **only** in Postgres, and
  only the backoffice creates them (wizard → finalize → alpha → train).
- There is no import. The only export is `GET /agent_versions/{id}/config.yaml`, which holds the
  flow-map config only. It has no prompts, no MCP visibility and no sub-agents.
- Every ID is a random UUID generated per database, so a version trained locally has no identity
  that still means something on another cluster.

This feature makes OpenBBC resources declarable in YAML:

1. **Export** a version from any instance as a self-contained YAML file. The file carries the
   version, its agent, every sub-agent version it binds to, and every MCP backend it uses, enough
   to rebuild it on an empty database.
2. **Import at boot.** open-bbcd reads seed YAML after migrations and before it starts serving. It
   makes sure the declared backends, agents and versions exist, then optionally deploys a version.
3. **Stable IDs.** Agents and versions carry an operator-chosen `static_id`. Deploy and the
   deployed runtime accept it wherever they accept a UUID.

The deployment model is **"the agent is stable, the deployed version changes."** Clients may also
train and deploy their own versions on their cluster. The import never overwrites their work. It
guarantees our version is **present**, so they can always deploy or roll back to it, and it deploys
it only when nothing is deployed yet, unless we explicitly force it.

Every version is **immutable once it leaves `INITIALIZING`/`DRAFT`**, and every writer enforces
it. Without that, "static ID X" could mean different content on different clusters, or a client
could change our imported version in place.

Success is judged by:

- A fresh cluster installed with Helm and a seed ConfigMap comes up serving the declared agent at
  `/deployed/<agent static_id>/…`, with no backoffice step.
- Restarting a cluster where the client deployed a different version leaves their deployment and
  their versions untouched, and our version is still present.
- A version exported locally and imported elsewhere behaves the same: same prompts, same MCP
  visibility and notes, same sub-agents, same architecture.

## Change level

**C2.** New columns and a new migration on `agents`, `agent_versions` and `tool_backends`. A new
startup step that can block boot. Changed route contracts: path and body values accept static IDs;
the MCP write routes are locked outside `INITIALIZING`/`DRAFT`; version forks always append at the
tip. A new export API, and a new versioned file format (`openbbc-seed-v1`).

## Scope

### In

#### 1. Static IDs (data)

- **New columns.** `agents.static_id TEXT NULL UNIQUE` and `agent_versions.static_id TEXT NULL
  UNIQUE`.
  - Versions are unique **globally**, not per agent, so a version static ID resolves without its
    agent.
- **Format.** `^[a-z0-9][a-z0-9._-]{0,99}$`, **and** the value must not match the UUID pattern
  `^[0-9a-f]{8}-?[0-9a-f]{4}-?[0-9a-f]{4}-?[0-9a-f]{4}-?[0-9a-f]{12}$`.
  - The UUID pattern covers both the dashed and the 32-hex forms. Braces, `urn:` and upper case
    are already excluded by the first pattern.
  - That one pattern is used in the Go validator, the DB `CHECK`s and the route resolver (§6), so
    every lookup site can tell a static ID from a UUID without ambiguity.
- **Write-once.** Once set, a static ID never changes and is never cleared, as long as the row
  exists. It is set by the export (§5) or by an import (§3). The BO shows it read-only on the agent
  and version pages.
- **Backends.** `tool_backends.name` is already `UNIQUE` and serves as their static key, so there's
  no new column for that.
- **`managed_by_seed`.** New column `tool_backends.managed_by_seed BOOLEAN NOT NULL DEFAULT false`
  (see §3).
- **`v[N]` numbering.** The numbering scheme is unchanged.
  - A version's number is its position in the agent's chain of `parent_version_id` links, counted
    from the root (`GetVersionNum`, `versionChainCTE`, `assignVersionNumbers`,
    `dataset.go:262`).
  - `UNIQUE (parent_version_id)` (migration 007) keeps the chain strictly linear.
  - An imported version is appended at the **tip** (§3, §4), so it gets the next number. The tip
    is the agent's version that no other version names as its parent. The static ID is the
    reference that exists before the number does.

#### 2. Seed file format `openbbc-seed-v1`

The export produces the file; people may also write or edit it by hand. One file can declare any
number of resources.

```yaml
format: openbbc-seed-v1
backends:
  - name: crm-mcp                          # static key (tool_backends.name)
    kind: mcp_client                       # mcp_client | http_endpoint
    config:                                # same JSON shape as tool_backends.config
      url: ${CRM_MCP_URL}
      transport: streamable_http
      default_headers:
        Authorization: ${CRM_MCP_AUTHORIZATION}
  - name: crm-api
    kind: http_endpoint
    config:
      base_url: ${CRM_API_URL}
agents:
  - static_id: sales
    name: Sales assistant
    description: ...
    architecture: { ... }                  # agents.architecture verbatim (tools, flows, skills_meta, ...)
    flow_map_config: { ... }               # optional; root version's flow_map_config (Inputs tab)
    endpoint_backends:                     # agent_endpoint_backend, by backend name
      get_customer: crm-api
versions:
  - static_id: sales_v7
    agent: sales                           # agent static_id
    prompts: { main_prompt: ..., skill_prompts: { ... } }
    agent_tool_enabled: false
    mcp_backends:                          # agent_version_mcp_backend
      - backend: crm-mcp
        note: "..."
    sub_agents:                            # agent_version_subagent
      - name: researcher
        target: research_v3                # version static_id, declared in this seed set
        note: "..."
    deploy: if_none                        # none (default) | if_none | force
```

- **References.** Inside a seed set, everything refers to other resources by static key: agent
  `static_id`, version `static_id`, backend `name`. UUIDs never appear.
- **Self-contained.** Every reference (agent, sub-agent target version, backend) must resolve
  **inside the seed set**. A reference that matches only a DB row is a validation error.
  - The export always includes the full closure, so exported files satisfy this.
  - It also lets `seed validate` check every reference without a DB.
- **`${VAR}` interpolation, backends only.** Interpolation applies **after** YAML parsing, to
  scalar string values under `backends[].config` and nowhere else.
  - Prompts, notes and architecture are never interpolated, so a literal `${` in a prompt
    round-trips unchanged.
  - A value may be entirely a placeholder or contain one or more embedded placeholders
    (`https://${HOST}/mcp`). Substitution is textual on the already-parsed string, so `:`, `#`,
    quotes or newlines in a secret are safe.
  - A missing variable is an error.
  - `$${` escapes to a literal `${`.
- **Validation (structural, no DB):**
  - Every reference resolves inside the seed set.
  - Static IDs are valid (§1) and unique within the set.
  - Sub-agent binding names match `^[a-z][a-z0-9_-]{0,39}$` and are unique per caller.
  - Each target appears at most once per caller. This mirrors
    `UNIQUE (caller_version_id, target_version_id)`.
  - `caller ≠ target`.
  - **Agent-tool collision** (`checkNoCollision`): if a version has `agent_tool_enabled: true` or
    any `sub_agents`, its agent's `architecture` must not expose an endpoint tool named `agent`
    (`tools.ArchitectureHasEndpointTool`).
  - Backend kinds:
    - `mcp_backends` entries reference `mcp_client` backends.
    - `endpoint_backends` values reference `http_endpoint` backends.
    - Every `endpoint_backends` key is an endpoint tool ID in that agent's architecture.
  - **The binding graph over the seed set is acyclic.** Since every target is inside the set, this
    is a pure graph check.
  - At most one version per agent has `deploy` other than `none`.
  - Every declared agent has at least one declared version, so its architecture and
    `flow_map_config` always have a version to land with.
  - An unknown `format` value or an unknown key at any level is an error.

#### 3. Boot import

- **Configuration.** `SEED_PATH`, a file or a directory. A directory means every `*.yaml` / `*.yml`
  in it, non-recursive, in lexical order. All files merge into **one seed set**, and a duplicate
  key across files is an error.
- **When it runs.** In `cmd/open-bbcd` `run()`, after `database.Migrate` and before
  `ListenAndServe`. Nothing answers HTTP until the import finishes, so the probes can't pass early
  (see §7 for the startup probe).
- **On any error, the process exits non-zero.** Errors are validation failures, missing env vars,
  a failed deploy precondition, or a DB error. In k8s that shows up as a crash loop with the error
  in the log.
- **One transaction.** The whole import runs in a single DB transaction, so a partial import never
  persists.
- **Clearing `managed_by_seed` is boot-only.** Only the boot import (from `SEED_PATH`) clears the
  flag on backends missing from the set. A standalone `seed apply` sets flags but never clears
  them, so applying a partial file can't unlock the boot seed's backends.
- **`SEED_PATH` unset.** No import. The only write is clearing `managed_by_seed` on every backend,
  so switching seeding off unlocks them.
- **CLI:**
  - `open-bbcd seed validate [--no-env] <path>`: runs the §2 structural validation and
    interpolation without a DB.
    - With `--no-env`, it skips substituting values and prints the list of required env var names
      instead. This mode is for CI, which doesn't have client secrets.
  - `open-bbcd seed apply <path>`: runs the full import against `DATABASE_URL`, standalone.

Apply semantics, in this order:

1. **Backends: the file wins on every boot.**
   - Upsert by `name`. Set `config` from the interpolated file and set `managed_by_seed = true`.
     `kind` is set on insert only.
     - A `kind` differing from an existing backend's is an **error**. Flipping it would silently
       change how the immutable versions attached to it behave.
   - A BO-created backend that has the same name is taken over.
   - On boot only (see above), backends with `managed_by_seed = true` that the current seed set no
     longer declares get `managed_by_seed = false`. They are left in place, because rows may still reference them.
   - Reason: backends describe the environment (URLs and tokens). Rotating a token in the k8s
     Secret must reach the DB on the next restart.
2. **Agents: create if missing, otherwise leave alone.**
   - **Missing:** look the agent up by `static_id`. If it doesn't exist, insert it with name,
     description, architecture, `finalized_at = now()`, and endpoint wiring by backend name.
   - **Exists:** name, description and wiring are left alone, because the client may have rewired
     endpoints. Compare the canonical JSON of `architecture` (keys sorted, no insignificant
     whitespace).
     - Equal: nothing to do.
     - Different: **error**. A version's prompts are written against a specific architecture, so
       landing them on a different one would produce a broken version. Since the architecture is
       frozen after finalize, a different architecture means a different agent, which needs a new
       static ID.
3. **Versions: create if missing, otherwise leave alone.**
   - **Exists:** look the version up by `static_id`. If it exists, nothing is written to it.
     - If it belongs to a different agent than the file says, that's an **error**.
     - Otherwise compute the canonical content hash (§5, "content hash") of **the DB row** and of
       **the file entry**. A mismatch logs a warning naming the static ID and changes nothing. New
       content needs a new static ID.
     - No hash is stored; versions are immutable, so recomputing is exact.
   - **Missing:** insert it with status `READY`, so it skips the alpha drainer. It gets prompts,
     `agent_tool_enabled`, and MCP attachments by backend name.
     - Its `parent_version_id` is the agent's **tip**, read under the agent-row lock (§4). The new
       version therefore takes the next `v[N]`.
     - If the agent has no version yet, the new version is the root (`NULL`) and receives the
       agent's `flow_map_config`.
     - Seeded versions of one agent are appended in file order.
   - **Sub-agent bindings** are inserted in a **second pass**, after every version in the set
     exists, so bindings between versions in the same file resolve in any order.
     - They go through a transaction-aware form of `SubAgentRepository.AddBinding`'s checks: name
       regex, `checkNoCollision`, unique name and target per caller, `caller ≠ target`, target
       READY or DEPLOYED, and the **cycle check against the whole DB graph**
       (`subagent.go:187-204`).
     - Pass 1 also runs `checkNoCollision` for versions created with `agent_tool_enabled: true`.
     - The caller-status rule is skipped: these callers are `READY`. **The seed import is the one
       sanctioned exception to "only `INITIALIZING`/`DRAFT` callers bind."**
     - It only applies to versions created by this import, and only inside the import
       transaction. That keeps the acyclicity guarantee: the cycle check covers the whole graph,
       and a READY version gains bindings only at the moment it is created.
4. **Deploy, per version with `deploy` set:**
   - `if_none`: if the agent has no DEPLOYED version, deploy this one. Otherwise do nothing.
   - `force`: deploy this one if it isn't already, demoting the current one. This applies on
     **every** boot: `force` pins the deployment.
   - Applies the same rules as `POST /agents/{agent_id}/deploy`: the DEPLOYED singleton, the READY
     precondition, and the "all endpoints mapped" check.
     - These checks run only when a deploy is actually performed: `if_none` with nothing deployed,
       or `force` with the version not already deployed. An `if_none` that finds another version
       deployed checks nothing.
     - That check currently lives in the handler (`validateAllEndpointsMapped`), and
       `AgentVersionRepository.Deploy` opens its own transaction. Both move into a
       transaction-aware deploy service that the handler and the import share.
   - **Any deploy failure is a boot error.** This includes an agent that exists on the cluster with
     endpoints the client left unwired. This is deliberate: problems should surface at deploy time,
     not at runtime.
5. **Log a summary.** One structured log line per resource with the action taken (`created`,
   `updated`, `unchanged`, `warning`, `deployed`, `deploy_skipped`).
   - Backends log `created` on insert, otherwise `updated`.
   - `deploy_skipped` means `if_none` found another version already deployed.

#### 4. Version immutability and append-at-tip

**Immutable outside `INITIALIZING`/`DRAFT`.** Every per-version write is refused with `409`
(`ErrVersionLocked`) when the version's status is `PENDING`, `READY`, `TRAINING` or `DEPLOYED`. The
check lives in the repository layer, under a row lock on the version, so no caller can bypass it.

- **Prompts:** already never edited in place. Saving creates a new version
  (`CreateVersionFromPrompts`). Unchanged, except for the tip rule below.
- **Agent tool and sub-agent bindings:** already `409` outside `INITIALIZING`/`DRAFT`. Unchanged.
- **MCP visibility toggle and MCP notes:**
  - Routes: `POST …/architecture/mcp/{backendID}/toggle` and `POST …/architecture/mcp/notes`,
    which call `AttachMCP`, `DetachMCP` and the notes update.
  - **Changed:** these now return `409` outside `INITIALIZING`/`DRAFT`. Today they edit in place in
    any status.
  - The BO is not affected: on `main` the MCP tab is already read-only for every status except
    `INITIALIZING` (`configurator.go:219,631`). Only direct API callers lose the ability.
- **Alpha landing:** `PENDING → READY` through `LandBundle` (`agent_version.go:264`) stays the one
  writer allowed on a `PENDING` version.
  - `LandBundle` itself gains the guard: it writes only to `PENDING` versions, or to `INITIALIZING`
    and `PENDING` versions when `force` is set. With `force`, it doesn't rewrite a finalized
    architecture.
- **Other in-place writers:**
  - `AgentVersionRepository.SetPrompts` (`agent_version.go:317`) has no callers and is deleted.
  - `UpdateFlowMapConfig` and `UpdateFlowMapConfigForAgent` (`agent_version.go:529,594`) gain the
    repository-level guard: root version in `INITIALIZING` only, which matches their handler.
- **`scripts/seed_bundle.py`:** without `--force` it writes only to `PENDING` versions, the same
  rule as `LandBundle`. With `--force` it is limited to versions in `INITIALIZING` or `PENDING`. It
  refuses any other status, even with `--force`, and does not rewrite `agents.architecture` once
  `finalized_at` is set. Without this, a direct DB write would void both immutability and the
  "architecture frozen" check in §3.

**Forks append at the tip.**

- Every path that creates a version from an existing one appends it at the tip:
  - prompt save (`SavePrompts`)
  - prompt land (`LandPrompts`)
  - training completion (`training_session.go:228`)
  - the seed import
- These paths set `parent_version_id` to the agent's **current tip**, not to the version the
  content came from. Content is still copied from the edited version.
- **Lock order, for every fork path, the seed import and the shared deploy service:** agent rows
  first (`SELECT … FROM agents WHERE id = $1 FOR UPDATE`, several agents in ascending `id`), then
  version rows in ascending `id`. Each fork reads the tip after taking the agent lock. Today
  `insertVersionFromPromptsTx` locks only the source version, `FOR SHARE` (`agent_version.go:345`).
  With one lock, two concurrent forks can't both claim the same tip and hit
  `UNIQUE (parent_version_id)`.
- **Why:** today, forking from a non-tip version (for example the DEPLOYED v5 while v7 exists)
  violates `UNIQUE (parent_version_id)` and fails as a 500.
- **Result:** `parent_version_id` means "the previous version in this agent's history," and every
  fork gets the next `v[N]`.
- **The two meanings coincide on all existing rows**, because non-tip forks never succeeded. So no
  data changes meaning.
- **`parent_version_id` readers, checked against the new meaning:**
  - `GetVersionNum` and `versionChainCTE`: numbering; unaffected.
  - `assignVersionNumbers` (`agent.go:322`): numbering; unaffected.
  - The `dataset.go:262`, `subagent.go:366`, `eval.go:479` and `training_session.go:423` chain
    CTEs: numbering; unaffected.
  - The delete guard (`agent_version.go:61`, "has children"): only the tip is deletable, as today.
  - The root lookups (`agent_version.go:497,579,598`, `parent_version_id IS NULL`): exactly one
    root per agent still holds.
  - The `configurator.go:491` comment: update the wording.
  - `training_sessions.parent_version_id` is a separate column and keeps meaning "trained from."
- The source version's ID is not recorded. Recording it is out of scope.

#### 5. Export

- **`POST /agent_versions/{version_id}/export`** returns an `openbbc-seed-v1` YAML body
  (`Content-Disposition: attachment; filename=<version static_id>.seed.yaml`).
  - It's `POST` because it may write static IDs (below).
  - Optional JSON body:
    `{"static_ids": {"<agent or version uuid>": "<static id>", ...}, "deploy": "if_none"}`.
- **Status precondition.** The requested version must be `READY`, `TRAINING` or `DEPLOYED`.
  Otherwise the export returns `409`, so a static ID is only ever attached to immutable content.
  Sub-agent targets in the closure already satisfy this, because bindings require READY or
  DEPLOYED targets.
- **What goes in the file (the closure):**
  - The requested version.
  - Its agent, with the root version's `flow_map_config`.
  - Transitively, every sub-agent target version and that version's agent.
  - Every backend named in any exported agent's endpoint wiring or any exported version's MCP set.
- **Static IDs.**
  - A row that already has a static ID keeps it. Supplying a different one is a `409`.
  - For a row without one, the supplied value is used. If none is supplied, a default is generated:
    - Agents: `slug(name)`: lower-case, runs of characters outside `[a-z0-9]` collapsed to `-`,
      trimmed, and cut to 90 characters. If that leaves an empty or invalid result, `agent`.
    - Versions: `<agent static_id>_v<N>`, with the agent part cut so the whole ID is at most 90
      characters.
    - A collision with an existing static ID gets a `-2`, `-3`, … suffix. The 90-character cap
      leaves room for it.
  - The chosen IDs are **persisted locally** in the same transaction as the export, so exporting
    again yields the same file.
- **Deploy markers.** The requested version gets `deploy` from the body (default `if_none`), and
  sub-agent versions get `none`. A sub-agent's target version doesn't need to be deployed, because
  child sessions pin it.
- **Secrets → placeholders.**
  - Every `default_headers` value in a backend config becomes `${<BACKEND>_<HEADER>}`,
    upper-snake-cased (for example `${CRM_MCP_AUTHORIZATION}`).
  - If two names collide after upper-snake-casing, the later one (in backend name, then header
    order) gets `_2`, `_3`, ….
  - URLs and every other value export as plain values. Because import interpolates every string
    under `config`, the export escapes literal `$${` → `$$${` and `${` → `$${` in non-placeholder
    values, so they round-trip unchanged. The export modal warns that URLs are
    exported as-is, so a token in a query string would leak.
  - The response header `X-Seed-Env-Vars` lists the placeholder names, and the BO shows them after
    download.
- **Content hash** (shared with §3). SHA-256 over the canonical JSON of
  `{prompts, agent_tool_enabled, mcp_backends: sorted [(backend name, note)], sub_agents: sorted
  [(name, target static_id, note)]}`. The same function is used on a DB row and on a parsed file
  entry.
- **Not exported:** the discovery zip, chat and deployed sessions, datasets, evals, training
  history, and `parent_version_id` lineage.
- **BO.** An "Export" button on the version page, shown only for exportable statuses, opens a
  modal:
  - It lists every agent and version in the closure, with static IDs pre-filled (read-only where
    they're already set).
  - It has a `deploy` select for the requested version, and the URL warning.
  - Submitting downloads the file.

#### 6. Routes accept static IDs

- **Which routes.** In these routes, a value matching the §1 UUID pattern **case-insensitively** is
  looked up by `id`, and anything else by `static_id`.
  - Upper-case UUIDs keep working, and static IDs are always lower-case.
  - The resolver hands the handler the **stored** lower-case UUID, for both the path value and
    `version_id`, so string comparisons downstream (`deploy.go:90`) hold. Not found is the same `404` as today. The resolution is a shared
  helper applied before each handler's existing logic.
  - `POST /agents/{agent_id}/deploy`: the path, and the `version_id` body field.
  - `POST /agents/{agent_id}/undeploy`: the path only, since it takes no body.
  - Every `/deployed/{agent_id}/…` route, including the session-scoped artifact routes.
  - `POST /agent_versions/{version_id}/export`.
- **Stored keys stay UUIDs.** `deployed_sessions.agent_id` keeps storing the UUID. A session
  created via the static ID is the same session when addressed via the UUID, and the other way
  round.
- **Responses:** unchanged. They return UUIDs, plus `static_id` on the agent and version objects
  (additive field).

#### 7. Helm

- **New `seed:` values block:**

  ```yaml
  seed:
    enabled: false
    files: {}              # filename -> YAML string; rendered into a ConfigMap
    existingConfigMap: ""  # or mount an operator-managed ConfigMap instead
  ```

- When enabled, the ConfigMap is mounted read-only at `/etc/openbbc/seed` with
  `SEED_PATH=/etc/openbbc/seed`.
- The `${VAR}` values come from the existing `openbbcd.envFromSecret`. No new Secret knob.
- A `checksum/seed` pod annotation over the rendered `seed.files`, so changing the seed rolls the
  pod. An `existingConfigMap` can't be checksummed, and the README says it needs a manual
  `kubectl rollout restart`.
- A **`startupProbe`** on open-bbcd (`/health`, default `failureThreshold: 30`,
  `periodSeconds: 10`, configurable), so liveness can't kill the pod in the middle of migrations
  plus import.
- The README gets a "Seeding agents" section: export locally → put the file in values → put the
  `${VAR}`s in the `openbbcd.envFromSecret` Secret → install. It also documents the
  `force`/takeover sharp edges (Risks).

### Out

- **Deleting or pruning on boot.** Resources removed from the seed are never deleted. Only
  `managed_by_seed` is cleared on backends.
- **Updating an existing agent or version from the file.** The policy is "leave alone," and new
  content needs a new static ID.
- **A `DRAFT → READY` transition.** DRAFT versions created by a prompt save have no path to READY,
  today or after this feature. That's a pre-existing gap, not introduced here.
- **Exporting or importing the discovery zip**, datasets, evals, training sessions, or chat and
  deployed history.
- **A k8s operator or CRDs.** Re-applying while running (no restart) and watching the ConfigMap.
- **Multi-replica coordination.** The chart stays single-replica, as with migrations today.
- **Import through the REST API or a BO upload.** Import runs at boot and through the CLI only.
- **Encrypting secrets at rest** in `tool_backends.config`. That's unchanged.
- **Changing the existing `GET /agent_versions/{id}/config.yaml`.**
- **Making the BO MCP tab editable on DRAFT.**

## Contracts

1. **DB (one new migration, next free number on `main` — currently `031_static_ids.sql`):**
   - `agents.static_id TEXT NULL UNIQUE` + `CHECK` (§1 patterns).
   - `agent_versions.static_id TEXT NULL UNIQUE` + `CHECK` (§1 patterns).
   - `tool_backends.managed_by_seed BOOLEAN NOT NULL DEFAULT false`.
   - Purely additive. Existing rows get `NULL` / `false`. The goose Down drops the columns and
     their constraints.
2. **File format `openbbc-seed-v1`** (§2), versioned by the `format` key. Future versions get a new
   `format` value.
3. **Env / CLI:**
   - `SEED_PATH` env var.
   - Subcommands `open-bbcd seed validate [--no-env] <path>` and `open-bbcd seed apply <path>`.
4. **REST, new:** `POST /agent_versions/{version_id}/export`.
   - Returns the YAML body plus the `X-Seed-Env-Vars` header.
   - `409` when the version is not READY, TRAINING or DEPLOYED, or on a static ID conflict.
   - `404` for an unknown version.
5. **REST, changed:**
   - **Static IDs in routes.** `/agents/{agent_id}/deploy` (path and `version_id`),
     `/agents/{agent_id}/undeploy` (path) and `/deployed/{agent_id}/…` accept a static ID wherever
     they accept a UUID. The parameter names are unchanged; their values may now be static IDs.
   - **`static_id` field.** Added to agent and version JSON objects.
   - **MCP writes locked.** `POST …/architecture/mcp/{backendID}/toggle` and
     `POST …/architecture/mcp/notes` return `409` outside `INITIALIZING`/`DRAFT`. This is a
     behavior change for API callers only; the BO already disallows it.
   - **`parent_version_id` in the version JSON** now means "previous in history." This is safe:
     on every existing row it equals "copied from," because non-tip forks never succeeded.
   - **Forks append at the tip** (§4). Prompt save, prompt land and training completion append at
     the tip. Forking from a non-tip version now succeeds, where it fails today.
   - **Managed backends locked.** Backend update and delete on a `managed_by_seed` backend return
     `409`.
6. **Script:** `scripts/seed_bundle.py` writes only to `PENDING` versions. With `--force` it also
   accepts `INITIALIZING`, and it never rewrites a finalized architecture.
7. **Events:** none.

## Acceptance criteria

1. On an empty DB, booting with a seed set holding one backend, one agent and one `if_none`
   version produces:
   - those rows;
   - the version `READY → DEPLOYED`;
   - `/deployed/<agent static_id>/sessions` working end to end.

   `/health` doesn't answer until the import finishes.
2. Booting again with the same seed logs `unchanged` for the agent and the version, and `updated`
   for the backend. Agent and version rows are byte-identical.
3. After a client deploys another version and the pod restarts, the client's version is still
   DEPLOYED (`if_none`) and the seeded version still exists. With `force`, the seeded version is
   DEPLOYED after the restart.
4. Changing a seeded version's content under the same static ID logs a warning and leaves the row
   byte-identical. This also holds for a version that got its static ID from an export.
5. A seed whose agent architecture differs from the existing agent with that static ID makes boot
   exit non-zero with an error naming the agent. Nothing from that seed set is persisted.
6. Each of these fails both `seed validate` and boot, with an error naming the file and the key:
   - a missing `${VAR}`;
   - an unresolved reference;
   - two `deploy` markers on one agent;
   - a static ID matching the UUID pattern, including the 32-hex form;
   - an agent without versions;
   - an agent tool or binding on an architecture with an endpoint tool named `agent`;
   - a duplicate target per caller;
   - a backend kind mismatch;
   - a binding cycle (`a_v1 → b_v1 → a_v1`).

   `seed validate --no-env` passes without the env vars and prints the required names.
7. Exporting a READY version that binds a sub-agent produces a file with both versions, both agents
   and every referenced backend. Importing it on an empty DB yields versions whose prompts, MCP
   sets and notes, agent-tool flag, sub-agent bindings and architecture equal the source. The test
   compares canonical JSON and uses a prompt that contains a literal `${NOT_A_VAR}`.
8. Exporting the same version twice gives identical files, with static IDs persisted on the first
   export. Supplying a different static ID for a row that already has one returns `409`. Exporting
   a DRAFT, INITIALIZING or PENDING version returns `409`.
9. Exported backend headers appear only as `${…}` placeholders, and `X-Seed-Env-Vars` lists them.
   Placeholder names that collide get `_2`.
10. The MCP toggle or notes API on a READY or DEPLOYED version returns `409` and changes nothing. On
    an INITIALIZING or DRAFT version, the edit applies in place.
    `seed_bundle.py --force` on a READY version exits non-zero.
11. A seed version imported into an agent with v1…v7 becomes v8. Booting again leaves it v8, even
    after the client creates v9.
12. Saving prompts from v5 while v9 is the tip creates v10 with v5's content, where today it
    returns a 500.
13. Two concurrent forks on one agent both succeed, as consecutive versions.
14. Booting where a deploy is performed (`if_none` with nothing deployed, or `force`) and the
    agent has an unwired endpoint exits non-zero and persists nothing. An `if_none` that finds
    another version deployed boots fine, whatever the wiring.
15. `POST /agents/<static_id>/deploy` with `version_id=<version static_id>` deploys, and
    `POST /agents/<static_id>/undeploy` undeploys. Every `/deployed/<static_id>/…` route behaves
    the same as with the UUID and returns the same session rows.
16. Editing or deleting a `managed_by_seed` backend in the BO or API returns `409`. Removing it from
    the seed, or booting without `SEED_PATH`, clears the flag, after which it's editable again.
17. `helm template` with `seed.enabled=true` and `seed.files` renders the ConfigMap, the mount,
    `SEED_PATH`, the `checksum/seed` annotation and the `startupProbe`.

## Risks & assumptions

- **Assumption: the agent architecture is frozen after finalize**, so comparing it is a sound
  identity check. This spec closes the `seed_bundle.py --force` hole. If architecture becomes
  editable later, the "different architecture → error" rule needs revisiting.
- **Assumption: a sub-agent target only needs to be READY or DEPLOYED**, not deployed, which
  matches `subagent.go:183`. Exported sub-agent versions therefore default to `deploy: none`.
- **The import binds on READY callers.** This is a sanctioned exception to the DAG-by-construction
  rule. It is safe because the cycle check runs against the whole DB graph, inside the import
  transaction, and only for versions created in that transaction. The architecture invariant is
  updated to name it.
- **Lineage now means history order, not "copied from."** Every reader is listed in §4. They are
  either numbering or root and tip logic, so none changes behavior.
- **Strict deploy on boot can take a client's cluster down.** A seed with a `deploy` marker fails
  boot if the client unwired an endpoint of that agent. This is chosen deliberately: fail at deploy
  time rather than at runtime. `seed validate` in CI catches file errors, but not cluster state.
- **A boot failure takes the whole app down.** An operator can lock a client's cluster out with
  one bad seed or a missing Secret key. Mitigations: `seed validate` in CI and the clear log line.
- **`force` re-pins on every restart** and can silently undo a client's deliberate deploy after
  any pod restart. It is opt-in per version and documented as "pins the deployment."
- **Backend takeover by name.** A client's BO-created backend with the same name as a seeded one is
  overwritten and locked. This is documented in the chart README.
- **Secrets in `tool_backends.config` stay in plaintext in the DB**, the same as today. Exported
  files hold only header placeholders. URLs are exported as-is, with a warning in the modal.
- **One transaction for the whole import.** A large seed set holds locks during boot. That's fine
  at the expected size (a handful of agents).
- **Rollback loses identity.** The migration's goose Down drops `static_id`. After a downgrade and
  re-upgrade, exports may generate different IDs, and a seed's versions would be imported again as
  new rows.
- **Deleted seeded rows come back.** If a client deletes a seeded tip version or agent, the next
  boot re-creates it at the new tip, with a new UUID. That's consistent with "guarantees present."
- **Backend kind is fixed once created.** Changing it requires a new backend name.
- **API callers that toggled MCP on READY or DEPLOYED versions will get `409`.** No known caller
  does this, and the BO never could.
