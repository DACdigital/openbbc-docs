---
date: 2026-10-07
codename: sync-bifrost
kind: sync
---

# Sync — align current/ with 2026-10-07-bifrost-design

## Driver
Post-approval sync of `docs/architecture/current/` with the following spec(s):
- `docs/superpowers/specs/2026-10-07-bifrost-design.md`

## Decision
Apply the deltas listed below to `docs/architecture/current/`. AMBIGUOUS deltas (if any) are
recorded here for human resolution but were NOT applied.

## Rationale
Approved spec is the design contract; `current/` is the architecture-of-record and must reflect it.
Per project rule: L2 modularity nodes are inspirational — specs are the impl contract; sync applies
CONTRACT_SHAPE deltas without altering module Purpose, and only introduces new L1/L2 surface for
SCOPE_MOVE deltas.

## Alternatives rejected
- Leaving current/ stale until the next `/modularize` run — rejected: drift accumulates silently and
  a new dev reading current/ builds against the wrong contract.
- Auto-applying all deltas including AMBIGUOUS ones — rejected: silent overrides of prior arch-log
  decisions require human review.

## Impact

### CONTRACT_SHAPE deltas applied (no purpose change)
- `constraints.md:Hard technical limits` — Go bullet "Go 1.22+" → "Go 1.27+".
- `constraints.md:Hard technical limits` — `OPENBBC_DEFAULT_MODEL` on `bifrost` is required (no default), split on the first `/`, and the provider must be on the v1 key-only allow-list. Cloud-credential and keyless providers are refused. Only the selected provider's key is read.
- `constraints.md:Hard technical limits` — new `<PROVIDER>_BASE_URL` bullet: allowed only for openai/anthropic/cohere/mistral, and must be `https` (plain `http` only to a loopback host).
- `c4/integrations.md:LLM providers row` — "any Bifrost-supported provider" → v1 allow-list. Adds the `<PROVIDER>_BASE_URL` rule. Auth reads only the selected provider's key.
- `glossary.md:LLM adapter` — "any Bifrost-supported provider" → v1 key-only allow-list.
- `c4/containers.md:open-bbcd Purpose` — `bifrost` routes to the provider named in `OPENBBC_DEFAULT_MODEL`, limited to the v1 allow-list.
- `c4/containers.md:Diagram` — open-bbcd label "Go 1.22 plus" → "Go 1.27 plus".
- `c4/containers.md:open-bbcd Tech stack` — "Go 1.22+" → "Go 1.27+"; `golang:1.26` → `golang:1.27` builder.
- `c4/context.md:External systems` — LLM providers bullet narrowed to the v1 allow-list.
- `c4/deployment.md:Network zones` — external zone narrowed to the v1 allow-listed providers.
- `c4/deployment.md:Threat model` — provider-credential-leak row now covers the `<PROVIDER>_BASE_URL` exfiltration angle. Only the selected key is read; the mitigation is https-only except loopback.
- `modularity/open-bbcd/README.md:Purpose` — "Go 1.22+" → "Go 1.27+"; "any Bifrost-supported provider" → v1 key-only allow-list.
- `assumptions.md:Design decisions (locked)` — Bifrost decision gains the v1 allow-list sub-bullet; new providers are added "by extending the allow-list".

### SCOPE_MOVE deltas applied (new surface / ownership / boundary)
None.

### AMBIGUOUS deltas NOT applied — human review required
None.

## Links
- Source spec(s): `docs/superpowers/specs/2026-10-07-bifrost-design.md`, https://github.com/DACdigital/openbbc-docs/pull/16
- Related prior arch-log entries: `docs/architecture/logs/2026-10-07-bifrost/README.md` — extends (narrows v1 provider scope; adds `<PROVIDER>_BASE_URL` and Go 1.27 toolchain, which that entry's Impact did not list)
