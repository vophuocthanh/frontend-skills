---
title: Build a Reusable UI Component
category: FE
skills: [react-client-mastery]
principles: [OCP, LSP, ISP]
---

# 🧩 Prompt — Build a Reusable UI Component

> Use when creating a **shared, design-system-level** component (`Button`, `Input`, `Modal`, `DataTable`, `Card`) that will be consumed by 3+ features.
> Do **not** use this for one-off feature components — those belong in the feature folder (see `02-feature-page.md`).

---

```text
Follow the `react-client-mastery` skill. Build a reusable `{{COMPONENT}}` component.

CONTEXT
- Base HTML element: `<{{BASE_ELEMENT}}>`
- Variants required: {{VARIANTS}}
- Composition slots: {{SLOTS}}
- Design source: {{DESIGN_SOURCE}}

REPO CONTEXT: {{EXISTING}}
  (Existing conventions to reuse: folder layout, http client, error types, UI primitives,
   test setup. Write "greenfield" if there is none.)

NON-GOALS: {{NON_GOALS}}
  (Do NOT build these. If you believe one is actually required, STOP and ask me first.)

NON-NEGOTIABLE CONSTRAINTS (from the skill)

0. MATCH THE REPO BEFORE YOU MATCH THIS PROMPT.
   The paths in DELIVERABLES below are the default for a greenfield repo. If REPO CONTEXT shows
   an existing convention — folder layout, an http client, an error type, UI primitives, a test
   setup — REUSE IT instead of creating a parallel structure beside it. Creating a second way to
   do something that already exists is a failure, even if the new way follows every rule below.
   List every place you deviated from the tree below, and why.

1. LSP — the component MUST be substitutable for `<{{BASE_ELEMENT}}>`:
   - Type props as `React.ComponentPropsWithoutRef<'{{BASE_ELEMENT}}'> & VariantProps<typeof ...>`.
   - Wrap in `forwardRef` and forward the `ref`.
   - Spread `{...rest}` onto the base element so `disabled`, `type`, `aria-*`,
     and all native handlers keep working.
   - NEVER hand-pick a few props (`{ label, onClick }`) — that breaks forms and dialogs.

2. OCP — open for extension, closed for modification:
   - Model variants with a `cva` variant map. Adding a new visual style must mean
     adding a MAP ENTRY, never a new `if` branch inside the render body.
   - Expose composition via `children` / slots / compound components.
   - ZERO boolean props like `isPrimary`, `isLarge`, `showIcon`.
   - Accept a `className` prop and merge it LAST (via `cn`/`twMerge`) so consumers
     can override without forking the component.

3. ISP — narrow props only. Never accept a god-object (`user`, `config`) when
   `src` + `name` would do. If the prop surface covers unrelated concerns,
   split it into separate interfaces and intersect them.

4. SRP — this file renders. No data fetching, no business logic, no `useQuery`.
   Any internal behavior (open/close, focus trap) goes in a colocated custom hook.

5. A11y (WCAG 2.1 AA):
   - Semantic element first; ARIA only for custom interactive widgets.
   - Visible `:focus-visible` ring; keyboard reachable (Tab / Enter / Escape).
   - Icon-only variant REQUIRES `aria-label`; decorative icons get `aria-hidden="true"`.
   - Touch target >= 44x44px. Contrast >= 4.5:1 (3:1 for large text).
   - If it is an overlay (modal/drawer): trap focus, restore focus on close,
     close on Escape, mark the background `aria-hidden`.

6. Styling — TailwindCSS + `cva`. No inline `style={{}}` for layout/spacing/color.

7. TypeScript — `strict`. No `any`, no unsafe assertions. Export the props type.

DELIVERABLES
- `src/components/{{COMPONENT}}/{{COMPONENT}}.tsx`  (PascalCase file, rendering only)
- `src/components/{{COMPONENT}}/{{COMPONENT}}.variants.ts` (kebab-case, the `cva` map)
- `src/components/{{COMPONENT}}/index.ts` (public API — feature-level barrel is allowed)
- `src/components/{{COMPONENT}}/{{COMPONENT}}.test.tsx` (see below)

TESTS (React Testing Library, query by ROLE not by test-id)
- renders each variant
- forwards `ref` to the underlying DOM node
- forwards native props (`disabled`, `type="submit"`, `aria-label`)
- keyboard interaction works
- a11y: has an accessible name

Before writing code, state in 3 bullets HOW this component stays closed for
modification when the next variant arrives. Then write the code.

WHEN THE IMPLEMENTATION IS DONE (not before the gate above):
run typecheck, lint, and the tests, and paste the real output. If something fails, say so plainly
and show it. If you could not run them, say that explicitly. NEVER report "done" on code you have
not executed.
```
