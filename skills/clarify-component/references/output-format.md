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
- Interactions: <text covering hover, focus, keyboard, transitions>
- Accessibility: <text covering semantic markup, ARIA, focus order, contrast>
- Edge cases: <text covering long text, RTL, i18n, and any other named edge cases>

## Data / Content
- Schema: <text or table>
- Edge data: <text covering nullable, truncation, optimistic vs confirmed, page sizes, etc.>

## Non-goals
- <at least one bullet>
````

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
