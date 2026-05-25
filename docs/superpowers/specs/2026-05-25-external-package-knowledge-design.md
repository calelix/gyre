---
created: 2026-05-25
---

# External Package Knowledge Design

## Context

Two open issues recorded against the mode-toggle generation run share a
single structural root: `generate-component`'s pipeline has no place where
**knowledge about an external package** — how to import from it today and
what it requires of the host — can live and be consulted at generation time.

- [`docs/issues/2026-05-25-generate-component-host-preconditions-from-imports.md`](../../issues/2026-05-25-generate-component-host-preconditions-from-imports.md)
  — emit of `useTheme` from `next-themes` proceeded without checking that
  the host's `app/layout.tsx` had `<ThemeProvider>` wired or that `<html>`
  carried `suppressHydrationWarning`. The component typechecks and the
  Storybook story works (the story inlines its own decorator), so the
  verification loop reports success while the artifact silently fails in
  the real host.
- [`docs/issues/2026-05-25-stack-skills-icon-library-deprecation-drift.md`](../../issues/2026-05-25-stack-skills-icon-library-deprecation-drift.md)
  — emit of `Sun`, `Moon` from `@phosphor-icons/react` reproduces the
  library's *deprecated* names. The current names are `SunIcon`, `MoonIcon`,
  with `/ssr` as the RSC submodule. The host accepts the deprecated
  imports, so typecheck passes and the deprecation is silent.

Both issues propose, among their candidates, adding a new manifest
classification (per-package adapter skills, separate reference files) so
that import-time external-package knowledge has a home. This spec rejects
that direction in favor of a different one — described under Principle —
and **leaves the eight-entry manifest unchanged**.

## Principle

**Default to derivation; allow explicit override.**

External-package knowledge is *not* enumerated in a static catalog of
entries that has to be maintained per package. Instead, two derivation
channels run at generation time, and one optional spec section serves as
an escape hatch for the cases derivation cannot reach:

1. **Type-driven derivation** for import *form* (current symbol names,
   submodule paths). Answered by reading the host's installed package
   type definitions, which are guaranteed to exist post-`npm-install`.
2. **AI-knowledge derivation** for consumer *contract* (context providers,
   HTML attributes, environment, config flags the host must supply for the
   imported symbol to work). Answered by the AI from the spec body plus
   the already-loaded stack skill bodies.
3. **Optional spec section** (`## Host preconditions`) for the rare case
   where a precondition is a project policy not derivable from imports.

Two consequences follow:

- The stack-skills manifest stays at the eight major entries it has today
  (React, Next.js, composition, web guidelines, building, cache, shadcn,
  agent-browser). No new `kind` classification, no adapter skills, no
  per-package reference files.
- The cost of an unknown package is paid in *false negatives at runtime*
  (AI missing a precondition), not in *manifest debt* (entries silently
  rotting). The escape hatch and a conservative-disclosure rule (described
  under the AI-knowledge channel) bound how often false negatives happen
  in practice.

## Mechanism 1 — type-driven import resolution

Resolves the icon-deprecation gap and generalizes to any external package
whose current import form is encoded in its types.

### Where it lives

The existing **External dependencies** slot in
[`skills/generate-component/references/host-discovery.md`](../../../skills/generate-component/references/host-discovery.md)
is extended. The slot currently resolves *presence* (is the package in
`package.json`?) with failure label `npm-install`. It gains a second
responsibility: resolve the *current symbol name* for each abstract role
the spec names.

The extension is a property of the same slot, not a new slot, because the
inputs are the same (the spec's external dependency list) and the output
flows to the same Plan rows.

### Algorithm

For each external package the spec depends on, and for each abstract role
the spec names from that package (e.g., the icon role `moon`):

1. **Candidate generation.** The AI produces ordered candidate symbol
   names from convention-aware heuristics — for icon libraries, prefer
   the `<Capital>Icon` form, fall back to `<Capital>`.
2. **Type-definition check.** Read `node_modules/<pkg>/**/*.d.ts` in the
   host. Glob for export sites of each candidate. For Next.js / React
   library packages, prefer the package's main entry point or `exports`
   map; submodules (e.g., `/ssr` for RSC contexts) are recognized by
   their `package.json` `exports` field.
3. **Deprecation check.** For each candidate that appears in the types,
   inspect its surrounding JSDoc for an `@deprecated` tag. Tagged
   candidates rank below untagged ones.
4. **Selection.** Pick the highest-ranked candidate that exists and is
   not deprecated. If only deprecated candidates exist, pick the
   highest-ranked deprecated one and surface the deprecation note in
   the Plan row.
5. **Submodule selection.** When the spec's `## Where` or `## How`
   implies an RSC context, prefer a submodule export
   (`@phosphor-icons/react/ssr`) when the package's `exports` field
   advertises one.

### Plan rendering

Each resolved external symbol becomes one row in the Plan's Items table:

| slot | value | source |
|---|---|---|
| `Import: <role> from <pkg>` | `<SymbolName>` | `host` |

The `value` cell may also carry a parenthetical note when relevant:
`SunIcon (host types; "Sun" available but @deprecated)`, or
`SunIcon from @phosphor-icons/react/ssr (host types; RSC context)`.

`source: host` is used because the value is derived from a host file
(`node_modules` is part of the host's installed state).

### When the package is absent

If the package is missing from `package.json`, the existing `npm-install`
label fires first; type-driven resolution runs only after the install
prerequisite is realized in Step 7 (Realize prerequisites). This ordering
is already implied by the existing slot policy and is reaffirmed here so
that the resolution step has the file it needs.

## Mechanism 2 — AI-knowledge preconditions

Resolves the host-preconditions gap and generalizes to any external
package whose use requires the host to be set up a specific way.

### Where it lives

A new slot is added to the Slot policy table in
[`skills/generate-component/references/host-discovery.md`](../../../skills/generate-component/references/host-discovery.md):

| Slot | What discovery resolves | Label on resolve failure |
|---|---|---|
| Consumer-implied preconditions | For each external symbol the component will import, the host-side conditions that must hold for the symbol to function (context providers in the tree, HTML attributes, config flags, environment). | `host-setup-required` — non-blocking; user can approve the Plan with the precondition unmet. |

This slot is the sixth in the table; the existing five (Component output
directory, Reference primitive location, Active design tokens, External
dependencies, Storybook precondition) keep their current behavior.

### Algorithm

Input: the parsed spec body (from Step 2), the resolved external symbols
(from the extended External dependencies slot above), and the stack skill
bodies already loaded into context by Step 2.5 (Invoke stack skills).

For each external symbol the component will import:

1. **Precondition derivation.** The AI states, in one or two sentences
   per item, what the host must supply for the symbol to behave as the
   spec assumes. Drawn from the loaded stack skill bodies plus the AI's
   own knowledge of the imported package.
2. **Signal specification.** For each derived precondition, the AI also
   produces one concrete signal it can verify in the host with Read /
   Glob — a string to find in a file, an attribute on an HTML element, a
   key in a JSON config. The signal is small, deterministic, and named
   in the Plan row so the user can reproduce the check.
3. **Signal verification.** Each signal is verified against host files
   using Read / Glob.
4. **Plan row composition.** Verified-met preconditions produce no row.
   Verified-unmet preconditions produce one row each with label
   `host-setup-required`.
5. **Conservative disclosure for uncertain cases.** When the AI cannot
   confidently judge whether a precondition applies (the imported symbol
   is unfamiliar, the stack skill bodies do not cover it, the host's
   shape is ambiguous), the precondition is **still** added to the Plan
   with `host-setup-required` and a note that AI confidence is low and
   user confirmation is requested. This biases the channel toward false
   positives (over-disclosed attention rows) rather than false negatives
   (silent runtime failures) — the failure mode the issues recorded.

### Spec optional escape hatch

The spec format (`skills/clarify-component/references/output-format.md`)
gains an **optional** section:

```markdown
## Host preconditions
(optional; omit when none)

- <one-line requirement>: <one-line signal description>
```

Each bullet is a `requirement: signal` pair structured to feed the same
verification path used by Mechanism 2 step 3. `clarify-component` itself
does not ask for this section in the current iteration — it is filled by
the spec author (or by a future iteration of clarify, addressed in
Cluster A). When present, `generate-component` merges the spec's stated
preconditions with the AI-derived ones, deduplicating by requirement
text.

The section is optional. Its absence is normal and means "derivation
covers it."

## Plan label addition

[`skills/generate-component/references/plan-format.md`](../../../skills/generate-component/references/plan-format.md)'s
source-label inventory grows from seven to eight:

- `host-setup-required` — a runtime precondition for an imported symbol
  is not met in the host. **Non-blocking.** The user may approve the
  Plan knowing the precondition is unmet (e.g., intending to fix the
  host afterwards). When the user approves with `host-setup-required`
  rows present, generation proceeds normally; the generated component
  will typecheck and Storybook-build, but may not behave as expected
  in the live host until the precondition is supplied.

Boundary with the existing `attention` label:

- `attention` continues to mean **cosmetic** discrepancy — most commonly
  a spec / `DESIGN.md` token conflict. The component runs and looks
  approximately right; the attention is about polish.
- `host-setup-required` means **runtime** effect — the component runs
  but functional behavior is degraded or wrong without the precondition.

The two labels are intentionally distinct so the user can read the
severity directly from the Plan row, not from the surrounding prose.

The same label is shared by the new slot in `host-discovery.md` and by
the `plan-format.md` inventory, mirroring how the four existing failure
labels (`user-needed`, `shadcn-install`, `npm-install`, `attention`)
appear in both files.

## Reference file changes

Concrete edits to support the design:

- [`skills/generate-component/references/host-discovery.md`](../../../skills/generate-component/references/host-discovery.md)
  - Extend the External dependencies row to describe the symbol-name
    resolution responsibility (Mechanism 1).
  - Add the Consumer-implied preconditions row (Mechanism 2).
  - Add `host-setup-required` to the Plan slot labels section.
- [`skills/generate-component/references/plan-format.md`](../../../skills/generate-component/references/plan-format.md)
  - Add `host-setup-required` to the Source labels list with the
    definition above.
  - Note the non-blocking semantics in the Approval mechanism section
    so the Plan-approval prose does not need to ask the user
    differently when these rows are present.
- [`skills/generate-component/references/output-format.md`](../../../skills/generate-component/references/output-format.md)
  - In the Imports section, add the paragraph stating that external
    symbol names are determined by host type definitions (pointing
    at `host-discovery.md`'s External dependencies slot for the
    algorithm).
- [`skills/clarify-component/references/output-format.md`](../../../skills/clarify-component/references/output-format.md)
  - Add the optional `## Host preconditions` section definition under
    the full-path body template.
- [`skills/generate-component/SKILL.md`](../../../skills/generate-component/SKILL.md)
  - Step 4 (Discover host context) is unchanged at the step level — the
    new slot is one more row in the slot policy table it already
    consults, not a new step.
- [`skills/setup/references/stack-skills.md`](../../../skills/setup/references/stack-skills.md)
  - **Unchanged.** Zero new entries, zero new columns.

## Out of scope

- **clarify-component dimension changes.** Whether `## Host preconditions`
  is asked for during clarification — and the broader question of how
  clarify-component handles implementation-pattern decisions — is the
  scope of Cluster A
  ([`docs/issues/2026-05-25-clarify-component-implementation-leaks-into-spec.md`](../../issues/2026-05-25-clarify-component-implementation-leaks-into-spec.md),
  [`docs/issues/2026-05-25-clarify-component-interaction-model-variants.md`](../../issues/2026-05-25-clarify-component-interaction-model-variants.md)).
  This spec defines the section's format and consumption only.
- **Per-package adapter skills.** Considered and rejected in favor of
  derivation. If a future package proves derivation-resistant in
  practice, a follow-up spec can introduce adapters as a targeted
  remediation rather than a default mechanism.
- **Runtime-time precondition checks during verification.** Step 9's
  verification loop continues to run typecheck and Storybook build
  only. Live-host runtime checks (e.g., booting the host's `next dev`
  to confirm the rendered tree actually contains `<ThemeProvider>`)
  are deferred to a future verification slice. The current loop will
  continue to pass even when `host-setup-required` rows were approved;
  the user's choice to approve is the contract.
- **Manifest classification changes.** No skill is reclassified.

## Verification

After implementation, re-run the full pipeline on the mode-toggle case
plus one generalization case:

### Scenario A — mode-toggle re-run (closes both issues)

Same spec, same host (the host where `<ThemeProvider>` and
`suppressHydrationWarning` are not wired into `app/layout.tsx`).

Expected Plan changes from the prior run:

1. **Import rows reflect type-driven names.** Two import rows for the
   icon role become:
   - `Import: icon "moon" from @phosphor-icons/react` → `MoonIcon (host types; "Moon" available but @deprecated)`, source `host`.
   - `Import: icon "sun" from @phosphor-icons/react` → `SunIcon (host types; "Sun" available but @deprecated)`, source `host`.
2. **Two `host-setup-required` rows appear:**
   - `Precondition: <ThemeProvider> in app/layout.tsx` — signal: "the
     file `app/layout.tsx` contains the substring `<ThemeProvider`".
     Verification: signal not found in the host. Source:
     `host-setup-required`.
   - `Precondition: suppressHydrationWarning on <html> in
     app/layout.tsx` — signal: "the file `app/layout.tsx` contains the
     substring `<html` with `suppressHydrationWarning` on the same
     element". Verification: signal not found. Source:
     `host-setup-required`.
3. **Generated component file**, when the user approves the Plan
   knowing those rows: emits `import { MoonIcon, SunIcon } from
   "@phosphor-icons/react"` (was `Sun, Moon`) and `import { useTheme }
   from "next-themes"`. The mount-guard pattern
   (`useState(false) + useEffect(setMounted(true))`) **remains** in the
   emitted file because the spec's Edge cases prose still prescribes
   it — removing that prose is Cluster A's responsibility and not part
   of this scenario's verification.

Both
[`docs/issues/2026-05-25-stack-skills-icon-library-deprecation-drift.md`](../../issues/2026-05-25-stack-skills-icon-library-deprecation-drift.md)
and
[`docs/issues/2026-05-25-generate-component-host-preconditions-from-imports.md`](../../issues/2026-05-25-generate-component-host-preconditions-from-imports.md)
move to `resolved` when this scenario produces the expected Plan and
the user approves it.

### Scenario B — generalization to an unrelated package

Pick one spec that imports from a package not previously exercised by
the test corpus — for example, a component spec whose `## What` calls
for a `motion.div`-style animation from `framer-motion`. Run
`/gyre:generate-component` against it.

Expected:

- Type-driven slot resolves the `motion.div` import without any
  manifest entry for `framer-motion`. Plan rows show host-derived names.
- AI-knowledge slot produces no `host-setup-required` rows when
  framer-motion has no consumer-context requirement, or one such row
  for `<LazyMotion>` if the spec implies use of lazy features. Either
  outcome is acceptable; what is verified is that the channel runs
  without per-package configuration.

This scenario fails if either channel requires adding a manifest entry
or a package-specific reference file to function. That would falsify the
derivation principle and trigger a re-design.

## Follow-ups

- **AI-derived precondition accuracy.** If `host-setup-required` rows
  in production runs prove systematically wrong (false negatives on
  packages the AI knew nothing about, or false positives on
  unproblematic imports), the remediation is a small **calibration
  step** in Mechanism 2 — not a new manifest. Log offending runs as
  new issues under `docs/issues/` so the calibration draws from real
  cases.
- **Deprecation note display.** The Plan rendering above shows the
  `@deprecated` note inline. If the parenthetical clutters the table,
  the rendering may move to a footnote-style addendum below the
  Items table. Defer until the first user-facing run reveals which
  reads better.
- **Type-driven resolution caching.** Reading `node_modules` `.d.ts`
  files on every run is acceptable today (small generation workloads).
  If generation becomes a hot path, cache the resolved symbol map per
  package version. Out of scope until the pattern emerges.

## Related

- Open issues this design closes:
  - [`docs/issues/2026-05-25-stack-skills-icon-library-deprecation-drift.md`](../../issues/2026-05-25-stack-skills-icon-library-deprecation-drift.md)
  - [`docs/issues/2026-05-25-generate-component-host-preconditions-from-imports.md`](../../issues/2026-05-25-generate-component-host-preconditions-from-imports.md)
- Adjacent issues (Cluster A — out of scope here):
  - [`docs/issues/2026-05-25-clarify-component-implementation-leaks-into-spec.md`](../../issues/2026-05-25-clarify-component-implementation-leaks-into-spec.md)
  - [`docs/issues/2026-05-25-clarify-component-interaction-model-variants.md`](../../issues/2026-05-25-clarify-component-interaction-model-variants.md)
- Prior design that established the Step 2.5 stack-skill invocation
  this design relies on:
  [`2026-05-25-stack-skills-invocation-design.md`](2026-05-25-stack-skills-invocation-design.md)
- Manifest (intentionally unchanged):
  [`skills/setup/references/stack-skills.md`](../../../skills/setup/references/stack-skills.md)
- generate-component SKILL (Step 4 consumes the extended slot policy):
  [`skills/generate-component/SKILL.md`](../../../skills/generate-component/SKILL.md)
- Reference files edited by this design:
  - [`skills/generate-component/references/host-discovery.md`](../../../skills/generate-component/references/host-discovery.md)
  - [`skills/generate-component/references/plan-format.md`](../../../skills/generate-component/references/plan-format.md)
  - [`skills/generate-component/references/output-format.md`](../../../skills/generate-component/references/output-format.md)
  - [`skills/clarify-component/references/output-format.md`](../../../skills/clarify-component/references/output-format.md)
