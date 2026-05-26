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

Questions, prompts, multiple-choice presentations, and any follow-up messages delivered to the user are in the user's language, inferred from the user's first message. Everything else — the spec body in its entirety (Summary, Purpose, descriptions, rationale, edge cases, code-bound string literals), and any internal AI processing — is in English without exception. The skill body and reference files are written in English. The output Markdown file's section headings and content are both in English; the AI translates the user's natural-language answers into English at spec write time.

## Process

The skill executes eleven steps in order. Steps 3–9 are the **seven dimensions** documented in `references/dimensions.md`.

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

### 4. Interaction model (conditional)

Activated when the requested component class admits more than one legitimate interaction model in common practice. See `references/dimensions.md`. When not activated, the dimension produces no spec section.

### 5. What (always active)

Run the *What* dimension. The Naming question establishes `<kebab-name>` used as the output filename.

### 6. Look (conditional)

Open with the activation gate (the meta question about design-guide presence). Based on the answer, run *Look* in compressed or full form. See `references/dimensions.md`.

### 7. How (always active)

Run the *How* dimension. Depth is calibrated by the current risk read. The three sub-sections capturing observable behavior (`## How → Interactions`, `## How → Edge cases`, `## How → Accessibility`) follow a requirements-only voice; see `references/dimensions.md` and Step 10 Self-check for enforcement.

### 8. Data / Content (conditional)

Activated by the current risk read, per `references/risk-signals.md`. Run in full or compressed form.

### 9. Non-goals (always active)

Run the *Non-goals* dimension. At least one non-goal must be recorded.

### 10. Self-check

Evaluate, with no question asked to the user:

1. Could a code-generation step produce a coherent implementation from this spec, in one pass?
2. Are there any captured items that could reasonably be interpreted in two different ways?
3. Are there any areas the risk signals flagged that have not been explored in sufficient depth?
4. Does every bullet in `## How → Interactions`, `## How → Edge cases`, and `## How → Accessibility` read as a requirement rather than an implementation pattern? Apply the three tests defined in `references/dimensions.md` (*Note on requirements-only voice for `## How`*) and `references/output-format.md` (*`## How` voice rule*): the canonical Outcome test, the Procedural test, and the Mechanism citation test. The tests are language-agnostic; example sets in those files anchor the AI's judgment but are not exhaustive. When a bullet trips any of the three tests, return to dialogue with a single rephrase question. If the user confirms the pattern itself is the requirement, route the bullet to `## Implementation hints` with the user's stated rationale; otherwise rewrite the bullet as the underlying requirement.

The ambiguity threshold tightens with risk: a high-risk component requires a lower ambiguity tolerance than a low-risk one to pass self-check.

If any of the four checks finds remaining ambiguity or voice-rule violations, return to dialogue with additional questions targeted at the issue, then re-run self-check. **Repeat until no ambiguity or voice-rule violation remains. There is no iteration cap.**

### 11. Write spec

Write the Markdown file at `docs/gyre/specs/components/<kebab-name>.md` per `references/output-format.md`. If a file already exists at the target path, do not overwrite silently — tell the user and ask whether to overwrite or pick a different name.

After writing, tell the user the file path so they can review.

## Dialogue mode

**Sequential by default.** Ask one question at a time. Each answer is absorbed before the next question is composed, so the next question is calibrated to the current state of the dialogue — including the possibility that no further question in the current dimension is necessary.

This is the mechanism by which the dimension activation policy and the self-check actually do their work. Batching weakens it.

**Three accommodations** soften the friction without diluting the depth:

1. **Absorbing answers richer than asked.** If the user volunteers information beyond what was asked, absorb it and adjust (or skip) subsequent questions accordingly. The user may answer sequentially or in bulk; the AI must remain sequential in asking.
2. **Enumerative dimensions may be batched.** *Non-goals* is the clearest case; ask it in one prompt — a single multiSelect AskUserQuestion call under the delivery rule in accommodation 3.
3. **AskUserQuestion as default delivery.** Every question is delivered via the AskUserQuestion tool. The AI generates 2–4 candidate answers from the accumulated dialogue context (original request text + prior dimension answers) and presents them as options; the tool's automatic "Other" choice carries any free-form custom input. Option labels and descriptions are in the user's language. When the AI cannot generate ≥ 2 distinct candidates for a question, it falls back to a plain text prompt — this should be rare. See `references/dimensions.md` *Delivery mechanism* for the per-dimension shape (option count, multiSelect, candidate source).

## Self-check accommodations

If a user cannot answer a particular question, use these tools to keep clarification moving:

- **Propose a sensible default** for the user to explicitly accept or reject. Under the AskUserQuestion delivery rule (accommodation 3 above) every option is already a proposed default; if the user picks none of the offered options and types in "Other" something equivalent to "I don't know", treat the most-plausible option as the proposal and ask for accept/reject explicitly. Acceptance is a resolved decision, not a deferral.
- **Widen the candidate set** if the user reports the offered options don't match their intent. Re-issue the AskUserQuestion call with a different candidate set drawn from a broader interpretation of the context.
- **Offer concrete examples** in option descriptions to disambiguate intent — the AskUserQuestion option `description` field is the natural place for these.

The skill never produces a spec that contains unanswered questions. The output format has no `Open Questions` section. Vary the *form* of a question when the user is having trouble with it; do not repeat the same question in the same form.

## Constraints on generated documents

The spec Markdown files produced by this skill are formal documents. They must not:

- Reference local paths of the host project beyond the conventional output path.
- Use vocabulary specific to development plugins outside the AI Harness.
- Link to files maintained by external plugins.

The only informal vocabulary they may use is vocabulary belonging to the AI Harness itself.

## Reference files

- `references/dimensions.md` — Definitions, activation conditions, and question templates for the seven dimensions.
- `references/risk-signals.md` — Three risk axes, the heuristic guide for assigning low/medium/high, and the context-announcement phrasing rules.
- `references/output-format.md` — Front matter schema and Markdown body structures for the full and reuse paths.

The AI reads these reference files at invocation time and uses them to drive dialogue and rendering. None of their contents are duplicated in this file.
