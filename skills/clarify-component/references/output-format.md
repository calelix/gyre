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

The three sub-sections of `## How` that describe observable behavior — `Interactions`, `Edge cases`, `Accessibility` — follow a requirements-only voice. A bullet is rejected when it contains:

1. **Library or API names** in the body prose (`useEffect`, `useState`, `next-themes`, etc.) that the AI authored. Library names quoted verbatim from the user's request are not rejected.
2. **Procedural steps** ("...then ...", "first ... after which ...").
3. **Citations of specific implementation idioms** ("mount-guard pattern", "render prop" used as a how-to instruction).

The rule does not reject domain-level outcome statements like "the dropdown opens on click" or "focus moves to the first item" — these describe the observable result, not the means of achieving it. Likewise, ARIA attributes like `aria-expanded="true"` are observable outcomes the component must produce, not implementation choices.

When a bullet trips the rule, the AI returns to dialogue with a single rephrase question, and the user's answer routes to one of two outcomes: rewrite the bullet as the underlying requirement, or — when the user confirms the specific pattern itself is the requirement — record the prose under the optional `## Implementation hints` section above, with the user's stated rationale.

Enforcement happens at the *Self-check* step (Step 10 in `SKILL.md`); the rule's authoring-time and self-check-time application are also described in `dimensions.md` *Note on requirements-only voice for `## How`*.

### Open Questions

There is no `Open Questions` section. Every captured item must be a resolved decision. If the user could not answer a question, the AI must have proposed a default for the user to explicitly accept or reject, or re-cast the question as multiple choice, or offered concrete examples to disambiguate intent. Acceptance of a proposed default is a resolved decision.

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
