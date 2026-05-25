# Output Format

This file is read by `generate-component` Step 8 to govern the two files the skill emits for every successful run:

- `<components-alias>/<kebab-name>.tsx` — the component file
- `<components-alias>/<kebab-name>.stories.tsx` — the story file

This file owns only **spec → code mapping** (which section of the spec becomes which piece of the emitted code), the **imports rule**, the **comments rule**, and the **collision prompt**.

Everything about the form of the code itself — the framework's component conventions, the language's type-declaration syntax, the story framework's file format, how a primitive is imported in this stack, how a hook is used, how a className is composed — is **stack convention**, delegated to the relevant stack skills installed in the host by `setup` (see [`../../setup/references/stack-skills.md`](../../setup/references/stack-skills.md) for the current manifest). The runtime AI consults those stack skills (and, where they do not speak, the host project's existing patterns) for any code-form decision. This file does not duplicate or pin that knowledge. The call site that brings each stack skill's body into context is `generate-component` Step 2.5 ("Invoke stack skills"), which consults the manifest's `kind` and `condition` columns to decide whether to invoke each entry via the Skill tool.

---

## Spec → code mapping

This is the only mapping rule this skill owns. Each section of the spec becomes a specific piece of the emitted code; the *form* of that piece follows stack convention.

| Spec section | Becomes in `<kebab-name>.tsx` |
|---|---|
| `## What → Props` | The component's props (names, types, defaults, required-ness preserved verbatim). |
| `## What → Purpose` and `## What → Usage scenarios` | Drive the component body's behavior (what handlers do, what state is read). |
| `## How → Interactions` | Event handlers and the transitions/animations they trigger. |
| `## How → Accessibility` | ARIA attributes, semantic element choice, focus-handling. Dynamic ARIA values (changing with state) are computed in the render path. |
| `## How → Edge cases` | Conditional rendering, reduced-motion accommodations, RTL handling, and similar. |
| `## Look → Token overrides` | Token-bearing prop values (e.g., the class-name slot the host stack uses). If the override conflicts with `DESIGN.md`, the conflict appears as `attention` on the Plan, not silently in code. |
| `## Where → Reference` | Import statement(s) for the referenced primitive(s); the component body composes on top of them. |
| `## Interaction model` (when present) | The component's overall structural shape (primitive composition, event topology, exposed handlers). The chosen variant constrains which primitives the body composes and which handlers are exposed via props. When the section is absent from the spec, the AI infers the structural shape from `## What` and `## How` as it does today. |

| Spec section | Becomes in `<kebab-name>.stories.tsx` |
|---|---|
| `## How → States` (named, non-default, not inherited) | One story per state. States explicitly described as "inherited from" a Reference primitive without component-specific behavior are not given separate stories. |
| `## How → Interactions` / `## Data / Content` revealing a context-provider dependency | One story per visually-distinct provider configuration (sourced `inferred` on the Plan), with the provider supplied inline at the story site. |

Identifiers and string literals from the spec are preserved verbatim. If the spec writes a prop description in Korean, that description is preserved as is. If the spec fixes an English prop name, it is used as is.

---

## Imports

External symbols are imported from the paths the Plan's items table records (the import map). The Reference primitive uses the export form (default or named) that host discovery recorded on the Plan; the corresponding import statement form follows the stack's standard syntax.

External symbol *names* (not just paths) are also resolved by host discovery. The Plan's `Import: <role> from <pkg>` rows record the specific exported names selected from each package's installed type definitions, taking deprecation status into account. The selection algorithm (candidate generation, type-definition check, deprecation check, submodule selection) lives in [`host-discovery.md`](./host-discovery.md) under *External dependencies — symbol resolution*. Emit uses exactly the names the Plan recorded.

The import map is the single source of truth for paths. Do not hand-edit imports to "fix" them during Step 9's self-repair loop without recomposing the Plan.

---

## Implementation hints handling

When the spec carries an optional `## Implementation hints` section (see [`../../clarify-component/references/output-format.md`](../../clarify-component/references/output-format.md)), each bullet is a user-stated implementation pattern with a stated rationale. The hint is **a strong recommendation, not a contract**:

- The hint is followed by default during code emission.
- When a loaded stack skill (e.g., `vercel-react-best-practices`, `next-best-practices`) prescribes a different pattern for the same underlying requirement, the stack skill's prescription wins. The emitted code follows the stack prescription, not the hint.
- Each divergence produces a Plan row with the `attention` label, of the form:

  | slot | value | source |
  |---|---|---|
  | `Hint divergence: <hint text>` | `Stack skill recommends <stack pattern>; hint not followed.` | `attention` |

- The user can amend the divergence row at Plan approval. Amending it to "follow the hint anyway" is a spec-level change (per the existing spec-contradiction termination rule in `plan-format.md`) and the skill directs the user back to `clarify-component` to update the spec.

This contrasts with `## Host preconditions` (see `host-discovery.md` → *Consumer-implied preconditions — derivation and verification*), which is a contract: its bullets produce `host-setup-required` rows on mismatch. Implementation hints are non-contract; their divergence is informational and resolved by user judgment.

---

## Comments

No comments are added by default. The generated file contains no doc-block comments, no banner comments, and no "generated by" markers.

The single exception: when the spec's `## How → Edge cases` or `## Look` section carries a constraint whose intent would be lost on a future reader without annotation, exactly one short comment is added at the relevant site. The comment is a single line, never a paragraph.

This rule is a project-wide convention and is **not** delegated to stack skills — it stands regardless of what stack documentation suggests for doc-block or banner conventions.

---

## Collision prompt

When Step 8 resolves the target path and finds that a file already exists at that location, the skill stops before writing and presents this message to the user:

> "A file already exists at `<components-alias>/<kebab-name>.tsx`. What would you like to do?
>
> - **Overwrite** — replace the existing file at that path (its current contents will be lost).
> - **Pick a different name** — supply a new `<kebab-name>` and I will re-check for a collision at the new path."

The same check applies to the `.stories.tsx` path. If either path is occupied the skill pauses for user direction before writing either file.
