# Project rules

## Commits

**Do not create git commits during ongoing work.** The user directs when (and what) to commit at the end of a unit of work.

- Even if implementation plans include explicit commit steps, **skip those steps.** Continue with file edits and structural verification, but do not run `git commit`, `git add` for the purpose of committing, or any operation that produces a new commit.
- It is fine to inspect git state (`git status`, `git diff`, `git log`) and to refer to commit SHAs that already exist. Just do not create new ones.
- Reviewers and verification steps that read git state may report uncommitted or untracked files. That is the expected ongoing-work state — do not commit to make `git status` look tidy.
- When the user explicitly says "commit this," follow the standard commit process. Until they do, treat working-tree changes as the unit of work.
