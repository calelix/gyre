---
name: setup
description: Use when the user wants to prepare a host project for the AI Harness's component generation by installing the external stack skills it depends on. Reads the stack-skills manifest, detects which skills are already installed in the host project, installs the missing ones via the skills CLI, and reports the result. Invoked as /setup with no arguments.
allowed-tools: Bash(npx skills *), Bash(pnpm dlx skills *), Bash(bunx --bun skills *), Read, Glob
---

# setup

A skill that installs into the current host project the external stack skills
that the AI Harness's component-generation pipeline depends on. It reads a
manifest of required stack skills, installs any that are missing, and reports
what it did.

## Invocation

```
/setup
```

The skill takes no arguments. It operates on the current working directory,
which is treated as the host project root.

## Core policy — stack skills are installed only as skills

External stack skills are installed individually, as skills, into the host
project's skills directory via the `skills` CLI — the current CLI places the
Claude Code agent's skills under `.claude/skills/`. They are never installed as
plugins. Installing a whole plugin to obtain one stack skill would
pull unrelated skills, agents, and commands into the host project; skill-level
installation matches the Harness's per-skill dependency exactly. This is a
standing constraint, not a deferred feature.

## Language

The skill body and the manifest are written in English. At runtime the AI
conducts any dialogue and delivers the final report in the user's language,
inferred from the user's messages.

## The manifest

The external stack skills to install are declared in
`references/stack-skills.md`. Each entry has four fields — `name`, `source`,
`skill`, `purpose` — documented in that file. The AI reads the manifest at
invocation time; its contents are not duplicated here.

## Process

The skill executes the following six steps in order.

### 1. Confirm host project

Check that the current working directory contains a `package.json` file. If it
does not, tell the user that `setup` must be run from a project root, and
terminate without installing anything.

### 2. Load the manifest

Read `references/stack-skills.md` and parse every row of its Skills table into
the four fields. This is the list of external stack skills to ensure are
installed in the host project.

### 3. Determine the package runner

Inspect the host project's `package.json`:

- If it has a `packageManager` field, derive the runner from the named manager:
  `pnpm` → `pnpm dlx`, `bun` → `bunx --bun`, `npm` or `yarn` → `npx`.
- If there is no `packageManager` field, infer from the lockfile present at the
  project root: `pnpm-lock.yaml` → `pnpm dlx`; `bun.lockb` or `bun.lock` →
  `bunx --bun`; otherwise → `npx`.

The resulting runner string is used to invoke the `skills` CLI in step 5.

### 4. Detect install state

For each manifest entry, evaluate two signals against the host project:

- **Skill files** — a skill directory for the entry exists, carrying a
  `SKILL.md`, in EITHER of the two locations the `skills` CLI may use: at
  `.claude/skills/<name>/SKILL.md` OR at `.agents/skills/<name>/SKILL.md`.
  Both are checked because a host project may have been set up with different
  `skills` CLI versions or agent selectors — the current CLI installs the
  Claude Code agent's skills under `.claude/skills/`, but `.agents/skills/` is
  also a valid installed location.
- **Lock entry** — `skills-lock.json` exists at the project root and contains
  the key `<name>` under its `skills` object.

Classify the entry:

- **installed** — both signals hold.
- **incomplete** — exactly one signal holds.
- **missing** — neither signal holds.

`incomplete` and `missing` entries are both installation targets for step 5.
`installed` entries are skipped.

### 5. Install the missing skills

For each installation target, run this command from the host project root:

```
<runner> skills add <source> --agent claude-code --yes
```

- `<runner>` is the runner string from step 3.
- `<source>` is the entry's `source` field.
- If the entry's `skill` field is not `—`, append `--skill <skill>` to the
  command.
- `--agent claude-code` installs the skill for Claude Code, the agent the
  Harness runs under.
- `--yes` skips the CLI's interactive confirmation prompts so the command runs
  unattended.

Process entries independently: if one command fails, capture its error output,
record the entry as failed, and continue with the remaining entries. Do not
abort the run on a single failure.

### 6. Verify and report

For each entry that was an installation target, re-check the two detection
signals from step 4. The entry is **verified** when both now hold.

Report to the user, in their language:

- Skills that were already installed and therefore skipped.
- Skills that were newly installed and verified.
- Skills whose installation ran but did not verify, or whose command failed —
  with the `skills` CLI error output included.

If any entry failed or did not verify, the report says so plainly. A failed run
is never described as a success.

## Scope

This skill does NOT:

- Install any external stack skill as a plugin (see *Core policy*).
- Install the `skills` CLI itself — it is invoked through the host project's
  package runner, never installed.
- Install or configure the AI Harness itself.
- Scaffold the host stack — `components.json`, dependencies, and build
  configuration are out of scope.
- Invoke or generate UI components — that is a separate skill's responsibility.

## Reference files

- `references/stack-skills.md` — the manifest of external stack skills the
  component-generation pipeline depends on. Read at invocation time.
