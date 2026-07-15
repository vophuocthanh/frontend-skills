---
title: Example — Build a Reusable UI Component
type: example
pairs_with: 01-ui-component.md
---

# 📘 Example — Build a Reusable UI Component

> A fully filled-in run of [`01-ui-component.md`](01-ui-component.md) on a real scenario.
> Copy the prompt block below and paste it into your agent as-is.

---

```text
Follow the `react-client-mastery` skill. Build a reusable `Button` component.

CONTEXT
- Base HTML element: `<button>`
- Variants required: `intent: primary | danger | ghost`, `size: sm | md | lg`
- Composition slots: `icon-left`, `icon-right`
- Design source: Acme Console design system — primary = solid indigo-600 / white text;
  danger = solid red-600 / white text; ghost = transparent with slate-700 text and a
  slate-100 hover; sizes sm = h-8 text-sm, md = h-10 text-sm, lg = h-12 text-base;
  radius `rounded-md`; icons are 16px (sm/md) or 20px (lg), inline before/after the label.
  This Button will be consumed by 8 features (users, invitations, billing, settings,
  audit-log, integrations, roles, onboarding), including inside `<form>` and dialogs.

REPO CONTEXT: `src/components/` is where the design-system primitives live (Card, Badge,
  Spinner already there). A `cn()` helper (clsx + tailwind-merge) already exists at
  `src/lib/cn.ts` — use it, do not add a second class-merging util. There is NO Button
  primitive yet: every feature hand-rolls its own `<button className="...">`.
  Tailwind + `cva` are already installed; tests are Vitest + RTL.

NON-GOALS: No Button theming or dark-mode work. No icon-library swap (keep lucide-react).
  Do NOT go and rewrite the other features' hand-rolled buttons yet — that is a follow-up PR.

NON-NEGOTIABLE CONSTRAINTS (from the skill)

0. MATCH THE REPO BEFORE YOU MATCH THIS PROMPT.
   The paths in DELIVERABLES below are the default for a greenfield repo. If REPO CONTEXT shows
   an existing convention — folder layout, an http client, an error type, UI primitives, a test
   setup — REUSE IT instead of creating a parallel structure beside it. Creating a second way to
   do something that already exists is a failure, even if the new way follows every rule below.
   List every place you deviated from the tree below, and why.

1. LSP — the component MUST be substitutable for `<button>`:
   - Type props as `React.ComponentPropsWithoutRef<'button'> & VariantProps<typeof ...>`.
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
- `src/components/Button/Button.tsx`  (PascalCase file, rendering only)
- `src/components/Button/Button.variants.ts` (kebab-case, the `cva` map)
- `src/components/Button/index.ts` (public API — feature-level barrel is allowed)
- `src/components/Button/Button.test.tsx` (see below)

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
