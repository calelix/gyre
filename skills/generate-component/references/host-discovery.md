# Host Discovery

This file is read by `generate-component` Step 4 to map host context into Plan slots. It owns only the **slot policy** — which kinds of host context the Plan needs, and which Plan slot label is used when discovery cannot resolve a slot. The concrete *stack conventions* used to perform each discovery (component-library config file structure, type-system path resolution, primitive file naming, story-framework installation signals, stylesheet structure) are delegated to the stack skills installed in the host by `setup` (see [`../../setup/references/stack-skills.md`](../../setup/references/stack-skills.md) for the current manifest). The runtime AI consults those stack skills (or, where they do not speak, the host project's existing patterns) for the concrete details of each lookup. This file does not duplicate or pin that knowledge.

---

## Slot policy

Each row below names a kind of host context the Plan needs and the slot label used when discovery cannot resolve it. The "What discovery resolves" column states the abstract target, not the stack-specific procedure.

| Slot | What discovery resolves | Label on resolve failure |
|---|---|---|
| Component output directory | The on-disk directory where the generated component file is written. For a `Compose` spec whose Reference is a primitive, the primitives directory is used instead of the components directory. | `user-needed` — the directory could not be determined from host configuration. |
| Reference primitive location (Compose only) | For each identifier in the spec's `## Where → Reference`, the file that defines it and its export form (used by the import statement). | `shadcn-install` — the `shadcn` skill installs the primitive after Plan approval. |
| Active design tokens | The token values currently in effect in the host (read from `DESIGN.md` if present — a project convention from [`docs/foundation.md`](../../../docs/foundation.md) — and from the host's active stylesheet). | `attention` — conflicts with the spec's `## Look → Token overrides` are listed; they do not block Plan approval. |
| External dependencies | Whether each external module role the spec requires (icon library, theme provider, animation library) is present in the host's dependencies. | `npm-install` — the host's package manager installs the missing package after Plan approval. |
| Storybook precondition | Whether a story framework is installed in the host. | **Not a Plan item.** When the story framework is not detected, the skill terminates before composing the Plan and tells the user it must be installed first. See SKILL.md Step 4 termination branch. |

---

## Plan slot labels

The four labels below describe slots that discovery could not fully resolve. They appear in both this file and `plan-format.md`; the wording is identical in both places.

- `user-needed` — discovery could not resolve the slot; the user must supply or approve a value before Plan approval can proceed.
- `shadcn-install` — a Reference primitive is not in the host; the `shadcn` skill will be invoked to install it after Plan approval.
- `npm-install` — an external dependency the spec requires is not in `package.json`; the host's package manager will install it after Plan approval.
- `attention` — a non-blocking discrepancy the user should know about (most commonly a spec/`DESIGN.md` token conflict).

The other three labels (`spec`, `host`, `inferred`) are defined in `plan-format.md` since they describe Plan items that need no remediation action.
