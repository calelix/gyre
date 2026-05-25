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

## Skills

| name | source | skill | purpose |
|---|---|---|---|
| `shadcn` | `shadcn/ui` | — | Project-aware shadcn/ui context — CLI usage and composition rules — that component generation relies on. |
| `vercel-react-best-practices` | `vercel-labs/agent-skills` | `vercel-react-best-practices` | React idioms and anti-patterns the generated component must respect. |
| `vercel-composition-patterns` | `vercel-labs/agent-skills` | `vercel-composition-patterns` | Component composition rules that shape how primitives are arranged. |
| `next-best-practices` | `vercel-labs/next-skills` | `next-best-practices` | Next.js App Router conventions (RSC boundaries, file conventions, data patterns) the generated component must conform to. |
| `next-cache-components` | `vercel-labs/next-skills` | `next-cache-components` | Next.js Cache Components rules for any cached / RSC-cached behavior the spec requires. |
| `web-design-guidelines` | `vercel-labs/agent-skills` | `web-design-guidelines` | Web interface guidelines (interaction, accessibility, layout) the generated component must meet. |
| `building-components` | `vercel/components.build` | `building-components` | Component construction guidance the generated component must follow alongside the design / interaction guidelines. |
| `agent-browser` | `vercel-labs/agent-browser` | `agent-browser` | Browser automation reserved for future verification phases (visual / interaction / accessibility checks). |

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
