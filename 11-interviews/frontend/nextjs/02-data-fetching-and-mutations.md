# Next.js Data Fetching & Mutations — Server-First Patterns

## The Server-First Model

In the App Router, data fetching is radically simpler than Pages Router:

```
Pages Router (old):                App Router (new):
  getServerSideProps                 async Server Components
  getStaticProps                     fetch() with caching options
  getStaticPaths                     generateStaticParams
  API routes + client fetch          Server Actions
  useEffect + useState
```

The key insight: **Server Components are async**. You can `await`
directly in the component body. No useEffect, no loading state
management, no client-server waterfall.

---

## Fetching in Server Components

```tsx
// This runs on the server. No client JS. No loading state needed.
async function PostsPage() {
  const posts = await db.post.findMany({
    orderBy: { createdAt: 'desc' },
    take: 20,
  });

  return (
    <ul>
      {posts.map(post => (
        <li key={post.id}>{post.title}</li>
      ))}
    </ul>
  );
}
```

You can talk to your database directly. No API layer needed for reads.

### Parallel data fetching

```tsx
// BAD: sequential — second fetch waits for first
async function Dashboard() {
  const user = await getUser();        // 200ms
  const posts = await getPosts();      // 300ms
  // Total: 500ms

  return <>{/* ... */}</>;
}

// GOOD: parallel — both start immediately
async function Dashboard() {
  const [user, posts] = await Promise.all([
    getUser(),     // 200ms
    getPosts(),    // 300ms
  ]);
  // Total: 300ms

  return <>{/* ... */}</>;
}
```

### Parallel with Suspense

Even better — show each section as it resolves:

```tsx
async function Dashboard() {
  return (
    <div>
      <Suspense fallback={<UserSkeleton />}>
        <UserProfile />   {/* fetches its own data */}
      </Suspense>
      <Suspense fallback={<PostsSkeleton />}>
        <RecentPosts />   {/* fetches its own data */}
      </Suspense>
    </div>
  );
}

// Each section streams in independently.
// User profile appears as soon as it's ready.
// Posts appear as soon as they're ready.
// No waterfall. No loading spinner for the whole page.
```

---

## Request Memoization

Next.js automatically deduplicates fetch calls with the same URL
and options **within a single render**:

```tsx
// Both components call the same endpoint — only ONE fetch happens

async function Header() {
  const user = await fetch('/api/user');  // fetch #1
  return <nav>{user.name}</nav>;
}

async function Sidebar() {
  const user = await fetch('/api/user');  // deduplicated — uses #1's result
  return <div>{user.avatar}</div>;
}
```

This means you can fetch data where you need it without worrying
about duplicate requests. No need to hoist data fetching and drill props.

**Note:** This only works with the native `fetch` API. ORM calls,
database queries, and other functions need manual caching with
React's `cache()`:

```tsx
import { cache } from 'react';

export const getUser = cache(async (id: string) => {
  return db.user.findUnique({ where: { id } });
});

// Now getUser(id) is deduplicated within the same render
```

---

## Caching Strategies

### Static (default for fetch)

```tsx
// Cached indefinitely — like SSG
const data = await fetch('https://api.example.com/posts');
```

### Time-based revalidation (ISR)

```tsx
// Cached, revalidated every 60 seconds
const data = await fetch('https://api.example.com/posts', {
  next: { revalidate: 60 },
});
```

### No cache (SSR)

```tsx
// Fresh on every request
const data = await fetch('https://api.example.com/posts', {
  cache: 'no-store',
});
```

### Tag-based revalidation

```tsx
// Tag the fetch
const data = await fetch('https://api.example.com/posts', {
  next: { tags: ['posts'] },
});

// Later, invalidate by tag (e.g., in a Server Action)
import { revalidateTag } from 'next/cache';
revalidateTag('posts');
```

---

## Server Actions — Mutations Without API Routes

Server Actions are functions that run on the server, called from the client:

```tsx
// app/actions.ts
'use server';

import { revalidatePath } from 'next/cache';

export async function createPost(formData: FormData) {
  const title = formData.get('title') as string;
  const content = formData.get('content') as string;

  await db.post.create({
    data: { title, content },
  });

  revalidatePath('/posts');
}
```

### Usage in forms (progressive enhancement)

```tsx
// Works WITHOUT JavaScript (form submits normally)
// Enhanced WITH JavaScript (no full page reload)
import { createPost } from './actions';

function NewPostForm() {
  return (
    <form action={createPost}>
      <input name="title" required />
      <textarea name="content" required />
      <button type="submit">Publish</button>
    </form>
  );
}
```

### Usage with useActionState (pending states)

```tsx
'use client';

import { useActionState } from 'react';
import { createPost } from './actions';

function NewPostForm() {
  const [state, formAction, isPending] = useActionState(createPost, null);

  return (
    <form action={formAction}>
      <input name="title" required />
      <textarea name="content" required />
      <button disabled={isPending}>
        {isPending ? 'Publishing...' : 'Publish'}
      </button>
      {state?.error && <p className="error">{state.error}</p>}
    </form>
  );
}
```

### Optimistic updates

```tsx
'use client';

import { useOptimistic } from 'react';

function TodoList({ todos, addTodo }) {
  const [optimisticTodos, addOptimisticTodo] = useOptimistic(
    todos,
    (state, newTodo) => [...state, { ...newTodo, pending: true }]
  );

  async function handleSubmit(formData) {
    const title = formData.get('title');
    addOptimisticTodo({ title, id: crypto.randomUUID() });
    await addTodo(formData);
  }

  return (
    <>
      <form action={handleSubmit}>
        <input name="title" />
      </form>
      <ul>
        {optimisticTodos.map(todo => (
          <li key={todo.id} style={{ opacity: todo.pending ? 0.5 : 1 }}>
            {todo.title}
          </li>
        ))}
      </ul>
    </>
  );
}
```

---

## Route Handlers (API Routes)

For when you need a traditional API endpoint:

```tsx
// app/api/posts/route.ts
import { NextResponse } from 'next/server';

export async function GET(request: Request) {
  const posts = await db.post.findMany();
  return NextResponse.json(posts);
}

export async function POST(request: Request) {
  const body = await request.json();
  const post = await db.post.create({ data: body });
  return NextResponse.json(post, { status: 201 });
}
```

When to use route handlers vs Server Actions:
- **Server Actions**: form submissions, mutations from your own UI
- **Route Handlers**: external webhooks, third-party API consumption,
  endpoints needed by mobile apps or other clients

---

## The Request Waterfall Problem

```
Client-side (old pattern):              Server-side (new pattern):

Browser renders shell                   Server fetches everything
  │                                       │
  ▼                                       ├─ fetch user (parallel)
fetch('/api/user')  ─── 200ms             ├─ fetch posts (parallel)
  │                                       ├─ fetch comments (parallel)
  ▼                                       │
fetch('/api/posts') ─── 300ms             ▼
  │                                     Stream HTML to client
  ▼                                       │
fetch('/api/comments') ─── 150ms          ▼
  │                                     Done: 300ms
  ▼
Done: 650ms (sequential waterfall)
```

Server Components eliminate client-server waterfalls. All data is
fetched on the server, close to the database, in parallel.

---

## Interview Questions

---

**Q: How does data fetching differ between the Pages Router and App Router?**

**How to answer:** Contrast the two models, showing you understand both
and why the App Router is better.

**Answer:** The fundamental difference is **where** you fetch and **how**
data flows:

**Pages Router (old model):**
- Fetch at the **page level** using `getServerSideProps` or `getStaticProps`
- These functions return data as props to the page component
- The page must **prop-drill** the data down to child components
- Only pages can fetch server-side — components below the page can't

```tsx
// Pages Router: fetch at page level, drill props
export async function getServerSideProps() {
  const user = await getUser();
  const posts = await getPosts();
  return { props: { user, posts } };
}
export default function Dashboard({ user, posts }) {
  return <Sidebar user={user}><PostList posts={posts} /></Sidebar>;
}
```

**App Router (new model):**
- **Any Server Component** can be async and fetch directly
- Data is fetched **where it's needed** — no prop drilling
- Duplicate fetches are **automatically deduplicated** within a render
- Components are independent — each manages its own data

```tsx
// App Router: fetch where you need it
async function Sidebar() {
  const user = await getUser();  // fetches its own data
  return <nav>{user.name}</nav>;
}
async function PostList() {
  const posts = await getPosts();  // fetches its own data
  return posts.map(p => <Post key={p.id} {...p} />);
}
```

The App Router eliminates prop drilling for server data and allows each
component to independently manage its loading/error states via Suspense.

---

**Q: What are Server Actions?**

**How to answer:** Define them, explain the mechanism, show form usage,
and contrast with API routes.

**Answer:** Server Actions are functions annotated with `'use server'`
that execute on the server but can be invoked from Client Components.
They're the primary way to handle mutations (create, update, delete) in
the App Router.

**How they work under the hood:**
1. The `'use server'` directive tells the Next.js bundler to create a
   hidden API endpoint for the function
2. When the client calls the action, React serializes the arguments and
   sends a POST request to that endpoint
3. The function runs on the server with full access to databases, secrets, etc.
4. The return value is serialized and sent back

```tsx
// actions.ts
'use server';
export async function createComment(formData: FormData) {
  const session = await getSession();
  if (!session) throw new Error('Unauthorized');

  const text = formData.get('text') as string;
  await db.comment.create({ data: { text, userId: session.userId } });
  revalidatePath('/posts');  // refresh the page data
}
```

```tsx
// Used in a form — works WITH and WITHOUT JavaScript:
<form action={createComment}>
  <textarea name="text" required />
  <button type="submit">Comment</button>
</form>
```

**Key benefits:**
- **Progressive enhancement** — the form works before JS loads
- **Type-safe** — function signature shared between client and server
- **Built-in security** — CSRF protection via Origin header checking
- **No API boilerplate** — no route handler, no fetch call, no
  request/response parsing

---

**Q: How do you avoid waterfall fetches in Next.js?**

**How to answer:** Show the problem (sequential fetches), then give two
solutions with code.

**Answer:** A waterfall happens when fetches run sequentially — each
waits for the previous to finish:

```tsx
// WATERFALL: 200ms + 300ms + 500ms = 1000ms total
async function Dashboard() {
  const user = await getUser();          // 200ms
  const posts = await getPosts();        // 300ms (waits for user)
  const analytics = await getAnalytics(); // 500ms (waits for posts)
}
```

**Solution 1: Promise.all** — run fetches in parallel when they're
independent:
```tsx
// PARALLEL: max(200, 300, 500) = 500ms total
async function Dashboard() {
  const [user, posts, analytics] = await Promise.all([
    getUser(),       // starts immediately
    getPosts(),      // starts immediately
    getAnalytics(),  // starts immediately
  ]);
}
```

**Solution 2: Suspense boundaries** — even better, each section streams
independently and appears as soon as its data is ready:
```tsx
function Dashboard() {
  return (
    <>
      <Suspense fallback={<UserSkeleton />}>
        <UserSection />        {/* appears at 200ms */}
      </Suspense>
      <Suspense fallback={<PostsSkeleton />}>
        <PostsSection />       {/* appears at 300ms */}
      </Suspense>
      <Suspense fallback={<AnalyticsSkeleton />}>
        <AnalyticsSection />   {/* appears at 500ms */}
      </Suspense>
    </>
  );
}
```

Suspense is better than Promise.all because the user sees content
**progressively** instead of waiting for the slowest fetch. Each
component is self-contained and fetches its own data.

---

**Q: When would you use a route handler instead of a Server Action?**

**How to answer:** Define the use case for each with concrete examples.

**Answer:** The distinction is about **who the consumer is**:

**Server Actions** — use when **your own UI** triggers the mutation.
Forms, buttons, client-side event handlers in your Next.js app:
```tsx
// Your own form submits to a Server Action
<form action={createPost}>...</form>
```

**Route Handlers** — use when **external clients** need the endpoint:
- **Webhooks** — Stripe, GitHub, or any service that POSTs to your URL
- **Mobile apps** — a React Native app that calls your REST API
- **Third-party integrations** — other services that consume your API
- **CORS requests** — when a different frontend needs your data
- **Non-form triggers** — things like cron jobs, CLI tools, Zapier

```tsx
// app/api/webhooks/stripe/route.ts
export async function POST(request: Request) {
  const event = await verifyStripeWebhook(request);
  // Process payment event
  return new Response('OK', { status: 200 });
}
```

**Rule of thumb:** If the caller is your Next.js frontend, use Server
Actions. If the caller is anything else, use route handlers.

---

**Q: How does request memoization work?**

**How to answer:** Explain the scope (per-render), the mechanism, and
the important limitation (only for fetch).

**Answer:** Next.js automatically deduplicates `fetch()` calls that have
the **same URL and options** within a **single server render**. If
multiple Server Components request the same data, only one network
request is made:

```tsx
// Two components, same fetch — ONE network request:
async function Header() {
  const user = await fetch('/api/user');  // request #1
  return <nav>{user.name}</nav>;
}
async function Profile() {
  const user = await fetch('/api/user');  // deduplicated — uses #1's result
  return <div>{user.email}</div>;
}
```

**Scope:** Memoization lasts for the duration of a single render (one
request to your Next.js server). It does NOT persist across different
user requests — that's what the Data Cache does.

**Important limitation:** This only works with the native `fetch()` API.
If you use an ORM (Prisma, Drizzle) or a database client directly,
memoization doesn't apply. For those, use React's `cache()` function:

```tsx
import { cache } from 'react';

// Wrap your data function — now it's deduplicated within one render
export const getUser = cache(async (id: string) => {
  return db.user.findUnique({ where: { id } });
});
```

This pattern is key for the App Router's "fetch where you need it"
philosophy — you don't need to hoist data fetching to a parent and
drill props. Each component fetches independently, and Next.js
deduplicates automatically.

**What trips up experienced devs:**

1. **Same URL, different options = different cache entries.** `fetch(url)`
   and `fetch(url, { cache: 'no-store' })` are NOT deduplicated — the
   options are part of the cache key. Two components that "fetch the same
   thing" but with different cache settings will trigger two requests.

2. **Memoization is per-render, not per-session.** If user A and user B
   hit your page simultaneously, each has their own memoization scope.
   For cross-request caching, you need the Data Cache (fetch options) or
   an external cache (Redis).

3. **Failure handling:** If the first fetch fails, the second component
   gets the same error — they share the result, including failures.
   There's no automatic retry at the memoization layer.

---

**Q: How do you handle optimistic updates in Next.js?**

**How to answer:** Explain the UX motivation, show the useOptimistic
pattern with a Server Action, and explain the failure case.

**Answer:** Optimistic updates make the UI feel **instant** by showing
the expected result immediately, before the server confirms. If the
server request fails, the UI automatically reverts.

The pattern uses React 19's `useOptimistic` hook with a Server Action:

```tsx
'use client';
import { useOptimistic } from 'react';
import { addTodo } from './actions';

function TodoList({ todos }) {
  const [optimisticTodos, addOptimistic] = useOptimistic(
    todos,  // real state from server
    (currentTodos, newTodo) => [...currentTodos, { ...newTodo, pending: true }]
  );

  async function handleAdd(formData: FormData) {
    const title = formData.get('title') as string;

    // 1. Instantly show the new todo (at 50% opacity to indicate pending)
    addOptimistic({ id: crypto.randomUUID(), title });

    // 2. Actually send to server
    await addTodo(formData);
    // 3a. On success: server revalidates, todos prop updates, optimistic resolves
    // 3b. On failure: React reverts to the real todos — optimistic item disappears
  }

  return (
    <>
      <form action={handleAdd}>
        <input name="title" />
        <button>Add</button>
      </form>
      <ul>
        {optimisticTodos.map(todo => (
          <li key={todo.id} style={{ opacity: todo.pending ? 0.5 : 1 }}>
            {todo.title}
          </li>
        ))}
      </ul>
    </>
  );
}
```

**The key mechanism:** `useOptimistic` maintains two layers — the **real
state** (from the `todos` prop, driven by the server) and the
**optimistic state** (applied on top). When the real state updates (via
`revalidatePath` in the Server Action), the optimistic layer dissolves
and the real data takes over. If the action throws, the optimistic state
is discarded and the UI snaps back to the last known good state.
