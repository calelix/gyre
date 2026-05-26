# Dimensions

The seven axes of clarification that the dialogue covers. When a dimension is active, dimensions are visited in this order:
**Where → Interaction model → What → Look → How → Data → Non-goals**.

The ordering is deliberate. *Where* comes first because a positive answer ("an existing component already covers this") makes the rest unnecessary, terminating dialogue early.

All questions in this file are written in English as the source language. At runtime the AI must deliver each question in the user's language, inferred from the user's first message.

## Delivery mechanism

Every question in this file is delivered to the user via the AskUserQuestion tool. For each question the AI generates 2–4 candidate answers from the context accumulated so far (the original request text and prior dimension answers), presents them as options with a short description per option, and lets the tool's automatic "Other" choice carry any free-form custom input. The user picks one option (or types in "Other"); the AI absorbs the answer and proceeds.

**Option count.** Minimum 2, maximum 4 (tool constraint). If the AI cannot generate 2 distinct candidates the question falls back to a plain text prompt — this should be rare.

**multiSelect.** Used when the question's answer is a list (the `## How → States` sub-question, the *Non-goals* dimension, etc.). Indicated per-dimension in the table below and in the dimension's *Delivery* annotation.

**Candidate generation.** The AI uses the original request text plus every prior dimension answer as context. The *Candidate source* column below indicates the primary input each dimension should weight most heavily.

**Language.** Question text, option labels, and option descriptions are in the user's language. "Other" responses (typed in the user's language) are translated to English at spec write time per the overarching language policy in [`../SKILL.md`](../SKILL.md) and the *Spec language rule* in [`output-format.md`](output-format.md).

**Sequential discipline.** The "Sequential by default" rule (SKILL.md `## Dialogue mode`) is preserved. One AskUserQuestion call per dialogue turn, asking one dimension's one question. The exception is *Non-goals*, which is asked as a single multiSelect AskUserQuestion call (per the existing accommodation in `## Dialogue mode`).

### Per-dimension delivery table

| Dimension question | AskUserQuestion shape | multiSelect | Candidate source |
|---|---|---|---|
| Where Q1a (same-role candidate) | 2 options: "Yes — and I'll name it" → Other / "No" | false | none (always 2 fixed) |
| Where Q1b (base primitive) | 2–4 options: 1–3 AI-derived candidate primitives (from category word match) + "None" | true | extracted category word + request text |
| Where Q2 (reuse vs extend) | 2 fixed options | false | none |
| Interaction model | 2–4 options: AI-identified variants for the requested pattern | false | pattern name + general AI knowledge |
| What → Purpose | 2–3 AI-derived candidate purpose statements | false | original request text |
| What → Usage scenarios | 2–4 AI-derived candidate scenarios | true | purpose answer + request text |
| What → Naming | 2–3 AI-derived kebab-name candidates | false | purpose answer |
| What → Props | 3 options: "Accept proposed props set" / "Customize (per-prop loop)" / "No props"; Customize routes to a per-prop AskUserQuestion loop | false | purpose + scenarios |
| Look activation gate | 2 fixed options (defined / absent) | false | none |
| Look compressed (tokens) | 2–4 AI-derived override candidates + "None" | true | request text + active dimension answers |
| Look full path (4 sub-questions) | per sub-question, 2–3 AI candidates | false | request + purpose |
| How → States | multiSelect over the standard candidate set winnowed to ≤ 4 most applicable | true | pattern + interactions |
| How → Interactions | 2–4 AI-derived candidate behaviors (must pass voice rule below) | true | states + pattern |
| How → Accessibility | 2–4 AI-derived candidate requirements (semantic markup, ARIA, focus order, contrast) | true | pattern + states |
| How → Edge cases | 2–4 AI-derived candidate edge cases (long text, RTL, i18n, etc.) | true | scenarios + pattern |
| Data compressed (schema sentence) | 2 options: "<AI-inferred schema>" + Other | false | usage scenarios |
| Data full path (schema / edge data) | per sub-question, 2–3 candidates | false | usage scenarios |
| Non-goals | 2–4 AI-proposed non-goals; minimum 1 selection | true | accumulated dialogue |

Each dimension's *Question sequence* below carries a one-line *Delivery* annotation reflecting its row in this table.

## Activation matrix

| Dimension | Always active | Full condition | Compressed condition |
|---|---|---|---|
| Where | Yes | — | — |
| Interaction model | No | Requested pattern matches a class of UI with multiple legitimate interaction variants | Compose decision's Reference primitive fixes the variant |
| What | Yes | — | — |
| Look | No | Design guide absent or sparse | Design guide defined and substantive |
| How | Yes | — | — |
| Data / Content | No | Any risk axis is high | All risk axes low + presentational role |
| Non-goals | Yes (≥ 1 entry) | — | — |

Compressed does not mean skipped. A compressed dimension still appears in the output spec, rendered as a single line `(compressed: <reason>)` under its heading, so consumers can locate every dimension by name.

**Note on activation mechanism.** *Look*'s activation is decided by asking the user a meta question at the start of the dimension (see §Look). *Data / Content*'s activation is decided silently from the internal risk signals (see `risk-signals.md`) — no user question is involved in the activation decision itself. This is the only structural difference between the two conditional dimensions and must not be conflated.

**Note on composition-aware framing.** When the *Where* decision is `Extend` or `Compose`, the downstream dimensions (especially *Look*, *How*, and *Data / Content*) should distinguish what the referenced primitive already provides from what this component *adds*. Capture the added aspects, and explicitly note inherited aspects as inherited (e.g., "states inherit from Button"). Do not redescribe the primitive's responsibilities as if writing them from scratch — that creates spec/implementation drift.

**Note on requirements-only voice for `## How`.** The three sub-sections of *How* that describe observable behavior (`## How → Interactions`, `## How → Edge cases`, `## How → Accessibility`) follow a requirements-only voice. The rule is governed by a canonical semantic question and three tests; example sets in any language are anchors, not a complete vocabulary.

**Canonical question:** Does this bullet describe an *observable property* of the component (a requirement), or *a means by which the component achieves something* (an implementation)?

**Test 1 — Outcome test.** Can a black-box observer of the rendered component verify this bullet is true by looking at observable output alone (DOM state, ARIA attributes, visual presence, interaction response)? If yes, the bullet is a requirement and passes. If the bullet describes internal mechanism (state held inside the component, lifecycle moments, internal data flow), it is rejected.

**Test 2 — Procedural test.** Does the bullet describe a sequence of actions? Reject. Sequence indicators include but are not limited to:

- English: "then", "after", "first ... after which", "next".

Sequences imply construction.

**Test 3 — Mechanism citation test.** Could a future stack-skill prescription substitute a different named technique for the same higher-layer user-observable contract without changing what the user perceives? If yes, the bullet has cited a mechanism and must be restated at the higher layer.

Apply the test by attempting the substitution: name an alternative technique that would satisfy the same observable outcome. If you can name one (the spec author can substitute `<span class="sr-only">` text for `aria-label`, `<button>` for `<div role="button">`, `fetch` for `EventSource`), the bullet is mechanism-shaped and rejected. If no substitution preserves the observable outcome (`aria-expanded='true'` IS the observable contract — there is no alternative — and "the dropdown opens on click" IS the user-perceived behavior), the bullet passes.

The criterion applies to AI-authored prose in any language. Apply the counterfactual test to the candidate regardless of the language it was generated in. The anchors below are written in English (the spec's source language), illustrating the categorical *shape* of mechanism citations — not a closed vocabulary, and not a language-restricted vocabulary.

**Reject anchors** (illustrating categorical shapes the criterion rejects):

- JS/React runtime: `useEffect`, `useState`, `useContext`, `useRouter`, `mount-guard pattern`, `render prop`, `compound component`.
- ARIA attribute names (as opposed to ARIA values): `aria-label`, `aria-describedby`, `aria-labelledby`, `role="..."` selection.
- CSS utility class names: `sr-only`, `visually-hidden`, `dark:hidden`, `not-dark:`.
- HTML element selection among interactive controls: `<button>` vs `<a role="button">` vs `<div tabindex="0">`.
- CSS technique citations: `transform: scale()`, `transition-all`, `prefers-reduced-motion`, `:focus-visible`.
- Web-platform feature names: `matchMedia`, `IntersectionObserver`, `MutationObserver`, `localStorage`.
- Network primitive names: `fetch`, `XMLHttpRequest`, `EventSource`, `WebSocket`.
- Routing primitive names: `<Link>`, `useRouter().push`, `<a href>`, `redirect()`.

**Pass anchors** (illustrating shapes the criterion permits):

- Domain-level outcome statements: "the dropdown opens on click", "focus moves to the first item".
- ARIA attribute *values* describing observable state: `aria-expanded='true'`, `aria-current='page'`, `aria-pressed='false'`.
- User's verbatim request quoted back. The rule fires on AI-authored prose only.
- Observable constraints with thresholds: "contrast ratio ≥ 4.5:1", "control is reachable by keyboard tab order".

When a bullet trips any of the three tests, the AI returns to dialogue with a single rephrase question and routes the user's answer to one of two outcomes: rewrite as the underlying requirement, or move the prose to the optional `## Implementation hints` section in the spec (see `output-format.md`) with the user's stated rationale. Enforcement happens at the *Self-check* step (Step 10 in `SKILL.md`).

**Boundary: layered observability.** When a single user-observable contract admits multiple equally observable sub-statements at descending layers of specificity, state the requirement at the **highest layer where user intent is fully captured**. Lower-layer statements that name a particular technique among legitimate alternatives are mechanism citations under Test 3.

Worked example — mode-toggle accessibility:

- L0 (highest, user-observable): *"Button exposes an accessible name that, when read aloud, conveys the next action available given the current theme."*
- L1: *"Button has an `aria-label` attribute whose value conveys that action."* ← rejected by Test 3 (`<span class="sr-only">` text or `aria-labelledby` reference could substitute without changing what the user perceives).
- L2: *"Button has `aria-label='Switch to dark mode'` when in light mode."* ← rejected by Test 3 (same reason).

Spec authors state at L0. If the user explicitly requests a lower layer for parity (e.g., "I want `aria-label` specifically because the shadcn ModeToggle uses it"), route the prose to `## Implementation hints` with the user's rationale, per the existing reject flow.

---

## Where

**Captures:** whether an existing component should be reused, extended, or composed from, or whether a new component is needed.

**Decision concepts at a glance:**

| Decision | Same-role candidate | Base primitive | Meaning |
|---|---|---|---|
| Reuse | exists | — | Use the existing component as is; build nothing new. |
| Extend | exists | — | Take the existing component's identity and modify or augment it. |
| Compose | does not exist | exists | A new component identity, built by assembling existing primitives inside it. |
| New | does not exist | does not exist | A new component built from scratch. |

Two distinctions matter most:

- **Compose vs. New** — if the new component wraps or assembles a base primitive (Button, Card, Input, ...), record it as `Compose` with the primitive as `Reference`. Recording as `New` loses the dependency and risks downstream rebuild of a primitive that already exists.
- **Compose vs. Extend** — `Extend` continues the *identity* of an existing same-role component; `Compose` creates a *new identity* that uses existing primitives as parts. "ModeToggle uses Button as a part" is Compose, not Extend.

**Filled by:** asking the user directly. The skill does not scan the user's codebase. The skill *does* read the user's request text to identify category words that should drive a targeted follow-up question.

**Pre-question step — category word extraction (always run):**

Before asking the questions below, scan the request text for category words that name common UI primitives. A non-exhaustive list: *button, card, input, modal, dialog, tab, badge, toggle, switch, checkbox, radio, select, dropdown, tooltip, menu, list, table, avatar, banner, alert, breadcrumb, pagination, popover*. If multiple are present, the most specific category (not a modifier) drives Q1b. The extracted category word is referenced in Q1b's phrasing; if no category word is found, Q1b is asked generically.

**Question sequence:**

1. **Q1a — Same-role candidate.** "Is there already a component in this project that covers the same use case end-to-end — i.e., does what this request asks for, as a single unit? If so, what is its name?"
   - *Delivery:* AskUserQuestion with 2 options ("Yes — and I'll name it" / "No"). If Yes, a follow-up AskUserQuestion solicits the name (no AI candidates; the user types it in "Other" or accepts a category-word-derived guess).
2. **Q1b — Base primitive.** Phrasing depends on whether a category word was extracted:
   - Found (e.g., "button"): "Your request mentions 'button'. Is there already a base Button primitive in this project that this component should be built on top of? If so, what is its name?"
   - Not found: "Are there base UI primitives in this project (e.g., Button, Card, Input) that this component should be built on top of? If so, which one(s)?"
   - *Delivery:* AskUserQuestion with `multiSelect: true` and 2–4 options — 1–3 candidate primitives derived from category-word match (or general primitives if no category word) + "None".
3. **Q2 — Same-role disposition.** Asked only if Q1a yielded a name: "Should it be reused as is, or extended for this request?"
   - *Delivery:* AskUserQuestion with 2 fixed options (Reuse / Extend).

The order matters: Q1a → Q1b → (Q2 if applicable). A "yes" on Q1a does not skip Q1b; a component can both have a near-duplicate to extend AND compose from a base primitive, and both relationships should be recorded when they exist.

**Outcomes** (determined by the combination of answers):

- *Reuse* — Q1a yielded a name AND Q2 = "reused as is." Terminate dialogue early. Write the lightweight spec body (see `output-format.md` § Reuse path).
- *Extend* — Q1a yielded a name AND Q2 = "extended." Record the same-role identifier as `Reference`. Proceed to the next dimension.
- *Compose* — Q1a yielded no name AND Q1b yielded one or more base primitive names. Record those identifiers as `Reference`. Proceed to the next dimension.
- *New* — Q1a yielded no name AND Q1b yielded no name. No reference is recorded. Proceed to the next dimension.

If both Q1a (with "extended") and Q1b yield names, the Decision is **Extend** (Q1a dominates the identity axis); the composed primitive(s) are mentioned in `Rationale`. The `Reference` field carries the same-role identifier.

---

## Interaction model (conditional)

**Captures:** the specific interaction variant for components whose request name maps to a class of UI with more than one legitimate shape (e.g., `mode-toggle` → binary toggle vs cycle button vs 3-way dropdown; `combobox` → autocomplete-only vs selectable-list vs multi-select).

**Importance:** silently inferring one variant pins downstream decisions (props in *What*, states in *How*) to that variant. When the user is not asked, the AI's inference becomes a spec default the user never approved — the failure mode recorded in [`docs/issues/2026-05-25-clarify-component-interaction-model-variants.md`](../../../docs/issues/2026-05-25-clarify-component-interaction-model-variants.md).

**Activation:**

The dimension activates when the requested component matches a class of UI for which more than one legitimate interaction model exists in common practice. The AI uses its own knowledge for this judgment — no stack skill is invoked (clarify remains self-contained).

Examples that activate: `mode-toggle`, `combobox`, `command-palette`, `data-table`, `navigation-menu`, `date-picker`, `multi-select`, `autocomplete`.

Examples that do not activate: `primary-submit-button`, `card`, `avatar`, `badge`, `breadcrumb`.

If activation is uncertain, **default to activate**. The cost of an unnecessary question is one dialogue turn the user can dismiss; the cost of a silent miss is the original failure mode.

### Compressed path

When the *Where* decision is `Compose` and the Reference primitive itself fixes the variant (e.g., the user named `DropdownMenu` as the primitive — that already chooses the dropdown variant), the dimension compresses: no question is asked, and the spec records `(compressed: variant fixed by Reference primitive <name>)` followed by the implied variant name on a single line.

### Full path

Single question:

> "<pattern name> commonly has multiple interaction models. Which one should this component implement?
> - **<Variant 1 name>** — <one-line description> (e.g., used by <one real-world example, if known>)
> - **<Variant 2 name>** — <one-line description>
> - **<Variant 3 name>** — <one-line description>
>
> If none of the above matches what you want, describe the variant you have in mind."

*Delivery:* AskUserQuestion with 2–4 options corresponding to the AI-identified variants for the requested pattern. The "If none of the above matches" sentence is carried by the tool's automatic "Other" choice; the user types their alternative there.

The user's explicit choice is recorded in the spec's `## Interaction model` section per `output-format.md`. The chosen variant calibrates subsequent dimensions (*What* props surface, *How* states and interactions).

---

## What

**Captures:** the component's purpose, the scenarios it is used in, and the props (API) it exposes.

**Importance:** the dimension whose absence most reliably produces the wrong artifact. Two visually identical components can have entirely different stability requirements and exposed props depending on use case.

**Question sequence:**

1. Purpose: "What is this component's primary purpose? Please describe it in one sentence."
   - *Delivery:* AskUserQuestion with 2–3 AI-derived candidate purpose statements drawn from the original request text. "Other" carries any free-form rephrasing.
2. Scenarios: "In which screens or situations is this component used? List every known use site."
   - *Delivery:* AskUserQuestion with `multiSelect: true` and 2–4 AI-derived candidate scenarios. "Other" lets the user add a scenario the AI didn't propose.
3. Naming: "What name should this component have? It will be used both as the spec filename (in kebab-case) and as the component's identifying name."
   - *Delivery:* AskUserQuestion with 2–3 AI-derived kebab-name candidates (from the purpose answer). "Other" lets the user type a custom name.
4. Props: "What props does this component expose to its consumers? For each prop, please specify the name, type, whether it is required, its default value, and a short description."
   - *Delivery:* AskUserQuestion with 3 options ("Accept proposed prop set" / "Customize per-prop" / "No props"). If "Customize per-prop" is chosen, a follow-up per-prop loop asks (via AskUserQuestion, one prop at a time) whether to accept, amend, or remove each AI-proposed prop. The AI proposes the prop set from the purpose and scenarios answers.

---

## Look (conditional)

**Captures:** visual hierarchy, spacing, responsive behavior, and the way the component aligns with the project's design tokens.

**Activation gate (a single meta question, asked at the start of this dimension):**

> "Does this project already define a design guide or token system? For example, a `DESIGN.md` file or a documented token set."

*Delivery:* AskUserQuestion with 2 fixed options ("Defined and substantive" / "Absent or sparse").

Based on the answer:

- **Defined and substantive** → compressed.
- **Absent or sparse** → full.

### Compressed path

Single question:

> "Are there any design tokens that need to be overridden for this component? If so, which tokens and to what values?"

*Delivery:* AskUserQuestion with `multiSelect: true` and 2–4 AI-derived override candidates + "None". If "None" is selected, the spec body is `Token overrides: none`.

### Full path

Question sequence:

1. Visual hierarchy: "Within the surrounding UI, what visual weight should this component carry? Is it a primary, secondary, or auxiliary element?"
   - *Delivery:* AskUserQuestion with 3 fixed options (Primary / Secondary / Auxiliary).
2. Spacing: "What internal padding and external margin conventions should apply?"
   - *Delivery:* AskUserQuestion with 2–3 AI-derived spacing convention candidates.
3. Responsive behavior: "At which breakpoints should the component change its behavior or appearance, and how?"
   - *Delivery:* AskUserQuestion with 2–3 AI-derived responsive-behavior candidates (e.g., "no breakpoint change" / "stack at mobile" / "specific custom rule").
4. Token suggestions: "Are there any new design tokens this component might introduce (for example, a new spacing scale)? If so, list them."
   - *Delivery:* AskUserQuestion with `multiSelect: true` and 2–4 AI-derived token-suggestion candidates + "None".

---

## How

**Captures:** state model, interactions, accessibility, and UX edge cases.

**Importance:** the dimension the AI is most prone to silently fill with wrong defaults. Making it an explicit checklist closes that gap.

**Question sequence:**

1. States: "What states can this component be in? For example: default, hover, focus, disabled, loading, error, empty."
   - *Delivery:* AskUserQuestion with `multiSelect: true`. Candidates: the AI winnows the standard state vocabulary (default, hover, focus, disabled, loading, error, empty) to the ≤ 4 most applicable for the requested pattern, plus "Other".
2. Interactions: "How does the user interact with this component? Specify hover, focus, keyboard, transitions, and any custom behaviors."
   - *Delivery:* AskUserQuestion with `multiSelect: true` and 2–4 AI-derived candidate behaviors. Each candidate must pass the voice rule below (requirements-only voice — observable outcomes, not implementation mechanisms).
3. Accessibility: "What accessibility requirements apply? Specify semantic markup, ARIA, focus order, and contrast."
   - *Delivery:* AskUserQuestion with `multiSelect: true` and 2–4 AI-derived accessibility requirements (e.g., "semantic button element", "aria-label present", "keyboard-focusable", "contrast ≥ 4.5:1"). Each candidate must pass the voice rule.
4. Edge cases: "How should the component handle UX edge cases such as very long text, RTL languages, and i18n?"
   - *Delivery:* AskUserQuestion with `multiSelect: true` and 2–4 AI-derived edge cases (long text, RTL, i18n, empty state, etc.). Each candidate must pass the voice rule.

The voice rule below applies to candidates the AI proposes as much as to user-typed "Other" responses.

---

## Data / Content (conditional)

**Captures:** the actual data schema the component is bound to, and edge data.

**Activation:** governed by risk signals (see `risk-signals.md`). If any risk axis is high, this dimension is at full depth. Otherwise it is compressed.

### Compressed path

Single question:

> "What is the shape of the data this component renders? A one-sentence description is sufficient."

*Delivery:* AskUserQuestion with 2 options — the AI proposes one inferred schema sentence as option 1, plus "Other" automatically.

### Full path

Question sequence:

1. Schema: "What is the schema of the data this component binds to? For each field, specify the type and whether it is nullable."
   - *Delivery:* AskUserQuestion with 2–3 AI-derived candidate schema sketches drawn from the usage scenarios.
2. Edge data: "What extreme cases occur in real data? For example: very short and very long values, truncation indicators such as `999+`, the difference between optimistic and confirmed states, and page sizes."
   - *Delivery:* AskUserQuestion with `multiSelect: true` and 2–4 AI-derived edge-data candidates.

---

## Non-goals

**Captures:** things this component will *not* do — bounded scope, refused responsibilities.

**Importance:** the most undervalued dimension. The single line "this Button does not contain a dropdown" is a more effective constraint than five lines of "this Button behaves like X."

**Minimum:** at least one non-goal must be recorded.

**Question template:**

> "Please name at least one thing this component will *not* do. For example: 'this Button does not contain a dropdown,' or 'this Card is not clickable.'"

*Delivery:* AskUserQuestion with `multiSelect: true` and 2–4 AI-proposed non-goals derived from the accumulated dialogue (pattern, props, scenarios). The user must select at least one option (or type at least one in "Other") to satisfy the minimum.

This dimension is enumerative; it is asked as a single batched prompt — under the AskUserQuestion delivery rule, that batched prompt is a single multiSelect AskUserQuestion call.
