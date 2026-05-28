---
date: 2026-05-25
slug: clarify-component-interaction-model-variants
title: clarify-component locks in one interaction model without surfacing variants
status: resolved
---

# clarify-component locks in one interaction model without surfacing variants

## Summary
When the user's request maps to a UI pattern that has more than one
legitimate interaction model (e.g., `mode-toggle` → binary button toggle,
cycle button, 3-way dropdown), `clarify-component` produces a spec that
fixes one model silently — typically the variant the AI infers from the
phrasing — without presenting the alternatives to the user. The choice
becomes a stale spec default rather than an informed user decision, and
`generate-component` has no path to revisit it because its contract is
"spec is contract."

## Context
- Playground: a host project running `/gyre:clarify-component` →
  `/gyre:generate-component` against a "mode-toggle" spec.
- Tech stack: shadcn/ui + Next.js host; shadcn ships a canonical
  mode-toggle implementation that uses the 3-way dropdown variant.
- Detected by: post-generation comparison of the produced component
  against the shadcn/ui canonical at
  https://ui.shadcn.com/docs/dark-mode/next, plus user feedback noting
  that binary toggle is itself a legitimate choice — the issue is the
  absence of the question, not the answer.

## Problem
- **Expected:** when a request maps to a pattern with more than one
  legitimate interaction variant, clarify enumerates the variants and
  obtains an explicit choice from the user before fixing one in the spec.
- **Actual:** the user said
  *"Build a mode-toggle component that switches between dark mode and light mode."*
  The spec silently locked in the binary-toggle variant:
  `## Where → Reference: Button` (no `DropdownMenu`), and `## Non-goals`
  excluded "system" as an option without prompting. The user was never
  shown that a cycle button or a 3-way dropdown were alternatives. The
  variant choice was the AI's, not the user's.

## Root cause
`clarify-component`'s clarification dimensions (Where / What / Look / How /
Data / Non-goals) have no slot that explicitly asks "which interaction
model among the legitimate variants for this kind of UI." When a request
maps to a class of pattern with known variants, the AI fills in one
variant from inference and writes the downstream consequences (Reference
primitives, Non-goals, How → Interactions) accordingly. Once those
appear in the spec, `generate-component` treats them as contract and the
variant choice is no longer revisitable without a clarify round-trip.

The deeper structural issue is that clarify has no canonical-pattern
awareness: it does not consult any index, or the `shadcn` skill, to
detect that the requested name corresponds to a documented pattern with
multiple shipped variants. Without that, the variants never enter the
clarification dialogue at all.

## Remediation
Resolved by the Clarify-Component Silent Defaults design (see Related).
The applied remediation:

- **New Interaction model dimension** added to clarify, between
  `Where` and `What` in the dimension order
  ([`skills/clarify-component/references/dimensions.md`](../../skills/clarify-component/references/dimensions.md)).
  When activated, the dimension asks a single multiple-choice question
  enumerating the variants the AI identified for the requested
  pattern, and records the user's explicit choice in a new
  `## Interaction model` spec section.
- **Activation by AI judgment** from the request text. clarify does
  **not** invoke any stack skill for the activation decision (no
  `shadcn` registry call) — clarify remains a self-contained dialogue
  with no host or stack dependency. The general form of "canonical
  reference matching in clarify" is deferred; the activation criterion
  uses AI general knowledge with a bias toward activation when
  uncertain.
- **Compose-fixed compression.** When the *Where* decision is
  `Compose` and the Reference primitive itself fixes the variant
  (e.g., `Compose using DropdownMenu` → dropdown variant chosen),
  the dimension compresses to a single line in the spec rather than
  asking a redundant question.

The dimension's chosen variant calibrates subsequent dimensions
(*What* props surface, *How* states and interactions) so the spec
captures one coherent interaction model rather than a silent default
plus mismatched downstream fields.

## Verification
Manual end-to-end re-run owned by the design's Scenario A (see
Related → design spec). Re-running `/gyre:clarify-component` against
the original `mode-toggle` source text should:

- Activate the Interaction model dimension and ask a multiple-choice
  question enumerating the variants (binary toggle / cycle button /
  3-way dropdown).
- Record the user's explicit choice in a new `## Interaction model`
  spec section with the chosen variant and a rationale line.
- Even if the user chooses the same binary-toggle variant the
  original spec silently used, the choice is now an *explicit* user
  decision rather than a silent default.

A second confirmation against a different multi-variant pattern
(e.g., `combobox`) should produce the same dimension behavior — the
mechanism is general, not mode-toggle-specific.

## Follow-ups
- Variant-detection accuracy calibration. If activation produces
  frequent false negatives (multi-variant patterns the AI failed to
  recognize) or false positives (single-variant components where the
  dimension fires unnecessarily), log offending runs as new issues so
  calibration draws from real cases. The remediation is a refinement
  to the activation criterion in `dimensions.md`, not an external
  catalog or new skill invocation.
- Compose-fixed compression accuracy. If the compression decision
  proves brittle (e.g., the primitive *partly* fixes the variant but
  variations remain), the calibration is a refinement to the
  compression criterion in `dimensions.md`.

## Related
- Spec produced this run:
  `<host>/docs/gyre/specs/components/mode-toggle.md`.
- Comparison reference: https://ui.shadcn.com/docs/dark-mode/next.
- Adjacent clarify issue (archived):
  [`archived/2026-05-19-clarify-component-where-misses-primitives.md`](archived/2026-05-19-clarify-component-where-misses-primitives.md).
- Sibling issue (implementation patterns leaking into spec):
  [`2026-05-25-clarify-component-implementation-leaks-into-spec.md`](2026-05-25-clarify-component-implementation-leaks-into-spec.md).
