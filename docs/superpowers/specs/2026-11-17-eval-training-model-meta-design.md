# eval-training-model-meta — record and show which models scored and trained an agent

**Date:** 2026-11-17
**Status:** Draft — pending `/spec-review`
**Modularity attachment:**
- primary: [`open-bbcd/training`](../../architecture/current/modularity/open-bbcd/training/README.md)
  (L2), which owns the training-session state and its BO pages;
- also [`open-bbcd/evaluation`](../../architecture/current/modularity/open-bbcd/evaluation/README.md)
  (eval BO pages);
- also [`aikdm/training-loop`](../../architecture/current/modularity/aikdm/training-loop/README.md)
  (the training report);
- also [`open-bbcd/agent-lifecycle`](../../architecture/current/modularity/open-bbcd/agent-lifecycle/README.md)
  (the agent versions tab).

**Grounded against:** OpenBBC `main` after #64. Paths are relative to the OpenBBC repo root.

## Business value / Why

A score only means something if you know which models produced it. Today:

- **Evals record the models, but nobody can see them.** aikdm already sends `aikdm_meta` with
  each eval result: simulator, judge and target models, user-turn mode, history mode and
  duration (`aikdm/aikdm/eval/orchestrator.py:97-103`). open-bbcd stores it in
  `evals.aikdm_meta`, but no BO template shows it.
- **Trainings don't record the models at all.** `TrainingReport` (`aikdm/aikdm/train/schemas.py:44-53`)
  has scores, epochs and patches, but no models. A promoted agent version cannot say which
  target model it was tuned against, or which teacher and judge produced it.
- **The scoring model is silently decoupled from the runtime model.** The target model in evals
  and trainings is aikdm's `AIKDM_MODEL_TARGET` (default `claude-haiku-4-5`). The model the
  agent actually runs on is open-bbcd's `OPENBBC_DEFAULT_MODEL` (default `claude-sonnet-4-6`).
  There is no per-agent model; the global default is the deployed model
  (`internal/config/llm.go:78`).
- **Deployments can't choose these models today.** aikdm runs in CronJob pods
  (`scripts/process_pending_*.sh`), not inside open-bbcd. Neither the Helm chart
  (`cronjob-evals.yaml`, `values.yaml:116-121`) nor `docker-compose.yml` passes any
  `AIKDM_MODEL_*` / `AIKDM_EVAL_*` variable. Every eval and training in a deployed
  environment therefore runs on the Haiku defaults, whatever the agent runs on.

Success is judged by:
- For any eval or training, a PL can see in the BO which target, judge, simulator (and, for
  training, teacher) models and which eval modes produced the score.
- An agent version created by training shows which target model it was trained against.
- When the scoring target model differs from the model this open-bbcd runs agents on, the BO
  says so on the eval and training pages.
- Operators can set the aikdm models and modes in Helm and docker-compose without editing a
  secret.

## Change level

**C2.** It adds a field to the cross-service `training-report-v1` contract (aikdm →
`POST /training-sessions/{id}/complete`). Under `docs/conventions/api.md`, a contract change is C2
even when it is additive. Everything else is display, plus deployment config with unchanged
defaults:
- no DB migration (`training_report` is already stored as JSONB);
- no API route change, no event change;
- no change to `eval-result-v1` (`aikdm_meta` already carries the eval models).

## Scope

### In scope

1. **Training report `models` block** (aikdm). `TrainingReport` gains a required field:

   ```python
   class TrainingModels(BaseModel):
       model_config = ConfigDict(extra="forbid")
       teacher: str
       judge: str
       user_simulator: str
       target: str
       user_turn_mode: Literal["verbatim", "paraphrase"]
       history_mode: Literal["simulated", "reference"]

   class TrainingReport(BaseModel):
       ...
       models: TrainingModels
   ```

   - It is filled from `Settings` in `run_training` (`aikdm/aikdm/train/orchestrator.py:26`) at
     both places a report is built (`:49` perfect-baseline early return, `:167` normal end).
   - The values are the model strings **as configured** (`settings.model_teacher`, etc.), the same
     convention `aikdm_meta` uses for evals. They are not normalized.
   - `schema_version` stays `training-report-v1`, because the change is additive. open-bbcd decodes
     the report as raw JSON and never checks the version (`handler/training_session.go:156-228`).

2. **One normalization rule for comparing models** (open-bbcd, display only).
   `ui.CanonicalModel(s string) string` returns `provider/model`:
   - a string containing `/` is returned unchanged;
   - a bare `claude*` becomes `anthropic/<s>`;
   - a bare `gpt*` or `openai*` becomes `openai/<s>`;
   - a bare `gemini*` becomes `gemini/<s>`;
   - anything else is returned unchanged.

   This mirrors aikdm's `models.py:15-25`. The runtime model is canonicalized the same way:
   - anthropic adapter: `anthropic/<cfg.LLM.Model>`;
   - bifrost adapter: `<cfg.LLM.Provider>/<cfg.LLM.Model>` (`internal/config/llm.go:94-118`).

   The function lives where both the eval and training handlers can use it. Display always shows
   the raw configured string; canonical forms are used only to compare.

3. **Eval pages** (open-bbcd, `internal/handler/eval.go`, `web/templates/eval/*`).
   - **Detail** (`UIDetail`, `eval/detail.html` meta `<dl>` at :39-42): add rows for target model,
     judge model, user-simulator model, user-turn mode and history mode, parsed from
     `Eval.AikdmMeta`.
   - **Lists** (`eval/list.html`, `eval/list_per_agent_version.html`): one new column,
     "Target / judge", showing `<target> / <judge>`. The data is already selected
     (`repository/eval.go` `evalColumns` includes `aikdm_meta`).
   - When `aikdm_meta` is empty or lacks a key (old evals, failed evals, `PENDING` ones), show `—`.
     Unknown extra keys are ignored.

4. **Training pages** (open-bbcd, `internal/handler/training_session.go`, `web/templates/training/*`).
   - **Detail** (`trainingDetailPageData`, `training/detail.html` `<dl>` at :8-30): add rows for
     teacher, judge, user-simulator and target models and both modes, parsed from
     `TrainingSession.TrainingReport`'s `models`. This sits next to `parseEpochsForView`, as a
     `parseModelsForView`.
   - **List** (`training/list.html`): one new column, "Target model".
   - A report without `models` (trainings completed before this change) shows `—`.

5. **Agent versions tab: "trained on"** (open-bbcd, `internal/handler/agent_detail.go:189-260`,
   `web/templates/agent-detail/versions.html`).
   - For a version created by a training session, show "Trained on `<target>`", linked to that
     training session's detail page.
   - The data comes from
     `LEFT JOIN training_sessions ts ON ts.new_version_id = av.id` (indexed,
     `migrations/024_training_sessions.sql:28`) in `AgentRepository.ListGrouped`, reading
     `ts.id` and `ts.training_report->'models'->>'target'`. There is no new column.
   - Versions not created by training, or created before this change, show nothing.

6. **Runtime-model mismatch notice** (open-bbcd).
   - The eval detail and training detail pages show a one-line notice when
     `CanonicalModel(target) != CanonicalModel(runtime model)`:
     `Scored on <target>; this open-bbcd runs agents on <runtime>.`
   - The runtime model is passed into `NewEvalHandler` / `NewTrainingSessionHandler` from `cfg`
     in `newAPI` (`internal/handler/api.go:78, 222, 227`), using the same setter pattern as
     `WithClientContextMaxBytes`.
   - The notice is informational. It doesn't block anything and doesn't appear in lists, and no
     notice is shown when the target is unknown (`—`).

7. **Deployment config for aikdm models and modes**, with defaults unchanged.
   - **Helm:** a new `aikdm.env` map in `values.yaml` (default `{}`), rendered as plain `env:`
     entries on the eval and training CronJob containers next to `OPENBBCD_URL`. It is not part of
     the secret. `values.yaml` documents these keys:
     - `AIKDM_MODEL_TARGET`
     - `AIKDM_MODEL_JUDGE`
     - `AIKDM_MODEL_USER_SIMULATOR`
     - `AIKDM_MODEL_TEACHER`
     - `AIKDM_EVAL_USER_TURN_MODE`
     - `AIKDM_EVAL_HISTORY_MODE`
   - **docker-compose:** the `aikdm` service passes the same six variables through from the host
     environment (`${VAR:-}`). An unset variable keeps aikdm's default.
   - **aikdm README:** the "Eval and training modes" section gets one line saying these are set on
     the aikdm runner (Helm `aikdm.env` / compose), not on open-bbcd. This corrects the earlier
     wording that pointed at open-bbcd's environment.

8. **Tests.**
   - **aikdm:** `TrainingReport` requires `models`, and both report-building paths fill it from
     `Settings` (`test_train_orchestrator.py`, `test_train_schemas.py`). Update the `train-agent`
     CLI integration test to assert `models` in the written report.
   - **Go `CanonicalModel`:** a table test covering bare claude/gpt/gemini, already-prefixed
     values and unknown values. Runtime canonicalization covers both adapters.
   - **Go eval handler:**
     - the detail page renders all five rows from a populated `aikdm_meta`;
     - it renders `—` for `{}`;
     - the list shows `target / judge`;
     - the notice appears on a mismatch and not on a match (bare vs `anthropic/` prefixed counts
       as a match).
   - **Go training handler:** the same as eval for detail, list and notice, plus the report
     without `models`.
   - **Go agent detail / repository:** `ListGrouped` returns the training session id and target
     for a version created by training, and nothing for others. The versions tab renders the
     link. Follow the pattern of `eval_gate_test.go:344`.
   - **Helm:** `helm template` with `aikdm.env` set renders the env entries on both CronJobs.
     With the default, the rendered output is identical to today's.

### Out of scope

- **A per-agent or per-deployment runtime model.** The global `OPENBBC_DEFAULT_MODEL` remains the
  deployed model.
- **Making aikdm use open-bbcd's runtime model** automatically as the target. The notice makes
  the mismatch visible; choosing the model stays an operator decision.
- **BO chat test sessions** (manual chats) recording which model answered.
- **Changes to `eval-result-v1`.** Its `aikdm_meta` already carries the models.
- **Filtering or sorting lists by model.**
- **Showing generator or critic models** (`aikdm generate-agent`). Those aren't scoring models.
- **Backfilling** old trainings, which have no `models` in their stored report.

## Contracts

- **`training-report-v1`** (aikdm → `POST /training-sessions/{id}/complete`, field
  `training_report`): new required (from aikdm) object `models`:
  `{teacher, judge, user_simulator, target: string, user_turn_mode: "verbatim"|"paraphrase",
  history_mode: "simulated"|"reference"}`.
  - The change is additive and the version is not bumped.
  - open-bbcd stores the report as opaque JSONB and extracts only the four summary scalars, so
    older open-bbcd versions accept and ignore it.
  - The new open-bbcd treats `models` as optional when reading, because older stored reports
    don't have it.
- **`eval-result-v1`:** unchanged.
- **API routes, DB, events:** no change. No migration.
- **Deployment config:**
  - new optional Helm value `aikdm.env` (map, default `{}`);
  - docker-compose passes six optional `AIKDM_*` variables through to the `aikdm` service.

  Defaults and behaviour are unchanged when these are unset.

## Acceptance criteria

1. Every training report written by `aikdm train-agent` contains a `models` block whose values
   equal the effective `Settings` for that run, in both the perfect-baseline and normal paths.
2. The eval detail page shows target, judge and user-simulator models plus both modes, from
   `aikdm_meta`. The eval lists show "Target / judge". Missing data renders `—`, never an error.
3. The training detail page shows teacher, judge, user-simulator and target models plus both
   modes. The training list shows "Target model". Old reports without `models` render `—`.
4. The agent versions tab shows "Trained on `<target>`" with a link to the training session for
   versions created by training, and nothing for other versions.
5. The eval and training detail pages show the mismatch notice exactly when the canonical
   target model differs from the canonical runtime model (anthropic and bifrost adapters both
   covered). There is no notice when the target is unknown.
6. With no `aikdm.env` set, `helm template` output is unchanged. With values set, both CronJob
   containers get those env entries. docker-compose passes the six variables through.
7. `go test ./...`, `go vet`, `uv run pytest`, and `helm template` / lint for the chart all pass.

## Risks & assumptions

- **Assumption: the configured model string identifies the model well enough.** aikdm reports
  what was configured, such as `claude-haiku-4-5`, not the exact provider snapshot it resolved
  to. That is the same granularity open-bbcd uses for its runtime model, so comparisons are
  like for like.
- **Risk: normalization gaps.** Exotic bare names (e.g. `mistral-large`) aren't prefixed, so a
  bare `mistral-large` vs `mistral/mistral-large` would show a spurious mismatch notice. The
  notice is informational only, and operators can configure both sides in `provider/model` form
  to avoid it.
- **Risk: a deployed aikdm newer than open-bbcd, or the reverse.** Both directions are safe:
  - an old open-bbcd ignores `models`;
  - a new open-bbcd renders `—` for reports from an old aikdm.
- **Assumption:** the BO pages are the only consumers of `training_report`. The repo has no other
  reader besides the summary extraction.
- **Follow-up after merge:** run `/arch-review eval-training-model-meta` to note the `models`
  block in the `training-loop` L2 and the aikdm runner env in `c4/deployment.md`.
