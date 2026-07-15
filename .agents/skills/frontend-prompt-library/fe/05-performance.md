---
title: Performance Audit & Optimization
category: FE
skills: [react-client-mastery, nextjs-server-mastery]
principles: [ISP]
---

# ⚡ Prompt — Performance Audit & Optimization

> Use when a page is slow: janky typing, long TTI, huge bundle, re-render storms, network waterfalls.
> Rule of the house: **measure before optimizing**. No speculative `memo()` sprinkling.

---

```text
Follow the `react-client-mastery` skill (and `nextjs-server-mastery` if this route
has a server component). Audit and optimize `{{TARGET}}`.

SYMPTOM: {{SYMPTOM}}
BUDGET:  {{BUDGET}}

REPO CONTEXT: {{EXISTING}}
  (Existing conventions to reuse: folder layout, http client, error types, UI primitives,
   test setup. Write "greenfield" if there is none.)

NON-GOALS: {{NON_GOALS}}
  (Do NOT build these. If you believe one is actually required, STOP and ask me first.)

RULE 0 — MATCH THE REPO BEFORE YOU MATCH THIS PROMPT.
If REPO CONTEXT shows an existing convention — folder layout, an http client, an error type,
UI primitives, a test setup — REUSE IT instead of creating a parallel structure beside it.
Creating a second way to do something that already exists is a failure, even if the new way
follows every rule below. Judge the code against the conventions this repo actually uses,
not against a greenfield ideal.

PHASE 1 — MEASURE FIRST (no fixes yet)
Identify the ACTUAL bottleneck and classify it. Report findings as:

  | # | Category | Evidence (file:line) | Est. cost | Fix |
  |---|----------|----------------------|-----------|-----|

Audit these categories IN ORDER — the earlier ones dominate:

A. NETWORK WATERFALLS (usually the biggest win)
   - Sequential `await` on independent calls → must be `Promise.all()`.
   - Client fetch that fires only after a parent's fetch resolves (request chain).
   - Server: missing `React.cache()` dedup; the same query run in layout + page + sidebar.

B. BUNDLE SIZE
   - Library barrel imports killing tree-shaking:
     `import { Button } from '@mui/material'` → `import Button from '@mui/material/Button'`.
     (Feature-level `index.ts` barrels are fine — do NOT remove those.)
   - Heavy components (charts, editors, maps, date pickers) not behind `next/dynamic`.
   - Moment/lodash-style full-package imports.
   - `'use client'` sitting at the page/layout level, dragging the whole tree into the bundle.
     Push it down to the leaves.

C. RENDER STORMS
   - Controlled inputs re-rendering a large tree on every keystroke → uncontrolled (RHF)
     or isolate the input into its own leaf component.
   - Missing debounce (input, resize) / throttle (scroll, mousemove).
   - Scroll/wheel/touch listeners without `{ passive: true }`.
   - Context used for HIGH-FREQUENCY state → move to Zustand with slice selectors.
     ISP: `useStore(s => s.x)`, never `useStore()`.
   - High-frequency values kept in `useState` when they never hit the DOM → `useRef`.
   - Inline objects/functions passed to `memo()` children, breaking memoization.
   - `useEffect` syncing derived state → causes a guaranteed double render.

D. RENDER COST
   - Static JSX rebuilt every render → hoist out of the component body.
   - Expensive computation without `useMemo` (only after proving it's expensive).
   - Long lists without virtualization.
   - Interleaved DOM reads/writes → layout thrashing.

E. ASSETS
   - Raw `<img>` instead of `next/image`; missing width/height (CLS);
     missing `priority` on the LCP image; PNG where WebP/AVIF belongs.

PHASE 2 — FIX
- Fix in impact order (A → E). One concern per commit.
- For each fix, state the expected mechanism ("removes a 300ms serial round-trip",
  "drops 180KB from the initial chunk") — not just "faster".
- Do NOT add `memo`/`useMemo`/`useCallback` speculatively. Only where a measured
  re-render or a proven expensive computation justifies it, and say which.

PHASE 3 — VERIFY
- Re-measure against {{BUDGET}}. Show before/after numbers.
- If a fix did NOT help, revert it and say so. Dead optimizations are tech debt.

Do PHASE 1 only, then stop.

WHEN THE IMPLEMENTATION IS DONE (not before the gate above):
run typecheck, lint, and the tests, and paste the real output. If something fails, say so plainly
and show it. If you could not run them, say that explicitly. NEVER report "done" on code you have
not executed.
```
