---
created: 2026-05-17
updated: 2026-05-17
---

# Slices

## Confirmed Slices

1. When the user invokes a new skill `/[skill-name]`, the AI asks N questions and produces one component request specification markdown. — priority: high

## Tentative Slices

(none)

## Change Log

### 2026-05-17

- Clarified: "When the user invokes a new skill `/[skill-name]`, the AI asks N predefined questions and produces one component request specification markdown." — removed "predefined" to avoid implying a fixed in-skill question flow, which conflicts with obstacle #1 (per-request calibration of dialogue depth).
- Added: "When the user invokes a new skill `/[skill-name]`, the AI asks N predefined questions and produces one component request specification markdown." — first confirmed slice derived from the End Goal in `docs/foundation.md`; chosen as the seed of obstacle #1 (calibrating dialogue depth) via the `slicer:decomposing-slices` B4 fallback.

## Foundation Update Candidates

- Scope decision: AI Harness is specialized for the Next.js + shadcn/ui + Tailwind CSS v4 stack rather than being general-purpose — flagged on 2026-05-17 — resolved in foundation.md on 2026-05-21
- Tailwind CSS version pinned to v4 — flagged on 2026-05-17 — resolved in foundation.md on 2026-05-21
