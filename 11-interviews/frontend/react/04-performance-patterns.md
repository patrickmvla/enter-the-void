# React Performance Patterns — Avoiding Unnecessary Work

## The Rendering Cost Model

Every re-render means React:

```
1. Calls your component function
2. Diffs the returned elements against previous
3. If changes found → commits to DOM
```

Steps 1-2 happen even if the DOM doesn't change. For most components
this is fast. For large trees or expensive computations, it adds up.

---

## Pattern 1: Push State Down

The simplest optimization: move state into the component that uses it.

```jsx
// BAD: typing in the input re-renders the entire app
function App() {
  const [search, setSearch] = useState('');
  return (
    <div>
      <input value={search} onChange={e => setSearch(e.target.value)} />
      <ExpensiveTree />  {/* re-renders on every keystroke */}
    </div>
  );
}

// GOOD: state lives where it's used
function App() {
  return (
    <div>
      <SearchBar />
      <ExpensiveTree />  {/* never re-renders from search */}
    </div>
  );
}

function SearchBar() {
  const [search, setSearch] = useState('');
  return <input value={search} onChange={e => setSearch(e.target.value)} />;
}
```

---

## Pattern 2: Content as Children (Composition)

```jsx
// BAD: ScrollTracker re-renders everything when scroll changes
function App() {
  const [scroll, setScroll] = useState(0);
  return (
    <div onScroll={e => setScroll(e.target.scrollTop)}>
      <ScrollIndicator scroll={scroll} />
      <ExpensiveContent />  {/* re-renders on scroll! */}
    </div>
  );
}

// GOOD: children are created by the PARENT, so they're stable references
function App() {
  return (
    <ScrollTracker>
      <ExpensiveContent />  {/* same reference, no re-render */}
    </ScrollTracker>
  );
}

function ScrollTracker({ children }) {
  const [scroll, setScroll] = useState(0);
  return (
    <div onScroll={e => setScroll(e.target.scrollTop)}>
      <ScrollIndicator scroll={scroll} />
      {children}  {/* same React element object as before */}
    </div>
  );
}
```

Why this works: `children` is a React element created in the parent's
render. Since `App` doesn't re-render, the element reference is stable.
React sees the same object and skips re-rendering it.

---

## Pattern 3: React.memo

Wraps a component to skip re-renders when props haven't changed:

```jsx
const UserCard = React.memo(function UserCard({ user, onSelect }) {
  return (
    <div onClick={() => onSelect(user.id)}>
      <Avatar src={user.avatar} />
      <span>{user.name}</span>
    </div>
  );
});
```

**memo does a shallow comparison** of all props using `Object.is`.

For it to work, props must be **referentially stable**:

```jsx
function UserList({ users }) {
  // BAD: new function every render → memo is useless
  return users.map(u => (
    <UserCard key={u.id} user={u} onSelect={(id) => select(id)} />
  ));

  // GOOD: stable callback
  const handleSelect = useCallback((id) => select(id), []);
  return users.map(u => (
    <UserCard key={u.id} user={u} onSelect={handleSelect} />
  ));
}
```

### Custom comparison

```jsx
const UserCard = React.memo(UserCard, (prevProps, nextProps) => {
  // return true to SKIP re-render (props are "equal")
  return prevProps.user.id === nextProps.user.id;
});
```

---

## Pattern 4: Virtualization

For long lists, only render what's visible:

```jsx
import { useVirtualizer } from '@tanstack/react-virtual';

function VirtualList({ items }) {
  const parentRef = useRef(null);
  const virtualizer = useVirtualizer({
    count: items.length,
    getScrollElement: () => parentRef.current,
    estimateSize: () => 50,
  });

  return (
    <div ref={parentRef} style={{ height: '400px', overflow: 'auto' }}>
      <div style={{ height: virtualizer.getTotalSize() }}>
        {virtualizer.getVirtualItems().map(virtualRow => (
          <div
            key={virtualRow.key}
            style={{
              position: 'absolute',
              top: virtualRow.start,
              height: virtualRow.size,
            }}
          >
            {items[virtualRow.index].name}
          </div>
        ))}
      </div>
    </div>
  );
}
```

10,000 items? Only ~20 DOM nodes at any time.

---

## Pattern 5: Code Splitting with lazy()

```jsx
const Settings = lazy(() => import('./Settings'));

function App() {
  return (
    <Suspense fallback={<Spinner />}>
      <Routes>
        <Route path="/settings" element={<Settings />} />
      </Routes>
    </Suspense>
  );
}
```

The Settings chunk only loads when the user navigates to /settings.

---

## Pattern 6: Debouncing Expensive Updates

```jsx
function Search() {
  const [input, setInput] = useState('');
  const [deferredInput] = useDeferredValue(input);

  return (
    <>
      <input value={input} onChange={e => setInput(e.target.value)} />
      <SearchResults query={deferredInput} />
    </>
  );
}
```

`useDeferredValue` lets React prioritize the input update (urgent) over
the search results (can wait). The input stays responsive while results
update in the background.

### useTransition

```jsx
function TabContainer() {
  const [isPending, startTransition] = useTransition();
  const [tab, setTab] = useState('home');

  function selectTab(nextTab) {
    startTransition(() => {
      setTab(nextTab);  // marked as non-urgent
    });
  }

  return (
    <>
      <TabBar selectedTab={tab} onSelect={selectTab} />
      {isPending && <Spinner />}
      <TabContent tab={tab} />
    </>
  );
}
```

---

## Profiling: How to Find the Problem

### React DevTools Profiler

1. Open React DevTools → Profiler tab
2. Click Record
3. Interact with your app
4. Stop recording
5. Analyze: which components re-rendered and why

The "Why did this render?" feature shows:
- Props changed
- State changed
- Parent re-rendered
- Context changed

### Key metrics

```
Core Web Vitals:
  LCP  (Largest Contentful Paint)  — < 2.5s
  INP  (Interaction to Next Paint) — < 200ms
  CLS  (Cumulative Layout Shift)   — < 0.1
```

---

## Anti-Patterns

### Creating components inside components

```jsx
// BAD: Inner remounts every render (new reference)
function Outer() {
  function Inner() { return <div>hi</div>; }
  return <Inner />;
}

// GOOD: defined outside
function Inner() { return <div>hi</div>; }
function Outer() { return <Inner />; }
```

### Spreading objects as props

```jsx
// BAD: new object every render
<Component style={{ color: 'red' }} />

// GOOD: stable reference
const style = { color: 'red' };  // outside component or useMemo
<Component style={style} />
```

---

## Interview Questions

---

**Q: How do you optimize a React app with performance issues?**

**How to answer:** Show a systematic approach — measure first, then apply
targeted fixes. Don't say "add useMemo everywhere."

**Answer:** I follow a systematic process:

**Step 1: Measure.** Open React DevTools Profiler, record an interaction,
and identify which components re-render most and why. The "Why did this
render?" feature tells you: props changed, state changed, parent
re-rendered, or context changed. Without measurement, you're guessing.

**Step 2: Structural fixes (free, no extra code):**
- **Push state down** — if only the search bar needs `query`, don't put
  it in the parent that also renders a heavy list
- **Composition (children pattern)** — pass expensive content as children
  from a higher, stable component so it doesn't re-render when the
  wrapper's state changes

**Step 3: Memoization (targeted, based on profiling):**
- `React.memo` on components the profiler flagged as re-rendering
  unnecessarily
- `useCallback`/`useMemo` to stabilize props being passed to memo'd
  components

**Step 4: Reduce DOM work:**
- **Virtualize long lists** — react-virtual or TanStack Virtual. 10,000
  items in a list should only have ~20 DOM nodes at any time
- **Code-split heavy routes** — `React.lazy()` + Suspense so users don't
  download the settings page while viewing the dashboard

**Step 5: Concurrent features:**
- `useDeferredValue` for deferred rendering of expensive search results
- `useTransition` for non-urgent state updates (tab switches)

The key insight: most performance problems come from **too many
re-renders at the wrong level**. Structural fixes (steps 1-2) solve 80%
of problems. Memoization is a last resort, not a first instinct.

---

**Q: What's the difference between useMemo and React.memo?**

**How to answer:** Clarify they operate at different levels — one is
inside a component, the other wraps a component.

**Answer:** They're both memoization, but at different scales:

**`useMemo`** memoizes a **value** inside a component. It caches the
result of a computation and only recomputes when dependencies change:

```jsx
// Inside a component — memoizes the sorted array
const sorted = useMemo(() => items.sort(compareFn), [items]);
```

**`React.memo`** memoizes an **entire component**. It wraps a component
to skip its re-render entirely when props haven't changed (shallow ===):

```jsx
// Wraps a component — skips re-render if props are same
const UserCard = React.memo(({ user }) => <div>{user.name}</div>);
```

They work together: React.memo checks if props changed, and useMemo/
useCallback ensure that objects/functions passed as props maintain
stable references:

```jsx
// Parent:
const handleClick = useCallback(() => select(id), [id]);  // stable reference
const style = useMemo(() => ({ color: theme }), [theme]);  // stable reference
<MemoizedChild onClick={handleClick} style={style} />      // memo works!
```

Without useMemo/useCallback, the parent creates new objects each render,
so React.memo sees different props and re-renders anyway.

**The killer insight most candidates miss:** React.memo is a **code smell**,
not a best practice. If you find yourself wrapping a component in memo +
all its props in useCallback/useMemo, you have a **structural problem**.
The children pattern or pushing state down would eliminate the re-render
entirely, without any memoization overhead.

```jsx
// BAD: memo + useCallback + useMemo (3 APIs to prevent 1 re-render)
const Child = memo(({ data, onClick }) => ...);
const parent = () => {
  const data = useMemo(() => compute(), [dep]);
  const onClick = useCallback(() => handle(), [dep]);
  return <Child data={data} onClick={onClick} />;
};

// GOOD: composition (0 APIs, same result)
const Parent = ({ children }) => {
  const [localState, set] = useState(0);
  return <div onChange={set}>{children}</div>;  // children stable
};
```

Reach for composition first. Memo is the escape hatch when composition
doesn't fit — typically when the data genuinely changes and you need to
compare specific fields.

---

**Q: When would you use useDeferredValue vs useTransition?**

**How to answer:** Distinguish by whether you own the state update or
receive the value. Give a use case for each.

**Answer:** Both tell React "this update is low priority — keep the UI
responsive," but they're used in different situations:

**`useTransition`** — wrap a state update **you control**:
```jsx
const [isPending, startTransition] = useTransition();

function handleTabChange(tab) {
  startTransition(() => {
    setTab(tab);  // you own this setState call
  });
}
```
The tab state update is marked non-urgent. If the user clicks another
tab before the first finishes rendering, React abandons the first and
starts the second. `isPending` gives you a loading state.

**`useDeferredValue`** — defer a value **you receive** (e.g., from props):
```jsx
function SearchResults({ query }) {
  const deferredQuery = useDeferredValue(query);
  // query updates immediately (input stays responsive)
  // deferredQuery updates when React has time (results may lag behind)
  const results = useMemo(() => search(deferredQuery), [deferredQuery]);
}
```
You don't control when `query` changes (it comes from a parent), so you
can't wrap it in startTransition. Instead, you defer the value and let
React prioritize the input over the expensive search.

**Rule of thumb:** If you own the setState → useTransition. If you
receive the value → useDeferredValue.

---

**Q: Why is composition better than memo for preventing re-renders?**

**How to answer:** Explain the mechanism (stable element references) and
why it's "free" compared to memo's runtime cost.

**Answer:** Composition prevents re-renders **structurally** with zero
runtime cost. React.memo prevents them by **comparing props** every
render, which has a cost and can fail if props aren't stable.

Here's why composition works:

```jsx
function App() {
  return (
    <ScrollTracker>
      <ExpensiveContent />  {/* created here, in App */}
    </ScrollTracker>
  );
}

function ScrollTracker({ children }) {
  const [scroll, setScroll] = useState(0);
  return (
    <div onScroll={e => setScroll(e.target.scrollTop)}>
      {children}  {/* same React element object as last render */}
    </div>
  );
}
```

When ScrollTracker's scroll state changes, it re-renders. But `children`
is a React element created by **App**, not by ScrollTracker. Since App
didn't re-render, the element reference is exactly the same object
(`===`). React sees the same object and skips the subtree entirely.

With memo, you'd need:
```jsx
const ExpensiveContent = React.memo(({ data }) => { ... });
// Plus useCallback/useMemo for every prop to keep them stable
```

Composition is free, requires no extra API, and is impossible to break
with unstable props. It should be your first tool for performance — reach
for memo only when composition can't solve the problem (e.g., the data
genuinely changes and you need to compare specific props).
