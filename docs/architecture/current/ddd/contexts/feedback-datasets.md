# feedback-datasets

## Purpose

Own the backoffice test-chat flow, the per-assistant-message feedback surface, and the
versioned dataset lifecycle. Sits downstream of `agent-lifecycle` (provides the version being
tested) and upstream of `evaluation` (which scores against a CLOSED dataset version).

## Aggregates & entities

- **Chat session** (aggregate root) — `chat_sessions` row. Scoped to `(agent_version_id, user)`.
  Owns `chat_messages[]`, `backend_header_overrides` (JSONB, migration 016; per-backend layout
  `{backend_id: {header: value}}`), and `locked_at` (flips on dataset close).
- **Chat message** (entity within Chat session) — `chat_messages` row. Turns; role ∈
  `{user, assistant}`.
- **Feedback** (entity attached to assistant `chat_messages`) — `chat_message_feedback` row
  (migration 019). Fields: `rating` ∈ `{up, down}`, `comment`, `expected_output`,
  `judge_criteria` (JSONB array of acceptance-criteria bullets, migration 021).
- **Dataset** (aggregate root) — `datasets` row: identity + name.
- **Dataset version** (entity within Dataset) — `dataset_versions` row: `status ∈ {DRAFT,
  CLOSED}`, `version_num`, `close_note`.
- **Dataset-version session join** (link entity) — `dataset_version_sessions` row.

<!-- migrated from _migration-quarantine/ARCHITECTURE.md § Backoffice UI, § Feedback + datasets, § Chat header overrides on 2026-09-28 -->

## Domain events

<!-- ARCH_GAP: source describes state changes (draft close, session lock, feedback insert) but does not name explicit domain events.
     Section: Domain events
     Fill with: candidate events (`DatasetDraftClosed`, `ChatSessionLocked`, `FeedbackCaptured`, `DatasetDraftSeeded`).
     See: .claude/skills/check-setup/arch-schema.md#ddd-context-file -->

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

<!-- migrated from _migration-quarantine/ARCHITECTURE.md § Feedback + datasets, DESIGN.md § Phase II on 2026-09-28 -->

## Published surface

- **REST / UI:**
  - `GET/POST /datasets`, `/datasets/{id}`, `/datasets/{id}/close-draft`,
    `/datasets/{id}/close-draft/confirm`
  - `POST /agent_versions/{v}/chat/{s}/assign-dataset`
  - `GET/POST /agent_versions/{v}/chat/{s}/headers` (header-override modal)
  - `GET /agent_versions/{id}/chat` and turn-level POSTs
- **Emitted contract for `evaluation`:** CLOSED `dataset_version_id` + its
  `chat_sessions[]` with locked-at + feedback rows including `judge_criteria`.

<!-- migrated from _migration-quarantine/ARCHITECTURE.md § Backoffice UI, § REST API, § Feedback + datasets, § Chat header overrides on 2026-09-28 -->
