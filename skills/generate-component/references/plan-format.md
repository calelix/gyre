# Plan Format

This file is read by `generate-component` Step 5–6 to compose and present the Plan that bridges the spec and the discoveries from Step 4. The Plan is presented in a single message; the user approves, amends, or rejects it before any file is written or installed.

---

## Plan structure

The Plan presented to the user has exactly three sections, in this order:

1. **Header line** — states what will be created: the spec's component name, the two output paths (component file and story file), and the spec's `## Where → Decision` value.
2. **Items table** — each row resolves one slot the component generation depends on (see *Items table* below).
3. **Approval prompt** — a closing prompt offering the user the three outcomes described in *Approval mechanism* below.

---

## Items table

Each row in the table resolves one slot. The table has three columns:

| column | meaning |
|---|---|
| `slot` | What the row decides (e.g., "Output directory", "Reference: Button", "Package: next-themes"). |
| `value` | The resolved value, or a `?` placeholder when the source is `user-needed`. |
| `source` | One of seven labels defined below. |

### Source labels

- `spec` — value comes from the spec; the user changing it would be a spec-level change (see *Spec-contradiction termination*).
- `host` — value discovered by reading host files (e.g., the alias-resolved component directory).
- `inferred` — value chosen by the skill from host signals where no single source is authoritative (e.g., the package runner picked from the lockfile, mirroring `setup`'s logic).
- `user-needed` — discovery could not resolve the slot; the user must supply or approve a value before Plan approval can proceed.
- `shadcn-install` — a Reference primitive is not in the host; the `shadcn` skill will be invoked to install it after Plan approval.
- `npm-install` — an external dependency the spec requires is not in `package.json`; the host's package manager will install it after Plan approval.
- `attention` — a non-blocking discrepancy the user should know about (most commonly a spec/`DESIGN.md` token conflict).

---

## Story-set rendering

The spec's `## How → States` section drives story rows on the Plan. The minimum is one default story (always present). Each non-default named state the spec lists adds a further row — with one exception: states explicitly described as "inherited from" a reference primitive with no component-specific behavior are not made into separate stories; they are exercised through the base primitive's own stories.

In addition, when the spec's `## How → Interactions` or `## Data / Content` section reveals that the component depends on an external context provider (e.g., a ThemeProvider that controls which visual state is visible), the plan includes one story per meaningful provider configuration that produces a visually distinct rendering. These **context stories** are sourced from `inferred`, not `spec`, because they are derived from the implementation requirement, not from a named state in the spec.

Each story row takes the form:

| slot | value | source |
|---|---|---|
| `Story: <state>` | `<StateName>` | `spec` |
| `Story: <context-variant>` | `<ContextVariant>` | `inferred` |

The rows above are templates — the AI substitutes concrete values for the angle-bracket placeholders; this is not literal Plan text. The `value` cell holds the PascalCase named export in `<kebab-name>.stories.tsx` (e.g., `Default`, `Dark`).

The `default` state row uses `slot = "Story: default"`. Named-state rows follow in the order the spec lists them (excluding inherited-only states per the rule above). Context-variant rows follow after, one per distinct provider configuration that produces a visually distinct rendering.

---

## Approval mechanism

The Plan closes with a natural-language prompt. The user responds in their own language; no formal command syntax is required or expected.

Illustrative sentence the AI uses to close the Plan presentation:

> "Approve, amend, or reject this Plan — describe any changes in your own words and I will recompose."

Three outcomes are recognized:

- **Approve as is** — the user confirms the Plan is correct. The skill proceeds to Step 7 (Realize prerequisites).
- **Fill in or amend** — the user resolves a `user-needed` row by supplying the missing value, or amends another row. The skill returns to Step 5 (Compose Plan), recomposes the Plan with the change applied, and re-presents it. This loop continues until the user approves or rejects.
- **Reject** — the user declines the Plan. The skill terminates without writing or installing anything.

---

## Spec-contradiction termination

When a user amendment asks for a change that contradicts the spec — for example, renaming a prop that the spec fixes in `## What → Props`, or choosing a `Where` decision the spec did not make — the skill must not silently follow the request.

The skill must:

1. Not apply the change.
2. Tell the user the requested change is a spec-level change.
3. Direct the user to run `clarify-component` to update the spec, and to re-invoke `generate-component` once the spec reflects the change.
4. Terminate the current run without writing or installing anything.

Illustrative sentence the AI should use:

> "That change would alter the spec's `## What → Props` decision. Please update the spec via `clarify-component` and re-invoke `/generate-component` once the spec reflects the change."
