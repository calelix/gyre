---
date: 2026-05-26
slug: clarify-component-voice-rule-mechanism-scope-gap
title: clarify-component voice rule rejects only JS-runtime mechanisms, letting other mechanism categories leak into spec
status: resolved
---

# clarify-component voice rule rejects only JS-runtime mechanisms, letting other mechanism categories leak into spec

## Summary
The requirements-only voice rule on `## How → Interactions`,
`## How → Edge cases`, and `## How → Accessibility` (introduced by the
Clarify-Component Silent Defaults design) blocks spec bullets that
cite JavaScript runtime mechanisms — hooks (`useEffect`), lifecycle
moments (`mount`), and named React idioms (`render prop`,
`mount-guard pattern`). It does **not** block other categories of
mechanism that play the same role: ARIA attribute names, CSS utility
class names, HTML element selections among accessible alternatives,
CSS technique names, web-platform feature names. The rule's
underlying principle ("requirements describe observable properties,
not the means") covers all of these, but the example set is drawn
from one category only, so the AI fails to generalize. The
sibling-resolved issue
([`2026-05-25-clarify-component-implementation-leaks-into-spec.md`](2026-05-25-clarify-component-implementation-leaks-into-spec.md))
closed the JS-runtime subset; this issue is the same structural
fault re-surfacing in the remaining categories.

## Context
- Playground: a host project (`gyre-ui`) running
  `/gyre:clarify-component` → `/gyre:generate-component` against a
  `mode-toggle` spec consuming `next-themes`.
- Tech stack: Next.js App Router + next-themes + shadcn/ui +
  Tailwind, with Vercel stack skills loaded by `generate-component`
  Step 2.5 (`web-design-guidelines`, `building-components`,
  `vercel-react-best-practices`, `vercel-composition-patterns`,
  `next-best-practices`).
- Detected by: user feedback during post-generation comparison of
  the produced component against the shadcn canonical at
  https://ui.shadcn.com/docs/dark-mode/next. The user observed that
  the spec's `## How → Accessibility` bullet pinned `aria-label` as
  the mechanism, even though `web-design-guidelines` prescribes
  `aria-label` AND `building-components` treats `aria-label` and
  `<span class="sr-only">` as equally valid — i.e., the choice is
  squarely in stack-skill territory and the spec should not have
  pre-committed.

## Problem
- **Expected:** the voice rule blocks any spec bullet that names a
  specific technique that is one of several legitimate ways to
  satisfy an underlying user-observable requirement. The rule should
  fire regardless of *which category* the technique belongs to (JS
  runtime, ARIA attribute, CSS utility, HTML element name, CSS
  technique, web-platform feature, network primitive). Once blocked,
  the bullet is rephrased to state the higher-layer requirement,
  and stack skills loaded at generation time choose the concrete
  technique.
- **Actual:** the spec produced in this run records:

  > "`aria-label` describes the action available given the current
  > theme (for example, 'Switch to dark mode' when the current
  > theme is light)."

  This passes the voice rule today because `aria-label` is not in
  Test 3's example set (which enumerates `useEffect`, `useState`,
  `useContext`, `useRouter`, `mount-guard pattern`, `render prop`,
  and `compound component`).
  The bullet names a specific mechanism among legitimate
  alternatives (`aria-label` vs `<span class="sr-only">` vs
  `aria-labelledby` vs visible text) without the user having been
  asked which mechanism they want, and without deferring the choice
  to the stack skills that own this domain.

  `generate-component` then faithfully emitted `aria-label={...}` —
  not because the stack skills chose it, but because the spec
  pre-committed to it. The deference path in
  [`output-format.md`](../../skills/generate-component/references/output-format.md)
  *Pattern selection deference* never got to evaluate the
  alternatives.

## Root cause
1. **Voice rule Test 3 is defined ostensively, by example list.** The
   rule's prose statement is broad ("does the bullet name a specific
   technique, hook, pattern, or library identifier — in any
   language?"), but the example set is drawn from one mechanism
   category — React/JS runtime — only. When the AI evaluates a
   bullet at authoring or Self-check time, it pattern-matches
   against the examples rather than against the abstract criterion.
   Categories outside the example set escape detection.

2. **No positive criterion for what *qualifies* as a mechanism
   citation.** Test 3 distinguishes "named technique" from
   "observable outcome" through examples on both sides
   ("`useEffect`" rejected, "`aria-expanded='true'`" passed). But it
   provides no generative test — no way to decide a *novel* identifier
   (one not in either example list) by principle rather than by
   analogy. Novel cases default to "pass" because the rule's
   rejection path requires a positive match against the rejection
   examples.

3. **The contract/mechanism boundary is treated as binary when it is
   layered.** A button's accessibility contract has at least three
   layers:
   - L0 (highest, user-observable): "Button has an accessible name
     that, when read aloud, conveys the next action available given
     the current theme."
   - L1: "Button has an `aria-label` attribute whose value conveys
     that action."
   - L2: "Button has `aria-label='Switch to dark mode'` when in
     light mode and `aria-label='Switch to light mode'` when in
     dark mode."

   Each layer down is more mechanism-y; each is also genuinely
   verifiable. The voice rule does not articulate "state the
   requirement at the highest layer where user intent is fully
   captured", so it does not prefer L0 to L1 to L2. Spec authors
   default to L1 prose because it sounds technical and concrete.

4. **The `generate-component` deference principle is undermined
   silently.** Pattern selection deference
   ([`output-format.md`](../../skills/generate-component/references/output-format.md))
   says stack skills choose the mechanism. For that to actually
   operate, the spec must leave the mechanism unspecified at the
   layer where the skill speaks. Test 3's narrow scope means the
   spec routinely pre-commits at layers stack skills are authoritative
   on — making the deference rule a no-op for accessibility, CSS
   strategy, and HTML element choice, even though it still works
   for hydration handling (the JS-runtime category Test 3 actually
   covers).

## Mechanism categories the current rule misses
Non-exhaustive — each category has produced or could produce a leak:

| Category | Example identifiers | Legitimate alternatives among which a stack skill should choose |
|---|---|---|
| ARIA attributes | `aria-label`, `aria-describedby`, `aria-labelledby`, `aria-current`, `role` | sr-only text, visible label, `aria-labelledby` reference |
| CSS utility class names | `sr-only`, `visually-hidden`, `dark:hidden`, `not-dark:` | semantic markup choice, conditional rendering, attribute-based selectors |
| HTML element selection | `<button>` vs `<a role="button">` vs `<div tabindex="0">` | each satisfies "interactive control with accessible name" |
| CSS technique citations | `transform: scale()`, `transition-all`, `prefers-reduced-motion`, `:focus-visible` | each satisfies a higher-layer visual or interaction requirement |
| Web-platform feature names | `matchMedia`, `IntersectionObserver`, `MutationObserver`, `localStorage` | each satisfies "react to environment change X" |
| Network primitive names | `fetch`, `XMLHttpRequest`, `EventSource`, `WebSocket` | each satisfies "obtain data from server" |
| Routing primitive names | `<Link>`, `useRouter().push`, `<a href>`, `redirect()` | each satisfies "navigate to URL X under condition Y" |

The pattern is identical across all rows: a named technique that is
one of several legitimate ways to satisfy a higher-layer
user-observable contract.

## Remediation
Not yet applied. The proposed direction has four prongs; they should
land together because each addresses a distinct facet of the
structural fault above.

1. **Reframe Test 3 from an example list to a generative criterion.**
   Replace "does the bullet name a specific technique, hook,
   pattern, or library identifier" with: "could a future stack-skill
   prescription substitute a different named technique for the same
   higher-layer user-observable contract without changing what the
   user perceives? If yes, the bullet has cited a mechanism and
   must be restated at the higher layer." The example sets remain
   as anchors but lose their gatekeeping role.

2. **Expand the example sets to cover the categories in the table
   above.** The new examples are illustrative of the *generative
   criterion*, not a closed vocabulary. Both `dimensions.md` and
   `output-format.md` carry the rule prose; both must be updated
   identically.

3. **Add the contract-layering framing to the rule prose.** Document
   the L0 / L1 / L2 stratification with a concrete worked example
   (the mode-toggle accessibility case is a natural one). Spec
   authors should state requirements at L0 unless the user
   explicitly requests a lower layer for parity, in which case the
   prose routes to `## Implementation hints` per the existing
   reject flow.

4. **Cross-reference from `generate-component`'s Pattern selection
   deference back to the voice rule.** Deference and voice rule are
   two halves of the same principle: clarify must not pre-commit to
   what generate is supposed to choose. Currently each file
   describes its half independently; a one-line back-reference in
   each makes the contract explicit and surfaces the rule's
   spec-side gatekeeping role.

The fix is **rule scope expansion, not new dialogue dimensions**.
The Self-check step (Step 10) is the enforcement point and remains
unchanged in structure; only the rule the step evaluates against
broadens.

## Verification
**Resolved on 2026-05-26.** Verified on a separate host project re-running `/gyre:clarify-component` against the `mode-toggle` source text. The produced spec's `## How → Accessibility` was authored at L0 (an accessible-name-conveys-next-action sentence) rather than pinning `aria-label` as the mechanism. The downstream `/gyre:generate-component` run then selected the concrete attribute under stack-skill prescription rather than from a pre-committed spec value — see the sibling issue's verification for the emitted-code observations.

Host project identifier omitted by request; the regression scenarios below remain the durable record for future runs.

Re-running `/gyre:clarify-component` against the same `mode-toggle`
source text should:

- Produce `## How → Accessibility` bullets stated at L0:
  - "Button exposes an accessible name that, when read aloud,
    conveys the next action available given the current theme."
  - rather than: "`aria-label` describes the action..."
- If the AI authors an L1 bullet during dialogue, the Self-check
  step rejects it and dialogue resumes with a rephrase question.
- If the user confirms the specific mechanism is the requirement
  (e.g., "I want `aria-label` specifically for parity with the
  shadcn ModeToggle"), the prose routes to `## Implementation hints`
  with the user's rationale.

Regression checks against other categories should produce the same
behavior:

- A spec mentioning `sr-only` in `## How → Accessibility` is
  rephrased to "the action label is announced by assistive
  technology but is not visually rendered as flowing text".
- A spec mentioning `prefers-reduced-motion` in `## How → Edge cases`
  is rephrased to "users with the reduced-motion preference receive
  a presentation without animated transitions".
- A spec mentioning `<div role="button">` in `## How → Accessibility`
  is rephrased to "the control exposes button role to assistive
  technology and is reachable by keyboard tab order".

`generate-component` for each rephrased spec, with the relevant
stack skills loaded, then selects the concrete mechanism per
skill prescription. Where two loaded skills disagree on the
mechanism, the more-specific skill wins per the existing rule
(`output-format.md` *Pattern selection deference*).

## Follow-ups
- **False-positive calibration.** A broader rule risks rejecting
  legitimate references — most notably ARIA *value* citations
  (`aria-expanded='true'`) that capture *which state* the button
  exposes, which IS a user-observable contract. The current rule
  already exempts these by example; the broader rule must keep
  that exemption explicit. Log any false-positive runs as new
  issues so calibration draws from real cases.
- **Existing specs.** The mode-toggle spec produced during this
  conversation
  (`<host>/docs/gyre/specs/components/mode-toggle.md`) carries an
  L1 aria-label bullet. Backward migration of existing specs is
  out of scope; re-running `clarify-component` on the original
  source text produces a clean spec.
- **Test 1 (Outcome) and Test 2 (Procedural) review.** Both tests
  have the same example-list-as-criterion structure. If they
  exhibit analogous categorical gaps (e.g., Test 2 rejects English
  sequence words but lets visual sequence prose pass), open
  sibling issues. This issue scope is limited to Test 3.
- **`web-design-guidelines` interaction.** The Vercel Web Interface
  Guidelines explicitly prescribe `aria-label` for icon-only
  buttons but make no statement about `sr-only`. Even when this
  rule passes the spec to skill territory, the stack-skill side
  may pick `aria-label` again — that is the correct outcome.
  The point of the rule is not to *change* which mechanism is
  emitted in the current setup; it is to put the decision in the
  right layer so changing skills later changes the emitted code
  without a spec round-trip.

## Related
- Sibling issue (the JS-runtime subset of this same fault):
  [`2026-05-25-clarify-component-implementation-leaks-into-spec.md`](2026-05-25-clarify-component-implementation-leaks-into-spec.md).
- Voice rule definition sites (both must be updated):
  - [`skills/clarify-component/references/dimensions.md`](../../skills/clarify-component/references/dimensions.md)
    *Note on requirements-only voice for `## How`*.
  - [`skills/clarify-component/references/output-format.md`](../../skills/clarify-component/references/output-format.md)
    *`## How` voice rule*.
- Self-check enforcement point:
  [`skills/clarify-component/SKILL.md`](../../skills/clarify-component/SKILL.md)
  Step 10.
- Deference rule the voice rule must keep in scope:
  [`skills/generate-component/references/output-format.md`](../../skills/generate-component/references/output-format.md)
  *Pattern selection deference*.
- Spec produced this run that exhibits the leak:
  `<host>/docs/gyre/specs/components/mode-toggle.md`
  (`## How → Accessibility` bullet on `aria-label`).
- Loaded stack skills the spec should have deferred to:
  - `web-design-guidelines` — prescribes `aria-label` for
    icon-only buttons; silent on `sr-only`.
  - `building-components` — treats `aria-label` and
    `<span class="sr-only">` as equally valid; no preference.
- Stack skills manifest:
  [`skills/setup/references/stack-skills.md`](../../skills/setup/references/stack-skills.md).
- Reference shadcn canonical for the same component:
  https://ui.shadcn.com/docs/dark-mode/next.
