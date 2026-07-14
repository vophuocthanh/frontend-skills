---
name: nextjs-server-mastery
description: Production-ready AI coding ruleset for Next.js Server-Side Rendering (SSR) — SOLID principles, React Server Components, Server Actions, boundary architecture, caching, environment management, and server performance patterns.
---

# Next.js Server-Side Rendering (SSR) Ruleset

> This skill enforces Next.js App Router server-side patterns, RSC architecture, boundary security, caching strategies, and server performance. Use alongside `nextjs-client-ruleset` for full-stack Next.js projects.

## 🟢 DO / 🔴 DON'T Quick Reference

* **DO** use React Server Components (RSC) by default.
* **DON'T** use `"use client"` at the layout or page level unless absolutely necessary.
* **DO** authenticate Server Actions inside the action itself.
* **DON'T** rely solely on middleware for auth.
* **DO** use `import 'server-only'` to guard server modules.
* **DON'T** import DB clients or secrets from client components.
* **DO** pass only serializable primitives from Server to Client Components.
* **DON'T** pass functions, Date objects, or DB models across the RSC boundary.
* **DO** use `React.cache()` to deduplicate per-request queries.
* **DON'T** read static files inside request handlers (hoist to module level).
* **DO** prefix client-safe env vars with `NEXT_PUBLIC_`.
* **DON'T** use `process.env.SECRET` inside client components.
* **DO** keep Server Actions thin: auth → validate → delegate to a use case (SRP).
* **DON'T** put Prisma/SQL queries directly inside pages or actions (DIP — go through a repository).
* **DO** return the same result shape from every Server Action (LSP).

---

## 🧱 SOLID Principles (Server Edition)

> On the server, SOLID is what keeps route handlers, Server Actions, and RSC pages from turning into god-functions that do auth, validation, SQL, email, and rendering at once. The dependency arrow must always point **inward**: `page/action → use case → repository interface`, never outward to Prisma or Stripe.

### S — Single Responsibility Principle
* **Rule:** A Server Action does exactly four things in order — **authenticate, validate, delegate, revalidate**. Business rules, SQL, and side effects (email, billing) live in separate modules.
* **Why:** An action that inlines SQL + email + Stripe cannot be unit-tested, reused from a route handler or a cron job, and must be rewritten whenever *any* of those change.
* **Good:**
  ```tsx
  // actions/user-actions.ts — thin orchestration only
  'use server';
  import { createUserUseCase } from '@/lib/server/use-cases/create-user';

  export async function createUser(formData: FormData) {
    const session = await getSession();                       // 1. authenticate
    if (!session?.isAdmin) throw new Error('Unauthorized');

    const parsed = CreateUserSchema.safeParse(Object.fromEntries(formData)); // 2. validate
    if (!parsed.success) return { ok: false, fieldErrors: parsed.error.flatten().fieldErrors };

    const user = await createUserUseCase(parsed.data);        // 3. delegate
    revalidatePath('/users');                                 // 4. revalidate
    return { ok: true, data: { id: user.id } };
  }

  // lib/server/use-cases/create-user.ts — business rules, no HTTP/FormData knowledge
  import 'server-only';
  export async function createUserUseCase(input: CreateUserInput) {
    if (await userRepo.existsByEmail(input.email)) throw new ConflictError('Email taken');
    const user = await userRepo.create(input);
    await mailer.sendWelcome(user.email);
    return user;
  }
  ```
* **Bad:** One 150-line `createUser` action containing `db.$transaction`, raw SQL, `resend.emails.send()`, Stripe customer creation, and cache revalidation.
* **Smell:** The action imports `db`, `stripe`, and `resend` all at once.

### O — Open/Closed Principle
* **Rule:** Extend server behavior by **wrapping/composing** (higher-order actions, middleware, strategy maps), not by adding another `if (provider === ...)` branch inside an existing handler.
* **Why:** Every new payment provider or auth method that forces you to reopen a shared handler risks breaking the flows already in production.
* **Good:**
  ```tsx
  // Cross-cutting concerns composed, not copy-pasted into every action
  export const deleteUser = withAuth({ role: 'admin' })(
    withRateLimit({ limit: 5, window: '1 m' })(
      withValidation(DeleteUserSchema)(async (input, ctx) => {
        await userRepo.delete(input.id);
        revalidatePath('/users');
        return { ok: true };
      }),
    ),
  );

  // Strategy map — add a provider by adding an entry, not by editing the handler
  const paymentProviders: Record<Provider, PaymentGateway> = {
    stripe: stripeGateway,
    paypal: paypalGateway, // ✅ new provider = new entry, zero edits above
  };
  export async function charge(provider: Provider, amount: Money) {
    return paymentProviders[provider].charge(amount);
  }
  ```
* **Bad:**
  ```tsx
  export async function charge(provider: string, amount: number) {
    if (provider === 'stripe') { /* ... */ }
    else if (provider === 'paypal') { /* ... */ } // 🔴 reopened for every provider
  }
  ```

### L — Liskov Substitution Principle
* **Rule:** Every implementation of a server contract must be **interchangeable**. All Server Actions return the same discriminated result shape; every repository implementation (Postgres, in-memory fake, mock) honors the same contract — including how it fails.
* **Why:** If `updateUser` returns `{ error }` while `createUser` throws, and the in-memory test repo returns `null` where Prisma throws, then `useActionState` and your tests are lying to you.
* **Good:**
  ```tsx
  // One result contract for ALL Server Actions
  export type ActionResult<T> =
    | { ok: true; data: T }
    | { ok: false; fieldErrors?: Record<string, string[]>; message?: string };

  // Every repo impl behaves identically — same return type, same error type
  export interface UserRepository {
    findById(id: string): Promise<User | null>;   // ← null when missing, NEVER throws
    create(input: CreateUserInput): Promise<User>; // ← throws ConflictError on duplicate
  }
  export const prismaUserRepo: UserRepository = { /* ... */ };
  export const fakeUserRepo: UserRepository = { /* same contract, in-memory */ };
  ```
* **Bad:** `prismaUserRepo.findById()` throws on not-found while `fakeUserRepo.findById()` returns `null` — tests pass, production 500s.

### I — Interface Segregation Principle
* **Rule:** Consumers depend only on what they use. **Select columns, not tables**; pass **fields, not DB models** across the RSC boundary; split fat service interfaces by use case.
* **Why:** `<UserProfile user={user} />` serializes 50 DB fields — including `passwordHash` and `stripeCustomerId` — into the HTML payload sent to the browser. This is both a bundle-size problem and a **data-leak** problem.
* **Good:**
  ```tsx
  // Select only what the view needs
  const user = await db.user.findUnique({
    where: { id },
    select: { id: true, name: true, avatarUrl: true }, // ✅ no passwordHash in the payload
  });
  return <UserProfile name={user.name} avatarUrl={user.avatarUrl} />;

  // Split fat services by consumer
  interface UserReader { findById(id: string): Promise<User | null> }
  interface UserWriter { create(input: CreateUserInput): Promise<User> }
  // A read-only RSC page depends on UserReader alone.
  ```
* **Bad:** `const user = await db.user.findUnique({ where: { id } });` then `<UserProfile user={user} />` — the whole row, secrets included, crosses the boundary.

### D — Dependency Inversion Principle
* **Rule:** Pages, actions, and use cases depend on **interfaces you own** (`UserRepository`, `Mailer`, `PaymentGateway`), not on Prisma, Resend, or Stripe directly. The concrete adapter is chosen at the composition root.
* **Why:** Business logic coupled to Prisma cannot be tested without a database, and swapping the ORM/vendor means rewriting every call site.
* **Good:**
  ```tsx
  // lib/server/ports/mailer.ts — abstraction owned by the high-level module
  import 'server-only';
  export interface Mailer {
    sendWelcome(email: string): Promise<void>;
  }

  // lib/server/adapters/resend-mailer.ts — low-level detail depends on the abstraction
  export const resendMailer: Mailer = {
    async sendWelcome(email) { await resend.emails.send({ /* ... */ }); },
  };

  // lib/server/container.ts — composition root: the ONLY place that knows the concretes
  export const mailer: Mailer = resendMailer;
  export const userRepo: UserRepository = prismaUserRepo;

  // Use case depends on the interface only → unit-testable with fakes, no DB, no network
  export async function createUserUseCase(input: CreateUserInput, deps = { userRepo, mailer }) {
    const user = await deps.userRepo.create(input);
    await deps.mailer.sendWelcome(user.email);
    return user;
  }
  ```
* **Bad:**
  ```tsx
  // 🔴 Page welded to Prisma + Resend: untestable, unswappable
  export default async function Page() {
    const users = await db.user.findMany();
    await resend.emails.send({ /* ... */ });
  }
  ```

### SOLID → Pattern Map (Server)
| Principle | Enforce it with |
|---|---|
| **S**RP | Thin Server Actions (auth → validate → delegate → revalidate), separate use-case layer |
| **O**CP | Higher-order action wrappers (`withAuth`, `withRateLimit`), strategy maps over `if`-chains |
| **L**SP | One `ActionResult<T>` contract; repository impls with identical success **and** failure behavior |
| **I**SP | Prisma `select`, primitives across the RSC boundary, reader/writer interface split |
| **D**IP | Ports + adapters, `import 'server-only'` on ports, single composition root |

---

## 🏗️ Server-First Architecture

### Default to React Server Components
* **Rule:** Default to RSC. Push `"use client"` directives as far down the component tree as possible (to the leaves).
* **Why:** Reduces JavaScript bundle size, keeps sensitive logic on the server, and executes data fetching closer to the database.
* **Good:**
  ```tsx
  // app/page.tsx — Server Component (default)
  import { db } from '@/lib/db';
  import { InteractiveButton } from './InteractiveButton';

  export default async function Page() {
    const data = await db.query();
    return (
      <div>
        <h1>{data.title}</h1>
        <InteractiveButton id={data.id} /> {/* Only this is "use client" */}
      </div>
    );
  }
  ```
* **Bad:** Adding `'use client'` at the top of a page layout, forcing the entire page into the client bundle.

### When to Use `"use client"`
* **Rule:** Only add `"use client"` when the component **directly** uses browser APIs, React hooks (`useState`, `useEffect`, event handlers), or third-party client-only libraries.
* **Decision tree:**
  1. Does it use `useState`, `useEffect`, `useRef`, `useContext`, or event handlers? → `"use client"`
  2. Does it only display data passed via props? → Keep as Server Component
  3. Does it use a third-party library that requires the DOM? → `"use client"`
* **Good:**
  ```tsx
  // components/LikeButton.tsx — Client Component (minimal island)
  'use client';
  import { useState } from 'react';
  export function LikeButton({ initialCount }: { initialCount: number }) {
    const [count, setCount] = useState(initialCount);
    return <button onClick={() => setCount(c => c + 1)}>❤️ {count}</button>;
  }

  // app/posts/[id]/page.tsx — Server Component passes data down
  import { LikeButton } from '@/components/LikeButton';
  export default async function PostPage({ params }: { params: { id: string } }) {
    const post = await getPost(params.id);
    return (
      <article>
        <h1>{post.title}</h1>
        <p>{post.content}</p>
        <LikeButton initialCount={post.likes} />
      </article>
    );
  }
  ```
* **Bad:** Adding `"use client"` to the entire page because one button needs `onClick`.

### Server → Client Data Flow
* **Rule:** Pass **serializable primitives** (strings, numbers, booleans) or **pre-fetched data** from Server Components to Client Components via props. Never pass functions, classes, or non-serializable objects.
* **Why:** The RSC-to-Client boundary serializes props into HTML. Non-serializable values throw runtime errors.
* **Good:** `<UserProfile name={user.name} avatarUrl={user.avatar} />`
* **Bad:** `<UserProfile user={user} />` — passes 50 unused DB fields across the boundary.

---

## 🧱 Boundary Architecture

### Separate Server/Client Boundaries Explicitly
* **Rule:** Create a clear `boundary` layer between server and client code. Never import server-only modules (DB, env secrets) from client components.
* **Why:** Leaking server code into the client bundle exposes secrets and bloats the bundle.
* **Good:**
  ```tsx
  // lib/server-only.ts
  import 'server-only';
  import { db } from './db';
  export async function getUsers() { return db.user.findMany(); }
  ```
* **Bad:** Importing `db` directly inside a `'use client'` component.

### Use `server-only` and `client-only` Packages
* **Rule:** Mark modules with `import 'server-only'` or `import 'client-only'` to get **build-time** errors on wrong-side imports.
* **Why:** Fail fast at build time instead of leaking secrets at runtime.
* **Good:**
  ```tsx
  // lib/auth.ts — Server-only module
  import 'server-only';
  import { cookies } from 'next/headers';

  export async function getSession() {
    const token = cookies().get('session')?.value;
    if (!token) return null;
    return verifyToken(token);
  }
  ```
* **Bad:** Not marking server modules, allowing them to accidentally be imported client-side.

### Boundary File Organization
* **Rule:** Organize server and client code into clearly separated directories or use naming conventions.
* **Good:**
  ```text
  src/
  ├── lib/
  │   ├── server/         # All server-only utilities
  │   │   ├── db.ts
  │   │   ├── auth.ts
  │   │   └── email.ts
  │   └── client/         # All client-only utilities
  │       ├── analytics.ts
  │       └── storage.ts
  ├── actions/             # Server Actions
  │   ├── user-actions.ts
  │   └── billing-actions.ts
  ```

---

## ⚡ Server Actions

### Authenticate Server Actions
* **Rule:** Every Server Action (`"use server"`) MUST independently verify authentication and authorization.
* **Why:** Server Actions are publicly accessible API endpoints. Middleware alone is not sufficient protection.
* **Good:**
  ```tsx
  'use server';
  import { getSession } from '@/lib/server/auth';
  import { revalidatePath } from 'next/cache';

  export async function deleteUser(id: string) {
    const session = await getSession();
    if (!session?.isAdmin) throw new Error('Unauthorized');

    await db.user.delete({ where: { id } });
    revalidatePath('/users');
  }
  ```
* **Bad:** Executing database mutations without checking the session inside the action.

### Validate Server Action Inputs
* **Rule:** Always validate inputs to Server Actions using Zod schemas. Never trust client data.
* **Why:** Server Actions receive data from the client — it can be tampered with.
* **Good:**
  ```tsx
  'use server';
  import { z } from 'zod';

  const CreateUserSchema = z.object({
    name: z.string().min(1).max(100),
    email: z.string().email(),
    role: z.enum(['admin', 'user']),
  });

  export async function createUser(formData: FormData) {
    const session = await getSession();
    if (!session) throw new Error('Unauthorized');

    const parsed = CreateUserSchema.safeParse({
      name: formData.get('name'),
      email: formData.get('email'),
      role: formData.get('role'),
    });

    if (!parsed.success) {
      return { error: parsed.error.flatten().fieldErrors };
    }

    await db.user.create({ data: parsed.data });
    revalidatePath('/users');
    return { success: true };
  }
  ```
* **Bad:** Using `formData.get('role') as string` without validation — allows arbitrary role injection.

### Server Action Error Handling
* **Rule:** Return structured error objects from Server Actions instead of throwing errors (unless unauthorized). Use `useActionState` or `useFormState` to consume them.
* **Why:** Thrown errors trigger Error Boundaries. Validation errors should be handled gracefully in the form UI.

---

## 🚀 Server-Side Data Fetching & Performance

### Eliminate Waterfalls
* **Rule:** Never `await` sequential, independent operations. Use `Promise.all()`.
* **Good:**
  ```tsx
  // app/dashboard/page.tsx
  export default async function DashboardPage() {
    const [user, stats, notifications] = await Promise.all([
      getUser(),
      getStats(),
      getNotifications(),
    ]);
    return <Dashboard user={user} stats={stats} notifications={notifications} />;
  }
  ```
* **Bad:**
  ```tsx
  const user = await getUser();          // 200ms
  const stats = await getStats();        // 300ms
  const notifications = await getNotifications(); // 150ms
  // Total: 650ms (sequential) vs 300ms (parallel)
  ```

### Per-Request Deduplication
* **Rule:** Use `React.cache()` to deduplicate heavy DB queries within a single request. (Note: Next.js automatically deduplicates `fetch()`).
* **Why:** Calling `getUser()` in 5 different components only hits the database once.
* **Good:**
  ```tsx
  import { cache } from 'react';
  import { db } from '@/lib/server/db';

  export const getUser = cache(async (userId: string) => {
    return db.user.findUnique({ where: { id: userId } });
  });

  // Called in layout.tsx, page.tsx, and sidebar.tsx — only 1 DB query
  ```
* **Bad:** Making separate DB queries for the same data in multiple server components.

### Hoist Static I/O to Module Level
* **Rule:** If reading a static file (font, logo, config) in a route handler, hoist the read to module level.
* **Why:** Module-level code runs once per instance, not on every request.
* **Good:**
  ```tsx
  // ✅ Read once at module level
  const fontData = fs.readFileSync(path.join(process.cwd(), 'public/fonts/Inter.woff2'));

  export async function GET() {
    return new Response(fontData, { headers: { 'Content-Type': 'font/woff2' } });
  }
  ```
* **Bad:** Reading the font file inside the `GET()` handler on every request.

### Strategic Suspense Boundaries
* **Rule:** Wrap heavy, slow-loading server components in `<Suspense>` to stream them progressively.
* **Why:** Prevents a single slow query from blocking the entire page render.
* **Good:**
  ```tsx
  import { Suspense } from 'react';

  export default function DashboardPage() {
    return (
      <div>
        <h1>Dashboard</h1>
        <Suspense fallback={<StatsSkeleton />}>
          <SlowStatsPanel />  {/* Streams in when ready */}
        </Suspense>
        <Suspense fallback={<ChartSkeleton />}>
          <SlowChart />       {/* Streams independently */}
        </Suspense>
      </div>
    );
  }
  ```
* **Bad:** Not using Suspense, causing the entire page to wait for the slowest component.

### Deduplicate Global Event Listeners
* **Rule:** Use `useSWRSubscription` or a module-level Map to share global event listeners across multiple component instances.
* **Why:** 10 instances each attaching `keydown` listener = memory leaks.

---

## 🗄️ Caching Strategy

### Cache Hierarchy
* **Rule:** Follow a 3-layer caching strategy:
  1. **React Query/SWR (client)** — reduces network calls
  2. **`React.cache()` (per-request server)** — deduplicates within a single render
  3. **LRU/Redis (cross-request server)** — reduces database load
* **Why:** Each layer serves a different lifecycle.

### Next.js Cache Configuration
* **Rule:** Use `unstable_cache` or `fetch` with `next.revalidate` for cross-request server caching. Set explicit revalidation times.
* **Good:**
  ```tsx
  import { unstable_cache } from 'next/cache';

  const getCachedProducts = unstable_cache(
    async () => db.product.findMany(),
    ['products'],
    { revalidate: 3600 } // 1 hour
  );
  ```

### Revalidation Strategy
* **Rule:** Use `revalidatePath()` or `revalidateTag()` in Server Actions after mutations to clear stale cache.
* **Why:** Prevents users from seeing stale data after create/update/delete operations.

---

## 🔐 Environment Management

### Never Expose Server Secrets to the Client
* **Rule:** Prefix client-safe env vars with `NEXT_PUBLIC_`. All other env vars are server-only.
* **Why:** Non-prefixed env vars are stripped from the client bundle by Next.js.
* **Good:**
  ```env
  DATABASE_URL=postgresql://...         # Server only
  NEXT_PUBLIC_API_URL=https://api.com   # Safe for client
  ```
* **Bad:** Using `process.env.DATABASE_URL` inside a `'use client'` component.

### Validate Env at Startup
* **Rule:** Use `t3-env` or a Zod schema to validate all required environment variables at build/start time.
* **Why:** Catches missing or malformed env vars immediately, not in production at 3 AM.
* **Good:**
  ```tsx
  // env.ts
  import { createEnv } from '@t3-oss/env-nextjs';
  import { z } from 'zod';

  export const env = createEnv({
    server: {
      DATABASE_URL: z.string().url(),
      AUTH_SECRET: z.string().min(32),
    },
    client: {
      NEXT_PUBLIC_API_URL: z.string().url(),
    },
    runtimeEnv: {
      DATABASE_URL: process.env.DATABASE_URL,
      AUTH_SECRET: process.env.AUTH_SECRET,
      NEXT_PUBLIC_API_URL: process.env.NEXT_PUBLIC_API_URL,
    },
  });
  ```

---

## 📊 Logging & Monitoring

### Structured Logging
* **Rule:** Use structured JSON logging (e.g., `pino`, `winston`) instead of `console.log`. Include `requestId`, `userId`, and `action` fields.
* **Why:** `console.log` is unsearchable in production. Structured logs enable filtering, alerting, and dashboarding.
* **Good:**
  ```tsx
  logger.info({ action: 'user.created', userId: user.id, duration: ms });
  ```
* **Bad:** `console.log('User created:', user.id);`

### Error Tracking
* **Rule:** Integrate an error tracking service (Sentry, Datadog) and capture unhandled exceptions with context.
* **Why:** `console.error` is invisible in production.

### Request Tracing
* **Rule:** Generate a unique `requestId` per incoming request and propagate it through all server-side operations.
* **Why:** Enables end-to-end debugging of request flows across services.

---

## 🔒 Server Security

### CSRF Protection
* **Rule:** Next.js Server Actions include CSRF protection by default. Never bypass it by exposing Server Action logic via raw API routes without CSRF tokens.

### Rate Limiting
* **Rule:** Apply rate limiting on Server Actions that perform writes or authentication.
* **Why:** Prevents brute-force attacks and abuse.
* **Good:**
  ```tsx
  // middleware.ts or inside Server Action
  import { Ratelimit } from '@upstash/ratelimit';
  import { Redis } from '@upstash/redis';

  const ratelimit = new Ratelimit({
    redis: Redis.fromEnv(),
    limiter: Ratelimit.slidingWindow(10, '10 s'), // 10 requests per 10 seconds
  });

  export async function loginAction(formData: FormData) {
    const ip = headers().get('x-forwarded-for') ?? 'unknown';
    const { success } = await ratelimit.limit(ip);
    if (!success) throw new Error('Too many requests');
    // ... proceed with login
  }
  ```

### Content Security Policy
* **Rule:** Configure CSP headers in `next.config.js` to prevent inline script injection.
* **Good:**
  ```tsx
  // next.config.js
  const securityHeaders = [
    { key: 'Content-Security-Policy', value: "default-src 'self'; script-src 'self'" },
    { key: 'X-Frame-Options', value: 'DENY' },
    { key: 'X-Content-Type-Options', value: 'nosniff' },
  ];
  ```

### Token Storage
* **Rule:** Never store auth tokens in `localStorage` or `sessionStorage`. Use `httpOnly` cookies managed by the server.
* **Why:** `localStorage` is accessible to any JavaScript on the page — XSS attacks can steal tokens.

---

## 🛑 Server Anti-Patterns

1. **Missing Auth in Server Actions:** Relying only on middleware for auth — Server Actions are public endpoints.
2. **Unvalidated Server Action Inputs:** Using `formData.get('role') as string` without Zod validation.
3. **Sequential Fetches in Pages:** Awaiting independent queries one after another instead of `Promise.all`.
4. **Leaking Server Code to Client:** Importing `db` or secret env vars in `'use client'` components.
5. **Missing Suspense Boundaries:** Not wrapping slow server components, blocking entire page renders.
6. **Static File Reads in Handlers:** Reading fonts/configs inside route handlers instead of at module level.
7. **No Cache Invalidation:** Forgetting `revalidatePath()` after mutations, showing stale data.
8. **`console.log` in Server Code:** Using unstructured logging that's unsearchable in production.
9. **Exposing `process.env.SECRET` Client-Side:** Forgetting `NEXT_PUBLIC_` prefix rules.
10. **No Env Validation:** Deploying without checking required env vars, crashing at runtime.
11. **Fat Server Actions:** One action doing auth + SQL + email + Stripe + revalidation. *(violates SRP — delegate to a use case)*
12. **Provider `if`-Chains:** `if (provider === 'stripe') ... else if (provider === 'paypal')`. *(violates OCP — use a strategy map)*
13. **Inconsistent Action Contracts:** One action throws, another returns `{ error }`, a third returns `null`. *(violates LSP — use one `ActionResult<T>`)*
14. **Leaking Whole DB Rows:** `<Profile user={user} />` shipping `passwordHash` across the RSC boundary. *(violates ISP — use `select` + primitives)*
15. **ORM/SDK Welded Into Pages:** Importing `db`/`stripe`/`resend` directly in pages and actions. *(violates DIP — go through ports + adapters)*

---

## ✅ SSR PR Review Checklist

- [ ] **RSC Usage:** Are Server Components used by default? Is `"use client"` pushed to the leaves?
- [ ] **Boundaries:** Are `server-only` / `client-only` imports used correctly? No secret leakage?
- [ ] **Server Actions:** Do all Server Actions validate auth AND inputs (Zod)?
- [ ] **Data Fetching:** Any sequential `await` that could be `Promise.all()`? Using `React.cache()`?
- [ ] **Serialization:** Are only primitives/simple objects passed from Server to Client Components?
- [ ] **Caching:** Are `revalidatePath`/`revalidateTag` called after mutations? `staleTime` configured?
- [ ] **Suspense:** Are slow components wrapped in `<Suspense>` with skeleton fallbacks?
- [ ] **Env:** New env vars documented? Validated at startup? No `NEXT_PUBLIC_` leak of secrets?
- [ ] **Security:** Server Actions rate-limited where needed? CSP headers configured?
- [ ] **Logging:** Structured logging used? Error tracking integrated?
- [ ] **SRP:** Are Server Actions thin (auth → validate → delegate → revalidate), with business rules in a use case?
- [ ] **OCP:** Was a new provider/variant added via a strategy map or wrapper, not a new `if` branch in a shared handler?
- [ ] **LSP:** Do all Server Actions return the same `ActionResult<T>` shape? Do fake and real repos fail the same way?
- [ ] **ISP:** Does every query `select` only needed columns? Do only primitives cross the RSC boundary?
- [ ] **DIP:** Do pages/actions import `db`/`stripe`/`resend` directly, or depend on a port resolved at the composition root?

### Common Server-Side Mistakes
- Forgetting to authenticate inside Server Actions (relying only on middleware).
- Passing the entire DB object from Server to Client Component instead of specific fields.
- Using `console.log` instead of structured logger in server code.
- Using `process.env.SECRET` inside a client component (will be `undefined`).
- Forgetting `revalidatePath()` after a Server Action mutation.
- Not wrapping slow components in `<Suspense>`, blocking the entire page.
- Reading static files inside route handlers instead of at module level.
- Committing `.env.local` or hardcoded API keys to the repository.
- Not validating env vars at startup — crashing in production at 3 AM.
