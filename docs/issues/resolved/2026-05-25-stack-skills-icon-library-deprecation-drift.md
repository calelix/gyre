---
date: 2026-05-25
slug: stack-skills-icon-library-deprecation-drift
title: stack skills do not track icon-library deprecations such as Phosphor's *Icon suffix
status: resolved
---

# stack skills do not track icon-library deprecations such as Phosphor's *Icon suffix

## Summary
The generated `mode-toggle.tsx` imports
`import { Moon, Sun } from "@phosphor-icons/react"`. As of the current
`@phosphor-icons/react`, the unsuffixed names (`Sun`, `Moon`, `User`,
etc.) are deprecated; the recommended replacements are `SunIcon`,
`MoonIcon`, `UserIcon`. No stack skill in the current manifest carries
this knowledge, so every generation that needs Phosphor icons will
silently produce deprecated imports.

## Context
- Playground: a host project whose `components.json` declares
  `"iconLibrary": "phosphor"`, with `@phosphor-icons/react@^2.1.10`
  installed.
- Tech stack: the eight stack skills in
  [`skills/setup/references/stack-skills.md`](../../skills/setup/references/stack-skills.md).
  None of them owns icon-library guidance.
- Detected by: post-generation comparison and an external lookup that
  confirmed the deprecation (`Sun`/`Moon` deprecated in favor of
  `SunIcon`/`MoonIcon`; recent `@phosphor-icons/react` release notes;
  RSC contexts should additionally prefer
  `@phosphor-icons/react/ssr`).

## Problem
- **Expected:** when a generated component needs icons from the host's
  configured icon library, the import form follows the library's
  current recommendation, and known deprecations (`Sun` → `SunIcon`)
  are avoided.
- **Actual:** the skill emitted
  `import { Moon, Sun } from "@phosphor-icons/react"`. Both names are
  deprecated. The host accepts them (they still resolve), so typecheck
  and Storybook build pass — the deprecation is silent. The same gap
  exists for App Router RSC contexts that should prefer
  `@phosphor-icons/react/ssr`.

## Root cause
The stack-skills manifest classifies skills by their general subject
area (React idioms, Next.js conventions, composition, design
guidelines, component construction) but has no entry whose
responsibility is "the host-configured icon library's current import
conventions." `components.json`'s `iconLibrary` field is read by host
discovery to pick the library, but the *form* of imports from that
library (current names, current submodule paths, SSR vs CSR entries)
is not owned by any skill in the manifest. Generation defaults to the
AI's general knowledge of the library, which lags behind library
releases over time.

This is the general form of an **external-package drift** problem.
The icon library is the most visible instance; the same shape will
appear for any external package whose import conventions evolve
(router APIs, form libraries, animation libraries, etc.).

## Remediation
Resolved by the External Package Knowledge design (see Related).
The design rejected the package-adapter direction in favor of
**type-driven derivation**: the existing External dependencies slot in
[`skills/generate-component/references/host-discovery.md`](../../skills/generate-component/references/host-discovery.md)
gains a second responsibility — resolving the current export-name for
each abstract symbol the spec requires, by reading the package's
installed type definitions and respecting `@deprecated` JSDoc tags.
No new manifest entry, no package adapter skill.

The same design step covers
[`archived/2026-05-25-generate-component-host-preconditions-from-imports.md`](archived/2026-05-25-generate-component-host-preconditions-from-imports.md),
which the two issues' Related sections cross-link.

## Verification
Manual end-to-end re-run owned by the design's Scenario A: re-run
`/gyre:generate-component` against the existing `mode-toggle.md` spec
in a host with `@phosphor-icons/react` installed. The Plan's import
rows should show:

- `Import: icon "moon" from @phosphor-icons/react` → `MoonIcon (host types; "Moon" available but @deprecated)`, source `host`.
- `Import: icon "sun" from @phosphor-icons/react` → `SunIcon (host types; "Sun" available but @deprecated)`, source `host`.

The emitted component file imports the suffixed names:
`import { MoonIcon, SunIcon } from "@phosphor-icons/react"`.

## Follow-ups
- The applied design relies on the AI's general knowledge of icon
  libraries' naming conventions (Step 1, "Candidate generation"). If
  a future package proves derivation-resistant in practice, log a
  new issue so the design can be revisited — a per-package adapter
  remains a viable targeted remediation even though it was rejected
  as the default mechanism.

## Related
- Manifest (intentionally unchanged):
  [`skills/setup/references/stack-skills.md`](../../skills/setup/references/stack-skills.md).
- Discovery and import policy:
  [`skills/generate-component/references/host-discovery.md`](../../skills/generate-component/references/host-discovery.md),
  [`skills/generate-component/references/output-format.md`](../../skills/generate-component/references/output-format.md)
  ("Imports" section).
- Phosphor: https://github.com/phosphor-icons/react (releases —
  `*Icon` suffix rename, `/ssr` submodule).
- Sibling issue (same structural pattern, different package; archived):
  [`archived/2026-05-25-generate-component-host-preconditions-from-imports.md`](archived/2026-05-25-generate-component-host-preconditions-from-imports.md).
