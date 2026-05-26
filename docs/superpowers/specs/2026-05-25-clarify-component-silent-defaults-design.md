---
created: 2026-05-25
---

# Clarify-Component: Removing Silent Defaults

## Context

Two open issues recorded against the mode-toggle clarification run share
a single structural root: `clarify-component` writes decisions into the
spec that were *not* explicitly chosen by the user — they were inferred
by the AI and pinned silently. Once silent decisions land in the spec,
`generate-component` honors them under the "spec is the contract" rule,
even when they no longer reflect current best practice or even what the
user would have chosen if asked.

- [`docs/issues/2026-05-25-clarify-component-implementation-leaks-into-spec.md`](../../issues/2026-05-25-clarify-component-implementation-leaks-into-spec.md)
  — the spec's `## How → Edge cases` section recorded an
  *implementation pattern* (a `useState(false) + useEffect(setMounted(true))`
  mount-guard idiom for `next-themes`) as if it were a behavioral
  requirement. `generate-component` reproduced the scaffolding verbatim,
  even though the shadcn canonical satisfies the same requirement with
  only `<html suppressHydrationWarning>` and Tailwind `dark:` variants.
- [`docs/issues/2026-05-25-clarify-component-interaction-model-variants.md`](../../issues/2026-05-25-clarify-component-interaction-model-variants.md)
  — the spec silently locked in the binary-toggle interaction model for
  `mode-toggle`, when the same name commonly maps to multiple legitimate
  variants (binary toggle, cycle button, 3-way dropdown). The user was
  never shown the alternatives. The variant was the AI's choice, not the
  user's.

Both issues propose, among their candidates, mechanisms that this spec
formalizes into a single integrated design for `clarify-component`. The
sibling Cluster B design
([`2026-05-25-external-package-knowledge-design.md`](2026-05-25-external-package-knowledge-design.md))
closes the two adjacent issues
([#3 host-preconditions-from-imports](../../issues/2026-05-25-generate-component-host-preconditions-from-imports.md),
[#4 stack-skills-icon-library-deprecation-drift](../../issues/2026-05-25-stack-skills-icon-library-deprecation-drift.md))
in `generate-component` and the stack-skills manifest; this spec is its
counterpart for `clarify-component` and the spec format.

## Principle

**Clarify does not create silent defaults.**

Every item written to the spec must be one of:

1. The user's explicit answer,
2. An explicit default proposal the user accepted,
3. An AI-derived item phrased as a *requirement* (no implementation
   prose, no procedural steps, no library APIs in the body), or
4. An AI translation of the user's natural-language answer into
   English, written into the corresponding spec section. This
   shape does not require user acceptance because (i) the
   translation preserves the user's intent rather than introducing
   a new decision, and (ii) the resulting English text is editable
   in the generated code by the user without re-running clarify.
   Shape (4) applies only to translation — it does not authorize
   new requirements, states, props, or behavioral constraints. See
   [`2026-05-26-code-form-quality-gates-design.md`](2026-05-26-code-form-quality-gates-design.md)
   Mechanism 1 for the surrounding language enforcement.

Three consequences follow:

- The spec's behavior-bearing sections (`## How → Interactions`,
  `## How → Edge cases`, `## How → Accessibility`) are
  **requirements-only**. Implementation prose is rejected at clarify
  time and either rephrased as a requirement or moved to an explicit
  escape hatch (`## Implementation hints`).
- When a requested pattern has more than one legitimate interaction
  variant (a class of UI with multiple shipped shapes — binary toggle
  vs cycle vs dropdown for a theme switcher), clarify **must ask**.
  Silent variant inference is not permitted.
- When the user *does* want a specific implementation pattern (parity
  with an existing component, a project policy, an explicit personal
  choice), it is captured in an annotated `## Implementation hints`
  section — clearly marked as a hint, not part of the contract.

## Mechanism 1 — Interaction model dimension

Closes
[#2 clarify-component-interaction-model-variants](../../issues/2026-05-25-clarify-component-interaction-model-variants.md).

### Position in dimension order

The current order is **Where → What → Look → How → Data → Non-goals**.

The new order is **Where → Interaction model → What → Look → How → Data
→ Non-goals**.

Rationale: the interaction variant (binary toggle vs 3-way dropdown for
mode-toggle, for example) determines the props surface (What) and the
state/event model (How). It must be settled before those dimensions
are populated, or those dimensions get filled twice when the variant
later changes. Placement after Where preserves the existing early
termination — when Where = `Reuse`, the rest of the dialogue does not
run, and Interaction model is not asked.

### Activation

The Interaction model dimension is conditional. Its activation is
decided by AI judgment from the request text plus the result of
`Where`'s category-word extraction (see
[`dimensions.md`](../../../skills/clarify-component/references/dimensions.md)
§Where). The criterion:

> The requested component matches a class of UI for which **more than
> one legitimate interaction model** exists in common practice.

Examples that activate: `mode-toggle`, `combobox`, `command-palette`,
`data-table`, `navigation-menu`, `date-picker`, `multi-select`,
`autocomplete`.

Examples that do not activate: `primary-submit-button`, `card`,
`avatar`, `badge`, `breadcrumb` (each has one dominant shape in current
practice; AI may still consider variants but the bar for activation is
"multiple shipped variants in common UI libraries").

The AI uses its own knowledge for this judgment. The clarify-component
skill does **not** invoke any stack skill (e.g., the shadcn skill's
registry) for the activation decision. This preserves clarify as a
self-contained dialogue with no host or stack dependency — the same
posture the existing six dimensions take — and mirrors the Cluster B
principle of "derivation over enumeration."

If activation is uncertain (the AI cannot decide whether the requested
pattern has multiple legitimate variants), default to **activate**. The
cost of an unnecessary variant question is a single dialogue turn the
user can dismiss ("only one variant applies — proceed"); the cost of a
silent miss is the failure mode the issue records.

### Question shape

When activated, the dimension asks a single multiple-choice question
that enumerates the variants the AI identified:

> "<pattern name> commonly has multiple interaction models. Which one
> should this component implement?
> - **<Variant 1 name>** — <one-line description> (e.g., used by <one
>   real-world example, if known>)
> - **<Variant 2 name>** — <one-line description>
> - **<Variant 3 name>** — <one-line description>
> 
> If none of the above matches what you want, describe the variant you
> have in mind."

For `mode-toggle`, this would be:

> "Mode-toggle commonly has multiple interaction models. Which one
> should this component implement?
> - **Binary toggle** — a single button that flips between light and
>   dark.
> - **Cycle button** — a single button that cycles through light →
>   dark → system on each click.
> - **3-way dropdown** — a dropdown menu offering light, dark, and
>   system as separate items (the shadcn canonical)."

### Spec output

A new top-level section in the spec body when the dimension activated:

```markdown
## Interaction model
- Variant: <chosen variant name>
- Rationale: <one line: the user's reason in their own words, or "user accepted default proposal">
```

When the dimension was not activated, the section is omitted from the
spec entirely. Its absence carries information: it means the AI did
not detect multiple legitimate variants for this pattern, and a single
interaction model is implicit in the rest of the spec.

### Composition-aware framing

When the Where decision is `Compose` and the Reference primitive
itself constrains the variant (e.g., spec says "Compose using
DropdownMenu" — that already chooses the dropdown variant), the
Interaction model dimension is **compressed** rather than skipped: a
single line `## Interaction model` with the body
`(compressed: variant fixed by Reference primitive <name>)` plus the
variant restated. This preserves the section's discoverability for
spec readers while not asking a redundant question.

## Mechanism 2 — Requirements-only voice for `## How`

Closes
[#1 clarify-component-implementation-leaks-into-spec](../../issues/2026-05-25-clarify-component-implementation-leaks-into-spec.md).

### Where it applies

The voice rule applies to three sub-sections of `## How`:

- `## How → Interactions`
- `## How → Edge cases`
- `## How → Accessibility`

These sections describe *what the component must do* (its observable
behavior). They are not the place for *how it is built*. The other
`## How` sub-section, `## How → States`, is already requirement-shaped
(a list of named states) and is not affected.

### What the rule rejects

A statement is rejected from these sections when it contains:

1. **Library or API names in the body prose** (`useEffect`,
   `useState`, `next-themes`, `framer-motion`, `useContext`,
   `useRouter`, etc.). Library names that appear *in the user's
   verbatim request* and are simply quoted back are not rejected; the
   rule fires on prose the AI authored.
2. **Procedural steps** ("...then ...", "first ... after which ...",
   "render X and replace with Y"). Requirements describe end-states
   and constraints; procedures describe construction.
3. **Citations of specific implementation idioms** ("mount-guard
   pattern", "render prop", "compound component", "controlled
   component pattern" used as a how-to instruction rather than a prop
   shape).

### Enforcement

The voice rule fires at two points:

**Authoring time.** When the AI is composing a candidate sentence for
one of these sections, it self-checks against the rejection criteria
above *before* presenting the sentence to the user. If the candidate
fails, the AI rewrites in requirements voice (`useEffect(setMounted)`
→ "no flash of incorrect theme on initial render") and proceeds.

**Self-check time.** The existing Self-check step in
[`SKILL.md`](../../../skills/clarify-component/SKILL.md) (currently
Step 9; becomes Step 10 after the Interaction model dimension is
inserted as Step 4) gains a new item: re-read every bullet in
`## How → Interactions`, `## How → Edge cases`, and
`## How → Accessibility`. Apply the rejection criteria. If any
bullet matches, return to dialogue with a single question targeted
at that bullet:

> "I noticed this bullet looks like an implementation pattern rather
> than a requirement: *'<offending text>'*. Could you tell me what
> the *result* needs to be? For example: *'<helpful rephrase
> suggestion>'*. If the specific pattern itself is the requirement
> (for parity with another component, or because the project mandates
> it), I'll record it separately as an Implementation hint."

The user's answer routes to one of two outcomes:
- "The result is X" → rewrite the bullet as the requirement; remove
  the implementation prose.
- "The pattern itself is the requirement" → move the prose to the
  optional `## Implementation hints` section (Mechanism 3) with the
  user's stated reason as the rationale; the original `## How`
  section is rewritten as the underlying requirement.

The Self-check step repeats until no `## How` bullet trips the
rejection criteria.

### Boundary: terms that are not API names

The voice rule does not reject domain language ("the dropdown opens
on click", "focus moves to the first item"). It rejects implementation
mechanics (the *means* of achieving those behaviors). A bullet
"opening the dropdown sets `aria-expanded='true'`" is requirements
voice — `aria-expanded` is the observable outcome the component must
produce, not an implementation choice. A bullet "the open-state is
held in a `useState` hook" is implementation voice — `useState` is one
of several ways to hold open-state, and the spec should not pin it.

## Mechanism 3 — Optional `## Implementation hints` section

Closes the third design candidate of
[#1](../../issues/2026-05-25-clarify-component-implementation-leaks-into-spec.md):
"surface implementation patterns as explicit user-stated constraints
with annotation, not as unmarked defaults."

### Format

A new optional section in the spec body:

```markdown
## Implementation hints
(optional; omit when none)

- <one-line user-stated implementation pattern>: <rationale — why this specific pattern, not just the requirement>
```

Each bullet pairs the pattern with the user's reason. The rationale
is mandatory — a hint without a stated reason is rejected at clarify
time (because such a hint is indistinguishable from a silent default,
which the principle forbids).

### Authoring

`clarify-component` does **not** ask for this section on its own
initiative. It is filled only as the routed outcome of the voice-rule
reject flow (Mechanism 2): when the user answers "the pattern itself
is the requirement, here's why", clarify writes the bullet here and
records the rationale.

A user editing the spec after generation may also add bullets here
manually; the format is documented for that case.

### Consumption by generate-component

The hint is **a strong recommendation, not a contract**.
`generate-component` reads `## Implementation hints` alongside the
spec's other sections. For each hint:

- The hint is followed by default.
- If a loaded stack skill (e.g., `vercel-react-best-practices`,
  `next-best-practices`) prescribes a different pattern for the same
  underlying requirement, the stack skill's prescription wins, and an
  `attention` row appears on the Plan with the form:

  | slot | value | source |
  |---|---|---|
  | `Hint divergence: <hint text>` | `Stack skill recommends <stack pattern>; hint not followed.` | `attention` |

- The user can override the divergence at Plan approval by amending
  the row; that amendment is a spec-level change (per the existing
  spec-contradiction termination rule) and the user is directed back
  to `clarify-component` to update the spec.

This contrasts with `## Host preconditions` (added by the Cluster B
design), which is a *contract* — its bullets are verified against the
host and produce `host-setup-required` rows on mismatch.
Implementation hints are non-contract; their divergence is
informational and resolved by user judgment.

## Reference file changes

Concrete edits to support the design.

- [`skills/clarify-component/SKILL.md`](../../../skills/clarify-component/SKILL.md):
  - Process step list grows from ten steps to eleven. The new
    Interaction model dimension is inserted as **Step 4**, between
    current Step 3 (Where) and current Step 4 (What). Subsequent
    steps shift accordingly: What becomes Step 5, Look becomes
    Step 6, How becomes Step 7, Data/Content becomes Step 8,
    Non-goals becomes Step 9, Self-check becomes Step 10, Write
    spec becomes Step 11.
  - The Self-check step (renumbered to Step 10) gains a new item:
    voice-rule enforcement for `## How → Interactions / Edge cases
    / Accessibility`. The iteration-cap-free repeat language is
    preserved.
  - The "six dimensions" language in the Process introduction
    becomes "seven dimensions".

- [`skills/clarify-component/references/dimensions.md`](../../../skills/clarify-component/references/dimensions.md):
  - **New section `## Interaction model (conditional)`** between
    `## Where` and `## What`. Defines activation criterion, the
    multiple-choice question template, the Compose-fixed compression
    case, and the outcome (the `## Interaction model` spec section).
  - **Activation matrix** updated: new row `Interaction model | No |
    Multiple legitimate variants detected | Compose with variant-
    fixed Reference primitive`.
  - **Note on requirements-only voice for `## How`** added after the
    existing "Note on composition-aware framing", stating the voice
    rule and pointing at the Self-check step enforcement (Step 10
    after the renumbering above).
  - **Ordering line** at the top updated from
    `Where → What → Look → How → Data → Non-goals` to
    `Where → Interaction model → What → Look → How → Data →
    Non-goals`.

- [`skills/clarify-component/references/output-format.md`](../../../skills/clarify-component/references/output-format.md):
  - **`## Interaction model`** section definition added to the
    full-path body template, between `## Where` and `## What`. The
    compressed-rendering rules section gains a sentence covering the
    `(compressed: variant fixed by Reference primitive <name>)` case.
  - **`## Implementation hints`** (optional) section definition
    added to the full-path body template, between `## Non-goals` and
    `## Host preconditions` (the Cluster B addition). An explanatory
    subsection follows the body template (mirroring how Cluster B's
    `## Host preconditions` explanatory subsection was added), stating
    that the section is hint-only and filled only via the voice-rule
    reject flow.
  - **`## How` voice rule** stated in a short paragraph attached to
    the `## How` row of the body template, with the three rejection
    criteria summarized and a pointer to dimensions.md and the
    Self-check step in SKILL.md (Step 10 after renumbering) for the
    enforcement mechanism.

- [`skills/generate-component/references/output-format.md`](../../../skills/generate-component/references/output-format.md):
  - **spec → code mapping** table gains one row:
    `## Interaction model | The component's overall structural shape
    (primitive composition, event topology, exposed handlers). The
    chosen variant constrains which primitives the body composes and
    which handlers are exposed via props.`
  - **`## Implementation hints` handling** added as a new short
    section after the imports rule. States: hints are followed by
    default; stack skill prescriptions for the same requirement
    override hints; divergences produce `attention` rows per
    plan-format.md.

- [`skills/generate-component/references/plan-format.md`](../../../skills/generate-component/references/plan-format.md):
  - The `attention` label definition gains a second example: "spec
    `## Implementation hints` bullet conflicts with a loaded stack
    skill's prescription for the same requirement." The row format
    given in this design's Mechanism 3 is embedded.

- `skills/setup/references/stack-skills.md` — **unchanged**.
  Same posture as Cluster B's design: no new manifest entries; the
  existing eight major stack skills carry whatever knowledge is
  needed to evaluate hint vs prescription divergence.

## Out of scope

- **`shadcn` registry / external catalog invocation from clarify.**
  AI general knowledge is the first slice for variant detection. If
  variant-detection accuracy proves low in practice (false negatives
  on patterns clarify failed to recognize as multi-variant),
  calibration is a follow-up — not a fallback to a static catalog or
  a new skill invocation point.
- **Generate-component's precise hint-vs-prescription resolution
  logic.** This design defines the `attention` mechanism and the
  hint's non-contract status. *Which* prescription overrides *which*
  hint is owned by the relevant stack skill body — the same delegation
  pattern the existing slot policy uses for stack-convention
  decisions.
- **Retroactive cleanup of existing specs with implementation prose.**
  The mode-toggle spec produced in the original run still contains
  the mount-guard prose in `## How → Edge cases`. This design does
  not migrate it. The voice rule fires on *new* clarify runs only;
  the user may re-run `clarify-component` against mode-toggle (or
  edit the spec by hand) to produce a clean version. A bulk-migration
  tool is not built.
- **`clarify-component`'s natural-language delivery layer.** The
  question templates above are written in English as the source
  language; runtime delivery in the user's language (per the existing
  Language section of SKILL.md) is unchanged — the new dimension's
  question is delivered in the user's language the same way the
  existing six dimensions' questions are.

## Verification

After implementation, three scenarios cover the design.

### Scenario A — mode-toggle re-run, Interaction model dimension fires

Re-run `/gyre:clarify-component "Build a mode-toggle component that
switches between dark mode and light mode."` (the same request that
produced the original spec).

Expected dialogue:

1. Classify → component.
2. Where → category word "mode-toggle" extracted. Q1a (same-role
   candidate) and Q1b (base primitive) asked. Suppose user answers
   no for both → Decision: `New` (or `Compose` if user names a
   primitive like `Button` or `DropdownMenu`).
3. **Interaction model dimension fires.** AI identifies multiple
   legitimate variants for mode-toggle and asks the multiple-choice
   question (binary toggle / cycle button / 3-way dropdown). User
   chooses one explicitly.
4. What, Look, How, Data, Non-goals proceed as before, calibrated by
   the chosen variant (props surface and states match the variant).
5. The Self-check step (Step 10 after renumbering) runs the voice
   rule on the populated `## How` bullets.
6. Spec written.

Expected spec changes compared to the original:

- A new `## Interaction model` section appears with the chosen
  variant and a rationale line.
- The `## How → Edge cases` section no longer contains the
  `next-themes / mounted` placeholder prose. The same requirement is
  expressed as: *"No layout shift across the SSR → CSR transition
  for this control."*
- If the user chose the binary-toggle variant (matching the original
  spec's silent default), the spec's What/How sections match the
  original on the variant-shaped fields; if the user chose a
  different variant, those fields differ accordingly.

This scenario closes
[#2](../../issues/2026-05-25-clarify-component-interaction-model-variants.md)
when the variant choice is recorded explicitly. It closes the
*silent-default* portion of
[#1](../../issues/2026-05-25-clarify-component-implementation-leaks-into-spec.md)
when the mount-guard prose is replaced by the requirement.

### Scenario B — voice-rule reject flow with user-confirmed implementation pattern

In the same mode-toggle re-run, simulate the AI authoring a
mount-guard sentence into `## How → Edge cases`. The Self-check step
rejects, dialogue resumes with the rephrase question. Inject a user
answer of the form: *"The mount-guard pattern itself is the
requirement — it matches the pattern used by every other component in
this project."*

Expected:

- The `## How → Edge cases` bullet is rewritten as a requirement
  ("no layout shift across SSR → CSR transition").
- A new `## Implementation hints` section appears in the spec with:
  
  ```
  - Use the next-themes mount-guard pattern (`useState(false) + useEffect(setMounted(true))`): matches the pattern used by every other component in this project.
  ```
  
- generate-component, when run against this spec, emits the
  mount-guard scaffolding (following the hint by default). If a
  loaded stack skill prescribes a different pattern (e.g., shadcn's
  `<html suppressHydrationWarning>`-only approach), the Plan shows
  an `attention` row noting the divergence and the user resolves at
  Plan approval.

This scenario closes the *escape-hatch* portion of
[#1](../../issues/2026-05-25-clarify-component-implementation-leaks-into-spec.md).

### Scenario C — single-variant component (negative case)

Run `/gyre:clarify-component "primary submit button"` — a pattern
with one dominant shape in current practice.

Expected:

- Interaction model dimension is **not activated**. No question is
  asked, no spec section is written for it.
- The dialogue runs Where → What → Look → How → Data → Non-goals as
  it does today.
- The Self-check step's voice rule fires on `## How` bullets as in
  Scenarios A/B, but the rule applies regardless of whether
  Interaction model was active — it is a property of `## How`, not
  of the new dimension.

This scenario verifies the activation condition is correctly gated
and the new dimension does not add noise to single-variant requests.

## Follow-ups

- **Variant-detection accuracy calibration.** If Interaction model
  activation produces frequent false negatives (multi-variant
  patterns the AI failed to recognize) or false positives (single-
  variant components where the dimension fires unnecessarily), log
  the offending runs as new issues under `docs/issues/`. The
  remediation is a calibration of the activation criterion in
  `dimensions.md`, not an external catalog or new skill invocation.
- **Implementation hint divergence telemetry.** When generate-component
  produces `attention` rows for hint-vs-prescription divergence, the
  divergence carries information about how often spec-level hints
  conflict with current stack practice. Worth tracking informally so
  a future calibration of the voice-rule rejection criteria has data
  to draw on.
- **Compose-fixed compression accuracy.** The Compose case
  (Mechanism 1, Composition-aware framing) compresses the Interaction
  model dimension when the Reference primitive fixes the variant.
  If the compression decision proves brittle (e.g., the primitive
  *partly* fixes the variant but variations remain), the calibration
  is a refinement to the compression criterion in `dimensions.md`,
  not a removal of the compression path.

## Related

- Open issues this design closes:
  - [`docs/issues/2026-05-25-clarify-component-implementation-leaks-into-spec.md`](../../issues/2026-05-25-clarify-component-implementation-leaks-into-spec.md)
  - [`docs/issues/2026-05-25-clarify-component-interaction-model-variants.md`](../../issues/2026-05-25-clarify-component-interaction-model-variants.md)
- Sibling design (Cluster B — generate-component / stack-skills side
  of the mode-toggle gap):
  [`2026-05-25-external-package-knowledge-design.md`](2026-05-25-external-package-knowledge-design.md)
- Skill bodies touched by this design:
  - [`skills/clarify-component/SKILL.md`](../../../skills/clarify-component/SKILL.md)
  - [`skills/clarify-component/references/dimensions.md`](../../../skills/clarify-component/references/dimensions.md)
  - [`skills/clarify-component/references/output-format.md`](../../../skills/clarify-component/references/output-format.md)
  - [`skills/generate-component/references/output-format.md`](../../../skills/generate-component/references/output-format.md)
  - [`skills/generate-component/references/plan-format.md`](../../../skills/generate-component/references/plan-format.md)
- Manifest (intentionally unchanged):
  [`skills/setup/references/stack-skills.md`](../../../skills/setup/references/stack-skills.md)
- Adjacent prior design that established Interaction model's neighbor
  output section (`## Host preconditions`):
  [`2026-05-25-external-package-knowledge-design.md`](2026-05-25-external-package-knowledge-design.md)
  §Mechanism 2 (spec optional escape hatch).
