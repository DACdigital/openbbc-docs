---
date: 2026-10-01
codename: sync-deployed-runtime-artifacts
kind: sync
---

# Sync — align current/ with 2026-09-30-deployed-runtime-artifacts-design

## Driver
Post-approval sync of `docs/architecture/current/` with the following spec(s):
- `docs/superpowers/specs/2026-09-30-deployed-runtime-artifacts-design.md` (approved in
  openbbc-docs PR #10, amended in PR #11)

## Decision
Apply the deltas listed below to `docs/architecture/current/`. AMBIGUOUS deltas (if any) are
recorded here for human resolution but were NOT applied.

In short: user artifacts become **session-bound and staged** (upload → pending artifact →
claimed server-side by the next turn; the turn body carries text only); each session-owning
context owns a per-session artifact table (`chat_session_artifacts` in `feedback-datasets`,
`deployed_session_artifacts` in `deployed-runtime`) that is the single read allow-list; the
`artifacts` context keeps the env-hydrated registry, adapter dispatch, MCP normalisation and
MIME resolution and still owns no tables; retrieval routes are nested under the owning
session on both surfaces; `ARTIFACT_REF` is carried as an AG-UI `CUSTOM` event; the
agent-to-tool (inner-ref) leg is removed; artifact scope does not cross the agent-tool
boundary; eval replay is text-only.

## Rationale
Approved spec is the design contract; `current/` is the architecture-of-record and must reflect it.
Per project rule: L2 modularity nodes are inspirational — specs are the impl contract; sync applies
CONTRACT_SHAPE deltas without altering module Purpose, and only introduces new L1/L2 surface for
SCOPE_MOVE deltas.

Where a CONTRACT_SHAPE wording would otherwise depend on an AMBIGUOUS item, neutral wording
was used: retrieval headers name `nosniff` + `Content-Disposition` and "equivalent response
overrides" on signed URLs without asserting `Content-Type = row.mime` (AMBIGUOUS 2); the
missing-key credential requirement is stated for AWS S3 (`s3:ListBucket`) without claiming
how MinIO behaves (AMBIGUOUS 3); the BO access-model row describes the BO session preamble
without asserting its status code for a foreign-version session (AMBIGUOUS 4); nothing is
said about migration 027 backfill (AMBIGUOUS 1) or turn/upload locking (AMBIGUOUS 5).

## Alternatives rejected
- Leaving current/ stale until the next `/modularize` run — rejected: drift accumulates silently and
  a new dev reading current/ builds against the wrong contract.
- Auto-applying all deltas including AMBIGUOUS ones — rejected: silent overrides of prior arch-log
  decisions require human review.

## Impact

### CONTRACT_SHAPE deltas applied (no purpose change)
1. `ddd/contexts/artifacts.md § Published surface (REST)` — replaced `/chat-sessions/{id}/artifacts`
   and top-level `GET /artifacts/{store_id}/{uri}?session_id=…` with the nested BO / deployed
   upload + retrieval routes; upload returns the pending-artifact object; read = session-artifact
   row → `Stat` (`410`) → `302` / `200`; bytes responses carry `nosniff` + `Content-Disposition`
   inline/attachment.
2. `ddd/contexts/artifacts.md § Invariants (Session-scoped read authorisation)` — allow-list is the
   session's own session-artifact rows (any origin, pending or consumed), not message content.
3. `ddd/contexts/artifacts.md § Invariants (MCP tool-result normalisation)` — only inline
   `blob`/`text` EmbeddedResources + ImageContent/audio; URI-only not normalised; `isError`
   results normalised; failed item → `[artifact unavailable: <mime>, <size>]`, no raw fallback.
4. `ddd/contexts/artifacts.md § Invariants` — new server-side MIME resolution + measurement
   invariant (native-render set sniff + image decode check; mismatched native label →
   `application/octet-stream`).
5. Bytes invariants scoped to an enabled registry —
   `ddd/contexts/artifacts.md § Invariants`, `ddd/contexts/deployed-runtime.md § Invariants`,
   `constraints.md § Hard technical limits`, `assumptions.md § Design decisions (locked)`,
   `c4/containers.md § open-bbcd`, `modularity/open-bbcd/README.md § Purpose` — "when the
   artifact registry is enabled"; disabled → raw tool output streamed/persisted as before;
   rendered media never persisted.
6. `ddd/contexts/deployed-runtime.md § Published surface` and
   `modularity/open-bbcd/deployed-runtime/README.md § Purpose` — upload returns the
   pending-artifact object; turn reads only `text` blocks and claims pending artifacts; empty
   turn → `400` `empty_turn`; concurrent-removal race → in-band `RUN_ERROR` `empty_turn`.
7. `ARTIFACT_REF` as AG-UI `CUSTOM` event — `ddd/contexts/deployed-runtime.md § Invariants` +
   `§ Published surface`, `modularity/open-bbcd/deployed-runtime/README.md § Purpose`,
   `c4/integrations.md § Contracts (AG-UI ARTIFACT_REF)`, `glossary.md` AG-UI row — `name`
   `ARTIFACT_REF`, camelCase `value`; corrected the "unknown event" claims; emitted only for
   tool-result refs after the tool message commits; `TOOL_CALL_RESULT` carries the normalised
   remainder; additive-only payload.
8. `ddd/contexts/deployed-runtime.md § Invariants (artifact refs session-scoped)` — nested route;
   preamble (deployed agent + session/user + agent match) + `deployed_session_artifacts` row;
   any failure → `404`; child sessions not addressable.
9. Adapter contract — `c4/integrations.md § Contracts (artifact-store adapter interface)`,
   `modularity/open-bbcd/artifacts/README.md § Purpose`, `bizbok/capabilities.md
   artifact-store-adapter`, `ddd/contexts/artifacts.md § Published surface (consumed contract)` —
   `Sign(…, SignOptions{ContentType, ContentDisposition})`; `probe()` fails boot if a missing key
   is not reported as not-found; `stat` drives the render fallback and upload dedup.
10. `c4/integrations.md` Object store row (Auth) and `assumptions.md § Scope assumptions` — store
    credentials must let `Stat` report a missing key as not-found (for AWS S3 this needs
    `s3:ListBucket`); neutral on MinIO per AMBIGUOUS 3.
11. `c4/integrations.md § Contracts (AG-UI event stream)` and
    `ddd/contexts/feedback-datasets.md § Published surface` — BO chat stream also emits artifact
    refs with the same semantics (`artifact_ref` frame on JSONL, `CUSTOM` event on AG-UI).
12. Stale paths — `constraints.md § Hard technical limits (ARTIFACT_MAX_UPLOAD_MB)`,
    `nfrs.md § Security`, `c4/deployment.md § Threat model ("Artifact ref leak")`,
    `ddd/contexts/feedback-datasets.md § Invariants (locked sessions)` — nested routes;
    threat mitigation = row allow-list + staging + MIME resolution + nosniff +
    Content-Disposition; BO lock invariant also covers `DELETE …/pending-artifacts/{id}` → `409`.
13. `constraints.md § Hard technical limits` (new `ARTIFACT_MAX_PENDING` bullet) and
    `c4/deployment.md § Threat model` (oversized-upload row mitigation).
14. `c4/deployment.md § Environments` — Helm chart optional `artifacts:` values block, disabled by
    default.
15. No `artifact_stores` table — `glossary.md` Content block, `bizbok/information-map.md` Chat
    message / Deployed message (+ Artifact store row wording), `c4/context.md § External
    systems` Object store, `ddd/context-map.md § Relationships` artifacts → External object
    store (+ `§ Contexts` artifacts bullet), `c4/containers.md § open-bbcd` DDD contexts
    ("env-hydrated store registry").
16. `glossary.md` Artifact — two framework legs (staged upload, MCP tool output); tool-input
    pointers opaque.
17. Eval replay text-only — `ddd/contexts/feedback-datasets.md § Invariants`,
    `bizbok/value-streams.md § Multimodal chat with artifacts` step 6 + outcome,
    `ddd/context-map.md § Relationships` feedback-datasets → evaluation row,
    `assumptions.md § Design decisions (locked)` artifacts decision rationale,
    `bizbok/capabilities.md` chat-artifacts, `ddd/contexts/artifacts.md § Invariants` (refs
    immutable bullet) — refs exported unchanged, replay text-only, refs stay resolvable for
    retrieval.

### SCOPE_MOVE deltas applied (new surface / ownership / boundary)
- A. `modularity/open-bbcd/deployed-runtime/README.md § Purpose`,
  `ddd/contexts/deployed-runtime.md § Aggregates & entities`, `§ Domain events`,
  `§ Published surface` — owns `deployed_session_artifacts` (migration 028, cascade FK); new
  *Deployed session artifact* entity; publishes retrieval + `GET`/`DELETE …/pending-artifacts`.
- B. `modularity/open-bbcd/feedback-datasets/README.md § Purpose`,
  `ddd/contexts/feedback-datasets.md § Aggregates & entities`, `§ Domain events`,
  `§ Published surface` — owns `chat_session_artifacts` (migration 027); new *Chat session
  artifact* entity; publishes BO upload, retrieval, pending list/remove (locked → `409`); chat
  view renders pending artifacts.
- C. `ddd/contexts/artifacts.md § Purpose` + `§ Aggregates & entities`,
  `modularity/open-bbcd/artifacts/README.md § Purpose`, `ddd/context-map.md § Relationships`
  (OHS rows) — `artifacts` keeps registry / dispatch / normalisation / MIME resolution and owns no
  tables; session-artifact identity, lifecycle and allow-list live in the owning contexts, which
  publish the routes; dropped "not a standalone entity" and "stateless in the DB / union of
  artifact_ref blocks".
- D. `c4/containers.md § open-bbcd` (Data ownership) + `§ postgres` (diagram label, migration
  head), `modularity/open-bbcd/README.md § Purpose`, `constraints.md § Hard technical limits` —
  both tables added; migration head → `028_deployed_session_artifacts`; dropped "Postgres holds
  only artifact_ref blocks"; `bizbok/capabilities.md` Artifact management L1 wording.
- E. `nfrs.md § Compliance` — per-session artifact metadata (incl. possibly-PII `filename`) in
  Postgres, removed by session cascade; bytes stay in the store.
- F. `c4/data-flows.md § Multimodal chat with artifacts` (sequence diagram + narrative),
  `bizbok/value-streams.md § Multimodal chat with artifacts` steps 2–5,
  `bizbok/capabilities.md` chat-artifacts / deployed-runtime-artifacts — upload → pending
  (listable/removable) → text-only turn → claim in one transaction → tool_result rows →
  `ARTIFACT_REF` after commit → nested retrieval; assistant-emission step removed.
- G. `ddd/access-model.md § Policies` (End user deployed-runtime-artifacts row; Admin
  chat-artifacts row) — ops upload / read / list-pending / remove-pending on nested routes;
  own-session row required to read; turn-body refs ignored; child sessions → `404`.
- H. Sub-agent artifact scope (narrows multiagent-tools) — `nfrs.md § Security` (Sub-agent trust
  model), `ddd/contexts/feedback-datasets.md § Aggregates & entities` (Child chat session),
  `c4/integrations.md § Contracts (Agent tool)`, `glossary.md` Agent tool / Sub-agent,
  `assumptions.md § Design decisions (locked)` multi-agent decision,
  `bizbok/capabilities.md` sub-agent-dispatch, `bizbok/value-streams.md § Multi-agent topology`
  step 3, `c4/data-flows.md § Multi-agent delegated turn`, `ddd/contexts/deployed-runtime.md
  § Invariants` (Sub-agent progress), `ddd/contexts/deployed-runtime.md § Aggregates & entities`
  — `agent(subagent, description, prompt) → text`; per-session scope, no inheritance; child
  tool-result rows invisible to root/user; child `ARTIFACT_REF` not forwarded; `user_id` /
  header inheritance unchanged.
- I. Inner-ref pointers not resolved — `ddd/contexts/artifacts.md § Published surface` (former
  "leg 4" emitted contract) + `§ Aggregates & entities` + new `§ Invariants` bullet,
  `modularity/open-bbcd/artifacts/README.md § Purpose`, `bizbok/capabilities.md` chat-artifacts /
  deployed-runtime-artifacts, `bizbok/information-map.md` Artifact row, `assumptions.md`
  artifacts decision, `bizbok/value-streams.md § Multimodal` step 3, `c4/data-flows.md
  § Multimodal`, `ddd/contexts/deployed-runtime.md § Invariants` — agent-to-tool leg and
  materialisation removed; LLM-written `{store_id, uri}` is opaque.
- J. `glossary.md` (new *Session artifact*, *Pending artifact* rows) and
  `bizbok/information-map.md` (new *Session artifact*, *Pending artifact* rows, tag
  `user-content (deployer-classified)`).

### AMBIGUOUS deltas NOT applied — human review required
1. `ddd/contexts/feedback-datasets.md § Aggregates & entities` — spec: "Backfill of PR #53 data.
   None is needed: PR #53 never persisted artifact_ref blocks … so no chat_messages row holds a
   ref." Why ambiguous: the shipped `migrations/027_chat_session_artifacts.sql` backfills
   `origin='tool_result'` rows from tool-role `artifact_ref` blocks (PR #54 had already persisted
   them). Amend the spec first. current/ says nothing about a backfill.
2. `ddd/contexts/artifacts.md § Published surface` (retrieval response headers) — spec:
   "`Content-Type: <row.mime>` … the signed URL carries the same Content-Type". Why ambiguous:
   the implementation serves and signs non-native-render MIMEs as `application/octet-stream`,
   not `row.mime`. current/ does not state the served Content-Type.
3. `c4/integrations.md` Object store / Probe and `assumptions.md` — spec: "MinIO and AWS S3 both
   behave this way with list permission." Why ambiguous: MinIO returns 404 (not 403) for a
   missing key even without list permission; the list-permission requirement and its
   boot-failure consequence apply to AWS S3 only. current/ names only the AWS S3 requirement.
4. `ddd/access-model.md § Policies` (Admin chat-artifacts row) — spec: "same contract as the
   deployed pair, with the BO preamble (GetSession(session_id, version_id))". Why ambiguous: BO
   returns `403` when the session belongs to another version (deployed returns `404`). current/
   does not state the BO status for a foreign-version session.
5. `ddd/contexts/feedback-datasets.md § Invariants` — spec: "Turns never take the advisory lock:
   a claim concurrent with an upload either sees the new row or leaves it pending". Why
   ambiguous: BO turns first run `UPDATE chat_sessions … RETURNING locked_at`, serialising
   concurrent BO turns on a session — likely below the implementation line. current/ says
   nothing about turn/upload locking.

## Links
- Source spec(s): `docs/superpowers/specs/2026-09-30-deployed-runtime-artifacts-design.md`
  (approved in https://github.com/DACdigital/openbbc-docs/pull/10, amended in
  https://github.com/DACdigital/openbbc-docs/pull/11); supersedes
  `docs/superpowers/specs/2026-09-28-chat-artifacts-design.md` for the artifact flow.
- Related prior arch-log entries:
  - `docs/architecture/logs/2026-09-28-artifact-support/README.md` — **extends** (implements the
    `deployed-runtime-artifacts` L2); **supersedes** its stateless-DB / no-identity stance, the
    top-level `/artifacts` route, the client-embedded ref flow, and `ARTIFACT_REF` as a
    top-level AG-UI event type.
  - `docs/architecture/logs/2026-09-29-sync-2026-09-28-chat-artifacts/README.md` —
    **supersedes** the per-surface retrieval split (BO `session_id` query authority), JSONB-scan
    authorisation, and the inner-ref pointer convention; **confirms** the drop of the
    assistant-emission leg (the glossary *Artifact* row and value-streams step 5 missed by that
    sync are fixed here).
  - `docs/architecture/logs/2026-09-30-multiagent-tools/README.md` — **supersedes in part**
    (spec: "This narrows the merged multiagent-tools architecture"): artifact-scope inheritance,
    `artifacts?` on the agent tool, and refs in its result; `user_id` / header inheritance
    unchanged.
