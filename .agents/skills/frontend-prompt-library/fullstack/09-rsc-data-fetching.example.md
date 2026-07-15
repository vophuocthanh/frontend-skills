---
title: Example — Build an RSC Page with Server Data Fetching
type: example
pairs_with: 09-rsc-data-fetching.md
---

# 📘 Example — Build an RSC Page with Server Data Fetching

> A fully filled-in run of [`09-rsc-data-fetching.md`](09-rsc-data-fetching.md) on a real scenario.
> Copy the prompt block below and paste it into your agent as-is.

---

```text
Follow `nextjs-server-mastery` (+ `react-client-mastery` for the client islands).
Build the RSC page at `app/dashboard/page.tsx`.

DATA NEEDED: current user profile (fast), seat-usage stats (slow, ~3s), recent invitations (must always be fresh)
INTERACTIVE PARTS: date-range picker, "Export CSV" button
CACHING POLICY: seat-usage stats cached 1h; recent invitations always fresh

REPO CONTEXT: `lib/server/` already has `db.ts` (the Prisma client singleton) and
  `auth/session.ts` (`getSession()`, already wrapped in `React.cache()`), and every server
  module already carries an `import 'server-only'` guard — follow that. The
  `error.tsx` / `loading.tsx` conventions already exist under `app/billing/` — mirror them
  rather than inventing new shells.

NON-GOALS: No redesign of the dashboard layout. No new metrics beyond seat usage
  (revenue/churn tiles are a separate ticket).

RULES

0. MATCH THE REPO BEFORE YOU MATCH THIS PROMPT.
   The paths in DELIVERABLES below are the default for a greenfield repo. If REPO CONTEXT shows
   an existing convention — folder layout, an http client, an error type, UI primitives, a test
   setup — REUSE IT instead of creating a parallel structure beside it. Creating a second way to
   do something that already exists is a failure, even if the new way follows every rule below.
   List every place you deviated from the tree below, and why.

1. SERVER BY DEFAULT.
   - The page and layout are Server Components. NO `'use client'` at page/layout level.
   - `'use client'` goes ONLY on the leaves that actually need state, effects,
     event handlers, or browser APIs: the date-range picker and the "Export CSV" button.
   - Decision test per component: does it use hooks/handlers/DOM? If no → stays server.

2. ISP — THE RSC BOUNDARY IS A SECURITY BOUNDARY.
   - Query with an explicit `select` — never `SELECT *`.
   - Pass PRIMITIVES / narrow objects to client components:
       ✅ <Chart points={stats.points} currency="USD" />
       ❌ <Chart data={dbRow} />
   - Everything you pass is SERIALIZED INTO THE HTML sent to the browser.
     A whole DB row means passwordHash and stripeCustomerId are now in view-source.
   - Never pass functions, class instances, or `Date` objects across the boundary
     (not serializable — send ISO strings).

3. DIP — the page must not import Prisma/the SDK directly.
   - Page → use case / repository interface → adapter (Prisma, upstream API).
   - `import 'server-only'` at the top of every server module so a wrong-side import
     fails at BUILD time, not at 3am in production.

4. SRP — the page composes; it does not compute.
   - `page.tsx` = fetch (delegate) + compose layout. No business rules inline.
   - Business rules live in `lib/server/use-cases/*`.

5. NO WATERFALLS.
   - Independent fetches → `Promise.all()`.
   - Deduplicate per-request queries with `React.cache()` so calling `getUser()` in
     layout + page + sidebar hits the DB ONCE.
   - Hoist static file reads (fonts, configs) to module level — never inside the handler.

6. STREAMING — wrap slow, independent sections in `<Suspense>` with skeleton fallbacks
   so one slow query cannot block the whole page. Per the caching policy, decide per section:
   - fast + critical → await in the page (blocks, but it's fast) → the current user profile
   - slow + non-critical → own `<Suspense>` boundary, streams in → the ~3s seat-usage stats

7. CACHING per the policy (seat-usage stats cached 1h; recent invitations always fresh):
   - `React.cache()` → per-request dedup
   - `unstable_cache` / `fetch(..., { next: { revalidate } })` → cross-request
   - Set revalidate times EXPLICITLY and justify each one.
   - Any mutation touching this data MUST `revalidatePath`/`revalidateTag`.

8. ERRORS & STATES:
   - `error.tsx` (client) with a `reset()` retry + `captureException`.
   - `not-found.tsx` for 404s; call `notFound()` from the page when the entity is missing.
   - `loading.tsx` or Suspense fallbacks — skeletons that mirror the real layout.
   - Empty state handled explicitly.

9. SECURITY:
   - Authenticate in the page/layout via `getSession()` — do not rely on middleware alone.
   - Secrets are read ONLY in server modules. `NEXT_PUBLIC_` prefix = public, treat
     anything with it as if it were printed on a billboard.

DELIVERABLES
  app/dashboard/page.tsx                 (server, composes)
  app/dashboard/loading.tsx
  app/dashboard/error.tsx                ('use client')
  app/dashboard/not-found.tsx
  components/*.tsx                       (server presentational)
  components/*.client.tsx                ('use client' islands — the date-range picker and the "Export CSV" button)
  lib/server/use-cases/*.ts              ('server-only')
  lib/server/repositories/*.ts           (interface + adapter)

Before coding, print a table: | Component | Server or Client? | Why | and
list EXACTLY which fields cross the RSC boundary for each client island.
Then wait for my "go".

WHEN THE IMPLEMENTATION IS DONE (not before the gate above):
run typecheck, lint, and the tests, and paste the real output. If something fails, say so plainly
and show it. If you could not run them, say that explicitly. NEVER report "done" on code you have
not executed.
```
