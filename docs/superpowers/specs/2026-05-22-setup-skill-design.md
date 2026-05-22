---
created: 2026-05-22
topic: setup
status: design
---

# Design — `/setup` Skill

## Goal

A skill that, when run inside a host project, installs the external stack skills
that the AI Harness's component-generation pipeline depends on. The skill reads a
manifest of required external stack skills and installs any that are missing,
leaving the host project with those skills present and usable.

The skill realizes the third confirmed slice in [`docs/slices.md`](../../slices.md):
*"When the user runs the setup skill in a host project, the AI installs into that
project the external stack skills that the component-generation skill depends on."*

It exists because of a deliberate composition decision recorded in
[`docs/slices.md`](../../slices.md) (Foundation Update Candidates): the AI Harness
**bundles or vendors no third-party stack skills**. It ships only its own skills.
The external stack skills are installed into the host project **on demand** by
`setup`, and the component-generation skill then references them.

## Core Policy — external stack skills are consumed only as skills

External stack skills are installed **individually, as skills**, into the host
project's skills directory via the `skills` CLI (skills.sh) — the current CLI
places the Claude Code agent's skills under `.claude/skills/`. They are
**never** installed as plugins.

Rationale:

- A plugin (e.g. the Vercel plugin) bundles a fixed, curated subset of skills and
  does not cover the full, community-extensible skills directory — so a
  plugin-based channel is not even a complete alternative.
- Installing a whole plugin to obtain one stack skill drags unrelated skills,
  agents, commands, hooks, and telemetry into the host project. That is heavy
  vendoring, contrary to the "no third-party vendoring" stance of
  [`docs/foundation.md`](../../foundation.md).
- The Harness's dependency is *per skill* (currently the shadcn skill). Skill-level
  installation matches that granularity exactly.

This is a standing constraint, not a deferred feature. If a future stack publishes
its skill *only* in plugin form, that case reopens this decision through a separate
future slice — it is not accommodated by the present design.

## Scope

In scope:

- Reading a manifest of external stack skills the Harness's component generation
  depends on.
- Detecting, for each manifest entry, whether the skill is already installed in
  the host project.
- Installing missing skills via the `skills` CLI.
- Reporting the result to the user (already present / newly installed / failed).

Out of scope (deferred or excluded by policy):

- Plugin-based installation of any kind — **excluded by policy** (see *Core Policy*).
- Installing the `skills` CLI itself — it is run through the host's package runner
  (`npx` / `pnpm dlx` / `bunx --bun`), never installed globally.
- Installing the AI Harness (Gyre plugin) itself.
- Scaffolding the host stack — `components.json`, dependencies, Tailwind config,
  and similar are not touched.
- Invoking or generating components — that is the (future) component-generation
  skill's responsibility.

## Invocation

```
/setup
```

- Takes no arguments. The skill operates on the current working directory, which
  is assumed to be the host project root.
- The skill is callable from any project where the Harness is installed.

## Result

After a successful run, for each manifest entry the host project contains:

```
.claude/skills/<name>/        # the installed skill directory (current CLI)
skills-lock.json              # updated with a <name> entry (hash + source)
```

(An older `skills` CLI layout placed the directory at `.agents/skills/<name>/`;
detection accepts either location — see *Detect*.)

These artifacts are produced and maintained by the `skills` CLI; `setup`
orchestrates the CLI and verifies the artifacts but does not write them directly.

## Components

Two files, following the `SKILL.md` + `references/` pattern established by
`clarify-component`.

### 1. `skills/setup/SKILL.md` — skill body

- Front matter: `name: setup`, a `description`, and `allowed-tools` scoped to the
  `skills` CLI invocations and the read-only inspection the flow needs — e.g.
  `Bash(npx skills *)`, `Bash(pnpm dlx skills *)`, `Bash(bunx --bun skills *)`,
  plus the file inspection the detection step requires.
- Body: the `/setup` flow (below) and the single uniform `{detect, install,
  verify}` procedure. Because consumption is skill-only, there is exactly one
  procedure — no per-channel branching.

### 2. `skills/setup/references/stack-skills.md` — the stack-skills manifest

A reference file declaring the external stack skills the Harness's component
generation depends on. Each entry has the following fields:

| field | meaning |
|---|---|
| `name` | The skill's installed name. Detection key — matched against the skill directory (`.claude/skills/<name>/` or `.agents/skills/<name>/`) and the `skills-lock.json` entry key. |
| `source` | The `skills` CLI source coordinates: a GitHub `owner/repo`. |
| `skill` | Optional. The `--skill <name>` selector, used only when one source repo publishes multiple skills. |
| `purpose` | One line stating why the Harness's component generation needs this skill. |

This iteration's manifest contains a single entry:

| name | source | skill | purpose |
|---|---|---|---|
| `shadcn` | `shadcn/ui` | (none) | Provides project-aware shadcn/ui context — CLI usage and composition rules — that component generation relies on. |

The manifest nominally lists "the skills component-generation depends on," but the
component-generation skill does not exist yet (its slice surfaced the `setup` slice
and was set aside). The dependency is therefore derived from the tech stack pinned
in [`docs/foundation.md`](../../foundation.md) — Next.js + shadcn/ui + Tailwind v4
— not discovered from a skill. The manifest lives under `skills/setup/references/`
because `setup` is its only consumer today. If component generation later needs to
read it as well, it can be promoted to a shared location.

## Flow

`/setup` executes the following steps in order.

1. **Confirm host project.** Verify the working directory looks like a project
   root (a `package.json` is present). If not, tell the user and stop.

2. **Load the manifest.** Read `references/stack-skills.md` to obtain the list of
   required external stack skills.

3. **Determine the package runner.** Read the host `package.json` `packageManager`
   field (falling back to lockfile presence) and select `npx`, `pnpm dlx`, or
   `bunx --bun`. The `skills` CLI is invoked through this runner.

4. **Detect install state.** For each manifest entry, run the **detect** step
   (below). Classify each entry as *already installed*, *missing*, or
   *incompletely installed*.

5. **Install missing skills.** For each entry that is not cleanly installed, run
   the **install** step (below). Entries are independent — a failure on one does
   not abort the others.

6. **Verify and report.** For each installed entry, run the **verify** step
   (below). Report to the user: which skills were already present, which were
   newly installed, which failed. A failure is reported as a failure — never
   presented as success.

### Detect

A manifest entry is **already installed** when both hold in the host project:

- a `SKILL.md` exists in EITHER `.claude/skills/<name>/` OR
  `.agents/skills/<name>/` — both are checked because the install location
  varies with the `skills` CLI version (the current CLI uses `.claude/skills/`),
  and
- `skills-lock.json` contains an entry keyed by `<name>`.

If exactly one of the two holds, the entry is **incompletely installed** and is
treated as a target for installation in step 5 (running the install reconciles the
state). If neither holds, the entry is **missing**.

### Install

Run, from the host project root:

```
<runner> skills add <source> [--skill <skill>] --agent claude-code
```

where `<runner>` is the runner from step 3, and `<source>` / `<skill>` come from
the manifest entry. The `--agent claude-code` selector targets the agent the
Harness runs under, so the installed skill is wired for Claude Code; the
implementation also passes `--yes` to skip the CLI's interactive prompts. The
CLI owns skill-file placement (the current CLI copies the skill into
`.claude/skills/<name>/`), the `skills-lock.json` update, and hash computation.

### Verify

After install, re-check the two **detect** signals for the entry. If both now
hold, the entry is verified installed. If not, the entry is reported as failed.

## Error handling

- **Not a project** (no `package.json`) — stop with an explanatory message.
- **No Node.js / package runner available** — stop and tell the user Node.js 18+
  is required to run the `skills` CLI.
- **`skills` CLI install failure** (network error, bad coordinates, CLI error) —
  report that entry as failed, continue processing the remaining entries, and list
  every failure in the final report. Do not claim success.
- **Incomplete pre-existing install** — handled as a normal install target (see
  *Detect*); re-running `skills add` reconciles the state.

## Constraints and language

- **Language.** Following the `clarify-component` convention: the skill body and
  reference files are written in English; at runtime the AI conducts dialogue and
  delivers the final report in the user's language.
- The skill does not modify host project files other than through the `skills`
  CLI. The CLI's artifacts (the skill directory under `.claude/skills/` — or
  `.agents/skills/` for an older CLI — and `skills-lock.json`) are the only
  expected changes.

## Verification of this slice

The skill is prose instructions for the AI; "testing" means running `/setup` in
the playground [`experiments/shadcn`](../../../experiments/shadcn).

- **Idempotency path.** When the playground already has the shadcn skill installed,
  `/setup` must detect it as *already present* and report so without reinstalling.
- **Install path.** Remove the playground's installed shadcn skill directory
  (the committed fixture sits at `.agents/skills/shadcn/`) and the shadcn entry
  from `skills-lock.json` to create a clean host, then run `/setup`. The shadcn
  skill must be installed and both artifacts produced.
- **Visible Outcome.** The slice's Visible Outcome is satisfied when, after
  `/setup` on a clean host, the installed shadcn skill directory
  (`.claude/skills/shadcn/` with the current CLI) and a `skills-lock.json`
  shadcn entry both exist.

## Open design points (deferred to implementation)

- The exact `description` and `allowed-tools` strings in the `SKILL.md` front
  matter.
- The precise `skills` CLI flag set — in particular whether `--agent claude-code`
  is the correct selector. **Resolved during implementation:** `--agent
  claude-code` is correct; the current CLI installs the skill as a *copy* under
  `.claude/skills/<name>/` and does not create `.agents/skills/`. Detection was
  changed to accept the skill in either `.claude/skills/<name>/` or
  `.agents/skills/<name>/`. See
  [`docs/issues/2026-05-22-setup-detection-install-path-assumption.md`](../../issues/2026-05-22-setup-detection-install-path-assumption.md).
- The exact rendering of the user-facing report.

## Source materials

- [`docs/foundation.md`](../../foundation.md) — project vision; the pinned tech
  stack (Next.js + shadcn/ui + Tailwind v4) from which the manifest dependency is
  derived; the no-vendoring stance.
- [`docs/slices.md`](../../slices.md), [`docs/select.md`](../../select.md) — the
  third confirmed slice and its selection; the Harness composition decision
  recorded under Foundation Update Candidates.
- [`docs/obstacles.md`](../../obstacles.md) — obstacle #3 (the cost of discovering
  and reflecting existing-codebase context), which a present, installed stack skill
  helps reduce.
- The `skills` CLI / skills.sh and the shadcn/ui skill (`shadcn/ui` repo,
  `skills/shadcn/SKILL.md`) — the external installation mechanism and the one
  current external stack skill.
