---
title: Write Tests (Testing Trophy)
category: FE
skills: [react-client-mastery]
principles: [DIP, LSP]
---

# 🧪 Prompt — Write Tests

> Follows the **Testing Trophy**: integration tests carry the weight; unit tests cover hooks/utils/schemas; E2E covers only critical paths.
> Mock at the **network boundary** (`msw`) — never mock your own modules.

---

```text
Follow the `react-client-mastery` skill. Write tests for `{{TARGET}}`.

USER FLOWS: {{FLOWS}}
CRITICAL PATH (E2E): {{CRITICAL_PATH}}

REPO CONTEXT: {{EXISTING}}
  (Existing conventions to reuse: folder layout, http client, error types, UI primitives,
   test setup. Write "greenfield" if there is none.)

NON-GOALS: {{NON_GOALS}}
  (Do NOT build these. If you believe one is actually required, STOP and ask me first.)

STRATEGY — Testing Trophy, in this proportion:

0. MATCH THE REPO BEFORE YOU MATCH THIS PROMPT.
   The paths in DELIVERABLES below are the default for a greenfield repo. If REPO CONTEXT shows
   an existing convention — folder layout, an http client, an error type, UI primitives, a test
   setup — REUSE IT instead of creating a parallel structure beside it. Creating a second way to
   do something that already exists is a failure, even if the new way follows every rule below.
   List every place you deviated from the tree below, and why.

1. INTEGRATION (the bulk — React Testing Library)
   Test the user flow through the real component tree with real hooks.
   - Query by ROLE / LABEL / TEXT. `getByTestId` is a last resort — if you need it,
     the component probably has an a11y bug; fix the component instead.
   - Drive with `userEvent`, not `fireEvent`.
   - Mock ONLY at the network boundary with `msw` (`http.get('/api/users', ...)`).
     NEVER `vi.mock('@/features/users/api/users-api')` — mocking your own modules
     tests the mock, not the code.
   - Assert what the USER sees (text, roles, aria-live), not internal state.
   - No `waitFor(() => expect(mockFn).toHaveBeenCalled())` as the main assertion —
     assert the rendered outcome.

2. UNIT (fast, focused)
   - Custom hooks via `renderHook` — the hook IS the business logic, so this is
     where the logic coverage lives (skill target: 80%+ on business-logic hooks).
   - Pure utils and formatters.
   - Zod schemas: valid input parses; invalid input produces the expected field errors.

3. E2E (Playwright — only {{CRITICAL_PATH}})
   Real browser, real navigation. Keep the count small; these are slow and flaky-prone.

MANDATORY COVERAGE FOR EVERY DATA-DRIVEN VIEW
Every one of the four states must have a test:
   - loading  → skeleton visible
   - error    → error state + retry works (msw returns 500)
   - empty    → empty state (msw returns [])
   - success  → data rendered

MANDATORY COVERAGE FOR EVERY MUTATION
   - happy path → API called once with the correct payload; cache invalidated;
                  UI reflects the new state
   - failure    → error surfaced to the user (role="alert"); optimistic update
                  ROLLED BACK (this is the bug optimistic UI always ships with)
   - pending    → submit disabled / spinner shown

MANDATORY COVERAGE FOR SHARED COMPONENTS (LSP)
   - `ref` is forwarded to the DOM node
   - native props pass through (`disabled`, `type="submit"`, `aria-label`)

DIP payoff: if a hook takes its gateway/api as an injected dependency, unit-test it
with a fake — no msw, no network. Use that where the seam exists.

CONVENTIONS
- Colocate: `Component.test.tsx` next to `Component.tsx`.
- One `describe` per behavior, not per method.
- Test names read as user-facing sentences:
  ✅ 'shows an error when credentials are invalid'
  ❌ 'test handleSubmit returns false'
- No snapshot tests for logic. Snapshots are for stable, purely-visual markup only.
- Deterministic: fake timers for debounce, fixed dates. No `sleep()`.

DELIVERABLES
List the test files you will create and the cases in each, then wait for my "go".

WHEN THE IMPLEMENTATION IS DONE (not before the gate above):
run typecheck, lint, and the tests, and paste the real output. If something fails, say so plainly
and show it. If you could not run them, say that explicitly. NEVER report "done" on code you have
not executed.
```
