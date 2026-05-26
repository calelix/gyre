---
created: 2026-05-26
---

# Clarify-Component: AskUserQuestion Delivery

## Context

The clarify-component dialogue currently presents questions to the user as plain text in chat. When a question has bounded choices, the choices appear as a Markdown bulleted list in the question body, and the user types their selection. The user observed that this is inferior to the AskUserQuestion tool's structured selection UI (the same UI superpowers skills use during brainstorming), which:

- Pops a dedicated selection panel with each option as a button
- Renders option labels and descriptions separately for readability
- Adds an automatic "Other" choice for free-form custom input
- Removes ambiguity about what counts as a "valid" answer

This design wires AskUserQuestion in as clarify-component's default question-delivery mechanism, replacing the bulleted-list-in-prose pattern across every dimension.

## Principle

> **Every clarify-component question is delivered via the AskUserQuestion tool. The AI generates 2–4 candidate answers from the accumulated dialogue context, presents them as options, and lets the tool's automatic "Other" choice carry any free-form custom input the user types.**

The principle applies uniformly. There is no opt-out for "open-ended" questions — the AI is responsible for surfacing 2–4 plausible defaults even for questions like *Purpose* or *Naming*, where prior practice asked the user to free-type. The "Other" fallback preserves the user's ability to type anything.

This is a delivery-mechanism change, not a behavior change: the dimensions, their activation policies, the voice rule on `## How`, and the sequential discipline ("one question at a time") are all unchanged. Only the surface through which questions reach the user changes.

### Relation to silent-defaults

The silent-defaults design's principle ([2026-05-25-clarify-component-silent-defaults-design.md](2026-05-25-clarify-component-silent-defaults-design.md)) listed four permitted spec-item shapes. Shape (2) is "an explicit default proposal the user accepted." Under this design, *every* presented option is a shape (2) proposal — the user's selection of an option is an explicit acceptance of that proposal. The "Other" responses are shape (1) (the user's explicit answer). Shape (3) ("AI-derived requirement") becomes relevant only when the AI cannot generate ≥ 2 candidates and falls back to plain text (rare; see Mechanism §Fallback below). Shape (4) (translation) applies to whatever was selected/typed, translated into English at spec write time.

## Mechanism

### Per-dimension delivery

| Dimension question | AskUserQuestion shape | multiSelect | Candidate source |
|---|---|---|---|
| Where Q1a (same-role candidate) | 2 options: "Yes — enter name" → Other / "No" | false | none (always 2 fixed) |
| Where Q1b (base primitive) | 2–4 options: 1–3 AI-derived candidate primitives (from category word match) + "None" | true | extracted category word + user's request text |
| Where Q2 (reuse vs extend) | 2 fixed options | false | none |
| Interaction model | 2–4 options: AI-identified variants for the requested pattern | false | pattern name + general AI knowledge |
| What → Purpose | 2–3 AI-derived candidate purpose statements | false | original request text |
| What → Usage scenarios | 2–4 AI-derived candidate scenarios | true | purpose answer + request text |
| What → Naming | 2–3 AI-derived kebab-name candidates | false | purpose answer |
| What → Props | 3 options: "Accept AI-proposed props set" / "Customize (per-prop questions)" / "No props"; Customize routes to a per-prop AskUserQuestion loop | false | purpose + scenarios |
| Look activation gate | 2 fixed options (defined / absent) | false | none |
| Look compressed (tokens) | 2–4 AI-derived override candidates + "None" | true | request text + active dimension answers |
| Look full (visual hierarchy / spacing / responsive / token suggestions) | per sub-question, 2–3 AI candidates | false | request + purpose |
| How → States | multiSelect from the standard set (default, hover, focus, disabled, loading, error, empty) winnowed to ≤ 4 most applicable for the pattern | true | pattern + interactions |
| How → Interactions | 2–4 AI-derived candidate behaviors (must pass voice rule per [2026-05-26-code-form-quality-gates-design.md](2026-05-26-code-form-quality-gates-design.md) Mechanism 2) | true | states + pattern |
| How → Accessibility | 2–4 AI-derived candidate requirements (semantic markup, ARIA, focus order, contrast) | true | pattern + states |
| How → Edge cases | 2–4 AI-derived candidate edge cases (long text, RTL, i18n, etc.) | true | scenarios + pattern |
| Data compressed (schema sentence) | 2 options: "<AI-inferred schema>" + Other | false | usage scenarios |
| Data full path (schema / edge data) | per sub-question, 2–3 candidates | false | usage scenarios |
| Non-goals | 2–4 AI-proposed non-goals; minimum 1 selection | true | accumulated dialogue |

### Language

Question text, option labels, and option descriptions are in the user's language, inferred from the user's first message. This is the standard user-facing dialogue language posture (per [2026-05-26-code-form-quality-gates-design.md](2026-05-26-code-form-quality-gates-design.md) Overarching language policy).

User responses (whether selected option text or "Other" free input) are translated to English at spec write time per shape (4) of the silent-defaults principle.

### Sequential discipline

The "Sequential by default" rule in [skills/clarify-component/SKILL.md](../../../skills/clarify-component/SKILL.md) `## Dialogue mode` is preserved. One AskUserQuestion call per dialogue turn, asking one dimension's one question. The user's answer is absorbed before the next question is composed.

The AskUserQuestion tool supports 1–4 questions per call, but clarify-component uses one per call to preserve the dimension activation policy's calibration on incremental answers. The single exception is *Non-goals*, which (per the existing dimensions.md accommodation) may be asked in one batched prompt — under this design, that batched prompt is a single multiSelect AskUserQuestion call.

### Fallback to plain text

When the AI cannot generate ≥ 2 distinct candidates for a dimension question (e.g., the request is so generic that no plausible default emerges), the AI falls back to a plain text question with no options. This is the same plain text dialogue clarify-component currently uses everywhere.

The fallback is expected to be rare. If it occurs, no error is raised — the AI just asks the question as text. The user's typed answer is treated as shape (1) (explicit answer) and translated to English at spec write time as usual.

### Candidate generation policy

The AI generates 2–4 candidates per question by:

1. Reading the accumulated dialogue context (original request text + every prior dimension answer).
2. Applying the *Candidate source* indicated in the per-dimension table above as the primary input weight.
3. Drawing on general AI knowledge of the requested component class (e.g., for mode-toggle, knowledge of binary toggle / cycle button / 3-way dropdown).
4. Pruning to the top 2–4 candidates by plausibility for the current context.

Quality of generated candidates is not measured by this design. The "Other" fallback covers the worst case. If candidate quality proves systematically poor in practice, calibration is a follow-up.

## Reference file changes

Two files need edits.

### `skills/clarify-component/SKILL.md`

**`## Dialogue mode` section** — replace the "Three accommodations" intro and accommodation 3 to describe AskUserQuestion delivery as the default mechanism. The new shape:

- **Sequential by default.** (Unchanged.)
- **Accommodation 1 — Absorbing answers richer than asked.** (Unchanged.)
- **Accommodation 2 — Enumerative dimensions may be batched.** (Unchanged in intent; *Non-goals* batches as a single multiSelect AskUserQuestion call.)
- **Accommodation 3 — AskUserQuestion as default delivery.** Replaces the current accommodation 3 ("Multiple choice preferred when answers are bounded"). New body:

  > Every question is delivered via the AskUserQuestion tool. The AI generates 2–4 candidate answers from the accumulated dialogue context and presents them as options; the tool's automatic "Other" choice carries any free-form custom input. Option labels and descriptions are in the user's language. When the AI cannot generate ≥ 2 distinct candidates for a question, it falls back to a plain text prompt. See `references/dimensions.md` *Delivery mechanism* for the per-dimension shape (option count, multiSelect, candidate source).

### `skills/clarify-component/references/dimensions.md`

**New top-level section `## Delivery mechanism`** — inserted between the existing introductory prose (the "All questions in this file are written in English…" paragraph block) and the `## Activation matrix`. The section's body matches the **Per-dimension delivery** table from this design's Mechanism section plus the **Candidate generation policy** paragraph.

**Per-dimension question template additions** — each dimension's *Question sequence* / *Question template* / *Full path* / *Compressed path* sub-section gets a one-line *Delivery* annotation reflecting the table row. Examples:

- Where Q1a → `Delivery: 2 options ("Yes — enter name" → Other / "No"); if Yes, follow-up AskUserQuestion for the candidate's name with AI-derived candidates from category word match + Other.`
- Interaction model → `Delivery: AskUserQuestion with 2–4 options (AI-identified variants for the requested pattern).`
- What → Props → `Delivery: 3 options ("Accept AI proposal" / "Customize" / "None"); Customize routes to a per-prop AskUserQuestion loop with each prop's defaults as candidate.`
- Non-goals → `Delivery: AskUserQuestion with multiSelect, 2–4 AI-proposed non-goals; minimum 1 selection.`

The existing "Multiple choice preferred when answers are bounded" sentence in the *Dialogue mode* section of SKILL.md is removed (subsumed by the new accommodation 3 and the *Delivery mechanism* section).

## Out of scope

- **generate-component's Plan approval dialogue.** That dialogue currently happens in plain text. Adapting it to AskUserQuestion is a separate slice, not covered here.
- **AskUserQuestion's `preview` feature.** clarify-component's options are textual; visual previews are not needed. If a future dimension involves visual choices (e.g., layout variants with mockups), the preview feature can be adopted incrementally.
- **Candidate generation quality calibration.** Initial release relies on AI general knowledge for candidate plausibility. If users routinely choose "Other" instead of provided candidates, that signals the candidate generation needs work — to be addressed by a follow-up issue.
- **Per-prop AskUserQuestion loop format.** The Customize route in *What → Props* triggers a sub-dialogue per prop. The exact per-prop question shape (asking for name, type, required, default, description in one batched AskUserQuestion call vs sequential) is left to implementer discretion within the AskUserQuestion 1–4 questions per call limit. If the per-prop format proves clunky, calibration is a follow-up.
- **Fallback frequency telemetry.** This design assumes the plain-text fallback is rare. No measurement is built. If anecdotal evidence suggests fallback fires often, file an issue.

## Verification

Manual end-to-end verification in a host project. Two scenarios cover the design.

### Scenario A — AskUserQuestion fires across every dimension

Run `/gyre:clarify-component "toggle button that switches the theme"`. Step through each active dimension's question. For every question, expect:

- A selection panel to appear with 2–4 labeled options.
- An "Other" choice automatically added by the tool.
- Selection of an option produces no follow-up question for that dimension (unless the dimension has a multi-question sub-sequence like What → Props' Customize route).

If any dimension instead surfaces a plain text question, two possibilities:

1. The AI legitimately could not generate ≥ 2 candidates — acceptable per fallback policy.
2. The dimension's *Delivery* annotation was missed or the AI did not honor it — log an issue.

### Scenario B — "Other" path produces correct spec content

In the same run, on at least one question, choose "Other" and type a non-English custom answer that none of the proposed options covered. After spec write:

- The spec section corresponding to that question records the custom answer translated to English (per shape 4 of the silent-defaults principle and the overarching language policy).
- The translation preserves the user's intent.

If the spec section records the user-language text verbatim, the language policy enforcement at clarify write time has regressed — file an issue.

## Follow-ups

- **Candidate generation quality.** Track which dimensions most often produce "Other" responses (anecdotally; no measurement built). If a dimension's candidates are systematically rejected, refine its *Candidate source* in dimensions.md.
- **Per-prop loop ergonomics.** The Customize route in *What → Props* may need its own design pass after first use. Open an issue if the per-prop loop feels heavy.
- **Plain text fallback frequency.** If fallbacks fire often, the AI's candidate-generation prompting needs tuning. The dimensions' *Candidate source* notes may need to widen the input context the AI considers.
- **Apply same delivery model to generate-component Plan approval.** Once the clarify-side AskUserQuestion delivery proves out in practice, the same model can be applied to generate-component's Plan approval dialogue (currently plain text). Tracked as a separate future slice.

## Related

- Skill files touched by this design:
  - [`skills/clarify-component/SKILL.md`](../../../skills/clarify-component/SKILL.md) (`## Dialogue mode`)
  - [`skills/clarify-component/references/dimensions.md`](../../../skills/clarify-component/references/dimensions.md) (`## Delivery mechanism` new section + per-dimension *Delivery* annotations)
- Prior designs unaffected but referenced:
  - [`2026-05-25-clarify-component-silent-defaults-design.md`](2026-05-25-clarify-component-silent-defaults-design.md) — silent-defaults principle; this design refines how its shapes (1)/(2)/(3)/(4) map onto AskUserQuestion's option/Other surface.
  - [`2026-05-26-code-form-quality-gates-design.md`](2026-05-26-code-form-quality-gates-design.md) — overarching language policy + shape (4) translation; this design composes with both.
- AskUserQuestion tool reference: the tool is exposed by Claude Code; see the tool's schema for parameters (`questions[]` with `question` / `header` / `options[]` / `multiSelect`; each `option` has `label` / `description`). Only the 2–4 options per question constraint and the 1–4 questions per call constraint are load-bearing for this design.
