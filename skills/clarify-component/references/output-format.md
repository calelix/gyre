# Output format

The skill writes one Markdown file per successful invocation, at:

```
docs/gyre/specs/components/<kebab-name>.md
```

`<kebab-name>` is established during the *What* dimension (Naming question). The output path is fixed in this iteration and is not user-overridable.

Sibling directories are reserved for future artifact types:

- `docs/gyre/specs/pages/<kebab-name>.md`
- `docs/gyre/specs/DESIGN.md`

The skill must not write to those paths in this iteration.

## Front matter (always present)

```yaml
---
name: <kebab-name>
classification: component
created: <YYYY-MM-DD>
source: <verbatim original natural-language request>
---
```

- `source` preserves the user's original intent for the record. The full original request goes here, even if subsequent dialogue restated it.
- The internal risk assessment is **not** written here.

## Body — new, extended, or composed component (full path)

Use this body when the *Where* decision is `New`, `Extend`, or `Compose`. Render every dimension heading even if the dimension was compressed.

````markdown
# <Component name>

## Summary
<two- or three-line natural-language summary, intended for quick human review>

## Where
- Decision: New | Extend | Compose
- (Extend or Compose only) Reference: <one or more component identifiers the user named>
- Rationale: <one line>

## Interaction model
(rendered only when the dimension activated; omitted otherwise)

- Variant: <chosen variant name>
- Rationale: <one line: the user's reason in their own words, or `user accepted default proposal`>

## What
- Purpose: <one sentence>
- Usage scenarios:
  - <bullet per scenario>
- Props:

  | name | type | required | default | description |
  |---|---|---|---|---|
  | <name> | <type> | <yes/no> | <default or —> | <description> |

## Look
<one of the two — depending on Look activation>

### When the design guide is defined and substantive
- Token overrides: <list, or `none`>

### When the design guide is absent or sparse
- Visual hierarchy: <text>
- Spacing: <text>
- Responsive behavior: <text>
- Token suggestions: <list, or `none`>

## How
- States: <comma-separated list, e.g. default, hover, focus, disabled, loading>
- Interactions: <requirements-only — describe what the user can do and what observable outcomes occur; reject implementation prose>
- Accessibility: <requirements-only — semantic markup, ARIA, focus order, contrast as observable properties>
- Edge cases: <requirements-only — long text, RTL, i18n, and any other named edge cases as constraints the component must satisfy>

## Data / Content
- Schema: <text or table>
- Edge data: <text covering nullable, truncation, optimistic vs confirmed, page sizes, etc.>

## Non-goals
- <at least one bullet>

## Implementation hints
(optional; omit when none)

- <one-line user-stated implementation pattern>: <rationale — why this specific pattern, not just the requirement>

## Host preconditions
(optional; omit when none)

- <one-line requirement>: <one-line signal description>
````

### `## Implementation hints` (optional)

The `## Implementation hints` section is optional and omitted from the spec when none apply. It captures implementation patterns the *user explicitly stated* they want (for parity with an existing component, project policy, or personal preference) — not patterns the AI inferred. Each bullet pairs a one-line pattern with a one-line rationale; the rationale is mandatory, because a hint without a stated reason is indistinguishable from a silent default (which the principle forbids).

`clarify-component` does **not** ask for this section on its own initiative. It is filled only as the routed outcome of the `## How` voice-rule reject flow (see `dimensions.md` *Note on requirements-only voice for `## How`*): when the user answers "the pattern itself is the requirement, here's why" to a rephrase question, clarify records the bullet here.

The hint is consumed by `generate-component` (see [`output-format.md`](../../generate-component/references/output-format.md) → *Implementation hints handling*) as a **strong recommendation, not a contract**. The hint is followed by default; when a loaded stack skill prescribes a different pattern for the same underlying requirement, the stack skill's prescription wins and an `attention` row appears on the Plan noting the divergence.

### `## Host preconditions` (optional)

The `## Host preconditions` section is optional and omitted from the spec when none apply. It captures preconditions a spec author knows about that may not be derivable from the spec's import surface alone — for example, project policies ("host must have initialized Sentry before this loads") or non-obvious package contracts. The section is consumed by `generate-component` (see [`host-discovery.md`](../../generate-component/references/host-discovery.md) → *Consumer-implied preconditions — derivation and verification*), where each bullet's `requirement: signal` pair is fed through signal verification and merged with AI-derived preconditions (deduplicated by requirement text).

`clarify-component` does **not** ask for this section in this iteration. It is filled by the spec author at write-time or by the user editing the spec later. A future iteration of clarify may add a dimension or step that proposes preconditions to the user; that is out of scope here.

### Compressed-dimension rendering

When *Look* or *Data / Content* is treated in compressed form, render the section as:

```markdown
## <Dimension name>
(compressed: <reason>)
```

For *Data / Content* compressed, the `<reason>` is typically a phrase like `low risk; presentational role`. For *Look* compressed, the `<reason>` is typically `existing design guide covers visual/spacing/responsive tokens`.

The compressed line **must** be followed by the single substantive line that the compressed path captured (the schema sentence for *Data / Content*, the token-override answer for *Look*). Example:

```markdown
## Data / Content
(compressed: low risk; presentational role)

- Schema: a string `label` and an optional `icon`.
```

When the *Interaction model* dimension is compressed because the *Where* decision is `Compose` and the Reference primitive itself fixes the variant, render the section as:

```markdown
## Interaction model
(compressed: variant fixed by Reference primitive DropdownMenu)

- Variant: 3-way dropdown
```

The compressed line is followed by the single substantive line stating the implied variant, so a spec reader does not need to re-derive it from the Reference name.

### `## How` voice rule

The three sub-sections of `## How` that describe observable behavior — `Interactions`, `Edge cases`, `Accessibility` — follow a requirements-only voice. The rule is governed by a canonical semantic question and three tests; example sets in any language are anchors, not a complete vocabulary.

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

When a bullet trips any of the three tests, the AI returns to dialogue with a single rephrase question, and the user's answer routes to one of two outcomes: rewrite the bullet as the underlying requirement, or — when the user confirms the specific pattern itself is the requirement — record the prose under the optional `## Implementation hints` section above, with the user's stated rationale.

**Boundary: layered observability.** When a single user-observable contract admits multiple equally observable sub-statements at descending layers of specificity, state the requirement at the **highest layer where user intent is fully captured**. Lower-layer statements that name a particular technique among legitimate alternatives are mechanism citations under Test 3.

Worked example — mode-toggle accessibility:

- L0 (highest, user-observable): *"Button exposes an accessible name that, when read aloud, conveys the next action available given the current theme."*
- L1: *"Button has an `aria-label` attribute whose value conveys that action."* ← rejected by Test 3 (`<span class="sr-only">` text or `aria-labelledby` reference could substitute without changing what the user perceives).
- L2: *"Button has `aria-label='Switch to dark mode'` when in light mode."* ← rejected by Test 3 (same reason).

Spec authors state at L0. If the user explicitly requests a lower layer for parity (e.g., "I want `aria-label` specifically because the shadcn ModeToggle uses it"), route the prose to `## Implementation hints` with the user's rationale, per the existing reject flow.

Enforcement happens at the *Self-check* step (Step 10 in `SKILL.md`); the rule's authoring-time and self-check-time application are also described in `dimensions.md` *Note on requirements-only voice for `## How`*. This is the spec-side half of a two-part contract with `generate-component`'s *Pattern selection deference* ([`../../generate-component/references/output-format.md`](../../generate-component/references/output-format.md) §Pattern selection deference): clarify must not pre-commit at a layer where stack skills speak; generate must consult those skills at write time.

### Open Questions

There is no `Open Questions` section. Every captured item must be a resolved decision. Under the AskUserQuestion delivery rule (see `../SKILL.md` `## Dialogue mode` accommodation 3), every question is already presented with AI-proposed candidate options plus an automatic "Other" choice; the user's selection of an option is acceptance of that proposed default, and an "Other" response is the user's explicit answer. If the user types something equivalent to "I don't know" in "Other", the AI must widen the candidate set, offer concrete examples in option descriptions, or propose its most-plausible candidate for explicit accept/reject. Acceptance of a proposed default is a resolved decision.

## Body — reuse path (early termination)

Use this body when the *Where* decision is `Reuse`. No other dimensions are written.

````markdown
# <Component name>

## Summary
<one sentence restating the request, plus one sentence stating that an existing component covers it>

## Where
- Decision: Reuse
- Reference: <component identifier the user named>
- Rationale: <one line on why reuse is sufficient>
````

## Spec language rule

All spec content is written in English, regardless of the dialogue language. The AI translates the user's natural-language answers into English at spec write time. Documentation-style sections (Summary, Purpose, Rationale, descriptions) and code-bound string literals (default prop values, aria-label / aria-describedby text, sr-only content, edge-case literal indicators) are all in English. The user can adjust any string in the generated code if a different exact text is desired.

This rule is enforced at clarify write time and verified at `generate-component` Step 2 (Load spec); see [`../../generate-component/SKILL.md`](../../generate-component/SKILL.md) Step 2 for the safety-net validation.

## Constraints on the generated file

The generated Markdown file is a formal document. It must not:

- Reference local paths of the host project beyond the conventional output path itself.
- Use vocabulary specific to development plugins outside the AI Harness.
- Link to files maintained by external plugins.

The only informal vocabulary the file may use is vocabulary belonging to the AI Harness.

## Filename rules

- `<kebab-name>` is lowercase, words separated by hyphens, ASCII only.
- The filename is derived from the *What* dimension's Naming question. If the user proposed a name with spaces or mixed case, convert to kebab-case for the filename and use the user's chosen form as the heading `# <Component name>` in the body.
- If a file already exists at the target path, the skill must not overwrite it silently. Instead, it tells the user the path is taken and asks whether to overwrite or pick a different name.
