---
created: 2026-05-11
updated: 2026-05-29
---

# Foundation

## Core Problem & End Goal

### Core Problem
In frontend development, communication between product managers, designers, and developers is time-consuming and frequently produces conflict driven by differing perspectives. When that conflict is healthy, it lifts the quality of the outcome; when it isn't, it simply burns energy.

This project replaces that structure with **"a single director ↔ AI (acting as PM, designer, and developer)"** communication. The deeper problem is that when one person carries all three roles, three burdens — **intent, implementation, and verification** — converge on a single person, and that person's cognitive load grows. The quality lift that once came from human-to-human dialogue is now preserved by shifting the deep dialogue and verification *inside the AI*: the director conveys only short natural-language intent, while their cognitive load converges toward zero. **Driving that cognitive load down is what this project exists to do.**

### End Goal (What "Done" Looks Like)
When a director expresses intent for a UI component or page in natural language, an **AI Harness** — a bundle of skills, tools, context, and validation procedures distributed as a Claude Code Plugin — should produce, from that short intent and with little to no clarification round-trip, one of the following:

- **A custom UI component** built on top of Headless UI (e.g., Base UI, Radix UI) that conforms to the design guide, or
- **A page** that conforms to the design guide

Before the artifact reaches the director, the AI verifies its visual, accessibility, and edge-state quality, so that this burden rests with the AI rather than the director.

Stable execution of this behavior — producing the artifact with the director's follow-up review burden minimized — marks "completion." Beyond that point, the goal is to systematically manage and extend the component library while the AI keeps learning and evolving.

**Constraints**

- Scope is frontend UI only — non-frontend code, backend/API design, and infrastructure decisions are out of scope.
- The deliverable stops at a reviewable artifact: the AI does not autonomously merge or deploy, and nothing is pushed without the director's approval.

## Vision / Target Users / Success / Core Values

### Vision
In the AI era, the boundaries between roles are dissolving. Depth still matters, but advances in AI now let one person tackle problems that used to belong to multiple disciplines. Frontend UI/UX has always been a tangled problem across planning, design, and engineering — **this project exists to make that problem easy to solve for anyone who wants to solve it.**

### Target Users
The primary audience is the **director** — a single person who carries product, design, and engineering intent at once, and who struggles with design–development synchronization across that span. In a design system, planning intent is expressed through a medium (e.g., Figma) and then re-expressed as developer code, and drift accumulates along the way. The synchronization target does not have to be Figma; anyone who suffers from this chain breaking down qualifies.

### Success (As Felt by the Director)
The director gives short natural-language intent and receives an enterprise-grade UI component or page that needs **little to no follow-up touch-up** — without breaking the design system's consistency. Success is felt as **near-zero cognitive load**: the director is freed from the burden of decisions, re-review, and the nagging *"have I checked enough?"*

### Core Values
**Quality of the output is the top priority.** When the AI interprets the director's natural-language intent and produces something, poor quality renders everything else meaningless. Crucially, that quality must be delivered **without loading the director** — quality and a near-zero director burden are not a trade-off to balance but a single requirement to meet together.

Quality is defined as:
1. The director's intent is **fully reflected**
2. That intent and the decisions around it are **recorded**
3. The result is generated as **enterprise-grade code**
4. **Visual correctness, accessibility, and edge-state behavior** are part of the standard — enterprise-grade means these are demonstrably covered, not assumed

## Tech Stack (Initial)

**Scope:** The output produced by the AI Harness is limited to Next.js (App Router), shadcn/ui, and Tailwind CSS v4 — each used at its latest version. Other frameworks and styling systems are not under consideration at this time.

### Languages
- **TypeScript**, **React**
- **Node.js** where needed

### Frameworks / Major Libraries
- **Target framework for generated output**: Next.js (App Router)
- **Headless UI foundations**: Base UI, Radix UI, and similar — primitives forming the basis of design-guide-conformant custom components
- **Component starter set**: shadcn/ui — a copy-paste open-source component collection that picks one of Base UI or Radix UI as its primitive (built on Tailwind)
- **Styling**: Tailwind CSS v4
- **Component catalog**: Storybook

### Runtime / Distribution Form
- Runs as a **Claude Code Plugin**

### Harness Composition
The AI Harness bundles or vendors no third-party stack skills — it ships only its own skills. Among these is a `setup` skill that, on demand, installs the external stack skills the component-generation skill depends on into the host project. Those external stack skills are installed as individual skills via the `skills` CLI, never as plugins (the Harness itself remains a Claude Code Plugin, per Runtime / Distribution Form above).

### AI/LLM
- **SDK**: No separate SDK — uses Claude Code's built-in invocation path
- **Model**: Claude family (specific model line deferred)

### Package Manager
- **pnpm**

### TBD
- **Data storage approach** (where the component library, design guide, and conversation history are kept): TBD
- **Build tooling**: TBD
