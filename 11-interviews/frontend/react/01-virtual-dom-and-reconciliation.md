# Virtual DOM & Reconciliation — How React Actually Updates the Screen

## Why a Virtual DOM?

Direct DOM manipulation is expensive. Every time you touch the DOM —
change a class, update text, add a node — the browser may need to:

```
Mutate DOM → Recalculate styles → Layout → Paint → Composite
```

React's insight: batch changes in a lightweight JavaScript object tree
(the virtual DOM), diff it against the previous version, then apply the
**minimum set of real DOM mutations**.

```
State change
  │
  ▼
New virtual DOM tree (React elements)
  │
  ▼
Diff against previous virtual DOM tree  ← "reconciliation"
  │
  ▼
Minimal set of DOM mutations
  │
  ▼
Commit to real DOM
```

---

## What Is a React Element?

When you write JSX, it compiles to `React.createElement()` calls:

```jsx
<div className="card">
  <h1>Hello</h1>
  <p>World</p>
</div>
```

Becomes:

```js
{
  type: 'div',
  props: {
    className: 'card',
    children: [
      { type: 'h1', props: { children: 'Hello' } },
      { type: 'p',  props: { children: 'World' } }
    ]
  }
}
```

These are plain objects. They're cheap to create. React builds a full
tree of these every render — that's the virtual DOM.

---

## Fiber Architecture

React 16+ replaced the old stack reconciler with **Fiber**. Each React
element gets a corresponding Fiber node:

```
FiberNode {
  type          // 'div', MyComponent, etc.
  key           // used for list reconciliation
  stateNode     // reference to actual DOM node (or class instance)
  child         // first child fiber
  sibling       // next sibling fiber
  return        // parent fiber
  pendingProps  // new props from this render
  memoizedProps // props from last render
  memoizedState // state from last render
  flags         // what work needs to be done (Placement, Update, Deletion)
  lanes         // priority (part of concurrent features)
}
```

The fiber tree is a **linked list**, not a recursive tree. This is the
key insight — it lets React **pause, resume, and abort work**:

```
           App
          /
       Header
        /
     Logo ──→ Nav
                /
            Link1 ──→ Link2 ──→ Link3

Each node has: child (↓), sibling (→), return (↑)
```

---

## Reconciliation Algorithm (Diffing)

React's diffing algorithm is O(n) — not O(n³) like a general tree diff.
It achieves this through two heuristics:

### Heuristic 1: Different types produce different trees

```
Before:                After:
<div>                  <span>
  <Counter />            <Counter />
</div>                 </span>

React tears down the entire <div> subtree and rebuilds under <span>.
The Counter component unmounts and remounts (state is lost).
```

### Heuristic 2: Keys identify stable elements in lists

```
Before:              After:
<li key="a">A</li>   <li key="c">C</li>    ← new, inserted
<li key="b">B</li>   <li key="a">A</li>    ← moved
<li key="c">C</li>   <li key="b">B</li>    ← moved
```

Without keys, React diffs by index (position). With keys, React matches
by identity, so it knows to move nodes instead of recreating them.

**Why index as key is bad:**

```
Items: ['A', 'B', 'C'] → ['Z', 'A', 'B', 'C']

With key={index}:
  key=0: 'A' → 'Z'  (update text — but internal state stays!)
  key=1: 'B' → 'A'  (update text)
  key=2: 'C' → 'B'  (update text)
  key=3: null → 'C'  (create new)

  Every item re-renders. If items have local state (inputs, etc.),
  state gets mismatched to wrong items.

With key={item.id}:
  key='Z': create new
  key='A': move ✓
  key='B': move ✓
  key='C': move ✓

  Only the new item renders. State stays correct.
```

---

## The Two Phases

### Phase 1: Render (can be interrupted in concurrent mode)

React walks the fiber tree, calling your component functions:

```
beginWork(fiber)
  │
  ├─ Is this fiber's type a function? Call it.
  │  Get the new React elements returned.
  │
  ├─ Compare new children with old children (reconcileChildFibers)
  │  Mark fibers with flags: Placement, Update, Deletion
  │
  └─ Move to next fiber (child → sibling → return)
```

No DOM mutations happen here. This is pure computation.

### Phase 2: Commit (synchronous, cannot be interrupted)

```
1. Before mutation  — read DOM measurements (getSnapshotBeforeUpdate)
2. Mutation         — apply DOM changes (insertions, updates, deletions)
3. Layout           — run useLayoutEffect, update refs
```

After commit, `useEffect` callbacks are flushed asynchronously.

---

## What Triggers a Re-render?

A component re-renders when:

1. **setState is called** (or useState setter)
2. **Parent re-renders** (even if props haven't changed)
3. **Context value changes** (any consumer re-renders)
4. **forceUpdate** (class components)

When a component re-renders, **all its children re-render by default**.
This is where understanding bailout conditions matters.

---

## Bailout Conditions (When React Skips Work)

React can skip re-rendering a subtree if:

```
1. Same element type   AND
2. Same props (===)    AND
3. No pending state    AND
4. No pending context
```

`React.memo()` enables prop comparison for function components:

```jsx
const ExpensiveList = React.memo(({ items }) => {
  return items.map(item => <ListItem key={item.id} {...item} />);
});
```

Without `memo`, the component re-renders whenever its parent does,
even if `items` is the same reference.

**Gotcha:** If the parent creates new objects/arrays/functions each
render, `memo` won't help:

```jsx
// BAD: new array every render → memo is useless
<ExpensiveList items={data.filter(d => d.active)} />

// GOOD: stable reference via useMemo
const activeItems = useMemo(() => data.filter(d => d.active), [data]);
<ExpensiveList items={activeItems} />
```

---

## Interview Questions

---

**Q: What is the virtual DOM and why does React use it?**

**How to answer:** Start with the problem (DOM is slow), then the solution
(lightweight copy), then the mechanism (diffing + batching).

**Answer:**

**What most candidates say:** "The virtual DOM is a copy of the real DOM
in JavaScript. React diffs it to find changes and updates only what changed.
It's faster than the real DOM."

**What a senior candidate says:** The virtual DOM is a lightweight JavaScript
object tree (plain `{ type, props, children }` objects) that React rebuilds
on every state change by calling your component functions. It then diffs
the new tree against the previous one and applies the **minimum set of
real DOM mutations**.

But here's the insight most people miss: **the virtual DOM is not about
speed — it's about predictability.** Direct DOM manipulation with
`document.getElementById` is actually *faster* for a single targeted
update. The virtual DOM adds overhead (rebuild tree + diff + patch).

So why use it? Because **you trade raw performance for developer
ergonomics**. The cost of reconstructing a virtual tree and diffing it is
a small price for never having to manually orchestrate DOM updates. You
declare what the UI should look like, and React figures out the minimum
mutations to get there. This matters enormously when you have 50
developers on a team — imperative DOM code becomes unmaintainable.

```
State change → rebuild entire VDOM tree → diff old vs new → batch minimal DOM mutations
```

**The real performance win** isn't the diff itself — it's **batching**.
React collects all state changes in a frame, diffs once, and applies one
batch of DOM mutations. Without this, 10 independent `setState` calls
would each trigger layout recalculation: `mutate → layout → paint` × 10.
With React: `10 setState → 1 diff → 1 batch mutation → 1 layout → 1 paint`.

**Follow-up — "Is the virtual DOM always faster?"** No, and saying yes is
a red flag. For a single, targeted DOM update, `element.textContent = 'x'`
is faster. Frameworks like Svelte and SolidJS compile away the virtual DOM
entirely and achieve better raw performance. React's VDOM is a **design
trade-off**: worse micro-benchmark performance, better developer
experience and safer concurrent rendering (you can rebuild the virtual
tree multiple times without touching the DOM).

---

**Q: Why is React's diff O(n) instead of O(n³)?**

**How to answer:** Explain that a general tree diff is O(n³), then
describe the two heuristics React uses to shortcut to O(n).

**Answer:** A generic algorithm to diff two trees needs O(n³) comparisons
because it must consider moving any node to any position. React reduces
this to O(n) — a single pass over the tree — using two heuristics:

**Heuristic 1: Different types produce different trees.** If a `<div>`
becomes a `<span>`, React doesn't try to match children. It tears down
the entire `<div>` subtree and rebuilds from scratch under `<span>`. This
eliminates cross-type comparisons entirely.

**Heuristic 2: Keys identify stable elements in lists.** When React
encounters a list, it matches children by their `key` prop rather than
by position. This means if you prepend an item to a list of 1000, React
knows the other 1000 haven't changed — it just inserts the new one. Without
keys, it would diff by index and think every item changed.

```
Without keys (by index):       With keys (by identity):
  index 0: A → Z (update!)      key=Z: new → insert
  index 1: B → A (update!)      key=A: same → skip
  index 2: C → B (update!)      key=B: same → skip
  index 3: null → C (insert)    key=C: same → skip
  = 4 operations                 = 1 operation
```

---

**Q: What happens when a parent component re-renders?**

**How to answer:** State the default behavior, explain why, then explain
how to prevent it.

**Answer:**

**What most candidates say:** "Children re-render because React doesn't
do deep prop comparison. Use React.memo to fix it."

**What a senior candidate says:** When a parent re-renders, **all children
re-render by default**, regardless of whether props changed. This is a
deliberate design choice, not a flaw.

**Why React chose this default:** The alternative — automatically comparing
props before every render — has its own cost. For most components, the
render function is fast (it just returns JSX objects). Doing a shallow
comparison of every prop on every component would add overhead that exceeds
the cost of just re-rendering. React's philosophy: **render everything by
default, optimize selectively where it matters.**

This also enables a critical guarantee: **your component always reflects
current state.** If React tried to skip renders based on props alone, it
could miss context changes, ref updates, or effects that depend on render
count. The simple "re-render everything" rule is easy to reason about.

Every render creates new React element objects. `<Child name="Alice" />`
becomes `{ type: Child, props: { name: "Alice" } }` — a brand new object
every time. React can't know whether the output would be the same without
calling the function.

**How to prevent it (in order of preference):**

1. **Push state down** — move state into a smaller component so only
   that component re-renders. Free, zero extra API.

2. **Composition (children pattern)** — pass children from a higher level.
   Since the parent didn't re-render, the element reference is stable:
   ```jsx
   <ScrollTracker>
     <ExpensiveContent />  {/* same object reference — skipped */}
   </ScrollTracker>
   ```

3. **React.memo** — **last resort**, not first instinct. Only after
   profiling shows the re-render is actually expensive. And memo requires
   stable props (useCallback/useMemo), which adds complexity.

**The production insight:** Most re-renders are harmless. A component
re-rendering in <1ms is not a problem. Only optimize when the Profiler
shows a component taking >16ms (one frame budget) or when you can measure
a user-facing impact (dropped frames, slow interactions).

---

**Q: Why are keys important in lists?**

**How to answer:** Explain what keys do mechanically, then show the
consequences of bad keys with a concrete example.

**Answer:** Keys tell React which element in a list corresponds to which
element from the previous render. React uses keys to decide whether to
**update** an existing DOM node, **move** it, **create** a new one, or
**delete** an old one.

Without stable keys (or using `index` as key), React matches by position.
This causes two problems:

**Problem 1: Unnecessary work.** If you prepend an item, every item
shifts by one index. React thinks every item changed and re-renders all
of them.

**Problem 2: State corruption.** If list items have internal state (like
an input field), the state stays attached to the index, not the item. So
if you delete item 0, item 1's state moves to index 0 — now the wrong
item has the wrong state.

```jsx
// BAD — index as key
{items.map((item, i) => <Input key={i} />)}
// Delete first item → second item's input text appears in first slot

// GOOD — stable unique ID
{items.map(item => <Input key={item.id} />)}
// Delete first item → everything stays correct
```

**Rule:** Use a unique, stable identifier from your data (database ID,
slug, etc.). Only use index if the list is static, never reordered, and
items have no state.

---

**Q: What is React Fiber?**

**How to answer:** Explain the problem with the old reconciler, then
describe Fiber's structure and what it enables.

**Answer:** React Fiber is the reimplementation of React's core
reconciliation algorithm, shipped in React 16. The old "stack reconciler"
processed the entire tree synchronously — once it started rendering, it
couldn't stop until done. For large trees, this blocked the main thread
and caused janky UIs.

Fiber solves this by representing each component as a **fiber node** in
a **linked list** (not a recursive call stack). Each fiber has pointers
to its child, sibling, and parent:

```
          App
         ↓ child
       Header
      ↓ child
    Logo → Nav (sibling)
           ↓ child
         Link1 → Link2 → Link3 (siblings)
```

**Why a linked list and not a call stack?** This is the critical insight:
the old reconciler used recursive function calls. A call stack is
inherently synchronous — once you start, you can't pause until every
recursive call returns. If you have 10,000 components, that's 10,000
stack frames, and the main thread is blocked the entire time.

A linked list can be suspended at any node. React processes one fiber,
checks if the browser needs the main thread (using `shouldYield()`), and
if so, saves its position and yields. When the browser is free again, React
picks up from the exact fiber where it stopped. **This is impossible with
a recursive call stack.**

This enables:

- **Concurrent rendering** — React can work on multiple updates at
  different priorities simultaneously
- **Time slicing** — large renders are broken into 5ms chunks
- **Suspense** — components can "suspend" while waiting for data, and
  React tracks where to resume
- **Transitions** — a user click interrupts a background render by
  restarting from a higher-priority lane

Each fiber tracks flags (Placement, Update, Deletion) and **lanes**
(a bitmask priority system). When a high-priority update arrives mid-render,
React abandons the current work-in-progress tree and starts fresh with
the urgent update. The abandoned work is simply discarded — no DOM was
touched (render phase is pure), so there's nothing to clean up.

---

**Q: What's the difference between the render and commit phases?**

**How to answer:** Describe each phase, what happens in it, and the
critical difference in interruptibility.

**Answer:** React processes updates in two distinct phases:

**Render phase (can be interrupted):**
- React calls your component functions to produce new React elements
- It walks the fiber tree, comparing new elements against the previous
  tree (this is reconciliation/diffing)
- It marks fibers with flags: Placement (new), Update (changed),
  Deletion (removed)
- **No side effects happen here** — no DOM mutations, no effects fired
- In concurrent mode, this phase can be paused, aborted, or restarted

**Commit phase (synchronous, cannot be interrupted):**
- React applies all the accumulated DOM mutations in one batch
- It runs in three sub-phases:
  1. **Before mutation** — read DOM measurements (getSnapshotBeforeUpdate)
  2. **Mutation** — insert, update, delete DOM nodes
  3. **Layout** — run useLayoutEffect callbacks, update refs
- After commit completes, useEffect callbacks are scheduled (async)

The key insight: the render phase is **pure computation** — it just
figures out what changed. The commit phase is where **side effects**
happen. This separation is what makes concurrent features possible —
React can redo the render phase without the user seeing half-applied
DOM changes.
