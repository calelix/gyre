# Risk signals

Three internal axes the AI uses to calibrate the depth of subsequent questions and the activation of the *Data / Content* dimension. The risk assessment is **never** written into the output spec file. It is only used to shape dialogue depth at runtime.

The user does not see the risk assessment itself. When the assessment changes the depth of subsequent questions, the AI announces the *context* of that depth in one line, using everyday language. The exact phrasing rule is given in the *Context announcement* section at the bottom of this file.

## Axes

Each axis is classified as `low`, `medium`, or `high`.

### Axis 1 — Sharing scope

| Level | Meaning | Signals in the request text |
|---|---|---|
| low | Single one-off use in a specific screen or flow | "this page's confirm button", "for the checkout result screen", scoped possessive language |
| medium | Reused within a domain | "the buttons in the checkout area", "form components", domain-bound plural language |
| high | Shared library candidate | "design system button", "shared", "standard", language that implies cross-domain reuse |

The Where answer refines this axis. A user who names many existing use sites pushes the axis upward.

### Axis 2 — State character

| Level | Meaning | Signals in the request text |
|---|---|---|
| low | Purely presentational; renders from props with no internal state | "display", "show", "render", language about visual output only |
| medium | UI-stateful; carries hover, focus, open, or selected state | "dropdown", "toggle", "expandable", "tabs" |
| high | Coupled to async or external I/O | "submit", "fetch", "loading", "payment", "save", language implying server interaction or optimistic updates |

### Axis 3 — Exposure breadth

| Level | Meaning | Signals in the request text |
|---|---|---|
| low | Internal tooling, developer-facing screens | "admin", "internal", "debug", "operator" |
| medium | Limited audience (beta, gated, role-restricted) | "beta", "permissioned users", "for editors" |
| high | Production critical path | "payment", "checkout", "login", "signup", "order", language implying the broad user base |

## Evaluation timing

1. **First pass** — immediately after Classify, based on the request text alone.
2. **Refinement** — after the Where answer, re-evaluate. The Where answer may shift *Sharing scope* substantially.

## Effects on the dialogue

The risk assessment shapes:

1. **Activation of the *Data / Content* dimension.** Any axis at `high` → full activation. All three at `low` plus a presentational role → compressed.
2. **Depth inside *How*.** When any axis is `high`, the AI asks accessibility and edge-case questions in more detail and does not let surface-level answers pass.
3. **Self-check threshold.** Higher risk lowers the ambiguity tolerance. The AI returns to dialogue more readily when risk is high.
4. **Precision inside *What*.** When risk is high, the AI is stricter about props' required/optional, defaults, and type clarity.

The *Where*, *What*, and *Non-goals* dimensions themselves are always active and not gated by risk; risk affects how deeply they are explored, not whether they are explored.

## Context announcement

When the assessment changes the depth of subsequent questions, the AI announces the *reason* in one line before continuing — using everyday language, never the word "risk" or the axis names.

Form: a single sentence that states (a) the impression the AI has formed about the component and (b) the concrete adjustment to the dialogue.

Examples (English source; the AI delivers in the user's language):

- "This looks like a production checkout button, so I'll ask about accessibility and edge cases in more detail."
- "This appears to be an internal admin filter, so I'll keep the questions on data shape brief."
- "This sounds like a shared library candidate, so I'll be precise about the props' types and defaults."

The user can correct this impression. A correction triggers a re-evaluation of the affected axes.

## What is not announced

- Numeric or letter grades on any axis.
- The word "risk."
- The names "Sharing scope," "State character," "Exposure breadth."

These are internal vocabulary only.
