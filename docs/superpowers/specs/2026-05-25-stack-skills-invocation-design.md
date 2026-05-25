---
created: 2026-05-25
---

# Stack-Skills Invocation Design

## Context

The previous spec
([2026-05-24-stack-skills-precondition-and-manifest-expansion-design.md](2026-05-24-stack-skills-precondition-and-manifest-expansion-design.md))
closed the **install-state gap**: `generate-component` now refuses to run
unless every manifest entry passes the install detection signals at Step 1.5.
That spec did not close the **call-mechanism gap** — verifying that the skill
files are present is not the same as putting their knowledge into the runtime
context.

In the reported run recorded at
[`docs/issues/2026-05-25-generate-component-stack-skills-uninvoked.md`](../../issues/2026-05-25-generate-component-stack-skills-uninvoked.md):

- All eight stack skills were detected as `installed` by Step 1.5.
- None of their `SKILL.md` bodies were loaded; the Skill tool was not invoked
  for any of them.
- `shadcn` did not fire because the run produced no `shadcn-install` Plan
  item; the remaining seven have no defined invocation point in the current
  process.
- Conventions were extracted from host files directly. The host happened to
  be conventional enough that the output passed, masking the gap.

This spec defines *when* and *which* stack skills `generate-component`
invokes via Claude Code's Skill tool, so the existing "delegated to the stack
skills installed by `setup`" language across the reference files is backed
by actual calls.

## Principle

**The spec is the trigger input.**

Stack-skill invocation decisions are derived from the spec (loaded at
Step 2), not from Plan items or other intermediate artifacts. This extends
`generate-component`'s existing "spec is the contract" rule: the spec already
drives output paths, file contents, prerequisites, and verification commands;
under this spec it also drives which stack skills' knowledge enters the
runtime context.

Two consequences follow:

- Invocation decisions happen at a single point in the process
  (immediately after the spec is parsed). Same spec → same set of skills
  invoked. Determinism is preserved across re-runs.
- A skill whose applicability cannot be cleanly expressed as a spec-derived
  predicate must be classified `always`. Its `conditional` case would amount
  to guessing.

## Classification

The eight current manifest entries are classified as follows. The
*Trigger / reason* column gives the predicate (for `conditional`) or the
applicability statement (for the others).

| Skill | Kind | Trigger / reason |
|---|---|---|
| `building-components` | `always` | Every component is *built* — construction guidance applies regardless of spec content. |
| `web-design-guidelines` | `always` | Every component must meet interaction / a11y / layout guidelines. |
| `vercel-composition-patterns` | `always` | Every component arranges primitives in some shape; composition rules are universal. |
| `vercel-react-best-practices` | `always` | Gyre's stack is React-fixed. A spec-derived predicate that separates "needs React idioms" from "doesn't" would always be true; promoting to `always` avoids the guess. |
| `next-best-practices` | `always` | Gyre's stack is Next.js-fixed. Same logic as React above. |
| `next-cache-components` | `conditional` | Trigger: spec's `## Data` section mentions caching, RSC-cache, or revalidation. Otherwise the skill's body is noise for non-cached components. |
| `shadcn` | `external` | Auto-activates via `components.json` presence (per [ui.shadcn.com/docs/skills](https://ui.shadcn.com/docs/skills)). Not invoked through this manifest. Setup installs it; `generate-component` does not call it via the Skill tool. The existing Step 7 `shadcn-install` invocation is *primitive installation*, a different concern. |
| `agent-browser` | `deferred` | Reserved for a future verification slice. Installed by `setup` to keep the manifest one source of truth, but `generate-component` does not invoke it this iteration. |

The classification rests on one assumption that the **Verification** section
below tests: shadcn's auto-activation actually fires during a Compose
generation in a `components.json`-bearing host. If verification shows it does
not, the follow-up is to reclassify `shadcn` (likely `conditional` with
predicate `Where.Decision == Compose`) or `always` — see Follow-ups.

## Manifest schema change

`skills/setup/references/stack-skills.md` is extended with two columns:

- `kind` — one of `always`, `conditional`, `external`, `deferred`. Required.
- `condition` — natural-language single-line predicate evaluable against the
  spec body. Required when `kind: conditional`, empty (`—`) otherwise.

The existing columns (`name`, `source`, `skill`, `purpose`) are unchanged.

Setup's behavior is unchanged: every entry is installed regardless of `kind`.
The `kind` and `condition` columns are consumed by `generate-component` only.

The "How to add a stack skill" section of the manifest gains a step:
classify the entry (`always` / `conditional` / `external` / `deferred`) and,
if `conditional`, write the predicate as a single sentence evaluable from
the spec.

## generate-component process change

A new step is inserted between Step 2 (Load spec) and Step 3 (Classify Where
decision):

**Step 2.5. Invoke stack skills.**

Read the manifest. For each row, in row order:

- `always` — invoke the named skill via Claude Code's Skill tool.
- `conditional` — evaluate the `condition` predicate against the spec body
  loaded in Step 2. If true, invoke via the Skill tool; otherwise skip.
- `external` — skip. The skill loads through its own auto-activation
  mechanism; the manifest entry exists for installation parity only.
- `deferred` — skip. The manifest entry exists for installation parity only.

Invocations run sequentially in manifest row order so traces are deterministic.

If a Skill invocation errors (skill not found, body load failure), the step
terminates and reports the failure to the user. Step 1.5 should have caught
missing installs already, so this branch is defensive against drift.

This step does not consume tool output beyond loading the skill body into
context. The skill bodies are then available to all subsequent steps
(Discovery, Plan, Write, Verify) without re-invocation.

### Existing steps that interact

- **Step 1.5 (Precondition)** — unchanged. Still verifies installation
  signals for all manifest entries regardless of `kind`. A user who runs
  `/generate-component` without `/setup` should be told to run setup first
  whether the missing entry is `always` or `deferred`.
- **Step 7 (Realize prerequisites)** — unchanged. The existing
  `shadcn-install` invocation installs a missing primitive into the host
  and remains Plan-driven (it depends on Plan composition in Step 5).
  Step 7 does not duplicate Step 2.5's invocation of the shadcn skill
  body — and would not, since shadcn is classified `external` in Step 2.5.

### Reference file changes

The "delegated to the stack skills installed by `setup`" wording in
[`skills/generate-component/references/host-discovery.md`](../../../skills/generate-component/references/host-discovery.md),
[`output-format.md`](../../../skills/generate-component/references/output-format.md),
and [`verification.md`](../../../skills/generate-component/references/verification.md)
is retained — those statements are now backed by the explicit Step 2.5
invocation. A one-line note is added in each, pointing to Step 2.5 as the
call mechanism, so a future reader does not re-read the same gap into
the wording.

## Out of scope

- **MCP integration.** Adding the shadcn MCP server to the manifest or
  wiring `generate-component` to call its tools is a separate concern,
  deferred to a later slice if needed.
- **Loading reference files of stack skills beyond what their `SKILL.md`
  naturally pulls.** Step 2.5 invokes the skill; the skill itself decides
  what its body and refs say. We do not Read reference files directly.
- **Adding stack skills beyond the current eight.** The manifest schema
  change above suffices to onboard future entries as one-row edits.
- **Re-litigating the precondition gate.** Step 1.5 stays strict per
  the prior spec.

## Verification

After implementation, re-run the full pipeline on a host project:
`/gyre:setup` → `/gyre:clarify-component` → `/gyre:generate-component`.
Three scenarios cover the classification:

1. **Simple display component** (no state, no caching, `Where → Decision`
   is `New`): expect 5 `always` skills invoked (rows in manifest order),
   0 `conditional`, shadcn relies on auto-activation. Tool-call trace
   should show 5 Skill invocations between Step 2 and Step 3.
2. **Compose component referencing an installed primitive**: expect the
   same 5 `always` invocations. The shadcn auto-activation should put
   shadcn's knowledge into context — confirmed by the output following
   shadcn composition rules (`FieldGroup`, semantic colors, etc.) without
   a Skill-tool invocation appearing in the trace. If this confirmation
   fails, file a follow-up issue to promote shadcn to `conditional` or
   `always` in the manifest.
3. **Component whose spec's `## Data` section mentions caching/RSC-cache**:
   expect 5 `always` + 1 `conditional` (`next-cache-components`) invoked
   — total 6 Skill invocations.

Successful execution of all three scenarios closes
[`docs/issues/2026-05-25-generate-component-stack-skills-uninvoked.md`](../../issues/2026-05-25-generate-component-stack-skills-uninvoked.md).

## Follow-ups

- **shadcn auto-activation reliability.** If scenario 2 above shows
  shadcn's knowledge is not actually reaching the runtime context,
  reclassify in the manifest (`conditional` keyed on `Where.Decision ==
  Compose`, or `always`). Document outcome in a new issue under
  `docs/issues/`.
- **agent-browser activation.** When the verification slice for visual /
  interaction / a11y checks lands, this entry transitions from `deferred`
  to either `always` or `conditional` per that slice's design.
- **Predicate evaluation drift.** If a `conditional` predicate's natural
  language proves ambiguous in practice (false positives or negatives),
  tighten the wording in the manifest. Predicates remain natural language —
  no formal grammar is introduced.

## Related

- Open issue this design closes:
  [`docs/issues/2026-05-25-generate-component-stack-skills-uninvoked.md`](../../issues/2026-05-25-generate-component-stack-skills-uninvoked.md)
- Prior design (precondition + manifest expansion):
  [`2026-05-24-stack-skills-precondition-and-manifest-expansion-design.md`](2026-05-24-stack-skills-precondition-and-manifest-expansion-design.md)
- Manifest:
  [`skills/setup/references/stack-skills.md`](../../../skills/setup/references/stack-skills.md)
- generate-component SKILL:
  [`skills/generate-component/SKILL.md`](../../../skills/generate-component/SKILL.md)
  (Steps 1.5, 2, 7)
- shadcn skill mechanism (auto-activation): [ui.shadcn.com/docs/skills](https://ui.shadcn.com/docs/skills)
