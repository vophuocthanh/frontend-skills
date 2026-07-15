---
title: Example — Debug an FE/BE Integration Issue
type: example
pairs_with: 14-debug-integration.md
---

# 📘 Example — Debug an FE/BE Integration Issue

> A fully filled-in run of [`14-debug-integration.md`](14-debug-integration.md) on a real scenario.
> Copy the prompt block below and paste it into your agent as-is.

---

```text
Follow `react-client-mastery` + `nextjs-server-mastery`. Debug this integration bug.

SYMPTOM:  after revoking an invitation the row disappears, but navigating to /settings and back makes the revoked invitation reappear as pending
EXPECTED: the revoked invitation stays gone (or shows status=revoked) after navigating away and back
EVIDENCE: network tab: POST /invitations/inv_123/revoke → 200 OK; no console errors; React Query devtools show the ['invitations','list',{status:'pending'}] query still holding the old row
REPRO:    1. open /team  2. revoke a pending invitation  3. navigate to /settings  4. navigate back to /team → the invitation is back, status pending

REPO CONTEXT: the query key factory already exists at
  `src/features/invitations/api/invitation-keys.ts` — the invalidation MUST go through it,
  not through a hand-written key array. Billing already invalidates correctly through its own
  factory (`src/features/billing/api/billing-keys.ts`), so invitations is the odd one out —
  compare the two. TanStack Query v5; msw handlers in `src/mocks/handlers/`.

NON-GOALS: Do NOT refactor the invitations feature while fixing this. Fix the root cause only.

DO NOT CHANGE ANY CODE YET. Diagnose first.

RULE 0 — MATCH THE REPO BEFORE YOU MATCH THIS PROMPT.
If REPO CONTEXT shows an existing convention — folder layout, an http client, an error type,
UI primitives, a test setup — REUSE IT instead of creating a parallel structure beside it.
Creating a second way to do something that already exists is a failure, even if the new way
follows every rule below. Judge the code against the conventions this repo actually uses,
not against a greenfield ideal.

═══ PHASE 1 — LOCATE THE LAYER ═══
Walk the request end to end and tell me the FIRST layer where reality diverges
from expectation. State what you'd expect vs what actually happens at each hop:

  1. UI event        → is the handler firing? with what payload?
  2. Hook/mutation   → what exactly is sent (log the parsed input)?
  3. HTTP client     → method, URL, headers, cookies, body. Is auth attached?
  4. Network         → status, response body. Does it match the contract?
  5. Boundary parse  → does `Schema.parse` succeed? (a silent `as T` cast hides this)
  6. Cache           → was the right query key invalidated? is stale data being served?
  7. Render          → is the component reading the state it thinks it is?

Name the layer. Everything after it is a symptom, not the cause.

═══ PHASE 2 — CHECK THE USUAL SUSPECTS FOR THIS CLASS OF BUG ═══

CONTRACT DRIFT (the #1 cause of "works in Postman")
  - Is the response `Schema.parse`d, or cast with `as T`? A cast means the app has
    been lying about the shape — parse it and watch it fail loudly.
  - Does the BE return `null` where the schema says required? camelCase vs snake_case?
    A date as a string where a Date is expected? Money as a float?
  - Is there a wrapper envelope (`{ data: ... }`) the FE forgot to unwrap?

STALE DATA AFTER MUTATION (LSP/cache)
  - Missing `invalidateQueries` — or invalidating a DIFFERENT key than the one the
    query uses (a string-literal key typo the factory would have prevented).
  - Server Action succeeded but no `revalidatePath`/`revalidateTag`.
  - Optimistic update applied but never reconciled in `onSettled`.
  - Router cache serving a prefetched RSC payload from before the mutation.

AUTH / 401
  - Cookie not sent: missing `credentials: 'include'`, wrong domain, `SameSite` too strict.
  - Token read client-side (it can't be — it's httpOnly, by design).
  - Middleware redirected but the action still ran (or vice versa).

HYDRATION MISMATCH
  - `Date.now()`, `Math.random()`, `typeof window`, or locale formatting during render.
  - A `Date` object or function passed across the RSC boundary (not serializable).
  - Server and client rendering different branches of a conditional.

RACE / ORDERING
  - Two mutations in flight, last-write-wins clobbering.
  - Debounced search resolving out of order (an older response landing last).
  - Optimistic rollback restoring a snapshot taken AFTER another update.

═══ PHASE 3 — PROVE IT ═══
- Write a FAILING TEST that reproduces "after revoking an invitation the row disappears,
  but navigating to /settings and back makes the revoked invitation reappear as pending"
  at the layer you identified
  (msw for a contract bug; a mutation test for a cache bug).
- Show me it fails for the RIGHT REASON (paste the failure output).

═══ PHASE 4 — FIX ═══
- Fix the ROOT CAUSE, not the symptom.
  Symptom fixes I will reject: `refetch()` sprinkled after a mutation instead of
  correct invalidation; `as any` to silence a parse error; a `setTimeout` to
  "wait for" the cache; `key={Math.random()}` to force a remount.
- If the ROOT CAUSE is in the backend contract, say so plainly and give me the exact
  request/response evidence I can hand to the BE team. Do not paper over it on the
  client — but if we must ship a temporary boundary transform, isolate it in the
  api layer, comment WHY, and never let the bad shape leak past the boundary.

═══ PHASE 5 — PREVENT ═══
- What guardrail stops this whole CLASS of bug? (Zod at the boundary, a key factory,
  a shared contract package, a typed error union.) Propose ONE and tell me its cost.

Do PHASE 1 and PHASE 2 only. Then stop and tell me the layer and your top hypothesis.

WHEN YOU FIX IT (PHASE 4, after I approve the diagnosis): run the failing test you wrote and
paste the output showing it now passes, plus the full suite to prove you broke nothing.
NEVER report a bug fixed on code you have not executed.
```
