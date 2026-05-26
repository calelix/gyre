---
created: 2026-05-26
---

# Code-Form Quality Gates

## Context

The mode-toggle generation run produced a component file with two
defects:

1. A `useState(mounted) + useEffect(setMounted)` mount-guard branch
   inside the component body, even though the shadcn canonical at
   <https://ui.shadcn.com/docs/dark-mode/next> satisfies the same
   requirement with only `<html suppressHydrationWarning>` and Tailwind
   `dark:` variants — no mount guard.
2. User-language (non-English) string literals inside the component
   (e.g., `aria-label` and visible label strings written in the
   user's input language), even though the host project's source-code
   language is English.

Both defects share a single structural pattern: `clarify-component`
writes a spec, and `generate-component` consumes it as a contract.
Between the two transit points there is no **quality gate** that
catches code-form decisions — language of code-bound content,
implementation-pattern leakage, modern best-practice prescription —
before they crystallize into the emitted file.

Three prior designs already addressed parts of this surface:

- [2026-05-25-clarify-component-silent-defaults-design.md](2026-05-25-clarify-component-silent-defaults-design.md)
  closed the *silent-default* failure mode in clarify with a
  requirements-only voice rule on `## How` and an optional
  `## Implementation hints` escape hatch.
- [2026-05-25-stack-skills-invocation-design.md](2026-05-25-stack-skills-invocation-design.md)
  closed the *invocation gap* by inserting Step 2.5 in
  `generate-component` and extending the manifest schema with `kind`
  and `condition` columns.
- [2026-05-25-external-package-knowledge-design.md](2026-05-25-external-package-knowledge-design.md)
  closed the *icon-library deprecation drift* by making host
  discovery resolve current export names from installed type
  definitions.

This design closes the remaining gaps that the mode-toggle case
exposed: (a) the language policy treats spec content as a single
category, so user-language content leaks into the emitted code;
(b) the voice rule's rejection examples enumerate a closed set of
surface tokens, so semantically equivalent prose phrased outside
that set passes through; (c) the stack-skills-invocation design has
been applied to the skill files but never verified end-to-end, so
the consultation that would override the AI's general-knowledge
mount-guard default is not guaranteed to fire.

## Principle

> **At each transit point between user input, spec, and code,
> code-form decisions pass through an explicit quality gate.**

Code-form decisions affect the emitted file's shape: which language
its string literals use, which implementation patterns appear in its
body, which best practices it conforms to. The existing
`spec is the contract` rule (`generate-component` consumes the spec
verbatim) is preserved; this design adds gates at the points where
content crosses transit, so the spec is no longer the place where
unchecked decisions silently accumulate.

Three mechanisms, each closing one transit:

| Mechanism | Transit | Closes |
|---|---|---|
| **1. Spec language enforcement** | clarify dialogue → spec content | Defect (b) above: non-English string literals in code |
| **2. Voice rule semantic upgrade** | clarify dialogue → spec `## How` bullets | Defect (a) above (semantic side): implementation prose disguised in idiomatic phrasings outside the closed reject-anchor list |
| **3. Stack-skills-invocation lock-in** | spec → generate code emission | Defect (a) above (mechanism side): mount-guard fallback because stack skill knowledge never reached the runtime |

### Relationship to prior designs

- This design **does not supersede** the three prior designs above. It
  extends them.
- The silent-defaults design's principle gains one additional
  permitted spec-item shape (see Mechanism 1 §Relation to
  silent-defaults).
- The invocation design's Verification section gains one new scenario
  (see Mechanism 3 §A).
- The external-package-knowledge design is unaffected.

## Overarching language policy

> **Skill–user interaction language only.** Questions, prompts,
> multiple-choice presentations, Plan approval dialogue, and final
> reports — anything the skills deliver to or solicit from the user
> — are in the user's language, inferred from the user's messages.
> Everything else — the spec body in its entirety (Summary, Purpose,
> descriptions, rationale, edge cases, code-bound string literals),
> the emitted code (identifiers, string literals, comments), and any
> internal AI processing — is in **English without exception**.

This is the load-bearing policy under which Mechanism 1 reduces to a
simple authoring rule and Mechanism 2's multilingual coverage applies
specifically to the dialogue-input recognition step.

The policy applies to both skills. Each skill's existing Language
section is rewritten per the edits listed in §Reference file changes.

## Mechanism 1 — Spec language enforcement

Closes the language-leakage failure mode (defect b in Context).

### Definition

`clarify-component` writes the entire spec body in English, regardless
of the dialogue language. The AI translates the user's natural-language
answers into English at spec write time. No user re-prompt for the
exact English form is required — the AI's English rendering is the
default; the user adjusts in the spec file or the generated code if
the exact text matters.

### Relation to silent-defaults

The silent-defaults design's principle lists three permitted spec-item
shapes:

1. The user's explicit answer.
2. An explicit default proposal the user accepted.
3. An AI-derived item phrased as a requirement (no implementation
   prose, no procedural steps, no library APIs in the body).

This design adds a fourth permitted shape:

> **4. An AI translation of the user's natural-language answer into
> English, written into the corresponding spec section.** This shape
> does not require user acceptance because (i) the translation
> preserves the user's intent rather than introducing a new decision,
> and (ii) the resulting English text is editable in the generated
> code by the user without re-running clarify.

Shape (4) applies only to translation. It does not authorize the AI
to author new requirements, new states, new props, or new behavioral
constraints that the user did not provide — those still fall under
shapes (1)–(3) and remain subject to the silent-defaults principle.

### Dialogue flow

- Existing dimension questions are delivered in the user's language
  per the overarching language policy; their wording is unchanged.
- The user answers in their language. The AI processes the answer
  through the voice rule (Mechanism 2) before writing.
- For surviving content, the AI writes the corresponding spec
  section in English.
- No additional dialogue turns are added by this mechanism.

### Spec format

The spec format file is unchanged in structure. Every section's
content is in English.

`skills/clarify-component/references/output-format.md` gains one
short subsection that states the rule:

> **Spec language rule.** All spec content is written in English,
> regardless of the dialogue language. The AI translates the user's
> natural-language answers into English at spec write time.
> Documentation-style sections (Summary, Purpose, Rationale,
> descriptions) and code-bound string literals (default prop values,
> aria-label/aria-describedby text, sr-only content, edge-case
> literal indicators) are all in English. The user can adjust any
> string in the generated code if a different exact text is desired.

### Generate-side safety net

`generate-component` Step 2 (Load spec) gains a validation pass:

> After parsing the spec body, scan every section's content for
> non-ASCII characters. If any are found, terminate with the message:
>
> > "The spec at `<path>` contains non-ASCII content in section
> > `<section name>`. Spec content must be in English (enforced by
> > `clarify-component`). Re-run `/gyre:clarify-component
> > <kebab-name>` or edit the spec to use English, then re-invoke
> > `/gyre:generate-component`."
>
> Do not load stack skills, do not perform host discovery, do not
> write any file.

This protects against manual spec edits that reintroduce
user-language content and against legacy specs created before this
design landed.

## Mechanism 2 — Voice rule semantic upgrade

Closes the implementation-prose-disguised-as-requirement failure mode
(defect a in Context, on the clarify side).

### Current state

The voice rule from the silent-defaults design rejects bullets in
`## How → Interactions`, `## How → Edge cases`, and
`## How → Accessibility` when they contain:

1. Library or API names the AI authored (`useEffect`, `useState`,
   `next-themes`).
2. Procedural steps ("...then ...", "first ... after which ...").
3. Citations of named idioms ("mount-guard pattern", "render prop").

The rejection list is a closed set of surface tokens. The mode-toggle
spec's `## How → Edge cases` bullet expressed the same construction
("placeholder before client mount; swap to real icon after client
mount completes") in idiomatic prose that does not literally match
any listed token. The bullet slipped through because the AI applied
the rule by surface match against the example tokens rather than by
the underlying semantic question.

### Restructured rule

The rule is restructured around the **canonical semantic question**:

> Does this bullet describe an **observable property** of the
> component (a requirement), or **a means by which the component
> achieves something** (an implementation)?

Three tests apply, all anchored by English example sets that
illustrate the categorical shape of rejected bullets rather than a
closed vocabulary:

**Test 1 — Outcome test (canonical).** Can a black-box observer of
the rendered component verify this bullet is true by looking at
observable output alone — DOM state, ARIA attributes, visual
presence, interaction response? If yes, the bullet is a requirement
and passes. If the bullet describes internal mechanism (state held
inside the component, lifecycle moments, internal data flow), it is
rejected.

**Test 2 — Procedural test.** Does the bullet describe a sequence of
actions? Reject. Sequence indicators include but are not limited to
"then", "after", "first... after which", "next".

Sequences imply construction.

**Test 3 — Mechanism citation test.** Does the bullet name a
specific technique, hook, pattern, or library identifier? Reject.
Examples include but are not limited to `useEffect`, `useState`,
`useContext`, `useRouter`, `mount-guard pattern`, `render prop`, and
`compound component`.

The example set is non-exhaustive and illustrates the categorical
shape, not a closed vocabulary. The AI is instructed to recognize
semantically equivalent expressions using the listed examples as
anchors.

### Mode-toggle re-evaluation

Applying the restructured tests to the original "placeholder before
client mount; swap to real icon after client mount completes" bullet:

- Outcome test: "swap before/after mount" describes internal
  lifecycle, not observable output → reject.
- Procedural test: "before ... after ... completes" is a sequence
  → reject.
- Mechanism citation test: "mount", "mount completes", `next-themes`
  are mechanism citations → reject.

All three tests fire. The bullet is routed to the existing rephrase
question; the user's answer either becomes the requirement form
("no layout shift across the SSR → CSR transition for this control")
or moves to `## Implementation hints` with the user's stated reason.

### Enforcement points

The rule fires at the same two points the silent-defaults design
established:

- **Authoring time** — the AI self-checks every candidate bullet
  before writing. The check operates on the user's
  natural-language input (before English translation per Mechanism 1)
  so that idiomatic phrasings are caught before being washed into
  English-language spec prose where the categorical shape might be
  obscured.
- **Self-check time** — the existing Step 10 in `SKILL.md` re-applies
  the three tests to every bullet in the three `## How` sub-sections.
  The spec at this point is in English (per Mechanism 1). The
  wording is kept identical between dimensions.md and
  output-format.md per existing convention.

### Author-checker blind spot

Because the same model authors and self-checks, a residual blind
spot remains. An independent second-pass agent was considered and
rejected: same model, marginal benefit. The restructured tests' chief
defense is the canonical Outcome test, which is self-evaluable as an
explicit question rather than as a pattern match. The blind spot
shrinks because the AI is now answering "can an observer verify
this?" rather than "does this look like a familiar idiom?"

## Mechanism 3 — Stack-skills-invocation lock-in

Closes the modern-best-practice-not-applied failure mode (defect a
in Context, on the generate side).

### Current state

The Step 2.5 mechanism and the `kind`/`condition` manifest schema
from [2026-05-25-stack-skills-invocation-design.md](2026-05-25-stack-skills-invocation-design.md)
appear to be applied to the skill files:

- [skills/generate-component/SKILL.md](../../../skills/generate-component/SKILL.md)
  Step 2.5 ("Invoke stack skills") exists.
- [skills/setup/references/stack-skills.md](../../../skills/setup/references/stack-skills.md)
  carries the `kind` and `condition` columns.

The corresponding open issue,
[docs/issues/2026-05-25-generate-component-stack-skills-uninvoked.md](../../issues/2026-05-25-generate-component-stack-skills-uninvoked.md),
remains `open` because the design's Verification section was never
run end-to-end. The mode-toggle case is the natural canonical
regression for that verification.

### Two-tier risk

The mode-toggle defect implies two latent gaps Mechanism 3 must
address together:

1. **Invocation gap (primary).** Step 2.5 may not be firing in
   practice. Verification by tool-call trace inspection.
2. **Consultation gap (secondary).** Even if Step 2.5 fires and the
   skill bodies load into context, the downstream Step 8 (Write
   files) may not actually consult the loaded knowledge when picking
   implementation patterns. The current "delegated to the stack
   skills" wording is vague on this.

### Additions

**A. Mode-toggle regression scenario.** Add to the existing
invocation design's Verification section a new scenario:

> **Scenario M (regression) — mode-toggle, mount-guard prevention.**
>
> Run `/gyre:setup` → `/gyre:clarify-component "Build a mode-toggle
> component that switches between dark mode and light mode."` →
> `/gyre:generate-component mode-toggle`.
>
> Expected Step 2.5 tool-call trace (in manifest row order): five
> `always` Skill invocations — `vercel-react-best-practices`,
> `vercel-composition-patterns`, `next-best-practices`,
> `web-design-guidelines`, `building-components`.
> `next-cache-components` skipped (no caching mention in spec).
> `shadcn` skipped (external; auto-activates separately if the
> `Where` decision is `Compose`). `agent-browser` skipped (deferred).
>
> Expected emitted component file: NO `useState(mounted)`,
> NO `useEffect(() => setMounted(true), [])`, NO
> `if (!mounted) return placeholder` branch. The component uses
> CSS-only Tailwind `dark:` variants. (The host root layout's
> `<html suppressHydrationWarning>` is out of generate's scope;
> the relevant stack skill may surface it as a
> `host-setup-required` Plan row if absent.)
>
> If the trace shows Step 2.5 invocations but the emitted code still
> contains the mount-guard branch, this is the **consultation gap**.
> File a new issue under `docs/issues/` recording the gap; do not
> resolve the existing invocation issue.

**B. Consultation deference paragraph.** Add to
[skills/generate-component/references/output-format.md](../../../skills/generate-component/references/output-format.md)
(immediately after the existing `## Comments` section, before
`## Collision prompt`) the paragraph:

> ### Pattern selection deference
>
> When a code-form decision involves a pattern covered by a loaded
> stack skill (React hydration handling, Next.js RSC boundary
> placement, accessibility primitives, component composition
> patterns), the writer defers to the loaded stack skill's
> prescription over general knowledge of the same domain. When two
> loaded skills disagree, the more-specific skill wins (e.g.,
> `next-best-practices` over `vercel-react-best-practices` for
> Next.js-specific concerns).
>
> This rule operationalizes the existing "delegated to the stack
> skills installed by `setup`" language: invocation alone (Step 2.5)
> does not guarantee consultation at write time; this deference rule
> is the consultation half.

**C. Open issue updates.** Edit
[docs/issues/2026-05-25-generate-component-stack-skills-uninvoked.md](../../issues/2026-05-25-generate-component-stack-skills-uninvoked.md):

- **Remediation** section: replace the current "open. The fix
  requires a design decision before code changes" text with a
  pointer to this design plus the existing invocation design,
  stating that Step 2.5 implementation and the consultation
  deference paragraph together close the gap.
- **Verification** section: add the Scenario M run as the closing
  test, listing the tool-call trace expectations and the emitted
  code expectations from §A.
- **Status** remains `open` until Scenario M passes on a real run;
  transition to `resolved` only on a passing run with the trace
  documented.

### Scope (Mechanism 3 only)

In scope: §A, §B, §C above.

Out of scope:

- New stack skill manifest entries.
- Changes to Step 1.5 (precondition) or Step 7 (realize
  prerequisites).
- Per-spec invocation choices beyond the existing `always` /
  `conditional` classification.

### Residual risk

If Scenario M runs and the trace shows Step 2.5 invocations but the
emitted code still contains the mount-guard, the consultation
deference paragraph (§B) has not been sufficient. The further
possibility — Step 2.5 fires, the consultation deference rule
applies, but the loaded skill bodies do not actually carry guidance
on hydration / mount-guard / theme handling — would be a **stack
skill content gap**: a third structural defect Gyre does not own
(the skills are external). The protocol for that case is a new
issue, not a change to this design.

## Reference file changes

The following edits implement this design. Each line names the file,
the change, and the mechanism it serves.

- [skills/clarify-component/SKILL.md](../../../skills/clarify-component/SKILL.md)
  — rewrite the Language section per the overarching language
  policy. M1.
- [skills/clarify-component/SKILL.md](../../../skills/clarify-component/SKILL.md)
  Step 10 self-check item 4 — replace the string-matching criteria
  with the three semantic tests; reference dimensions.md and
  output-format.md as the canonical statement. M2.
- [skills/clarify-component/references/dimensions.md](../../../skills/clarify-component/references/dimensions.md)
  — rewrite the `Note on requirements-only voice for ## How` to lead
  with the canonical Outcome test and supply the three tests with
  bilingual example sets. M2.
- [skills/clarify-component/references/output-format.md](../../../skills/clarify-component/references/output-format.md)
  — rewrite the `### ## How voice rule` subsection to match
  dimensions.md verbatim. Add a new short subsection "Spec language
  rule" stating the all-English spec policy. M1, M2.
- [skills/generate-component/SKILL.md](../../../skills/generate-component/SKILL.md)
  — rewrite the Language section per the overarching language
  policy. Add the spec-body non-ASCII validation pass to Step 2
  with the termination message specified in Mechanism 1 §Generate-side
  safety net. M1.
- [skills/generate-component/references/output-format.md](../../../skills/generate-component/references/output-format.md)
  — insert the `### Pattern selection deference` paragraph between
  `## Comments` and `## Collision prompt`. M3.
- [docs/superpowers/specs/2026-05-25-clarify-component-silent-defaults-design.md](2026-05-25-clarify-component-silent-defaults-design.md)
  — add to the `## Principle` section's permitted-shapes list the
  fourth shape "AI translation of the user's natural-language answer
  into English" with the qualification statement from Mechanism 1
  §Relation to silent-defaults. M1.
- [docs/superpowers/specs/2026-05-25-stack-skills-invocation-design.md](2026-05-25-stack-skills-invocation-design.md)
  — add Scenario M to the `## Verification` section per
  Mechanism 3 §A. M3.
- [docs/issues/2026-05-25-generate-component-stack-skills-uninvoked.md](../../issues/2026-05-25-generate-component-stack-skills-uninvoked.md)
  — Remediation and Verification updates per Mechanism 3 §C. M3.

The manifest at
[skills/setup/references/stack-skills.md](../../../skills/setup/references/stack-skills.md)
is **unchanged**: no new rows, no `kind`/`condition` adjustments.

## Verification

Three new scenarios plus the regression scenario in Mechanism 3 §A
together cover this design.

### Scenario A — language policy round-trip

Run `/gyre:clarify-component "toggle button that switches the
theme"`, supplying the request and each dimension's answer in a
non-English user language.

Expected: every section of the produced spec body is in English
(Summary, Purpose, descriptions, code-bound string literals,
rationale, edge cases). No non-ASCII characters present in any
section's content. Section headings remain English per the existing
output-format.md template.

Run `/gyre:generate-component <kebab-name>` against the spec. Step 2
spec-validation passes (no non-ASCII content). The emitted component
file contains no non-English string literals.

### Scenario B — voice rule semantic test fires on idiomatic implementation prose

In the same run as Scenario A, when the dialogue reaches `## How →
Edge cases`, answer with prose that paraphrases "show an empty box at
initial render and the icon after mount" without using any of the
listed anchor tokens verbatim.

Expected: the authoring-time check applies the three semantic tests
to the answer.

- Outcome test: "show the icon after mount" is not observable from
  output → fails.
- Procedural test: "at initial render ... after mount" is a sequence
  → fails.
- Mechanism citation test: "mount", "after mount" are mechanism
  citations → fails.

The AI returns to dialogue with the existing rephrase question. If
the user answers "no layout shift across SSR/CSR for this control",
the bullet is written into spec `## How → Edge cases` as
"No layout shift across the SSR → CSR transition for this control."
(English, per Mechanism 1.)

If the user answers "the specific pattern is required because all
other components in this project use it", the bullet is written into
`## How → Edge cases` as the underlying requirement, and a separate
bullet appears in `## Implementation hints` with the pattern and the
user's stated reason — in English per Mechanism 1 (the AI translates
the user's rationale).

### Scenario C — stack-skills invocation trace

Re-run the three scenarios already specified in the invocation
design's Verification section (simple display, Compose, caching).
Confirm the expected tool-call traces in the run trace.

### Scenario M — mode-toggle regression (per Mechanism 3 §A)

Final composite scenario combining all three mechanisms. Passing
Scenario M is the closing condition for the open invocation issue.

## Out of scope

- **Retroactive cleanup of existing specs.** Legacy specs (notably the
  mode-toggle spec produced before this design) may contain
  user-language prose in non-`## How` sections. They are not
  migrated. Re-running clarify produces a clean spec; bulk migration
  is not built.
- **i18n key emission.** Host i18n discovery (detecting `next-intl`,
  `react-i18next`, etc., and emitting i18n keys instead of inline
  string literals) is deferred. The current design assumes inline
  English literals are correct for the host's source-code language.
- **Translation quality calibration.** AI's user-language → English
  translation quality is not measured or calibrated by this design.
  The user can adjust the emitted code if a specific phrasing is
  required.
- **Stack skill content gaps.** If `vercel-react-best-practices` or
  `next-best-practices` does not actually carry guidance on
  hydration / mount-guard / theme handling, Mechanism 3 alone cannot
  prevent the mount-guard emission. That is a separate defect, owned
  by the stack skills' upstream maintainers, and is recorded as a new
  issue if it surfaces (per Mechanism 3 §Residual risk).
- **agent-browser activation.** Remains `deferred` per the existing
  invocation design.

## Follow-ups

- **Voice-rule over-rejection telemetry.** If the restructured tests
  reject legitimate requirement-shaped bullets (false positives —
  bullets that mention API surfaces the spec must reference, like
  `aria-expanded` in an accessibility bullet), log the offending
  bullets as new issues. The calibration target is the test
  wording in dimensions.md and output-format.md.
- **Voice-rule under-rejection telemetry.** Likewise for bullets
  that pass the three tests but represent implementation prose in
  phrasings the example sets do not anchor. The calibration target
  is the example sets, not the tests themselves.
- **Safety-net termination rate.** Track how often
  `generate-component` Step 2 terminates due to non-ASCII spec
  content. A high rate suggests either users editing specs after
  clarify or clarify's translation step missing content; either is
  worth investigating.
- **Consultation gap monitoring.** Scenario M passes the first time
  on the strength of (i) Step 2.5 firing AND (ii) Step 8 actually
  consulting the loaded body. If the gap reopens in future runs
  (regression), the consultation deference paragraph may need
  per-pattern specificity (e.g., naming hydration handling
  explicitly) rather than the current category-level wording.

## Related

- Open issues this design closes (via verification):
  - [`docs/issues/2026-05-25-generate-component-stack-skills-uninvoked.md`](../../issues/2026-05-25-generate-component-stack-skills-uninvoked.md)
    (via Scenario M).
- Open issues this design partially closes (re-opening case):
  - [`docs/issues/2026-05-25-clarify-component-implementation-leaks-into-spec.md`](../../issues/2026-05-25-clarify-component-implementation-leaks-into-spec.md)
    was marked `resolved` by the silent-defaults design, but the
    mode-toggle re-run shows the resolution was insufficient for
    semantically equivalent prose outside the closed reject-anchor
    set. Mechanism 2 extends that resolution; the issue's status is
    re-evaluated after Scenario B passes.
- Prior designs this design extends:
  - [`2026-05-25-clarify-component-silent-defaults-design.md`](2026-05-25-clarify-component-silent-defaults-design.md)
    — Principle extended with shape (4); voice rule restructured
    per Mechanism 2.
  - [`2026-05-25-stack-skills-invocation-design.md`](2026-05-25-stack-skills-invocation-design.md)
    — Verification extended with Scenario M.
- Prior design unaffected:
  - [`2026-05-25-external-package-knowledge-design.md`](2026-05-25-external-package-knowledge-design.md).
- Skill files this design edits (see §Reference file changes for
  the full list):
  - [`skills/clarify-component/SKILL.md`](../../../skills/clarify-component/SKILL.md)
  - [`skills/clarify-component/references/dimensions.md`](../../../skills/clarify-component/references/dimensions.md)
  - [`skills/clarify-component/references/output-format.md`](../../../skills/clarify-component/references/output-format.md)
  - [`skills/generate-component/SKILL.md`](../../../skills/generate-component/SKILL.md)
  - [`skills/generate-component/references/output-format.md`](../../../skills/generate-component/references/output-format.md)
- Manifest (intentionally unchanged):
  - [`skills/setup/references/stack-skills.md`](../../../skills/setup/references/stack-skills.md)
