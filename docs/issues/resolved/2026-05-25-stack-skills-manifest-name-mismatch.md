---
date: 2026-05-25
slug: stack-skills-manifest-name-mismatch
title: stack-skills manifest names diverged from upstream for two vercel-labs skills
status: resolved
---

# stack-skills manifest names diverged from upstream for two vercel-labs skills

## Summary
The stack-skills manifest declared two entries with bare names
(`react-best-practices`, `composition-patterns`) but the upstream
`vercel-labs/agent-skills` publishes them with a `vercel-` prefix
(`vercel-react-best-practices`, `vercel-composition-patterns`). `setup` passes
the manifest's `skill` value through to `--skill` on the CLI, so both entries
failed to install with "No matching skills found", while a sibling entry
(`web-design-guidelines`) — which the upstream publishes without a prefix —
installed fine.

## Context
- Playground: host project running `/gyre:setup` against a clean tree
- Tech stack: `skills` CLI installing into Claude Code (`.claude/skills/<name>/`)
- Detected by: user feedback after `/gyre:setup` reported six skipped (already
  installed) and two failed, the failures attributed to upstream-name lookup
  errors

## Problem
- **Expected:** every manifest entry resolves on the upstream and either
  installs cleanly or is reported as already installed.
- **Actual:** the CLI returned `No matching skills found for:
  react-best-practices` and the same for `composition-patterns`, listing
  `vercel-react-best-practices` / `vercel-composition-patterns` among the
  available names. The host tree and `skills-lock.json` were left unchanged
  for the two failed entries.

## Root cause
The manifest's `name` / `skill` columns for those two entries were written
without the upstream's `vercel-` prefix. The `skill` value flows straight into
the CLI's `--skill` selector (`skills/setup/SKILL.md` Step 5), so a name that
the upstream does not publish produces a hard install failure. The mismatch
also breaks the precondition check in `generate-component` (Step 1.5), which
keys detection off the manifest's `name` field — so even if the user
side-installs the correctly named skill, the precondition would not recognize
it.

The structural fault is that the manifest was authored against the assumption
that the upstream `vercel-labs/agent-skills` publishes its skills under bare
names. That holds for some entries in that repo (`web-design-guidelines`) but
not others — the repo namespaces React/composition skills with a `vercel-`
prefix and the manifest did not reflect the per-skill reality.

## Remediation
- [`skills/setup/references/stack-skills.md`](../../skills/setup/references/stack-skills.md)
  — the two affected rows' `name` and `skill` columns are updated to
  `vercel-react-best-practices` and `vercel-composition-patterns`. No other
  rows change. `generate-component`'s precondition (Step 1.5) reads `name`
  from the manifest dynamically, so its detection keys move with the change
  without a separate edit.

## Verification
The corrected manifest values match the upstream names the CLI listed in its
failure output (`vercel-react-best-practices`, `vercel-composition-patterns`),
which is the authoritative signal for what `--skill` will accept against
`vercel-labs/agent-skills`. A re-run of `/gyre:setup` on the reporting host
will install the two previously failed entries; that re-run is queued for the
user and not executed from this session.

## Follow-ups
None blocking. A broader hardening — having `setup` cross-check the
manifest's `skill` selectors against the upstream's published names before
attempting an install, so a future drift surfaces as a manifest error rather
than an install failure — is a candidate for a separate issue if the same
class of drift recurs.

## Related
- Manifest: [`skills/setup/references/stack-skills.md`](../../skills/setup/references/stack-skills.md)
- Setup skill: [`skills/setup/SKILL.md`](../../skills/setup/SKILL.md) (Step 5 — install command construction)
- Generate skill: [`skills/generate-component/SKILL.md`](../../skills/generate-component/SKILL.md) (Step 1.5 — precondition)
