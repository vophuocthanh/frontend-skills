---
name: frontend-engineering-ruleset
description: The ultimate production-ready AI coding ruleset for modern React, scalable architecture, strict TypeScript, and performance-first patterns.
---

# Frontend Engineering Ruleset

> This skill enforces modern React (v19+), Next.js App Router patterns, strict TypeScript, and scalable architecture. It is designed to guide AI agents and engineering teams in producing highly performant, maintainable code.

## 🟢 DO / 🔴 DON'T Quick Reference

* **DO** use React Server Components (RSC) by default.
* **DON'T** use `"use client"` at the layout or page level unless absolutely necessary.
* **DO** fetch data in parallel using `Promise.all()` or `better-all`.
* **DON'T** create sequential network waterfalls.
* **DO** use composition (Compound Components) for UI variants.
* **DON'T** use boolean props (e.g., `<Button isPrimary isLarge />`) to configure components.
* **DO** calculate derived state during the render cycle.
* **DON'T** use `useEffect` to synchronize state.
* **DO** strictly type everything; use discriminated unions for complex states.
* **DON'T** ever use `any`.
* **DO** authenticate Server Actions inside the action itself.
* **DON'T** use barrel files (`index.ts` re-exports) for large libraries.

---

## 🏗️ Architecture & File Structure

### Server-First by Default
* **Rule:** Default to React Server Components (RSC). Push `"use client"` directives as far down the component tree as possible (to the leaves).
* **Why:** Reduces JavaScript bundle size, keeps sensitive logic on the server, and executes data fetching closer to the database.
* **Good:**
  ```tsx
  // app/page.tsx
  import { db } from '@/lib/db';
  import { InteractiveButton } from './InteractiveButton';

  export default async function Page() {
    const data = await db.query();
    return (
      <div>
        <h1>{data.title}</h1>
        <InteractiveButton id={data.id} />
      </div>
    );
  }
  ```
* **Bad:** Adding `'use client'` at the top of a page layout, forcing the entire page into the client bundle.

### Feature-Sliced Colocation
* **Rule:** Organize files by business domain/feature rather than technical type.
* **Why:** Makes features self-contained, easier to test, and scalable. Prevents the `src/components` folder from becoming a dumping ground.
* **Good:**
  ```text
  src/features/auth/
  ├── api/         # Server actions, queries
  ├── components/  # LoginForm, RegisterForm
  ├── hooks/       # useAuth
  └── utils/       # validation schemas
  ```
* **Bad:** Grouping all components into one `/components` folder and all hooks into `/hooks`.

---

## ⚛️ React Component & Hook Patterns

### Composition over Configuration
* **Rule:** Use compound components and `children` props instead of adding boolean flags for every possible UI state.
* **Why:** Prevents monolithic components filled with complex `if/else` logic that are impossible to maintain.
* **Good:**
  ```tsx
  <Modal.Root>
    <Modal.Header>Title</Modal.Header>
    <Modal.Content>Body</Modal.Content>
  </Modal.Root>
  ```
* **Bad:**
  ```tsx
  <Modal title="Title" content="Body" showFooter={true} closeButtonText="Close" />
  ```

### Extract Static JSX
* **Rule:** Hoist static JSX elements out of the component body if they don't depend on state.
* **Why:** React won't need to recreate these elements on every render, saving CPU cycles.
* **Good:**
  ```tsx
  const staticHeader = <h1>Dashboard</h1>;
  export function Dashboard() { return <div>{staticHeader}</div>; }
  ```
* **Bad:** Defining the static header inline inside a frequently updating component.

### Calculate Derived State During Render
* **Rule:** Never use `useEffect` to synchronize state or calculate derived values. Do it directly during render.
* **Why:** `useEffect` state updates cause an immediate second render, leading to layout thrashing and poor performance.
* **Good:**
  ```tsx
  function UserList({ users, query }) {
    const filteredUsers = users.filter(u => u.name.includes(query));
    return <ul>...</ul>;
  }
  ```
* **Bad:** Syncing `filteredUsers` via `useState` and `useEffect`.

### Primitive Dependencies in Hooks
* **Rule:** Only pass primitives (strings, numbers, booleans) into dependency arrays (`useEffect`, `useCallback`, `useMemo`), or use stable references.
* **Why:** Objects and arrays are recreated on every render (unless memoized), triggering infinite loops or unnecessary effects.
* **Good:** `useEffect(() => { fetchUser(userId); }, [userId]);`
* **Bad:** `useEffect(() => { fetchUser(user.id); }, [user]);`

### Use useRef for Transient Values
* **Rule:** If a value updates frequently (like scroll position) and doesn't directly affect the DOM, store it in a `ref`.
* **Why:** Bypasses the React render cycle entirely, vastly improving performance for high-frequency events.

### Explicit Conditional Rendering
* **Rule:** Use explicit conditionals (ternaries) over `&&` when the condition might be a falsy number (`0`).
* **Why:** React renders `0` to the DOM if you do `count && <span>{count}</span>` when `count` is `0`.
* **Good:** `count > 0 ? <span>{count}</span> : null`
* **Bad:** `count && <span>{count}</span>`

### Logic-in-Hook, UI-in-Component
* **Rule:** Move ALL business logic, state management, side effects, and data transformations into a custom hook. The `.tsx` component file must ONLY contain JSX rendering and event handler wiring.
* **Why:** Separating logic from presentation makes components trivially testable, infinitely reusable, and easy to review. The component becomes a pure "view" layer.
* **Good:**
  ```tsx
  // hooks/use-user-list.ts — ALL logic lives here
  export function useUserList(initialQuery: string) {
    const [query, setQuery] = useState(initialQuery);
    const { data: users, isLoading } = useQuery({ queryKey: userKeys.lists(), queryFn: fetchUsers });
    const filteredUsers = useMemo(() => users?.filter(u => u.name.includes(query)) ?? [], [users, query]);
    const handleSearch = useCallback((value: string) => setQuery(value), []);
    return { filteredUsers, isLoading, query, handleSearch };
  }

  // components/UserList.tsx — ONLY rendering, zero logic
  export function UserList() {
    const { filteredUsers, isLoading, query, handleSearch } = useUserList('');
    if (isLoading) return <Skeleton />;
    return (
      <div>
        <SearchInput value={query} onChange={handleSearch} />
        <ul>{filteredUsers.map(u => <UserCard key={u.id} user={u} />)}</ul>
      </div>
    );
  }
  ```
* **Bad:**
  ```tsx
  // ❌ Component contains logic, state, effects, AND rendering
  export function UserList() {
    const [query, setQuery] = useState('');
    const [users, setUsers] = useState([]);
    useEffect(() => { fetchUsers().then(setUsers); }, []);
    const filtered = users.filter(u => u.name.includes(query));
    return <ul>...</ul>;
  }
  ```

### Colocate Related State in a Single Hook
* **Rule:** All state that must update together MUST live inside the SAME custom hook. Never split tightly-coupled state across multiple hooks or multiple `useState` calls in different files.
* **Why:** When related state is split across separate hook instances, React batches updates per hook — leading to **one render with stale values** from the other hook. This causes UI desync, race conditions, and subtle bugs that are extremely hard to trace.
* **Good:**
  ```tsx
  // ✅ All related state in ONE hook — always consistent
  function useCheckout() {
    const [items, setItems] = useState<CartItem[]>([]);
    const [discount, setDiscount] = useState(0);
    const total = useMemo(() => {
      const subtotal = items.reduce((sum, i) => sum + i.price * i.qty, 0);
      return subtotal * (1 - discount / 100);
    }, [items, discount]);

    const addItem = useCallback((item: CartItem) => {
      setItems(prev => [...prev, item]);
    }, []);

    const applyDiscount = useCallback((pct: number) => {
      setDiscount(pct);
    }, []);

    return { items, total, discount, addItem, applyDiscount };
  }
  ```
* **Bad:**
  ```tsx
  // ❌ Related state scattered across 2 hooks — desync guaranteed
  function useCartItems() {
    const [items, setItems] = useState<CartItem[]>([]);
    return { items, setItems };
  }

  function useCartDiscount() {
    const [discount, setDiscount] = useState(0);
    return { discount, setDiscount };
  }

  // In component: these are SEPARATE instances with SEPARATE render cycles
  function Checkout() {
    const { items } = useCartItems();       // instance A
    const { discount } = useCartDiscount(); // instance B — may be stale when A updates!
    const total = items.reduce(...) * (1 - discount / 100); // 🔴 Potentially wrong
  }
  ```

---

## 🚀 Performance: Waterfalls & Bundle Size

### Eliminate Waterfalls
* **Rule:** Never `await` sequential, independent operations. Always use `Promise.all()`.
* **Why:** Sequential awaits multiply network latency and severely degrade TTFB (Time to First Byte).
* **Good:**
  ```tsx
  const [user, posts] = await Promise.all([fetchUser(), fetchPosts()]);
  ```
* **Bad:**
  ```tsx
  const user = await fetchUser(); // 🔴 Blocks posts fetch
  const posts = await fetchPosts();
  ```

### No Barrel Files
* **Rule:** Never use `index.ts` to re-export modules from large libraries (e.g., `lucide-react`, `@mui/material`) unless your bundler is explicitly configured to optimize them.
* **Why:** Barrel files force the bundler to parse thousands of unused modules, destroying tree-shaking and bloating the bundle.
* **Good:** `import Button from '@mui/material/Button';`
* **Bad:** `import { Button } from '@mui/material';`

### Dynamic Imports for Heavy Components
* **Rule:** Use `next/dynamic` to lazy-load heavy components (e.g., Monaco Editor, charts, maps) that are not needed for the initial render.
* **Why:** Reduces the initial JS payload, drastically improving Time to Interactive (TTI).

### Strategic Suspense Boundaries
* **Rule:** Wrap heavy, slow-loading components in `<Suspense>` to stream them in.
* **Why:** Prevents a single slow component from blocking the rendering of the entire layout.

---

## 🌍 Server-Side & Client-Side Data Fetching

### Authenticate Server Actions
* **Rule:** Every Server Action (`"use server"`) MUST independently verify authentication and authorization.
* **Why:** Server Actions are publicly accessible API endpoints. Middleware alone is not sufficient protection.
* **Good:**
  ```tsx
  'use server';
  export async function deleteUser(id: string) {
    const session = await getSession();
    if (!session?.isAdmin) throw new Error('Unauthorized');
    await db.delete(id);
  }
  ```
* **Bad:** Executing database mutations without checking the session inside the action.

### Minimize RSC Serialization
* **Rule:** Only pass the specific fields needed from Server Components to Client Components.
* **Why:** The RSC-to-Client boundary serializes all object properties into strings embedded in the HTML. Huge objects bloat the payload.
* **Good:** `<UserProfile name={user.name} />`
* **Bad:** `<UserProfile user={user} /> // Passes 50 unused DB fields`

### Hoist Static I/O to Module Level
* **Rule:** If reading a static file (e.g., a font, logo, config) in a route handler, hoist the read operation to the module level.
* **Why:** Code at the module level runs once per instance, avoiding expensive file reads on every single request.

### Per-Request Deduplication
* **Rule:** Use `React.cache()` to deduplicate heavy DB queries or computations within a single request. (Note: Next.js automatically deduplicates `fetch()`).
* **Why:** Ensures that calling `getUser()` in 5 different components only hits the database once per request.

### Deduplicate Global Event Listeners
* **Rule:** Use `useSWRSubscription` or a module-level Map to share global event listeners (like `keydown`) across multiple component instances.
* **Why:** If 10 instances of a component each attach a `keydown` listener, it causes memory leaks and performance drops.

---

## 📘 TypeScript Strict Rules & State Management

### Avoid `any` Completely
* **Rule:** Do not use `any`. Use `unknown` for unknown data, and narrow it with type guards or validation libraries (e.g., Zod).
* **Why:** `any` bypasses the compiler, nullifying the safety benefits of TypeScript.

### Discriminated Unions for State
* **Rule:** Use discriminated unions to model state machines, rather than multiple boolean flags.
* **Why:** Prevents impossible states (e.g., being in a `loading` state while also having `error` and `data` populated).
* **Good:**
  ```tsx
  type RequestState =
    | { status: 'idle' }
    | { status: 'loading' }
    | { status: 'success'; data: User }
    | { status: 'error'; error: Error };
  ```
* **Bad:** `{ isLoading: boolean; data?: User; error?: Error }`

### Keep State Local First
* **Rule:** Keep state local to the component or custom hook until a sibling needs it, then lift it to the nearest common ancestor.
* **Why:** Global state (Redux/Zustand) introduces coupling. Use Zustand for high-frequency global state, and Context for dependency injection (themes, APIs).

---

## 📝 Engineering Standards

### Strict Naming Conventions
* **Rule:** Follow explicit casing and prefix rules.
* **Why:** A predictable naming system makes the codebase instantly scannable and reduces cognitive load.
* **Good:**
  * **`.tsx` files (Components):** `PascalCase` → `UserProfile.tsx`, `InvoiceRow.tsx`, `LoginForm.tsx`
  * **`.ts` files (Hooks, Utils, Schemas, API):** `kebab-case` → `use-auth.ts`, `format-currency.ts`, `user-schema.ts`, `billing-api.ts`
  * **Component names (inside file):** `PascalCase` → `export function UserProfile()`
  * **Hooks (inside file):** `camelCase` starting with `use` → `export function useAuth()`
  * **Booleans:** Prefix with `is`, `has`, `should`, or `can` → `isLoading`, `hasPermission`
  * **Event Handlers:** Prefix with `handle` → `handleSubmit`; Props with `on` → `onSubmit`
  * **Folders:** Always `kebab-case` → `user-profile/`, `billing-api/`
* **Bad:**
  * `userProfile.tsx` (camelCase for component file)
  * `FormatCurrency.ts` (PascalCase for utility file)
  * `useAuth.ts` (camelCase for non-component `.ts` file — should be `use-auth.ts`)

### Error Handling & Validation
* **Rule:** Use Error Boundaries (`error.tsx` or `<ErrorBoundary>`) to isolate crashes. Validate all incoming API data and form inputs using `Zod`.
* **Why:** Prevents the entire app from crashing due to localized errors and prevents malicious data injection.

### Accessibility (A11y)
* **Rule:** Use correct semantic tags (`<button>`, `<a>`, `<nav>`) and manage focus states.
* **Why:** Ensures the application is usable by screen readers and keyboard navigators.
* **Good:** Use `:focus-visible` for focus rings. Never use `outline: none` without providing a visual alternative.

### Security
* **Rule:** Avoid `dangerouslySetInnerHTML`.
* **Why:** Opens the door to Cross-Site Scripting (XSS). If necessary, use `dompurify` to sanitize HTML.

### Testing Strategy
* **Rule:** Focus on Integration Tests over implementation details.
* **Why:** Testing user flows (e.g., Playwright/React Testing Library) makes tests resilient to refactoring, unlike testing specific state values.

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
* **Rule:** Mark modules with `import 'server-only'` or `import 'client-only'` to get build-time errors on wrong-side imports.
* **Why:** Fail fast at build time instead of leaking secrets at runtime.

---

## 📡 API Contract & Data Layer

### Type-Safe API Contracts
* **Rule:** Define shared request/response types using Zod schemas. Infer TypeScript types from schemas (`z.infer<typeof schema>`).
* **Why:** Ensures frontend and backend agree on data shape. Catches contract changes at compile time.
* **Good:**
  ```tsx
  // schemas/user.ts
  import { z } from 'zod';
  export const UserSchema = z.object({ id: z.string(), name: z.string(), email: z.string().email() });
  export type User = z.infer<typeof UserSchema>;

  // usage
  const data = UserSchema.parse(apiResponse); // Runtime validation + type inference
  ```
* **Bad:** Using `as User` type assertion without runtime validation.

### Centralized API Client
* **Rule:** Create a single, typed API client (e.g., `lib/api.ts`) with interceptors for auth headers, error transformation, and base URL.
* **Why:** Prevents scattered `fetch()` calls with inconsistent error handling and headers.

---

## 📋 Forms

### Use React Hook Form + Zod
* **Rule:** Use `react-hook-form` with `@hookform/resolvers/zod` for all forms. Define validation schemas with Zod.
* **Why:** Provides uncontrolled form performance, eliminates re-renders per keystroke, and shares validation schemas with the API layer.
* **Good:**
  ```tsx
  const schema = z.object({ email: z.string().email(), password: z.string().min(8) });
  const { register, handleSubmit, formState: { errors } } = useForm({ resolver: zodResolver(schema) });
  ```
* **Bad:** Using `useState` for every input field with manual `onChange` handlers and inline validation.

### Optimistic UI for Mutations
* **Rule:** Show the expected result immediately while the mutation is in-flight. Revert on error.
* **Why:** Makes the app feel instant. Users don't stare at spinners for simple actions like toggling a favorite.

---

## 🎨 Styling Strategy

### Utility-First with Escape Hatches
* **Rule:** Use TailwindCSS as the primary styling approach. For complex, dynamic styles, use CSS Modules or `cva` (class-variance-authority).
* **Why:** Tailwind eliminates naming decisions and dead CSS. `cva` provides type-safe variant management for design system components.
* **Good:**
  ```tsx
  // Using cva for component variants
  import { cva } from 'class-variance-authority';
  const button = cva('px-4 py-2 rounded font-medium', {
    variants: {
      intent: { primary: 'bg-blue-600 text-white', danger: 'bg-red-600 text-white' },
      size: { sm: 'text-sm', lg: 'text-lg px-6 py-3' },
    },
    defaultVariants: { intent: 'primary', size: 'sm' },
  });
  ```
* **Bad:** Mixing inline styles, global CSS, Styled Components, and Tailwind in the same project.

### No Inline Styles for Layout
* **Rule:** Never use inline `style={{}}` for layout, spacing, or color. Reserve inline styles only for truly dynamic values (e.g., `style={{ width: `${percentage}%` }}`).
* **Why:** Inline styles bypass design tokens, are not cacheable, and cannot be responsive.

---

## 🔄 React Query / SWR Conventions

### Colocate Query Keys
* **Rule:** Define query keys as constants alongside their fetcher functions. Use a factory pattern for related keys.
* **Why:** Prevents typo-based cache misses and makes invalidation explicit and traceable.
* **Good:**
  ```tsx
  // features/users/api/queries.ts
  export const userKeys = {
    all: ['users'] as const,
    lists: () => [...userKeys.all, 'list'] as const,
    detail: (id: string) => [...userKeys.all, 'detail', id] as const,
  };

  export function useUsers() {
    return useQuery({ queryKey: userKeys.lists(), queryFn: fetchUsers });
  }
  ```
* **Bad:** Scattering string literal query keys (`['users']`, `['user', id]`) across multiple files.

### Separate Queries from Mutations
* **Rule:** Use `useQuery` for reads, `useMutation` for writes. Never mix fetching and mutating in a single hook.
* **Why:** Keeps caching, deduplication, and retry logic clean. Mutations have different error/loading semantics.

---

## ✏️ Mutation Patterns

### Invalidate After Mutation
* **Rule:** Always call `queryClient.invalidateQueries()` after a successful mutation to refresh stale data.
* **Why:** Prevents the UI from showing outdated data after a create/update/delete.
* **Good:**
  ```tsx
  const mutation = useMutation({
    mutationFn: updateUser,
    onSuccess: () => { queryClient.invalidateQueries({ queryKey: userKeys.all }); },
  });
  ```
* **Bad:** Manually setting query data everywhere without invalidation, causing inconsistent cache states.

### Use Optimistic Updates for UX-Critical Actions
* **Rule:** For actions the user expects to be instant (like, bookmark, toggle), use `onMutate` to optimistically update the cache. Roll back in `onError`.
* **Why:** Eliminates perceived latency for frequent interactions.

---

## 🗄️ Caching Strategy

### Cache Hierarchy
* **Rule:** Follow a 3-layer caching strategy: **React Query/SWR (client)** → **`React.cache()` (per-request server)** → **LRU/Redis (cross-request server)**.
* **Why:** Each layer serves a different lifecycle. Client cache reduces network calls; server per-request cache deduplicates within a render; cross-request cache reduces database load.

### Set Explicit `staleTime` and `gcTime`
* **Rule:** Always configure `staleTime` (when data is considered fresh) and `gcTime` (garbage collection time) per query.
* **Why:** Default `staleTime: 0` causes unnecessary refetches. Set it according to data volatility (e.g., `staleTime: 5 * 60 * 1000` for user profiles, `staleTime: 0` for real-time chat).

---

## 🔐 Environment Management

### Never Expose Server Secrets to the Client
* **Rule:** Prefix client-safe env vars with `NEXT_PUBLIC_`. All other env vars are server-only.
* **Why:** Non-prefixed env vars are stripped from the client bundle by Next.js. Accidentally using them client-side results in `undefined`.
* **Good:**
  ```env
  DATABASE_URL=postgresql://...         # Server only
  NEXT_PUBLIC_API_URL=https://api.com   # Safe for client
  ```
* **Bad:** Using `process.env.DATABASE_URL` inside a `'use client'` component.

### Validate Env at Startup
* **Rule:** Use `t3-env` or a Zod schema to validate all required environment variables at build/start time.
* **Why:** Catches missing or malformed env vars immediately, not in production at 3 AM.

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
* **Rule:** Integrate an error tracking service (Sentry, Datadog) and capture unhandled exceptions with context (user, route, action).
* **Why:** `console.error` is invisible in production. Error tracking aggregates, deduplicates, and alerts on real issues.

---

## 🧰 DX & Tooling

### Lint & Format on Save
* **Rule:** Enforce ESLint + Prettier on every save and pre-commit (via `lint-staged` + `husky`).
* **Why:** Eliminates style debates in code reviews and catches bugs early.

### Path Aliases
* **Rule:** Use `@/` path aliases (configured in `tsconfig.json`) instead of relative imports beyond 2 levels deep.
* **Why:** `import { Button } from '@/components/Button'` is cleaner and refactor-safe compared to `'../../../components/Button'`.

### Strict TypeScript Config
* **Rule:** Enable `"strict": true`, `"noUncheckedIndexedAccess": true`, and `"exactOptionalPropertyTypes": true` in `tsconfig.json`.
* **Why:** Catches entire classes of runtime bugs (null access, undefined array elements, optional property confusion) at compile time.

---

## 🌿 Git Conventions

### Conventional Commits
* **Rule:** Follow Conventional Commits format: `type(scope): description`.
* **Why:** Enables automated changelogs, semantic versioning, and clear commit history.
* **Good:**
  ```
  feat(auth): add OAuth2 Google login
  fix(dashboard): resolve chart flickering on resize
  refactor(api): extract shared fetcher utility
  ```
* **Bad:** `"fix stuff"`, `"update"`, `"WIP"`, `"asdf"`

### Branch Naming
* **Rule:** Use `type/ticket-description` format (e.g., `feat/AUTH-123-google-login`, `fix/DASH-456-chart-flicker`).
* **Why:** Links branches to tickets for traceability. Makes branch lists scannable.

### Small, Focused PRs
* **Rule:** One PR = one concern. Max 400 lines changed (excluding generated files). Split large features into stacked PRs.
* **Why:** Large PRs get rubber-stamped. Small PRs get reviewed thoroughly.

---

## 📂 Folder Public / Private Convention

### Public vs Private Modules
* **Rule:** Each feature folder exposes only its public API through a single `index.ts`. Internal modules are considered private.
* **Why:** Prevents other features from depending on internal implementation details, making refactoring safe.
* **Good:**
  ```text
  src/features/billing/
  ├── index.ts                # Public API: export { BillingPage, useBilling }
  ├── components/
  │   ├── BillingPage.tsx      # .tsx → PascalCase ✅ (Public, exported via index.ts)
  │   └── InvoiceRow.tsx       # .tsx → PascalCase ✅ (Private, NOT exported)
  ├── hooks/
  │   └── use-billing.ts       # .ts → kebab-case ✅ (Public, exported via index.ts)
  └── utils/
      └── format-currency.ts   # .ts → kebab-case ✅ (Private, NOT exported)
  ```
* **Bad:** Importing `features/billing/utils/format-currency` directly from another feature. This couples two features at the implementation level.

### Shared vs Feature Code
* **Rule:** Code used by **3+ features** goes into `src/components/` or `src/lib/`. Code used by **1-2 features** stays inside the feature folder.
* **Why:** Prevents premature abstraction. Don't generalize until there's proven reuse.

---

## 🛑 Anti-Patterns

1. **Prop Drilling:** Passing props down 5+ levels. Use React Context for dependency injection or compound components.
2. **Global State for Local Forms:** Putting form input values into a global Redux/Zustand store. Keep state local to the form.
3. **Huge Components:** Components over 300 lines of code. Break them down into smaller, composable units.
4. **Interleaved DOM Reads/Writes:** Reading `offsetWidth` and immediately writing `style.width` in a loop, causing layout thrashing.
5. **Boolean Prop Explosion:** Adding a new boolean prop every time a component's appearance changes slightly.
6. **Scattered Query Keys:** Using inline string arrays as query keys across multiple files instead of a centralized key factory.
7. **Unvalidated API Responses:** Casting `as User` without runtime validation. Trust nothing from the network.
8. **`console.log` in Production:** Using `console.log` for debugging and forgetting to remove it. Use structured logging.
9. **God Service Files:** A single `api.ts` file with 50+ functions. Split by domain (e.g., `userApi.ts`, `billingApi.ts`).
10. **Missing Error Boundaries:** Not wrapping route segments in `error.tsx`, causing white-screen crashes.

---

## ✅ Checklists

### PR Review Checklist
- [ ] **Architecture:** Are Server Components used optimally? Is `"use client"` pushed down to the leaves?
- [ ] **Boundaries:** Are `server-only` / `client-only` imports used correctly? No secret leakage?
- [ ] **Performance:** Are there any sequential `await` calls that could be `Promise.all()`? Are barrel files avoided?
- [ ] **State:** Is derived state calculated during render (no `useEffect` state syncing)?
- [ ] **Types:** Are there any `any` types or unsafe type assertions (`as unknown as X`)?
- [ ] **API Contract:** Are API responses validated with Zod schemas (not just `as Type`)?
- [ ] **Security:** Are Server Actions validating authentication internally? Is external data parsed with Zod?
- [ ] **Mutations:** Do mutations invalidate the correct query keys? Is optimistic UI used where appropriate?
- [ ] **A11y:** Are semantic HTML tags used? Do interactive elements have focus states?
- [ ] **Clean Code:** Are components small, composable, and free of boolean prop explosions?
- [ ] **Git:** Does the commit follow Conventional Commits format? Is the PR focused (<400 lines)?
- [ ] **Env:** Are all new env vars documented and validated? No `NEXT_PUBLIC_` leak of secrets?

### Common Mistakes
- Forgetting to handle the `error` state in data fetching.
- Mutating React state directly (e.g., `state.list.push(item)` instead of `setList([...state.list, item])`).
- Passing inline objects/functions to `memo` components, breaking memoization.
- Not adding `{ passive: true }` to wheel/scroll event listeners, causing scroll jank.
- Passing the entire DB object from a Server Component to a Client Component instead of just the required string/number properties.
- Using `console.log` instead of a structured logger in server code.
- Forgetting `queryClient.invalidateQueries()` after a successful mutation.
- Not setting `staleTime` on React Query hooks, causing excessive refetching.
- Using `process.env.SECRET` inside a client component (will be `undefined`).
- Committing `.env.local` or hardcoded API keys to the repository.
