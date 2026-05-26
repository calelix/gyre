---
created: 2026-05-26
---

# Clarify-Component: Voice Rule Mechanism Scope Expansion

## Context

The requirements-only voice rule on `## How → Interactions`, `## How
→ Edge cases`, and `## How → Accessibility` — introduced by the
Clarify-Component Silent Defaults design
([`2026-05-25-clarify-component-silent-defaults-design.md`](2026-05-25-clarify-component-silent-defaults-design.md)
§Mechanism 2) — rejects spec bullets that cite JavaScript-runtime
mechanisms (hooks like `useEffect`, lifecycle moments like `mount`,
named React idioms like `render prop`). It does **not** reject other
categories of mechanism that play the same role: ARIA attribute names,
CSS utility class names, HTML element selections among accessible
alternatives, CSS technique names, web-platform feature names, network
primitives, routing primitives.

The rule's underlying principle ("requirements describe observable
properties, not the means") covers all of these. The failure is in the
rule's *delivery shape*: it is defined ostensively by an example list
drawn from one mechanism category only. AI authoring at clarify time
and self-checking at Step 10 pattern-matches against the examples
rather than against the abstract criterion, so categories outside the
example set escape detection.

Observed instance:
[`docs/issues/2026-05-26-clarify-component-voice-rule-mechanism-scope-gap.md`](../../issues/2026-05-26-clarify-component-voice-rule-mechanism-scope-gap.md).
The mode-toggle spec produced via `/gyre:clarify-component` →
`/gyre:generate-component` recorded `## How → Accessibility` as
*"`aria-label` describes the action available given the current
theme"* — pre-committing to one of several legitimate accessible-name
techniques (`aria-label` vs `<span class="sr-only">` vs
`aria-labelledby` vs visible text). `web-design-guidelines` prescribes
`aria-label`; `building-components` treats `aria-label` and
`<span class="sr-only">` as equally valid. The spec pre-committed at
a layer where stack skills are authoritative, making
`generate-component`'s Pattern selection deference rule a no-op for
this mechanism.

The sibling issue
([`docs/issues/2026-05-25-clarify-component-implementation-leaks-into-spec.md`](../../issues/2026-05-25-clarify-component-implementation-leaks-into-spec.md))
closed the JS-runtime subset of this fault. This design extends the
same closure to the remaining categories.

## Principle

**The voice rule's Test 3 is governed by a generative criterion, not
an example list, and applies language-neutrally to AI-authored prose
in any dialogue language.**

Three consequences follow:

- **Generative criterion.** A bullet is rejected by Test 3 when a
  future stack-skill prescription could substitute a different named
  technique for the same higher-layer user-observable contract without
  changing what the user perceives. Examples remain as anchors
  illustrating the criterion; they are not a closed vocabulary and
  not the rule's gatekeeping mechanism.

- **Generative application.** The criterion is applied to the AI's
  prose at authoring time. English anchors illustrate the categorical
  *shape* of mechanism citations, and the AI applies the criterion to
  candidates via the counterfactual question rather than by string
  matching against the anchor list.

- **Layered observability.** A single user-observable contract can
  admit multiple equally observable sub-statements at descending
  layers of specificity. The rule requires stating the requirement at
  the highest layer where user intent is fully captured. Lower-layer
  statements that name a particular technique among legitimate
  alternatives are mechanism citations under Test 3.

The two halves of the spec/code contract — clarify's voice rule (spec
side) and generate's Pattern selection deference (code side) — are
made mutually explicit by one-line back-references.

## Mechanism 1 — Test 3 generative criterion

Closes the rule-shape facet of
[`2026-05-26-clarify-component-voice-rule-mechanism-scope-gap.md`](../../issues/2026-05-26-clarify-component-voice-rule-mechanism-scope-gap.md)
(root cause #1, #2).

### Current rule

> **Test 3 — Mechanism citation test.** Does the bullet name a
> specific technique, hook, pattern, or library identifier — in any
> language? Reject. Examples include but are not limited to: …

Rejection is gated by example match. Novel categories of identifier
default to "pass" because the rule's rejection path requires a
positive match against the rejection examples.

### Replacement rule

> **Test 3 — Mechanism citation test.** Could a future stack-skill
> prescription substitute a different named technique for the same
> higher-layer user-observable contract without changing what the
> user perceives? If yes, the bullet has cited a mechanism and must
> be restated at the higher layer.
>
> Apply the test by attempting the substitution: name an alternative
> technique that would satisfy the same observable outcome. If you can
> name one (the spec author can substitute `<span class="sr-only">`
> text for `aria-label`, `<button>` for `<div role="button">`, `fetch`
> for `EventSource`), the bullet is mechanism-shaped and rejected. If
> no substitution preserves the observable outcome
> (`aria-expanded='true'` IS the observable contract — there is no
> alternative — and "the dropdown opens on click" IS the user-perceived
> behavior), the bullet passes.
>
> Examples below are anchors illustrating the criterion, not a closed
> vocabulary.

The false-positive concern raised in the issue's Follow-ups (ARIA
*value* citations like `aria-expanded='true'`) is handled by the
criterion itself: there is no alternative technique that exposes
"button is expanded" to assistive technology while replacing
`aria-expanded='true'` — the attribute value *is* the user-observable
contract. The counterfactual test returns "no alternative" and the
bullet passes.

## Mechanism 2 — Anchor expansion

Closes the categorical-coverage facet.

### Anchor restructuring

The example block following Test 3 is restructured into two lists —
*Reject anchors* and *Pass anchors* — organized by category.

**Reject anchors** (illustrating categorical shapes the criterion
rejects):

- JS/React runtime: `useEffect`, `useState`, `useContext`,
  `useRouter`, `mount-guard pattern`, `render prop`, `compound
  component`.
- ARIA attribute names (as opposed to ARIA values): `aria-label`,
  `aria-describedby`, `aria-labelledby`, `role="..."` selection.
- CSS utility class names: `sr-only`, `visually-hidden`,
  `dark:hidden`, `not-dark:`.
- HTML element selection among interactive controls: `<button>` vs
  `<a role="button">` vs `<div tabindex="0">`.
- CSS technique citations: `transform: scale()`, `transition-all`,
  `prefers-reduced-motion`, `:focus-visible`.
- Web-platform feature names: `matchMedia`, `IntersectionObserver`,
  `MutationObserver`, `localStorage`.
- Network primitive names: `fetch`, `XMLHttpRequest`, `EventSource`,
  `WebSocket`.
- Routing primitive names: `<Link>`, `useRouter().push`, `<a href>`,
  `redirect()`.

**Pass anchors** (illustrating shapes the criterion permits):

- Domain-level outcome statements: "the dropdown opens on click",
  "focus moves to the first item".
- ARIA attribute *values* describing observable state:
  `aria-expanded='true'`, `aria-current='page'`,
  `aria-pressed='false'`.
- User's verbatim request quoted back. The rule fires on AI-authored
  prose only.
- Observable constraints with thresholds: "contrast ratio ≥ 4.5:1",
  "control is reachable by keyboard tab order".

## Mechanism 3 — Layered observability boundary

Closes the layer-collapse facet (root cause #3).

### Where it lands

After Mechanism 2's anchor block and the existing reject-flow routing
paragraph ("When a bullet trips any of the three tests, the AI
returns to dialogue with a single rephrase question..."), a new
*Boundary: layered observability* paragraph is added, before the
enforcement-pointer paragraph.

### Content

> **Boundary: layered observability.** When a single user-observable
> contract admits multiple equally observable sub-statements at
> descending layers of specificity, state the requirement at the
> **highest layer where user intent is fully captured**. Lower-layer
> statements that name a particular technique among legitimate
> alternatives are mechanism citations under Test 3.
>
> Worked example — mode-toggle accessibility:
>
> - L0 (highest, user-observable): *"Button exposes an accessible
>   name that, when read aloud, conveys the next action available
>   given the current theme."*
> - L1: *"Button has an `aria-label` attribute whose value conveys
>   that action."* ← rejected by Test 3 (`<span class="sr-only">`
>   text or `aria-labelledby` reference could substitute without
>   changing what the user perceives).
> - L2: *"Button has `aria-label='Switch to dark mode'` when in
>   light mode."* ← rejected by Test 3 (same reason).
>
> Spec authors state at L0. If the user explicitly requests a lower
> layer for parity (e.g., "I want `aria-label` specifically because
> the shadcn ModeToggle uses it"), route the prose to
> `## Implementation hints` with the user's rationale, per the
> existing reject flow.

The L0/L1/L2 labels are illustrative for the worked example. They are
not introduced as a formal vocabulary the AI must use at evaluation
time — the rule's gatekeeping mechanism remains Test 3's
counterfactual substitution. The layering framing is an aid for spec
authors and reviewers to see *why* a lower-layer statement is mechanism-
shaped even when it sounds requirement-like.

## Mechanism 4 — Cross-skill contract

Closes root cause #4 (deference rule silently undermined).

### Forward reference (clarify → generate)

In `clarify-component/references/output-format.md`'s `## How` voice
rule section, after the existing enforcement-pointer sentence, one
sentence is added:

> This is the spec-side half of a two-part contract with
> `generate-component`'s *Pattern selection deference*
> ([`../../generate-component/references/output-format.md`](../../../skills/generate-component/references/output-format.md)
> §Pattern selection deference): clarify must not pre-commit at a
> layer where stack skills speak; generate must consult those skills
> at write time.

### Back reference (generate → clarify)

In `generate-component/references/output-format.md`'s `## Pattern
selection deference` section, after the existing prose, one sentence
is added:

> This rule's effectiveness depends on the spec leaving the
> mechanism unspecified at the layer where the skill speaks; that
> spec-side discipline is enforced by `clarify-component`'s `## How`
> voice rule
> ([`../../clarify-component/references/output-format.md`](../../../skills/clarify-component/references/output-format.md)
> §`## How` voice rule). A spec that pre-commits at a mechanism
> layer makes this deference rule a no-op for that mechanism.

The cross-references are minimal — one sentence each direction. The
goal is to surface the contract to a reader of either file, not to
duplicate one rule's prose in the other.

## Reference file changes

Concrete edits to support the design.

The structure of the rule body after the edits is:

1. Voice rule intro paragraph (unchanged).
2. Canonical question (unchanged).
3. Test 1 — Outcome test (unchanged).
4. Test 2 — Procedural test with sequence indicators (unchanged; see
   *Out of scope*).
5. Test 3 — Mechanism citation test, **replaced** with Mechanism 1's
   generative criterion.
6. Language-neutral application sentence (**new**, Mechanism 2).
7. Reject anchors block (**replaces** the current example list under
   Test 3; Mechanism 2).
8. Pass anchors block (**new**, Mechanism 2; absorbs the pass cases
   currently embedded in the "Library names..." paragraph — verbatim
   user request, domain-level outcomes, ARIA attribute values).
9. Reject-flow routing paragraph: "When a bullet trips any of the
   three tests, the AI returns to dialogue with a single rephrase
   question..." (**preserved**; only the pass-example sentences are
   removed because they have moved into Pass anchors above).
10. *Boundary: layered observability* paragraph (**new**, Mechanism
    3).
11. Enforcement-pointer paragraph (**preserved**).
12. Cross-reference to deference (**new**, Mechanism 4 forward
    reference).

- [`skills/clarify-component/references/dimensions.md`](../../../skills/clarify-component/references/dimensions.md):
  - Apply the structural changes above to lines 67–86 (the
    *Note on requirements-only voice for `## How`* block).

- [`skills/clarify-component/references/output-format.md`](../../../skills/clarify-component/references/output-format.md):
  - Apply the same structural changes to lines 144–167 (the
    `### `## How` voice rule` block).
  - **Cross-reference to deference** (item 12 above) is added only
    to this file. `dimensions.md` does not get the cross-reference
    sentence — its existing pointer to `output-format.md` covers the
    contract surface for `dimensions.md` readers.

- [`skills/generate-component/references/output-format.md`](../../../skills/generate-component/references/output-format.md):
  - **Cross-reference to voice rule.** Mechanism 4's back-reference
    sentence is appended to the `## Pattern selection deference`
    section (currently lines 76–80), after the existing prose.

- [`skills/clarify-component/SKILL.md`](../../../skills/clarify-component/SKILL.md):
  - **Unchanged.** Step 10 (Self-check) continues to apply the
    voice rule. Its structure is unchanged; only the rule it
    evaluates against is broadened by the changes above.

- `skills/setup/references/stack-skills.md` — **unchanged.** No
  manifest changes.

The rule prose is currently duplicated across `dimensions.md` and
`output-format.md`. This design preserves that duplication (both
files must be edited identically) for symmetry with how the silent-
defaults design landed. Deduplicating the rule prose to a single
location is a candidate refactor noted under Follow-ups.

## Out of scope

- **Test 1 (Outcome) and Test 2 (Procedural) generative reframing.**
  Test 2 currently carries a closed list of sequence indicators
  ("then/after/next"), the same pattern this design removes from
  Test 3. The same generative reframing should apply to Test 2
  (a procedural pattern is a procedural pattern), but the issue's
  scope is limited to Test 3 and the diagnostic evidence in this
  design's Context is Test 3 only. Test 1/2 review is captured in
  the issue's Follow-ups as a sibling cleanup.

- **Retroactive migration of existing specs.** The mode-toggle spec
  produced during the original run still records the L1 `aria-label`
  prose in `## How → Accessibility`. This design does not migrate it.
  The expanded rule fires on *new* clarify runs only; the user may
  re-run `/gyre:clarify-component` against the same source text to
  produce a clean spec, or hand-edit. A bulk-migration tool is not
  built.

- **`web-design-guidelines` ↔ `building-components` divergence.**
  The Vercel Web Interface Guidelines prescribe `aria-label` for
  icon-only buttons; the building-components skill treats
  `aria-label` and `<span class="sr-only">` as equally valid. Even
  after this design lands and clarify passes the accessibility
  decision to stack skills, the stack-skill consultation may select
  `aria-label` again. That is the correct outcome. The point of this
  design is not to *change* which mechanism is emitted in the current
  setup; it is to put the decision in the right layer so changing
  skills later changes the emitted code without a spec round-trip.

- **Rule-prose deduplication.** Whether `dimensions.md` and
  `output-format.md` should continue to carry parallel copies of the
  voice rule prose is an architectural question with implications
  beyond this design. Noted as a Follow-up.

## Verification

Re-run `/gyre:clarify-component` against the same `mode-toggle`
source text used in the issue:

```
Build a mode-toggle component that switches between dark mode and light mode.
```

### Scenario A — accessibility L0 is produced

Expected: the `## How → Accessibility` section of the generated spec
contains a bullet phrased at L0, not L1:

- Pass: *"Button exposes an accessible name that, when read aloud,
  conveys the next action available given the current theme."*
- Reject (Test 3 fires): *"`aria-label` describes the action
  available given the current theme."*

If the AI authors an L1 bullet during dialogue, the Step 10 Self-check
rejects it. Dialogue resumes with the existing rephrase question. The
user's response routes to one of two outcomes per the existing flow:
rewrite as L0, or move to `## Implementation hints`.

### Scenario B — categorical regression checks

The expanded rule should produce equivalent behavior across the new
categories. For each, a candidate spec bullet at the mechanism layer
is rejected and rephrased at the higher layer:

- CSS utility class (`sr-only`): "the action label is announced by
  assistive technology but is not visually rendered as flowing text"
  replaces "uses the `sr-only` class on the action label".
- CSS technique (`prefers-reduced-motion`): "users with the reduced-
  motion preference receive a presentation without animated
  transitions" replaces "respects `prefers-reduced-motion`".
- HTML element selection (`<div role="button">`): "the control
  exposes button role to assistive technology and is reachable by
  keyboard tab order" replaces "renders as `<div role='button'>`".
- Web-platform feature (`matchMedia`): "the component reacts to
  changes in the system color-scheme preference" replaces "uses
  `matchMedia` to observe color-scheme changes".

In each case, `generate-component`, with the relevant stack skills
loaded, then selects the concrete mechanism per skill prescription.

### Scenario C — false-positive guard intact

The expanded rule must continue to pass legitimate ARIA value
citations. Candidate bullets that name `aria-expanded='true'`,
`aria-current='page'`, or `aria-pressed='false'` describe the
observable contract — they are not mechanism citations because no
substitution preserves the observable outcome. Step 10 Self-check
must not fire on these.

### Scenario D — implementation hint escape hatch

In Scenario A, if the user responds to the rephrase question with
*"I want `aria-label` specifically because the shadcn ModeToggle
uses it"*, the existing escape hatch routes the prose to
`## Implementation hints`:

```
- Use `aria-label` for the accessible name: matches the pattern used by the shadcn ModeToggle.
```

The L0 requirement is recorded in `## How → Accessibility`; the
mechanism preference is recorded as a hint. `generate-component`
follows the hint by default; a stack skill prescribing a different
mechanism produces an `attention` row on the Plan per the existing
hint-divergence rule.

### Scenario E — anchor-free counterfactual application

Re-run the same source text with the AI generating candidate
phrasings that do not appear in the anchor lists, e.g., "Use the
`<button>` tag to expose an accessible name" or "Use the `sr-only`
class to hide the visual indicator." These must trip Test 3 by the
counterfactual test (`<button>` could be substituted with
`<a role="button">`; `sr-only` could be substituted with
`aria-label`), even though neither phrase appears in the anchor
lists. The anchors are not consulted by surface match; the criterion
is consulted by counterfactual substitution.

## Follow-ups

- **Test 1 / Test 2 generative reframing.** Capture as a sibling
  issue. Test 2's procedural sequence indicators carry the same
  closed-list shape Test 3's mechanism citations did; the
  generative reframing applied here should apply there too.

- **Calibration of the counterfactual test.** If post-landing runs
  surface false positives (legitimate observable bullets the AI
  rejects because it imagined a non-equivalent "alternative") or
  false negatives (mechanism bullets that pass because the AI failed
  to imagine an alternative), log them under `docs/issues/` for
  rule-prose calibration. The calibration target is the criterion's
  phrasing, not a return to example-list gatekeeping.

- **Rule-prose deduplication.** The voice rule body is duplicated
  across `dimensions.md` and `output-format.md`. Whether to move it
  to a single canonical location and have the other file refer to it
  is an architectural question worth considering. This design
  preserves the duplication for change-locality.

- **Hint divergence telemetry.** Scenarios A–D will produce
  `attention` rows when stack skills disagree with user-stated
  hints. Tracking the frequency informs future calibration of which
  mechanisms warrant a hint vs which warrant a spec-level
  requirement.

## Related

- Open issue this design closes:
  [`docs/issues/2026-05-26-clarify-component-voice-rule-mechanism-scope-gap.md`](../../issues/2026-05-26-clarify-component-voice-rule-mechanism-scope-gap.md).
- Sibling issue (JS-runtime subset, already closed):
  [`docs/issues/2026-05-25-clarify-component-implementation-leaks-into-spec.md`](../../issues/2026-05-25-clarify-component-implementation-leaks-into-spec.md).
- Prior design that introduced the voice rule (this design extends
  its scope):
  [`docs/superpowers/specs/2026-05-25-clarify-component-silent-defaults-design.md`](2026-05-25-clarify-component-silent-defaults-design.md).
- Skill bodies touched by this design:
  - [`skills/clarify-component/references/dimensions.md`](../../../skills/clarify-component/references/dimensions.md)
  - [`skills/clarify-component/references/output-format.md`](../../../skills/clarify-component/references/output-format.md)
  - [`skills/generate-component/references/output-format.md`](../../../skills/generate-component/references/output-format.md)
- Self-check enforcement point (structure unchanged):
  [`skills/clarify-component/SKILL.md`](../../../skills/clarify-component/SKILL.md)
  Step 10.
- Deference rule the voice rule contracts with:
  [`skills/generate-component/references/output-format.md`](../../../skills/generate-component/references/output-format.md)
  §Pattern selection deference.
- Stack skills manifest (intentionally unchanged):
  [`skills/setup/references/stack-skills.md`](../../../skills/setup/references/stack-skills.md).
- Reference shadcn canonical for the mode-toggle example:
  https://ui.shadcn.com/docs/dark-mode/next.
