---
name: generate-component
description: Use when the user wants to generate a working custom UI component file plus a Storybook story file from an existing component request specification produced by clarify-component. Reads the spec at docs/gyre/specs/components/<kebab-name>.md, discovers host context (component output alias, primitive location, dependencies, Storybook setup), presents a Plan for approval, then writes the two files and verifies them via typecheck and Storybook build. Handles spec `Where` decisions `New` and `Compose` only; terminates with a pointer to clarify-component for `Extend` and `Reuse`. Invoked as /generate-component <kebab-name>.
allowed-tools: Read, Glob, Write, Edit, Bash(npx *), Bash(pnpm *), Bash(pnpm dlx *), Bash(bun *), Bash(bunx --bun *), Bash(npm *), Bash(yarn *)
---

# generate-component

A skill that takes a component request specification produced by `clarify-component` and generates a working custom UI component file together with a Storybook story file. The spec at `docs/gyre/specs/components/<kebab-name>.md` is the contract: the skill consumes it and emits code without reopening the clarification dimensions. If the spec is wrong, the user is directed back to `clarify-component` — the skill does not re-interpret, amend, or extend the spec's decisions on its own. The skill realizes the second confirmed slice in [`docs/slices.md`](../../docs/slices.md) and is tracked in [`docs/select.md`](../../docs/select.md): *"When the user runs a skill in a host project against an existing component request specification produced by the clarify-component skill, the AI generates a working custom UI component from that spec, which the user can see on screen."*

## Invocation

```
/generate-component <kebab-name>
```

`<kebab-name>` identifies the spec at `docs/gyre/specs/components/<kebab-name>.md`. If the argument is omitted, the skill lists every `.md` file under `docs/gyre/specs/components/` and asks the user to pick one. The skill operates on the current working directory, which is treated as the host project root. Like `setup`, it requires a `package.json` to be present there.

## Output

After a successful run, the host project contains:

```
<components-alias>/<kebab-name>.tsx          # component source
<components-alias>/<kebab-name>.stories.tsx  # Storybook story
```

`<components-alias>` is the host project's component output directory (or its primitives directory, when the Reference in a Compose spec is a primitive). The directory is resolved by host discovery per `references/host-discovery.md`; the stack-specific configuration files consulted are owned by the relevant stack skill.

If a file already exists at either target path, the skill does not overwrite silently. It pauses, tells the user the path is taken, and asks whether to overwrite or pick a different name before proceeding. Collision handling is specified in `references/output-format.md`.

The run may additionally cause:

- New npm dependencies added to the host `package.json` — only when the spec requires a package the host does not yet have, and only after the user approves the Plan that lists them (`npm-install` items).
- A Reference primitive installed by the `shadcn` skill — only when a Compose spec names a primitive the host does not have, and only after Plan approval (`shadcn-install` items).

Both kinds of host mutation are explicit items on the Plan; neither occurs without user approval.

## Language

The skill body and reference files are written in English. At runtime the AI conducts Plan presentation, approval dialogue, and final reporting in the user's language, inferred from the user's messages. The generated code's identifiers and string literals follow the spec's decisions verbatim — if the spec writes a prop description in Korean, that description is preserved as is; if the spec fixes an English prop name, it is used as is.

## Scope

In scope:

- spec entries whose `## Where → Decision` is `New` or `Compose`.
- Discovering host context (component output alias, primitive source location,
  active design tokens, package dependencies, Storybook setup) by reading
  host files directly.
- Asking the `shadcn` skill to install the Reference primitive when a Compose
  spec names a primitive the host does not yet have.
- Writing one component file and one Storybook story file.
- Verifying the result via TypeScript typecheck and Storybook build, with a
  bounded self-repair loop.

Out of scope (deferred or excluded):

- spec entries whose `## Where → Decision` is `Extend` or `Reuse`. The
  classifier recognizes these and terminates with a clear message. Extend
  requires reading and modifying existing host source code in ways this slice
  does not commit to; Reuse has nothing to generate.
- Installing or scaffolding the host stack itself — `components.json`,
  Storybook, Tailwind, the host's package manager, the `@workspace/ui`
  monorepo wiring, etc. If a precondition is missing the skill explains and
  terminates.
- Re-opening clarification dialogue. The skill does not re-interpret the
  spec's decisions; ambiguity is treated as a spec defect and the user is
  pointed back to `clarify-component`.
- Generating demo routes, tests, or component documentation beyond the story
  file.
- Page and `DESIGN.md` generation — separate slices in future iterations.
- Running when required stack skills are not installed in the host. The skill terminates at Step 1.5 with a pointer to `/setup` instead of falling back to host conventions alone.

## Process

The skill executes the following steps in order. Each step has an explicit termination branch on failure.

### 1. Confirm host project

Check that the current working directory contains a `package.json` file. If it does not, tell the user that `generate-component` must be run from a project root, and terminate without reading the spec or writing anything.

### 1.5. Precondition — stack skills installed

Read the manifest at [`../setup/references/stack-skills.md`](../setup/references/stack-skills.md). For each entry, evaluate the two detection signals **as defined in [`../setup/SKILL.md`](../setup/SKILL.md) Step 4 (Detect install state)**:

- a skill directory carrying `SKILL.md` exists at EITHER `.claude/skills/<name>/SKILL.md` OR `.agents/skills/<name>/SKILL.md`, and
- `skills-lock.json` at the host project root contains the key `<name>` under its `skills` object.

Classify each entry as `installed` (both signals hold), `incomplete` (exactly one holds), or `missing` (neither holds) — the same classification used by `setup` Step 4.

If every entry is `installed`, proceed to Step 2.

Otherwise, terminate. In the user's language:

- name each non-`installed` entry and state which detection signal is missing (skill files / lock entry / both),
- instruct the user to run `/setup` from this host project root and then retry `/generate-component`,
- do not load the spec, do not perform host discovery, do not write any file.

A failed precondition is never reported as success — the same posture `setup` takes in its own Step 6 ("Verify and report").

### 2. Load spec

Read `docs/gyre/specs/components/<kebab-name>.md`. Parse both the YAML front matter and the body sections (Where / What / Look / How / Data / Non-goals). If the file is absent, tell the user that no spec was found at that path and that they should run `clarify-component <kebab-name>` to create one, then terminate.

### 2.5. Invoke stack skills

Read the manifest at [`../setup/references/stack-skills.md`](../setup/references/stack-skills.md). For each row, in the table's row order:

- `kind: always` — invoke the row's `name` via Claude Code's Skill tool.
- `kind: conditional` — evaluate the row's `condition` predicate against the spec body loaded in Step 2. If the predicate is true, invoke the row's `name` via the Skill tool. If false, skip the row.
- `kind: external` — skip. The skill loads through its own auto-activation mechanism; the manifest row exists so that `setup` installs it.
- `kind: deferred` — skip. The manifest row exists so that `setup` installs it for parity with future iterations.

Run the invocations sequentially in row order so the resulting tool-call trace is deterministic for a given spec.

If a Skill invocation errors (skill not found, body load failure), report the failure to the user — naming the row and the error — and terminate without proceeding to Step 3. Step 1.5 should have caught uninstalled entries already, so reaching this branch indicates drift between the install state and the manifest.

Loaded skill bodies remain available to all subsequent steps (Discovery in Step 4, Plan composition in Step 5, file Write in Step 8, Verify in Step 9) without re-invocation.

### 3. Classify Where decision

Read the spec's `## Where → Decision` value.

- `New` or `Compose` → proceed to step 4.
- `Extend` or `Reuse` → tell the user this slice handles `New` and `Compose` only; `Extend` and `Reuse` are planned for future iterations. Terminate without writing any file.

### 4. Discover host context

Map host context into Plan slots per the slot policy in `references/host-discovery.md`. The six slot kinds are:

- **Component output directory** — resolves the spec's output path; failure label `user-needed`.
- **Reference primitive location** (Compose only) — resolves each identifier in the spec's `## Where → Reference`; failure label `shadcn-install`.
- **Active design tokens** — surfaces conflicts with the spec's `## Look → Token overrides`; conflict label `attention` (non-blocking).
- **External dependencies** — checks for the packages the spec requires (icon library, theme provider, animation library) and resolves the current export name to emit for each abstract symbol the spec names (by reading the package's installed type definitions); failure label `npm-install`.
- **Consumer-implied preconditions** — derives, per external symbol the component will import, the host-side conditions that must hold for the symbol to function, verifies each via a host signal; unmet preconditions are surfaced with label `host-setup-required` (non-blocking).
- **Storybook precondition** — checks the host's story framework is installed. Absence is **not** a Plan item: the skill terminates here and tells the user it must be installed first.

The concrete stack conventions used to perform each lookup (configuration file structure, path resolution, primitive file naming, installation-detection signals) are not duplicated here — they belong to the stack skills installed by `setup` (see [`../setup/references/stack-skills.md`](../setup/references/stack-skills.md) for the current manifest).

### 5. Compose Plan

Assemble a single Plan document combining the spec and the discoveries from step 4. The Plan's structure is defined in `references/plan-format.md`. It includes the two output file paths, the import map, all prerequisite actions, story rows derived from the spec's `## How → States`, and any `user-needed`, `attention`, or `host-setup-required` items. The eight source labels that may appear on Plan rows are: `spec`, `host`, `inferred`, `user-needed`, `shadcn-install`, `npm-install`, `attention`, and `host-setup-required`.

### 6. Plan approval

Present the Plan to the user in one message. Three outcomes are recognized:

- **Approve as is** → proceed to step 7.
- **Fill in or amend** any item, including `user-needed` items → return to step 5, recompose the Plan with the change applied, and re-present it. This loop continues until the user approves or rejects.
- **Reject** → terminate without writing or installing anything.

If the user requests a change that would contradict the spec — for example, renaming a prop that the spec fixes in `## What → Props`, or choosing a `Where` decision the spec did not make — the skill must not silently follow. It stops, tells the user the requested change is a spec-level change, and directs the user to run `clarify-component` to update the spec and re-invoke `/generate-component` once the spec reflects the change. Exact phrasing rules for this termination branch are in `references/plan-format.md`. This preserves the spec-is-the-contract rule.

### 7. Realize prerequisites

For each `shadcn-install` item on the approved Plan: invoke the `shadcn` skill through Claude Code's Skill tool (not Bash). If a `shadcn-install` fails, report the CLI error and terminate without writing files.

For each `npm-install` item: run the host's package manager using the same runner-resolution logic the `setup` skill uses (see `../setup/SKILL.md` Step 3). If an install fails, report the error and terminate.

Run all prerequisites before writing any files — failures here surface before the skill's own files are written.

### 8. Write files

Write the component file and the story file at the paths fixed in step 5. For each file, immediately before writing, check whether the path is already occupied. If it is, ask the user whether to overwrite or pick a different name, then act on the answer before writing either file. File contents follow `references/output-format.md`.

### 9. Verify and self-repair

Run the typecheck command, then the Storybook-build command (only when typecheck passes), treating their outputs as the truth. Command resolution, the bounded self-repair loop rules, error-normalization definitions, and the exact context given to the repair step are all specified in `references/verification.md`. The stack-standard form of each command (TypeScript's `tsc`, Storybook's `storybook build`) is not pinned here — it follows the host's stack convention as already encoded by `setup`'s runner-resolution.

The loop terminates on one of two outcomes:

- **Both commands pass** — report success, including the generated file paths and the host's recommended dev command (the script the host uses to launch the story framework in dev mode) so the user can launch it themselves to see the result on screen. Termination: success.
- **The most recent attempt produces the same normalized error signature as the previous attempt for the same command** — the loop cannot make progress. Report the repeated error signature and the diagnosis, and leave the files in their last state for the user to inspect. Termination: failure with report.

The skill never runs `storybook dev`; it builds only.

## Constraints

- The generated component file and story file must not reference paths or
  vocabulary outside the host project. The same constraint the spec format
  imposes carries through to the code.
- The skill does not generate JSDoc, banner comments, or "generated by"
  markers. Code is identified by its identifiers; provenance is recorded in
  the spec, not in the file.
- The skill does not modify host files other than:
  - the two new files it writes,
  - `package.json` (when an `npm-install` is approved on the Plan),
  - any files the `shadcn` skill writes during a `shadcn-install`.
  No other host file is touched, including no edits to barrel files,
  index files, or routes.
- The skill never re-enters the clarification dimensions. Spec ambiguity
  surfaces as a termination with a pointer to `clarify-component`.

## Reference files

These files show spec → code mapping (and the host-discovery / Plan-composition / verification policies that surround it). They contain no stack convention details — React, Next.js, shadcn/ui, the type system, the story framework — those are delegated to the stack skills installed in the host by `setup` (see [`../setup/references/stack-skills.md`](../setup/references/stack-skills.md) for the current manifest).

- `references/host-discovery.md` — slot policy (which kinds of host context the Plan needs, which Plan slot label is used on resolve failure) and the five resolve-failure labels.
- `references/plan-format.md` — the Plan's three-section structure, the eight source labels, story-set rendering, the approval mechanism, and the spec-contradiction termination phrasing rules.
- `references/output-format.md` — the spec → code mapping (which spec section becomes which piece of the emitted code), the imports rule, the no-comments-by-default rule with its single exception, and the collision prompt.
- `references/verification.md` — command resolution policy (which host scripts to look up and the binary fallback), the self-repair loop rules, error-normalization definitions, and the context provided to each repair invocation.

The AI reads these reference files at invocation time and uses them to drive host discovery, Plan composition, file generation, and verification. None of their contents are duplicated in this file.
