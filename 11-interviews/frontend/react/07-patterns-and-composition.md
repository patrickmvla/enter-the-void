# React Patterns & Composition — Architecture Questions

## Component Composition

The most important React pattern. Prefer composition over props.

### The Children Pattern

```tsx
// Instead of configuring via props:
<Card title="Hello" body="World" footer="Done" />

// Compose with children:
<Card>
  <CardHeader>Hello</CardHeader>
  <CardBody>World</CardBody>
  <CardFooter>Done</CardFooter>
</Card>
```

### Render Props

Pass a function as a prop that returns JSX:

```tsx
function MouseTracker({ render }) {
  const [pos, setPos] = useState({ x: 0, y: 0 });

  return (
    <div onMouseMove={e => setPos({ x: e.clientX, y: e.clientY })}>
      {render(pos)}
    </div>
  );
}

// Usage:
<MouseTracker render={({ x, y }) => <p>Mouse: {x}, {y}</p>} />
```

Largely replaced by custom hooks, but still useful for inversion
of control in component libraries.

---

## Higher-Order Components (HOCs)

A function that takes a component and returns an enhanced component:

```tsx
function withAuth(Component) {
  return function AuthenticatedComponent(props) {
    const user = useAuth();
    if (!user) return <Navigate to="/login" />;
    return <Component {...props} user={user} />;
  };
}

const ProtectedDashboard = withAuth(Dashboard);
```

**Problems with HOCs:**
- Wrapper hell (multiple HOCs stack wrappers)
- Props collision (multiple HOCs may inject same prop name)
- Hard to type in TypeScript
- Obscure where props come from

**Modern alternative:** Custom hooks + composition.

---

## Custom Hooks

Extract reusable stateful logic:

```tsx
function useDebounce<T>(value: T, delay: number): T {
  const [debouncedValue, setDebouncedValue] = useState(value);

  useEffect(() => {
    const timer = setTimeout(() => setDebouncedValue(value), delay);
    return () => clearTimeout(timer);
  }, [value, delay]);

  return debouncedValue;
}

// Usage:
function Search() {
  const [query, setQuery] = useState('');
  const debouncedQuery = useDebounce(query, 300);

  useEffect(() => {
    if (debouncedQuery) searchAPI(debouncedQuery);
  }, [debouncedQuery]);
}
```

### Common custom hooks interviewers expect you to know:

```
useFetch         — data fetching with loading/error states
useDebounce      — debounce a value
useLocalStorage  — sync state with localStorage
useMediaQuery    — responsive breakpoint detection
useOnClickOutside — detect clicks outside a ref
usePrevious      — access previous render's value
useInterval      — declarative setInterval
useIntersectionObserver — lazy loading / infinite scroll trigger
```

---

## Compound Components

Components that work together, sharing implicit state:

```tsx
function Tabs({ children, defaultTab }) {
  const [activeTab, setActiveTab] = useState(defaultTab);
  return (
    <TabsContext.Provider value={{ activeTab, setActiveTab }}>
      {children}
    </TabsContext.Provider>
  );
}

function TabList({ children }) {
  return <div role="tablist">{children}</div>;
}

function Tab({ id, children }) {
  const { activeTab, setActiveTab } = useContext(TabsContext);
  return (
    <button
      role="tab"
      aria-selected={activeTab === id}
      onClick={() => setActiveTab(id)}
    >
      {children}
    </button>
  );
}

function TabPanel({ id, children }) {
  const { activeTab } = useContext(TabsContext);
  if (activeTab !== id) return null;
  return <div role="tabpanel">{children}</div>;
}

// Usage — clean, declarative API:
<Tabs defaultTab="one">
  <TabList>
    <Tab id="one">Tab 1</Tab>
    <Tab id="two">Tab 2</Tab>
  </TabList>
  <TabPanel id="one">Content 1</TabPanel>
  <TabPanel id="two">Content 2</TabPanel>
</Tabs>
```

---

## Controlled vs Uncontrolled Components

### Controlled — React owns the value

```tsx
function ControlledInput() {
  const [value, setValue] = useState('');
  return (
    <input
      value={value}
      onChange={e => setValue(e.target.value)}
    />
  );
}
```

Every keystroke: onChange → setState → re-render → input shows new value.

### Uncontrolled — DOM owns the value

```tsx
function UncontrolledInput() {
  const inputRef = useRef(null);

  function handleSubmit() {
    console.log(inputRef.current.value);
  }

  return <input ref={inputRef} defaultValue="" />;
}
```

React doesn't track the value. Read it on demand via ref.

**When to use which:**
- Controlled: form validation, conditional disabling, formatted inputs
- Uncontrolled: simple forms, file inputs, integrating non-React code

---

## Error Boundaries

Catch JavaScript errors in the component tree:

```tsx
class ErrorBoundary extends React.Component {
  state = { hasError: false, error: null };

  static getDerivedStateFromError(error) {
    return { hasError: true, error };
  }

  componentDidCatch(error, errorInfo) {
    logErrorToService(error, errorInfo.componentStack);
  }

  render() {
    if (this.state.hasError) {
      return <ErrorFallback error={this.state.error} />;
    }
    return this.props.children;
  }
}

// Usage:
<ErrorBoundary>
  <RiskyComponent />
</ErrorBoundary>
```

**What they catch:** Render errors, lifecycle errors, constructor errors.
**What they don't catch:** Event handlers, async code, SSR, errors in
the boundary itself.

In Next.js App Router: `error.tsx` files act as error boundaries.

---

## Portals

Render into a DOM node outside the parent hierarchy:

```tsx
import { createPortal } from 'react-dom';

function Modal({ children, isOpen }) {
  if (!isOpen) return null;

  return createPortal(
    <div className="modal-overlay">
      <div className="modal-content">{children}</div>
    </div>,
    document.getElementById('modal-root')
  );
}
```

The component renders into `#modal-root` but **events still bubble
through the React tree** (not the DOM tree). A click inside the portal
still triggers handlers on React ancestors.

---

## Suspense

Declarative loading states:

```tsx
<Suspense fallback={<Skeleton />}>
  <AsyncComponent />
</Suspense>
```

What can "suspend":
- `React.lazy()` — code splitting
- `use()` — reading promises (React 19)
- Data fetching libraries that integrate with Suspense (TanStack Query, Relay)
- Server Components in Next.js

Suspense boundaries nest — the **nearest ancestor** catches the suspension:

```tsx
<Suspense fallback={<PageSkeleton />}>
  <Header />
  <Suspense fallback={<ContentSkeleton />}>
    <MainContent />     {/* suspends here → shows ContentSkeleton */}
  </Suspense>
  <Sidebar />
</Suspense>
```

---

## Strict Mode

```tsx
<React.StrictMode>
  <App />
</React.StrictMode>
```

In development only, Strict Mode:
1. **Double-renders** components (detects impure renders)
2. **Double-runs effects** (detects missing cleanup)
3. Warns about deprecated APIs

Does nothing in production. If your component breaks under Strict Mode,
it has a bug (usually a side effect in render or a missing cleanup).

---

## Interview Questions

---

**Q: What are compound components?**

**How to answer:** Define the pattern, explain how it works internally
(context), and give a real use case with code.

**Answer:** Compound components are a group of components that work
together and share implicit state through context. The parent component
manages the state, and children consume it — creating a clean,
declarative API for the consumer.

The classic example is a Tabs component:

```jsx
// Consumer API — clean and declarative:
<Tabs defaultTab="settings">
  <TabList>
    <Tab id="profile">Profile</Tab>
    <Tab id="settings">Settings</Tab>
  </TabList>
  <TabPanel id="profile">Profile content</TabPanel>
  <TabPanel id="settings">Settings content</TabPanel>
</Tabs>
```

Internally, `Tabs` creates a context with `activeTab` and `setActiveTab`.
`Tab` reads the context to highlight itself and handle clicks. `TabPanel`
reads it to show/hide content. The consumer never manages state manually.

**Why this pattern is good:**
- **Flexible** — consumers can reorder children, add wrappers, inject
  content between tabs and panels
- **Encapsulated** — state management is hidden inside the component group
- **Type-safe** — each sub-component has clear props
- **Testable** — you can test the compound component as a unit

Used extensively in UI libraries: Radix UI, Headless UI, Reach UI.

---

**Q: When would you use a portal?**

**How to answer:** Explain the DOM problem, how portals solve it, and
the event bubbling behavior (which surprises many interviewers).

**Answer:** Portals render a component's output into a **different DOM
node** than its parent, while keeping it in the **same React tree**.

**The problem they solve:** CSS stacking contexts. If a parent has
`overflow: hidden`, `position: relative`, or a `z-index`, a child modal
or tooltip gets clipped or trapped behind other elements:

```jsx
<div style={{ overflow: 'hidden' }}>
  <Tooltip>This gets cut off!</Tooltip>  {/* clipped by parent */}
</div>
```

**With a portal:** The tooltip renders into `document.body` (or a
dedicated `#portal-root`), escaping the parent's CSS constraints:

```jsx
function Tooltip({ children }) {
  return createPortal(
    <div className="tooltip">{children}</div>,
    document.body  // renders here in the DOM
  );
}
```

**Critical detail (interviewers test this):** Even though the portal
renders in a different DOM location, **React events still bubble through
the React tree, not the DOM tree**. So if you have:

```jsx
<div onClick={handleClick}>
  <Modal />  {/* portal renders in document.body */}
</div>
```

A click inside the Modal **will** trigger handleClick, because in the
React tree, Modal is still a child of the div. This is useful (parent
can catch events) but can also be surprising if you don't expect it.

---

**Q: What do error boundaries catch and not catch?**

**How to answer:** List both categories clearly, explain why async errors
aren't caught, and mention the modern alternatives.

**Answer:**

**What they catch:**
- Errors thrown during **rendering** (in the component function body)
- Errors in **lifecycle methods** (componentDidMount, etc.)
- Errors in **constructors** of child components

**What they DON'T catch:**
- **Event handlers** — errors in onClick, onChange, etc. These run outside
  the React rendering cycle. Use try/catch inside the handler.
- **Async code** — promises, setTimeout, requestAnimationFrame. The error
  is thrown after rendering is complete.
- **Server-side rendering** — error boundaries only work on the client.
- **Errors in the boundary itself** — if the boundary's render method
  throws, it can't catch its own error. The nearest parent boundary
  catches it instead.

```jsx
class ErrorBoundary extends React.Component {
  state = { hasError: false };
  static getDerivedStateFromError(error) {
    return { hasError: true };  // triggers fallback UI
  }
  componentDidCatch(error, info) {
    logToService(error, info.componentStack);  // send to Sentry etc.
  }
  render() {
    if (this.state.hasError) return <FallbackUI />;
    return this.props.children;
  }
}
```

**In Next.js App Router:** You don't write class components. Instead,
`error.tsx` files in each route segment act as error boundaries
automatically. They must be Client Components (`'use client'`) and
receive `error` and `reset` props.

---

**Q: Controlled vs uncontrolled — when to use each?**

**How to answer:** Define each, show the code difference, and give
specific scenarios where each shines.

**Answer:**

**Controlled component** — React owns the value. The input's value is
always driven by state, and every change goes through a handler:

```jsx
const [email, setEmail] = useState('');
<input value={email} onChange={e => setEmail(e.target.value)} />
```

Every keystroke: onChange fires → setState → re-render → input displays
new value. React is the "single source of truth."

**Uncontrolled component** — the DOM owns the value. React doesn't track
it. You read the value on demand via a ref:

```jsx
const emailRef = useRef(null);
<input ref={emailRef} defaultValue="" />
// Later: emailRef.current.value
```

**When to use controlled:**
- Real-time validation (show error as user types)
- Conditional disabling (disable submit until form is valid)
- Formatted inputs (phone number formatting, uppercase enforcement)
- Dependent fields (city dropdown changes when state changes)
- Any time you need to know the value during render

**When to use uncontrolled:**
- Simple forms where you only need values on submit
- File inputs (`<input type="file" />` — always uncontrolled)
- Integrating with non-React libraries that manage their own DOM
- Performance-critical forms with many fields (avoids re-render per keystroke)

**In practice:** Most forms should be controlled. Libraries like React
Hook Form use uncontrolled inputs internally for performance but expose
a controlled-like API.

---

**Q: Why does Strict Mode render twice?**

**How to answer:** Explain what it's detecting (impure renders) and why
that matters for concurrent features.

**Answer:**

**What most candidates say:** "Strict Mode renders twice to catch bugs."
This is correct but surface-level.

**What a senior candidate says:** Strict Mode enforces **purity** — the
property that React's entire concurrent rendering model depends on.

A pure component must return the same output given the same inputs, with
no external mutations. If you violate this:

```jsx
let count = 0;
function Counter() {
  count++;  // external mutation during render
  return <p>{count}</p>;
}
// Strict Mode: call 1 → count=1, call 2 → count=2. Different output. Bug.
```

**Why purity is non-negotiable for concurrent React:** React's Fiber
architecture can start rendering a component tree, **pause** it to handle
a higher-priority update (user click), and then **re-render from scratch**
or **resume**. If your render mutated external state, that mutation would
happen twice — once in the abandoned render, once in the real render.
API calls, analytics events, DOM mutations — all would fire twice.

This isn't theoretical. `useTransition` and `useDeferredValue` routinely
cause React to start and abandon renders. If your component has side
effects in the render body, transitions will produce **double
side-effects in production**. Strict Mode catches this in development.

Strict Mode also **double-mounts effects** to detect missing cleanup:
```jsx
useEffect(() => {
  const ws = new WebSocket(url);  // opens
  return () => ws.close();         // must close
}, []);
// Strict Mode: mount → cleanup → mount. If your cleanup doesn't
// properly tear down, you'll have two WebSocket connections.
```

**The production consequence:** If your app breaks in Strict Mode, it
**will** break in production — just intermittently and harder to debug.
The double-render/double-effect in Strict Mode simulates what concurrent
features do at unpredictable times. Fix Strict Mode warnings immediately;
they're not false positives.

---

**Q: What replaced HOCs and render props?**

**How to answer:** Explain the problems with HOCs/render props, then
show how custom hooks solve them cleanly.

**Answer:** **Custom hooks** replaced both patterns as the primary way to
share stateful logic between components.

**Problems with HOCs:**
```jsx
// 3 HOCs = 3 wrapper components, prop collision risk, unclear data source
export default withAuth(withTheme(withRouter(MyComponent)));
// Where does `user` come from? withAuth? Hard to tell.
// What if withAuth and withTheme both inject a `loading` prop? Collision.
```

**Problems with render props:**
```jsx
// Deeply nested, hard to read, "callback hell" of JSX
<DataFetcher render={data => (
  <ThemeConsumer render={theme => (
    <AuthContext.Consumer>{user => (
      <MyComponent data={data} theme={theme} user={user} />
    )}</AuthContext.Consumer>
  )} />
)} />
```

**Custom hooks solve all of this:**
```jsx
function MyComponent() {
  const { user } = useAuth();         // clear where user comes from
  const theme = useTheme();           // no wrapper components
  const { data } = useFetch('/api');  // no prop collision
  // Everything is explicit, composable, and easy to type
}
```

Custom hooks are just functions — they compose naturally, have explicit
return values (no prop injection magic), are easy to type with TypeScript,
and add zero wrapper components to the React tree. They also work with
React DevTools, showing the hook values directly on the component.

**The nuance most candidates miss:** Custom hooks are NOT a perfect
replacement. Hooks can't provide **inversion of control** the way render
props can. A render prop lets the child component decide *what to render*
with the data — the hook consumer always controls rendering.

This matters for library design: headless UI libraries (Downshift,
React Aria) sometimes use render props or compound components precisely
because they need to inject arbitrary JSX into the consumer's render tree.
A hook can give you data and handlers, but it can't wrap your JSX or
inject elements.

**When to still use each pattern:**
- **Custom hooks:** 95% of shared logic (data fetching, subscriptions, form state)
- **Compound components:** Complex UI with implicit parent-child relationships (Tab/TabPanel, Accordion)
- **Render props:** When the library needs to control layout while the consumer controls content
- **HOCs:** Almost never in modern React. Legacy codebases only.
