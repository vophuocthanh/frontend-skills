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
* **Good:**
  ```tsx
  function useMouseTracker() {
    const positionRef = useRef({ x: 0, y: 0 });
    useEffect(() => {
      const handler = (e: MouseEvent) => {
        positionRef.current = { x: e.clientX, y: e.clientY }; // No re-render
      };
      window.addEventListener('mousemove', handler);
      return () => window.removeEventListener('mousemove', handler);
    }, []);
    return positionRef;
  }
  ```
* **Bad:** Storing mouse position in `useState`, causing 60+ re-renders per second.

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

### No Barrel Files (for Libraries)
* **Rule:** Never use `index.ts` to re-export modules from **large external libraries** (e.g., `lucide-react`, `@mui/material`). However, using a single `index.ts` per **feature folder** to expose a public API is allowed and encouraged (see [Folder Public/Private Convention](#-folder-public--private-convention)).
* **Why:** Library barrel files force the bundler to parse thousands of unused modules, destroying tree-shaking and bloating the bundle. Feature-level `index.ts` files are small and scoped, so tree-shaking handles them easily.
* **Good:** `import Button from '@mui/material/Button';`
* **Good:** `import { useBilling } from '@/features/billing';` (feature-level index.ts — OK)
* **Bad:** `import { Button } from '@mui/material';` (library-level barrel — kills tree-shaking)

### Dynamic Imports for Heavy Components
* **Rule:** Use `next/dynamic` to lazy-load heavy components (e.g., Monaco Editor, charts, maps) that are not needed for the initial render.
* **Why:** Reduces the initial JS payload, drastically improving Time to Interactive (TTI).
* **Good:**
  ```tsx
  import dynamic from 'next/dynamic';

  const MonacoEditor = dynamic(() => import('@/components/MonacoEditor'), {
    loading: () => <Skeleton height={400} />,
    ssr: false,
  });
  ```
* **Bad:** Statically importing a 500KB chart library at the top of a page that only shows the chart on user interaction.

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

## 🖥️ Client-Side Patterns

### When to Use `"use client"`
* **Rule:** Only add `"use client"` when the component **directly** uses browser APIs, React hooks (`useState`, `useEffect`, `useRef`, event handlers), or third-party client-only libraries. Never add it "just in case."
* **Why:** Every `"use client"` component and all its imports are shipped to the browser as JavaScript. Unnecessary client boundaries bloat the bundle.
* **Decision tree:**
  1. Does it use `useState`, `useEffect`, `useRef`, `useContext`, or event handlers? → `"use client"`
  2. Does it only display data passed via props? → Keep it as a Server Component
  3. Does it use a third-party library that requires the DOM? → `"use client"`
* **Good:**
  ```tsx
  // ✅ Only the interactive part is a client component
  // components/LikeButton.tsx
  'use client';
  import { useState } from 'react';

  export function LikeButton({ initialCount }: { initialCount: number }) {
    const [count, setCount] = useState(initialCount);
    return <button onClick={() => setCount(c => c + 1)}>❤️ {count}</button>;
  }

  // app/posts/[id]/page.tsx — Server Component (NO "use client")
  import { LikeButton } from '@/components/LikeButton';
  export default async function PostPage({ params }: { params: { id: string } }) {
    const post = await getPost(params.id);
    return (
      <article>
        <h1>{post.title}</h1>
        <p>{post.content}</p>
        <LikeButton initialCount={post.likes} /> {/* Client island */}
      </article>
    );
  }
  ```
* **Bad:** Adding `"use client"` to the entire page because one button needs `onClick`.

### Client Component Composition (Server → Client Data Flow)
* **Rule:** Pass **serializable primitives** (strings, numbers, booleans) or **pre-fetched data** from Server Components to Client Components via props. Never pass functions, classes, or non-serializable objects.
* **Why:** The RSC-to-Client boundary serializes props into the HTML payload. Non-serializable values will throw runtime errors.
* **Good:**
  ```tsx
  // Server Component fetches, Client Component interacts
  // app/dashboard/page.tsx (Server)
  export default async function DashboardPage() {
    const stats = await getStats(); // { revenue: 42000, users: 1200 }
    return <DashboardCharts revenue={stats.revenue} userCount={stats.users} />;
  }

  // components/DashboardCharts.tsx (Client)
  'use client';
  export function DashboardCharts({ revenue, userCount }: { revenue: number; userCount: number }) {
    // Interactive charts using client-side libraries
  }
  ```
* **Bad:** Passing `db` connection, functions, or `Date` objects from Server to Client Components.

### URL State Management
* **Rule:** Use URL search params (`useSearchParams`) as the single source of truth for UI state that should survive page refreshes, back/forward navigation, and link sharing (e.g., filters, pagination, sort order, tabs).
* **Why:** URL state is shareable, bookmarkable, and survives refreshes. `useState` state is lost on navigation.
* **Good:**
  ```tsx
  'use client';
  import { useSearchParams, useRouter, usePathname } from 'next/navigation';

  export function UserFilters() {
    const searchParams = useSearchParams();
    const router = useRouter();
    const pathname = usePathname();
    const currentRole = searchParams.get('role') ?? 'all';

    function setFilter(role: string) {
      const params = new URLSearchParams(searchParams.toString());
      params.set('role', role);
      router.push(`${pathname}?${params.toString()}`);
    }

    return (
      <select value={currentRole} onChange={(e) => setFilter(e.target.value)}>
        <option value="all">All</option>
        <option value="admin">Admin</option>
        <option value="user">User</option>
      </select>
    );
  }
  ```
* **Bad:** Using `useState` for filter/pagination state, losing it when the user navigates away and comes back.

### Event Handling Patterns
* **Rule:** Apply `debounce` for input events (search, resize), `throttle` for continuous events (scroll, mousemove), and `{ passive: true }` for scroll/wheel/touch listeners.
* **Why:** Unthrottled events fire 60+ times per second, causing jank. Missing `passive` blocks the browser's scroll optimization.
* **Good:**
  ```tsx
  'use client';
  import { useDebouncedCallback } from 'use-debounce';

  export function SearchInput({ onSearch }: { onSearch: (q: string) => void }) {
    const debouncedSearch = useDebouncedCallback((value: string) => {
      onSearch(value);
    }, 300);

    return <input onChange={(e) => debouncedSearch(e.target.value)} placeholder="Search..." />;
  }

  // For scroll/wheel — always use passive
  useEffect(() => {
    const handler = () => { /* ... */ };
    window.addEventListener('scroll', handler, { passive: true });
    return () => window.removeEventListener('scroll', handler);
  }, []);
  ```
* **Bad:** Calling `fetch()` on every keystroke without debounce. Adding scroll listeners without `{ passive: true }`.

### Controlled vs Uncontrolled Components
* **Rule:** Use **uncontrolled** components (via `ref` or `react-hook-form`) for forms to avoid re-renders per keystroke. Use **controlled** components only when you need to react to every value change in real-time (e.g., live preview, character counter).
* **Why:** Controlled inputs trigger a re-render on every keystroke. For forms with 10+ fields, this creates noticeable lag.
* **Good:**
  ```tsx
  // ✅ Uncontrolled with react-hook-form — no re-renders per keystroke
  const { register, handleSubmit } = useForm<FormData>();
  <input {...register('email')} />
  ```
* **Bad:**
  ```tsx
  // ❌ Controlled with useState — re-renders entire form on every keystroke
  const [email, setEmail] = useState('');
  <input value={email} onChange={(e) => setEmail(e.target.value)} />
  ```

### Browser Web APIs
* **Rule:** Wrap browser-specific APIs (`IntersectionObserver`, `ResizeObserver`, `matchMedia`, `navigator.clipboard`) in custom hooks with proper cleanup. Always check for API availability before using.
* **Why:** Server-side rendering has no access to browser APIs. Missing cleanup causes memory leaks.
* **Good:**
  ```tsx
  // hooks/use-intersection-observer.ts
  export function useIntersectionObserver(ref: RefObject<Element>, options?: IntersectionObserverInit) {
    const [isIntersecting, setIsIntersecting] = useState(false);

    useEffect(() => {
      const element = ref.current;
      if (!element || typeof IntersectionObserver === 'undefined') return;

      const observer = new IntersectionObserver(([entry]) => {
        setIsIntersecting(entry.isIntersecting);
      }, options);

      observer.observe(element);
      return () => observer.disconnect(); // ✅ Cleanup
    }, [ref, options]);

    return isIntersecting;
  }
  ```
* **Bad:** Using `window.innerWidth` directly in render without checking `typeof window !== 'undefined'`, causing SSR hydration mismatches.

### Client-Side Navigation
* **Rule:** Use `next/link` for all internal navigation. Use `useRouter().push()` only for programmatic navigation (e.g., after form submission). Never use `<a href>` for internal routes.
* **Why:** `next/link` provides prefetching, client-side transitions, and maintains scroll position. Raw `<a>` causes full page reloads.
* **Good:**
  ```tsx
  import Link from 'next/link';
  <Link href="/dashboard" prefetch={true}>Dashboard</Link>

  // Programmatic navigation after action
  const router = useRouter();
  const onSubmit = async (data: FormData) => {
    await createUser(data);
    router.push('/users');
  };
  ```
* **Bad:** `<a href="/dashboard">Dashboard</a>` — full page reload, no prefetching.

### Global Client State (Zustand / Context)
* **Rule:** Use **Zustand** for high-frequency, cross-component client state (theme toggle, sidebar open/close, cart items). Use **React Context** for low-frequency dependency injection (current user, API client, feature flags). Never use Context for state that updates frequently.
* **Why:** Context re-renders ALL consumers on every state change. Zustand uses subscriptions — only components that read the changed slice re-render.
* **Good:**
  ```tsx
  // ✅ Zustand for frequently changing state
  // stores/use-sidebar-store.ts
  import { create } from 'zustand';

  interface SidebarStore {
    isOpen: boolean;
    toggle: () => void;
  }

  export const useSidebarStore = create<SidebarStore>((set) => ({
    isOpen: true,
    toggle: () => set((state) => ({ isOpen: !state.isOpen })),
  }));

  // In any component — only re-renders when `isOpen` changes
  const isOpen = useSidebarStore((state) => state.isOpen);
  ```
* **Bad:**
  ```tsx
  // ❌ Context for high-frequency state — re-renders ALL consumers
  const SidebarContext = createContext({ isOpen: true, toggle: () => {} });
  // Every component wrapped in <SidebarContext.Consumer> re-renders on toggle
  ```

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
* **Good:**
  ```tsx
  // app/dashboard/error.tsx
  'use client';
  import { useEffect } from 'react';
  import { captureException } from '@sentry/nextjs';

  export default function DashboardError({ error, reset }: { error: Error; reset: () => void }) {
    useEffect(() => { captureException(error); }, [error]);
    return (
      <div role="alert">
        <h2>Something went wrong</h2>
        <p>{error.message}</p>
        <button onClick={reset}>Try again</button>
      </div>
    );
  }
  ```
* **Bad:** Not wrapping route segments in `error.tsx`, causing the entire app to white-screen on a single component error.

### Accessibility (A11y)
* **Rule:** Follow WCAG 2.1 AA standards. Use correct semantic HTML, manage focus, and ensure sufficient color contrast.
* **Why:** Ensures the application is usable by people with disabilities, and is legally required in many jurisdictions.
* **Requirements:**
  * **Color contrast:** Minimum 4.5:1 for normal text, 3:1 for large text (WCAG AA).
  * **Keyboard navigation:** All interactive elements must be reachable and operable via keyboard (Tab, Enter, Escape).
  * **Focus management:** Use `:focus-visible` for focus rings. Never use `outline: none` without providing a visual alternative. Trap focus inside modals.
  * **ARIA attributes:** Add `aria-label`, `aria-describedby`, and `role` to custom interactive elements that lack semantic meaning.
  * **Dynamic content:** Use `aria-live="polite"` for content that updates asynchronously (toasts, search results).
  * **Skip navigation:** Add a skip-to-content link for screen reader users.
  * **Touch targets:** Interactive elements must be at least 44x44px.
* **Good:**
  ```tsx
  <button aria-label="Close dialog" onClick={onClose}>
    <XIcon aria-hidden="true" />
  </button>
  ```
* **Bad:** `<div onClick={onClose}>X</div>` — not focusable, no role, no label.

### Security
* **Rule:** Follow defense-in-depth principles for all client and server code.
* **Why:** Frontend apps are the first attack surface. A single vulnerability can expose user data.
* **Requirements:**
  * **XSS:** Avoid `dangerouslySetInnerHTML`. If unavoidable, sanitize with `dompurify`.
  * **CSRF:** Next.js Server Actions include CSRF protection by default — never bypass it.
  * **Token storage:** Never store auth tokens in `localStorage` or `sessionStorage`. Use `httpOnly` cookies.
  * **Input sanitization:** Validate and sanitize all user inputs before rendering or sending to APIs.
  * **CSP headers:** Configure `Content-Security-Policy` headers in `next.config.js` to prevent inline script injection.
  * **Rate limiting:** Apply rate limiting on Server Actions that perform writes or authentication.
  * **Dependencies:** Run `npm audit` regularly. Never ignore critical/high severity vulnerabilities.

### Testing Strategy
* **Rule:** Follow the Testing Trophy: prioritize Integration Tests, supplement with Unit and E2E tests.
* **Why:** Integration tests give the highest confidence-to-effort ratio. They test real user flows and are resilient to refactoring.
* **Layers:**
  * **Unit tests:** Custom hooks (`renderHook`), pure utility functions, Zod schemas. Keep them fast and isolated.
  * **Integration tests:** Full user flows with React Testing Library. Render components, simulate interactions, assert on DOM output.
  * **E2E tests:** Critical business paths (login, checkout, payment) with Playwright. Run in CI against a staging environment.
  * **Coverage:** Target 80%+ for business logic hooks. No coverage requirement for pure presentational components.
* **Conventions:**
  * Test file naming: `*.test.ts` / `*.test.tsx`, colocated next to the source file.
  * Mock at the network boundary using `msw` (Mock Service Worker). Never mock internal modules.
  * Write tests that assert on what the user sees (`getByRole`, `getByText`), not on implementation details (`state`, `props`).
* **Good:**
  ```tsx
  // features/auth/components/LoginForm.test.tsx
  test('shows error on invalid credentials', async () => {
    server.use(http.post('/api/login', () => HttpResponse.json({ error: 'Invalid' }, { status: 401 })));
    render(<LoginForm />);
    await userEvent.type(screen.getByLabelText('Email'), 'test@test.com');
    await userEvent.type(screen.getByLabelText('Password'), 'wrong');
    await userEvent.click(screen.getByRole('button', { name: 'Sign in' }));
    expect(await screen.findByRole('alert')).toHaveTextContent('Invalid');
  });
  ```
* **Bad:** Testing internal state values (`expect(hook.result.current.isLoading).toBe(true)`) instead of DOM output.

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
* **Rule:** For actions the user expects to be instant (like, bookmark, toggle, form submit), show the expected result immediately while the mutation is in-flight. Use `onMutate` to optimistically update the cache. Roll back in `onError`.
* **Why:** Eliminates perceived latency for frequent interactions. Users don't stare at spinners for simple actions.
* **Good:**
  ```tsx
  const mutation = useMutation({
    mutationFn: toggleFavorite,
    onMutate: async (itemId) => {
      await queryClient.cancelQueries({ queryKey: itemKeys.detail(itemId) });
      const previous = queryClient.getQueryData(itemKeys.detail(itemId));
      queryClient.setQueryData(itemKeys.detail(itemId), (old: Item) => ({
        ...old, isFavorite: !old.isFavorite,
      }));
      return { previous };
    },
    onError: (_err, itemId, context) => {
      queryClient.setQueryData(itemKeys.detail(itemId), context?.previous);
    },
    onSettled: (_data, _err, itemId) => {
      queryClient.invalidateQueries({ queryKey: itemKeys.detail(itemId) });
    },
  });
  ```
* **Bad:** Showing a full-page spinner while toggling a like button.

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

## 🖼️ Image & Asset Optimization

### Use Framework Image Components
* **Rule:** Always use `next/image` (Next.js) or equivalent optimized image component. Never use raw `<img>` tags.
* **Why:** Framework image components provide automatic lazy loading, responsive sizing, format conversion (WebP/AVIF), and prevent Cumulative Layout Shift (CLS).
* **Good:**
  ```tsx
  import Image from 'next/image';
  <Image src="/hero.jpg" alt="Hero banner" width={1200} height={600} priority />
  ```
* **Bad:** `<img src="/hero.jpg" />` — no dimensions (causes CLS), no lazy loading, no format optimization.

### Image Requirements
* **Rule:** Every image element MUST have:
  * `alt` text (descriptive for content images, `alt=""` for decorative)
  * Explicit `width` and `height` (or `fill` with a sized container)
  * `priority` attribute only for above-the-fold images (hero, LCP)
* **Why:** Missing dimensions cause layout shift. Missing `alt` fails accessibility. Missing `priority` delays LCP.

### Static Assets
* **Rule:** Store static assets in `public/` directory. Use content-hash filenames for cache busting where possible. Prefer SVG for icons, WebP/AVIF for photos.
* **Why:** `public/` assets are served directly by the CDN. Content hashes enable aggressive caching.

---

## 📱 Responsive Design

### Mobile-First Breakpoints
* **Rule:** Design for mobile first, then add complexity for larger screens using `min-width` breakpoints.
* **Why:** Mobile-first ensures the base experience works on the smallest screens. Adding features upward is simpler than removing them downward.
* **Good:** `@media (min-width: 768px) { ... }` or Tailwind `md:flex`
* **Bad:** `@media (max-width: 768px) { ... }` — leads to override chains and specificity wars.

### Container Sizing
* **Rule:** Never use fixed pixel widths for layout containers. Use `max-w-*` + auto margins, or percentage/viewport-based units.
* **Why:** Fixed widths break on screens smaller than the specified value.

### Touch Targets
* **Rule:** All interactive elements (buttons, links, inputs) must have a minimum touch target of 44x44px.
* **Why:** Required by WCAG 2.1 for motor accessibility. Small targets frustrate users on mobile.

### Breakpoint Testing
* **Rule:** Test all layouts at these standard breakpoints: 320px, 375px, 768px, 1024px, 1440px.
* **Why:** Covers the range from small phones to large desktops. Bugs frequently hide at breakpoint boundaries.

---

## 🔄 Loading, Empty & Error States

### Handle All Data States
* **Rule:** Every data-fetching component MUST explicitly handle 4 states: `loading`, `error`, `empty`, and `success`.
* **Why:** Unhandled states cause blank screens, broken layouts, or misleading UI.
* **Good:**
  ```tsx
  function UserList() {
    const { data: users, isLoading, error } = useUsers();
    if (isLoading) return <UserListSkeleton />;
    if (error) return <ErrorState message={error.message} />;
    if (users.length === 0) return <EmptyState icon={<UsersIcon />} message="No users found" />;
    return <ul>{users.map(u => <UserCard key={u.id} user={u} />)}</ul>;
  }
  ```
* **Bad:** Only handling the success state, rendering nothing or crashing when data is loading or empty.

### Use Skeletons Over Spinners
* **Rule:** Use Skeleton placeholders that mirror the layout of the final content. Avoid generic spinners.
* **Why:** Skeletons reduce Cumulative Layout Shift (CLS) and give users a sense of progress. Spinners provide no layout context.

### Create Reusable State Components
* **Rule:** Build shared `<EmptyState>`, `<ErrorState>`, and `<Skeleton>` components in `src/components/`.
* **Why:** Ensures consistent UX across all features without duplicating markup.

---

## 🌐 Internationalization (i18n)

### No Hardcoded User-Facing Strings
* **Rule:** Never hardcode user-visible text in components. Use an i18n library (`next-intl`, `react-i18next`, or equivalent).
* **Why:** Hardcoded strings make localization impossible without rewriting components.
* **Good:**
  ```tsx
  import { useTranslations } from 'next-intl';
  function LoginPage() {
    const t = useTranslations('auth');
    return <h1>{t('login.title')}</h1>;
  }
  ```
* **Bad:** `<h1>Welcome back!</h1>` — cannot be translated without code changes.

### Colocate Translations
* **Rule:** Store translation keys alongside their feature, or in a centralized `messages/` directory organized by locale.
* **Why:** Makes it easy to find and update translations when features change.

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
