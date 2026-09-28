---
id: feedback-datasets
level: 2
parent: open-bbcd
title: feedback-datasets
---

# feedback-datasets

## Purpose

Backoffice test-chat surface (`/agent_versions/{v}/chat`), per-assistant-message feedback
capture, and versioned dataset lifecycle (DRAFT / CLOSED with cumulative seeding).

Owns tables: `chat_sessions` (+ `backend_header_overrides` JSONB per migration 016,
`locked_at` flip on dataset close), `chat_messages` (typed content-block JSONB — `text` +
`artifact_ref` blocks post artifact-support), `chat_message_feedback` (+ `judge_criteria`
JSONB array per migration 021), `datasets`, `dataset_versions` (partial unique index for
one-DRAFT-per-dataset, migration 019), `dataset_version_sessions`.

Publishes: `/agent_versions/{v}/chat[/{s}/*]` (turn POST, feedback POST, header-overrides
modal, assign-dataset), `/datasets/*` (create, list, detail, `/close-draft`,
`/close-draft/confirm`).

Invariants enforced here (repo layer, not schema): feedback attaches only to
assistant-role messages; a session belongs to at most one dataset; close-draft refuses when
any member session has an empty `judge_criteria` list.

Implements the [`feedback-datasets`](../../../ddd/contexts/feedback-datasets.md) DDD
context. Consumes the `artifact_ref` content-block shape published by the
[`artifacts`](../artifacts/README.md) L2.

## Scope (in / out)
