# Data flows

## Discovery → alpha agent

```mermaid
sequenceDiagram
    participant DA as Discovery author
    participant CC as Claude Code plus flow map compiler
    participant Admin as Admin BO
    participant OBBCD as open bbcd
    participant DB as postgres
    participant Runner as aikdm runner drainer
    participant AIK as aikdm

    DA->>CC: run skill against target frontend repo
    CC-->>DA: flow map tree plus zip
    Admin->>OBBCD: POST agents new, upload zip via wizard
    OBBCD->>DB: INSERT agents with architecture, capabilities, discovery_zip BYTEA per mig 026
    OBBCD->>DB: INSERT agent_versions v1 with status INITIALIZING
    Admin->>OBBCD: configure scope, guardrails, personality, bind tool_backends per endpoint
    Admin->>OBBCD: POST Finalize
    OBBCD->>DB: UPDATE agent_versions set status PENDING per mig 025
    Note over Runner: cron every 5 min, process_pending_alphas.sh, k8s CronJob cronjob alphas
    Runner->>OBBCD: GET agent_versions.json filtered by status PENDING
    OBBCD-->>Runner: list of PENDING versions
    loop per PENDING version
        Runner->>OBBCD: GET config for version
        Runner->>AIK: uv run aikdm generate agent
        AIK-->>Runner: bundle yaml
        Runner->>DB: seed_bundle.py UPSERT prompts and UPDATE status READY in one txn
    end
    OBBCD-->>Admin: PENDING badge flips to READY badge
```

<!-- migrated from _migration-quarantine/DESIGN.md § Phase 0, § Phase I, ARCHITECTURE.md § Data Flow / Flow 1 on 2026-09-28. Updated 2026-09-28 for OpenBBC PR #50 — Finalize is now an async transition to PENDING, drained by the aikdm-runner CronJob; mig 026 keeps the zip inline. Rewritten 2026-09-28 to be safe against GitHub's strict mermaid parser (no slashes, hyphens, em dashes, parens, semicolons, or asterisks in message bodies). -->

## Feedback → dataset

```mermaid
sequenceDiagram
    participant Admin as Admin BO
    participant OBBCD as open bbcd
    participant DB as postgres
    participant BE as Client backend

    Admin->>OBBCD: GET agent_versions v chat
    OBBCD->>DB: INSERT chat_sessions
    Admin->>OBBCD: POST turn with user message
    OBBCD->>BE: tool call with backend_header_overrides if set
    BE-->>OBBCD: tool result
    OBBCD-->>Admin: assistant reply
    Admin->>OBBCD: POST feedback with rating, comment, expected_output, judge_criteria
    OBBCD->>DB: INSERT chat_message_feedback
    Admin->>OBBCD: POST chat s assign dataset
    OBBCD->>DB: INSERT dataset_version_sessions on DRAFT
    Admin->>OBBCD: POST datasets id close draft confirm
    OBBCD->>DB: UPDATE dataset_versions set status CLOSED, UPDATE chat_sessions set locked_at now, INSERT next DRAFT seeded from CLOSED
    OBBCD-->>Admin: dataset detail with CLOSED v_n and DRAFT v_n plus 1
```

<!-- migrated from _migration-quarantine/DESIGN.md § Phase II, ARCHITECTURE.md § Feedback + datasets, § Data Flow / Flow 2, § Chat header overrides on 2026-09-28. Rewritten 2026-09-28 for GitHub-safe mermaid syntax. -->

## Evaluation

```mermaid
sequenceDiagram
    participant Admin as Admin BO
    participant OBBCD as open bbcd
    participant DB as postgres
    participant Op as Operator or drainer
    participant AIK as aikdm
    participant BE as Client backend

    Admin->>OBBCD: POST evals with agent_version_id, dataset_version_id, mock_mcp_tools, header_overrides
    OBBCD->>DB: INSERT evals with status PENDING
    Op->>OBBCD: GET evals id export yaml
    Op->>OBBCD: POST evals id start
    OBBCD->>DB: UPDATE evals set status IN_PROGRESS
    Op->>AIK: uv run aikdm evaluate
    loop per session
        AIK->>AIK: simulator then target then tool_mock or real MCP call
        alt mock_mcp_tools is false
            AIK->>BE: MCP tool call with header_overrides
            BE-->>AIK: tool result
        end
        AIK->>AIK: judge scores each criterion
    end
    AIK-->>Op: eval result json
    Op->>OBBCD: POST evals id result or fail
    OBBCD->>DB: UPDATE evals set status DONE plus score, passed_criteria, total_criteria, INSERT eval_sessions
    OBBCD-->>Admin: eval detail with per session breakdown plus Train button if score below 1.0
```

<!-- migrated from _migration-quarantine/DESIGN.md § Phase III, ARCHITECTURE.md § Evals, § Data Flow / Flow 3, § Chat header overrides on 2026-09-28. Rewritten 2026-09-28 for GitHub-safe mermaid syntax. -->

## Automated training

```mermaid
sequenceDiagram
    participant Admin as Admin BO
    participant OBBCD as open bbcd
    participant DB as postgres
    participant Op as Operator or drainer
    participant AIK as aikdm

    Admin->>OBBCD: POST training sessions via Train button
    OBBCD->>DB: INSERT training_sessions with status PENDING, source_eval_id, parent_version_id
    Op->>OBBCD: GET training sessions id json
    Op->>OBBCD: GET evals source_eval_id export yaml
    Op->>OBBCD: POST training sessions id start
    OBBCD->>DB: UPDATE training_sessions set status IN_PROGRESS
    Op->>AIK: uv run aikdm train agent with epochs N and patience K
    AIK->>AIK: baseline eval on parent version
    loop each epoch until early stop or perfect score
        AIK->>AIK: teacher LLM proposes patches
        AIK->>AIK: apply then run eval as reward
        AIK->>AIK: promote if candidate score greater than best, else record non improvement
    end
    AIK-->>Op: bundle yaml plus training report json plus score diff
    Op-->>Admin: interactive prompt for yes or no via script
    Op->>OBBCD: POST training sessions id complete or fail
    OBBCD->>DB: INSERT agent_versions new row, UPDATE training_sessions set status DONE plus new_version_id and training_report
    OBBCD-->>Admin: new agent version linked to the training session
```

<!-- migrated from _migration-quarantine/DESIGN.md § Phase IV, ARCHITECTURE.md § Training sessions, § Data Flow / Flow 4 on 2026-09-28. Rewritten 2026-09-28 for GitHub-safe mermaid syntax. -->

## Multimodal chat with artifacts

```mermaid
sequenceDiagram
    participant User as End user or Admin
    participant OBBCD as open bbcd
    participant DB as postgres
    participant STORE as Object store
    participant BE as Client backend

    Note over OBBCD, STORE: precondition, operator declared at least one ARTIFACT_STORE via env at deploy time and set ARTIFACT_STORE_DEFAULT
    User->>OBBCD: POST sessions sid artifacts multipart file
    OBBCD->>STORE: put bytes bounded by ARTIFACT_MAX_UPLOAD_MB via adapter
    STORE-->>OBBCD: uri plus size_bytes plus sha256
    OBBCD-->>User: 201 with store_id uri mime size sha256

    User->>OBBCD: POST turn user content is text plus artifact_ref block
    OBBCD->>DB: INSERT message content JSONB with typed blocks
    OBBCD->>BE: tool call, artifact arg materialised as inline bytes or presigned url per backend contract
    BE-->>OBBCD: tool result with ImageContent or EmbeddedResource
    OBBCD->>STORE: put unpacked bytes via adapter
    STORE-->>OBBCD: uri
    OBBCD->>DB: INSERT tool role message with artifact_ref block

    OBBCD-->>User: assistant reply on chat or ARTIFACT_REF event on AG UI stream
    User->>OBBCD: GET artifacts store_id uri with session_id user_id
    OBBCD->>STORE: get or sign uri
    STORE-->>OBBCD: bytes or signed url
    OBBCD-->>User: bytes or 302 redirect
```

<!-- new data flow added 2026-09-28 for artifact-support. GitHub-safe mermaid syntax per diagram conventions. -->

## Multi-agent delegated turn

```mermaid
sequenceDiagram
    participant User as End user or Admin
    participant OBBCD as open bbcd root orchestrator
    participant SUB as open bbcd sub agent run
    participant DB as postgres
    participant LLM as LLM provider
    participant BE as Client backend

    Note over OBBCD, DB: precondition, root version has agent_tool_enabled and pinned agent_version_subagent rows
    User->>OBBCD: POST turn on root session
    OBBCD->>DB: INSERT user message on root session
    OBBCD->>LLM: completion with agent tool whose subagent enum lists binding names and notes
    LLM-->>OBBCD: tool call agent with subagent, description, prompt, optional artifact refs
    OBBCD-->>User: TOOL_CALL_START then STEP_STARTED named after the binding
    par up to AGENT_TOOL_MAX_PARALLEL spawns, refused past AGENT_TOOL_MAX_DEPTH
        OBBCD->>DB: INSERT child session with parent_session_id, parent_tool_call_id, depth, target version, root user_id
        OBBCD->>SUB: run pinned target version with fresh context holding only prompt and artifacts
        SUB->>LLM: completion with the target version tools
        SUB->>BE: MCP tool call via target wiring, header_overrides inherited on BO and eval paths
        BE-->>SUB: tool result
        SUB-->>User: child tagged TOOL_CALL events, no sub agent text tokens
        SUB->>DB: INSERT child messages
        SUB-->>OBBCD: final text plus artifact refs
    end
    OBBCD-->>User: STEP_FINISHED then TOOL_CALL_END with the sub agent result
    OBBCD->>LLM: continue root completion with tool result
    LLM-->>OBBCD: final assistant reply
    OBBCD->>DB: INSERT tool and assistant messages on root session
    OBBCD-->>User: TEXT_MESSAGE events then TURN_END
```

A sub-agent run is the same turn loop as the root, entered in-process for a different
(pinned) agent version and persisted as a child session. It may itself spawn sub-agents
if its version has the agent tool enabled; depth is counted from the root. In
`aikdm evaluate` the same shape runs inside aikdm, with pinned bundles from
`eval-input.yaml` instead of Postgres and simulated or real MCP calls per
`mock_mcp_tools`.

<!-- new data flow added 2026-09-30 for multiagent-tools. GitHub-safe mermaid syntax per diagram conventions. -->

## Deployment

```mermaid
sequenceDiagram
    participant Admin as Admin BO
    participant User as End user
    participant GW as Operator gateway
    participant OBBCD as open bbcd
    participant DB as postgres
    participant BE as Client backend

    Admin->>OBBCD: POST agents agent_id deploy
    OBBCD->>DB: UPDATE agent_versions set is_deployed true, rotate any prior DEPLOYED in the chain
    User->>GW: chat request with session cookie or bearer or mTLS
    GW->>OBBCD: POST deployed agent_id sessions with verified rewritten user_id
    OBBCD->>DB: INSERT deployed_sessions
    OBBCD-->>GW: 201 Created with session id
    GW-->>User: session id
    User->>GW: POST turn with content, session id, user_id
    GW->>OBBCD: POST deployed agent_id sessions sid turn with verified user_id
    OBBCD->>DB: INSERT deployed_messages user turn
    loop AG UI event stream
        OBBCD-->>GW: emits RUN_STARTED then TEXT_MESSAGE then TOOL_CALL then TURN_END events
        alt agent decides to call a tool
            OBBCD->>BE: MCP tool call using server to server credentials only, no per session header override on deployed path today
            BE-->>OBBCD: tool result
        end
    end
    OBBCD->>DB: INSERT deployed_messages assistant turn
    GW-->>User: streamed AG UI events
```

<!-- migrated from _migration-quarantine/DESIGN.md § Phase V, ARCHITECTURE.md § Data Flow / Flow 5, PRODUCTION.md § 2 Integrating your frontend, § 4 Headers, § 5 Auth model on 2026-09-28. Rewritten 2026-09-28 for GitHub-safe mermaid syntax. -->
