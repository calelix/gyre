---
created: 2026-05-23
updated: 2026-05-23
topic: generate-component
status: design
---

# Design — `/generate-component` Skill

## Goal

A skill that, when run inside a host project, takes a component request
specification produced by `clarify-component` and generates a working custom
UI component file together with a Storybook story file. The user reviews the
result by viewing the new story in Storybook.

The skill realizes the second confirmed slice in
[`docs/slices.md`](../../slices.md):
*"When the user runs a skill in a host project against an existing component
request specification produced by the clarify-component skill, the AI generates
a working custom UI component from that spec, which the user can see on screen."*

It sits one stage downstream of `clarify-component` in the AI Harness pipeline:
the spec is the contract; this skill consumes it and emits code. The skill
does not re-open the dialogue dimensions; if the spec is wrong, the user is
directed back to `clarify-component`.

This skill carries no stack convention knowledge of its own. React / Next.js /
shadcn/ui / the type system / the story framework — anything about the *form*
of a well-formed component file or story file — is delegated to the stack
skills installed in the host by `setup` (see
[`../../../skills/setup/references/stack-skills.md`](../../../skills/setup/references/stack-skills.md)
for the current manifest). The runtime AI consults those stack skills (or, where
they do not speak, the host project's existing patterns) for any code-form
decision. This skill owns only spec → code mapping, host-discovery slot policy,
Plan composition and approval, and the verification protocol.

## Scope

In scope:

- spec entries whose `## Where → Decision` is `New` or `Compose`.
- Discovering host context (component output directory, Reference primitive
  location, active design tokens, external package dependencies, story-framework
  precondition) by reading host files. Stack-specific reading conventions are
  consulted from the relevant stack skill, not encoded here.
- Asking the `shadcn` skill to install the Reference primitive when a Compose
  spec names a primitive the host does not yet have.
- Writing one component file and one story file.
- Verifying the result via the host's typecheck command and story-build
  command, with a bounded self-repair loop.

Out of scope (deferred or excluded):

- spec entries whose `## Where → Decision` is `Extend` or `Reuse`. The
  classifier recognizes these and terminates with a clear message. Extend
  requires reading and modifying existing host source code in ways this slice
  does not commit to; Reuse has nothing to generate.
- Installing or scaffolding the host stack itself — component-library config
  files, the story framework, the styling system, the host's package manager,
  monorepo wiring, etc. If a precondition is missing the skill explains and
  terminates.
- Re-opening clarification dialogue. The skill does not re-interpret the
  spec's decisions; ambiguity is treated as a spec defect and the user is
  pointed back to `clarify-component`.
- Generating demo routes, tests, or component documentation beyond the story
  file.
- Page and `DESIGN.md` generation — separate slices in future iterations.

## Invocation

```
/generate-component <kebab-name>
```

- `<kebab-name>` identifies the spec at
  `docs/gyre/specs/components/<kebab-name>.md`.
- If the argument is omitted, the skill lists the entries under
  `docs/gyre/specs/components/` and asks the user to pick one.
- The skill operates on the current working directory, which is treated as the
  host project root. Like `setup`, it requires a `package.json` to be present
  there.

## Output

After a successful run, the host project contains:

```
<components-alias>/<kebab-name>.tsx          # component source
<components-alias>/<kebab-name>.stories.tsx  # story file
```

- `<components-alias>` is the host project's component output directory
  (or its primitives directory, when the Reference in a Compose spec is a
  primitive). The directory is resolved by host discovery per
  *Flow → Discover host context*; the stack-specific configuration files
  consulted are owned by the relevant stack skill.
- If a file already exists at either target path, the skill does not overwrite
  silently. It tells the user the path is taken and asks whether to overwrite
  or pick a different name (the same pattern `clarify-component` uses for spec
  collisions).

The run may additionally cause:

- New external dependencies added to the host project — only when the spec
  requires a package the host does not yet have, and only after the user
  approves the Plan that lists them.
- A Reference primitive installed by the `shadcn` skill — only when a Compose
  spec names a primitive the host does not have, and only after Plan approval.

The skill writes no other files. Tests, demo routes, and barrel re-exports are
the host's decisions, not this skill's.

## Flow

`/generate-component` executes the following nine steps in order. Each step
has an explicit termination branch on failure.

### 1. Confirm host project

Check that the current working directory contains a `package.json`. If not,
tell the user `generate-component` must be run from a project root and
terminate.

### 2. Load spec

Read `docs/gyre/specs/components/<kebab-name>.md`. Parse the YAML front matter
and the body (Where / What / Look / How / Data / Non-goals). If the file is
absent, tell the user that no spec was found at that path and that they should
run `clarify-component <kebab-name>` to create one, then terminate.

### 3. Classify Where decision

Read the spec's `## Where → Decision` value.

- `New` or `Compose` → proceed to step 4.
- `Extend` or `Reuse` → tell the user this slice handles `New` and `Compose`
  only, and terminate without writing any file. The user is informed that the
  other paths are planned for future iterations.

### 4. Discover host context

Map host context into Plan slots per the slot policy in
`references/host-discovery.md`. The five slot kinds and their resolve-failure
labels are:

- **Component output directory** — the on-disk directory where the generated
  component file is written; failure label `user-needed`.
- **Reference primitive location** (Compose only) — for each identifier in the
  spec's `## Where → Reference`, the file that defines it and its export form;
  failure label `shadcn-install`.
- **Active design tokens** — token values currently in effect (from `DESIGN.md`
  if present, and from the host's active stylesheet); conflicts with the spec's
  `## Look → Token overrides` surface as `attention` items (non-blocking).
- **External dependencies** — whether each external module role the spec
  requires (icon library, theme provider, animation library) is present in the
  host's dependencies; failure label `npm-install`.
- **Storybook precondition** — whether the host's story framework is installed.
  Absence is **not** a Plan item: the skill terminates here and tells the user
  the story framework must be installed first.

The concrete stack conventions used to perform each lookup (configuration file
structure, path resolution, primitive file naming, installation-detection
signals, stylesheet structure) are owned by the stack skills, not by this skill.

### 5. Compose Plan

Assemble a single Plan document combining the spec and the discoveries from
step 4. The Plan's structure is defined in `references/plan-format.md`. It
includes:

- The two output file paths.
- The import map: where each external symbol (Reference primitive, icon,
  theme provider, etc.) is imported from in the host.
- `npm-install` items, if any.
- `shadcn-install` items, if any.
- The set of stories to generate, derived from the spec's
  `## How → States`. At minimum: one `Default` story. Additional stories for
  named, non-default states that are not inherited from a Reference primitive,
  plus context-provider variation stories when the spec reveals a context
  dependency.
- `user-needed` items: decisions that host discovery could not resolve (most
  commonly the output path when host configuration is absent).
- `attention` items: discrepancies that do not block but the user should know
  about (e.g., a spec token override that conflicts with `DESIGN.md`).

### 6. Plan approval

Present the Plan to the user in one message. The user may:

- **Approve as is** → proceed to step 7.
- **Fill in or amend** any item, including `user-needed` items → return to
  step 5, recompose, request approval again.
- **Reject** → terminate without writing files.

If the user requests a change that would contradict the spec (e.g., asks to
change a prop name the spec fixes), the skill does not silently follow. It
stops and explains that this is a spec-level change, asking the user to
update the spec via `clarify-component` and re-invoke. This preserves the
"spec is the contract" boundary.

### 7. Realize prerequisites

For each `shadcn-install` item: invoke the `shadcn` skill (through Claude
Code's Skill tool, not Bash) to install the named primitive. If a
`shadcn-install` fails, report the error and terminate without writing files.

For each `npm-install` item: run the host's package manager to add the
dependency. The runner is resolved using the same logic the `setup` skill
uses. If an install fails, report the error and terminate.

Run prerequisites first, files later — failures here surface before the
skill's own files are written. (A successful `shadcn-install` does mutate the
host tree by adding the primitive's files; an `npm-install` mutates the
host's package metadata and lockfile. Both are explicit Plan items the user
has approved.)

### 8. Write files

Write the component file and the story file at the paths fixed in step 5.
For each file, if the path is already occupied, ask the user whether to
overwrite or pick a different name before proceeding.

File contents follow `references/output-format.md` (spec → code mapping,
imports rule, comments rule, collision prompt). The *form* of the code itself
follows stack convention, delegated.

### 9. Verify and self-repair

Run two verification commands and treat their outputs as the truth:

- **Typecheck.** Resolved by `references/verification.md` — typically a host
  `package.json` script (`typecheck`, `type-check`, or `tsc`), falling back
  to a direct invocation of the stack's typecheck binary via the host's
  package runner.
- **Story-build.** Resolved by `references/verification.md` — a host script
  (`build-storybook`, `storybook:build`), falling back to a direct invocation
  of the stack's story-build binary.

On failure, capture the error output, identify the failing files, and edit
them. After each edit, re-run the failing command.

The self-repair loop terminates when either:

- both commands pass → report success with the generated paths and the host's
  recommended dev command; or
- the most recent attempt produces the same normalized error signature as
  the previous attempt → stop, report the error and the diagnosis ("this
  error pattern repeated, the underlying issue is likely in the spec or in
  host state"), and leave the files in their last state for the user to
  inspect.

The skill does not run the story framework in dev mode. It only builds. The
user runs the dev command themselves to see the result on screen; the
success report includes the recommended command.

## Components

Five files: one `SKILL.md` plus four files under `references/`, following the
pattern established by `clarify-component` and `setup`.

### 1. `skills/generate-component/SKILL.md` — skill body

- Front matter: `name: generate-component`; a `description` formulated as an
  invocation trigger; and `allowed-tools` restricted to what the flow needs:
  `Read`, `Glob`, `Write`, `Edit`, plus `Bash` patterns scoped to the package
  runners and managers the install/verify paths invoke
  (`npx *`, `pnpm *`, `pnpm dlx *`, `bun *`, `bunx --bun *`, `npm *`,
  `yarn *`). The `shadcn` skill is invoked through Claude Code's Skill tool,
  not Bash, so no additional Bash permission is needed for that path.
- Body: the nine-step flow, each step's termination branch, and the
  spec-is-the-contract rule. Details delegate to the four reference files
  below.

### 2. `skills/generate-component/references/host-discovery.md`

Owns only the **slot policy** — which kinds of host context the Plan needs,
and which Plan slot label is used when discovery cannot resolve a slot.
Contains:

- A short *What this file owns vs. delegates* statement, including a pointer
  to the stack-skills manifest.
- The five-row slot policy table (Component output directory; Reference
  primitive location; Active design tokens; External dependencies; Storybook
  precondition) with a "What discovery resolves" column stating the abstract
  target and a "Label on resolve failure" column.
- The four Plan slot labels (`user-needed`, `shadcn-install`, `npm-install`,
  `attention`) defined with wording identical to `plan-format.md`.

The concrete stack conventions used to perform each lookup (configuration
file structure, path resolution rules, primitive file naming, installation
detection signals, stylesheet structure) are **not** encoded here — they
belong to the stack skills.

### 3. `skills/generate-component/references/plan-format.md`

The shape of the Plan presented to the user in step 6:

- A short header summarizing what will be created.
- A table of items: `slot / value / source` where `source` is one of
  `spec`, `host`, `inferred`, `user-needed`, `shadcn-install`, `npm-install`,
  `attention`.
- Story-set rendering rules: one `Default` story always present; one per
  named non-default state in the spec's `## How → States` (excluding states
  explicitly described as inherited from a Reference primitive); one per
  context-provider variation when the spec reveals a context dependency.
- The approval mechanism: natural-language approval, amendment, or rejection.
  No formal command syntax.
- The phrasing rules for the `spec-contradiction` termination branch (when
  the user asks for a change that contradicts the spec).

### 4. `skills/generate-component/references/output-format.md`

Owns only **spec → code mapping**, the **imports rule**, the **comments
rule**, and the **collision prompt**. Contains:

- A short *What this file owns vs. delegates* statement, including a pointer
  to the stack-skills manifest and to the host's existing patterns.
- The **spec → code mapping** — two tables (one per emitted file) stating
  which section of the spec becomes which piece of the code, in abstract
  terms (e.g., `## What → Props` → "the component's props, preserved
  verbatim"; `## How → Accessibility` → "ARIA attributes and semantic
  element choice; dynamic ARIA values are computed at render time").
- The **imports rule**: external symbols are imported from the paths the
  Plan's items table records; the import map is the single source of truth.
- The **comments rule**: no-comments-by-default, with one exception (a
  single-line comment at the relevant site when the spec's
  `## How → Edge cases` or `## Look` carries a constraint whose intent would
  be lost without annotation). This rule is project-wide and is **not**
  delegated.
- The **collision prompt**: the exact English message the AI uses when an
  output path is already occupied (two outcomes: Overwrite / Pick a different
  name).

The *form* of the code itself — single default export, props type naming,
JSX, hooks, the story framework's file format (CSF version, decorator
placement, autodocs), import statement syntax, comment syntax — is **not**
encoded here. It is stack convention, delegated.

No fully-rendered component or story example is included; any code snippet
in this file is a spec-agnostic illustration of one of this skill's own rules
and is never tied to a specific spec in the catalog.

### 5. `skills/generate-component/references/verification.md`

The verification protocol:

- A short *What this file owns vs. delegates* statement: this skill owns the
  command-resolution policy, the self-repair loop logic, the
  error-normalization rule, the repair context, the two termination outcomes,
  and the scope exclusions. The choice of typecheck/story-build tool itself
  and the package-runner forms are stack and ecosystem conventions.
- The resolution order for the typecheck command: a host `typecheck` /
  `type-check` / `tsc` script first (first match wins), then a direct binary
  invocation via the host's package runner.
- The resolution order for the story-build command: a host `build-storybook`
  / `storybook:build` script first (first match wins), then a direct binary
  invocation.
- The self-repair loop: typecheck and story-build are gated sequentially;
  only the failing command is re-run on each repair iteration.
- The normalization rule that decides whether two error outputs count as
  "the same error" for the self-repair loop's termination check — for
  typecheck, the deduplicated set of `<file>:<line>:<col> TS<code>` tokens
  sorted lexicographically by the full token string (with the message text
  up to the first newline); for story-build, the first line matching
  `^[A-Z][a-zA-Z]*Error:`.
- The context the LLM is given when self-repairing (spec, Plan, current file
  contents, previous error). The repair step never receives the dialogue
  history — only the spec file itself (re-read fresh).
- Scope exclusions: the skill never runs the story framework in dev mode
  (the success report includes the recommended dev command instead); unit
  tests are not run even if the host has a `test` script.

## Language

Following the `clarify-component` and `setup` convention:

- The skill body and reference files are written in English.
- At runtime the AI conducts Plan presentation, approval dialogue, and final
  reporting in the user's language, inferred from the user's messages.
- The generated code's identifiers and string literals follow the spec's
  decisions verbatim. If the spec writes a prop description in Korean, that
  description is preserved as is; if the spec fixes an English prop name,
  it is used as is.

## Constraints

- The generated component file and story file must not reference paths or
  vocabulary outside the host project. The same constraint the spec format
  imposes carries through to the code.
- The skill does not generate doc-block comments, banner comments, or
  "generated by" markers. Code is identified by its identifiers; provenance
  is recorded in the spec, not in the file. (This rule is project-wide and
  is **not** delegated to stack skills.)
- The skill does not modify host files other than:
  - the two new files it writes,
  - the host's package metadata (when an `npm-install` is approved on the
    Plan),
  - any files the `shadcn` skill writes during a `shadcn-install`.
  No other host file is touched, including no edits to barrel files,
  index files, or routes.
- The skill never re-enters the clarification dimensions. Spec ambiguity
  surfaces as a termination with a pointer to `clarify-component`.

## Verification of this slice

The skill is prose instructions for the AI; "testing" the slice means
running `/generate-component` against a real spec inside a real host project.

- **Demonstration spec.** [`docs/gyre/specs/components/mode-toggle.md`](../../gyre/specs/components/mode-toggle.md)
  — the only spec currently in the catalog. Where decision is `Compose`,
  Reference is `Button`. This case exercises the Compose path, the
  primitive-resolution branch of host discovery, and a context-provider
  variation story. The skill's own reference files do not embed a worked
  example of this spec — the demonstration is the live `/generate-component
  mode-toggle` run, not an in-file rendering.
- **Demonstration playground.** [`experiments/shadcn`](../../../experiments)
  — currently empty. Before this slice can be exercised end-to-end, the
  playground must be populated as a Next.js 16 + React 19 + shadcn/ui +
  Tailwind v4 + `@workspace/ui` + Storybook project. That bring-up is a
  separate slice or task; it is not the responsibility of `generate-component`.
  Once the playground exists, `/generate-component mode-toggle` is the
  end-to-end demonstration of this slice.
- **Visible Outcomes mapping.**
  - *"The skill that generates a component from a specification is created"*
    — satisfied once `skills/generate-component/SKILL.md` and the four
    reference files are committed.
  - *"When the skill is run against a request specification, a UI component
    is generated"* — satisfied when `/generate-component mode-toggle` in a
    populated `experiments/shadcn` produces `mode-toggle.tsx` and
    `mode-toggle.stories.tsx` at the alias-resolved paths, typecheck passes,
    and the story framework builds without errors.
- **"see on screen" (closed demo line).** The user runs the host's dev
  command (whatever script launches the story framework in dev mode) and
  observes the new `ModeToggle` story rendering, with light↔dark toggling
  functional. The skill does not run dev servers; the success report includes
  the recommended command.

## Open design points (deferred to implementation)

Resolved during initial implementation:

- The exact `description` and `allowed-tools` strings in the `SKILL.md` front
  matter — *resolved* with the additions of `Bash(npm *)` and `Bash(yarn *)`
  on top of the originally listed runner patterns, to support npm and yarn
  hosts.
- The exact phrasing of Plan-presentation lines and termination messages —
  *resolved* in `plan-format.md`.
- The exact name normalization used by the self-repair loop's same-error
  detection — *resolved* in `references/verification.md` (`^[A-Z][a-zA-Z]*Error:`
  for story-build; `<file>:<line>:<col> TS<code>` + message-first-line for
  typecheck).

Still open:

- Whether `shadcn-install` invocations pass any additional flags beyond what
  the `setup` skill already uses. Currently the skill invokes `shadcn`
  through the Skill tool with no extra flags; if a future use case demands
  flags, the open point reopens.

## Source materials

- [`docs/foundation.md`](../../foundation.md) — the pinned tech stack
  (Next.js + shadcn/ui + Tailwind v4 + Storybook) the discovery rules
  assume; the Harness composition stance (no third-party vendoring).
- [`docs/slices.md`](../../slices.md),
  [`docs/select.md`](../../select.md) — the second confirmed slice this skill
  realizes.
- [`docs/obstacles.md`](../../obstacles.md) — obstacle #2 (the verification
  gap) motivates the typecheck + story-build verification step;
  obstacle #3 (existing-codebase context cost) motivates the host-discovery
  step.
- [`docs/superpowers/specs/2026-05-18-clarify-component-design.md`](./2026-05-18-clarify-component-design.md)
  — defines the spec format this skill consumes.
- [`docs/superpowers/specs/2026-05-22-setup-skill-design.md`](./2026-05-22-setup-skill-design.md)
  — defines the package-runner resolution logic this skill reuses, the
  installation channel the `shadcn` skill enters through, and the
  stack-skills manifest this skill delegates stack conventions to.
- [`docs/gyre/specs/components/mode-toggle.md`](../../gyre/specs/components/mode-toggle.md)
  — the demonstration spec for this slice.
