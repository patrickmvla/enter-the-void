# React 19 & Modern APIs — What's New and What They'll Ask

## React 19 Overview

React 19 is the biggest shift since hooks. The theme: **server-first,
forms-first, less boilerplate**.

```
New in React 19:
  Actions          — async functions in transitions
  useActionState   — form state + pending + last result
  useFormStatus    — read parent form status from children
  useOptimistic    — instant UI feedback before server confirms
  use()            — read promises and context in render
  ref as prop      — no more forwardRef
  <Context>        — Context as provider directly
  Metadata support — <title>, <meta> in components
  Stylesheet support — automatic ordering and dedup
  Preloading APIs  — prefetchDNS, preconnect, preload, preinit
```

---

## Actions

An "action" is any async function used in a transition. React manages
the pending state, errors, and optimistic updates automatically.

```tsx
// Before React 19:
function UpdateName() {
  const [name, setName] = useState('');
  const [error, setError] = useState(null);
  const [isPending, setIsPending] = useState(false);

  const handleSubmit = async () => {
    setIsPending(true);
    const error = await updateName(name);
    setIsPending(false);
    if (error) {
      setError(error);
      return;
    }
    redirect('/profile');
  };

  return (/* ... */);
}

// React 19 with Actions:
function UpdateName() {
  const [name, setName] = useState('');
  const [error, setError] = useState(null);
  const [isPending, startTransition] = useTransition();

  const handleSubmit = () => {
    startTransition(async () => {
      const error = await updateName(name);
      if (error) {
        setError(error);
        return;
      }
      redirect('/profile');
    });
  };

  return (/* ... */);
}
```

React handles `isPending` automatically. Error boundaries catch
unhandled errors. No manual try/catch/finally boilerplate.

---

## useActionState

Combines form action + state + pending into one hook:

```tsx
import { useActionState } from 'react';

async function increment(previousState, formData) {
  return previousState + 1;
}

function Counter() {
  const [count, formAction, isPending] = useActionState(increment, 0);

  return (
    <form action={formAction}>
      <p>Count: {count}</p>
      <button disabled={isPending}>
        {isPending ? 'Updating...' : 'Increment'}
      </button>
    </form>
  );
}
```

**Signature:** `useActionState(action, initialState, permalink?)`
- `action`: async function `(prevState, formData) => newState`
- Returns `[state, wrappedAction, isPending]`

Works with Server Actions in Next.js — the action runs on the server,
state is returned to the client.

---

## useFormStatus

Read the status of a parent `<form>` from any child component:

```tsx
import { useFormStatus } from 'react-dom';

function SubmitButton() {
  const { pending, data, method, action } = useFormStatus();

  return (
    <button type="submit" disabled={pending}>
      {pending ? 'Submitting...' : 'Submit'}
    </button>
  );
}

function MyForm() {
  return (
    <form action={serverAction}>
      <input name="email" />
      <SubmitButton />  {/* reads form status without prop drilling */}
    </form>
  );
}
```

**Key rule:** `useFormStatus` must be called from a component that is
a **child** of a `<form>`. It reads the parent form's status.

---

## useOptimistic

Show immediate feedback before an async action completes:

```tsx
import { useOptimistic } from 'react';

function Messages({ messages, sendMessage }) {
  const [optimisticMessages, addOptimistic] = useOptimistic(
    messages,
    (currentMessages, newMessage) => [
      ...currentMessages,
      { text: newMessage, sending: true },
    ]
  );

  async function handleSend(formData) {
    const text = formData.get('message');
    addOptimistic(text);           // instantly shows in UI
    await sendMessage(text);       // actual server request
    // on success: messages prop updates, optimistic state resolves
    // on error: React reverts to the real messages
  }

  return (
    <div>
      {optimisticMessages.map((msg, i) => (
        <p key={i} style={{ opacity: msg.sending ? 0.5 : 1 }}>
          {msg.text}
        </p>
      ))}
      <form action={handleSend}>
        <input name="message" />
      </form>
    </div>
  );
}
```

---

## The `use` API

Read promises and context during render — not a hook (no rules of hooks):

```tsx
import { use, Suspense } from 'react';

// Reading a promise
function Comments({ commentsPromise }) {
  const comments = use(commentsPromise);  // suspends until resolved
  return comments.map(c => <p key={c.id}>{c.text}</p>);
}

function Page({ commentsPromise }) {
  return (
    <Suspense fallback={<Spinner />}>
      <Comments commentsPromise={commentsPromise} />
    </Suspense>
  );
}

// Reading context conditionally (can't do this with useContext!)
function ThemeText({ show }) {
  if (show) {
    const theme = use(ThemeContext);  // called conditionally — legal!
    return <p style={{ color: theme.color }}>Themed text</p>;
  }
  return <p>No theme</p>;
}
```

`use` can be called inside loops and conditionals. It's not bound by
the rules of hooks because it's not a hook — it's a new API category.

---

## ref as a Prop (No More forwardRef)

```tsx
// React 18: needed forwardRef wrapper
const Input = forwardRef((props, ref) => {
  return <input ref={ref} {...props} />;
});

// React 19: ref is just a prop
function Input({ ref, ...props }) {
  return <input ref={ref} {...props} />;
}
```

`forwardRef` will be deprecated in a future version.

---

## Context as Provider

```tsx
// React 18:
const ThemeContext = createContext('light');

<ThemeContext.Provider value="dark">
  {children}
</ThemeContext.Provider>

// React 19: Context IS the provider
<ThemeContext value="dark">
  {children}
</ThemeContext>
```

`<Context.Provider>` still works but will be deprecated.

---

## Document Metadata in Components

```tsx
function BlogPost({ post }) {
  return (
    <article>
      <title>{post.title}</title>
      <meta name="description" content={post.excerpt} />
      <link rel="canonical" href={post.url} />
      <h1>{post.title}</h1>
      <p>{post.content}</p>
    </article>
  );
}
```

React 19 hoists `<title>`, `<meta>`, and `<link>` to the `<head>`
automatically. No more `react-helmet` or `next/head` for simple cases.

---

## Improved Error Handling

### Better hydration errors

React 19 shows a single diff of the mismatch instead of multiple
confusing console errors:

```
Uncaught Error: Hydration failed because the server rendered HTML
didn't match the client.

<App>
  <Header>
-   <p>Server text</p>
+   <p>Client text</p>
```

### onCaughtError / onUncaughtError

```tsx
createRoot(document.getElementById('root'), {
  onCaughtError: (error) => {
    // errors caught by error boundaries
    trackError(error);
  },
  onUncaughtError: (error) => {
    // errors NOT caught by any error boundary
    showGlobalErrorUI(error);
  },
  onRecoverableError: (error) => {
    // errors React auto-recovers from (hydration mismatches)
    logWarning(error);
  },
});
```

---

## React Compiler (formerly React Forget)

An **opt-in compiler** that automatically memoizes components and hooks:

```tsx
// You write:
function ProductList({ products, filter }) {
  const filtered = products.filter(p => p.category === filter);
  return filtered.map(p => <ProductCard key={p.id} product={p} />);
}

// Compiler outputs (conceptually):
function ProductList({ products, filter }) {
  const filtered = useMemo(
    () => products.filter(p => p.category === filter),
    [products, filter]
  );
  // Also auto-memoizes the callback, stabilizes JSX references, etc.
}
```

The compiler understands React's rules and automatically adds
`useMemo`, `useCallback`, and `memo` where beneficial. You can
remove manual memoization.

---

## Interview Questions

---

**Q: What are Actions in React 19?**

**How to answer:** Explain the pain point (manual pending/error state),
then show how Actions eliminate it.

**Answer:** Actions are async functions used inside transitions. Before
React 19, handling an async operation (like a form submission) required
managing isPending, error, and success states manually with multiple
useState calls. Actions let React handle this automatically.

```jsx
// Before (manual):
const [isPending, setIsPending] = useState(false);
const [error, setError] = useState(null);
const handleSubmit = async () => {
  setIsPending(true);
  try {
    await submitForm(data);
  } catch (e) {
    setError(e);
  } finally {
    setIsPending(false);
  }
};

// After (React 19 Action):
const [isPending, startTransition] = useTransition();
const handleSubmit = () => {
  startTransition(async () => {
    const error = await submitForm(data);
    if (error) setError(error);  // React handles isPending automatically
  });
};
```

React 19 allows async functions inside `startTransition`. React sets
isPending to true before the action starts and false when it completes.
Error boundaries catch unhandled errors. The pending state is managed
for you, reducing boilerplate significantly.

---

**Q: What's the difference between useActionState and useFormStatus?**

**How to answer:** Define each hook's purpose and show where each is
used in a form architecture.

**Answer:** They solve different parts of the form problem:

**`useActionState`** manages the **full lifecycle** of a form action in
the component that owns the form. It gives you:
- The last result returned by the action (success message, validation errors)
- A wrapped action function to pass to `<form action={...}>`
- An isPending boolean

```jsx
const [state, formAction, isPending] = useActionState(serverAction, initialState);
// state = { error: 'Title is required' } (returned from last submission)
// formAction = wrapped version of serverAction for the form
// isPending = true while submitting
```

**`useFormStatus`** lets **child components** read the parent form's
submission status without prop drilling. It must be called from a component
that is a child of a `<form>`:

```jsx
function SubmitButton() {
  const { pending } = useFormStatus();  // reads parent <form>'s status
  return <button disabled={pending}>{pending ? 'Saving...' : 'Save'}</button>;
}
```

**In practice they work together:** useActionState in the form owner to
manage the action and its result, useFormStatus in reusable child
components (like a generic SubmitButton) to show loading state.

---

**Q: How does useOptimistic work?**

**How to answer:** Explain the UX motivation (instant feedback), the
mechanism, and what happens on failure.

**Answer:** useOptimistic provides **instant UI feedback** before an
async operation completes. The user sees their action reflected
immediately, making the app feel faster.

```jsx
const [optimisticMessages, addOptimistic] = useOptimistic(
  messages,                             // current real state
  (current, newMsg) => [...current, newMsg]  // how to apply optimistic update
);
```

**The flow:**

1. User submits a message
2. `addOptimistic({ text: 'Hello', sending: true })` — UI instantly
   shows the message (at 50% opacity to indicate it's pending)
3. Server Action runs in the background (`await sendMessage(...)`)
4. **On success:** the `messages` prop updates from the server, replacing
   the optimistic state with real data. The message goes to full opacity.
5. **On failure:** React **automatically reverts** to the previous real
   state. The optimistic message disappears, and you can show an error.

The key insight is that `useOptimistic` maintains two layers: the **real
state** (from props/server) and the **optimistic state** (applied on
top). When the real state updates, the optimistic layer resolves.

**What most people miss — how it differs from TanStack Query optimistic
updates:** TanStack Query's `onMutate` gives you a manual rollback
mechanism — you save the old cache, update it, and restore on error.
React's `useOptimistic` does this automatically: the optimistic layer is
ephemeral and always resolves to the real state. You don't manage rollback.

**The production gotcha:** Optimistic updates can create **visual
inconsistency** in high-latency networks. If the user adds 3 items
quickly, all 3 appear instantly (optimistic). If the server processes
them slowly, the items might "flash" — disappearing briefly when the
real state updates with only 1 or 2 items, then reappearing. This is
the cost of optimism: the UX during partial server catches is awkward.
For critical operations (payments, deletions), don't use optimistic
updates — show a loading state instead.

---

**Q: How is `use()` different from `useContext()`?**

**How to answer:** Highlight the two key differences (conditional calling
and promise support) and why they matter.

**Answer:** `use()` is a new React 19 API that's fundamentally different
from hooks — it doesn't follow the rules of hooks:

**Difference 1: Conditional calling.** `useContext` must be called at
the top level — never inside if/else or loops. `use()` can be called
anywhere:

```jsx
function Component({ showTheme }) {
  // useContext(ThemeContext);  ← must be called even if showTheme is false

  if (showTheme) {
    const theme = use(ThemeContext);  // ← legal! only reads when needed
    return <p style={{ color: theme.color }}>Themed</p>;
  }
  return <p>No theme</p>;
}
```

**Difference 2: Promise support.** `use()` can read promises, causing
the component to **suspend** until the promise resolves:

```jsx
function Comments({ commentsPromise }) {
  const comments = use(commentsPromise);  // suspends until resolved
  return comments.map(c => <p key={c.id}>{c.text}</p>);
}

// Must be wrapped in Suspense:
<Suspense fallback={<Spinner />}>
  <Comments commentsPromise={fetchComments()} />
</Suspense>
```

`use()` is not a hook — it's a new category of API. It works because
React can track suspensions at the framework level regardless of call
order. This makes it more flexible for patterns where you conditionally
need context or want to unwrap promises inline.

---

**Q: What is the React Compiler?**

**How to answer:** Explain what it does, why it exists, and what changes
for developers.

**Answer:** The React Compiler (formerly "React Forget") is an **opt-in
build-time tool** that analyzes your React code and automatically inserts
memoization — `useMemo`, `useCallback`, and `React.memo` — wherever it
determines they'd be beneficial.

**Why it exists:** Manual memoization is the #1 source of complexity in
React performance optimization. Developers either over-memoize (wasting
memory and adding code noise) or under-memoize (leaving performance
on the table). The compiler makes the optimal decision automatically.

**What it does:**
```jsx
// You write plain React:
function ProductList({ products, filter }) {
  const filtered = products.filter(p => p.category === filter);
  const handleClick = (id) => select(id);
  return filtered.map(p => <Card key={p.id} onClick={handleClick} />);
}

// Compiler output (conceptually):
// - filtered is memoized based on [products, filter]
// - handleClick is memoized based on [select]
// - JSX element references are stabilized
```

**What changes for developers:**
- You can **remove manual useMemo/useCallback** — the compiler handles it
- You write simpler, more readable code
- The compiler respects React's rules (purity, no side effects in render)
  so if your code violates them, the compiler will warn you
- It's opt-in per file or per project — you can adopt incrementally

---

**Q: What changed about forwardRef in React 19?**

**How to answer:** Show the before/after and explain why it's simpler.

**Answer:** In React 18 and earlier, function components couldn't receive
a `ref` prop directly. You had to wrap them with `forwardRef`:

```jsx
// React 18: forwardRef required
const Input = forwardRef<HTMLInputElement, InputProps>((props, ref) => {
  return <input ref={ref} {...props} />;
});
```

In React 19, **ref is just a regular prop**:

```jsx
// React 19: ref is a prop like any other
function Input({ ref, ...props }: InputProps & { ref?: React.Ref<HTMLInputElement> }) {
  return <input ref={ref} {...props} />;
}
```

This matters because:
- **Less boilerplate** — no `forwardRef` wrapper, no special typing
- **Better TypeScript DX** — ref is in the props interface, not a separate parameter
- **Simpler mental model** — ref is just a prop, nothing special about it
- **Easier debugging** — the component function has a real name in DevTools
  (forwardRef components often showed as "Anonymous")

`forwardRef` still works in React 19 for backwards compatibility but
will be deprecated in a future version. New code should use ref as a prop.
