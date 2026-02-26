# Vercel Engineering Best Practices — Performance Patterns That Win Interviews

Source: Vercel Engineering (January 2026), 45 rules across 8 categories.
Knowing these signals you understand production-grade React/Next.js.

---

## Priority 1: Eliminating Waterfalls (CRITICAL)

Waterfalls are the #1 performance killer. Each sequential await adds
full network round-trip latency.

### Defer await until needed

```tsx
// BAD: fetches even when not needed
async function handleRequest(userId: string, skip: boolean) {
  const data = await fetchUser(userId);  // always waits
  if (skip) return { skipped: true };    // data was wasted
  return process(data);
}

// GOOD: only fetch when the branch needs it
async function handleRequest(userId: string, skip: boolean) {
  if (skip) return { skipped: true };    // returns immediately
  const data = await fetchUser(userId);  // only fetches if needed
  return process(data);
}
```

### Promise.all for independent operations

```tsx
// BAD: sequential — 200ms + 300ms + 500ms = 1000ms
const user = await fetchUser();
const posts = await fetchPosts();
const comments = await fetchComments();

// GOOD: parallel — max(200, 300, 500) = 500ms
const [user, posts, comments] = await Promise.all([
  fetchUser(), fetchPosts(), fetchComments()
]);
```

### Start promises early, await late (API routes)

```tsx
// BAD: config waits for auth to finish
export async function GET() {
  const session = await auth();           // 100ms
  const config = await fetchConfig();     // 200ms (waits for auth)
  const data = await fetchData(session.user.id);
  return Response.json({ data, config });
}

// GOOD: auth and config start simultaneously
export async function GET() {
  const sessionPromise = auth();          // starts immediately
  const configPromise = fetchConfig();    // starts immediately
  const session = await sessionPromise;   // wait for auth
  const [config, data] = await Promise.all([
    configPromise,                        // may already be done
    fetchData(session.user.id)
  ]);
  return Response.json({ data, config });
}
```

### Strategic Suspense boundaries

```tsx
// BAD: entire page blocked by slow data
async function Page() {
  const data = await fetchSlowData();  // blocks everything
  return <Layout><Content data={data} /></Layout>;
}

// GOOD: shell renders instantly, data streams in
function Page() {
  return (
    <Layout>
      <Suspense fallback={<Skeleton />}>
        <Content />  {/* fetches own data, streams when ready */}
      </Suspense>
    </Layout>
  );
}
```

---

## Priority 2: Bundle Size (CRITICAL)

### Avoid barrel file imports

```tsx
// BAD: imports entire library (1,583 modules for lucide-react)
import { Check, X, Menu } from 'lucide-react';

// GOOD: import only what you use
import Check from 'lucide-react/dist/esm/icons/check';
import X from 'lucide-react/dist/esm/icons/x';

// ALSO GOOD: use Next.js optimizePackageImports
// next.config.js
module.exports = {
  experimental: {
    optimizePackageImports: ['lucide-react', '@mui/material']
  }
};
```

Libraries commonly affected: lucide-react, @mui/material, react-icons,
@radix-ui, lodash, date-fns.

### Dynamic imports for heavy components

```tsx
// BAD: Monaco (300KB) bundles with main chunk
import { MonacoEditor } from './monaco-editor';

// GOOD: loads on demand
import dynamic from 'next/dynamic';
const MonacoEditor = dynamic(
  () => import('./monaco-editor').then(m => m.MonacoEditor),
  { ssr: false }
);
```

### Preload on user intent

```tsx
function EditorButton({ onClick }) {
  const preload = () => void import('./monaco-editor');
  return (
    <button
      onMouseEnter={preload}  // starts loading on hover
      onFocus={preload}       // starts loading on focus
      onClick={onClick}
    >
      Open Editor
    </button>
  );
}
```

### Defer non-critical third-party libraries

```tsx
// Load analytics AFTER hydration, not blocking initial render
const Analytics = dynamic(
  () => import('@vercel/analytics/react').then(m => m.Analytics),
  { ssr: false }
);
```

---

## Priority 3: Server-Side Performance (HIGH)

### Authenticate Server Actions like API routes

Server Actions are public endpoints. Always validate + authenticate
inside the function, never rely on middleware alone:

```tsx
'use server';
export async function deleteUser(userId: string) {
  const validated = z.string().uuid().parse(userId);   // validate
  const session = await verifySession();                // authenticate
  if (session.user.role !== 'admin') throw new Error('Forbidden'); // authorize
  await db.user.delete({ where: { id: validated } });
}
```

### Minimize serialization at RSC boundaries

Only pass what the client needs — RSC serializes ALL props into HTML:

```tsx
// BAD: serializes all 50 user fields
<Profile user={user} />

// GOOD: serializes only what's rendered
<Profile name={user.name} avatar={user.avatar} />
```

### React.cache() for per-request deduplication

```tsx
import { cache } from 'react';

export const getCurrentUser = cache(async () => {
  const session = await auth();
  return db.user.findUnique({ where: { id: session.user.id } });
});

// Multiple components call getCurrentUser() → only ONE db query per request
```

Note: fetch() is auto-deduplicated in Next.js. Use React.cache() for
database queries, auth checks, and other non-fetch async work.

### Use after() for non-blocking operations

```tsx
import { after } from 'next/server';

export async function POST(request: Request) {
  await updateDatabase(request);

  // Logging happens AFTER response is sent — doesn't slow the user
  after(async () => {
    await logUserAction({ userAgent: request.headers.get('user-agent') });
  });

  return Response.json({ status: 'success' });
}
```

### Parallel data fetching with component composition

```tsx
// BAD: Sidebar waits for Page's fetch
async function Page() {
  const header = await fetchHeader();  // blocks Sidebar
  return <div>{header}<Sidebar /></div>;
}

// GOOD: both fetch simultaneously (no parent await blocking children)
function Page() {
  return <div><Header /><Sidebar /></div>;  // both async, both start immediately
}
async function Header() { const data = await fetchHeader(); return <div>{data}</div>; }
async function Sidebar() { const items = await fetchSidebarItems(); return <nav>...</nav>; }
```

---

## Priority 4: Re-render Optimization (MEDIUM)

### Defer state reads to usage point

```tsx
// BAD: component re-renders on every theme change even though it
// only uses theme in a callback
function Toolbar() {
  const theme = useTheme();  // subscribes to theme changes
  return <button onClick={() => applyTheme(theme)}>Apply</button>;
}

// GOOD: read inside the callback, not during render
function Toolbar() {
  const getTheme = useThemeGetter();  // no subscription
  return <button onClick={() => applyTheme(getTheme())}>Apply</button>;
}
```

### Functional setState for stable callbacks

```tsx
// BAD: new function every render (breaks memo on children)
const increment = () => setCount(count + 1);

// GOOD: stable reference, no dependency on count
const increment = useCallback(() => setCount(c => c + 1), []);
```

### Lazy state initialization

```tsx
// BAD: expensive() runs every render, result thrown away after first
const [data, setData] = useState(expensive());

// GOOD: only runs on mount
const [data, setData] = useState(() => expensive());
```

### Use transitions for non-urgent updates

```tsx
function Search() {
  const [query, setQuery] = useState('');
  const [isPending, startTransition] = useTransition();

  return (
    <input onChange={e => {
      setQuery(e.target.value);  // urgent: update input immediately
      startTransition(() => {
        filterResults(e.target.value);  // non-urgent: can be interrupted
      });
    }} />
  );
}
```

### Subscribe to derived state, not raw values

```tsx
// BAD: re-renders on every cart change (items added, removed, updated)
const cart = useCartStore(state => state.cart);
const isEmpty = cart.length === 0;

// GOOD: only re-renders when empty ↔ non-empty changes
const isEmpty = useCartStore(state => state.cart.length === 0);
```

---

## Priority 5: Rendering Performance (MEDIUM)

### CSS content-visibility for long lists

```css
/* Browser skips rendering off-screen items entirely */
.list-item {
  content-visibility: auto;
  contain-intrinsic-size: auto 200px;  /* estimated height */
}
```

### Hoist static JSX outside components

```tsx
// BAD: recreated every render
function App() {
  return <div><header><Logo /><Nav /></header><main>{children}</main></div>;
}

// GOOD: static parts created once, reused
const header = <header><Logo /><Nav /></header>;
function App() {
  return <div>{header}<main>{children}</main></div>;
}
```

### Prevent hydration mismatch without flicker

```tsx
// BAD: flash of wrong content
function ThemeProvider({ children }) {
  const [theme, setTheme] = useState('light');
  useEffect(() => setTheme(localStorage.getItem('theme') ?? 'light'), []);
  // Flash: renders light → useEffect → re-renders dark
}

// GOOD: inline script sets data attribute before React runs
<html>
  <head>
    <script dangerouslySetInnerHTML={{ __html: `
      document.documentElement.dataset.theme =
        localStorage.getItem('theme') ?? 'light'
    `}} />
  </head>
  <body>
    {children}  {/* CSS reads [data-theme] — no flash */}
  </body>
</html>
```

### Use ternary, not && for conditionals

```tsx
// BAD: renders "0" when count is 0
{count && <Badge count={count} />}

// GOOD: explicit null fallback
{count > 0 ? <Badge count={count} /> : null}
```

---

## Priority 6: JavaScript Performance (LOW-MEDIUM)

### Use Set/Map for O(1) lookups

```tsx
// BAD: O(n) on every check
const isSelected = (id) => selectedIds.includes(id);

// GOOD: O(1) lookup
const selectedSet = useMemo(() => new Set(selectedIds), [selectedIds]);
const isSelected = (id) => selectedSet.has(id);
```

### Build index maps for repeated lookups

```tsx
// BAD: O(n) find for each user
users.forEach(u => {
  const dept = departments.find(d => d.id === u.deptId);
});

// GOOD: O(1) lookup
const deptMap = new Map(departments.map(d => [d.id, d]));
users.forEach(u => {
  const dept = deptMap.get(u.deptId);
});
```

### Combine multiple array iterations

```tsx
// BAD: 3 passes over the array
const active = users.filter(u => u.active);
const names = active.map(u => u.name);
const sorted = names.sort();

// GOOD: single pass + sort
const names = [];
for (const u of users) {
  if (u.active) names.push(u.name);
}
names.sort();
```

---

## Interview Questions

---

**Q: What are the most critical performance optimizations in React/Next.js?**

**How to answer:** Prioritize by impact, show you think in terms of
user-facing metrics, not code aesthetics.

**Answer:** From Vercel's engineering guidelines, the priorities are:

1. **Eliminating waterfalls (CRITICAL)** — sequential async calls are the
   #1 performance killer. Each `await` adds full network latency. Fix with
   Promise.all, Suspense boundaries, and starting promises early (before
   you need the result). This can give 2-10x improvement.

2. **Bundle size (CRITICAL)** — directly affects Time to Interactive.
   Avoid barrel file imports (lucide-react barrel loads 1,583 modules),
   use dynamic imports for heavy components, defer third-party scripts
   after hydration. This affects LCP and TTI by hundreds of milliseconds.

3. **Server-side performance (HIGH)** — minimize RSC serialization (only
   pass fields the client uses), use React.cache() for deduplication,
   parallelize data fetching through component composition, and use
   after() for non-blocking side effects.

4. **Re-render optimization (MEDIUM)** — often overprioritized. Use
   functional setState, subscribe to derived values instead of raw state,
   use transitions for non-urgent updates. But measure before optimizing.

The key insight: waterfall elimination and bundle reduction give 10x
more impact than re-render optimization, but most developers focus on
re-renders first.

---

**Q: How do you eliminate request waterfalls in Next.js?**

**How to answer:** Give multiple techniques at different levels.

**Answer:** Waterfalls happen when async operations run sequentially
instead of in parallel. I fix them at every level:

**In API routes / Server Actions:** Start promises immediately, await
later. If `auth()` and `fetchConfig()` are independent, fire both before
awaiting either:
```tsx
const authPromise = auth();
const configPromise = fetchConfig();
const session = await authPromise;
const [config, data] = await Promise.all([configPromise, fetchData(session.id)]);
```

**In Server Components:** Don't await in parent components — it blocks
all children. Instead, make each component responsible for its own data:
```tsx
// Page doesn't await anything — Header and Sidebar fetch in parallel
function Page() {
  return <><Header /><Sidebar /></>;
}
```

**At the page level:** Use Suspense boundaries so slow data doesn't
block the shell. The layout, nav, and footer render instantly while the
data-dependent section streams in.

**Between client and server:** Use Server Components for data fetching
instead of client-side useEffect + fetch. This eliminates the
render → hydrate → fetch → render waterfall entirely.

---

**Q: How do you optimize bundle size in a Next.js app?**

**How to answer:** Give specific, measurable techniques.

**Answer:** Five techniques, ordered by impact:

1. **Push 'use client' to the leaves.** Only interactive components need
   to be Client Components. A page with a single Like button shouldn't
   ship its markdown parser and date library to the client.

2. **Avoid barrel file imports.** `import { Icon } from 'lucide-react'`
   loads all 1,583 icons. Import from `lucide-react/dist/esm/icons/icon`
   directly, or use Next.js `optimizePackageImports`.

3. **Dynamic import heavy components.** Monaco editor (300KB), chart
   libraries, rich text editors — use `next/dynamic` with `ssr: false`
   so they only load when rendered. Preload on hover for perceived speed.

4. **Defer third-party scripts.** Analytics, error tracking, chat widgets
   don't need to block initial render. Dynamic import them with
   `ssr: false` so they load after hydration.

5. **Analyze the bundle.** Run `ANALYZE=true next build` to see what's
   in each chunk. Look for surprises — often a single import pulls in an
   entire library.

---

**Q: What is RSC serialization and why does it matter?**

**How to answer:** Explain the mechanism and the optimization.

**Answer:** When a Server Component passes props to a Client Component,
React serializes those props into a special format (the RSC payload)
that's embedded in the HTML and sent to the browser. This serialization
happens at the server/client boundary — every prop is converted to a
string representation.

**Why it matters:** If you pass a 50-field user object to a Client
Component that only displays the name, all 50 fields are serialized,
sent over the network, and parsed on the client. This adds to page
weight and slows both server response time and client parsing.

**The fix:** Only pass what the client needs:
```tsx
// BAD: serializes entire user object
<ClientProfile user={user} />

// GOOD: serializes only what's rendered
<ClientProfile name={user.name} avatar={user.avatar} />
```

Also avoid creating duplicate references — `users.toSorted()` creates
a new array that can't be deduplicated by React's serialization. Do
sorting/filtering in the Client Component instead.

**What most people miss — the type restriction:** RSC serialization can
only handle JSON-compatible types. You **cannot** pass functions, class
instances, Maps, Sets, Dates (serialized as strings), or circular
references from a Server Component to a Client Component. This is why
you can't pass event handlers from server to client — it forces an
architectural boundary where all interactivity must originate in Client
Components.

```tsx
// BROKEN: function can't be serialized across the RSC boundary
<ClientComponent onClick={() => console.log('hi')} />

// WORKS: pass data, define the handler in the Client Component
<ClientComponent productId={product.id} />
// ClientComponent defines its own onClick internally
```

This constraint is **the** reason RSCs exist as a separate concept from
regular components — the serialization boundary is a fundamental
architecture decision, not a performance optimization.

---

**Q: When should you use React.cache() vs Next.js fetch memoization?**

**How to answer:** Distinguish the two mechanisms and their scopes.

**Answer:** They solve the same problem (deduplication) but for different
types of operations:

**Next.js fetch memoization** — automatically deduplicates `fetch()` calls
with the same URL and options within a single server render. If two
Server Components both `fetch('/api/user')`, only one network request
happens. No setup needed — it's built into Next.js's extended fetch.

**React.cache()** — wraps any async function for per-request deduplication.
Use it for everything that ISN'T a fetch call:
- Database queries (Prisma, Drizzle)
- Authentication checks
- File system reads
- Heavy computations

```tsx
export const getUser = cache(async (id: string) => {
  return db.user.findUnique({ where: { id } });
});
// Multiple components calling getUser('123') → one DB query
```

**Scope:** Both only work within a single request. For cross-request
caching, use an LRU cache (in-memory) or unstable_cache / external
caching (Redis).

---

**Q: What are common re-render optimization mistakes?**

**How to answer:** Show you know what NOT to do, which signals experience.

**Answer:** The most common mistakes:

1. **Memoizing everything.** Developers add useMemo/useCallback to every
   value and function "just in case." This adds memory overhead and
   comparison cost. Only memoize when profiling shows a problem.

2. **Subscribing to too much state.** Using `const cart = useStore(s => s.cart)`
   re-renders on every cart change. If you only need to know if it's empty,
   subscribe to the derived boolean: `const empty = useStore(s => s.cart.length === 0)`.

3. **Creating new objects in render.** `style={{ color: 'red' }}` creates
   a new object every render, breaking React.memo. Hoist static objects
   outside the component or useMemo them.

4. **Not using functional setState.** `setCount(count + 1)` captures
   `count` in a closure, requiring it as a dependency everywhere.
   `setCount(c => c + 1)` is dependency-free and creates a stable callback
   with useCallback.

5. **Ignoring composition.** Before reaching for memo, try structural
   solutions: push state down, use the children pattern, or split
   components. These prevent re-renders for free.
