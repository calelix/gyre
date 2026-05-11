# Git Commit

Analyze all staged and unstaged changes, group them into logical commits, and create them sequentially.

---

## Commit Message Format

```
<type>: <what was done>

<why - background, reason, purpose>
```

### Title Rules

- **No scope** — `type: Subject` only. Never `type(scope): ...`
- Under 50 characters
- Imperative mood ("Add" not "Added")

### Types

`feat` | `fix` | `refactor` | `test` | `style` | `docs` | `build` | `ci` | `revert` | `chore`

---

## Process

### Step 1: Gather Information

Run in parallel:

```bash
git status
git diff --stat
git diff             # unstaged changes (working dir vs index)
git diff --cached    # staged changes (index vs HEAD)
```

### Step 2: Logical Grouping (Mandatory)

**Every commit request requires analysis and grouping.** Never collapse unrelated work into one commit — even if the user was terse, listed paths in one message, or said "just commit".

Group by these criteria (each warrants a separate commit):

1. **Different purpose** — bug fix vs new feature
2. **Different type** — `refactor` vs `feat`
3. **Independent feature/area** — auth logic vs UI component
4. **Different codebase area** — frontend vs backend
5. **Temporal independence** — refactor then feature add → commit refactor first

One commit is correct **only** when analysis yields a single logical unit.

### Step 3: Order by Dependency

1. External dependencies (package.json)
2. Utilities / shared functions
3. Type / interface definitions
4. Business logic
5. UI components
6. Pages / routes
7. Tests
8. Docs / style

### Step 4: Create Commits

Stage specific files per group — **never** `git add .` or `git add -A`.

```bash
git add auth/jwt.ts auth/middleware.ts
git commit -m "$(cat <<'EOF'
feat: Add JWT-based authentication system

Replaces session-based auth to reduce server load and support
stateless authentication.
EOF
)"
```

### Step 5: Line-Level Split (when needed)

When a single file has changes for different purposes, prefer `git stash` (safer, no risk of losing uncommitted work):

```bash
git add -p file              # interactively stage first logical chunk
git stash --keep-index       # stash everything not staged
git commit -m "..."          # commit first chunk
git stash pop                # restore remainder
git add file
git commit -m "..."          # commit remainder
```

Alternative (cp method) — use only when stash isn't suitable (e.g., conflicts on pop):

1. Backup: `cp file file.full`
2. Reset: `git checkout HEAD -- file`
3. Apply first logical change only, stage and commit
4. Restore: `cp file.full file`
5. Stage and commit remainder
6. Cleanup: `rm file.full`

Use line-level split only when necessary. Prefer file-level split.

### Step 6: Verify

```bash
git log --oneline -n 10
```

---

## Edge Cases

- **On main branch** → Warn user, suggest creating a new branch first
- **No changes** → Inform "nothing to commit"
- **Merge conflict** → Inform user, guide resolution
- **Pre-existing staged files** (files already in index from prior `git add`) → Inform user of the existing staged state, ask whether to use as-is or reset, and **exit** until confirmed
- **Commit creation failed** (pre-commit hook, signing error, etc.) → Inform user with the failure output and **exit**. Do not retry, do not `--amend`, do not skip hooks. User must diagnose and re-run
- **Sensitive files detected** (`.env*`, `*.key`, `*.pem`, `*.p12`, `credentials.*`, `*.secret`, `id_rsa*`, `.npmrc` with token, `*.crt`) → Warn user, exclude from staging unless explicitly approved
- **User wants review** → Show planned groups before executing (see Confirmation Mode below)

---

## Confirmation Mode

Triggered when the user requests review before committing (e.g. "show me first", "review", "preview").

Present the plan in this format and **wait for user confirmation** before executing any `git commit`:

```
Planned commits (3):

[1/3] refactor: Extract auth utilities
      Files: auth/utils.ts, auth/types.ts

[2/3] feat: Add JWT authentication system
      Files: auth/jwt.ts, auth/middleware.ts
      Body: Replaces session-based auth to reduce server load.

[3/3] test: Add auth integration tests
      Files: tests/auth.test.ts

Proceed? (yes / edit / cancel)
```

User responses:
- **yes / ok** → Execute commits in order
- **edit** → Ask which commit to modify (title, body, files, order)
- **cancel** → Abort, leave working tree untouched

---

## Quality Checklist

- [ ] Grouped per logical unit (no shortcut to one commit)
- [ ] Title under 50 chars, imperative mood
- [ ] No scope in title — `type: Subject` only
- [ ] Body explains "why" for significant changes
- [ ] Each commit is independently buildable
- [ ] Single logical change per commit
