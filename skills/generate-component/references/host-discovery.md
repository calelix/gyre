# Host Discovery

This file is read by `generate-component` Step 4 to map host context into Plan slots. It owns only the **slot policy** — which kinds of host context the Plan needs, and which Plan slot label is used when discovery cannot resolve a slot. The concrete *stack conventions* used to perform each discovery (component-library config file structure, type-system path resolution, primitive file naming, story-framework installation signals, stylesheet structure) are delegated to the stack skills installed in the host by `setup` (see [`../../setup/references/stack-skills.md`](../../setup/references/stack-skills.md) for the current manifest). The runtime AI consults those stack skills for the concrete details of each lookup. Stack-skill presence is guaranteed by `generate-component`'s Step 1.5 precondition; the host's project-specific state (components catalog, active design tokens) is consulted alongside the stack skills for context the skills do not own, not as a fallback when they are missing. This file does not duplicate or pin that knowledge. The call site that brings each stack skill's body into context is `generate-component` Step 2.5 ("Invoke stack skills"), which consults the manifest's `kind` and `condition` columns to decide whether to invoke each entry via the Skill tool.

---

## Slot policy

Each row below names a kind of host context the Plan needs and the slot label used when discovery cannot resolve it. The "What discovery resolves" column states the abstract target, not the stack-specific procedure.

| Slot | What discovery resolves | Label on resolve failure |
|---|---|---|
| Component output directory | The on-disk directory where the generated component file is written. For a `Compose` spec whose Reference is a primitive, the primitives directory is used instead of the components directory. | `user-needed` — the directory could not be determined from host configuration. |
| Reference primitive location (Compose only) | For each identifier in the spec's `## Where → Reference`, the file that defines it and its export form (used by the import statement). | `shadcn-install` — the `shadcn` skill installs the primitive after Plan approval. |
| Active design tokens | The token values currently in effect in the host (read from `DESIGN.md` if present — a project convention from [`docs/foundation.md`](../../../docs/foundation.md) — and from the host's active stylesheet). | `attention` — conflicts with the spec's `## Look → Token overrides` are listed; they do not block Plan approval. |
| External dependencies | Two responsibilities: (a) whether each external module role the spec requires (icon library, theme provider, animation library) is present in the host's dependencies, and (b) for each such role, the current export name to use in the emitted import — resolved by reading the package's installed type definitions per the algorithm in *External dependencies — symbol resolution* below. | `npm-install` — the host's package manager installs the missing package after Plan approval. (Symbol resolution runs after the install prerequisite is realized, so the package's files are guaranteed to be present.) |
| Consumer-implied preconditions | For each external symbol the component will import, the host-side conditions that must hold for the symbol to function (context providers in the tree, HTML attributes on root elements, required config flags or environment). Derived from the spec body plus the stack skill bodies already loaded by Step 2.5, per the algorithm in *Consumer-implied preconditions — derivation and verification* below. The optional `## Host preconditions` section in the spec, when present, contributes additional preconditions that are merged with the derived set (deduplicated by requirement text). | `host-setup-required` — non-blocking; the user can approve the Plan with the precondition unmet. |
| Storybook precondition | Whether a story framework is installed in the host. | **Not a Plan item.** When the story framework is not detected, the skill terminates before composing the Plan and tells the user it must be installed first. See SKILL.md Step 4 termination branch. |

---

## External dependencies — symbol resolution

For each external package the spec depends on, and for each abstract role the spec names from that package (e.g., the icon role `moon`), the symbol-name to emit is selected by the following algorithm. The package is guaranteed to be present in `node_modules` at this point — symbol resolution runs after the `npm-install` prerequisite is realized.

1. **Candidate generation.** Produce ordered candidate symbol names from convention-aware heuristics. For icon libraries, prefer the `<Capital>Icon` form (e.g., `MoonIcon`), fall back to `<Capital>` (e.g., `Moon`). For other libraries, the AI generates candidates from its knowledge of the package's naming convention.
2. **Type-definition check.** Glob and Read `node_modules/<pkg>/**/*.d.ts` in the host. Locate export sites for each candidate. Recognize submodule entries (e.g., `/ssr` for RSC contexts) by their `package.json` `exports` field.
3. **Deprecation check.** For each candidate that appears in the types, inspect surrounding JSDoc for an `@deprecated` tag. Tagged candidates rank below untagged ones.
4. **Selection.** Pick the highest-ranked candidate that exists and is not deprecated. If only deprecated candidates exist, pick the highest-ranked deprecated one and surface the deprecation note inline in the resulting Plan row.
5. **Submodule selection.** When the spec's `## How → Interactions` and `## How → States` contain no client-only APIs (no hooks, no event handlers, no browser APIs), the component runs in a Server Component context; prefer a submodule export (e.g., `@phosphor-icons/react/ssr`) when the package's `exports` field advertises one. When client-only APIs are present, use the main module export.

Each resolved external symbol becomes one row in the Plan's Items table:

| slot | value | source |
|---|---|---|
| `Import: <role> from <pkg>` | `<SymbolName>` | `host` |

The `value` cell may carry a parenthetical note when relevant: `SunIcon (host types; "Sun" available but @deprecated)`, or `SunIcon from @phosphor-icons/react/ssr (host types; RSC context)`.

---

## Consumer-implied preconditions — derivation and verification

Input: the parsed spec body (from Step 2), the resolved external symbols from the extended External dependencies slot above, the stack skill bodies loaded into context by Step 2.5, and the spec's optional `## Host preconditions` section when present.

For each external symbol the component will import:

1. **Precondition derivation.** State, in one or two sentences per item, what the host must supply for the symbol to behave as the spec assumes. Drawn from the loaded stack skill bodies plus general knowledge of the imported package.
2. **Signal specification.** For each derived precondition, produce one concrete verification signal — a string to find in a file, an attribute on an HTML element, a key in a JSON config. The signal is small, deterministic, and named in the Plan row so the user can reproduce the check.
3. **Signal verification.** Verify each signal against host files using Read and Glob.
4. **Plan row composition.** Verified-met preconditions produce no row. Verified-unmet preconditions produce one row each with label `host-setup-required`, of the form:

   | slot | value | source |
   |---|---|---|
   | `Precondition: <one-line requirement>` | `<signal description> — not found` | `host-setup-required` |

5. **Conservative disclosure for uncertain cases.** When confidence on whether a precondition applies is low (the imported symbol is unfamiliar, the stack skill bodies do not cover it, the host's shape is ambiguous), the precondition is **still** added to the Plan with `host-setup-required`, and the row's `value` cell carries the suffix `— AI confidence low, user confirmation requested`. This biases toward false positives (over-disclosed rows) rather than false negatives (silent runtime failures).

When the spec contains a `## Host preconditions` section, each bullet in that section is also fed through steps 2–4 (its `requirement: signal` pair plugs directly into signal verification). Spec-stated preconditions are merged with derived ones, deduplicated by requirement text — if the same requirement is both stated in the spec and derived by the AI, only one row appears.

---

## Plan slot labels

The five labels below describe slots that discovery could not fully resolve. They appear in both this file and `plan-format.md`; the wording is identical in both places.

- `user-needed` — discovery could not resolve the slot; the user must supply or approve a value before Plan approval can proceed.
- `shadcn-install` — a Reference primitive is not in the host; the `shadcn` skill will be invoked to install it after Plan approval.
- `npm-install` — an external dependency the spec requires is not in `package.json`; the host's package manager will install it after Plan approval.
- `attention` — a non-blocking discrepancy the user should know about (most commonly a spec/`DESIGN.md` token conflict).
- `host-setup-required` — a runtime precondition for an imported symbol is not met in the host. Non-blocking — the user can approve the Plan with the precondition unmet. See `plan-format.md` for the full definition and approval semantics.

The other three labels (`spec`, `host`, `inferred`) are defined in `plan-format.md` since they describe Plan items that need no remediation action.
