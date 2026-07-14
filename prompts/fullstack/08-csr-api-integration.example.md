---
title: Example — Integrate a REST API (Client-Side / React Query)
type: example
pairs_with: 08-csr-api-integration.md
---

# 📘 Example — Integrate a REST API (Client-Side)

> A fully filled-in run of [`08-csr-api-integration.md`](08-csr-api-integration.md) on a real scenario.
> Copy the prompt block below and paste it into your agent as-is.

---

```text
Follow the `react-client-mastery` skill. Integrate the Users + Invitations API client-side.

CONTRACT: src/shared/api/user/ and src/shared/api/invitation/ (already defined — import it, do not redefine)
ENDPOINTS: users: list / detail — invitations: list / create / revoke
AUTH: httpOnly cookie + refresh on 401
OPTIMISTIC UI REQUIRED FOR: revoke invitation

REPO CONTEXT: `src/lib/http-client.ts` ALREADY exists (axios + interceptors + status→typed
  error mapping + refresh-on-401) — reuse it. Do NOT create a second client or a second
  error type. The query key factory pattern is already established by
  `src/features/billing/api/billing-keys.ts` — copy its shape exactly.
  TanStack Query v5 provider is already mounted; msw handlers live in `src/mocks/handlers/`.

NON-GOALS: No offline support / persisted cache. No websocket or live updates —
  polling and manual invalidation are enough for now.

LAYERS — the dependency arrow points INWARD. Each layer may only import the one below it.

  components/*.tsx           → JSX only. Knows nothing about HTTP.
        ↓
  hooks/use-*.ts             → business rules, React Query wiring
        ↓
  api/invitations-api.ts     → implements the InvitationGateway interface
  api/users-api.ts           → implements the UserGateway interface
        ↓
  lib/http-client.ts         → the ONLY file in the codebase that imports axios/fetch

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
   - Define an `InvitationGateway` interface (and a `UserGateway` interface);
     `invitations-api.ts` and `users-api.ts` implement them.
     Hooks depend on the INTERFACE. Tests inject a fake implementation.

2. SRP — `invitations-api.ts` / `users-api.ts` only speak HTTP: request in, parsed data out.
   No React, no caching decisions, no toasts, no routing.

3. VALIDATE EVERY RESPONSE at the boundary:
   `return InvitationSchema.parse(res.data)` — `as Invitation` is BANNED.
   A backend that changes shape must fail loudly here, not corrupt the UI three
   screens later.

4. Query key factory — centralized, never string literals:
     export const invitationsKeys = {
       all: ['invitations'] as const,
       lists: () => [...invitationsKeys.all, 'list'] as const,
       list: (filters: InvitationQuery) => [...invitationsKeys.lists(), filters] as const,
       details: () => [...invitationsKeys.all, 'detail'] as const,
       detail: (id: string) => [...invitationsKeys.details(), id] as const,
     };
   Mirror the same factory for `usersKeys`.

5. Queries vs mutations — never mix:
   - `useQuery` for reads, `useMutation` for writes.
   - EVERY query sets explicit `staleTime` and `gcTime` based on data volatility.
     State your reasoning per query (e.g. "user list: staleTime 30s — changes rarely,
     but must not look stale after a teammate's edit").
   - EVERY successful mutation calls `queryClient.invalidateQueries` with the
     narrowest key that is still correct.

6. Optimistic updates for revoke invitation — the full three-step pattern, no shortcuts:
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
  src/features/invitations/
  ├── api/invitations-gateway.ts   (interface — the abstraction)
  ├── api/invitations-api.ts       (implementation)
  ├── api/invitations-keys.ts
  ├── hooks/use-invitations-list.ts
  ├── hooks/use-invitations-detail.ts
  ├── hooks/use-create-invitation.ts
  ├── hooks/use-revoke-invitation.ts
  └── (no delete hook — revoke is the only removal path in this API)

  src/features/users/
  ├── api/users-gateway.ts         (interface — the abstraction)
  ├── api/users-api.ts             (implementation)
  ├── api/users-keys.ts
  ├── hooks/use-users-list.ts
  └── hooks/use-users-detail.ts

  + msw handlers derived from the contract
  + tests: success / 500 error / empty / optimistic ROLLBACK on revoke

Print the gateway interface and the key factory first, then wait for my "go".

WHEN THE IMPLEMENTATION IS DONE (not before the gate above):
run typecheck, lint, and the tests, and paste the real output. If something fails, say so plainly
and show it. If you could not run them, say that explicitly. NEVER report "done" on code you have
not executed.
```
