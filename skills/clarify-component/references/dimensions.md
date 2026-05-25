# Dimensions

The seven axes of clarification that the dialogue covers. When a dimension is active, dimensions are visited in this order:
**Where → Interaction model → What → Look → How → Data → Non-goals**.

The ordering is deliberate. *Where* comes first because a positive answer ("an existing component already covers this") makes the rest unnecessary, terminating dialogue early.

All questions in this file are written in English as the source language. At runtime the AI must deliver each question in the user's language, inferred from the user's first message.

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

**Note on requirements-only voice for `## How`.** The three sub-sections of *How* that describe observable behavior (`## How → Interactions`, `## How → Edge cases`, `## How → Accessibility`) follow a requirements-only voice. The dialogue rejects bullets that name library/API symbols the AI authored (e.g., `useEffect`, `next-themes`), express procedural steps ("...then ..."), or cite named implementation idioms ("mount-guard pattern"). When a bullet trips the rule, the AI returns to dialogue with a single rephrase question and routes the user's answer to one of two outcomes: rewrite as the underlying requirement, or move the prose to the optional `## Implementation hints` section in the spec (see `output-format.md`) with the user's stated rationale. Enforcement happens at the *Self-check* step (Step 10 in `SKILL.md`).

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

Before asking the questions below, scan the request text for category words that name common UI primitives. A non-exhaustive list: *button, card, input, modal, dialog, tab, badge, toggle, switch, checkbox, radio, select, dropdown, tooltip, menu, list, table, avatar, banner, alert, breadcrumb, pagination, popover*. The user's language may differ; the skill must recognize the user's language's equivalents (e.g., Korean: 버튼/카드/입력/모달/탭/뱃지/토글/스위치/체크박스/라디오/셀렉트/드롭다운/툴팁/메뉴/리스트/테이블/아바타/배너/얼럿/페이지네이션/팝오버). If multiple are present, the most specific category (not a modifier) drives Q1b. The extracted category word is referenced in Q1b's phrasing; if no category word is found, Q1b is asked generically.

**Question sequence:**

1. **Q1a — Same-role candidate.** "Is there already a component in this project that covers the same use case end-to-end — i.e., does what this request asks for, as a single unit? If so, what is its name?"
2. **Q1b — Base primitive.** Phrasing depends on whether a category word was extracted:
   - Found (e.g., "button"): "Your request mentions 'button'. Is there already a base Button primitive in this project that this component should be built on top of? If so, what is its name?"
   - Not found: "Are there base UI primitives in this project (e.g., Button, Card, Input) that this component should be built on top of? If so, which one(s)?"
3. **Q2 — Same-role disposition.** Asked only if Q1a yielded a name: "Should it be reused as is, or extended for this request?"

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

The user's explicit choice is recorded in the spec's `## Interaction model` section per `output-format.md`. The chosen variant calibrates subsequent dimensions (*What* props surface, *How* states and interactions).

---

## What

**Captures:** the component's purpose, the scenarios it is used in, and the props (API) it exposes.

**Importance:** the dimension whose absence most reliably produces the wrong artifact. Two visually identical components can have entirely different stability requirements and exposed props depending on use case.

**Question sequence:**

1. Purpose: "What is this component's primary purpose? Please describe it in one sentence."
2. Scenarios: "In which screens or situations is this component used? List every known use site."
3. Naming: "What name should this component have? It will be used both as the spec filename (in kebab-case) and as the component's identifying name." If the user is unsure, propose a name derived from the purpose and use scenarios, and ask the user to accept or amend it.
4. Props: "What props does this component expose to its consumers? For each prop, please specify the name, type, whether it is required, its default value, and a short description."
   - If the user has no concrete prop list in mind, propose a default set as a multiple-choice list derived from the purpose and use scenarios, and ask the user to accept, amend, or reject each.

---

## Look (conditional)

**Captures:** visual hierarchy, spacing, responsive behavior, and the way the component aligns with the project's design tokens.

**Activation gate (a single meta question, asked at the start of this dimension):**

> "Does this project already define a design guide or token system? For example, a `DESIGN.md` file or a documented token set."

Based on the answer:

- **Defined and substantive** → compressed.
- **Absent or sparse** → full.

### Compressed path

Single question:

> "Are there any design tokens that need to be overridden for this component? If so, which tokens and to what values?"

If the user answers "none," the dimension's body in the output spec is a single bullet: `Token overrides: none`.

### Full path

Question sequence:

1. Visual hierarchy: "Within the surrounding UI, what visual weight should this component carry? Is it a primary, secondary, or auxiliary element?"
2. Spacing: "What internal padding and external margin conventions should apply?"
3. Responsive behavior: "At which breakpoints should the component change its behavior or appearance, and how?"
4. Token suggestions: "Are there any new design tokens this component might introduce (for example, a new spacing scale)? If so, list them."

---

## How

**Captures:** state model, interactions, accessibility, and UX edge cases.

**Importance:** the dimension the AI is most prone to silently fill with wrong defaults. Making it an explicit checklist closes that gap.

**Question sequence:**

1. States: "What states can this component be in? For example: default, hover, focus, disabled, loading, error, empty."
2. Interactions: "How does the user interact with this component? Specify hover, focus, keyboard, transitions, and any custom behaviors."
3. Accessibility: "What accessibility requirements apply? Specify semantic markup, ARIA, focus order, and contrast."
4. Edge cases: "How should the component handle UX edge cases such as very long text, RTL languages, and i18n?"

If a question's expected answer set is bounded, ask it as a multiple-choice question and present the choices.

---

## Data / Content (conditional)

**Captures:** the actual data schema the component is bound to, and edge data.

**Activation:** governed by risk signals (see `risk-signals.md`). If any risk axis is high, this dimension is at full depth. Otherwise it is compressed.

### Compressed path

Single question:

> "What is the shape of the data this component renders? A one-sentence description is sufficient."

### Full path

Question sequence:

1. Schema: "What is the schema of the data this component binds to? For each field, specify the type and whether it is nullable."
2. Edge data: "What extreme cases occur in real data? For example: very short and very long values, truncation indicators such as `999+`, the difference between optimistic and confirmed states, and page sizes."

---

## Non-goals

**Captures:** things this component will *not* do — bounded scope, refused responsibilities.

**Importance:** the most undervalued dimension. The single line "this Button does not contain a dropdown" is a more effective constraint than five lines of "this Button behaves like X."

**Minimum:** at least one non-goal must be recorded.

**Question template:**

> "Please name at least one thing this component will *not* do. For example: 'this Button does not contain a dropdown,' or 'this Card is not clickable.'"

This dimension is enumerative; it may be asked as a single batched prompt regardless of the dialogue mode rules.
