---
created: 2026-05-17
updated: 2026-05-17
---

# Obstacles to AI-Generated UI Components and Pages

The system [foundation.md](./foundation.md) describes — a "single director ↔ AI (acting as PM, designer, and developer)" channel on which an AI Harness turns a natural-language request into either a custom UI component conformant to the design guide, or a `DESIGN.md` itself — sets the promises against which the AI's output must hold up.

This document records, ordered by weight, the structural obstacles the AI faces in meeting those promises when working from React/TypeScript and an existing codebase.

## 1. Calibrating how far dialogue should close the intent-to-spec gap (heaviest)

A single line of natural language strips out almost every decision a PM, a designer, and a developer would normally negotiate separately — state model (loading/empty/error/disabled), accessibility, whether to reuse an existing component, visual hierarchy, responsive behavior.

- **Ask too much** and the "single director ↔ AI" speed promise breaks. The director ends up shouldering the PM, design, and engineering review loads again — the human-to-human dialogue burden this system was meant to replace returns in a new shape.
- **Ask too little** and output quality collapses. The AI fills in plausible-looking defaults that diverge from actual intent, and the director has to reject finished artifacts after the fact.
- On every request the AI has to redraw the line between "decisions I must ask about" and "decisions I can reasonably default." Drawing that line itself requires the simultaneous judgment of a PM, a designer, and a developer.

The other two obstacles can be reduced through tooling and automation; this calibration is a fresh judgment on every request with no standard solution, which is what makes it the heaviest.

## 2. The verification gap

Even when TypeScript compiles and unit tests pass, the following defects are not caught automatically.

- **Visual-consistency defects** — design tokens used correctly, yet composition, spacing, or hierarchy drifts from the design guide.
- **Interaction defects** — missing or unnatural hover, focus, keyboard navigation, transitions.
- **Accessibility defects** — missing semantic markup, ARIA, contrast, focus traps.
- **Edge-state defects** — long text, empty states, loading, errors, internationalization, RTL handling left unaddressed.

The "enterprise-grade code" bar set in [foundation.md](./foundation.md) is, in practice, validated by human re-review. Once that re-review is required, the director's cognitive load climbs back up and the single-channel promise weakens.

The key lever for closing this gap is embedding self-verification loops into the AI Harness itself — Storybook-based visual regression tests, automated accessibility checks, state-matrix scenarios — so the AI verifies its own output.

## 3. The cost of discovering and reflecting existing-codebase context

On every generation, the AI has to rediscover:

- shadcn components that have already been customized, and their variants
- Project helpers, hooks, and utilities
- The currently active design tokens (the YAML front matter in `DESIGN.md`)
- The history of past decisions — why component X looks the way it does

The problems:

- A full scan every time is wasteful in both latency and tokens.
- Caching or indexing introduces drift between the code and the index.
- The AI is biased toward "create new" over "extend existing," so catalog consistency tends to degrade over time.

## The central trade-off

**Depth of dialogue and verification ↔ the director's cognitive load.**

- More depth raises quality but shakes the "single director" promise.
- Less depth eases the director's load but shakes output quality.

Where to set the balance between these two axes — and how to shift it dynamically based on the request — is the central design question for the AI Harness.
