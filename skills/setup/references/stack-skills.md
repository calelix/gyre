# Stack Skills Manifest

The external stack skills that the AI Harness's component-generation pipeline
depends on. The `setup` skill reads this file to decide what to install into a
host project.

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
