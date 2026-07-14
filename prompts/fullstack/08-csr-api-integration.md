---
title: Integrate a REST API (Client-Side / React Query)
category: FE + BE
skills: [react-client-mastery]
principles: [SRP, DIP, ISP, LSP]
---

# 🔌 Prompt — Integrate a REST API (Client-Side)

> Use for CSR data fetching against an external/BE API with React Query (or SWR).
> Prerequisite: the contract from `07-api-contract.md` exists.
> The golden rule: **the UI never knows that `axios` exists.**

---

```text
Follow the `react-client-mastery` skill. Integrate the {{RESOURCE}} API client-side.

CONTRACT: src/shared/api/{{RESOURCE}}/ (already defined — import it, do not redefine)
ENDPOINTS: {{ENDPOINTS}}
AUTH: {{AUTH}}
OPTIMISTIC UI REQUIRED FOR: {{OPTIMISTIC}}

REPO CONTEXT: {{EXISTING}}
  (Existing conventions to reuse: folder layout, http client, error types, UI primitives,
   test setup. Write "greenfield" if there is none.)

NON-GOALS: {{NON_GOALS}}
  (Do NOT build these. If you believe one is actually required, STOP and ask me first.)

LAYERS — the dependency arrow points INWARD. Each layer may only import the one below it.

  components/*.tsx        → JSX only. Knows nothing about HTTP.
        ↓
  hooks/use-*.ts          → business rules, React Query wiring
        ↓
  api/{{FEATURE}}-api.ts  → implements the {{RESOURCE}}Gateway interface
        ↓
  lib/http-client.ts      → the ONLY file in the codebase that imports axios/fetch

RULES

0. MATCH THE REPO BEFORE YOU MATCH THIS PROMPT.
   The paths in DELIVERABLES below are the default for a greenfield repo. If REPO CONTEXT shows
   an existing convention — folder layout, an http client, an error type, UI primitives, a test
   setup — REUSE IT instead of creating a parallel structure beside it. Creating a second way to
   do something that already exists is a failure, even if the new way follows every rule below.
   List every place you deviated from the tree below, and why.

1. DIP — one centralized, typed HTTP client:
   - `lib/http-client.ts` owns base URL, credentials, timeout, retry, and interceptors.
   - It converts HTTP status → the typed domain errors from the contract
     (401 → UnauthorizedError, 409 → ConflictError, 5xx → ServerError).
   - It attaches auth and handles refresh-on-401 ONCE, centrally.
   - NOTHING else in the app imports axios/fetch. If a component or hook does,
     the layering is broken.
   - Define a `{{RESOURCE}}Gateway` interface; `{{FEATURE}}-api.ts` implements it.
     Hooks depend on the INTERFACE. Tests inject a fake implementation.

2. SRP — `{{FEATURE}}-api.ts` only speaks HTTP: request in, parsed data out.
   No React, no caching decisions, no toasts, no routing.

3. VALIDATE EVERY RESPONSE at the boundary:
   `return {{RESOURCE}}Schema.parse(res.data)` — `as {{RESOURCE}}` is BANNED.
   A backend that changes shape must fail loudly here, not corrupt the UI three
   screens later.

4. Query key factory — centralized, never string literals:
     export const {{FEATURE}}Keys = {
       all: ['{{FEATURE}}'] as const,
       lists: () => [...{{FEATURE}}Keys.all, 'list'] as const,
       list: (filters: {{RESOURCE}}Query) => [...{{FEATURE}}Keys.lists(), filters] as const,
       details: () => [...{{FEATURE}}Keys.all, 'detail'] as const,
       detail: (id: string) => [...{{FEATURE}}Keys.details(), id] as const,
     };

5. Queries vs mutations — never mix:
   - `useQuery` for reads, `useMutation` for writes.
   - EVERY query sets explicit `staleTime` and `gcTime` based on data volatility.
     State your reasoning per query (e.g. "user list: staleTime 30s — changes rarely,
     but must not look stale after a teammate's edit").
   - EVERY successful mutation calls `queryClient.invalidateQueries` with the
     narrowest key that is still correct.

6. Optimistic updates for {{OPTIMISTIC}} — the full three-step pattern, no shortcuts:
   - `onMutate`: `cancelQueries` → snapshot previous → `setQueryData`
   - `onError`:  restore the snapshot (THE ROLLBACK IS MANDATORY)
   - `onSettled`: `invalidateQueries`
   Anything less ships a UI that lies to the user when the network fails.

7. LSP — every hook in this feature returns the same shape family:
   `{ data, isLoading, error }` for queries; `{ mutate, isPending, error }` for mutations.
   Do not have one hook return an array and another an object.

8. ISP — pass narrow props to components; select store slices.
   Never hand a raw API DTO to a presentational component if it needs two fields.

9. ALL FOUR states are wired in the UI: loading (skeleton), error (with retry),
   empty, success. Errors are rendered from the TYPED domain error, not
   `error.response.data.message` dug out at the call site.

10. Auth: tokens live in httpOnly cookies. NEVER localStorage/sessionStorage.

DELIVERABLES
  src/lib/http-client.ts
  src/features/{{FEATURE}}/
  ├── api/{{FEATURE}}-gateway.ts   (interface — the abstraction)
  ├── api/{{FEATURE}}-api.ts       (implementation)
  ├── api/{{FEATURE}}-keys.ts
  ├── hooks/use-{{FEATURE}}-list.ts
  ├── hooks/use-{{FEATURE}}-detail.ts
  ├── hooks/use-create-{{RESOURCE}}.ts
  ├── hooks/use-update-{{RESOURCE}}.ts
  └── hooks/use-delete-{{RESOURCE}}.ts

  + msw handlers derived from the contract
  + tests: success / 500 error / empty / optimistic ROLLBACK

Print the gateway interface and the key factory first, then wait for my "go".

WHEN THE IMPLEMENTATION IS DONE (not before the gate above):
run typecheck, lint, and the tests, and paste the real output. If something fails, say so plainly
and show it. If you could not run them, say that explicitly. NEVER report "done" on code you have
not executed.
```
