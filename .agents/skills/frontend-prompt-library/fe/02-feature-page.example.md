---
title: Example — Build a Client-Side Feature Page
type: example
pairs_with: 02-feature-page.md
---

# 📘 Example — Build a Client-Side Feature Page

> A fully filled-in run of [`02-feature-page.md`](02-feature-page.md) on a real scenario.
> Copy the prompt block below and paste it into your agent as-is.

---

```text
Follow the `react-client-mastery` skill. Build the `users` feature page at `/users`.

REQUIREMENTS
- Entity: User (id, name, email, role: admin|member|viewer, isActive, avatarUrl, createdAt)
- UI state: search query, role filter (admin | member | viewer), page, sort by createdAt
- Data source: GET /api/users?query=&role=&page=&sort=
- User actions: deactivate a user

REPO CONTEXT: `src/features/*` slices already exist (billing, settings) and follow the
  api/hooks/components layering — mirror them exactly. `src/lib/http-client.ts` already
  wraps fetch and maps status codes to typed errors; use it instead of calling fetch/axios.
  `Button` and `EmptyState` primitives already exist in `src/components/`.
  TanStack Query v5 is already configured with a provider at the app root.

NON-GOALS: No bulk actions (select-many + deactivate). No CSV export. No column
  customisation / saved views.

ARCHITECTURE — MANDATORY LAYERING (SRP: one reason to change per file)

  api/users-api.ts        → changes when the ENDPOINT changes
  api/users-schema.ts     → Zod schemas + inferred types
  hooks/use-users-list.ts → changes when BUSINESS RULES change
  components/*.tsx        → changes when the DESIGN changes

  The .tsx files contain JSX ONLY. Zero useState, zero useEffect, zero fetch,
  zero filtering/sorting logic. If a component has logic, you have failed this rule.

RULES

0. MATCH THE REPO BEFORE YOU MATCH THIS PROMPT.
   The paths in DELIVERABLES below are the default for a greenfield repo. If REPO CONTEXT shows
   an existing convention — folder layout, an http client, an error type, UI primitives, a test
   setup — REUSE IT instead of creating a parallel structure beside it. Creating a second way to
   do something that already exists is a failure, even if the new way follows every rule below.
   List every place you deviated from the tree below, and why.

1. SRP — every file must be describable in one sentence WITHOUT the word "and".

2. URL as the source of truth:
   - The search query, role filter, page and sort that are shareable/bookmarkable
     MUST live in `useSearchParams`, NOT `useState`.
   - Only ephemeral UI state (dropdown open, hover) may use `useState`.

3. Derived state during render:
   - NEVER sync state with `useEffect`. Compute filtered/sorted/derived values
     directly in the hook body (wrapped in `useMemo` only when measurably heavy).

4. Colocated state:
   - All state that must update TOGETHER lives in the SAME custom hook.
     Do not split tightly-coupled state across hooks — it desyncs.

5. Data layer (DIP):
   - The hook depends on `usersApi`, never on `axios`/`fetch` directly.
   - Validate every API response with Zod (`Schema.parse`). NEVER `as User`.
   - React Query: centralized key factory (usersKeys.all / .lists() / .detail(id)),
     `useQuery` for reads, `useMutation` for writes, explicit `staleTime` + `gcTime`,
     `invalidateQueries` after every successful mutation.
   - For UX-critical toggles, use optimistic updates (onMutate + rollback in onError).
     Deactivating a user is exactly such a toggle.

6. ISP — pass narrow props. `<UserCard name={u.name} email={u.email} />`,
   not `<UserCard user={u} />`, unless the child genuinely needs the whole entity.
   Zustand reads MUST use a slice selector: `useStore(s => s.isOpen)`, never `useStore()`.

7. ALL FOUR data states are required: loading (Skeleton mirroring the layout,
   not a spinner), error (<ErrorState> + retry), empty (<EmptyState>), success.

8. Performance: `Promise.all` for independent fetches (no waterfalls), debounce
   the search input (300ms), throttle scroll, `{ passive: true }` on scroll listeners,
   `next/dynamic` for heavy widgets (charts, editors).

DELIVERABLES
  src/features/users/
  ├── api/users-api.ts
  ├── api/users-schema.ts
  ├── api/users-keys.ts
  ├── hooks/use-users-list.ts
  ├── hooks/use-users-filters.ts   (URL state)
  ├── components/UserList.tsx
  ├── components/UserCard.tsx
  ├── components/UserFilters.tsx
  ├── components/UserListSkeleton.tsx
  └── index.ts                      (public API — internals stay private)

Start by printing the file tree and one sentence per file stating its single
reason to change. Wait for my "go" before writing the implementation.

WHEN THE IMPLEMENTATION IS DONE (not before the gate above):
run typecheck, lint, and the tests, and paste the real output. If something fails, say so plainly
and show it. If you could not run them, say that explicitly. NEVER report "done" on code you have
not executed.
```
