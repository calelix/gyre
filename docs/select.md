---
created: 2026-05-17
updated: 2026-05-21
---

# Select

## Active

### When the user runs the setup skill in a host project, the AI installs into that project the external stack skills that the component-generation skill depends on.

**Closed demo line**: When the user runs the setup skill in a host project, the AI installs into that project the external stack skills that the component-generation skill depends on.

**Visible Outcomes**:
- When the user runs the setup skill in a host project, the external stack skills that the component-generation skill depends on are installed into that project.

**Selected on**: 2026-05-21

## History

### 2026-05-21 — When the user runs a skill against an existing component request specification produced by the clarify-component skill, the AI generates a working custom UI component from that spec, which the user can see on screen.

**Closed demo line**: For a user's natural-language component request specification produced by the clarify-component skill, a skill is created that actually implements the corresponding component.

**Visible Outcomes**:
- The skill described in the closed demo line exists.
- A component based on the specification is generated.

**Selected on**: 2026-05-21

**Status**: superseded

### When the user invokes a new skill `/[skill-name]`, the AI asks N questions and produces one component request specification markdown.

**Closed demo line**: A skill is created such that, when the user invokes `/[skill-name]`, the AI asks N questions and produces one component request specification markdown.

**Visible Outcomes**:
- A skill that can produce a component request specification markdown exists.
- Invoking the skill produces a component request specification markdown file at `docs/gyre/specs/components/<kebab-name>.md`.

**Selected on**: 2026-05-17
**Completed on**: 2026-05-19
**Evidence**:
- Skill implementation: `skills/clarify-component/` (commit 31b2657).
- First produced artifact: `docs/gyre/specs/components/mode-toggle.md` (commit d4f33ff).

**Completion note**: Visible Outcome #2 was rewritten on 2026-05-19 to match the skill's actual design. The original wording — "Invoking the skill from `experiments/shadcn` produces a component request specification markdown file at that path" — implied a per-invocation working-directory-relative output path. The realized skill fixes the output path at `docs/gyre/specs/components/<kebab-name>.md` and explicitly states it is not user-overridable in this iteration (see `skills/clarify-component/SKILL.md` and `skills/clarify-component/references/output-format.md`). The mode-toggle artifact was produced from a request referencing `experiments/shadcn` content, satisfying the spirit of the original outcome (playground-driven invocation) while writing to the canonical spec path.

## Foundation Update Candidates

(none)
