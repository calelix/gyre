# Stack Skills Manifest

The external stack skills that the AI Harness's component-generation pipeline
depends on. The `setup` skill reads this file to decide what to install into a
host project. `generate-component` reads it at its Step 1.5 precondition to
verify those skills are installed before running.

## Entry fields

Each row of the Skills table below describes one external stack skill:

- `name` — the skill's installed name. Detection key: matched against the
  skill directory (`.claude/skills/<name>/` or `.agents/skills/<name>/`) and
  the `skills-lock.json` entry key in the host project.
- `source` — the `skills` CLI source coordinates: a GitHub `owner/repo`.
- `skill` — the `--skill` selector, used only when one source repo publishes
  multiple skills. `—` when the repo publishes a single skill.
- `purpose` — why the component-generation pipeline needs this skill.
- `kind` — invocation classification consumed by `generate-component`
  Step 2.5. One of:
  - `always` — invoke via Claude Code's Skill tool on every run.
  - `conditional` — invoke only when the row's `condition` predicate
    evaluates true against the loaded spec.
  - `external` — do not invoke through this manifest. The skill loads via
    its own auto-activation mechanism (e.g., `shadcn` auto-activates on
    `components.json` presence). The entry remains so that `setup` still
    installs it.
  - `deferred` — do not invoke this iteration. The entry remains so that
    `setup` still installs it for parity with future iterations.
- `condition` — single-line natural-language predicate evaluable against
  the spec body (parsed at `generate-component` Step 2). Required when
  `kind: conditional`; `—` otherwise.

Setup installs every entry regardless of `kind`. The `kind` and `condition`
columns are read by `generate-component` only.

## Skills

| name | source | skill | purpose | kind | condition |
|---|---|---|---|---|---|
| `shadcn` | `shadcn/ui` | — | Project-aware shadcn/ui context — CLI usage and composition rules — that component generation relies on. | `external` | — |
| `vercel-react-best-practices` | `vercel-labs/agent-skills` | `vercel-react-best-practices` | React idioms and anti-patterns the generated component must respect. | `always` | — |
| `vercel-composition-patterns` | `vercel-labs/agent-skills` | `vercel-composition-patterns` | Component composition rules that shape how primitives are arranged. | `always` | — |
| `next-best-practices` | `vercel-labs/next-skills` | `next-best-practices` | Next.js App Router conventions (RSC boundaries, file conventions, data patterns) the generated component must conform to. | `always` | — |
| `next-cache-components` | `vercel-labs/next-skills` | `next-cache-components` | Next.js Cache Components rules for any cached / RSC-cached behavior the spec requires. | `conditional` | spec's `## Data` section mentions caching, RSC-cache, or revalidation. |
| `web-design-guidelines` | `vercel-labs/agent-skills` | `web-design-guidelines` | Web interface guidelines (interaction, accessibility, layout) the generated component must meet. | `always` | — |
| `building-components` | `vercel/components.build` | `building-components` | Component construction guidance the generated component must follow alongside the design / interaction guidelines. | `always` | — |
| `agent-browser` | `vercel-labs/agent-browser` | `agent-browser` | Browser automation reserved for future verification phases (visual / interaction / accessibility checks). | `deferred` | — |

## How to add a stack skill

1. Append one row to the Skills table above with `name`, `source`, `skill`,
   `purpose`. Use `—` in the `skill` column when the source repo publishes
   a single skill.
2. Classify the entry's `kind` (`always` / `conditional` / `external` /
   `deferred`). If `conditional`, write the `condition` predicate as a
   single sentence evaluable against the spec body; otherwise put `—`. See
   the field definitions above for what each `kind` means.
3. (Optional) If `generate-component` needs to *use* the new skill's
   knowledge in discovery beyond what existing prose covers, update
   `skills/generate-component/references/host-discovery.md` accordingly.
4. Commit. No other file needs changes — `setup` (install) and
   `generate-component` (Step 1.5 precondition + Step 2.5 invocation) both
   read this manifest dynamically.
