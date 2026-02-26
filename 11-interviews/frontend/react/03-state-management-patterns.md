# State Management Patterns — When to Use What

## The State Spectrum

Not all state is the same. Choosing the right tool depends on what
kind of state you're managing:

```
Local UI state ─────── Shared UI state ─────── Server state ─────── URL state
(component)            (cross-component)        (async/cached)       (route params)
     │                      │                        │                    │
  useState              Context / Zustand       TanStack Query        useSearchParams
  useReducer            Redux (if complex)       SWR                  URL params
```

---

## Local State (useState / useReducer)

**Default choice.** Keep state as close to where it's used as possible.

```jsx
function SearchBar() {
  const [query, setQuery] = useState('');
  return <input value={query} onChange={e => setQuery(e.target.value)} />;
}
```

Lift state up only when two siblings need the same data:

```
Before:                          After:
  Parent                           Parent  ← owns [filter]
   ├─ FilterBar (owns state)        ├─ FilterBar (receives filter, onChange)
   └─ ResultsList (needs state?)    └─ ResultsList (receives filter)
```

---

## Context — Shared State Without Prop Drilling

Context solves the "pass props through 5 levels" problem:

```
App → Layout → Sidebar → Nav → NavItem → ThemeIcon
                                          ↑ needs theme

Without context: theme prop drilled through every level
With context:    ThemeIcon reads directly from ThemeContext
```

### Pattern: Context + useReducer

For complex shared state, combine them:

```jsx
const TodoContext = createContext(null);
const TodoDispatchContext = createContext(null);

function TodoProvider({ children }) {
  const [todos, dispatch] = useReducer(todoReducer, []);
  return (
    <TodoContext.Provider value={todos}>
      <TodoDispatchContext.Provider value={dispatch}>
        {children}
      </TodoDispatchContext.Provider>
    </TodoContext.Provider>
  );
}

// Separate contexts so components that only dispatch
// don't re-render when todos change
```

### When context falls short

- **High-frequency updates** (mouse position, animations) — every
  consumer re-renders on every change.
- **Selector patterns** — you can't subscribe to a slice. Every
  consumer gets the full value.
- **Complex state with many actions** — context + reducer works but
  gets unwieldy at scale.

---

## External State Libraries

### Zustand — Minimal, flexible

```jsx
import { create } from 'zustand';

const useStore = create((set) => ({
  count: 0,
  increment: () => set((state) => ({ count: state.count + 1 })),
}));

function Counter() {
  const count = useStore((state) => state.count);  // selector
  const increment = useStore((state) => state.increment);
  return <button onClick={increment}>{count}</button>;
}
```

Key advantage: **selectors**. Components only re-render when their
selected slice changes. No provider wrapper needed.

### Redux Toolkit — For complex, predictable state

```jsx
const counterSlice = createSlice({
  name: 'counter',
  initialState: { value: 0 },
  reducers: {
    increment: (state) => { state.value += 1; },  // Immer under the hood
  },
});
```

Use Redux when you need:
- DevTools time-travel debugging
- Middleware (logging, async orchestration)
- Strict unidirectional data flow across a large app
- Team convention / predictability

---

## Server State (TanStack Query / SWR)

Server state is fundamentally different from UI state:

```
UI State:                    Server State:
- Synchronous               - Asynchronous
- Owned by the client       - Owned by the server
- Always up to date         - Potentially stale
- You control mutations     - Shared across clients
```

### TanStack Query

```jsx
function UserProfile({ userId }) {
  const { data, isLoading, error } = useQuery({
    queryKey: ['user', userId],
    queryFn: () => fetchUser(userId),
    staleTime: 5 * 60 * 1000,  // 5 minutes
  });

  if (isLoading) return <Skeleton />;
  if (error) return <ErrorMessage error={error} />;
  return <Profile user={data} />;
}
```

What it handles for you:
- **Caching** — same queryKey returns cached data
- **Deduplication** — multiple components requesting same data → one fetch
- **Background refetch** — stale data shown immediately, fresh data fetched
- **Retry** — automatic retry with exponential backoff
- **Garbage collection** — unused cache entries cleaned up
- **Optimistic updates** — update UI before server confirms

### Mutations

```jsx
const mutation = useMutation({
  mutationFn: updateUser,
  onSuccess: () => {
    queryClient.invalidateQueries({ queryKey: ['user'] });
  },
});
```

---

## Decision Framework

```
What kind of state is it?

├─ Only used in one component?
│  └─ useState / useReducer
│
├─ Shared by a few nearby components?
│  └─ Lift state up to nearest common ancestor
│
├─ Shared across distant components (low-frequency updates)?
│  └─ Context + useReducer
│
├─ Shared across distant components (high-frequency or complex)?
│  └─ Zustand or Redux
│
├─ Data from the server?
│  └─ TanStack Query / SWR
│
└─ In the URL?
   └─ useSearchParams / URL state
```

---

## Interview Questions

---

**Q: When would you use Context vs an external state library?**

**How to answer:** Show you understand Context's limitations and can
pick the right tool based on the situation.

**Answer:**

**What most candidates say:** "Context is for theme and auth. Use Redux
for complex state." This is a memorization-level answer.

**What a senior candidate says:** The real distinction is about **update
frequency and subscription granularity**, not complexity.

Context has a fundamental architectural limitation: **every consumer
re-renders when the context value changes, period.** There's no built-in
selector. If you store `{ user, theme, notifications }` in one context
and notifications update every 5 seconds, every component reading user
or theme also re-renders every 5 seconds.

You can try to fight this — split into multiple contexts, memo children,
wrap values in useMemo — but you're papering over a fundamental
limitation. Context was designed for **dependency injection** (providing
values down a tree), not for **state management** (subscribing to
changes). Using it as a state manager is fighting React's design.

An external library like Zustand solves the hard problem — **selectors**:

```jsx
// Context: ALL consumers re-render on ANY change
const { user, notifications } = useContext(AppContext);

// Zustand: only re-renders when this specific slice changes
const user = useStore(s => s.user);
// notifications updated 100 times? This component: 0 re-renders.
```

**Decision framework:**
```
Context: theme, auth, locale, feature flags
  → changes rarely (once per session), read everywhere
  → dependency injection, not high-frequency subscriptions

Zustand/Jotai: cart, filters, UI state, real-time data
  → changes often, components need selective subscriptions
  → selectors prevent the re-render cascade

Redux: large teams with complex business logic
  → time-travel debugging, middleware ecosystem, enforced patterns
  → the overhead is justified when you have 20+ developers
```

**The production insight:** I've seen teams spend weeks optimizing Context
re-renders with useMemo/memo/split contexts. Then they switch to Zustand
(20 lines of code) and the problem disappears. Know when you're fighting
the tool vs using the right tool.

---

**Q: What's the difference between server state and client state?**

**How to answer:** Define each by its characteristics, then explain why
they need fundamentally different tools.

**Answer:**

**What most candidates say:** "Server state comes from the server, client
state is local." This is a definition, not an understanding.

**What a senior candidate says:** The distinction isn't about *where*
the data lives — it's about **ownership and staleness**, which demands
fundamentally different tools.

| | Client State | Server State |
|---|---|---|
| **Owned by** | This browser tab | The server (shared across all clients) |
| **Can go stale?** | Never — you're the only writer | Always — another user/tab can modify it |
| **Async?** | No — immediately available | Yes — network round-trip required |
| **Persisted?** | Only in memory | Yes — database/API |

**Why this matters in practice:** Most bugs in React apps come from
**treating server state like client state.** A developer writes:

```jsx
const [users, setUsers] = useState([]);
useEffect(() => { fetch('/api/users').then(...).then(setUsers) }, []);
```

This is broken infrastructure code disguised as a simple pattern. They've
built a cache with no expiration, no deduplication, no error retry, no
background refresh, and no invalidation. Three components need the same
users? Three fetches. User navigates away and back? Refetches. Network
blip? Crashes.

**The insight:** If you're writing data fetching with useState, you're
building a worse version of TanStack Query. It will be incomplete, have
edge cases, and fail in production in subtle ways (stale data shown after
mutations, race conditions between concurrent fetches, memory leaks from
unmounted components).

Client state (form inputs, modal open/closed, selected tab) is simple
and synchronous — useState is perfect. Server state needs a library that
handles the hard problems: caching, deduplication, staleness, retries,
garbage collection, optimistic updates, and cache invalidation.

---

**Q: Why separate state from dispatch in context?**

**How to answer:** Explain the re-render problem and show how splitting
fixes it.

**Answer:** When you put both state and dispatch in one context:

```jsx
<TodoContext.Provider value={{ todos, dispatch }}>
```

Every component that calls `useContext(TodoContext)` re-renders when
**anything** in the value changes — including components that only call
`dispatch` and never read `todos`. A button that adds a todo re-renders
every time the todo list changes, even though it doesn't display the list.

**Fix:** Split into two contexts:

```jsx
<TodoStateContext.Provider value={todos}>
  <TodoDispatchContext.Provider value={dispatch}>
```

Now components that only dispatch (`useContext(TodoDispatchContext)`) don't
re-render when todos change. `dispatch` is a stable reference from
useReducer — it never changes between renders, so the dispatch context
value never changes, and its consumers never re-render unnecessarily.

This pattern is especially important when the state is large or changes
frequently. It's the same principle as Redux's `useSelector` — only
subscribe to what you need.

---

**Q: What problem does TanStack Query solve that useState doesn't?**

**How to answer:** Walk through what you'd have to build manually, then
contrast with what TanStack Query gives you.

**Answer:** If you fetch data with useState + useEffect, you need to
handle all of this manually:

```jsx
// Manual approach — what you have to build:
const [data, setData] = useState(null);
const [loading, setLoading] = useState(false);
const [error, setError] = useState(null);

useEffect(() => {
  let cancelled = false;
  setLoading(true);
  fetch('/api/users')
    .then(r => r.json())
    .then(d => { if (!cancelled) { setData(d); setLoading(false); }})
    .catch(e => { if (!cancelled) { setError(e); setLoading(false); }});
  return () => { cancelled = true; };
}, []);
```

And you're still missing: caching (navigate away and back → refetches),
deduplication (3 components need the same data → 3 fetches), background
refresh (data goes stale while user is on the page), retry on failure,
optimistic updates, pagination, infinite scroll, and cache invalidation
after mutations.

**Here's what actually breaks in production:** Three components request
the same user. Three fetches fire. One completes, one fails silently, one
hangs. Now you have stale data in one component and an error in another.
The user sees their old name in the header and their new name in the
profile. This is what "build your own cache" gets you.

TanStack Query handles all of it:

```jsx
const { data, isLoading, error } = useQuery({
  queryKey: ['users'],
  queryFn: () => fetch('/api/users').then(r => r.json()),
  staleTime: 5 * 60 * 1000,  // cached for 5 minutes
});
```

Three components call this → **one** fetch. Navigate away and back →
cached data shown instantly, background refetch if stale. Network error →
automatic retry with exponential backoff. Mutation happens → invalidate
the query key, all three components get fresh data simultaneously.

The rule: **useState for client state, TanStack Query for server state.**
If you find yourself writing `useState + useEffect + fetch`, stop and use
a server state library. The edge cases you'll miss are bugs waiting to
happen.

---

**Q: How do you decide where to put state?**

**How to answer:** Describe the decision hierarchy from simplest to most
complex, showing you default to the simplest option.

**Answer:** I follow this hierarchy — start at the top and only move
down when you have to:

1. **Local state (useState)** — default choice. Keep state as close to
   where it's used as possible. A search input's query belongs in the
   search component, not in a global store.

2. **Lift state up** — when two sibling components need the same data,
   move the state to their nearest common parent and pass it down as
   props. This is the standard React pattern.

3. **Composition** — sometimes "prop drilling" can be solved by
   restructuring components. Pass children or render props instead of
   threading data through 5 levels.

4. **Context** — when you genuinely need data everywhere (auth, theme)
   and it changes rarely. Split state from dispatch if needed.

5. **External store (Zustand/Redux)** — when you need selectors,
   middleware, or high-frequency updates across distant components.

6. **Server state library (TanStack Query)** — for any data that comes
   from an API. This isn't really a choice — it's a different category
   of state entirely.

7. **URL state (searchParams)** — for state that should be shareable
   via URL: filters, pagination, sort order, active tab.

The mistake I see most often is jumping straight to Redux or Context for
everything. Most state is local. Start simple and add complexity only
when the simpler approach breaks down.
