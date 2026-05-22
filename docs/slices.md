---
created: 2026-05-17
updated: 2026-05-22
---

# Slices

## Confirmed Slices

1. When the user invokes a new skill `/[skill-name]`, the AI asks N questions and produces one component request specification markdown. — priority: high
2. When the user runs a skill against an existing component request specification produced by the clarify-component skill, the AI generates a working custom UI component from that spec, which the user can see on screen. — priority: high
3. When the user runs the setup skill in a host project, the AI installs into that project the external stack skills that the component-generation skill depends on. — priority: high

## Tentative Slices

(none)

## Change Log

### 2026-05-21

- Added: "When the user runs the setup skill in a host project, the AI installs into that project the external stack skills that the component-generation skill depends on." — third confirmed slice; a separate setup capability surfaced while designing the generate-component slice — the Harness installs the external stack skills into the host project rather than vendoring them, so generate-component can reference them.
- Added: "When the user runs a skill against an existing component request specification produced by the clarify-component skill, the AI generates a working custom UI component from that spec, which the user can see on screen." — second confirmed slice; derived via the B4 fallback from candidate A, it consumes the component request spec produced by the now-completed first slice and realizes End Goal output type 1 (a custom UI component).

### 2026-05-17

- Clarified: "When the user invokes a new skill `/[skill-name]`, the AI asks N predefined questions and produces one component request specification markdown." — removed "predefined" to avoid implying a fixed in-skill question flow, which conflicts with obstacle #1 (per-request calibration of dialogue depth).
- Added: "When the user invokes a new skill `/[skill-name]`, the AI asks N predefined questions and produces one component request specification markdown." — first confirmed slice derived from the End Goal in `docs/foundation.md`; chosen as the seed of obstacle #1 (calibrating dialogue depth) via the `slicer:decomposing-slices` B4 fallback.

## Foundation Update Candidates

- Scope decision: AI Harness is specialized for the Next.js + shadcn/ui + Tailwind CSS v4 stack rather than being general-purpose — flagged on 2026-05-17 — resolved in foundation.md on 2026-05-21
- Tailwind CSS version pinned to v4 — flagged on 2026-05-17 — resolved in foundation.md on 2026-05-21
- Harness composition: the AI Harness bundles or vendors no third-party stack skills. It ships only its own skills — including a `setup` skill that installs the needed external stack skills into the host project on demand — which the component-generation skill then references. — flagged on 2026-05-21
- Harness external-skill consumption: external stack skills are installed into the host project only as individual skills (via the `skills` CLI into `.agents/skills/`), never as plugins. Sharpens the 2026-05-21 Harness-composition candidate. — flagged on 2026-05-22 (surfaced during `setup` skill brainstorming; see `docs/superpowers/specs/2026-05-22-setup-skill-design.md`)
