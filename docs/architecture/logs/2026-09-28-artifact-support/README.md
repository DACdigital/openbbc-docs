# artifact-support — chat and deployed-runtime artifacts in both directions via pluggable storage

**Date**: 2026-09-28
**Codename**: artifact-support

**Driver**: new feature: artifact-support. A downstream project consuming OpenBBC has
repeatedly asked for chat artifacts (files in and out of chat, MCP-tool-produced files
surfacing to the agent, agent-generated files reaching the frontend). Making this a
first-class framework capability — rather than a bespoke build per deployer — matches
OpenBBC's ship pattern of "framework interface + deployer picks the infra".

**Decision**: add a new L1 capability `Artifact management` plus per-surface L2s
(`chat-artifacts` under Feedback & dataset curation, `deployed-runtime-artifacts` under
Deployed agent runtime), a new `artifacts` bounded context, and the `artifact-store-adapter`
integration abstraction. Artifacts are exchanged in all four legs (user upload, MCP tool
output normalised from `ImageContent` / `EmbeddedResource`, agent emission, agent-to-tool
argument), on both chat surfaces (backoffice and deployed runtime), with `chat_messages.content`
and `deployed_messages.content` moving from opaque-text JSONB to a typed content-block list
that includes `artifact_ref` blocks. Bytes live in a deployer-configured Object store reached
through a pluggable `artifact-store-adapter`; `open-bbcd` holds only refs and an
`artifact_stores` config row (kind + opaque config + `is_default` flag). First shipped adapter
kind: `s3_compatible` (covers AWS S3, MinIO, GCS-HMAC, R2, B2, any S3-API endpoint). Uploads
gated by env `ARTIFACT_MAX_UPLOAD_MB`. Deployed runtime carries artifacts on the AG-UI wire
via a new `ARTIFACT_REF` event-type extension.

**Rationale**: One downstream OpenBBC consumer needs this now; treating it as framework
substrate rather than deployer-side plumbing prevents N bespoke implementations across future
deployers. Keeping bytes out of Postgres preserves the "no local disk state" invariant's
operational intent — Postgres is state, blob storage is content — and protects backup /
replication / vacuum from media-workload pressure. The adapter pattern mirrors `tool_backends`
(kinds + opaque config + one-of-many selectable at runtime), which the codebase already
practises, so no new architectural pattern is introduced. Session-scoped read authorisation on
`GET /artifacts/{store_id}/{uri}` (404 on mismatch) reuses the same trusted-`user_id` model
that already scopes messages, keeping the trust-boundary story consistent. Eval replay stays
deterministic through the invariant "refs stay resolvable while any locked session references
them", upheld by `artifacts` refusing deletion of blobs referenced by locked chat sessions.

**Alternatives rejected**:
- **Store artifact bytes inline in Postgres (BYTEA on `chat_messages.content` or a sibling
  blob column)** — rejected. DB size would explode under real media workloads; the
  "no local disk state" invariant added in migration 026 was about *operational* state, and
  loading Postgres with multi-MB media (or GB video) works against the same intent (backup
  size, replication lag, vacuum pressure). Postgres large-object API was not evaluated
  separately because it inherits the same DB-bloat cost class. Pass-through references only
  (no framework-side storage) was implicitly rejected too — it moves the replay-safety
  problem onto every deployer, and closed dataset versions must remain resolvable months
  later.

**Impact**:
- `current/README.md` — added Object store to the top-level "System at a glance" diagram.
- `current/nfrs.md` — Security section extended with artifact-store credentials and
  session-scoped ref access model; Compliance section extended with the new user-content
  locus outside Postgres.
- `current/assumptions.md` — added locked decision "Artifacts live in a pluggable
  artifact store; open-bbcd holds only refs" (design decision, dated 2026-09-28).
- `current/constraints.md` — added `ARTIFACT_MAX_UPLOAD_MB` env cap, one-`is_default`
  invariant, no-bytes-in-Postgres rule, kind-versioning rule.
- `current/glossary.md` — added Artifact, Artifact store, Artifact-store adapter, Content
  block.
- `current/bizbok/capabilities.md` — added L1 "Artifact management" (position 8) with L2s
  `artifact-store-management`, `artifact-store-adapter`; added L2 `chat-artifacts` under
  Feedback & dataset curation; added L2 `deployed-runtime-artifacts` under Deployed agent
  runtime; capability→context/container map extended with four new rows.
- `current/bizbok/information-map.md` — added Artifact and Artifact store rows; updated
  Chat message and Deployed message rows to reflect typed content-block semantics.
- `current/bizbok/value-streams.md` — added new "Multimodal chat with artifacts" stream
  covering the four legs across both chat surfaces.
- `current/ddd/context-map.md` — added `artifacts` context and its OHS relationships
  (to feedback-datasets and deployed-runtime) plus its ACL relationship to the external
  Object store; added Conformist relationship feedback-datasets → evaluation for ref replay;
  extended the diagram.
- `current/ddd/contexts/artifacts.md` — NEW context file.
- `current/ddd/contexts/feedback-datasets.md` — Chat-message entity updated for typed
  content blocks; added invariant "artifact_ref blocks on locked sessions stay resolvable".
- `current/ddd/contexts/deployed-runtime.md` — Deployed-message entity updated for typed
  content blocks; added invariants for session-scoped artifact reads and the AG-UI
  `ARTIFACT_REF` event-type extension; published surface extended with the artifact upload
  route and the `ARTIFACT_REF` emitted event.
- `current/ddd/access-model.md` — added End-user policy for `deployed-runtime-artifacts`;
  extended Admin policies to include `artifact-store-management` and `chat-artifacts`.
- `current/c4/context.md` — added Object store as a new external system, edge from OpenBBC.
- `current/c4/containers.md` — extended C4-Container diagram with Object store external and
  the corresponding edge; open-bbcd data-ownership updated to note `artifact_stores` config
  and adapter dispatch; DDD-context list updated to include `artifacts`.
- `current/c4/integrations.md` — added Object store row; documented the artifact-store
  adapter contract and the AG-UI `ARTIFACT_REF` event-type extension under Contracts.
- `current/c4/data-flows.md` — added new "Multimodal chat with artifacts" sequence diagram
  showing upload, tool passthrough, and retrieval.
- `current/c4/deployment.md` — Object store added to External integration zone; deployment
  diagram edge added; App↔External trust boundary paragraph extended; threat model gained
  three new rows (artifact-store credential leak, ref-based cross-user read, oversized
  upload DoS).
- + `docs/architecture/logs/2026-09-28-artifact-support/README.md` (this file).

**Links**:
- (user may add related spec PRs or tracker items before merge)
