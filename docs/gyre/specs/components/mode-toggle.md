---
name: mode-toggle
classification: component
created: 2026-05-19
source: "@experiments/shadcn/ 의 메인 페이지에서 d 키를 이용해서 다크모드를 토글하는데, 테마 토글 버튼을 만들어야한다."
---

# ModeToggle

## Summary
An icon-only button that toggles the application theme between light and dark on click. It is composed on top of the existing Button primitive in the UI package (as an icon-sized variant); theme state is read from and written to an external ThemeProvider; it is reused across the main page and other pages.

## Where
- Decision: Compose
- Reference: Button (the base button component already defined in the UI package)
- Rationale: No same-role component exists, but the visual, interaction, and accessibility foundation is already provided by the existing Button primitive. ModeToggle assembles on top of Button by adding icon-swap behavior.

## What
- Purpose: An icon button that toggles the application theme between light and dark in a single click.
- Usage scenarios:
  - Placed on the main page as the entry point for switching themes.
  - Reused on other pages as well (e.g., shared layouts, header).
- Props:

  | name | type | required | default | description |
  |---|---|---|---|---|
  | className | string | no | — | Class for overriding button styling from the outside. Internally merged into the underlying Button. |

  Other Button APIs (`variant`, `size`, `onClick`, etc.) are not re-exposed. They are fixed internally to the icon variant.

## Look
(compressed: existing design guide covers visual/spacing/responsive tokens)

- Token overrides: none

## How
- States: default, hover, focus-visible, active — all state styles are inherited from the base Button primitive. ModeToggle does not define state styles of its own.
- Interactions: The base Button's standard click and keyboard (Enter/Space) activation is used as is. ModeToggle attaches an onClick handler that calls the external ThemeProvider's setTheme and toggles light ↔ dark. The Sun and Moon icons are mounted simultaneously inside the button; depending on the current theme, one is in the rotate(0deg) · scale(1) · opacity(1) state and the other is in rotate(90deg) · scale(0) · opacity(0), and they exchange smoothly via CSS transitions.
- Accessibility: Inherits the semantic `<button>` element, focus-visible focus ring, and color-contrast rules from the base Button. To compensate for the icon-only presentation, ModeToggle adds a dynamic aria-label that switches with the current theme between "Switch to dark mode" and "Switch to light mode".
- Edge cases: When `prefers-reduced-motion: reduce` is set, the rotate/scale/opacity transition for the icon swap is disabled and the swap is immediate.

## Data / Content
(compressed: low risk; presentational role)

- Schema: Reads a single `"light" | "dark"` string from the external ThemeProvider and writes the same type on toggle. No domain data binding.

## Non-goals
- Does not support a `system` option (auto-follow OS theme) — light/dark only.
- Does not register the 'd' keyboard shortcut from inside the component — left to the page or a higher layer.
- Does not handle theme persistence (localStorage, etc.) — the external ThemeProvider's responsibility.
- Does not provide dropdowns, menus, or multi-option selection UI — single-action toggle button only.
- Does not reimplement the base Button's visual, interaction, or accessibility rules — all delegated to Button.
