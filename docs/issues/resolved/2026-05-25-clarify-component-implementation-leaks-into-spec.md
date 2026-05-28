---
date: 2026-05-25
slug: clarify-component-implementation-leaks-into-spec
title: clarify-component lets implementation patterns leak into spec requirements
status: resolved
---

# clarify-component lets implementation patterns leak into spec requirements

## Summary
The spec produced by `clarify-component` records concrete implementation
patterns (e.g., a `mounted` boolean guarded by `useEffect`, a specific
placeholder element) inside requirement sections (`## How → Edge cases`,
`## How → Interactions`) as if they were behavioral requirements. Once
those patterns are pinned in the spec, `generate-component` reproduces
them verbatim under the "spec is contract" rule, even when newer
best-practices in the relevant stack would replace them with simpler or
more idiomatic patterns.

## Context
- Playground: a host project running `/gyre:clarify-component` →
  `/gyre:generate-component` against a `mode-toggle` spec consuming
  `next-themes`.
- Tech stack: Next.js App Router + next-themes + Tailwind `dark:`
  variants — a stack where the modern best practice for hydration
  handling differs from the older "mounted guard" pattern.
- Detected by: post-generation comparison against the shadcn/ui
  canonical at https://ui.shadcn.com/docs/dark-mode/next, which
  satisfies the same requirement using only
  `<html suppressHydrationWarning>` and Tailwind `dark:` variants —
  no mount guard at all.

## Problem
- **Expected:** the spec describes the *requirement* ("no layout shift
  across SSR → CSR for this control") and leaves the implementation
  pattern that satisfies it to `generate-component`, which can apply
  the current stack best practice.
- **Actual:** the spec's `## How → Edge cases` explicitly prescribes
  the implementation:

  > "Before SSR and client mount, next-themes' resolvedTheme is
  > undefined, so render a neutral placeholder (transparent/empty box)
  > of the same size to prevent layout shift, then swap to the real
  > icon after client mount completes."

  That sentence is a specific pattern (the `mounted` + `useEffect`
  guard idiom from older next-themes guides), not a requirement
  statement. `generate-component` reproduced it faithfully —
  including the `useState(false)` + `useEffect(setMounted(true))`
  scaffolding — even though the shadcn canonical satisfies the same
  requirement without any of it.

## Root cause
`clarify-component`'s output format does not separate two distinct
things:

1. **Behavioral requirement** — *what* must be true about the component
   (no layout shift, click toggles theme, sr-only label present).
2. **Implementation pattern** — *how* to achieve a requirement on a
   given stack (mount-guard pattern, CSS-only dark variant, etc.).

The current `## How` section conflates both. Edge-case prose is written
in implementation terms when the AI knows a specific pattern that
satisfies the requirement. This conflation makes the pattern itself
part of the spec contract, removing `generate-component`'s ability to
apply a better stack-current pattern at generation time.

The deeper structural fault is that clarify is allowed to pre-decide
implementation when the user did not explicitly request that pattern.
The user's source text in this run made no mention of `mounted` guards
or placeholders — the AI added that prose.

## Remediation
Resolved by the Clarify-Component Silent Defaults design (see Related).
The applied remediation has two prongs:

- **Requirements-only voice rule** on `## How → Interactions`,
  `## How → Edge cases`, and `## How → Accessibility`. Enforced at
  authoring time and at the Self-check step (Step 10 in
  [`skills/clarify-component/SKILL.md`](../../skills/clarify-component/SKILL.md)).
  The rule rejects bullets containing library/API names the AI
  authored, procedural steps, or named implementation idioms.
- **Optional `## Implementation hints` escape hatch** in the spec
  format
  ([`skills/clarify-component/references/output-format.md`](../../skills/clarify-component/references/output-format.md)).
  When the user confirms a specific pattern itself is the requirement
  (e.g., parity with an existing component), the voice-rule reject
  flow routes the prose to this section with the user's stated
  rationale. `clarify-component` does not ask for this section
  unprompted.
- `generate-component` consumes hints as **strong recommendations,
  not contracts**. Stack skill prescriptions for the same underlying
  requirement override hints, with a Plan `attention` row noting
  the divergence
  ([`skills/generate-component/references/output-format.md`](../../skills/generate-component/references/output-format.md)
  → *Implementation hints handling*).

## Verification
Manual end-to-end re-run owned by the design's Scenario A and B
(see Related → design spec). Re-running `/gyre:clarify-component`
against the same `mode-toggle` source text should:

- Produce a spec whose `## How → Edge cases` records the SSR/CSR
  no-layout-shift requirement as a *requirement* ("no layout shift
  across the SSR → CSR transition for this control"), not as the
  `mounted` guard implementation.
- If the AI authors the mount-guard sentence during dialogue, the
  Self-check step (Step 10) rejects it and dialogue resumes with a
  rephrase question.
- If the user confirms the pattern itself is the requirement, the
  spec gains a `## Implementation hints` section with the bullet
  and the user's rationale.

## Follow-ups
- The applied design fixes the *write-side* (clarify rejects
  implementation prose at authoring; generate surfaces divergence as
  `attention`). The *read-side* — legacy specs that still contain
  implementation prose authored before this change — is not migrated.
  Re-running `/gyre:clarify-component` on the same source text
  produces a clean spec; a bulk migration is out of scope.
- Voice-rule rejection-criteria calibration. If the rule rejects too
  aggressively (false positives on legitimate requirements that
  happen to mention API surfaces the spec must reference) or too
  loosely (false negatives), log offending runs as new issues so
  calibration draws from real cases.

## Related
- Spec produced this run:
  `<host>/docs/gyre/specs/components/mode-toggle.md`
  (sections `## How → Edge cases`, `## Data / Content → Edge data`).
- Generate-component contract:
  [`skills/generate-component/SKILL.md`](../../skills/generate-component/SKILL.md)
  ("spec is contract"; spec ambiguity surfaces as termination with a
  pointer to clarify).
- Comparison reference: https://ui.shadcn.com/docs/dark-mode/next.
- Sibling issue (interaction-model variants):
  [`2026-05-25-clarify-component-interaction-model-variants.md`](2026-05-25-clarify-component-interaction-model-variants.md).
