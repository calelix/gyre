---
created: 2026-05-24
---

# Stack-Skills Precondition Gate + Manifest Expansion

## Context

Two issues surfaced while reviewing whether the `setup` skill could be removed:

1. The Harness's "no vendoring of third-party skills" rule
   ([foundation.md](../../foundation.md) — Harness Composition, resolved 2026-05-22)
   stands. Vendoring third-party stack skills into Gyre's repo carries
   real costs (code staleness, license/maintenance burden, blocking the
   host user's own `skills`-CLI update path). The `setup` skill stays.

2. The current contract between `generate-component` and `setup` is **soft**.
   `generate-component`'s host-discovery references the stack skills installed
   by `setup`, but
   [host-discovery.md:3](../../../skills/generate-component/references/host-discovery.md)
   leaves a silent fallback open:

   > "the runtime AI consults those stack skills (**or, where they do not
   > speak, the host project's existing patterns**) for the concrete details
   > of each lookup."

   This is the door through which "warn but proceed → silent quality
   degradation" enters. A user who skips `/setup` still gets a generated
   component, but it is produced without the stack skills that the Harness's
   output quality depends on. This contradicts the foundation's "Quality of
   the output is the top priority" stance.

The fix is to close that door: `generate-component` must refuse to run when
the stack skills the manifest declares are not installed in the host, and
direct the user to `/setup`.

This spec also expands the stack-skills manifest with seven additional
Vercel-published skills the Harness wants to depend on, and documents the
workflow for adding more in the future.

## Goals

- Convert the implicit `setup` → `generate-component` dependency into an
  explicit, hard precondition. No silent fallback path remains.
- Keep the manifest at `skills/setup/references/stack-skills.md` as the
  single source of truth for both install (via `setup`) and gate (via
  `generate-component` Step 1.5).
- Expand the manifest to include the additional Vercel-published skills the
  Harness now depends on.
- Document the procedure for adding a stack skill so adding the next one is
  a one-row edit.

## Non-goals

- Bundling or vendoring third-party stack skills into Gyre's repo. The
  foundation's Harness Composition rule stands.
- Introducing MCP support into the manifest. The current scope is skills
  only; MCPs can be added when concretely needed by extending the manifest
  format.
- Per-spec gating. The gate is **strict** — all manifest entries must be
  installed before `generate-component` runs, regardless of which entries a
  given spec uses. This is a deliberate simplification; see *Open question*
  below.
- Changes to `clarify-component`. It does not consume stack skills.
- Changes to the active slice in
  [docs/select.md](../../select.md). This is a tightening of the active
  slice's existing skill, not a new slice.

## Architecture

### Single source of truth

`skills/setup/references/stack-skills.md` remains the manifest. Both skills
read it dynamically at invocation time:

- `setup` reads it to install missing entries.
- `generate-component` reads it to verify entries are installed, terminating
  if not.

Adding a stack skill = one row appended to the table + commit. Both skills
pick up the new entry without code changes.

### Detection logic — defined once, cited

`setup` Step 4 ("Detect install state")
already defines the detection logic: an entry is **installed** when both of
two signals hold —

- a skill directory exists at
  `.claude/skills/<name>/SKILL.md` OR `.agents/skills/<name>/SKILL.md`, and
- `skills-lock.json` at the host root contains the key `<name>` under its
  `skills` object.

`generate-component`'s new Step 1.5 reuses this logic by citation, not by
duplication. If the detection rules ever change (new install location, new
lockfile schema), they change in one place.

## What changes

### 1. New step in `generate-component`

Insert **Step 1.5: Precondition — stack skills installed** between the
current Step 1 ("Confirm host project") and Step 2 ("Load spec") in
[`skills/generate-component/SKILL.md`](../../../skills/generate-component/SKILL.md).

Behavior:

1. Read the manifest at `../setup/references/stack-skills.md`.
2. For each entry, evaluate the two detection signals **as defined in
   `../setup/SKILL.md` Step 4 (Detect install state)**.
3. Classify each entry as `installed`, `incomplete`, or `missing` — same
   classification used by `setup`.
4. If every entry is `installed` → proceed to Step 2.
5. Otherwise → terminate. The skill must not load the spec, must not
   perform host discovery, must not write any file.

Termination message requirements:

- Conducted in the user's language (consistent with the language policy
  already stated in `generate-component/SKILL.md`).
- Lists each non-`installed` entry, named, with which detection signal is
  missing (skill files / lock entry / both).
- Instructs the user to run `/setup` from this host project root before
  retrying `/generate-component`.
- A failed precondition is never reported as a success — the same posture
  `setup` takes in its own Step 6 ("Verify and report").

### 2. Scope tightening in `generate-component/SKILL.md`

Append to the "Out of scope" list:

> Running when required stack skills are not installed in the host. The
> skill terminates at Step 1.5 with a pointer to `/setup` instead of
> falling back to the host project's existing patterns alone.

### 3. Remove the silent fallback in `host-discovery.md`

In [`skills/generate-component/references/host-discovery.md`](../../../skills/generate-component/references/host-discovery.md),
strike the parenthetical:

> ~~"(or, where they do not speak, the host project's existing patterns)"~~

Replace with a statement that the precondition at Step 1.5 has already
guaranteed the stack skills are present; the runtime AI consults those
stack skills (and the host's existing patterns for project-specific
context the skills do not own — components catalog, design tokens), but
absence of a manifest skill is never reached at runtime.

### 4. Expand the manifest

Replace the Skills table in
[`skills/setup/references/stack-skills.md`](../../../skills/setup/references/stack-skills.md)
with the following set. The first row (`shadcn`) is the existing entry.
Rows 2-8 are added.

| name | source | skill | purpose |
|---|---|---|---|
| `shadcn` | `shadcn/ui` | — | Project-aware shadcn/ui context — CLI usage and composition rules — that component generation relies on. |
| `react-best-practices` | `vercel-labs/agent-skills` | `react-best-practices` | React idioms and anti-patterns the generated component must respect. |
| `composition-patterns` | `vercel-labs/agent-skills` | `composition-patterns` | Component composition rules that shape how primitives are arranged. |
| `next-best-practices` | `vercel-labs/next-skills` | `next-best-practices` | Next.js App Router conventions (RSC boundaries, file conventions, data patterns) the generated component must conform to. |
| `next-cache-components` | `vercel-labs/next-skills` | `next-cache-components` | Next.js Cache Components rules for any cached / RSC-cached behavior the spec requires. |
| `web-design-guidelines` | `vercel-labs/agent-skills` | `web-design-guidelines` | Web interface guidelines (interaction, accessibility, layout) the generated component must meet. |
| `building-components` | `vercel/components.build` | `building-components` | Component construction guidance the generated component must follow alongside the design / interaction guidelines. |
| `agent-browser` | `vercel-labs/agent-browser` | `agent-browser` | Browser automation reserved for future verification phases (visual / interaction / accessibility checks). |

### 5. "How to add a stack skill" section

Append to the bottom of `stack-skills.md`:

```markdown
## How to add a stack skill

1. Append one row to the Skills table above with `name`, `source`, `skill`,
   `purpose`. Use `—` in the `skill` column when the source repo publishes
   a single skill.
2. (Optional) If `generate-component` needs to *use* the new skill's
   knowledge in discovery beyond what existing prose covers, update
   `skills/generate-component/references/host-discovery.md` accordingly.
3. Commit. No other file needs changes — both `setup` (install) and
   `generate-component` (Step 1.5 precondition) read this manifest
   dynamically.
```

## Open question (deferred)

**Per-spec gating vs. strict gating.** The gate as designed requires all
manifest entries be installed before `generate-component` runs, regardless
of which entries a given spec actually consults. With the manifest at 7-8
entries (all expected to be present in any host targeting the Next.js +
shadcn/ui + Tailwind v4 stack the foundation pins), this is acceptable.

If the manifest grows further and entries diverge in applicability (e.g.,
a stack skill needed only by `DESIGN.md` generation, not by component
generation), the gate can be refined to consult only the entries relevant
to the invocation. This is reserved for the future and not part of this
spec.

## Out of scope (deferred or excluded)

- Any change to `clarify-component`.
- Any change to other reference files of `generate-component`
  (`output-format.md`, `plan-format.md`, `verification.md`).
- Any change to `setup`'s install logic, Step 4 detection, or `allowed-tools`.
- MCP support in the manifest.
- Promotion of "generate-component hard-precondition on setup manifest" to
  a Foundation Composition clause. The current scope keeps the rule local
  to the two skills' SKILL files; if a future slice surfaces a need for
  the rule to be a project-level invariant, the foundation update can be
  proposed then.
