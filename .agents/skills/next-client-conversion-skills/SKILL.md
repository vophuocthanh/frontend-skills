---
name: react-client-mastery
description: Production-ready AI coding ruleset for Next.js Client-Side Rendering (CSR) — SOLID principles, React hooks, TypeScript, state management, styling, forms, testing, and performance patterns.
---

# Next.js Client-Side Rendering (CSR) Ruleset

> This skill enforces modern React (v19+) client-side patterns, strict TypeScript, and scalable architecture for projects that primarily use Client-Side Rendering. Optimized for AI agents and engineering teams.

## 🟢 DO / 🔴 DON'T Quick Reference

* **DO** fetch data in parallel using `Promise.all()`.
* **DON'T** create sequential network waterfalls.
* **DO** use composition (Compound Components) for UI variants.
* **DON'T** use boolean props (e.g., `<Button isPrimary isLarge />`).
* **DO** calculate derived state during the render cycle.
* **DON'T** use `useEffect` to synchronize state.
* **DO** strictly type everything; use discriminated unions for complex states.
* **DON'T** ever use `any`.
* **DO** use `useSearchParams` for shareable UI state (filters, pagination).
* **DON'T** store URL-worthy state in `useState`.
* **DO** debounce input events and throttle scroll events.
* **DON'T** use barrel files (`index.ts` re-exports) for large libraries.
* **DO** give every component/hook exactly one reason to change (SRP).
* **DON'T** extend a component by adding `if (variant === ...)` branches (OCP).
* **DO** depend on abstractions (props, interfaces, injected clients), not concrete modules (DIP).
---

## 🧱 SOLID Principles (React Edition)

> SOLID applies to components, hooks, and modules — not just classes. These rules are the *reasoning layer* behind the patterns below.

### S — Single Responsibility Principle
* **Rule:** One component, one hook, one module = **one reason to change**. Rendering, business logic, and data fetching are three different reasons.
* **Why:** A component that fetches, transforms, validates, and renders must be rewritten whenever *any* of those change — and can't be tested in isolation.
* **Good:**
  ```tsx
  // api/user-api.ts        → changes when the endpoint changes
  export async function fetchUsers(): Promise<User[]> { /* ... */ }

  // hooks/use-user-list.ts → changes when business rules change
  export function useUserList(query: string) {
    const { data, isLoading } = useQuery({ queryKey: userKeys.lists(), queryFn: fetchUsers });
    const users = useMemo(() => filterUsers(data ?? [], query), [data, query]);
    return { users, isLoading };
  }

  // components/UserList.tsx → changes when the design changes
  export function UserList({ query }: { query: string }) {
    const { users, isLoading } = useUserList(query);
    if (isLoading) return <UserListSkeleton />;
    return <ul>{users.map(u => <UserCard key={u.id} user={u} />)}</ul>;
  }
  ```
* **Bad:** A 300-line `UserList.tsx` containing `fetch()`, filtering, sorting, analytics, and JSX.
* **Smell:** You describe the file with "and" — "it renders the table **and** exports CSV **and** handles pagination".

### O — Open/Closed Principle
* **Rule:** Components must be **open for extension, closed for modification**. Extend via `children`, render props, slots, and variant maps — never by editing the component to add another `if`.
* **Why:** Every new use case that forces you to reopen a shared component risks breaking existing consumers.
* **Good:**
  ```tsx
  // Extend by composition — DataTable never changes
  <DataTable data={users}>
    <DataTable.Toolbar><ExportButton /></DataTable.Toolbar>
    <DataTable.Empty><EmptyState message="No users" /></DataTable.Empty>
  </DataTable>

  // Extend by variant map (cva) — add a variant, don't touch the render body
  const button = cva('rounded font-medium', {
    variants: { intent: { primary: '...', danger: '...', ghost: '...' } },
  });
  ```
* **Bad:**
  ```tsx
  function Button({ isPrimary, isDanger, isGhost, isIconOnly }: Props) {
    if (isPrimary) return <button className="bg-blue-600">...</button>;
    if (isDanger) return <button className="bg-red-600">...</button>;
    // 🔴 every new style = another branch inside the component
  }
  ```

### L — Liskov Substitution Principle
* **Rule:** A specialized component must be usable **anywhere its base is used**, without surprises. Wrappers must forward `ref`, `...rest` props, and standard event handlers.
* **Why:** If `<PrimaryButton>` silently drops `onClick`, `disabled`, or `type="submit"`, it is not substitutable for `<button>` and will break inside forms and dialogs.
* **Good:**
  ```tsx
  type ButtonProps = React.ComponentPropsWithoutRef<'button'> &
    VariantProps<typeof button>;

  export const Button = forwardRef<HTMLButtonElement, ButtonProps>(
    function Button({ intent, className, ...rest }, ref) {
      return <button ref={ref} className={button({ intent, className })} {...rest} />;
    },
  );
  ```
* **Bad:**
  ```tsx
  // 🔴 Drops ref, type, disabled, aria-*; breaks the moment it's used in a form
  export function Button({ label, onClick }: { label: string; onClick: () => void }) {
    return <button onClick={onClick}>{label}</button>;
  }
  ```
* **Also applies to hooks:** every variant of a hook must return the **same shape**. `useUsers()` and `useArchivedUsers()` should both return `{ data, isLoading, error }` — never one returning an array and the other an object.

### I — Interface Segregation Principle
* **Rule:** Components must depend only on the props they actually use. Pass **narrow, specific props**, not whole god-objects "just in case".
* **Why:** Passing the full `user` object into `<Avatar>` couples the avatar to the entire `User` type — a change to `User.billing` forces `Avatar` to recompile, retest, and re-render.
* **Good:**
  ```tsx
  <Avatar src={user.avatarUrl} name={user.name} />

  // Split fat prop interfaces by responsibility
  interface Sortable { sortKey: string; onSort: (key: string) => void }
  interface Selectable { selectedIds: string[]; onSelect: (id: string) => void }
  type TableProps = Sortable & Selectable & { rows: Row[] };
  ```
* **Bad:**
  ```tsx
  <Avatar user={user} /> // 🔴 Avatar now depends on all 30 fields of User
  <Table config={hugeConfigObject} /> // 🔴 one fat interface for 5 unrelated concerns
  ```
* **Also applies to stores:** select **slices**, not the whole store — `useSidebarStore(s => s.isOpen)`, never `useSidebarStore()`.

### D — Dependency Inversion Principle
* **Rule:** High-level components depend on **abstractions**, not concrete implementations. Inject dependencies (API client, storage, analytics, feature flags) via props, custom hooks, or Context — never import a concrete module deep inside a UI component.
* **Why:** A component that imports `axios` directly cannot be tested, mocked, or reused with a different transport. The dependency arrow must point *away* from the UI.
* **Good:**
  ```tsx
  // Abstraction (owned by the high-level module)
  export interface AuthGateway {
    login(input: LoginInput): Promise<Session>;
  }

  // Injection point
  const AuthGatewayContext = createContext<AuthGateway | null>(null);
  export function useAuthGateway(): AuthGateway {
    const gateway = useContext(AuthGatewayContext);
    if (!gateway) throw new Error('AuthGatewayProvider is missing');
    return gateway;
  }

  // Consumer depends on the interface only
  export function useLogin() {
    const gateway = useAuthGateway();
    return useMutation({ mutationFn: (input: LoginInput) => gateway.login(input) });
  }

  // Tests swap in a fake — no network, no msw needed for unit tests
  <AuthGatewayContext.Provider value={fakeAuthGateway}>
  ```
* **Bad:**
  ```tsx
  export function LoginForm() {
    async function onSubmit(data: LoginInput) {
      await axios.post('https://api.example.com/login', data); // 🔴 UI welded to axios + URL
      localStorage.setItem('token', '...');                    // 🔴 UI welded to storage
    }
  }
  ```

### SOLID → Pattern Map
| Principle | Enforce it with |
|---|---|
| **S**RP | Logic-in-Hook / UI-in-Component, one feature folder per domain |
| **O**CP | Compound Components, `children`/slots, `cva` variant maps |
| **L**SP | `forwardRef` + `ComponentPropsWithoutRef` + `{...rest}` spread |
| **I**SP | Narrow props, split prop interfaces, Zustand slice selectors |
| **D**IP | Context-injected gateways, centralized typed API client, Zod at the boundary |

---

## ⚛️ React Component & Hook Patterns

### Composition over Configuration
* **Rule:** Use compound components and `children` props instead of adding boolean flags for every UI state.
* **Why:** Prevents monolithic components filled with complex `if/else` logic.
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
* **Why:** React won't recreate these elements on every render, saving CPU cycles.
* **Good:**
  ```tsx
  const staticHeader = <h1>Dashboard</h1>;
  export function Dashboard() { return <div>{staticHeader}</div>; }
  ```

### Calculate Derived State During Render
* **Rule:** Never use `useEffect` to synchronize state or calculate derived values. Do it directly during render.
* **Why:** `useEffect` state updates cause an immediate second render, leading to layout thrashing.
* **Good:**
  ```tsx
  function UserList({ users, query }) {
    const filteredUsers = users.filter(u => u.name.includes(query));
    return <ul>...</ul>;
  }
  ```
* **Bad:** Syncing `filteredUsers` via `useState` and `useEffect`.

### Primitive Dependencies in Hooks
* **Rule:** Only pass primitives (strings, numbers, booleans) into dependency arrays, or use stable references.
* **Why:** Objects and arrays are recreated on every render, triggering infinite loops or unnecessary effects.
* **Good:** `useEffect(() => { fetchUser(userId); }, [userId]);`
* **Bad:** `useEffect(() => { fetchUser(user.id); }, [user]);`

### Use useRef for Transient Values
* **Rule:** If a value updates frequently and doesn't directly affect the DOM, store it in a `ref`.
* **Why:** Bypasses the React render cycle entirely.
* **Good:**
  ```tsx
  function useMouseTracker() {
    const positionRef = useRef({ x: 0, y: 0 });
    useEffect(() => {
      const handler = (e: MouseEvent) => {
        positionRef.current = { x: e.clientX, y: e.clientY };
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
* **Why:** React renders `0` to the DOM if you do `count && <span>{count}</span>`.
* **Good:** `count > 0 ? <span>{count}</span> : null`
* **Bad:** `count && <span>{count}</span>`

### Logic-in-Hook, UI-in-Component
* **Rule:** Move ALL business logic, state management, side effects into a custom hook. The `.tsx` component file must ONLY contain JSX rendering.
* **Why:** Separating logic from presentation makes components trivially testable and reusable.
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
* **Bad:** Component contains logic, state, effects, AND rendering all mixed together.

### Colocate Related State in a Single Hook
* **Rule:** All state that must update together MUST live inside the SAME custom hook. Never split tightly-coupled state across multiple hooks.
* **Why:** When related state is split across separate hook instances, React batches updates per hook — leading to one render with stale values. This causes UI desync and subtle bugs.
* **Good:**
  ```tsx
  function useCheckout() {
    const [items, setItems] = useState<CartItem[]>([]);
    const [discount, setDiscount] = useState(0);
    const total = useMemo(() => {
      const subtotal = items.reduce((sum, i) => sum + i.price * i.qty, 0);
      return subtotal * (1 - discount / 100);
    }, [items, discount]);
    const addItem = useCallback((item: CartItem) => setItems(prev => [...prev, item]), []);
    const applyDiscount = useCallback((pct: number) => setDiscount(pct), []);
    return { items, total, discount, addItem, applyDiscount };
  }
  ```
* **Bad:** Splitting `items` and `discount` into separate hooks, causing desync.

---

## 🖥️ Client-Side Patterns

### URL State Management
* **Rule:** Use URL search params (`useSearchParams`) as the single source of truth for UI state that should survive page refreshes and link sharing (filters, pagination, sort, tabs).
* **Why:** URL state is shareable, bookmarkable, and survives refreshes.
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
      </select>
    );
  }
  ```
* **Bad:** Using `useState` for filter/pagination state, losing it on navigation.

### Event Handling Patterns
* **Rule:** Apply `debounce` for input events (search, resize), `throttle` for continuous events (scroll, mousemove), and `{ passive: true }` for scroll/wheel/touch listeners.
* **Why:** Unthrottled events fire 60+ times per second, causing jank.
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
* **Bad:** Calling `fetch()` on every keystroke without debounce.

### Controlled vs Uncontrolled Components
* **Rule:** Use **uncontrolled** components (via `ref` or `react-hook-form`) for forms. Use **controlled** only when you need real-time value reactions (live preview, character counter).
* **Why:** Controlled inputs trigger a re-render on every keystroke.
* **Good:**
  ```tsx
  const { register, handleSubmit } = useForm<FormData>();
  <input {...register('email')} />
  ```
* **Bad:** Using `useState` for every input field with manual `onChange`.

### Browser Web APIs
* **Rule:** Wrap browser-specific APIs (`IntersectionObserver`, `ResizeObserver`, `matchMedia`) in custom hooks with proper cleanup. Always check for API availability.
* **Why:** SSR has no access to browser APIs. Missing cleanup causes memory leaks.
* **Good:**
  ```tsx
  export function useIntersectionObserver(ref: RefObject<Element>, options?: IntersectionObserverInit) {
    const [isIntersecting, setIsIntersecting] = useState(false);
    useEffect(() => {
      const element = ref.current;
      if (!element || typeof IntersectionObserver === 'undefined') return;
      const observer = new IntersectionObserver(([entry]) => {
        setIsIntersecting(entry.isIntersecting);
      }, options);
      observer.observe(element);
      return () => observer.disconnect();
    }, [ref, options]);
    return isIntersecting;
  }
  ```
* **Bad:** Using `window.innerWidth` directly in render without checking `typeof window`.

### Client-Side Navigation
* **Rule:** Use `next/link` for all internal navigation. Use `useRouter().push()` only for programmatic navigation. Never use `<a href>` for internal routes.
* **Why:** `next/link` provides prefetching and client-side transitions. Raw `<a>` causes full page reloads.

### Global Client State (Zustand / Context)
* **Rule:** Use **Zustand** for high-frequency, cross-component client state (theme, sidebar, cart). Use **React Context** for low-frequency dependency injection (current user, API client, feature flags).
* **Why:** Context re-renders ALL consumers on every state change. Zustand uses subscriptions — only components that read the changed slice re-render.
* **Good:**
  ```tsx
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
* **Bad:** Using Context for high-frequency state — re-renders ALL consumers.

---

## 🚀 Performance: Waterfalls & Bundle Size

### Eliminate Waterfalls
* **Rule:** Never `await` sequential, independent operations. Always use `Promise.all()`.
* **Why:** Sequential awaits multiply network latency.
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
* **Rule:** Never use `index.ts` to re-export modules from **large external libraries**. Feature-level `index.ts` for public API is allowed.
* **Why:** Library barrel files destroy tree-shaking and bloat the bundle.
* **Good:** `import Button from '@mui/material/Button';`
* **Good:** `import { useBilling } from '@/features/billing';` (feature-level — OK)
* **Bad:** `import { Button } from '@mui/material';` (library-level — kills tree-shaking)

### Dynamic Imports for Heavy Components
* **Rule:** Use `next/dynamic` or `React.lazy` to lazy-load heavy components not needed for initial render.
* **Good:**
  ```tsx
  import dynamic from 'next/dynamic';
  const MonacoEditor = dynamic(() => import('@/components/MonacoEditor'), {
    loading: () => <Skeleton height={400} />,
    ssr: false,
  });
  ```

---

## 📘 TypeScript Strict Rules

### Avoid `any` Completely
* **Rule:** Do not use `any`. Use `unknown` and narrow with type guards or Zod.
* **Why:** `any` bypasses the compiler, nullifying TypeScript's safety.

### Discriminated Unions for State
* **Rule:** Use discriminated unions to model state machines, not multiple boolean flags.
* **Why:** Prevents impossible states.
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
* **Rule:** Keep state local until a sibling needs it, then lift to the nearest common ancestor.
* **Why:** Global state introduces coupling. Use Zustand for high-frequency global state, Context for DI.

---

## 📝 Engineering Standards

### Strict Naming Conventions
* **`.tsx` files (Components):** `PascalCase` → `UserProfile.tsx`, `LoginForm.tsx`
* **`.ts` files (Hooks, Utils, Schemas, API):** `kebab-case` → `use-auth.ts`, `format-currency.ts`
* **Component names:** `PascalCase` → `export function UserProfile()`
* **Hooks:** `camelCase` starting with `use` → `export function useAuth()`
* **Booleans:** Prefix with `is`, `has`, `should`, `can` → `isLoading`, `hasPermission`
* **Event Handlers:** `handle` prefix → `handleSubmit`; Props with `on` → `onSubmit`
* **Folders:** Always `kebab-case` → `user-profile/`, `billing-api/`

### Error Handling & Validation
* **Rule:** Use Error Boundaries to isolate crashes. Validate all API data and form inputs using Zod.
* **Good:**
  ```tsx
  'use client';
  export default function DashboardError({ error, reset }: { error: Error; reset: () => void }) {
    useEffect(() => { captureException(error); }, [error]);
    return (
      <div role="alert">
        <h2>Something went wrong</h2>
        <button onClick={reset}>Try again</button>
      </div>
    );
  }
  ```

### Accessibility (A11y)
* **Rule:** Follow WCAG 2.1 AA standards.
* **Requirements:**
  * Color contrast: 4.5:1 for normal text, 3:1 for large text
  * Keyboard navigation: Tab, Enter, Escape for all interactive elements
  * Focus management: `:focus-visible` for focus rings, trap focus in modals
  * ARIA attributes for custom interactive elements
  * `aria-live="polite"` for async content updates
  * Touch targets: at least 44x44px
* **Good:** `<button aria-label="Close dialog" onClick={onClose}><XIcon aria-hidden="true" /></button>`
* **Bad:** `<div onClick={onClose}>X</div>` — not focusable, no role, no label.

### Security
* **XSS:** Avoid `dangerouslySetInnerHTML`. Sanitize with `dompurify` if unavoidable.
* **Token storage:** Never store auth tokens in `localStorage`. Use `httpOnly` cookies.
* **Input sanitization:** Validate all user inputs before rendering or sending to APIs.
* **Dependencies:** Run `npm audit` regularly.

### Testing Strategy
* **Rule:** Follow the Testing Trophy: prioritize Integration Tests.
* **Unit tests:** Custom hooks (`renderHook`), pure utility functions, Zod schemas.
* **Integration tests:** Full user flows with React Testing Library.
* **E2E tests:** Critical paths (login, checkout) with Playwright.
* **Coverage:** 80%+ for business logic hooks.
* **Conventions:** Test files colocated (`*.test.tsx`), mock at network boundary (`msw`).
* **Good:**
  ```tsx
  test('shows error on invalid credentials', async () => {
    server.use(http.post('/api/login', () => HttpResponse.json({ error: 'Invalid' }, { status: 401 })));
    render(<LoginForm />);
    await userEvent.type(screen.getByLabelText('Email'), 'test@test.com');
    await userEvent.click(screen.getByRole('button', { name: 'Sign in' }));
    expect(await screen.findByRole('alert')).toHaveTextContent('Invalid');
  });
  ```

---

## 📡 API Contract & Data Layer

### Type-Safe API Contracts
* **Rule:** Define shared request/response types using Zod schemas. Infer types from schemas (`z.infer<typeof schema>`).
* **Good:**
  ```tsx
  import { z } from 'zod';
  export const UserSchema = z.object({ id: z.string(), name: z.string(), email: z.string().email() });
  export type User = z.infer<typeof UserSchema>;
  const data = UserSchema.parse(apiResponse);
  ```
* **Bad:** Using `as User` type assertion without runtime validation.

### Centralized API Client
* **Rule:** Create a single, typed API client with interceptors for auth headers, error transformation, and base URL.
* **Why:** Prevents scattered `fetch()` calls with inconsistent error handling.

---

## 📋 Forms

### Use React Hook Form + Zod
* **Rule:** Use `react-hook-form` with `@hookform/resolvers/zod` for all forms.
* **Good:**
  ```tsx
  const schema = z.object({ email: z.string().email(), password: z.string().min(8) });
  const { register, handleSubmit, formState: { errors } } = useForm({ resolver: zodResolver(schema) });
  ```
* **Bad:** Using `useState` for every input field with manual validation.

---

## 🎨 Styling Strategy

### Utility-First with Escape Hatches
* **Rule:** Use TailwindCSS as primary. For complex dynamic styles, use CSS Modules or `cva`.
* **Good:**
  ```tsx
  import { cva } from 'class-variance-authority';
  const button = cva('px-4 py-2 rounded font-medium', {
    variants: {
      intent: { primary: 'bg-blue-600 text-white', danger: 'bg-red-600 text-white' },
      size: { sm: 'text-sm', lg: 'text-lg px-6 py-3' },
    },
    defaultVariants: { intent: 'primary', size: 'sm' },
  });
  ```
* **Bad:** Mixing inline styles, global CSS, Styled Components, and Tailwind.

### No Inline Styles for Layout
* **Rule:** Never use inline `style={{}}` for layout/spacing/color. Reserve for truly dynamic values only.

---

## 🔄 React Query / SWR Conventions

### Colocate Query Keys
* **Rule:** Define query keys as constants alongside their fetcher functions. Use a factory pattern.
* **Good:**
  ```tsx
  export const userKeys = {
    all: ['users'] as const,
    lists: () => [...userKeys.all, 'list'] as const,
    detail: (id: string) => [...userKeys.all, 'detail', id] as const,
  };
  export function useUsers() {
    return useQuery({ queryKey: userKeys.lists(), queryFn: fetchUsers });
  }
  ```
* **Bad:** Scattering string literal query keys across multiple files.

### Separate Queries from Mutations
* **Rule:** Use `useQuery` for reads, `useMutation` for writes. Never mix.

### Invalidate After Mutation
* **Rule:** Always call `queryClient.invalidateQueries()` after a successful mutation.

### Use Optimistic Updates for UX-Critical Actions
* **Rule:** For instant-feel actions (like, bookmark, toggle), use `onMutate` to optimistically update the cache. Roll back in `onError`.
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

### Set Explicit `staleTime` and `gcTime`
* **Rule:** Always configure `staleTime` and `gcTime` per query based on data volatility.

---

## 🖼️ Image & Asset Optimization

### Use Optimized Image Components
* **Rule:** Always use `next/image` or equivalent. Never use raw `<img>` tags.
* **Rule:** Every image MUST have `alt`, `width`/`height`, and `priority` only for above-the-fold.
* **Rule:** Prefer SVG for icons, WebP/AVIF for photos. Store static assets in `public/`.

---

## 📱 Responsive Design

* **Rule:** Mobile-first breakpoints (`min-width`, not `max-width`).
* **Rule:** Never use fixed pixel widths for containers.
* **Rule:** Touch targets must be at least 44x44px.
* **Rule:** Test at 320px, 375px, 768px, 1024px, 1440px.

---

## 🔄 Loading, Empty & Error States

### Handle All Data States
* **Rule:** Every data-fetching component MUST handle: `loading`, `error`, `empty`, `success`.
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

### Use Skeletons Over Spinners
* **Rule:** Use Skeleton placeholders that mirror the final layout. Avoid generic spinners.

### Create Reusable State Components
* **Rule:** Build shared `<EmptyState>`, `<ErrorState>`, `<Skeleton>` in `src/components/`.

---

## 🌐 Internationalization (i18n)

* **Rule:** Never hardcode user-visible text. Use `react-i18next` or `next-intl`.
* **Rule:** Colocate translations with features or in a centralized `messages/` directory.

---

## 🧰 DX & Tooling

* **Rule:** Enforce ESLint + Prettier on every save and pre-commit (`lint-staged` + `husky`).
* **Rule:** Use `@/` path aliases instead of relative imports beyond 2 levels.
* **Rule:** Enable `"strict": true`, `"noUncheckedIndexedAccess": true` in `tsconfig.json`.

---

## 🌿 Git Conventions

* **Conventional Commits:** `type(scope): description` (e.g., `feat(auth): add Google login`)
* **Branch Naming:** `type/ticket-description` (e.g., `feat/AUTH-123-google-login`)
* **Small PRs:** One PR = one concern. Max 400 lines changed.

---

## 📂 Folder Public / Private Convention

* **Rule:** Each feature folder exposes only its public API through `index.ts`. Internal modules are private.
* **Rule:** Code used by 3+ features → `src/components/` or `src/lib/`. Otherwise stays in the feature folder.

---

## 🛑 Anti-Patterns

1. **Prop Drilling:** Passing props down 5+ levels. Use Context or compound components.
2. **Global State for Local Forms:** Keep form state local.
3. **Huge Components:** Components over 300 lines. Break them down.
4. **Interleaved DOM Reads/Writes:** Causes layout thrashing.
5. **Boolean Prop Explosion:** Use composition instead.
6. **Scattered Query Keys:** Use centralized key factory.
7. **Unvalidated API Responses:** Never `as User` without runtime validation.
8. **`console.log` in Production:** Use structured logging.
9. **God Service Files:** Split by domain. *(violates SRP)*
10. **Missing Error Boundaries:** Causes white-screen crashes.
11. **Variant `if`-Chains:** Adding a branch inside a shared component for each new style. *(violates OCP — use `cva` or composition)*
12. **Leaky Wrappers:** Custom `<Button>`/`<Input>` that swallow `ref`, `disabled`, `type`, or `aria-*`. *(violates LSP)*
13. **God-Object Props:** `<Avatar user={user} />` instead of `<Avatar src name />`. *(violates ISP)*
14. **Hard-Wired Dependencies:** Importing `axios`/`localStorage`/SDKs directly inside a UI component. *(violates DIP — inject via Context)*
15. **Whole-Store Subscriptions:** `useStore()` without a selector, re-rendering on every unrelated change. *(violates ISP)*

---

## ✅ PR Review Checklist

- [ ] **Performance:** Any sequential `await` that could be `Promise.all()`? Barrel files avoided?
- [ ] **State:** Derived state calculated during render (no `useEffect` state syncing)?
- [ ] **Types:** Any `any` types or unsafe type assertions?
- [ ] **API Contract:** API responses validated with Zod schemas?
- [ ] **Mutations:** Correct query key invalidation? Optimistic UI where appropriate?
- [ ] **A11y:** Semantic HTML? Focus states? Color contrast?
- [ ] **Clean Code:** Components small, composable, free of boolean prop explosions?
- [ ] **SRP:** Can each new/changed file be described without the word "and"?
- [ ] **OCP:** Was a shared component extended by composition/variants rather than a new `if` branch?
- [ ] **LSP:** Do wrapper components forward `ref` and spread `...rest`? Do hook variants return the same shape?
- [ ] **ISP:** Are props narrow (`src`, `name`) instead of god-objects (`user`)? Are store reads done with selectors?
- [ ] **DIP:** Do UI components import `axios`/`localStorage`/SDKs directly, or receive them through an injected abstraction?
- [ ] **Git:** Conventional Commits? PR focused (<400 lines)?
- [ ] **Testing:** Integration tests for user flows? Mocks at network boundary?

### Common Mistakes
- Forgetting to handle `error` state in data fetching.
- Mutating React state directly (`state.list.push(item)`).
- Passing inline objects/functions to `memo` components, breaking memoization.
- Not adding `{ passive: true }` to scroll/wheel listeners.
- Forgetting `queryClient.invalidateQueries()` after mutation.
- Not setting `staleTime` on React Query hooks.
- Storing auth tokens in `localStorage`.
