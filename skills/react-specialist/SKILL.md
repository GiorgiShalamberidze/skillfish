---
name: react-specialist
description: Advanced React patterns: Server Components, Suspense, concurrent rendering, custom hooks, state management, and performance optimization guides.
---

# React Specialist

Deep technical reference for senior React developers building production applications. Covers React 19 Server Components, concurrent rendering primitives, advanced hook composition, state management trade-offs, performance optimization strategies, error boundaries, and testing patterns. Every section includes working code, rationale, and anti-patterns to avoid.

## Table of Contents

- [Server Components and Client Components](#server-components-and-client-components)
- [Concurrent Features](#concurrent-features)
- [Advanced Hook Patterns](#advanced-hook-patterns)
- [State Management Strategies](#state-management-strategies)
- [Performance Optimization](#performance-optimization)
- [Error Handling and Boundaries](#error-handling-and-boundaries)
- [Testing Patterns](#testing-patterns)
- [Common Patterns Quick Reference](#common-patterns-quick-reference)

---

## Server Components and Client Components

### The RSC Mental Model

Server Components execute on the server and produce a serializable tree (the RSC payload). They never ship JavaScript to the client. Client Components run in the browser and are the only place you can use hooks, event handlers, or browser APIs.

The boundary is defined by the `"use client"` directive at the top of a module. Everything imported by a Client Component becomes part of the client bundle. Everything without the directive defaults to a Server Component in frameworks that support RSC (Next.js App Router, Waku, etc.).

### When to Use Each

| Use Server Components when | Use Client Components when |
|---|---|
| Fetching data (direct DB/API access) | Handling user interactions (onClick, onChange) |
| Accessing backend resources (fs, env vars) | Using hooks (useState, useEffect, useRef) |
| Keeping secrets server-side | Using browser APIs (localStorage, IntersectionObserver) |
| Reducing client bundle size | Managing local UI state (modals, tabs, forms) |
| Heavy computation that doesn't need interactivity | Third-party libs that use `useEffect` internally |

### Serialization Boundary

Props passed from Server to Client Components must be serializable: strings, numbers, booleans, plain objects, arrays, Date, Map, Set, TypedArrays, ReadableStream, and React elements (JSX). Functions, class instances, and Symbols cannot cross the boundary.

```tsx
// app/page.tsx — Server Component (default)
import { db } from "@/lib/db";
import { ProductList } from "./product-list";

export default async function Page() {
  const products = await db.product.findMany(); // direct DB access
  return <ProductList products={products} />;
}
```

```tsx
// app/product-list.tsx — Client Component
"use client";

import { useState } from "react";
import type { Product } from "@/lib/types";

export function ProductList({ products }: { products: Product[] }) {
  const [search, setSearch] = useState("");
  const filtered = products.filter((p) =>
    p.name.toLowerCase().includes(search.toLowerCase())
  );

  return (
    <div>
      <input value={search} onChange={(e) => setSearch(e.target.value)} />
      {filtered.map((p) => (
        <div key={p.id}>{p.name} — ${p.price}</div>
      ))}
    </div>
  );
}
```

### Composition Pattern: Children Slot

Pass Server Components as children to Client Components to avoid pulling server-only code into the client bundle.

```tsx
// app/layout.tsx — Server Component
import { Sidebar } from "./sidebar"; // Client Component
import { Nav } from "./nav";         // Server Component

export default function Layout({ children }: { children: React.ReactNode }) {
  return (
    <Sidebar>
      <Nav />       {/* Server Component passed as children — stays on server */}
      {children}
    </Sidebar>
  );
}
```

### Anti-patterns

- Marking a component `"use client"` just because a child needs interactivity. Push `"use client"` as far down the tree as possible.
- Importing a Server Component inside a Client Component — this silently converts it to a Client Component and ships the code to the browser.
- Passing non-serializable props (functions, class instances) across the boundary — this throws at runtime.

---

## Concurrent Features

### Suspense

Suspense lets you declare a loading state for a subtree that is waiting on asynchronous work (data fetching, lazy components, RSC streaming).

```tsx
import { Suspense } from "react";
import { Comments } from "./comments";

export default function PostPage({ id }: { id: string }) {
  return (
    <article>
      <PostContent id={id} />
      <Suspense fallback={<CommentsSkeleton />}>
        <Comments postId={id} />
      </Suspense>
    </article>
  );
}
```

Nest Suspense boundaries to create independent loading sequences. The outer shell can render immediately while inner sections stream in.

### useTransition

Marks a state update as non-urgent so React can keep the UI responsive.

```tsx
"use client";

import { useTransition, useState } from "react";

export function SearchPage() {
  const [query, setQuery] = useState("");
  const [results, setResults] = useState<Item[]>([]);
  const [isPending, startTransition] = useTransition();

  function handleSearch(value: string) {
    setQuery(value);                // urgent: update the input immediately
    startTransition(async () => {   // non-urgent: fetch can be interrupted
      const data = await fetchResults(value);
      setResults(data);
    });
  }

  return (
    <div>
      <input value={query} onChange={(e) => handleSearch(e.target.value)} />
      {isPending && <Spinner />}
      <ResultList items={results} />
    </div>
  );
}
```

In React 19, `startTransition` accepts async functions, enabling `await` inside transitions without extra state management.

### useDeferredValue

Defers re-rendering of a value until the browser is idle. Useful when an expensive component depends on rapidly changing input.

```tsx
"use client";

import { useDeferredValue, useMemo } from "react";

export function FilteredList({ query, items }: Props) {
  const deferredQuery = useDeferredValue(query);
  const isStale = query !== deferredQuery;

  const filtered = useMemo(
    () => items.filter((item) => item.name.includes(deferredQuery)),
    [deferredQuery, items]
  );

  return (
    <div style={{ opacity: isStale ? 0.6 : 1 }}>
      {filtered.map((item) => <Row key={item.id} item={item} />)}
    </div>
  );
}
```

### Streaming SSR and Selective Hydration

With Suspense on the server, React streams HTML in chunks. Each Suspense boundary can independently hydrate on the client. Users can interact with already-hydrated sections while others are still loading.

Priority: React prioritizes hydrating the component the user is interacting with, even if it appears later in the stream.

### Anti-patterns

- Wrapping every component in Suspense — adds unnecessary boundaries and delays rendering of content that is already available.
- Using `useTransition` for instant feedback actions (form submission with redirect) — the delay makes the UI feel sluggish.
- Calling `useDeferredValue` on values that rarely change — the overhead provides no benefit.

---

## Advanced Hook Patterns

### Custom Hook Architecture

Extract stateful logic into hooks that compose cleanly. A custom hook should do one thing and return a stable interface.

```tsx
function useLocalStorage<T>(key: string, initialValue: T) {
  const [stored, setStored] = useState<T>(() => {
    if (typeof window === "undefined") return initialValue;
    try {
      const item = window.localStorage.getItem(key);
      return item ? (JSON.parse(item) as T) : initialValue;
    } catch {
      return initialValue;
    }
  });

  const setValue = useCallback(
    (value: T | ((prev: T) => T)) => {
      setStored((prev) => {
        const next = value instanceof Function ? value(prev) : value;
        window.localStorage.setItem(key, JSON.stringify(next));
        return next;
      });
    },
    [key]
  );

  return [stored, setValue] as const;
}
```

### useReducer for Complex State

When state transitions depend on previous state or multiple fields update together, `useReducer` is clearer than multiple `useState` calls.

```tsx
type State = { items: Item[]; status: "idle" | "loading" | "error"; error: string | null };
type Action =
  | { type: "FETCH_START" }
  | { type: "FETCH_SUCCESS"; payload: Item[] }
  | { type: "FETCH_ERROR"; error: string };

function reducer(state: State, action: Action): State {
  switch (action.type) {
    case "FETCH_START":
      return { ...state, status: "loading", error: null };
    case "FETCH_SUCCESS":
      return { items: action.payload, status: "idle", error: null };
    case "FETCH_ERROR":
      return { ...state, status: "error", error: action.error };
  }
}

function useFetchItems(url: string) {
  const [state, dispatch] = useReducer(reducer, {
    items: [],
    status: "idle",
    error: null,
  });

  useEffect(() => {
    dispatch({ type: "FETCH_START" });
    fetch(url)
      .then((r) => r.json())
      .then((data) => dispatch({ type: "FETCH_SUCCESS", payload: data }))
      .catch((e) => dispatch({ type: "FETCH_ERROR", error: e.message }));
  }, [url]);

  return state;
}
```

### useSyncExternalStore

The correct way to subscribe to external stores (Redux, Zustand internals, browser APIs) without tearing during concurrent rendering.

```tsx
import { useSyncExternalStore } from "react";

function useOnlineStatus() {
  return useSyncExternalStore(
    (callback) => {
      window.addEventListener("online", callback);
      window.addEventListener("offline", callback);
      return () => {
        window.removeEventListener("online", callback);
        window.removeEventListener("offline", callback);
      };
    },
    () => navigator.onLine,       // client snapshot
    () => true                     // server snapshot
  );
}
```

### React 19: useOptimistic and useFormStatus

```tsx
"use client";

import { useOptimistic } from "react";
import { useFormStatus } from "react-dom";

function SubmitButton() {
  const { pending } = useFormStatus();
  return <button disabled={pending}>{pending ? "Saving..." : "Save"}</button>;
}

function TodoList({ todos, addAction }: Props) {
  const [optimisticTodos, addOptimistic] = useOptimistic(
    todos,
    (state, newTodo: string) => [...state, { id: crypto.randomUUID(), text: newTodo, pending: true }]
  );

  return (
    <form action={async (formData) => {
      const text = formData.get("text") as string;
      addOptimistic(text);
      await addAction(formData);
    }}>
      <input name="text" />
      <SubmitButton />
      <ul>
        {optimisticTodos.map((t) => (
          <li key={t.id} style={{ opacity: t.pending ? 0.5 : 1 }}>{t.text}</li>
        ))}
      </ul>
    </form>
  );
}
```

### Hook Composition

Compose small hooks into larger abstractions without prop drilling.

```tsx
function useAuth() {
  const session = useSession();
  const permissions = usePermissions(session?.user?.id);
  const isAdmin = useMemo(() => permissions.includes("admin"), [permissions]);
  return { session, permissions, isAdmin, isAuthenticated: !!session };
}
```

### Anti-patterns

- Calling hooks conditionally or inside loops — breaks React's hook ordering invariant.
- Using `useEffect` for derived state — compute it during render instead.
- Returning unstable references from custom hooks (new objects/arrays every render) without memoization.

---

## State Management Strategies

### Decision Framework

| Scenario | Recommended Tool |
|---|---|
| Local UI state (toggle, input value) | `useState` |
| Complex local state with transitions | `useReducer` |
| Shared state across a few nearby components | Lift state up + props |
| Theme, locale, auth token (rarely changes) | React Context |
| Shared client state, frequent updates | Zustand or Jotai |
| Server/async state (API data, caching) | TanStack Query (React Query) |
| Form state | React Hook Form or native form actions (React 19) |
| URL state (filters, pagination) | `useSearchParams` / `nuqs` |

### Context Pitfalls and Fixes

Context triggers a re-render of every consumer when the value changes. Mitigate this:

```tsx
// Split contexts by update frequency
const ThemeContext = createContext<Theme>("light");          // rarely changes
const UserContext = createContext<User | null>(null);        // changes on login/logout
const CartContext = createContext<CartState>(initialCart);    // changes often

// For frequently changing context, memoize the value
function CartProvider({ children }: { children: React.ReactNode }) {
  const [cart, dispatch] = useReducer(cartReducer, initialCart);
  const value = useMemo(() => ({ cart, dispatch }), [cart]);
  return <CartContext.Provider value={value}>{children}</CartContext.Provider>;
}
```

### Zustand: Minimal Global State

```tsx
import { create } from "zustand";
import { immer } from "zustand/middleware/immer";

interface BearStore {
  bears: number;
  increase: () => void;
  reset: () => void;
}

const useBearStore = create<BearStore>()(
  immer((set) => ({
    bears: 0,
    increase: () => set((state) => { state.bears += 1; }),
    reset: () => set({ bears: 0 }),
  }))
);

// Select only what you need to avoid unnecessary re-renders
function BearCount() {
  const bears = useBearStore((s) => s.bears);
  return <span>{bears}</span>;
}
```

### TanStack Query for Server State

```tsx
import { useQuery, useMutation, useQueryClient } from "@tanstack/react-query";

function useProducts() {
  return useQuery({
    queryKey: ["products"],
    queryFn: () => fetch("/api/v1/products").then((r) => r.json()),
    staleTime: 5 * 60 * 1000,      // 5 minutes before refetch
    gcTime: 30 * 60 * 1000,         // 30 minutes in cache
  });
}

function useCreateProduct() {
  const qc = useQueryClient();
  return useMutation({
    mutationFn: (data: NewProduct) =>
      fetch("/api/v1/products", { method: "POST", body: JSON.stringify(data) }),
    onSuccess: () => qc.invalidateQueries({ queryKey: ["products"] }),
  });
}
```

### Anti-patterns

- Using Context for high-frequency state (cursor position, scroll offset) — re-renders the entire subtree.
- Duplicating server state in client stores — use TanStack Query as the cache; don't copy API responses into Zustand.
- Creating a single monolithic store — split by domain (auth, cart, ui) so selectors stay focused.

---

## Performance Optimization

### React.memo — When It Helps

`React.memo` skips re-rendering when props have not changed (shallow comparison). It helps when a component is expensive to render and receives stable props from a parent that re-renders frequently.

```tsx
const ExpensiveChart = React.memo(function Chart({ data }: { data: Point[] }) {
  // heavy SVG rendering
  return <svg>{/* ... */}</svg>;
});
```

Do not wrap every component in `React.memo` — the shallow comparison itself has a cost, and most components are cheap to re-render.

### useMemo and useCallback Correctly

Only memoize when there is a measurable performance problem or when a stable reference is required (dependency of another hook, prop to a memoized child).

```tsx
// Correct: expensive computation
const sorted = useMemo(
  () => items.toSorted((a, b) => a.score - b.score),
  [items]
);

// Correct: stable callback for memoized child
const handleClick = useCallback(
  (id: string) => dispatch({ type: "SELECT", id }),
  [dispatch]
);

return <MemoizedList items={sorted} onSelect={handleClick} />;
```

### React Compiler (React 19+)

The React Compiler (formerly React Forget) auto-memoizes components and hooks at build time. When enabled, you can remove most manual `useMemo`, `useCallback`, and `React.memo` calls.

```js
// babel.config.js or next.config.js
module.exports = {
  experimental: {
    reactCompiler: true,
  },
};
```

The compiler analyzes your code and inserts memoization where it detects benefit. Manual memoization still works alongside it, but becomes redundant in most cases.

### Code Splitting and Lazy Loading

```tsx
import { lazy, Suspense } from "react";

const AdminPanel = lazy(() => import("./admin-panel"));

function App() {
  return (
    <Suspense fallback={<Spinner />}>
      {isAdmin && <AdminPanel />}
    </Suspense>
  );
}
```

For route-level splitting in Next.js, the App Router handles this automatically per page segment.

### Virtualization

For lists with 1000+ items, render only visible rows.

```tsx
import { useVirtualizer } from "@tanstack/react-virtual";

function VirtualList({ items }: { items: Item[] }) {
  const parentRef = useRef<HTMLDivElement>(null);
  const virtualizer = useVirtualizer({
    count: items.length,
    getScrollElement: () => parentRef.current,
    estimateSize: () => 48,
    overscan: 5,
  });

  return (
    <div ref={parentRef} style={{ height: 600, overflow: "auto" }}>
      <div style={{ height: virtualizer.getTotalSize(), position: "relative" }}>
        {virtualizer.getVirtualItems().map((virtual) => (
          <div
            key={virtual.key}
            style={{
              position: "absolute",
              top: virtual.start,
              height: virtual.size,
              width: "100%",
            }}
          >
            {items[virtual.index].name}
          </div>
        ))}
      </div>
    </div>
  );
}
```

### Anti-patterns

- Premature memoization — wrapping everything in `useMemo`/`useCallback` without profiling first. Adds complexity for no gain.
- Memoizing inline objects or arrays passed as props without memoizing the prop itself — the parent still creates a new reference each render.
- Ignoring key stability in lists — using array index as key when items can reorder causes incorrect reconciliation.

---

## Error Handling and Boundaries

### Error Boundaries

Error Boundaries catch JavaScript errors in the component tree during rendering, lifecycle methods, and constructors. They do not catch errors in event handlers, async code, or server-side rendering.

```tsx
"use client";

import { Component, type ErrorInfo, type ReactNode } from "react";

interface Props {
  children: ReactNode;
  fallback?: ReactNode;
  onError?: (error: Error, info: ErrorInfo) => void;
}

interface State {
  hasError: boolean;
  error: Error | null;
}

class ErrorBoundary extends Component<Props, State> {
  state: State = { hasError: false, error: null };

  static getDerivedStateFromError(error: Error): State {
    return { hasError: true, error };
  }

  componentDidCatch(error: Error, info: ErrorInfo) {
    this.props.onError?.(error, info);
    // Send to error tracking service
    reportError(error, info.componentStack);
  }

  render() {
    if (this.state.hasError) {
      return this.props.fallback ?? (
        <div role="alert">
          <h2>Something went wrong</h2>
          <button onClick={() => this.setState({ hasError: false, error: null })}>
            Try again
          </button>
        </div>
      );
    }
    return this.props.children;
  }
}
```

### Suspense + Error Boundary Composition

Combine Suspense for loading states and Error Boundary for failure states to handle the full async lifecycle.

```tsx
<ErrorBoundary fallback={<ErrorFallback />}>
  <Suspense fallback={<Skeleton />}>
    <AsyncDataComponent />
  </Suspense>
</ErrorBoundary>
```

### Error Recovery Patterns

```tsx
"use client";

import { useTransition } from "react";

function ErrorFallback({ error, reset }: { error: Error; reset: () => void }) {
  const [isPending, startTransition] = useTransition();

  return (
    <div role="alert">
      <p>Failed to load: {error.message}</p>
      <button
        disabled={isPending}
        onClick={() => startTransition(() => reset())}
      >
        {isPending ? "Retrying..." : "Retry"}
      </button>
    </div>
  );
}
```

In Next.js App Router, create an `error.tsx` file in any route segment to automatically wrap that segment in an Error Boundary with a `reset` function.

### Granular Boundaries

Place Error Boundaries around independent sections so a failure in one area does not take down the entire page.

```tsx
export default function Dashboard() {
  return (
    <div className="grid grid-cols-3 gap-4">
      <ErrorBoundary fallback={<WidgetError name="Revenue" />}>
        <Suspense fallback={<WidgetSkeleton />}>
          <RevenueChart />
        </Suspense>
      </ErrorBoundary>
      <ErrorBoundary fallback={<WidgetError name="Users" />}>
        <Suspense fallback={<WidgetSkeleton />}>
          <UserMetrics />
        </Suspense>
      </ErrorBoundary>
      <ErrorBoundary fallback={<WidgetError name="Orders" />}>
        <Suspense fallback={<WidgetSkeleton />}>
          <OrderFeed />
        </Suspense>
      </ErrorBoundary>
    </div>
  );
}
```

### Anti-patterns

- Catching errors in event handlers with Error Boundaries — they only catch render errors. Use try/catch for event handlers and async operations.
- Having a single top-level Error Boundary — one failure blanks the entire app.
- Swallowing errors silently — always log to an error tracking service (Sentry, Datadog, etc.).

---

## Testing Patterns

### Testing Library Best Practices

Query by how users find elements: `getByRole`, `getByLabelText`, `getByPlaceholderText`. Avoid `getByTestId` unless there is no accessible alternative.

```tsx
import { render, screen } from "@testing-library/react";
import userEvent from "@testing-library/user-event";
import { LoginForm } from "./login-form";

test("submits credentials and shows success", async () => {
  const user = userEvent.setup();
  const onSubmit = vi.fn();

  render(<LoginForm onSubmit={onSubmit} />);

  await user.type(screen.getByLabelText(/email/i), "user@test.com");
  await user.type(screen.getByLabelText(/password/i), "secret123");
  await user.click(screen.getByRole("button", { name: /sign in/i }));

  expect(onSubmit).toHaveBeenCalledWith({
    email: "user@test.com",
    password: "secret123",
  });
});
```

### Testing Custom Hooks

Use `renderHook` from Testing Library to test hooks in isolation.

```tsx
import { renderHook, act } from "@testing-library/react";
import { useCounter } from "./use-counter";

test("increments and resets", () => {
  const { result } = renderHook(() => useCounter(0));

  act(() => result.current.increment());
  expect(result.current.count).toBe(1);

  act(() => result.current.reset());
  expect(result.current.count).toBe(0);
});
```

### Testing Async Components

For components that fetch data, mock the network layer with MSW (Mock Service Worker) instead of mocking `fetch` directly.

```tsx
import { http, HttpResponse } from "msw";
import { setupServer } from "msw/node";
import { render, screen } from "@testing-library/react";
import { QueryClient, QueryClientProvider } from "@tanstack/react-query";
import { ProductList } from "./product-list";

const server = setupServer(
  http.get("/api/v1/products", () =>
    HttpResponse.json([
      { id: "1", name: "Widget", price: 9.99 },
      { id: "2", name: "Gadget", price: 19.99 },
    ])
  )
);

beforeAll(() => server.listen());
afterEach(() => server.resetHandlers());
afterAll(() => server.close());

test("renders products from API", async () => {
  const qc = new QueryClient({ defaultOptions: { queries: { retry: false } } });

  render(
    <QueryClientProvider client={qc}>
      <ProductList />
    </QueryClientProvider>
  );

  expect(await screen.findByText("Widget")).toBeInTheDocument();
  expect(screen.getByText("Gadget")).toBeInTheDocument();
});

test("shows error state on failure", async () => {
  server.use(
    http.get("/api/v1/products", () => HttpResponse.error())
  );

  const qc = new QueryClient({ defaultOptions: { queries: { retry: false } } });

  render(
    <QueryClientProvider client={qc}>
      <ProductList />
    </QueryClientProvider>
  );

  expect(await screen.findByRole("alert")).toBeInTheDocument();
});
```

### Anti-patterns

- Testing implementation details (internal state values, component instance methods) — test behavior the user sees.
- Using `fireEvent` when `userEvent` is available — `userEvent` simulates real browser interactions (focus, keydown, input, keyup).
- Not wrapping state updates in `act` — leads to flaky tests and React warnings.
- Snapshot testing as the primary strategy — snapshots break on every visual change and rarely catch real bugs.

---

## Common Patterns Quick Reference

### Conditional Rendering

```tsx
// Preferred: early return
function Greeting({ user }: { user: User | null }) {
  if (!user) return <LoginPrompt />;
  return <h1>Welcome, {user.name}</h1>;
}

// Inline conditional for small elements
<div>{isLoading ? <Spinner /> : <Content />}</div>

// Render nothing
{showBanner && <Banner />}
```

### Compound Components

```tsx
function Tabs({ children }: { children: ReactNode }) {
  const [active, setActive] = useState(0);
  return (
    <TabsContext.Provider value={{ active, setActive }}>
      {children}
    </TabsContext.Provider>
  );
}

Tabs.List = function TabList({ children }: { children: ReactNode }) {
  return <div role="tablist">{children}</div>;
};

Tabs.Panel = function TabPanel({ index, children }: { index: number; children: ReactNode }) {
  const { active } = useContext(TabsContext);
  if (active !== index) return null;
  return <div role="tabpanel">{children}</div>;
};

// Usage
<Tabs>
  <Tabs.List>
    <Tab index={0}>Details</Tab>
    <Tab index={1}>Reviews</Tab>
  </Tabs.List>
  <Tabs.Panel index={0}><Details /></Tabs.Panel>
  <Tabs.Panel index={1}><Reviews /></Tabs.Panel>
</Tabs>
```

### Render Props (Still Useful)

```tsx
function DataFetcher<T>({ url, children }: { url: string; children: (data: T) => ReactNode }) {
  const { data, error, isLoading } = useFetch<T>(url);
  if (isLoading) return <Spinner />;
  if (error) return <ErrorMessage error={error} />;
  return <>{children(data!)}</>;
}

<DataFetcher<User[]> url="/api/v1/users">
  {(users) => <UserTable users={users} />}
</DataFetcher>
```

### Polymorphic Components

```tsx
type AsProps<C extends React.ElementType> = {
  as?: C;
} & React.ComponentPropsWithoutRef<C>;

function Box<C extends React.ElementType = "div">({
  as,
  children,
  ...props
}: AsProps<C> & { children: ReactNode }) {
  const Component = as || "div";
  return <Component {...props}>{children}</Component>;
}

// Usage
<Box as="section" className="p-4">Content</Box>
<Box as="a" href="/about">Link styled as Box</Box>
```

### React 19 Server Actions

```tsx
// app/actions.ts
"use server";

import { revalidatePath } from "next/cache";
import { db } from "@/lib/db";

export async function createTodo(formData: FormData) {
  const text = formData.get("text") as string;
  await db.todo.create({ data: { text } });
  revalidatePath("/todos");
}
```

```tsx
// app/todos/page.tsx
import { createTodo } from "../actions";

export default async function TodoPage() {
  const todos = await db.todo.findMany();
  return (
    <form action={createTodo}>
      <input name="text" required />
      <button type="submit">Add</button>
    </form>
  );
}
```

### Key Prop Strategy

```tsx
// Reset component state by changing key
<ProfileEditor key={userId} userId={userId} />

// Stable keys for lists — use unique IDs, never array index
{items.map((item) => <Card key={item.id} item={item} />)}
```
