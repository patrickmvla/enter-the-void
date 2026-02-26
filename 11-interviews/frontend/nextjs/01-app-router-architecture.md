# Next.js App Router — Architecture & Routing

## File-Based Routing

The `app/` directory maps files to routes:

```
app/
├── page.tsx                    →  /
├── about/
│   └── page.tsx                →  /about
├── blog/
│   ├── page.tsx                →  /blog
│   └── [slug]/
│       └── page.tsx            →  /blog/my-post
├── shop/
│   └── [...slug]/
│       └── page.tsx            →  /shop/a, /shop/a/b, /shop/a/b/c
├── (marketing)/
│   ├── layout.tsx              →  shared layout for group (not in URL)
│   ├── pricing/page.tsx        →  /pricing
│   └── features/page.tsx       →  /features
└── @modal/
    └── login/page.tsx          →  parallel route (rendered alongside)
```

### Special Files

```
page.tsx       — the route's UI, makes the path publicly accessible
layout.tsx     — shared UI wrapper, preserves state across navigations
loading.tsx    — Suspense fallback, shown while page loads
error.tsx      — error boundary, catches errors in the subtree
not-found.tsx  — 404 UI for the segment
template.tsx   — like layout but remounts on navigation (no state preservation)
route.ts       — API route handler (GET, POST, etc.)
```

---

## Layouts: Nested & Persistent

Layouts wrap their children and **don't re-render on navigation**:

```
app/
├── layout.tsx          ← root layout (required, wraps everything)
└── dashboard/
    ├── layout.tsx      ← dashboard layout (sidebar, nav)
    ├── page.tsx        ← /dashboard
    └── settings/
        └── page.tsx    ← /dashboard/settings

Navigation /dashboard → /dashboard/settings:
  - Root layout:      preserved ✓ (no re-render)
  - Dashboard layout: preserved ✓ (no re-render)
  - Page:             swapped (dashboard → settings)
```

```tsx
// app/dashboard/layout.tsx
export default function DashboardLayout({ children }) {
  return (
    <div className="flex">
      <Sidebar />        {/* stays mounted */}
      <main>{children}</main>  {/* page swaps here */}
    </div>
  );
}
```

---

## Route Groups

Parenthesized folders organize routes **without affecting the URL**:

```
app/
├── (auth)/
│   ├── layout.tsx      ← auth-specific layout (centered card)
│   ├── login/page.tsx  → /login
│   └── signup/page.tsx → /signup
├── (app)/
│   ├── layout.tsx      ← app-specific layout (sidebar + header)
│   └── dashboard/page.tsx → /dashboard
```

---

## Dynamic Routes

```tsx
// app/blog/[slug]/page.tsx
export default async function BlogPost({ params }) {
  const { slug } = await params;
  const post = await getPost(slug);
  return <Article post={post} />;
}

// Generate static pages at build time
export async function generateStaticParams() {
  const posts = await getAllPosts();
  return posts.map((post) => ({ slug: post.slug }));
}
```

### Catch-all segments

```
app/docs/[...slug]/page.tsx

/docs/a       → params.slug = ['a']
/docs/a/b     → params.slug = ['a', 'b']
/docs/a/b/c   → params.slug = ['a', 'b', 'c']
```

### Optional catch-all

```
app/docs/[[...slug]]/page.tsx

/docs         → params.slug = undefined (matches root too)
/docs/a       → params.slug = ['a']
```

---

## Parallel Routes

Render multiple pages simultaneously in the same layout:

```
app/
├── layout.tsx
├── page.tsx
├── @analytics/
│   └── page.tsx
└── @team/
    └── page.tsx
```

```tsx
// app/layout.tsx
export default function Layout({ children, analytics, team }) {
  return (
    <div>
      {children}    {/* app/page.tsx */}
      {analytics}   {/* app/@analytics/page.tsx */}
      {team}        {/* app/@team/page.tsx */}
    </div>
  );
}
```

Use case: dashboards, modals that preserve URL state.

---

## Intercepting Routes

Intercept a route to show it in a different context (e.g., modal):

```
app/
├── feed/
│   └── page.tsx                 ← the feed
├── photo/[id]/
│   └── page.tsx                 ← full photo page (direct URL)
└── @modal/
    └── (.)photo/[id]/
        └── page.tsx             ← photo in a modal (intercepted)

Click photo in feed → modal opens (intercepted route)
Direct URL /photo/123 → full page (not intercepted)
Refresh while modal open → full page
```

Convention: `(.)` = same level, `(..)` = one level up, `(...)` = root.

---

## Middleware

Runs **before** every request, at the edge:

```tsx
// middleware.ts (root of project)
import { NextResponse } from 'next/server';

export function middleware(request) {
  const { pathname } = request.nextUrl;

  // Redirect unauthenticated users
  if (pathname.startsWith('/dashboard')) {
    const token = request.cookies.get('session');
    if (!token) {
      return NextResponse.redirect(new URL('/login', request.url));
    }
  }

  return NextResponse.next();
}

export const config = {
  matcher: ['/dashboard/:path*'],
};
```

Middleware can:
- Rewrite URLs (A/B testing, feature flags)
- Redirect (auth, locale detection)
- Set headers (CORS, CSP)
- Read/write cookies

Middleware CANNOT:
- Access the database directly (runs at edge, limited runtime)
- Use Node.js APIs (fs, path, etc.)

---

## Caching in Next.js

Next.js has **four layers** of caching:

```
Request Memoization
  │  Same fetch in multiple components → deduplicated within one render
  │
Data Cache
  │  fetch() results cached on the server across requests
  │  Controlled by: { next: { revalidate: 60 } } or { cache: 'no-store' }
  │
Full Route Cache
  │  Rendered HTML + RSC payload cached at build time (static routes)
  │  Invalidated by: revalidatePath(), revalidateTag()
  │
Router Cache (client-side)
     RSC payloads cached in the browser during session
     Prefetched routes stored for instant navigation
```

### Opting out of caching

```tsx
// Dynamic data — no caching
const data = await fetch(url, { cache: 'no-store' });

// Or mark the entire route as dynamic
export const dynamic = 'force-dynamic';

// Time-based revalidation
const data = await fetch(url, { next: { revalidate: 3600 } });

// On-demand revalidation
import { revalidatePath, revalidateTag } from 'next/cache';
revalidatePath('/blog');
revalidateTag('posts');
```

---

## Interview Questions

---

**Q: What's the difference between a layout and a template?**

**How to answer:** Define each behavior, then give a use case where
templates are better.

**Answer:** Both wrap child pages, but they handle navigation differently:

**Layout (`layout.tsx`):** Persists across navigations. When you navigate
from `/dashboard` to `/dashboard/settings`, the dashboard layout does
**not** unmount. Its state (open sidebar, scroll position, active
websocket connections) is preserved. React only swaps the child page
component.

```tsx
// This sidebar stays open when navigating between dashboard pages
export default function DashboardLayout({ children }) {
  const [sidebarOpen, setSidebarOpen] = useState(true);
  return (
    <div>
      <Sidebar isOpen={sidebarOpen} />
      <main>{children}</main>  {/* only this part swaps */}
    </div>
  );
}
```

**Template (`template.tsx`):** Remounts on every navigation. A new
instance is created, state is reset, effects re-run. Use when you need
a fresh start on each page.

**When to use a template:**
- **Page entry animations** — you need the component to mount fresh to
  trigger a CSS/framer-motion entrance animation
- **Per-page analytics** — useEffect in a template fires on every page
  visit; in a layout it only fires once
- **Reset form state** — a template resets all useState to initial values
  on navigation

**What most candidates miss:** Layouts don't just persist UI — they persist
**all state, effects, and connections**. A WebSocket connection opened in
a layout stays open across page navigations. An interval started in a
layout's useEffect keeps running. This is a feature for dashboards (don't
reconnect on every page) but a bug for pages with entry animations
(the animation never re-runs).

**When templates matter:** Entry animations (re-trigger on every visit),
per-page analytics logging (`useEffect` in a template fires on every
navigation, in a layout it fires once), forms that should reset on
navigation.

**In practice:** 99% of the time you want a layout. Templates are rare —
but knowing WHY they exist shows you understand React's component
lifecycle, not just Next.js's file conventions.

---

**Q: How does caching work in Next.js?**

**How to answer:** Walk through all four layers, what each caches, and
how to control each.

**Answer:** Next.js has four caching layers, each solving a different
problem:

**Layer 1: Request Memoization** (per-render)
Multiple components in the same render that call `fetch()` with the same
URL → only one network request. The result is shared. This means you
can fetch data where you need it without worrying about duplicates.
```tsx
// Both components fetch /api/user → only ONE request
async function Header() { const user = await fetch('/api/user'); }
async function Sidebar() { const user = await fetch('/api/user'); }
```

**Layer 2: Data Cache** (cross-request, server-side)
`fetch()` results are cached on the server and persisted across
requests. By default, fetches are cached indefinitely (static).
Control with:
```tsx
fetch(url, { cache: 'no-store' });           // never cache
fetch(url, { next: { revalidate: 60 } });    // cache for 60s
fetch(url, { next: { tags: ['posts'] } });   // tag-based invalidation
```

**Layer 3: Full Route Cache** (build-time, server-side)
Static routes are rendered at build time and the HTML + RSC payload is
cached. Dynamic routes skip this cache. Invalidated by `revalidatePath()`
or `revalidateTag()`.

**Layer 4: Router Cache** (client-side, in-memory)
RSC payloads are cached in the browser during the session. Prefetched
routes (from `<Link>` hover) are stored for instant navigation. This is
why navigating between pages feels instant in Next.js.

**How to opt out:** `{ cache: 'no-store' }` for individual fetches,
`export const dynamic = 'force-dynamic'` for entire routes, or
`revalidatePath`/`revalidateTag` for on-demand invalidation.

**The gotcha that trips up experienced devs:** Request memoization (Layer 1)
is **per-render**, not per-request. This means if 100 users hit your page
simultaneously, each gets their own memoization scope. Two components in
**the same render** calling `fetch('/api/user')` get deduplicated. Two
**different users** calling `fetch('/api/user')` do NOT — that's what
the Data Cache (Layer 2) is for. Confusing these two scopes is the #1
caching bug.

**Another gotcha:** `revalidateTag('posts')` invalidates the Data Cache
AND the Full Route Cache for any route that used that tagged fetch. But
it doesn't invalidate the Router Cache (Layer 4) — the client still shows
the old RSC payload until the user hard-refreshes or the cache expires
(30 seconds for dynamic, 5 minutes for static segments). This is why users
sometimes report seeing stale data after you've already invalidated.

---

**Q: What is middleware in Next.js and when would you use it?**

**How to answer:** Explain where it runs, what it can do, its
limitations, and give specific use cases.

**Answer:** Middleware is a function in `middleware.ts` at the project
root that runs **before every matched request**, at the **edge** (close
to the user, before hitting your server).

```tsx
export function middleware(request: NextRequest) {
  // Runs BEFORE the route renders
  const session = request.cookies.get('session');
  if (!session && request.nextUrl.pathname.startsWith('/dashboard')) {
    return NextResponse.redirect(new URL('/login', request.url));
  }
  return NextResponse.next();
}
export const config = { matcher: ['/dashboard/:path*'] };
```

**What it can do:**
- **Redirect** — auth checks, login redirects, old URL migrations
- **Rewrite** — A/B testing (rewrite /page to /page-variant-b), multi-tenancy
  (map subdomain.app.com to /tenant/subdomain/...)
- **Set headers** — add CORS headers, Content-Security-Policy, custom
  headers for analytics
- **Read/write cookies** — session management, locale preferences

**What it CANNOT do:**
- Access the database (runs on edge runtime, not Node.js)
- Use Node.js APIs (no `fs`, `path`, `child_process`)
- Run heavy computation (edge has CPU/memory limits)
- Stream responses or use WebSockets

**When to use it:** Auth redirects (most common), locale detection and
redirect, bot detection, feature flags, rate limiting headers. For
anything that needs database access, use Server Components or route
handlers instead.

**The production insight most candidates miss:** Middleware runs on
**every request** — including static assets, images, and prefetches.
If your middleware does a 5ms auth check on every request and your page
loads 40 assets, that's 200ms of middleware overhead. Use the `matcher`
config aggressively to limit which paths trigger middleware:

```tsx
// BAD: runs on every request (including /favicon.ico, /_next/static/...)
export function middleware(request) { ... }

// GOOD: only runs on pages that need auth
export const config = {
  matcher: ['/dashboard/:path*', '/admin/:path*', '/api/:path*'],
};
```

**When NOT to use middleware for auth:** If your auth check needs
database access (checking if the user's subscription is active), do it
in a Server Component, not middleware. Middleware can only read cookies
and check lightweight conditions. Heavy checks in middleware add latency
to every matched request.

---

**Q: What are parallel routes?**

**How to answer:** Explain the concept with the @folder convention,
show a real layout, and explain why they're useful.

**Answer:** Parallel routes let you render **multiple independent page
components simultaneously** in the same layout. Each slot has its own
loading, error, and not-found states.

Define them with the `@folder` naming convention:

```
app/
├── layout.tsx          ← receives slots as props
├── page.tsx            ← default slot (children)
├── @analytics/
│   ├── page.tsx        ← analytics slot
│   └── loading.tsx     ← independent loading state
└── @feed/
    ├── page.tsx        ← feed slot
    └── error.tsx       ← independent error boundary
```

```tsx
// layout.tsx — receives parallel routes as named props
export default function Layout({ children, analytics, feed }) {
  return (
    <div className="grid grid-cols-3">
      <main>{children}</main>     {/* app/page.tsx */}
      <aside>{feed}</aside>       {/* app/@feed/page.tsx */}
      <section>{analytics}</section>  {/* app/@analytics/page.tsx */}
    </div>
  );
}
```

**Why this matters:**
- Each slot loads and errors **independently**. If analytics fails, the
  feed and main content still work.
- Each slot can have its own `loading.tsx` — so the analytics spinner
  doesn't block the feed from appearing.
- **Modal pattern:** Use intercepting routes + parallel routes to show a
  modal (parallel) that intercepts a route, while the full page still
  works on direct URL access.

---

**Q: How does file-based routing work in the App Router?**

**How to answer:** Walk through the folder structure, special files, and
dynamic segment syntax.

**Answer:** In the App Router, the file system IS the router. Every
folder inside `app/` is a potential route segment:

**The rules:**
1. **Folders** define URL segments: `app/blog/post/` → `/blog/post`
2. **`page.tsx`** makes a segment publicly accessible. Without page.tsx,
   the folder is just a grouping mechanism — it won't match any URL.
3. **Special files** add UI behavior to a segment:

```
page.tsx      → the route's UI (required to make route accessible)
layout.tsx    → shared wrapper (persists across child navigations)
loading.tsx   → Suspense fallback (shown while page.tsx loads)
error.tsx     → error boundary (catches errors in the segment)
not-found.tsx → 404 UI (shown when notFound() is called)
route.ts      → API endpoint (GET, POST handlers — no UI)
```

**Dynamic segments:**
```
[slug]      → single param:   /blog/hello-world → params.slug = 'hello-world'
[...slug]   → catch-all:      /docs/a/b/c → params.slug = ['a', 'b', 'c']
[[...slug]] → optional catch:  /docs → params.slug = undefined, /docs/a → ['a']
```

**Route groups** — parenthesized folders that organize code without
affecting the URL:
```
app/(marketing)/pricing/page.tsx  → /pricing  (not /marketing/pricing)
app/(app)/dashboard/page.tsx      → /dashboard
```

Each group can have its own layout — so marketing pages get a centered
layout while app pages get a sidebar layout, without nesting in the URL.

**What a senior would emphasize:** The file system being the router means
**colocation**. Components, tests, styles, and utilities can live next to
the page that uses them without accidentally becoming routes (only
`page.tsx` creates a route). This was a deliberate design decision —
in the Pages Router, every file in `/pages` was a route, making colocation
impossible.

```
app/dashboard/
├── page.tsx          ← the route (/dashboard)
├── _components/      ← private folder (not a route)
│   ├── Chart.tsx     ← colocated component
│   └── Chart.test.ts ← colocated test
└── utils.ts          ← colocated utility
```
