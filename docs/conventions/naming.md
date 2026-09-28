# Naming Conventions

Names should tell a reader what a thing is without opening it; consistency lets tooling, search, and
new contributors move across every repo predictably.

## Repos & services
- Repo name = service name, `kebab-case`, prefixed by domain when there's a collision risk:
  `billing-api`, `billing-worker`.
- One repo = one deployable service (or one clearly scoped library). No monorepo-of-everything unless
  explicitly decided and recorded in `architecture.md`.

## Files & directories
- `kebab-case` for filenames and directories; deviate only for a strong ecosystem norm (e.g.
  `PascalCase.tsx` for React components).
- Directory structure mirrors layering (`domain/`, `application/`, `infrastructure/`), not a flat
  pile of feature folders at the root.

## Types & code symbols
- Types/classes: `PascalCase` nouns. Functions/methods: `camelCase` or `snake_case` per language
  convention, verbs. Constants: `UPPER_SNAKE_CASE`.
- Suffix by role, not by technology: `UserRepository`, `UserService`, `CreateUserCommand` — not
  `UserDAO`, `UserImpl`.

## API endpoints
- Plural nouns for resources: `/users`, `/users/{id}/orders`. No verbs in the path — verbs are HTTP
  methods.
- `kebab-case` for multi-word path segments: `/password-resets`.

## Events & messages
- `<context>.<entity>.<pastTenseAction>`, e.g. `billing.invoice.paid`, `onboarding.user.registered`.
  Past tense signals "this happened," not a command to act.
- The schema version lives in the payload envelope, not the event name itself (see `events.md`).

## Git branches
- `<type>/<short-description>`, type ∈ `feat`, `fix`, `chore`, `docs`, `refactor` — e.g.
  `feat/invoice-export`.
- Reference the tracker issue where one exists: `feat/PROJ-123-invoice-export`.
