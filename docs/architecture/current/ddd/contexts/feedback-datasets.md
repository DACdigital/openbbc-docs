# feedback-datasets

## Purpose

Own the backoffice test-chat flow, the per-assistant-message feedback surface, and the
versioned dataset lifecycle. Sits downstream of `agent-lifecycle` (provides the version being
tested) and upstream of `evaluation` (which scores against a CLOSED dataset version).

## Aggregates & entities

- **Chat session** (aggregate root) — `chat_sessions` row. Scoped to `(agent_version_id, user)`.
  Owns `chat_messages[]`, `backend_header_overrides` (JSONB, migration 016; per-backend layout
  `{backend_id: {header: value}}`), and `locked_at` (flips on dataset close).
- **Child chat session** (entity within the root Chat session's tree) — a `chat_sessions`
  row created by the agent tool for one sub-agent run: `parent_session_id`,
  `parent_tool_call_id`, `depth` (root = 0), and the pinned target `agent_version_id`.
  Same `chat_messages` shape as any session. Inherits the root's user, artifact scope, and
  `backend_header_overrides` (applied to any backend id the sub-agent calls).
- **Chat message** (entity within Chat session) — `chat_messages` row. Turns; role ∈
  `{user, assistant, tool}` (DB check constraint includes `tool` since migration 009).
  `content` is a typed content-block list (JSONB): `text` blocks for prompt/completion
  text and `artifact_ref` blocks pointing to a blob in a configured artifact store — see
  [`artifacts`](artifacts.md). **No data migration ran** to convert historical rows:
  writers always emit the new array shape after ship (no dual-write logic); readers wrap
  legacy non-array (opaque-text) rows into a synthetic single-`text`-block list on load.
- **Feedback** (entity attached to assistant `chat_messages`) — `chat_message_feedback` row
  (migration 019). Fields: `rating` ∈ `{up, down}`, `comment`, `expected_output`,
  `judge_criteria` (JSONB array of acceptance-criteria bullets, migration 021).
- **Dataset** (aggregate root) — `datasets` row: identity + name.
- **Dataset version** (entity within Dataset) — `dataset_versions` row: `status ∈ {DRAFT,
  CLOSED}`, `version_num`, `close_note`.
- **Dataset-version session join** (link entity) — `dataset_version_sessions` row.

<!-- migrated from _migration-quarantine/ARCHITECTURE.md § Backoffice UI, § Feedback + datasets, § Chat header overrides on 2026-09-28 -->

## Domain events

N/A because OpenBBC does not emit domain events on any transport. State transitions in
this context are Postgres-only — `INSERT` / `UPDATE` against `chat_sessions`,
`chat_messages`, `chat_message_feedback`, `datasets`, `dataset_versions`,
`dataset_version_sessions` — and downstream consumers read via the REST surface. Draft
close (`POST /datasets/{id}/close-draft/confirm`) writes `status=CLOSED`, flips
`chat_sessions.locked_at`, and seeds the next DRAFT in a single transaction; there is no
event bus notification alongside.

## Invariants

- **At most one DRAFT per dataset** — partial unique index (migration 019).
- **A session belongs to at most one dataset** — repo-layer enforcement, not schema
  (migration 020 dropped schema uniqueness to allow cross-version reuse within one dataset).
- **CLOSED versions are immutable snapshots.** Closing a DRAFT flips
  `chat_sessions.locked_at` on all member sessions.
- **Cumulative seeding.** Closing a DRAFT immediately seeds the next DRAFT with the same
  session set (migration 020), so users see cumulative content rather than an empty next
  version.
- **Close-draft refuses when any member session has an empty `judge_criteria` list on any
  feedback row** (migration 021). Enforced repo-side.
- **Feedback only attaches to assistant-role messages** — enforced at the repo layer (Postgres
  doesn't do partial FKs) via `chat_message_feedback`.
- **Only root sessions are dataset members.** Child sessions cannot be assigned to a
  dataset; `assign-dataset` on a child returns `409`.
- **Locking cascades to the session tree.** Closing a DRAFT flips `locked_at` on each member
  root session **and** all its descendant child sessions, so their `artifact_ref`s stay
  resolvable for replay.
- **Feedback attaches to root-session assistant messages only.** Child transcripts are
  read-only in the BO (inspectable from the parent's tool call) — judge criteria describe
  the topology's user-visible behaviour, not individual workers.
- **`artifact_ref` blocks on locked sessions stay resolvable.** Closing a DRAFT flips
  `chat_sessions.locked_at`; from that point on, the `artifacts` context refuses deletion of
  any blob or `artifact_stores` row referenced by a locked session's messages (see
  [`artifacts.md § Invariants`](artifacts.md#invariants)). This is the replay-safety
  guarantee evals depend on.
- **Locked chat sessions refuse new artifact uploads.**
  `POST /chat-sessions/{id}/artifacts` returns `409` when `chat_sessions.locked_at IS NOT
  NULL` — a closed dataset version's session must remain a fixed snapshot for deterministic
  replay; accepting new bytes after close would silently mutate the eval input.

<!-- migrated from _migration-quarantine/ARCHITECTURE.md § Feedback + datasets, DESIGN.md § Phase II on 2026-09-28. Updated 2026-09-30 for multiagent-tools — child-session invariants. -->

## Published surface

- **REST / UI:**
  - `GET/POST /datasets`, `/datasets/{id}`, `/datasets/{id}/close-draft`,
    `/datasets/{id}/close-draft/confirm`
  - `POST /agent_versions/{v}/chat/{s}/assign-dataset`
  - `GET/POST /agent_versions/{v}/chat/{s}/headers` (header-override modal)
  - `GET /agent_versions/{id}/chat` and turn-level POSTs
  - `GET /agent_versions/{v}/chat/{s}/children/{child_id}` (read-only child transcript)
- **Emitted contract for `evaluation`:** CLOSED `dataset_version_id` + its
  `chat_sessions[]` with locked-at + feedback rows including `judge_criteria`.

<!-- migrated from _migration-quarantine/ARCHITECTURE.md § Backoffice UI, § REST API, § Feedback + datasets, § Chat header overrides on 2026-09-28 -->
