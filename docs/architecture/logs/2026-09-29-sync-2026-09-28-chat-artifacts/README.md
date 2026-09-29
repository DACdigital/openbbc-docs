---
date: 2026-09-29
codename: sync-2026-09-28-chat-artifacts
kind: sync
---

# Sync — align current/ with 2026-09-28-chat-artifacts

## Driver
Post-approval sync of `docs/architecture/current/` with the following spec:
- `docs/superpowers/specs/2026-09-28-chat-artifacts-design.md` (merged in PR #7)

## Decision
Apply the CONTRACT_SHAPE deltas listed below to `docs/architecture/current/`. Additionally apply
the four-leg → three-leg reduction across all affected `current/` surfaces: the spec explicitly
drops "assistant emission" (leg 3) on the grounds that the LLM does not itself generate binary
content — only tools return artifacts to the assistant. The user (arch owner) confirmed this
reduction during the `/arch-review --apply` invocation.

## Rationale
Approved spec is the design contract; `current/` is the architecture-of-record and must reflect
it. Per project rule: L2 modularity nodes are inspirational — specs are the impl contract; sync
applies CONTRACT_SHAPE deltas without altering module Purpose, and only introduces new L1/L2
surface for SCOPE_MOVE deltas. The three-leg reduction was flagged AMBIGUOUS by
`architecture-review` because the spec deferred the arch amendment to a "separate motion";
the arch owner resolved the ambiguity in this session by explicitly authorising the reduction,
which makes this sync the intended vehicle.

## Alternatives rejected
- Leaving current/ stale until the next `/modularize` run — rejected: drift accumulates
  silently and a new dev reading current/ builds against the wrong contract.
- Applying the CONTRACT_SHAPE deltas but leaving the four-leg framing intact — rejected:
  three-leg is now the arch owner's explicit position, and leaving `bizbok/capabilities.md`,
  `assumptions.md`, `information-map.md`, and the L2 modularity node describing a leg the
  design has ruled out would silently mislead readers.
- Auto-applying every AMBIGUOUS delta — rejected as a general policy (silent override of
  prior arch-log decisions is unsafe); resolved here only because the arch owner explicitly
  authorised the specific reduction.

## Impact

### CONTRACT_SHAPE deltas applied (no purpose change)
- `docs/architecture/current/ddd/contexts/artifacts.md § Aggregates & entities` — added
  optional `filename` field to the `artifact_ref` value-object shape with the "display-only,
  never used to construct storage URIs" invariant; added the `{store_id, uri}` inner-ref
  pointer convention for tool-call re-references.
- `docs/architecture/current/ddd/contexts/artifacts.md § Published surface` — split the
  retrieval-route contract by surface (BO uses `session_id` alone as authoritative under the
  operator-gateway trust boundary; deployed additionally requires `user_id`); documented the
  `200`-bytes-vs-`302`-redirect response driven by `PreferredDelivery()`; added `409` on
  upload-to-locked-session; added `410` on externally-removed blob (`stat.exists == false`);
  clarified `404` also covers unknown `store_id`; added `filename` to the upload response
  shape; added the new "Emitted contract for tool-call / tool-result payloads" sub-section
  describing the `{store_id, uri}` inner-ref pointer convention.
- `docs/architecture/current/ddd/contexts/artifacts.md § Consumed contract from deploy env` —
  added `ARTIFACT_SIGNED_URL_TTL_SECONDS`; tightened boot-error surface to include duplicate
  `<ID>`; explicit `PreferredDelivery` in the "Consumed contract from external object store"
  bullet.
- `docs/architecture/current/c4/integrations.md § Contracts (artifact-store adapter interface)` —
  rewrote the adapter contract to lead with `PreferredDelivery() → Bytes | SignedURL`; `get`
  and `sign` are now described as delivery paths driven by that declaration (not parallel
  alternatives); explicit `Exists` in `stat`; enumerated the boot-fail conditions on `probe`
  (unset default, failed probe after retries, malformed group, duplicate `<ID>`, unset
  `ARTIFACT_MAX_UPLOAD_MB`).
- `docs/architecture/current/constraints.md § Hard technical limits` — added
  `ARTIFACT_SIGNED_URL_TTL_SECONDS` bullet (optional, default `300`, TTL passed to
  `adapter.Sign` when `PreferredDelivery == SignedURL`).
- `docs/architecture/current/ddd/contexts/feedback-datasets.md § Aggregates & entities` —
  tightened the Chat message content-shape normalisation to explicitly say no data migration
  ran, writers always emit the new array shape after ship, readers wrap legacy non-array
  rows into a synthetic single-`text`-block list on load.
- `docs/architecture/current/ddd/contexts/feedback-datasets.md § Invariants` — added
  "Locked chat sessions refuse new artifact uploads (`409` when
  `chat_sessions.locked_at IS NOT NULL`)".

### SCOPE_MOVE deltas applied (new surface / ownership / boundary)
- None. The spec ships a slice (backoffice test-chat only) of the arch-of-record already
  captured in `docs/architecture/logs/2026-09-28-artifact-support/` and does not introduce a
  new bounded context, persistence store, event channel, or ownership boundary beyond what
  current/ already documents. Deployed-runtime-artifacts remains a separate follow-on.

### AMBIGUOUS delta resolved by arch owner and applied
- Four-leg → three-leg reduction (drop "assistant emission"). Applied across:
  - `docs/architecture/current/bizbok/capabilities.md § L2 chat-artifacts` — rewritten as
    a three-leg description (admin upload, MCP tool result, agent-to-tool inner-ref).
  - `docs/architecture/current/bizbok/capabilities.md § L2 deployed-runtime-artifacts` —
    rewritten as "same three-leg pattern"; assistant-emitted phrasing replaced with
    tool-produced.
  - `docs/architecture/current/bizbok/information-map.md § Artifact row` — three legs.
  - `docs/architecture/current/assumptions.md § Design decisions (locked) (pluggable
    artifact-store adapter)` — "all three legs" with an explicit "no assistant-emission
    leg" note.
  - `docs/architecture/current/modularity/open-bbcd/artifacts/README.md § Purpose` — three
    legs.
  - `docs/architecture/current/ddd/contexts/deployed-runtime.md § Invariants (AG-UI
    event-type extension)` — "tool-produced artifacts stream as `ARTIFACT_REF`" (no
    assistant-emission leg).

### AMBIGUOUS deltas NOT applied — human review required
- None remaining. The single AMBIGUOUS delta from `architecture-review` was resolved in this
  session by the arch owner and applied.

## Links
- Source spec: `docs/superpowers/specs/2026-09-28-chat-artifacts-design.md` (merged in
  https://github.com/DACdigital/openbbc-docs/pull/7)
- Related prior arch-log entries:
  - `docs/architecture/logs/2026-09-28-artifact-support/README.md` — **extends** (this spec
    is the first implementation slice; the CONTRACT_SHAPE deltas sharpen contracts left at
    capability-level; the leg-3-drop **supersedes** that log's four-leg Decision).
