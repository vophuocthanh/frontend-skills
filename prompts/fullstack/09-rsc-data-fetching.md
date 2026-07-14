---
title: Build an RSC Page with Server Data Fetching
category: FE + BE
skills: [nextjs-server-mastery, react-client-mastery]
principles: [SRP, ISP, DIP]
---

# 🖥️ Prompt — Build an RSC Page with Server Data Fetching

> Use for a Next.js App Router page that fetches on the **server** (DB or upstream API) and streams to the client.
> The two failure modes this prompt exists to prevent: **leaking DB rows into the HTML payload**, and **`'use client'` at the top of the page**.

---

```text
Follow `nextjs-server-mastery` (+ `react-client-mastery` for the client islands).
Build the RSC page at `{{ROUTE}}`.

DATA NEEDED: {{DATA}}
INTERACTIVE PARTS: {{INTERACTIVE}}
CACHING POLICY: {{CACHING}}

REPO CONTEXT: {{EXISTING}}
  (Existing conventions to reuse: folder layout, http client, error types, UI primitives,
   test setup. Write "greenfield" if there is none.)

NON-GOALS: {{NON_GOALS}}
  (Do NOT build these. If you believe one is actually required, STOP and ask me first.)

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
     event handlers, or browser APIs: {{INTERACTIVE}}.
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
   so one slow query cannot block the whole page. Per {{CACHING}}, decide per section:
   - fast + critical → await in the page (blocks, but it's fast)
   - slow + non-critical → own `<Suspense>` boundary, streams in

7. CACHING per {{CACHING}}:
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
  app/{{ROUTE}}/page.tsx          (server, composes)
  app/{{ROUTE}}/loading.tsx
  app/{{ROUTE}}/error.tsx         ('use client')
  app/{{ROUTE}}/not-found.tsx
  components/*.tsx                (server presentational)
  components/*.client.tsx         ('use client' islands — {{INTERACTIVE}})
  lib/server/use-cases/*.ts       ('server-only')
  lib/server/repositories/*.ts    (interface + adapter)

Before coding, print a table: | Component | Server or Client? | Why | and
list EXACTLY which fields cross the RSC boundary for each client island.
Then wait for my "go".

WHEN THE IMPLEMENTATION IS DONE (not before the gate above):
run typecheck, lint, and the tests, and paste the real output. If something fails, say so plainly
and show it. If you could not run them, say that explicitly. NEVER report "done" on code you have
not executed.
```
