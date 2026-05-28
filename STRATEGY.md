# Strategy

## Promise

Gyre makes one promise to a single director — one person carrying the
intent of PM, designer, and engineer simultaneously:

> **The director's short natural-language intent is converted into an
> enterprise-grade UI component or page, with follow-up review burden
> minimized.**

Previously, collaboration across multiple roles produced a *quality
lift*. That lift must now be reproduced in the dialogue between
director and AI — and the dialogue must be *small*. The limit target
is zero turns.

## Core Trade-off

The deeper the dialogue and verification, the higher the quality — but
the promise breaks. Deep dialogue and verification raise the director's
*cognitive load*. Cognitive load includes not only decision,
interpretation, and re-review, but also the unending self-questioning
of *"have I looked enough?"* A short natural-language input does not
mean a small cognitive load.

Gyre shifts this burden from the director to the AI. Deep dialogue and
verification happen inside the AI; the director only conveys short
natural-language intent. Quality is preserved as the result of deep
dialogue and verification, and the director's cognitive load converges
to zero.

## Stance

There are three burdens — *intent*, *implementation*, and
*visual·accessibility·edge-state verification*.

When these three converge on one person, conflicts repeat without
resolution and cognitive load grows.

Gyre redistributes the responsibility for these three.

- **Intent** is expressed by the director in short natural language.
- **Implementation** is inferred autonomously by the AI from stack
  skills and the host codebase.
- **Visual·accessibility·edge-state verification** is performed by the
  AI immediately after the artifact is produced.

## Decision Principles

To keep the responsibility shift intact, the following five principles
apply.

### 1. Separation of Intent and Implementation

Implementation decisions are not asked of the director. The director
expresses *intent*; the AI handles *implementation*. Traces of
implementation appearing in the specification are treated as a Phase 1
defect.

### 2. Question Minimization

Every question asked of the director is an immediate increase in
cognitive load. Questions are limited to **UI branches** — cases where
the same intent admits fundamentally different interaction models.
Implementation details and UI expression are never asked under any
circumstances.

UI branch questions are permitted only when both of the following
conditions hold simultaneously:

(a) The branch is not inferable from the intent, the catalog, or the
    design guide.
(b) The result of proceeding with a reasonable default would be
    difficult for the director to recover from with a small edit.

Edge states of data-dependent components are not asked about. *Which
states are possible* is inferred from the data source; *the expression
of each state* is inferred from the catalog and design guide.

### 3. Codebase and Stack Skill First

Implementation decisions are inferred from two sources — the patterns
of the host codebase and the best practices of stack skills. When the
two conflict, **the host codebase pattern takes precedence.** Patterns
already adopted by the catalog reflect this project's context.
Replacing them with general prescriptions breaks catalog consistency
and forces the director to compare *why is only this one different* —
a violation of the responsibility shift.

The exception is when the host codebase pattern is explicitly marked
*deprecated* or *incorrect*; in that case, the stack skill's
recommendation applies.

### 4. Reuse and Extension First

The AI is biased toward creating new components. When the catalog has
a similar component, *reuse*; when only minor differences exist,
*extend*; only when neither is possible, *create new* — in that
order. Extension is limited to *addition*; existing interfaces are
not changed — this prevents regression in existing call sites.

However, **primitive components are not modified or extended.**
Primitives are the foundation on which the entire catalog depends;
when a variant is needed, it is solved through *composition*.

### 5. Self-Verification Duty

Visual, accessibility, and edge-state verification is performed by the
AI on its own, immediately after the artifact is produced. The moment
the verification mechanism demands the director's *interpretation* or
*decision*, the responsibility shift is broken. Verification inputs
are derived automatically from the component type and from the
catalog and design guide; they do not require input from the director
into the specification.

## Signals of Progress

Whether Gyre is heading in the right direction is measured by the
following signals.

### 1. Frequency of Director's Follow-up Edits

How much the director had to *touch up* after receiving the artifact.
The top-level signal of whether the responsibility shift is working.
Must converge to zero.

### 2. Implementation Traces in the Specification

The rate at which implementation details appear in specifications.
The signal of whether [Principle 1](#1-separation-of-intent-and-implementation) is working. Must converge to zero.

### 3. Questions per Artifact

The average number of questions asked of the director per artifact.
The signal of whether [Principle 2](#2-question-minimization) is working. Must converge to zero.

### 4. Self-Verification Coverage

The fraction of all defects caught by self-verification. The signal of
whether [Principle 5](#5-self-verification-duty) is being realized. Must converge to one.

## Non-goals

- **Multi-director collaboration.** There is only one director. Intent
  conflicts among multiple directors and consensus-building are not
  addressed.
- **Beyond UI.** Non-frontend code, backend API design, and
  infrastructure decisions are not addressed.
- **Autonomous merge or deploy.** Artifacts are produced for the
  director's review. They are not automatically pushed without the
  director's approval.
