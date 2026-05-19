# Strategy

Upstream anchor for the Gyre project. Downstream stages — skills like `clarify-component`, future skills, and the issue records under `docs/issues/` — read this document to understand the problem being solved, who we're solving it for, and what counts as progress. Edit as the project's understanding sharpens.

## Target problem

Frontend development typically routes work through three roles — product manager, designer, and developer. The handoff between them is slow and prone to conflict driven by differing perspectives. When the conflict is healthy it lifts quality; when it isn't, it just burns energy.

When a single person carries all three roles — a *director* running solo — the role-to-role dialogue collapses, and with it the quality lift that used to come from it. Gyre's bet: the quality lift can be reproduced through structured human-to-AI dialogue, with the AI taking on all three roles deliberately.

## Persona

The *director*: a single human who carries product, design, and engineering intent. They drive the work end-to-end, formulate intent in natural language, and review what the AI produces. They do not want to context-switch between role-shaped tools.

## Approach

Given a natural-language request for a UI component or page, Gyre refines under-specified intent through structured dialogue (the AI playing PM, designer, and developer in sequence as the request requires), and produces one of:

- A custom UI component built on top of Headless UI primitives (Base UI, Radix UI) that conforms to the project's design guide, or
- A design guide itself (`DESIGN.md`).

The dialogue is not free-form chat. It is structured by skills that enforce specific dimensions of clarification — for example, `clarify-component` runs Where / What / Look / How / Data / Non-goals in sequence and produces a formal Markdown artifact. The artifact is the contract that downstream code generation consumes.

## Definition of done

Stable execution of the behavior above. *Stable* means: a director can give a natural-language UI request and receive an artifact that needs no clarification round-trips; downstream code generation produces working code from that artifact in one pass without semantic surprises.

Beyond done: systematic component-library management and ongoing learning — Gyre's behavior should improve cycle-over-cycle as `docs/issues/` accumulates concrete lessons.

## Tracks

Active workstreams:

- **Skills.** The natural-language → artifact pipeline. `skills/clarify-component` is the first complete example. Future skills will cover pages and the design-guide path.
- **Playgrounds.** Concrete tech-stack instances where artifacts are produced and validated against real code. `experiments/shadcn` is the current playground — Next.js 16 + React 19 + shadcn/ui + Tailwind 4 + next-themes + the internal `@workspace/ui` package (carrying primitives such as Button).
- **Artifacts.** The formal docs under `docs/gyre/specs/`. Components today; pages and `DESIGN.md` planned. These are the contracts between skills and downstream code generation.
- **Learning loop.** `docs/issues/` records what failed and how it was fixed. Each entry is meant to make the next unit of work easier.

## Key signals

How to tell whether the project is on track. Proposed; refine as evidence accumulates.

- **Clarification round-trips per artifact.** If a director needs more than one correction round to land a spec, the skill is under-asking.
- **Spec → code surprise rate.** How often downstream code generation has to re-ask the director something the spec should already have covered.
- **`Compose` vs. `New` ratio in clarify-component artifacts.** A healthy ratio suggests the skill is surfacing primitive reuse, not silently rebuilding what the playground already provides.
- **Issue accumulation rate.** Not a problem signal in itself — but a plateau on familiar failure modes (e.g., the same skill repeatedly missing the same dimension) *is* a problem signal.

## Non-goals (current phase)

- Multi-user collaboration features. The director is one person.
- Coverage of non-frontend codebases. UI components and design guides are the surface for now.
- Auto-merging or auto-deployment. Artifacts are produced for review, not for autonomous push.
