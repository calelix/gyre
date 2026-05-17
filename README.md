# Gyre

An AI Harness for frontend UI/UX, distributed as a Claude Code Plugin.

## Why

Frontend development typically routes through three roles — product manager, designer, and developer — and the back-and-forth between them is slow and prone to conflict driven by differing perspectives. When that conflict is healthy it lifts quality; when it isn't, it just burns energy.

Gyre replaces that structure with **a single director ↔ AI (acting as PM, designer, and developer)**. The quality lift that used to come from human-to-human dialogue now has to be delivered by human-to-AI dialogue.

## What "Done" Looks Like

Given a natural-language request for a UI component or page, Gyre refines the under-specified intent through dialogue and produces one of:

- **A custom UI component** built on top of Headless UI primitives (Base UI, Radix UI) that conforms to the design guide, or
- **A design guide itself** (`DESIGN.md`).

Stable execution of this behavior marks completion. Beyond that point, the goal is to systematically manage and extend the component library while the AI keeps learning and evolving.
