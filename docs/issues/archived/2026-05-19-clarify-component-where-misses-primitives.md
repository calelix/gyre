---
date: 2026-05-19
slug: clarify-component-where-misses-primitives
title: clarify-component's Where step missed the base primitive
status: resolved
---

# clarify-component's Where step missed the base primitive

## Summary
The Where dimension of the clarify-component skill only asked about same-role components, never about base primitives. As a result, the "theme toggle button" request was resolved as Decision: New, missing the existing Button component. The schema itself (Reuse/Extend/New with `Reference` allowed only under Extend) also had a structural gap, so fixing the question alone left nowhere to record the dependency. Decision was extended with `Compose`, and the Where question was split into Q1a/Q1b — closing all three causes together.

## Context
- Playground: [experiments/shadcn](../../experiments/shadcn)
- Tech stack: pnpm + Turborepo monorepo, Next.js 16.1.6 (Turbopack), React 19, TypeScript 5.9, Tailwind CSS 4, next-themes 0.4, @phosphor-icons/react, internal `@workspace/ui` package (carrying primitives such as Button)
- Detected by: User feedback. Immediately after the clarify-component artifact `mode-toggle.md` was written, the user pointed out that the request was for a *button*, and `packages/ui/src/components/button.tsx` should have been the foundation.

## Problem
While drafting the ModeToggle component spec via the clarify-component skill, the Where step asked a single question — "is there already a similar component?" — and the user's answer of "no — build a new one" was taken to close the Where decision as New. As a consequence, the existing Button primitive dependency surfaced nowhere in the artifact, and the downstream dimensions (How / Look / Data) were all written without the understanding that this component is assembled on top of Button.

Expected: the word "button" present in the request ("theme toggle *button*") should have surfaced the category, and a separate question — "is there already a base Button primitive in this project?" — should have captured the Button dependency in the spec.

## Root cause
Three structural causes compounded.

1. **Schema gap (most structural).** Where Outcomes only had `Reuse / Extend / New`, and output-format.md restricted the `Reference` field to `(Extend only)`. There was no slot to express "new identity + assembled on top of an existing primitive" — the most common case for leaf UI components.
2. **Question ambiguity.** "Is there already a *similar* component...?" naturally read as "is there a duplicate (same role)?". With no separate question about a same-category primitive, a single negative answer closed both possibilities at once.
3. **No use of request-text signals.** The request contained the category word "button" explicitly, but the skill had no pre-step to extract category words and feed them into a targeted follow-up.

Fixing only (2) and (3) without (1) would leave no field to record the dependency, forcing it back into Extend (semantically wrong). All three had to close together.

## Remediation
Skill enhanced to close the three causes (Option A — add `Compose` to Decision).

- [skills/clarify-component/references/dimensions.md](../../../skills/clarify-component/references/dimensions.md)
  - Added a *composition-aware framing* note at the top: when Where is Extend or Compose, downstream dimensions (Look/How/Data) must distinguish "what the Reference already provides" from "what this component *adds*".
  - Rewrote the Where section: a pre-step to extract category words from the request text; questions split into **Q1a same-role candidate + Q1b base primitive + Q2 disposition**; an explicit note that a negative answer to Q1a does not close Q1b.
  - Added **`Compose`** to Outcomes: Q1a = none + Q1b = exists → Compose, with the primitive identifier recorded as `Reference`.
  - Added a **Decision concepts table** at the top of the Where section (Reuse / Extend / Compose / New). The two boundaries that matter most — *Compose vs. New* and *Compose vs. Extend* — are spelled out, so the same mistake ("building on top of Button but recording as New") is blocked at the conceptual level.
- [skills/clarify-component/references/output-format.md](../../../skills/clarify-component/references/output-format.md)
  - Body heading: "new, extended, or composed component".
  - `Decision: New | Extend | Compose`.
  - `Reference: (Extend or Compose only)` and now allows multiple identifiers.
- [skills/clarify-component/SKILL.md](../../../skills/clarify-component/SKILL.md)
  - Step 3 branch list now includes Compose.
- The `mode-toggle.md` artifact (formerly at `docs/gyre/specs/components/`) was updated to the new schema (Decision: Compose, Reference: Button), acting as the first consistent example of Compose.

## Verification
- Checked text-level consistency across the 3 modified skill files and 1 artifact. Decision/Reference/Outcomes/Step 3 all handle Compose consistently.
- mode-toggle.md now follows the new schema, with `Button` explicitly named in the `Reference` field.
- End-to-end re-run — observing Q1a/Q1b branching and a Compose decision recorded in the artifact during the next `/gyre:clarify-component` invocation — is deferred to the next invocation (see Follow-ups).

## Follow-ups
- On the next `/gyre:clarify-component` invocation, observe whether Q1a/Q1b actually fire and whether the artifact records Compose correctly. If divergence appears, file a new issue.
- Review whether `references/risk-signals.md` should reflect the Compose decision — assembling on a stable primitive may justify lowering State character by one level.

## Related
- [skills/clarify-component/SKILL.md](../../../skills/clarify-component/SKILL.md)
- [skills/clarify-component/references/dimensions.md](../../../skills/clarify-component/references/dimensions.md)
- [skills/clarify-component/references/output-format.md](../../../skills/clarify-component/references/output-format.md)
