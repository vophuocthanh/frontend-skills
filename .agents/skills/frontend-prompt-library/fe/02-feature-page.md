---
title: Build a Client-Side Feature Page
category: FE
skills: [react-client-mastery]
principles: [SRP, ISP, DIP]
---

# 📄 Prompt — Build a Client-Side Feature Page

> Use when building a complete CSR screen: list + filters + pagination + detail, with loading/empty/error states.
> This is the **Logic-in-Hook, UI-in-Component** prompt — the single most important structural rule in the skill.

---

```text
Follow the `react-client-mastery` skill. Build the `{{FEATURE}}` feature page at `{{ROUTE}}`.

REQUIREMENTS
- Entity: {{ENTITY}}
- UI state: {{UI_STATE}}
- Data source: {{DATA_SOURCE}}
- User actions: {{ACTIONS}}

REPO CONTEXT: {{EXISTING}}
  (Existing conventions to reuse: folder layout, http client, error types, UI primitives,
   test setup. Write "greenfield" if there is none.)

NON-GOALS: {{NON_GOALS}}
  (Do NOT build these. If you believe one is actually required, STOP and ask me first.)

ARCHITECTURE — MANDATORY LAYERING (SRP: one reason to change per file)

  api/{{FEATURE}}-api.ts        → changes when the ENDPOINT changes
  api/{{FEATURE}}-schema.ts     → Zod schemas + inferred types
  hooks/use-{{FEATURE}}-list.ts → changes when BUSINESS RULES change
  components/*.tsx              → changes when the DESIGN changes

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
   - {{UI_STATE}} that is shareable/bookmarkable (filters, pagination, sort, tabs)
     MUST live in `useSearchParams`, NOT `useState`.
   - Only ephemeral UI state (dropdown open, hover) may use `useState`.

3. Derived state during render:
   - NEVER sync state with `useEffect`. Compute filtered/sorted/derived values
     directly in the hook body (wrapped in `useMemo` only when measurably heavy).

4. Colocated state:
   - All state that must update TOGETHER lives in the SAME custom hook.
     Do not split tightly-coupled state across hooks — it desyncs.

5. Data layer (DIP):
   - The hook depends on `{{FEATURE}}Api`, never on `axios`/`fetch` directly.
   - Validate every API response with Zod (`Schema.parse`). NEVER `as {{ENTITY}}`.
   - React Query: centralized key factory ({{FEATURE}}Keys.all / .lists() / .detail(id)),
     `useQuery` for reads, `useMutation` for writes, explicit `staleTime` + `gcTime`,
     `invalidateQueries` after every successful mutation.
   - For UX-critical toggles, use optimistic updates (onMutate + rollback in onError).

6. ISP — pass narrow props. `<UserCard name={u.name} email={u.email} />`,
   not `<UserCard user={u} />`, unless the child genuinely needs the whole entity.
   Zustand reads MUST use a slice selector: `useStore(s => s.isOpen)`, never `useStore()`.

7. ALL FOUR data states are required: loading (Skeleton mirroring the layout,
   not a spinner), error (<ErrorState> + retry), empty (<EmptyState>), success.

8. Performance: `Promise.all` for independent fetches (no waterfalls), debounce
   the search input (300ms), throttle scroll, `{ passive: true }` on scroll listeners,
   `next/dynamic` for heavy widgets (charts, editors).

DELIVERABLES
  src/features/{{FEATURE}}/
  ├── api/{{FEATURE}}-api.ts
  ├── api/{{FEATURE}}-schema.ts
  ├── api/{{FEATURE}}-keys.ts
  ├── hooks/use-{{FEATURE}}-list.ts
  ├── hooks/use-{{FEATURE}}-filters.ts   (URL state)
  ├── components/{{ENTITY}}List.tsx
  ├── components/{{ENTITY}}Card.tsx
  ├── components/{{ENTITY}}Filters.tsx
  ├── components/{{ENTITY}}ListSkeleton.tsx
  └── index.ts                            (public API — internals stay private)

Start by printing the file tree and one sentence per file stating its single
reason to change. Wait for my "go" before writing the implementation.

WHEN THE IMPLEMENTATION IS DONE (not before the gate above):
run typecheck, lint, and the tests, and paste the real output. If something fails, say so plainly
and show it. If you could not run them, say that explicitly. NEVER report "done" on code you have
not executed.
```
