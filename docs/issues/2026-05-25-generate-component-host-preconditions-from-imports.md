---
date: 2026-05-25
slug: generate-component-host-preconditions-from-imports
title: generate-component does not verify host preconditions implied by emitted imports
status: resolved
---

# generate-component does not verify host preconditions implied by emitted imports

## Summary
When a generated component imports a symbol whose correct runtime
behavior requires the host application to be set up a specific way
(e.g., `useTheme` from `next-themes` requires a `<ThemeProvider>` in
the tree and `<html suppressHydrationWarning>` on the root element),
`generate-component` does not check the host for those preconditions
and does not flag missing ones to the user. The component typechecks
and the Storybook story works (because the story inlines its own
decorator), so the verification loop passes. Yet the same component,
dropped into the real app, may render incorrectly or fail to toggle
at runtime — and the user gets no warning.

## Context
- Playground: a host project running `/gyre:generate-component`
  against `mode-toggle.md`, which compose-imports `Button` from
  `@workspace/ui/components/button` and `useTheme` from
  `next-themes`.
- Tech stack: Next.js App Router + next-themes; the host does have a
  `theme-provider.tsx` file in `apps/web/components/`, but the skill
  never verified that it is actually wired into `app/layout.tsx` or
  that `<html>` carries `suppressHydrationWarning`.
- Detected by: post-generation comparison against shadcn/ui's
  "Dark Mode for Next.js" setup guide at
  https://ui.shadcn.com/docs/dark-mode/next, which lists
  `<html suppressHydrationWarning>` and a `<ThemeProvider attribute=
  "class" defaultTheme="system" enableSystem disableTransitionOnChange>`
  in `app/layout.tsx` as required preconditions.

## Problem
- **Expected:** when the generated component will import a symbol
  with documented host setup requirements (a context provider must
  exist in the tree, an HTML attribute must be set, a CSS variable
  must be loaded, a Next.js config flag must be on, etc.),
  `generate-component`'s host discovery checks the host for those
  requirements and surfaces unmet ones as Plan items (`attention`)
  so the user knows what to fix before the component will work.
- **Actual:** discovery only checks what the spec or the Plan
  explicitly names (component output directory, Reference primitive
  presence, external dependencies in `package.json`, Storybook
  installation). Implicit setup requirements that come along with an
  imported symbol — like next-themes' ThemeProvider and
  `suppressHydrationWarning` — are never verified. The Plan
  therefore reports "all green" while a runtime requirement quietly
  goes unmet.

## Root cause
The current slot policy in
[`references/host-discovery.md`](../../skills/generate-component/references/host-discovery.md)
covers slots whose *value* is needed to write the file (output
directory, primitive location, etc.) and dependency checks for
packages directly listed in `package.json`. It has no slot for
**consumer-implied preconditions** — runtime requirements that
accompany the *use* of an imported symbol but are not visible from
`package.json` alone.

These preconditions are properties of the external package's usage
contract (next-themes requires X in the host), not properties of the
spec or the host's project layout. They therefore belong to a
stack-skill body, not to the spec format. But the stack skills
currently loaded at Step 2.5 (`vercel-react-best-practices`,
`next-best-practices`, etc.) do not carry per-package consumer
guides at the granularity discovery would need to look them up
automatically.

The Storybook story file masks the gap: the story decorator inlines
a `<ThemeProvider>` for visualization, so verification (typecheck +
storybook build) passes even when the real host is misconfigured.
Verification is testing the artifact in isolation, not the artifact
in its target host.

## Remediation
Resolved by the External Package Knowledge design (see Related). The
applied remediation:

- A **Consumer-implied preconditions slot** is added to discovery in
  [`skills/generate-component/references/host-discovery.md`](../../skills/generate-component/references/host-discovery.md).
  For each external symbol the component will import, the AI derives
  host-side preconditions from the loaded stack skill bodies plus
  general knowledge, specifies a verification signal per precondition,
  and verifies each signal against host files. Unmet preconditions
  produce Plan rows with the new label `host-setup-required`
  (non-blocking).
- The import → precondition mapping does **not** live in a static
  catalog. The design rejected the per-package adapter direction
  (parallel to the resolution of
  [`2026-05-25-stack-skills-icon-library-deprecation-drift.md`](2026-05-25-stack-skills-icon-library-deprecation-drift.md))
  and instead derives the mapping at generation time from spec body
  + stack skill bodies + general knowledge. Conservative disclosure
  surfaces uncertain cases as `host-setup-required` rows with an
  explicit confidence note, biasing toward false positives.
- The spec format gains an optional `## Host preconditions` section
  in
  [`skills/clarify-component/references/output-format.md`](../../skills/clarify-component/references/output-format.md)
  as an escape hatch for preconditions a spec author knows about but
  the derivation channel might miss. `clarify-component` does not
  ask for this section in this iteration. The default path remains
  derivation; the section is consumed alongside derived preconditions
  by the same slot.

The label inventory in
[`skills/generate-component/references/plan-format.md`](../../skills/generate-component/references/plan-format.md)
grows by one entry, `host-setup-required`, distinguished from
`attention` by being a runtime (not cosmetic) signal.

## Verification
Manual end-to-end re-run owned by the design's Scenario A: re-run
`/gyre:generate-component` against the existing `mode-toggle.md` spec
in a host where `<ThemeProvider>` is not wired into `app/layout.tsx`
and `<html>` does not carry `suppressHydrationWarning`. The Plan
should surface two `host-setup-required` rows:

- `Precondition: <ThemeProvider> in app/layout.tsx` — signal: the
  file `app/layout.tsx` contains the substring `<ThemeProvider`.
  Verification: signal not found.
- `Precondition: suppressHydrationWarning on <html> in
  app/layout.tsx` — signal: the file `app/layout.tsx` contains
  `<html` with `suppressHydrationWarning` on the same element.
  Verification: signal not found.

(Note that the original prediction in this issue named the label as
`attention`. The applied design distinguished `attention` from the
new `host-setup-required` label; see the Remediation section.)

## Follow-ups
- AI-derived precondition accuracy may need calibration over time;
  log offending runs as new issues so calibration draws from real
  cases.

## Related
- Design that closed this issue:
  [`docs/superpowers/specs/2026-05-25-external-package-knowledge-design.md`](../superpowers/specs/2026-05-25-external-package-knowledge-design.md).
- Discovery slot policy (extended):
  [`skills/generate-component/references/host-discovery.md`](../../skills/generate-component/references/host-discovery.md).
- Plan label inventory (extended with `host-setup-required`):
  [`skills/generate-component/references/plan-format.md`](../../skills/generate-component/references/plan-format.md).
- Spec format optional section:
  [`skills/clarify-component/references/output-format.md`](../../skills/clarify-component/references/output-format.md).
- Comparison reference: https://ui.shadcn.com/docs/dark-mode/next
  ("Add the `ThemeProvider` to your root layout and add the
  `suppressHydrationWarning` prop to the `html` tag.").
- Sibling issue (same structural pattern, different package):
  [`2026-05-25-stack-skills-icon-library-deprecation-drift.md`](2026-05-25-stack-skills-icon-library-deprecation-drift.md).
