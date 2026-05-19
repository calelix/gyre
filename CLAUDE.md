# Gyre

## Project orientation

Upstream anchor: [STRATEGY.md](STRATEGY.md). Read it when work demands project-level context (problem framing, persona, definition of done, tracks).

Adjacent context:
- [docs/foundation.md](docs/foundation.md) — concrete End Goal and initial tech stack.
- [docs/obstacles.md](docs/obstacles.md) — three structural obstacles the AI Harness must navigate.
- [docs/slices.md](docs/slices.md), [docs/select.md](docs/select.md) — current slicing-methodology state.
- [docs/issues/](docs/issues/) — failure records; see the [README](docs/issues/README.md) inside for the entry convention.

Generated artifacts (component specs from skills) live at `docs/gyre/specs/components/<kebab-name>.md`.

## Commits

**Do not create git commits without explicit user direction.** Plans, hooks, or verification steps that suggest committing — ignore. Inspecting git state (`git status`, `git diff`, `git log`) is fine; producing new commits is not.

## Issue records

When a structural failure surfaces, fix the root cause — not just the surface symptom — and record it under [docs/issues/](docs/issues/) following the convention in [docs/issues/README.md](docs/issues/README.md). An entry should leave the next contributor better equipped to avoid the same kind of failure, not just report what happened.
