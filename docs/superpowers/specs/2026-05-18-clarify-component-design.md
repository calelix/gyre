---
created: 2026-05-18
topic: clarify-component
status: design
---

# Design — `/clarify-component` Skill

## Goal

A skill that turns an under-specified natural-language UI component request into a precise component specification through dialogue, and writes the result to a Markdown file. The skill resolves ambiguity by asking the user follow-up questions until the AI judges no ambiguity remains.

The skill is the first concrete realization of the AI Harness described in [`docs/foundation.md`](../../foundation.md) — specifically the front edge of the pipeline, before code generation. It directly addresses the heaviest obstacle identified in [`docs/obstacles.md`](../../obstacles.md): calibrating how far dialogue should close the intent-to-spec gap.

## Scope

In scope:

- Recognizing whether the user's natural-language request is a component, a page, or a design-guide request.
- For component requests: running a dimension-driven dialogue and producing one component specification Markdown file.

Out of scope (deferred to later work):

- Page and design-guide specifications. The classifier recognizes these but the skill terminates after notifying the user.
- Automatic codebase exploration of existing components, design tokens, helpers. The skill resolves all such context through dialogue with the user.
- Overriding the output path. The output path is fixed in this iteration.
- Persisting the internal risk assessment to the spec file.

## Invocation

```
/clarify-component "<natural-language request>"
```

- The natural-language request can be supplied as an argument. If omitted, the AI asks for it as the first message.
- The skill is intended to be callable from any project where the plugin is installed; nothing about its behavior assumes a particular host repository layout.

## Output

A single Markdown file at:

```
docs/gyre/specs/components/<kebab-name>.md
```

- `<kebab-name>` is derived from the component's identifying name, which is established during the **What** dimension of dialogue.
- The output path is not user-overridable in this iteration. The convention reserves sibling directories for future artifact types:
  - `docs/gyre/specs/pages/<kebab-name>.md`
  - `docs/gyre/specs/DESIGN.md`

## Skeleton Flow

The skill executes the following steps in order. Steps 4–8 are the **dimensions** described in the next section.

1. **Classify** — Determine whether the request describes a component, a page, or a design guide.
2. **Risk read** *(internal, not shown to the user)* — Estimate risk signals from the request text.
3. **Where** *(always active, always first)* — Ask whether a similar component already exists. If yes, record the reuse decision and skip the remaining dimensions (early termination).
4. **What** *(always active)*.
5. **Look** *(conditionally active — see policy below)*.
6. **How** *(always active)*.
7. **Data / Content** *(conditionally active — see policy below)*.
8. **Non-goals** *(always active)*.
9. **Self-check** — The AI evaluates whether ambiguity remains. If yes, return to dialogue.
10. **Write spec** — Produce the Markdown file described under *Output Format*.

Throughout the flow, the AI uses the **Sequential + three accommodations** dialogue mode described in *Dialogue Mode*.

## Dimensions

The dimensions are the axes of clarification the dialogue covers. They are ordered to maximize early reduction of unnecessary work — **Where** comes first because a positive answer ("there is already a usable component") makes the rest unnecessary.

### Where

What it captures: whether an existing component should be reused or extended, or whether a new component is needed.

How it is filled: by asking the user directly. The skill does not scan the user's codebase in this iteration. If the user names an existing component, that name is recorded verbatim.

Behavior on a reuse answer: write a lightweight spec (front matter + Summary + Where) and terminate.

### What

What it captures: the component's purpose, the scenarios it is used in, and the props (API) it exposes.

Importance: this is the dimension whose absence most reliably produces the wrong artifact. Two visually identical components can have entirely different stability requirements and exposed props depending on use case.

### Look (conditional)

What it captures: visual hierarchy, spacing, responsive behavior, and the way the component aligns with the project's design tokens.

Activation: a single meta question — *"Is a design guide or token system already defined in this project?"* — is asked at the start of this dimension. Based on the answer:

- **Defined and substantive** → compressed: a single question about whether token overrides are needed.
- **Absent or sparse** → full: visual hierarchy, spacing, responsive behavior, and token suggestions are all asked.

### How

What it captures: state model (loading / empty / error / disabled), interactions (hover / focus / keyboard / transitions), accessibility (semantic markup, ARIA, focus order, contrast), and UX edge cases (long text, RTL, i18n).

Importance: this is the dimension the AI is most prone to silently fill with wrong defaults. Making it an explicit checklist closes that gap.

### Data / Content (conditional)

What it captures: the actual data schema the component will be bound to, and edge data (nullable fields, very short and very long values, truncation indicators, page sizes, the difference between optimistic and confirmed states).

Activation: governed by risk signals (see *Dimension Activation Policy*). High risk on any axis activates this dimension at full depth; otherwise it is compressed to a single line capturing the schema shape.

Rationale: this is the dimension that most commonly separates a demo-quality artifact from a production-quality one. AI is fluent at fabricating plausible mock data and will design around it; real data shapes are what cause padding to break and empty spaces to look awkward in production.

### Non-goals

What it captures: things this component will *not* do — bounded scope, refused responsibilities.

Importance: the most undervalued dimension. The single line "this Button does not contain a dropdown" is a more effective constraint than five lines of "this Button behaves like X." Without explicit non-goals, the prop list inflates over time.

Minimum: at least one non-goal must be recorded.

## Dimension Activation Policy

| Dimension | Always active | Full activation condition | Compressed activation condition |
|---|---|---|---|
| Where | Yes | — | — |
| What | Yes | — | — |
| Look | No | Design guide absent or sparse | Design guide defined and substantive |
| How | Yes | — | — |
| Data / Content | No | Any risk axis is high | All risk axes low + presentational role |
| Non-goals | Yes (≥ 1 entry) | — | — |

**Risk signals** (internal, three axes, each low / medium / high):

- **Sharing scope** — single one-off use vs. reuse within a domain vs. shared library candidate.
- **State character** — purely presentational vs. UI-stateful vs. coupled to async / external I/O.
- **Exposure breadth** — internal tooling vs. limited audience vs. production traffic.

**Evaluation timing**:

- *First pass*: immediately after Classify, based on the request text alone.
- *Refinement*: re-evaluated after the Where answer, which may shift sharing scope.

**Visibility**: the risk assessment itself is not shown to the user. However, when the assessment changes the depth of subsequent questions, the AI announces the *context* of that depth in one line — using everyday language, not the term "risk." For example: *"This component looks like it is on a production critical path, so I will ask about accessibility and edge cases in more detail."* The user can correct this read, which triggers a re-evaluation.

A compressed dimension is not silently skipped: the resulting spec records a short marker indicating the dimension was treated in compressed form and why.

## Dialogue Mode

**Sequential by default.** The AI asks one question at a time. Each answer is absorbed before the next question is composed, so the next question is calibrated to the current state — including the possibility that no further question in the current dimension is necessary.

This is the mechanism by which the dimension activation policy and the self-check actually do their work; batching would weaken it.

**Three accommodations** soften the friction without diluting the depth:

1. **Absorbing answers richer than asked.** If the user volunteers information beyond what was asked, the AI absorbs it and adjusts (or skips) subsequent questions accordingly. The user may answer sequentially or in bulk; the AI must remain sequential in asking.
2. **Enumerative dimensions may be batched.** Dimensions whose content is genuinely a list — *Non-goals* most clearly — can be asked in one prompt at the AI's discretion.
3. **Multiple choice preferred when answers are bounded.** When a question has a finite set of reasonable answers, the AI presents them as choices rather than asking open-endedly. Open-ended is reserved for cases where free-form input is necessary.

## Self-check

**Trigger**: all active dimensions have been answered.

**Evaluation questions the AI asks of itself**:

1. Could a code-generation step produce a coherent implementation from this spec, in one pass?
2. Are there any captured items that could reasonably be interpreted in two different ways?
3. Are there any areas the risk signals flagged that have not been explored in sufficient depth?

**Threshold**: the ambiguity threshold tightens with risk. High-risk components require a lower ambiguity tolerance than low-risk ones to pass self-check.

**On finding remaining ambiguity**: return to dialogue with additional questions, then re-run self-check. Repeat until no ambiguity remains.

**No iteration cap.** The skill does not exit on remaining ambiguity. If the user cannot answer a particular question, the AI uses the same three accommodations as above to make answering easier:

- Propose a sensible default for the user to explicitly accept or reject.
- Re-cast the question as multiple choice.
- Offer concrete examples to disambiguate intent.

Acceptance of a proposed default is a clarification, not a deferral. The skill never produces a spec that contains unanswered questions — the spec format has no "Open Questions" section.

The AI's responsibility is to vary the *form* of a question when the user is having trouble with it. Repeating the same question in the same form is not acceptable.

## Classification handling

**On a clear component classification**: proceed to Risk read.

**On a clear page or design-guide classification**: tell the user the skill currently handles component specifications only, and terminate without writing a file.

**On an ambiguous classification**: ask the user directly which of the three the request is, and branch on the answer.

## Output Format

The output is a Markdown file with YAML front matter.

### Front matter

```yaml
---
name: <kebab-name>
classification: component
created: <YYYY-MM-DD>
source: <verbatim original natural-language request>
---
```

- `source` preserves the user's original intent for the record.
- The internal risk assessment is **not** written here.

### Body (new or extended component — full path)

```markdown
# <Component name>

## Summary
<two- or three-line natural-language summary, intended for quick human review>

## Where
- Decision: New / Extend
- (Extend only) Reference: <component identifier the user named>
- Rationale: <one line>

## What
- Purpose
- Usage scenarios
- Props
  | name | type | required | default | description |
  |---|---|---|---|---|

## Look
<one of the two — depending on Look activation>

### When the design guide is defined and substantive
- Token overrides: <or "none">

### When the design guide is absent or sparse
- Visual hierarchy
- Spacing
- Responsive behavior
- Token suggestions

## How
- States: loading / empty / error / disabled / ...
- Interactions: hover / focus / keyboard / transitions
- Accessibility: semantic markup / ARIA / focus order / contrast
- Edge cases: long text / RTL / i18n / ...

## Data / Content
- Schema
- Edge data (nullable, truncation, optimistic vs. confirmed, page sizes, etc.)

## Non-goals
- <at least one bullet>
```

Compressed dimensions render as a single line: `(compressed: <reason>)` placed under the dimension heading. The heading itself is always present, so consumers can find every dimension by name.

### Body (reuse path — early termination)

When the Where answer is "an existing component already covers this":

```markdown
# <Component name>

## Summary
<original request restated in one sentence, plus a line stating that an existing component covers it>

## Where
- Decision: Reuse
- Reference: <component identifier the user named>
- Rationale: <one line on why reuse is sufficient>
```

No other sections are written. The spec is intentionally lightweight in this case.

## Constraints on Generated Documents

The spec Markdown files produced by this skill are formal documents. They must not:

- Reference local paths of the host project.
- Use vocabulary specific to development plugins outside the AI Harness.
- Link to files maintained by external plugins.

The only informal vocabulary they may use is vocabulary belonging to the AI Harness itself.

## Open Design Points (deferred)

These are intentionally not pinned in this design and will be settled during implementation or in later iterations:

- The exact wording of dimension question templates.
- The exact heuristic mapping from request-text features to risk signal levels.
- The precise phrasing of the one-line context announcement when risk shifts dialogue depth.

## Source materials

- [`docs/foundation.md`](../../foundation.md) — project vision, the AI Harness end goal, the `DESIGN.md` convention.
- [`docs/obstacles.md`](../../obstacles.md) — the three structural obstacles. Obstacle #1 (intent-to-spec calibration) is the primary motivator of this skill.
- [`docs/slices.md`](../../slices.md), [`docs/select.md`](../../select.md) — the iteration in which this skill is the current target.
