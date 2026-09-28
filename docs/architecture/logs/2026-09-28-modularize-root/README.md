# modularize-root — decompose root into 3 L1 children (flow-map-compiler, open-bbcd, aikdm)

**Date**: 2026-09-28
**Codename**: modularize-root

**Driver**: /modularize run on root

**Decision**: decomposed root into 3 L1 children: `flow-map-compiler`, `open-bbcd`,
`aikdm` — the three shipped software artifacts of OpenBBC. Each L1 is a service unit
with its own code, its own release cadence, and its own maintainers-of-record.

**Rationale**: The modularity tree captures **what we build**, not deployment forms or
shared infrastructure. OpenBBC ships exactly three primary software artifacts: a
Claude Code plugin skill (`flow-map-compiler`), a Go daemon (`open-bbcd`), and a Python
CLI (`aikdm`). Everything else in the arch surface is either (a) a runtime packaging of
one of those three (the `aikdm-runner` image bundles `aikdm` + `scripts/` for k8s), (b)
deployment topology the deployer configures (Helm chart), or (c) infrastructure the
deployer provides (Postgres — a shared datastore, not a service unit we own). Keeping
those out of L1 keeps the tree focused on the code/product surface rather than the
operational surface — the latter is already captured in `c4/deployment.md`,
`c4/containers.md`, and the aikdm-runner subsection there.

**Alternatives rejected**:
- Include `postgres` as an L1 — rejected: it is shared infrastructure with no code we
  own and no independent release cycle. Its role as the store for every stateful thing
  is already documented in `c4/containers.md § postgres` and its placement in
  `c4/deployment.md § Network zones (Data zone)`.
- Include `aikdm-runner` as an L1 — rejected: it is a container image that packages
  `aikdm` + `scripts/` + shell tooling for the Kubernetes CronJob runtime. Its shipping
  surface, code, and lifecycle are aikdm's. The image lives in `c4/containers.md
  § aikdm-runner` as a runtime form of the aikdm L1; noted in the aikdm L1 Purpose.
- Include `scripts/` (`process_pending_alphas.sh`, `generate_alpha.sh`, `seed_bundle.py`,
  `run_eval.sh`, `train_from_session.sh`) as an L1 — rejected: helper wrappers absorbed
  into aikdm's release cycle; shipped inside the aikdm-runner image.
- Include the Helm chart as an L1 — rejected: deployment topology, not a service. Owned
  by `c4/deployment.md`.

**Impact**:
- + docs/architecture/current/modularity/flow-map-compiler/README.md
- + docs/architecture/current/modularity/open-bbcd/README.md
- + docs/architecture/current/modularity/aikdm/README.md

**Links**:
- (user may add related spec PRs, feature docs, or tracker items before commit)
