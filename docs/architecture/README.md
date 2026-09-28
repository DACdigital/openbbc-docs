# Architecture

This project's target solution architecture. Two subdirectories, one purpose each.

## Layout

- **`current/`** — the current target solution architecture. `README.md` (topology narrative),
  `nfrs.md`, `assumptions.md`, and the modularity tree under `modularity/<L1>/[<L2>/]README.md`.
  Populated by `/setup-project` during onboarding (adopt an existing vault or scaffold), then
  evolved by `/new-feature` and `/modularize`. Only L1 and L2 nodes are architecture-of-record.
- **`logs/`** — append-only log of architectural decisions. One entry per change, at
  `YYYY-MM-DD-<codename>/README.md`. Every PR that touches `current/` must add exactly one paired
  log entry; `/arch-log-review` enforces this.

See [`docs/process/README.md`](../process/README.md) for how these fit into the C0/C1/C2 flow.
