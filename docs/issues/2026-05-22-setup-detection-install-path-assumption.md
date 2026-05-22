---
date: 2026-05-22
slug: setup-detection-install-path-assumption
title: setup skill detection assumed a single skills-CLI install path
status: resolved
---

# setup skill detection assumed a single skills-CLI install path

## Summary
The `setup` skill's install-state detection assumed the `skills` CLI places a
skill's files at `.agents/skills/<name>/`. The current CLI installs the Claude
Code agent's skills at `.claude/skills/<name>/` instead, so detection would
never recognize a freshly CLI-installed skill.

## Context
- Playground: `experiments/shadcn`
- Tech stack: `skills` CLI (skills.sh / `vercel-labs/skills`), Claude Code plugin skill
- Detected by: self-check during implementation — the install-path verification run against the playground failed its assertion.

## Problem
- **Expected:** after `pnpm dlx skills add shadcn/ui --agent claude-code --yes`,
  the `setup` skill's detect step recognizes `shadcn` as installed.
- **Actual:** detection checked only `.agents/skills/shadcn/SKILL.md`, but the
  CLI installed the files to `.claude/skills/shadcn/SKILL.md`. The "skill files"
  signal never held for a CLI install, so a real install was classified
  `incomplete` and would be reinstalled on every run.

## Root cause
The spec hardcoded one install location — `.agents/skills/<name>/` — into
the detection signal. That location reflects an older `skills` CLI layout
(the committed `experiments/shadcn` fixture still uses it). The current CLI
(`claude-code_2-1-145`) installs the Claude Code agent's skills under
`.claude/skills/<name>/` as a copy — no `.agents/` directory is created and no
symlink bridges the two (verified: a clean `skills add` run produced zero
symlinks anywhere in the host tree).

The structural fault is the single-location assumption itself: a host project
that consumes Gyre as a plugin may legitimately carry either layout, depending
on the `skills` CLI version and agent selector used to install. The spec had
flagged the CLI flag/path as an open design point to be "confirmed against the
live CLI during implementation" — but that single-path assumption was carried
into `SKILL.md` rather than confirmed first.

## Remediation
- [`skills/setup/SKILL.md`](../../skills/setup/SKILL.md) — Step 4 detection now
  treats the "skill files" signal as holding when a `SKILL.md` exists in
  *either* `.claude/skills/<name>/` *or* `.agents/skills/<name>/`. The *Core
  policy* section names `.claude/skills/` as the current CLI's location.
- [`skills/setup/references/stack-skills.md`](../../skills/setup/references/stack-skills.md)
  — the `name` field description now documents the dual-location detection key.
- [`docs/superpowers/specs/2026-05-22-setup-skill-design.md`](../superpowers/specs/2026-05-22-setup-skill-design.md)
  — the open design point is resolved and the `.agents/skills/`-only path
  references are corrected to the dual-location signal.
- No fixture migration: the `experiments/shadcn` fixture is kept at
  `.agents/skills/shadcn/` deliberately, so the idempotency test continues to
  exercise the `.agents/` detection branch.

## Verification
Both verification paths were re-run against `experiments/shadcn` after the fix:

- **Idempotency path** — fixture present at `.agents/skills/shadcn/` plus the
  `skills-lock.json` entry: dual-location detection's `.agents/` branch matched,
  `shadcn` was classified `installed`, nothing was installed, and
  `git status --short -- experiments/shadcn` was empty.
- **Install path** — clean host: `pnpm dlx skills add shadcn/ui --agent
  claude-code --yes` installed to `.claude/skills/shadcn/`; dual-location
  detection's `.claude/` branch matched and `skills-lock.json` carried the
  `shadcn` key, so the entry verified as installed. The committed fixture was
  then restored and `git status` confirmed clean.

## Follow-ups
None. The symlink hypothesis (`.agents/` ↔ `.claude/` linked) was raised and
ruled out by inspection, so no further investigation is outstanding.

## Related
- Spec: [`docs/superpowers/specs/2026-05-22-setup-skill-design.md`](../superpowers/specs/2026-05-22-setup-skill-design.md)
- Skill: [`skills/setup/`](../../skills/setup/)
