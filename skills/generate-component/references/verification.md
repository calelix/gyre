# Verification

This file is read by `generate-component` Step 9 to verify the emitted files and bound the self-repair loop. The two verification commands' outputs are treated as the truth — if a command says "fail," the run is failed regardless of how the AI feels about the generated code.

## What this file owns vs. delegates

**This skill owns** the command-resolution policy (which scripts to look up and the fallback rule), the self-repair loop logic, the error-normalization rule (the precise definition of "the same error"), what context the repair step is given, the two termination outcomes, and the scope exclusions below.

**Delegated to the host's stack skills** (see [`../../setup/references/stack-skills.md`](../../setup/references/stack-skills.md) for the current manifest) **and to ecosystem conventions:**

- The choice of typecheck tool and story-build tool the host stack uses (currently `tsc` and `storybook build`; these names appear in the resolution policy below because the manifest pins this stack).
- The package-runner forms used to invoke a binary without a host-installed script — resolved per [`../setup/SKILL.md`](../setup/SKILL.md) Step 3, which already encodes the ecosystem convention.
- The conventional script names hosts use for typecheck and story-build, and the shape of host package metadata where those scripts are declared.

The exact invocation form on a given host follows stack convention; this file only specifies the lookup policy.

---

## Command resolution policy

For each of the two verification commands, the resolution policy is:

1. Look up the host's `package.json` `scripts` for a known name in the order listed below. Use the first name found; stop at the first match.
2. If none of the names is found, fall back to direct binary invocation via the host's package runner (resolved per [`../setup/SKILL.md`](../setup/SKILL.md) Step 3).
3. Record the chosen command on the Plan as an `inferred` row so the user sees what will run.

**Typecheck command** — script names checked, in order: `typecheck`, `type-check`, `tsc`. Fallback binary: `tsc --noEmit`.

**Storybook build command** — script names checked, in order: `build-storybook`, `storybook:build`. Fallback binary: `storybook build`.

---

## The self-repair loop

On the first run:

1. Run the typecheck command.
2. If typecheck passes, run the Storybook-build command.
3. If both pass, the loop terminates with success.

The two commands are gated sequentially — Storybook build only runs when typecheck passes. On any failure:

1. Capture the full command output (stdout and stderr combined).
2. Extract the failing file paths from the output.
3. Edit those files to address the errors.
4. Re-run the failing command only — not both. (If typecheck failed, re-run typecheck. If Storybook build failed, re-run Storybook build. Typecheck is not re-run after a Storybook-build repair.)

The loop terminates on either of two outcomes:

- **(a) Both commands pass** — success.
- **(b) The most recent attempt produces the same normalized error signature as the previous attempt for the same command** — the loop cannot make progress; terminate and report to the user (see *Termination on (b)* below).

---

## Error-normalization rule

Before comparing two outputs, normalize each according to the rules below. Two outputs count as "the same error" when their normalized signatures match exactly.

### Typecheck

Strip the following from the raw output before extracting tokens:

- ANSI color escape codes.
- Blank lines.
- Any line matching `Found N error` (with or without the trailing `s`).

Extract every token of the form `<file-relative-path>:<line>:<col> TS<code>`, followed by the message text up to (and not including) the first newline.

Example token: `src/components/<kebab-name>.tsx:12:5 TS2322 Type 'string' is not assignable to type 'number'.`

The signature is the deduplicated set, sorted lexicographically by the full token string.

### Storybook build

Any line appearing before the first line matching `<ErrorClass>:` (defined below) is excluded from the signature.

Locate the error banner — the first line that matches the regular expression `^[A-Z][a-zA-Z]*Error:` (this covers all standard JS error class names: Error, RangeError, TypeError, SyntaxError, ReferenceError, etc.).

The signature is that single banner line (the first fatal line). Everything after the first newline of the banner is excluded from the signature.

---

## What the repair step is given as context

The repair step receives exactly:

- The spec file — re-read fresh from disk, not taken from the dialogue history.
- The Plan — the most recently approved version.
- The current contents of both emitted files (component file and story file).
- The full output of the failing command (stdout + stderr).

**Not given:** the dialogue history with the user. The dialogue history is not passed to the repair invocation. The repair step must derive everything it needs from the four inputs above.

---

## Termination on (b) — report to the user

When the loop terminates because the same normalized error signature repeated across two attempts for the same command, the skill reports:

1. The error signature that repeated (the normalized form defined above).
2. The skill's diagnosis: "This error pattern repeated across two attempts. The likely cause is either a spec defect or host state the skill cannot resolve. The generated files are left in their last state at `<paths>` for you to inspect."

No further automatic editing is performed after this report.

---

## What this file does NOT cover

- **`storybook dev` is never run.** The skill runs the Storybook build command only. The success report includes the recommended dev command (e.g., `pnpm storybook` or whatever the host's start script is named) so the user can launch Storybook themselves.
- **Unit tests are not run.** Even if the host has a `test` script, tests are out of scope for this slice. The skill runs typecheck and Storybook build only.
