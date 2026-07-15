---
title: Refactor a Legacy Component to SOLID
category: FE
skills: [react-client-mastery]
principles: [SRP, OCP, LSP, ISP, DIP]
---

# ♻️ Prompt — Refactor a Legacy Component to SOLID

> Use on a god-component: 300+ lines, fetching + state + business logic + JSX all mixed, boolean prop explosion, `axios` imported inline.
> The refactor is **behavior-preserving**. If behavior must change, that is a separate PR.

---

```text
Follow the `react-client-mastery` skill. Refactor `{{TARGET}}` to comply with SOLID.

CONSTRAINTS
- {{CONSTRAINTS}}
- Existing tests: {{HAS_TESTS}}
- BEHAVIOR MUST NOT CHANGE. This is a pure refactor. No new features, no bug fixes
  (if you spot a bug, LIST it separately — do not silently fix it).

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
{{HAS_TESTS}}
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
