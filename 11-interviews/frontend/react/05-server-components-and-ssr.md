# Server Components & SSR — The Rendering Spectrum

## The Problem

Traditional React (client-side rendering):

```
Browser requests page
  │
  ▼
Server returns empty HTML + JS bundle
  │
  ▼
Browser downloads JS (could be 500KB+)
  │
  ▼
JS executes, renders DOM          ← user sees content HERE
  │
  ▼
React attaches event handlers     ← app is interactive HERE
```

User stares at a blank screen while JS downloads and executes.

---

## Server-Side Rendering (SSR)

Server renders HTML, ships it immediately:

```
Browser requests page
  │
  ▼
Server runs React, generates HTML ← happens on every request
  │
  ▼
HTML sent to browser              ← user sees content HERE
  │
  ▼
JS bundle downloads in parallel
  │
  ▼
React "hydrates" the HTML         ← app is interactive HERE
  (attaches event handlers to existing DOM)
```

Faster first paint. But the app isn't interactive until hydration
completes — the "uncanny valley" where buttons look clickable but aren't.

---

## Static Site Generation (SSG)

Same as SSR but HTML is generated **at build time**, not per-request:

```
Build time: React renders → static HTML files → deployed to CDN

Request time:
  Browser → CDN → cached HTML (instant) → hydrate
```

Best for pages that don't change per-user (blog posts, docs, marketing).

---

## Incremental Static Regeneration (ISR)

SSG + background revalidation:

```
First request:  serve cached static page
Background:     after staleTime, regenerate the page
Next request:   serve fresh page

stale-while-revalidate for entire pages.
```

```tsx
// Next.js App Router
async function Page() {
  const data = await fetch('https://api.example.com/posts', {
    next: { revalidate: 60 },  // revalidate every 60 seconds
  });
  return <PostList posts={data} />;
}
```

---

## React Server Components (RSC)

The newest model. Components that **only run on the server** and send
their rendered output (not their code) to the client:

```
Server Component:
  - Runs on the server
  - Can access databases, file system, secrets directly
  - Zero JS shipped to client
  - Cannot use state, effects, or browser APIs
  - Cannot handle user interaction

Client Component:
  - Runs on the client (and server for SSR)
  - Can use hooks, state, effects
  - Shipped as JS to the browser
  - Handles interactivity
```

### The mental model

```
          Server                              Client

  ┌─────────────────────┐          ┌──────────────────────┐
  │   Server Component  │          │  Client Component    │
  │   (async, DB access)│   ───►   │  (interactive, hooks)│
  │                     │  payload │                      │
  │   Renders to a      │          │  Hydrates & handles  │
  │   serialized stream │          │  events              │
  └─────────────────────┘          └──────────────────────┘
```

### What gets sent to the client

Server Components don't send their source code. They send a **serialized
description** of their rendered output (RSC payload):

```
// Server Component
async function UserProfile({ userId }) {
  const user = await db.query('SELECT * FROM users WHERE id = $1', [userId]);
  return <div className="profile">{user.name}</div>;
}

// What the client receives (simplified RSC payload):
// ["div", { className: "profile" }, "Alice"]
// Not the component code, not the DB query — just the result.
```

---

## The "use client" Boundary

```
'use client' marks the boundary where server → client handoff happens.

         Server tree
  ┌──────────────────────┐
  │  Layout (server)     │
  │    │                 │
  │  Page (server)       │
  │    │                 │
  │  UserProfile(server) │
  │    │                 │
  └────┼─────────────────┘
       │ 'use client' boundary
  ┌────┼─────────────────┐
  │  LikeButton (client) │
  │    │                 │
  │  Counter (client)    │  ← children of client component
  └──────────────────────┘     are also client
```

```tsx
// LikeButton.tsx
'use client';

import { useState } from 'react';

export function LikeButton({ postId }) {
  const [liked, setLiked] = useState(false);
  return <button onClick={() => setLiked(!liked)}>♥</button>;
}
```

**Key rules:**
1. Server Components can import Client Components.
2. Client Components CANNOT import Server Components.
3. But Client Components CAN receive Server Components as `children`:

```tsx
// This works — ServerContent is rendered on the server,
// passed as serialized children to ClientWrapper
<ClientWrapper>
  <ServerContent />
</ClientWrapper>
```

---

## Hydration Deep Dive

Hydration is the process of making server-rendered HTML interactive:

```
1. Server renders HTML → sent to browser
2. Browser displays HTML (fast, non-interactive)
3. React JS loads
4. React walks the existing DOM and attaches event handlers
5. React "adopts" the DOM — it now owns it
```

### Hydration mismatch

If the server and client render different output, React throws a
hydration error:

```tsx
// BUG: server and client render different content
function Greeting() {
  return <p>Hello at {new Date().toLocaleTimeString()}</p>;
  // Server: "Hello at 10:00:00"
  // Client: "Hello at 10:00:02"  → MISMATCH
}

// FIX: use useEffect for client-only values
function Greeting() {
  const [time, setTime] = useState(null);
  useEffect(() => setTime(new Date().toLocaleTimeString()), []);
  return <p>Hello{time ? ` at ${time}` : ''}</p>;
}
```

### Selective Hydration (React 18+)

With Suspense, React can hydrate components independently:

```tsx
<Suspense fallback={<Skeleton />}>
  <HeavyComponent />
</Suspense>
```

If the user clicks on a not-yet-hydrated Suspense boundary, React
**prioritizes hydrating that boundary first**. This is called
"selective hydration with event replay."

---

## Streaming SSR

React 18 can stream HTML as it renders:

```
Traditional SSR:
  Server renders entire page → sends complete HTML

Streaming SSR:
  Server sends HTML shell immediately
  │
  ├─ <header> ✓ sent
  ├─ <main>
  │   ├─ <Suspense> → sends fallback HTML
  │   │               later: streams in real content + inline <script>
  │   └─ <aside> ✓ sent
  └─ <footer> ✓ sent
```

The browser progressively receives and displays content. Slow data
fetches don't block the entire page.

---

## Next.js App Router Model

```
app/
├── layout.tsx        ← Server Component (shared layout)
├── page.tsx          ← Server Component (route)
├── loading.tsx       ← Suspense fallback (auto-wrapped)
├── error.tsx         ← Error boundary (must be 'use client')
└── components/
    ├── Header.tsx    ← Server Component (no interactivity needed)
    └── SearchBar.tsx ← Client Component ('use client')
```

### Data Fetching

```tsx
// Server Component — fetch directly, no useEffect, no loading state
async function PostsPage() {
  const posts = await db.post.findMany();
  return (
    <ul>
      {posts.map(post => <li key={post.id}>{post.title}</li>)}
    </ul>
  );
}
```

### Server Actions

Mutations from the client that execute on the server:

```tsx
// actions.ts
'use server';

export async function createPost(formData: FormData) {
  const title = formData.get('title');
  await db.post.create({ data: { title } });
  revalidatePath('/posts');
}

// Client Component
'use client';
import { createPost } from './actions';

function NewPostForm() {
  return (
    <form action={createPost}>
      <input name="title" />
      <button type="submit">Create</button>
    </form>
  );
}
```

---

## Interview Questions

---

**Q: What's the difference between SSR, SSG, and ISR?**

**How to answer:** Define each, when HTML is generated, and the
trade-off between freshness and speed.

**Answer:**

**What most candidates say:** "SSG is at build time, SSR is per request,
ISR is somewhere in between." This is accurate but shallow.

**What a senior candidate says:** These are points on a **freshness vs
performance trade-off curve**, and the right choice depends on your
tolerance for stale data:

**SSG:** HTML generated **at build time**, served from CDN. Zero server
computation per request. But content is frozen until the next deploy.
Best for: docs, marketing, blog posts.

**ISR:** Static pages that **revalidate in the background**. During the
revalidation window (e.g., 60 seconds), users get the stale cached page.
After the window expires, the *next* request triggers a background
rebuild — but **that user still gets the stale page**. The *following*
user gets the fresh one.

This is the critical detail most people miss: ISR has a **staleness
window**. If your product price changes and the next request comes 59
seconds later, the user sees the old price. Is that acceptable? For a
blog, yes. For a stock ticker, absolutely not.

**SSR:** HTML generated **on every request**. Always fresh, but every
user pays the render latency (database query + template rendering). For
a page that does 3 database queries at 50ms each, that's 150ms+ added
to every response.

```
SSG:  Build → CDN → instant (stale until next deploy)
ISR:  Build → CDN → instant (stale for N seconds, then background refresh)
SSR:  Request → Server → Render → Respond (always fresh, always slow)
```

**The production decision:** Start with SSG. Move to ISR when content
changes faster than your deploy cycle. Move to SSR only when the data
MUST be request-fresh (personalized dashboards, search results, real-time
pricing). In Next.js, check `next build` output: `○` = static, `●` = ISR,
`λ` = dynamic. Every `λ` should have a clear justification.

---

**Q: What are React Server Components and why do they matter?**

**How to answer:** Explain what they are, what they can't do, what
problem they solve (bundle size + data access), and how they relate to
Client Components.

**Answer:** React Server Components (RSCs) are components that execute
**exclusively on the server**. Their JavaScript code is never sent to
the client — the client only receives the **rendered output** as a
serialized payload (not HTML, but a special React format called the
RSC payload).

**What they can do:**
- Access databases, file systems, and secrets directly
- Use `await` at the top level (they're async)
- Import large libraries (markdown parsers, syntax highlighters) without
  affecting bundle size

**What they can't do:**
- Use state (`useState`, `useReducer`)
- Use effects (`useEffect`, `useLayoutEffect`)
- Use browser APIs (`window`, `document`, event handlers)
- Handle user interaction (no `onClick`, `onChange`)

**Why they matter (the real answer, not the marketing pitch):**

1. **Zero bundle size** — a Server Component that imports a 50KB markdown
   parser adds 0 bytes to the client bundle. But this only matters if
   your bundle is already large. If your total bundle is 100KB, saving
   50KB is significant. If it's 500KB, the 50KB savings is irrelevant
   compared to the other 450KB you need to address first.

2. **The data fetching model changes fundamentally** — this is the actual
   revolution. Traditional React: `render shell → download JS → hydrate →
   fetch() → render with data`. RSC: `fetch data on server → stream
   rendered result to client`. You go from **render-then-fetch** to
   **fetch-then-render**, which eliminates the client-server waterfall
   entirely. The data is fetched on the server, milliseconds from the
   database, not on the client hundreds of milliseconds from the API.

3. **Composable** — Server and Client Components mix freely. A Server
   Component can render a Client Component as a child. The boundary is
   marked with `'use client'`.

```
Server Component (fetches data, 0 JS to client)
  └── Client Component ('use client', handles clicks, shipped as JS)
       └── Server Component (rendered on server, passed as children)
```

**The trade-off nobody mentions:** RSCs can't use state or effects. This
is a huge constraint. If you need to add a simple hover tooltip to a
Server Component, you need to extract it into a Client Component. Over
time, the `'use client'` boundaries multiply as interactivity requirements
grow. RSCs work best for **content-heavy pages** (blog, docs, marketing)
where most of the page is static. For highly interactive apps (dashboards,
editors, real-time collaboration), you'll end up with mostly Client
Components anyway.

---

**Q: What is hydration?**

**How to answer:** Explain the full process — what arrives, what React
does, and what "interactive" means.

**Answer:**

**What most candidates say:** "Hydration is when React attaches event
handlers to server-rendered HTML."

**What a senior candidate says:** Hydration is the process where React
takes static, non-interactive server-rendered HTML and makes it fully
interactive. But the devil is in the **performance characteristics**.

Here's the timeline:

1. **Server renders HTML** → complete HTML string sent to browser
2. **Browser paints HTML** → user sees content immediately (fast)
3. **JS bundle downloads** → React code + your Client Components
4. **React hydrates** → walks the entire DOM tree, compares against
   expected output, attaches event handlers, initializes state and refs
5. **App is interactive** → buttons work, state updates happen

**The critical problem:** Step 4 is **expensive**. React must parse and
execute JavaScript for every Client Component, walk the entire DOM tree,
and reconcile it. For a large page, hydration can take **2-5 seconds**.
During this time, the page looks interactive (buttons are visible, inputs
are rendered) but **nothing works**. Users click buttons that don't
respond. They type into inputs that don't update. This is the **"uncanny
valley"** — worse than a loading spinner because it looks ready but isn't.

**Why this matters for architecture:** Hydration cost is directly
proportional to how much Client Component code you ship. This is why
"push `'use client'` to the leaves" is critical — a Server Component's
output is already HTML, it doesn't need hydration. Only Client Components
need to be hydrated.

**React 18 fixes:** Selective hydration + Suspense. Different parts of
the page hydrate independently behind Suspense boundaries. If the user
clicks a not-yet-hydrated section, React **prioritizes hydrating that
section first** and replays the click event after hydration completes.
This makes the uncanny valley much shorter for the specific element the
user is trying to interact with.

---

**Q: What causes hydration mismatches?**

**How to answer:** Explain the root cause, give common examples, and
show the fix pattern.

**Answer:** A hydration mismatch occurs when the HTML generated on the
server doesn't match what React tries to render on the client. React
compares the DOM against its expected output during hydration — if they
differ, it throws a warning (or error in strict mode).

**Common causes:**

1. **Date/time** — `new Date()` returns different values on server vs client:
   ```jsx
   // BUG: server says "10:00:00", client says "10:00:02"
   <p>{new Date().toLocaleTimeString()}</p>
   ```

2. **Browser-only APIs** — `window.innerWidth`, `navigator.userAgent`,
   `localStorage`:
   ```jsx
   // BUG: server doesn't have window
   <p>Width: {window.innerWidth}</p>
   ```

3. **Random values** — `Math.random()`, `crypto.randomUUID()` differ
   between server and client.

4. **Conditional rendering based on auth/cookies** — if the server sees
   a different auth state than the client.

**The fix pattern:** Defer client-only values to useEffect (which only
runs on the client):

```jsx
function Greeting() {
  const [time, setTime] = useState<string | null>(null);

  useEffect(() => {
    setTime(new Date().toLocaleTimeString());
  }, []);

  return <p>Hello{time ? ` at ${time}` : ''}</p>;
  // Server renders: "Hello"
  // Client hydrates: "Hello" (match!)
  // After hydration: "Hello at 10:00:02" (useEffect updates)
}
```

React 19 improved mismatch errors significantly — it now shows a diff
of exactly what didn't match instead of multiple confusing console errors.

---

**Q: Can a Client Component import a Server Component?**

**How to answer:** State the rule clearly, explain why, then show the
workaround.

**Answer:** No — a Client Component **cannot directly import** a Server
Component. The reason: when you mark a file with `'use client'`, everything
it imports becomes part of the client bundle. A Server Component can't
be in the client bundle because it may contain server-only code (database
queries, secret keys, file system access).

**But** a Client Component **can receive** a Server Component as `children`
or any other prop:

```tsx
// This works:
// ServerPage.tsx (Server Component)
export default function ServerPage() {
  const data = await db.query('SELECT * FROM posts');
  return (
    <ClientSidebar>         {/* Client Component */}
      <PostList posts={data} />  {/* Server Component, passed as children */}
    </ClientSidebar>
  );
}

// ClientSidebar.tsx
'use client';
export function ClientSidebar({ children }) {
  const [isOpen, setIsOpen] = useState(true);
  return (
    <aside>
      <button onClick={() => setIsOpen(!isOpen)}>Toggle</button>
      {isOpen && children}  {/* Renders the server-rendered content */}
    </aside>
  );
}
```

This works because the Server Component is rendered on the server
**first**, and its serialized output (not its code) is passed as the
`children` prop. The Client Component just receives rendered content,
not a component function to execute.

---

**Q: What are Server Actions?**

**How to answer:** Define what they are, how they work under the hood,
and when to use them vs API routes.

**Answer:** Server Actions are functions marked with `'use server'` that
run on the server but can be called directly from Client Components —
typically as form actions or event handlers.

**How they work under the hood:**

1. You define a function with `'use server'` — this tells the bundler
   to create an API endpoint for this function automatically
2. When the client calls it, React sends a POST request to that endpoint
   with the arguments serialized
3. The function executes on the server with full access to databases,
   secrets, etc.
4. The return value is serialized and sent back to the client
5. React automatically handles revalidation of affected data

```tsx
// actions.ts
'use server';
export async function addToCart(productId: string) {
  const session = await getSession();        // server-side auth
  await db.cart.add(session.userId, productId);  // database write
  revalidatePath('/cart');                     // refresh cached data
}

// ClientComponent.tsx
'use client';
import { addToCart } from './actions';

function AddButton({ productId }) {
  return (
    <form action={addToCart.bind(null, productId)}>
      <button type="submit">Add to Cart</button>
    </form>
  );
}
```

**Key advantages:**
- **Progressive enhancement** — forms work even without JavaScript
- **No API route boilerplate** — no route.ts, no fetch calls, no
  request/response handling
- **Built-in CSRF protection** — Origin header is validated automatically
- **Type-safe** — the function signature is shared between client and server

**What most people miss — the serialization boundary:** Server Actions
look like regular function calls, but they're POST requests with
serialized arguments. You can only pass **JSON-serializable** data. No
functions, no class instances, no Maps/Sets, no circular references. The
return value has the same constraint.

```tsx
// This WON'T work:
const result = await serverAction({
  callback: () => console.log('hi'),  // Functions can't be serialized
  data: new Map([['key', 'value']]),  // Map can't be serialized
});
```

This is why Server Actions are great for **forms** (FormData is
serializable) but not a universal replacement for API routes. For complex
request/response patterns, streaming, or non-serializable data, use route
handlers.

**When to use API routes instead:** When external clients need the
endpoint (mobile apps, webhooks, third-party integrations). Server
Actions are for your own UI only.
