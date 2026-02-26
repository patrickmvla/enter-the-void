# Hooks Deep Dive — How They Actually Work

## The Hooks Linked List

Hooks aren't magic. Each component's fiber has a `memoizedState` field
that points to a **linked list of hook nodes**:

```
FiberNode.memoizedState → Hook1 → Hook2 → Hook3 → null
                          (useState) (useEffect) (useMemo)
```

Each hook node:

```js
{
  memoizedState: any,    // the stored value
  queue: UpdateQueue,    // pending state updates (for useState)
  next: Hook | null      // next hook in the list
}
```

**This is why hooks must be called in the same order every render.**
React doesn't use names — it uses position in the linked list. If you
skip a hook conditionally, every subsequent hook reads the wrong node.

```jsx
// BROKEN — hook order changes between renders
if (loggedIn) {
  const [name, setName] = useState('');  // Hook #1 sometimes
}
const [count, setCount] = useState(0);   // Hook #1 or #2

// React walks the linked list by position.
// If loggedIn flips, count reads name's state.
```

---

## useState

### Mounting (first render)

```
1. Create hook node
2. Set memoizedState = initialState (or call initializer function)
3. Create update queue
4. Return [state, dispatchAction.bind(null, fiber, queue)]
```

### Updating

```jsx
setCount(prev => prev + 1);
```

```
1. Create update object { action: prev => prev + 1, next: ... }
2. Enqueue on the hook's queue
3. Schedule a re-render of this fiber
4. On next render, process all queued updates to compute new state
```

Multiple `setState` calls in the same event handler are **batched**:

```jsx
function handleClick() {
  setCount(c => c + 1);  // queued
  setName('Alice');       // queued
  // → one re-render, not two
}
```

React 18+: batching happens **everywhere** (promises, timeouts, native
event handlers), not just React event handlers.

### Lazy initialization

```jsx
// BAD — runs expensive() every render, result discarded after first
const [data, setData] = useState(expensive());

// GOOD — only runs on mount
const [data, setData] = useState(() => expensive());
```

---

## useEffect

### Mental model

`useEffect` is not a lifecycle method. It's a **synchronization mechanism**.
It synchronizes side effects with your component's state/props.

```
useEffect(() => {
  // effect: runs after paint
  return () => {
    // cleanup: runs before next effect or unmount
  };
}, [dependencies]);
```

### Execution timeline

```
Render phase:   Component function runs, returns JSX
Commit phase:   DOM mutations applied
Browser paint:  Screen updated
After paint:    useEffect callbacks fire (asynchronously)
```

```
Mount:     effect runs
Update:    if deps changed → cleanup of prev effect → new effect runs
Unmount:   cleanup of last effect runs
```

### Dependency array

```jsx
useEffect(() => { ... }, [a, b]);
```

React stores the previous deps. On each render, it does a shallow
comparison (Object.is) of each dependency:

```
prevDeps[0] === nextDeps[0]  &&
prevDeps[1] === nextDeps[1]
→ true: skip effect
→ false: run cleanup + effect
```

**No dependency array** → runs after every render.
**Empty array `[]`** → runs once on mount, cleanup on unmount.

### Common mistakes

```jsx
// BUG: stale closure
useEffect(() => {
  const id = setInterval(() => {
    setCount(count + 1);  // captures count from this render
  }, 1000);
  return () => clearInterval(id);
}, []);  // never re-runs, count is always 0

// FIX: use functional update
useEffect(() => {
  const id = setInterval(() => {
    setCount(c => c + 1);  // doesn't need count in closure
  }, 1000);
  return () => clearInterval(id);
}, []);
```

---

## useLayoutEffect

Same API as `useEffect` but fires **synchronously after DOM mutations,
before the browser paints**:

```
Render → Commit (DOM mutations) → useLayoutEffect → Browser paint → useEffect
```

Use it when you need to:
- Measure DOM elements before the user sees them
- Prevent visual flicker (e.g., tooltip positioning)
- Synchronously mutate the DOM before paint

```jsx
useLayoutEffect(() => {
  const { height } = ref.current.getBoundingClientRect();
  setTooltipPosition(calculatePosition(height));
}, []);
```

---

## useRef

A mutable container that persists across renders **without triggering
re-renders**:

```jsx
const ref = useRef(initialValue);
// ref = { current: initialValue }
```

Two primary uses:

### 1. DOM references

```jsx
const inputRef = useRef(null);
// <input ref={inputRef} />
// inputRef.current → the actual DOM node
```

### 2. Mutable instance variables

```jsx
const timerIdRef = useRef(null);

useEffect(() => {
  timerIdRef.current = setInterval(tick, 1000);
  return () => clearInterval(timerIdRef.current);
}, []);
```

Unlike state, mutating `.current` doesn't cause a re-render. Use it for
values you need to track but don't need to display.

---

## useMemo & useCallback

Both are **memoization** hooks. They cache computed values between renders.

### useMemo — memoize a value

```jsx
const sorted = useMemo(
  () => items.sort((a, b) => a.name.localeCompare(b.name)),
  [items]
);
```

Only recomputes when `items` changes (by reference).

### useCallback — memoize a function

```jsx
const handleClick = useCallback(
  () => setFilter(id),
  [id]
);
```

`useCallback(fn, deps)` is equivalent to `useMemo(() => fn, deps)`.

### When to actually use them

**Don't memoize everything.** Memoization has costs:
- Memory to store the cached value
- Comparison logic on every render
- Code complexity

Use them when:

```
1. Passing callbacks to memoized children (React.memo)
   Without useCallback, the new function reference breaks memo.

2. Expensive computations
   Sorting/filtering large lists, complex math.

3. Stable references for effect dependencies
   Prevents useEffect from re-running on every render.
```

Don't use them for:
- Simple calculations (a + b)
- Values that change every render anyway
- Components that aren't memoized (memo won't help downstream)

---

## useReducer

useState for complex state logic:

```jsx
function reducer(state, action) {
  switch (action.type) {
    case 'increment': return { count: state.count + 1 };
    case 'decrement': return { count: state.count - 1 };
    case 'reset':     return { count: 0 };
    default: throw new Error(`Unknown action: ${action.type}`);
  }
}

const [state, dispatch] = useReducer(reducer, { count: 0 });
```

Prefer useReducer when:
- Next state depends on previous state in complex ways
- Multiple sub-values are related
- You want to pass dispatch down instead of multiple callbacks
- State transitions are easier to reason about declaratively

---

## useContext

Reads from the nearest `<Context.Provider>` above in the tree:

```jsx
const ThemeContext = createContext('light');

function App() {
  return (
    <ThemeContext.Provider value="dark">
      <Toolbar />
    </ThemeContext.Provider>
  );
}

function Button() {
  const theme = useContext(ThemeContext);
  // theme === 'dark'
}
```

### The re-render problem

When a context value changes, **every component that calls useContext
for that context re-renders** — even if it only uses part of the value.

```jsx
// BAD: every consumer re-renders when ANY value changes
<AppContext.Provider value={{ user, theme, locale }}>

// BETTER: split into separate contexts
<UserContext.Provider value={user}>
  <ThemeContext.Provider value={theme}>
    <LocaleContext.Provider value={locale}>
```

---

## Interview Questions

---

**Q: Why must hooks be called in the same order?**

**How to answer:** Explain the internal data structure, then show what
breaks when the rule is violated.

**Answer:** React stores hooks as a **linked list** on each component's
fiber node. There are no names or IDs — React identifies each hook purely
by its **position** in the call sequence. On the first render, React
builds the list: hook #1 → hook #2 → hook #3. On re-renders, React walks
that same list in order, pairing each hook call with its stored state.

If you call a hook conditionally, the list shifts:

```jsx
// Render 1: loggedIn = true
useState('')     // hook #1 → stores name
useState(0)      // hook #2 → stores count

// Render 2: loggedIn = false, skip first hook
useState(0)      // hook #1 → reads name's slot! BUG
```

React now reads the wrong state for every subsequent hook. This is why
the ESLint rule `rules-of-hooks` exists — hooks must always be called at
the top level, never inside conditions, loops, or nested functions.

---

**Q: What's the difference between useEffect and useLayoutEffect?**

**How to answer:** Draw the timeline, explain when each fires, then give
a concrete use case for useLayoutEffect.

**Answer:** Both run after render, but at different points in the
browser's lifecycle:

```
Component renders → DOM mutations applied → useLayoutEffect → browser paints → useEffect
                                             (synchronous)                      (asynchronous)
```

**useEffect** fires **after the browser has painted**. This is the right
default — the user sees the UI immediately, and the effect runs in the
background. Use it for data fetching, subscriptions, logging, and most
side effects.

**useLayoutEffect** fires **synchronously after DOM mutations but before
the browser paints**. The browser waits for it to finish. Use it when
you need to:

- **Measure a DOM element** and position something based on the measurement
  (tooltip placement, dynamic sizing)
- **Prevent visual flicker** — if useEffect causes a visible jump (element
  renders in wrong position then snaps to correct position), switch to
  useLayoutEffect

```jsx
// Without useLayoutEffect: tooltip flickers (renders at 0,0 then jumps)
// With useLayoutEffect: tooltip appears in correct position immediately
useLayoutEffect(() => {
  const rect = ref.current.getBoundingClientRect();
  setPosition({ top: rect.bottom, left: rect.left });
}, []);
```

**Warning — this is a footgun:** useLayoutEffect **blocks paint**. If your
effect takes 50ms, the user stares at a frozen screen for 50ms. Your team
will debug this as "the page hangs sometimes" and never connect it to a
useLayoutEffect doing too much work. In production, I've seen
useLayoutEffect calls that measure 100+ elements cause visible jank.

**Rule:** 99% of the time, useEffect is correct. Only switch to
useLayoutEffect when you can point to a specific visual flicker that
useEffect causes. If you're unsure, use useEffect — the worst case is a
one-frame flicker, which is usually invisible.

---

**Q: When should you use useMemo/useCallback?**

**How to answer:** Give the three legitimate use cases, then explain when
NOT to use them (this shows maturity).

**Answer:** Both are memoization hooks — they cache a value between renders
so React can skip recomputation. But memoization isn't free (memory +
comparison cost), so use them strategically:

**Use useMemo when:**
1. **Expensive computation** — sorting/filtering a large dataset:
   ```jsx
   const sorted = useMemo(() => items.sort(compareFn), [items]);
   ```
2. **Stable reference for effect deps** — prevent useEffect from re-running:
   ```jsx
   const options = useMemo(() => ({ page, limit }), [page, limit]);
   useEffect(() => { fetch(url, options); }, [options]);
   ```

**Use useCallback when:**
3. **Passing callbacks to memoized children** — without it, React.memo
   can't prevent re-renders because the function reference changes:
   ```jsx
   const handleClick = useCallback(() => select(id), [id]);
   <MemoizedChild onClick={handleClick} />
   ```

**Don't use them for:**
- Simple calculations (`a + b`, `items.length`)
- Values that change every render anyway (memoization never hits cache)
- Components that aren't wrapped in React.memo (stable callback doesn't
  help if the child re-renders anyway)
- Premature optimization — profile first, optimize second

---

**Q: How does useState batching work?**

**How to answer:** Explain the old behavior, the React 18 change, and
what it means practically.

**Answer:** When you call multiple setState functions in the same
synchronous execution context, React **batches** them into a single
re-render instead of re-rendering after each call:

```jsx
function handleClick() {
  setCount(c => c + 1);   // queued, no re-render yet
  setName('Alice');         // queued, no re-render yet
  setAge(30);              // queued, no re-render yet
  // → ONE re-render with all three updates
}
```

**Before React 18:** Batching only worked inside React event handlers.
Updates in promises, setTimeout, or native event listeners triggered
separate re-renders for each setState call.

**React 18+ (automatic batching):** Batching works **everywhere** —
event handlers, promises, setTimeout, native events. This was a major
performance improvement.

```jsx
// React 18: these are batched (one re-render)
setTimeout(() => {
  setCount(1);
  setFlag(true);
}, 1000);

// If you NEED synchronous re-renders (rare), use flushSync:
import { flushSync } from 'react-dom';
flushSync(() => setCount(1));  // re-renders immediately
flushSync(() => setFlag(true)); // re-renders again
```

**Production gotcha most people miss:** State is NOT updated synchronously
after calling setState. This is the #1 source of confusion:

```jsx
function handleClick() {
  setCount(count + 1);
  console.log(count);  // Still the OLD value! Not count + 1.
  // The new value only exists in the next render.
}
```

If you need to act on the new value immediately, compute it first:
```jsx
function handleClick() {
  const newCount = count + 1;
  setCount(newCount);
  sendAnalytics(newCount);  // use the computed value, not state
}
```

**flushSync is a code smell:** If you're reaching for it, your code
probably has a structural issue. The only legitimate use case is forcing a
DOM update before reading a measurement (e.g., scroll position after
adding a list item). If you're using it to "make setState synchronous,"
you're fighting React's design.

---

**Q: What's the stale closure problem?**

**How to answer:** Explain what a closure captures, show a concrete bug,
then give the two fixes.

**Answer:** In JavaScript, a function "closes over" the variables in its
surrounding scope at the time it's created. In React, each render creates
a new closure with that render's state values. If a callback is created
in render #1 and never recreated, it permanently sees render #1's state.

```jsx
// BUG: count is always 0 inside the interval
useEffect(() => {
  const id = setInterval(() => {
    console.log(count);      // always 0 — captured from first render
    setCount(count + 1);     // always sets to 1
  }, 1000);
  return () => clearInterval(id);
}, []);  // empty deps → effect never re-runs → closure never refreshes
```

**Fix 1: Functional update** (preferred for state):
```jsx
setCount(prev => prev + 1);  // doesn't need count in the closure
```

**Fix 2: Add to dependency array:**
```jsx
useEffect(() => {
  const id = setInterval(() => {
    setCount(count + 1);     // sees current count
  }, 1000);
  return () => clearInterval(id);
}, [count]);  // re-runs interval every time count changes
```

Fix 1 is better here because Fix 2 creates and destroys a new interval
on every count change. The stale closure problem is one of the most
common hook bugs — interviewers love asking about it.

---

**Q: When would you use useReducer over useState?**

**How to answer:** Give concrete scenarios where useReducer wins, and
explain the dispatch stability advantage.

**Answer:** useReducer is preferable when:

1. **Complex state transitions** — next state depends on current state
   in non-trivial ways:
   ```jsx
   // useState: hard to follow, easy to miss a field
   setState(prev => ({ ...prev, loading: true, error: null, data: null }));

   // useReducer: clear, named transitions
   dispatch({ type: 'FETCH_START' });
   ```

2. **Multiple related values** — when updating one value requires
   updating others (loading + error + data all change together).

3. **Passing update logic to children** — `dispatch` is a **stable
   reference** (it never changes between renders), so you can pass it
   to child components without causing re-renders. With useState, each
   setter is stable, but if you need to pass a composed callback like
   `() => { setA(...); setB(...); }`, that's a new function each render.

4. **Testability** — the reducer is a pure function. You can test state
   transitions in isolation without rendering any components:
   ```jsx
   expect(reducer({ count: 0 }, { type: 'increment' }))
     .toEqual({ count: 1 });
   ```

**Rule of thumb:** If you have more than 3 related state updates that
change together, or you find yourself writing complex setState callbacks,
switch to useReducer.
