---
name: clarify-component
description: Use when the user supplies a natural-language UI component request that needs to be clarified into a precise component specification. Asks targeted follow-up questions until ambiguity is resolved, then writes a Markdown spec to docs/gyre/specs/components/<kebab-name>.md. Component-only in this iteration; classifier recognizes page and design-guide requests but the skill terminates without writing a file in those cases.
---

# clarify-component

A skill that turns an under-specified natural-language UI component request into a precise component specification through dialogue, then writes the result to a Markdown file. The skill resolves ambiguity by asking follow-up questions until no ambiguity remains.

## Invocation

```
/clarify-component "<natural-language request>"
```

The natural-language request may be supplied as an argument. If it is omitted, the AI asks for it as the first message of dialogue.

## Output

A single Markdown file at:

```
docs/gyre/specs/components/<kebab-name>.md
```

The output path is not user-overridable in this iteration. See `references/output-format.md` for the file's full structure.

## Language

The skill body and reference files are written in English. At runtime the AI **must** deliver questions to the user in the user's language, inferred from the user's first message. The output Markdown file uses English-language section headings (as templated in `references/output-format.md`); the *content* the user supplies inside those headings may be in the user's language.

## Process

The skill executes ten steps in order. Steps 4–8 are the **dimensions** documented in `references/dimensions.md`.

### 1. Classify

Determine whether the request describes a single UI component, a page or layout, or a design guide.

- **Clear component classification** → proceed to step 2.
- **Clear page or design-guide classification** → tell the user the skill currently handles component specifications only, and terminate without writing a file. The user is informed that page and design-guide handling are planned for future iterations.
- **Ambiguous classification** → ask the user directly which of the three the request is, and branch on the answer.

### 2. Risk read (internal)

Estimate the three risk axes documented in `references/risk-signals.md` from the request text alone. This estimate is not shown to the user and is not written to the output file. It is used solely to shape the depth of subsequent questions.

If the risk estimate would change the depth of subsequent dialogue (relative to a default low-risk component), announce the *reason* to the user in a single sentence, per the *Context announcement* rules in `references/risk-signals.md`. The user can correct the impression; a correction triggers re-evaluation.

### 3. Where (always first)

Run the *Where* dimension as documented in `references/dimensions.md`.

- If the answer is **Reuse** → write the reuse-path body (per `references/output-format.md`) and terminate.
- If the answer is **Extend**, **Compose**, or **New** → re-evaluate risk axes with the Where answer absorbed, then proceed.

### 4. What (always active)

Run the *What* dimension. The Naming question establishes `<kebab-name>` used as the output filename.

### 5. Look (conditional)

Open with the activation gate (the meta question about design-guide presence). Based on the answer, run *Look* in compressed or full form. See `references/dimensions.md`.

### 6. How (always active)

Run the *How* dimension. Depth is calibrated by the current risk read.

### 7. Data / Content (conditional)

Activated by the current risk read, per `references/risk-signals.md`. Run in full or compressed form.

### 8. Non-goals (always active)

Run the *Non-goals* dimension. At least one non-goal must be recorded.

### 9. Self-check

Evaluate, with no question asked to the user:

1. Could a code-generation step produce a coherent implementation from this spec, in one pass?
2. Are there any captured items that could reasonably be interpreted in two different ways?
3. Are there any areas the risk signals flagged that have not been explored in sufficient depth?

The ambiguity threshold tightens with risk: a high-risk component requires a lower ambiguity tolerance than a low-risk one to pass self-check.

If any of the three checks finds remaining ambiguity, return to dialogue with additional questions targeted at the ambiguity, then re-run self-check. **Repeat until no ambiguity remains. There is no iteration cap.**

### 10. Write spec

Write the Markdown file at `docs/gyre/specs/components/<kebab-name>.md` per `references/output-format.md`. If a file already exists at the target path, do not overwrite silently — tell the user and ask whether to overwrite or pick a different name.

After writing, tell the user the file path so they can review.

## Dialogue mode

**Sequential by default.** Ask one question at a time. Each answer is absorbed before the next question is composed, so the next question is calibrated to the current state of the dialogue — including the possibility that no further question in the current dimension is necessary.

This is the mechanism by which the dimension activation policy and the self-check actually do their work. Batching weakens it.

**Three accommodations** soften the friction without diluting the depth:

1. **Absorbing answers richer than asked.** If the user volunteers information beyond what was asked, absorb it and adjust (or skip) subsequent questions accordingly. The user may answer sequentially or in bulk; the AI must remain sequential in asking.
2. **Enumerative dimensions may be batched.** *Non-goals* is the clearest case; ask it in one prompt.
3. **Multiple choice preferred when answers are bounded.** When a question has a finite set of reasonable answers, present them as choices rather than asking open-endedly. Open-ended is reserved for cases where free-form input is necessary.

## Self-check accommodations

If a user cannot answer a particular question, use these tools to keep clarification moving:

- **Propose a sensible default** for the user to explicitly accept or reject. Acceptance is a resolved decision, not a deferral.
- **Re-cast as multiple choice** when the question was open-ended.
- **Offer concrete examples** to disambiguate intent.

The skill never produces a spec that contains unanswered questions. The output format has no `Open Questions` section. Vary the *form* of a question when the user is having trouble with it; do not repeat the same question in the same form.

## Constraints on generated documents

The spec Markdown files produced by this skill are formal documents. They must not:

- Reference local paths of the host project beyond the conventional output path.
- Use vocabulary specific to development plugins outside the AI Harness.
- Link to files maintained by external plugins.

The only informal vocabulary they may use is vocabulary belonging to the AI Harness itself.

## Reference files

- `references/dimensions.md` — Definitions, activation conditions, and question templates for the six dimensions.
- `references/risk-signals.md` — Three risk axes, the heuristic guide for assigning low/medium/high, and the context-announcement phrasing rules.
- `references/output-format.md` — Front matter schema and Markdown body structures for the full and reuse paths.

The AI reads these reference files at invocation time and uses them to drive dialogue and rendering. None of their contents are duplicated in this file.
