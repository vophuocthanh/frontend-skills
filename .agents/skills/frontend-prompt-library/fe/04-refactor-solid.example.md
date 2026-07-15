---
title: Example — Refactor a Legacy Component to SOLID
type: example
pairs_with: 04-refactor-solid.md
---

# 📘 Example — Refactor a Legacy Component to SOLID

> A fully filled-in run of [`04-refactor-solid.md`](04-refactor-solid.md) on a real scenario.
> Copy the prompt block below and paste it into your agent as-is.

---

```text
Follow the `react-client-mastery` skill. Refactor `src/components/UserDashboard.tsx` to comply with SOLID.

CONSTRAINTS
- 380 lines; `axios` called inline; a `useEffect` syncing derived state; 6 boolean props
  (isCompact, isAdminView, showFilters, showAvatars, hideInactive, isReadOnly);
  used in 6 places; the public props MUST NOT change.
- Existing tests: no tests exist yet
- BEHAVIOR MUST NOT CHANGE. This is a pure refactor. No new features, no bug fixes
  (if you spot a bug, LIST it separately — do not silently fix it).

REPO CONTEXT: this repo has already migrated 3 other features (billing, settings, audit-log)
  to the api/hooks/components layering under `src/features/*` — land `UserDashboard` in the
  same shape, it is the last hold-out. `src/lib/http-client.ts` is the shared client and
  msw handlers live in `src/mocks/handlers/` (add the characterization-test handlers there).

NON-GOALS: No new features and no bug fixes during the refactor — behavior is frozen.
  Do NOT change the public props of `UserDashboard`; all 6 call sites must keep compiling.

RULE 0 — MATCH THE REPO BEFORE YOU MATCH THIS PROMPT.
If REPO CONTEXT shows an existing convention — folder layout, an http client, an error type,
UI primitives, a test setup — REUSE IT instead of creating a parallel structure beside it.
Creating a second way to do something that already exists is a failure, even if the new way
follows every rule below. Judge the code against the conventions this repo actually uses,
not against a greenfield ideal.

PHASE 1 — DIAGNOSE (no code yet)
Produce a table of violations found in the file:

  | Principle | Violation | Evidence (line) | Fix |
  |---|---|---|---|

Check specifically for:
- SRP: how many reasons to change does this file have? Name each one.
        (fetching, business rules, formatting, rendering, analytics, routing…)
- OCP: `if (variant === ...)` / `if (isPrimary)` chains inside the render body.
- LSP: wrapper elements that swallow `ref`, `disabled`, `type`, `aria-*`, `...rest`.
        Hook variants returning different shapes.
- ISP: god-object props (`<Child user={user} />`), whole-store subscriptions
        (`useStore()` with no selector), fat prop interfaces mixing concerns.
- DIP: direct imports of `axios` / `fetch` / `localStorage` / SDKs inside the UI.

Also flag skill violations that are not SOLID but matter:
- `useEffect` used to derive/sync state
- state that belongs in the URL sitting in `useState`
- unvalidated API responses (`as User`)
- missing loading/error/empty states
- `any` types

PHASE 2 — SAFETY NET
No tests exist yet.
If no tests exist: FIRST write characterization tests (RTL, query by role, msw at
the network boundary) that pin down the CURRENT behavior — including the ugly parts.
Run them green BEFORE touching the implementation.

PHASE 3 — REFACTOR (one commit per step, tests green after each)
1. Extract the data layer   → `api/*-api.ts` + Zod schema (DIP + validation)
2. Extract business logic   → `hooks/use-*.ts` (SRP: logic-in-hook)
3. Replace derived-state `useEffect` with render-time computation
4. Replace boolean prop chains with a `cva` variant map / composition (OCP)
5. Restore substitutability: forwardRef + ComponentPropsWithoutRef + {...rest} (LSP)
6. Narrow the props; add store selectors (ISP)
7. Split the remaining JSX into presentational components < 150 lines each
8. Delete dead code

PHASE 4 — VERIFY
- All characterization tests still green.
- Print a before/after summary: file count, longest file, reasons-to-change per file.
- List any bug you found but did NOT fix, so I can triage it.

Do PHASE 1 only, then stop and wait for my approval.

WHEN THE IMPLEMENTATION IS DONE (not before the gate above):
run typecheck, lint, and the tests, and paste the real output. If something fails, say so plainly
and show it. If you could not run them, say that explicitly. NEVER report "done" on code you have
not executed.
```
