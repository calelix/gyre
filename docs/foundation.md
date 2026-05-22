---
created: 2026-05-11
updated: 2026-05-22
---

# Foundation

## Core Problem & End Goal

### Core Problem
In frontend development, communication between product managers, designers, and developers is time-consuming and frequently produces conflict driven by differing perspectives. When that conflict is healthy, it lifts the quality of the outcome; when it isn't, it simply burns energy.

This project replaces that structure with **"a single director ↔ AI (acting as PM, designer, and developer)"** communication. The quality lift that previously came from human-to-human dialogue must now be delivered by human-to-AI dialogue.

### End Goal (What "Done" Looks Like)
When a user requests a UI component or page in natural language, an **AI Harness** — a bundle of skills, tools, context, and validation procedures distributed as a Claude Code Plugin — should be able to refine the under-specified request through dialogue and produce one of the following:

- **A custom UI component** built on top of Headless UI (e.g., Base UI, Radix UI) that conforms to the design guide, or
- **A design guide itself** (`DESIGN.md`)

Stable execution of this behavior marks "completion." Beyond that point, the goal is to systematically manage and extend the component library while the AI keeps learning and evolving.

## Vision / Target Users / Success / Core Values

### Vision
In the AI era, the boundaries between roles are dissolving. Depth still matters, but advances in AI now let one person tackle problems that used to belong to multiple disciplines. Frontend UI/UX has always been a tangled problem across planning, design, and engineering — **this project exists to make that problem easy to solve for anyone who wants to solve it.**

### Target Users
The primary audience is **anyone struggling with design–development synchronization**. In a design system, planning intent is expressed through a medium (e.g., Figma) and then re-expressed as developer code, and drift accumulates along the way. The synchronization target does not have to be Figma. Anyone who suffers from this chain breaking down qualifies as a user.

### Success (As Felt by the User)
**Through AI conversation alone**, a new UI component or page is added without breaking the consistency of the design system, and work that previously took significant time is **dramatically shortened**.

### Core Values
**Quality of the output is the top priority.** When the AI interprets a user's natural-language intent and produces something, poor quality renders everything else meaningless.

Quality is defined as:
1. The user's intent is **fully reflected**
2. That intent and the decisions around it are **recorded**
3. The result is generated as **enterprise-grade code**

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

### Design ↔ Code Communication Medium
- Design intent is expressed and recorded as a **`DESIGN.md`** file, following the format specified by [google-labs-code/design.md](https://github.com/google-labs-code/design.md): YAML front matter for machine-readable design tokens combined with markdown prose for design rationale.

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
