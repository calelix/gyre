---
date: 2026-05-25
slug: generate-component-stack-skills-uninvoked
title: generate-component never loads or invokes the stack skills setup installs
status: open
---

# generate-component never loads or invokes the stack skills setup installs

## Summary
After `/gyre:setup` successfully installed the eight stack skills declared in
the manifest, the following `/gyre:clarify-component` → `/gyre:generate-component`
run did not load any stack skill's `SKILL.md` body and did not invoke any
stack skill via Claude Code's Skill tool. The skills were detected by the
Step 1.5 precondition (which only verifies installation signals — directory +
lock entry) but their content never entered the runtime context, so the
"stack conventions are delegated to the stack skills" wording across the
generate-component reference files did not result in any actual consultation.
The AI extracted conventions by reading host files directly instead.

## Context
- Playground: a host project (separate from the gyre repo) where the user ran
  `/gyre:setup`, then `/gyre:clarify-component`, then `/gyre:generate-component`
  against a single component spec.
- Tech stack: the eight skills in the current manifest at
  [`skills/setup/references/stack-skills.md`](../../skills/setup/references/stack-skills.md)
  — `shadcn`, `vercel-react-best-practices`, `vercel-composition-patterns`,
  `next-best-practices`, `next-cache-components`, `web-design-guidelines`,
  `building-components`, `agent-browser`.
- Detected by: user observation that none of the eight installed skills had
  their body loaded or were called via the Skill tool across the
  `generate-component` execution.

## Problem
- **Expected:** during generation, the stack skills that own React idioms,
  composition patterns, web design guidelines, Next.js conventions, and
  component-construction guidance are consulted as the source of truth for
  the corresponding decisions — at minimum loaded via the Skill tool at the
  points where their knowledge applies (host discovery, code-form decisions,
  verification).
- **Actual:** for the reported run, only the install-state signals were
  checked (Step 1.5). None of the eight stack skills' `SKILL.md` bodies were
  loaded, and the Skill tool was not invoked for any of them. `shadcn` did
  not fire because the run produced no `shadcn-install` Plan item (the
  referenced primitive already existed in the host). The remaining seven have
  no path in the current skill design that would have caused them to be
  invoked. Conventions ended up extracted from host files alone.

## Root cause
The structural fault is in `generate-component`'s process: the skill body and
the four reference files repeatedly *delegate* to "the stack skills installed
by `setup`," but they do not specify a *call mechanism* by which that
delegation actually fires.

Concretely:
- [`SKILL.md` Step 1.5](../../skills/generate-component/SKILL.md) only
  **verifies installation** (directory + lock entry). It does not load any
  stack skill's body.
- [`SKILL.md` Step 4](../../skills/generate-component/SKILL.md) says
  "stack conventions … are delegated to the stack skills installed by
  `setup`" but does not direct the runtime to invoke any of them — discovery
  is left to direct host-file reads.
- [`SKILL.md` Step 7](../../skills/generate-component/SKILL.md) is the only
  step that names "invoke … through Claude Code's Skill tool," and the
  invocation it specifies is **conditional and single-target**: it fires only
  for `shadcn-install` (`npm-install` runs through Bash, not the Skill tool).
  When the Plan contains no such items — as in the reported run — no Skill
  invocation occurs.
- [`references/host-discovery.md`](../../skills/generate-component/references/host-discovery.md),
  [`references/output-format.md`](../../skills/generate-component/references/output-format.md),
  and [`references/verification.md`](../../skills/generate-component/references/verification.md)
  each repeat the delegation language ("the runtime AI consults those stack
  skills") without specifying when, by what trigger, or for which decisions
  that consultation should occur.

There is therefore no step in the skill that unconditionally brings the stack
skills' knowledge into the runtime context for the decisions that depend on
it. The seven non-`shadcn` skills (`vercel-react-best-practices`,
`vercel-composition-patterns`, `next-best-practices`, `next-cache-components`,
`web-design-guidelines`, `building-components`, `agent-browser`) have no
defined invocation step at all in the current process.

For the simple spec reported here, host-file observation alone was sufficient
to produce a passing output, which masks the gap. For more demanding specs
the same gap risks decisions that diverge from the stack guidance the
manifest claims to enforce.

## Remediation
Open. The fix requires a design decision before code changes:

- choose which `generate-component` steps need stack-skill knowledge
  unconditionally (likely Step 4 host discovery, Step 8 file writing,
  Step 9 verification), and
- specify the call mechanism per step — which stack skill to invoke via the
  Skill tool, for what purpose, at what point — so the delegation in the
  reference files is no longer wording-only.

Until that design lands, no code changes are made and the issue remains
`open`.

## Verification
N/A — recorded as `open`. When a remediation lands, verification should
re-run `/gyre:setup` → `/gyre:clarify-component` → `/gyre:generate-component`
against a non-trivial spec on a host project and confirm that the affected
stack skills are actually invoked at the specified steps (visible in the
run's tool-call trace).

## Follow-ups
- During the design step, consider whether `agent-browser` belongs in the
  manifest today. `generate-component`'s SKILL explicitly defers browser
  verification to a future iteration; if it has no current consumer, the
  precondition cost of requiring it installed is paid for no current benefit.

## Related
- Generate skill: [`skills/generate-component/SKILL.md`](../../skills/generate-component/SKILL.md)
  (Steps 1.5, 4, 7) and references in
  [`skills/generate-component/references/`](../../skills/generate-component/references/).
- Manifest: [`skills/setup/references/stack-skills.md`](../../skills/setup/references/stack-skills.md).
- Prior manifest issue: [`2026-05-25-stack-skills-manifest-name-mismatch.md`](2026-05-25-stack-skills-manifest-name-mismatch.md).
- Design spec: [`docs/superpowers/specs/2026-05-24-stack-skills-precondition-and-manifest-expansion-design.md`](../superpowers/specs/2026-05-24-stack-skills-precondition-and-manifest-expansion-design.md)
  (introduced the precondition; the call-mechanism gap was not addressed there).
