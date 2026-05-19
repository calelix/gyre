# docs/issues

This directory accumulates records of issues that surfaced during project work, their root causes, and the remediation applied. The point is to leave a trail so the same failure does not recur.

> **Intent.** Each entry should make the *next* unit of work easier than this one was. If a record reads only as a report of what happened, it has missed its job; it should also leave the next contributor better equipped to avoid the same kind of failure.

## Filename convention

`YYYY-MM-DD-<slug>.md`

- `slug` is kebab-case, ASCII.
- Prefer a noun phrase describing the *thing*, not a verb describing the action (e.g., ❌ `fix-where-question`, ⭕ `clarify-component-where-misses-primitives`).
- Six words or fewer is the working limit.

## Template

Copy the template below into a new issue file and fill it in.

````markdown
---
date: YYYY-MM-DD
slug: <kebab-case>
title: <one-line summary>
status: resolved | partial | open
---

# <title>

## Summary
One or two sentences.

## Context
- Playground: <path, e.g., experiments/shadcn>
- Tech stack: <target stack>
- Detected by: <how it surfaced — user feedback / self-check / retrospective / ...>

## Problem
State the expected behavior and the actual behavior, distinctly.

## Root cause
The structural reason, not the surface symptom. Number them if there are several.

## Remediation
Concrete changes. Link the files / PRs / commits that changed.

## Verification
Evidence that the change actually closed the problem — re-runs, tests, artifact checks.

## Follow-ups
(Optional) Outstanding items not closed by this entry. Candidates for new issues.

## Related
(Optional) Links to related issues, skills, or documents.
````

## Status semantics

- `resolved` — Remediation applied and verified.
- `partial` — Partially closed; follow-up work remains (see Follow-ups).
- `open` — Not yet closed.

## Index

While the count is small, no separate index is kept. Filename sort and grepping by `status:` are enough. When the count grows, an index section will be added to this README.
