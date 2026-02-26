# Next.js Enterprise Patterns — Senior-Level Questions

## Authentication

### Server-side auth check (middleware)

```tsx
// middleware.ts
import { NextResponse } from 'next/server';

export function middleware(request) {
  const session = request.cookies.get('session')?.value;

  // Protect dashboard routes
  if (request.nextUrl.pathname.startsWith('/dashboard')) {
    if (!session) {
      return NextResponse.redirect(new URL('/login', request.url));
    }
  }

  // Redirect logged-in users away from login
  if (request.nextUrl.pathname === '/login' && session) {
    return NextResponse.redirect(new URL('/dashboard', request.url));
  }

  return NextResponse.next();
}

export const config = {
  matcher: ['/dashboard/:path*', '/login'],
};
```

### Server Component auth

```tsx
import { cookies } from 'next/headers';
import { redirect } from 'next/navigation';

async function DashboardPage() {
  const session = await getSession(cookies());
  if (!session) redirect('/login');

  return <Dashboard user={session.user} />;
}
```

### Auth patterns comparison

```
Middleware auth:
  ✓ Runs before any rendering
  ✓ Protects entire route groups
  ✓ Fast (edge runtime)
  ✗ Can't access database directly (edge limitations)
  ✗ Limited to cookie/header checks

Server Component auth:
  ✓ Full database access
  ✓ Can check permissions per-component
  ✓ Can render different UI based on role
  ✗ Route already started rendering

Client-side auth:
  ✓ Can protect interactive elements
  ✗ Flash of unauthenticated content
  ✗ JS must load first
```

---

## Role-Based Access Control (RBAC)

```tsx
// lib/auth.ts
type Role = 'admin' | 'editor' | 'viewer';

type Permission = 'posts:create' | 'posts:edit' | 'posts:delete' |
                  'users:manage' | 'settings:edit';

const rolePermissions: Record<Role, Permission[]> = {
  admin:  ['posts:create', 'posts:edit', 'posts:delete', 'users:manage', 'settings:edit'],
  editor: ['posts:create', 'posts:edit'],
  viewer: [],
};

export function hasPermission(role: Role, permission: Permission): boolean {
  return rolePermissions[role].includes(permission);
}

// Server Component usage
async function AdminPanel() {
  const session = await getSession();
  if (!hasPermission(session.role, 'users:manage')) {
    return <AccessDenied />;
  }
  return <UserManagement />;
}

// Client Component usage
function DeleteButton({ postId }) {
  const { role } = useAuth();
  if (!hasPermission(role, 'posts:delete')) return null;
  return <button onClick={() => deletePost(postId)}>Delete</button>;
}
```

---

## Internationalization (i18n)

```
app/
├── [locale]/
│   ├── layout.tsx
│   ├── page.tsx
│   └── about/page.tsx
├── middleware.ts         ← detect locale, redirect
└── dictionaries/
    ├── en.json
    └── fr.json
```

```tsx
// middleware.ts — locale detection
import { match } from '@formatjs/intl-localematcher';
import Negotiator from 'negotiator';

const locales = ['en', 'fr', 'de'];
const defaultLocale = 'en';

function getLocale(request) {
  const headers = { 'accept-language': request.headers.get('accept-language') };
  const languages = new Negotiator({ headers }).languages();
  return match(languages, locales, defaultLocale);
}

export function middleware(request) {
  const { pathname } = request.nextUrl;
  const hasLocale = locales.some(
    l => pathname.startsWith(`/${l}/`) || pathname === `/${l}`
  );

  if (!hasLocale) {
    const locale = getLocale(request);
    return NextResponse.redirect(
      new URL(`/${locale}${pathname}`, request.url)
    );
  }
}
```

```tsx
// app/[locale]/page.tsx
import { getDictionary } from './dictionaries';

export default async function Page({ params }) {
  const dict = await getDictionary(params.locale);
  return <h1>{dict.home.title}</h1>;
}
```

---

## Error Handling Strategy

```
app/
├── global-error.tsx    ← catches errors in root layout
├── error.tsx           ← catches errors in page
├── not-found.tsx       ← 404 page
└── dashboard/
    ├── error.tsx       ← catches errors in dashboard subtree
    └── page.tsx
```

```tsx
// app/dashboard/error.tsx
'use client';  // error boundaries must be client components

export default function DashboardError({
  error,
  reset,
}: {
  error: Error & { digest?: string };
  reset: () => void;
}) {
  useEffect(() => {
    logError(error);  // send to error tracking service
  }, [error]);

  return (
    <div>
      <h2>Something went wrong</h2>
      <p>{error.message}</p>
      <button onClick={reset}>Try again</button>
    </div>
  );
}
```

### Error handling layers

```
Layer 1: try/catch in Server Actions     → return error state to form
Layer 2: error.tsx per route segment     → catch render/data errors
Layer 3: global-error.tsx                → catch root layout errors
Layer 4: not-found.tsx                   → handle 404s
Layer 5: middleware                       → redirect on auth errors
```

---

## Security Best Practices

### Content Security Policy (CSP)

```tsx
// middleware.ts
export function middleware(request) {
  const nonce = crypto.randomUUID();
  const csp = [
    `default-src 'self'`,
    `script-src 'self' 'nonce-${nonce}'`,
    `style-src 'self' 'unsafe-inline'`,
    `img-src 'self' data: https:`,
    `connect-src 'self' https://api.example.com`,
  ].join('; ');

  const response = NextResponse.next();
  response.headers.set('Content-Security-Policy', csp);
  response.headers.set('x-nonce', nonce);
  return response;
}
```

### Server Action validation

```tsx
'use server';

import { z } from 'zod';

const CreatePostSchema = z.object({
  title: z.string().min(1).max(200),
  content: z.string().min(1),
});

export async function createPost(formData: FormData) {
  // Always validate — formData can be tampered with
  const parsed = CreatePostSchema.safeParse({
    title: formData.get('title'),
    content: formData.get('content'),
  });

  if (!parsed.success) {
    return { error: parsed.error.flatten() };
  }

  // Also check authorization
  const session = await getSession();
  if (!session) {
    throw new Error('Unauthorized');
  }

  await db.post.create({ data: parsed.data });
  revalidatePath('/posts');
}
```

### CSRF protection

Server Actions include CSRF protection by default — they check the
`Origin` header matches your domain. No extra setup needed.

---

## Performance Monitoring

### Core Web Vitals reporting

```tsx
// app/layout.tsx
import { SpeedInsights } from '@vercel/speed-insights/next';
import { Analytics } from '@vercel/analytics/next';

export default function RootLayout({ children }) {
  return (
    <html>
      <body>
        {children}
        <SpeedInsights />
        <Analytics />
      </body>
    </html>
  );
}
```

### Custom performance tracking

```tsx
// Report Web Vitals to your analytics
export function reportWebVitals(metric) {
  switch (metric.name) {
    case 'LCP':  // Largest Contentful Paint
    case 'FID':  // First Input Delay
    case 'CLS':  // Cumulative Layout Shift
    case 'INP':  // Interaction to Next Paint
    case 'TTFB': // Time to First Byte
      analytics.track(metric.name, {
        value: metric.value,
        rating: metric.rating,  // 'good' | 'needs-improvement' | 'poor'
      });
  }
}
```

---

## Multi-Tenancy

```tsx
// middleware.ts — subdomain-based multi-tenancy
export function middleware(request) {
  const hostname = request.headers.get('host');
  const subdomain = hostname?.split('.')[0];

  if (subdomain && subdomain !== 'www' && subdomain !== 'app') {
    // Rewrite to tenant-specific route
    return NextResponse.rewrite(
      new URL(`/tenant/${subdomain}${request.nextUrl.pathname}`, request.url)
    );
  }
}
```

---

## Interview Questions

---

**Q: How do you implement authentication in Next.js?**

**How to answer:** Describe the three-layer approach and explain what
each layer is best at.

**Answer:** I use a layered approach because each layer has different
capabilities and trade-offs:

**Layer 1: Middleware** — runs at the edge before any rendering. Fast,
but limited to cookie/header checks (can't query a database directly):
```tsx
// middleware.ts
export function middleware(request) {
  const session = request.cookies.get('session');
  if (!session && request.nextUrl.pathname.startsWith('/dashboard')) {
    return NextResponse.redirect(new URL('/login', request.url));
  }
}
```
Use for: protecting entire route groups, redirecting unauthenticated users,
preventing unnecessary server work.

**Layer 2: Server Components** — run on the server with full Node.js
access. Can query the database, verify tokens, check permissions:
```tsx
async function AdminPage() {
  const session = await getSession();
  if (!session) redirect('/login');
  if (session.role !== 'admin') redirect('/unauthorized');

  const users = await db.user.findMany();  // only admins see this
  return <UserTable users={users} />;
}
```
Use for: per-page authorization, role-based data access, rendering
different UI based on permissions.

**Layer 3: Client Components** — for UI-level protection (hiding buttons,
disabling features):
```tsx
function DeleteButton({ postId }) {
  const { role } = useAuth();
  if (role !== 'admin') return null;  // hide for non-admins
  return <button onClick={() => deletePost(postId)}>Delete</button>;
}
```
Use for: UX only. **Never rely on client-side checks for security** —
they can be bypassed. Always enforce on the server.

The layers work together: middleware catches unauthenticated users early
(fast, before any rendering), Server Components enforce authorization
(full DB access), and Client Components refine the UI.

**What most candidates miss — the cost trade-off:** Middleware runs on
**every matched request**. If your middleware verifies a JWT (even just
parsing it), that's CPU time on every request, including prefetches. A
senior would use middleware for the **cheapest possible check** (does
the session cookie exist?) and move expensive checks (is the token valid?
is the subscription active? does the user have this specific permission?)
to Server Components, where they only run on pages that need them.

**Race condition to watch for:** Middleware allows a request through
(session cookie exists), but by the time the Server Component runs, the
session has expired or the user's role has changed. This is rare but
real in apps with short-lived sessions. The Server Component is the
**source of truth** — middleware is an optimization, not a guarantee.

**Token refresh:** Where do you refresh expired tokens? Middleware can
set cookies but can't do async database lookups (edge runtime). Server
Components can refresh tokens but the response is already committed.
The common pattern: middleware checks token expiry, if expired, redirects
to a `/refresh` API route that does the async work and redirects back.

---

**Q: How do you handle errors in a Next.js production app?**

**How to answer:** Walk through each layer of error handling, from most
granular to most global.

**Answer:** I implement error handling in layers, with the most specific
boundary catching first:

**Layer 1: try/catch in Server Actions** — for mutation errors that
should return form feedback, not crash the page:
```tsx
'use server';
export async function createPost(formData: FormData) {
  try {
    const parsed = PostSchema.safeParse(Object.fromEntries(formData));
    if (!parsed.success) return { error: parsed.error.flatten() };
    await db.post.create({ data: parsed.data });
    revalidatePath('/posts');
    return { success: true };
  } catch (e) {
    return { error: 'Something went wrong. Please try again.' };
  }
}
```

**Layer 2: error.tsx per route segment** — catches render and data
fetching errors within that segment:
```tsx
// app/dashboard/error.tsx
'use client';
export default function DashboardError({ error, reset }) {
  useEffect(() => { Sentry.captureException(error); }, [error]);
  return (
    <div>
      <h2>Dashboard Error</h2>
      <p>{error.message}</p>
      <button onClick={reset}>Try again</button>
    </div>
  );
}
```

**Layer 3: global-error.tsx** — catches errors in the root layout
(the nuclear option — the entire app errored):
```tsx
// app/global-error.tsx
'use client';
export default function GlobalError({ error, reset }) {
  return (
    <html><body>
      <h1>Something went wrong</h1>
      <button onClick={reset}>Reload</button>
    </body></html>
  );
}
```

**Layer 4: not-found.tsx** — handles 404 errors when `notFound()` is
called or a route doesn't exist.

**Layer 5: Error tracking** — integrate Sentry, Datadog, or similar in
every error boundary. Include the `error.digest` property (a hash that
lets you correlate client and server errors without exposing stack traces).

**What most candidates miss:**

1. **error.tsx doesn't catch layout errors** — only errors in the page
   and its children. If the layout itself throws, the parent segment's
   error boundary catches it. If the root layout throws, only
   `global-error.tsx` catches it.

2. **error.tsx is a Client Component** — you can't inspect database state
   or server context when debugging. By the time the error boundary
   renders, you only have the error message. **Log detailed context
   (user ID, request params, database state) to Sentry BEFORE throwing.**

3. **Double-reporting risk** — if a Server Action catches an error and
   reports to Sentry, then re-throws, and error.tsx also reports to
   Sentry, you get duplicate reports. Use `error.digest` (a hash
   generated by Next.js) to correlate and deduplicate.

**Principle:** Errors should be caught at the **narrowest possible scope**.
A broken analytics widget shouldn't take down the entire dashboard.

---

**Q: How do you secure Server Actions?**

**How to answer:** Cover the three security concerns (input validation,
authorization, CSRF) with code examples.

**Answer:** Server Actions are effectively public API endpoints — anyone
can call them with any data. You must validate everything:

**1. Input validation** — never trust formData. Use Zod or a similar
schema validator:
```tsx
'use server';
const CreatePostSchema = z.object({
  title: z.string().min(1).max(200).trim(),
  content: z.string().min(1).max(10000),
});

export async function createPost(formData: FormData) {
  const parsed = CreatePostSchema.safeParse({
    title: formData.get('title'),
    content: formData.get('content'),
  });
  if (!parsed.success) {
    return { errors: parsed.error.flatten().fieldErrors };
  }
  // Use parsed.data (validated and typed), not raw formData
}
```

**2. Authorization** — always verify the session and check permissions.
A user can call any Server Action directly, even ones that aren't wired
to any visible button:
```tsx
export async function deletePost(postId: string) {
  const session = await getSession();
  if (!session) throw new Error('Unauthorized');

  const post = await db.post.findUnique({ where: { id: postId } });
  if (post.authorId !== session.userId && session.role !== 'admin') {
    throw new Error('Forbidden');
  }

  await db.post.delete({ where: { id: postId } });
}
```

**3. CSRF protection** — built-in. Next.js automatically checks that the
`Origin` header of Server Action requests matches your domain. No setup
needed, but it's good to know this exists.

**4. Rate limiting** — Server Actions don't have rate limiting by default.
For sensitive actions (login, password reset), add rate limiting via
middleware or a library like `@upstash/ratelimit`.

---

**Q: How would you implement i18n in the App Router?**

**How to answer:** Describe the architecture (dynamic segment + middleware
+ dictionaries) and show the key parts.

**Answer:** The approach uses three pieces: a `[locale]` dynamic segment,
middleware for detection/redirect, and server-side dictionary loading.

**1. Route structure** — every page lives under a locale prefix:
```
app/
├── [locale]/
│   ├── layout.tsx
│   ├── page.tsx          → /en, /fr, /de
│   └── about/page.tsx    → /en/about, /fr/about
├── middleware.ts
└── dictionaries/
    ├── en.json
    └── fr.json
```

**2. Middleware** — detects the user's preferred locale from the
Accept-Language header and redirects if no locale is in the URL:
```tsx
export function middleware(request) {
  const pathname = request.nextUrl.pathname;
  const hasLocale = locales.some(l => pathname.startsWith(`/${l}`));
  if (!hasLocale) {
    const locale = detectLocale(request);  // from Accept-Language header
    return NextResponse.redirect(new URL(`/${locale}${pathname}`, request.url));
  }
}
```

**3. Dictionary loading** — Server Components load the correct dictionary
at render time (zero JS to client):
```tsx
// dictionaries.ts
const dictionaries = {
  en: () => import('./dictionaries/en.json').then(m => m.default),
  fr: () => import('./dictionaries/fr.json').then(m => m.default),
};

export const getDictionary = async (locale: string) => dictionaries[locale]();

// app/[locale]/page.tsx
export default async function Page({ params }) {
  const dict = await getDictionary(params.locale);
  return <h1>{dict.home.title}</h1>;  // "Welcome" or "Bienvenue"
}
```

**4. generateStaticParams** — pre-render pages for all locales at build time:
```tsx
export function generateStaticParams() {
  return [{ locale: 'en' }, { locale: 'fr' }, { locale: 'de' }];
}
```

For formatting dates, numbers, and plurals within components, use the
`Intl` API (built into JavaScript) or a library like `next-intl`.

**What a senior candidate would flag — the scaling problem:** If you have
10 locales and 1,000 blog posts, `generateStaticParams` creates 10,000
pages at build time. Build times scale linearly with locales × pages.
For large sites, this becomes a real bottleneck.

**Solutions:**
- **On-demand generation:** Use `dynamicParams = true` to generate pages
  on first request instead of at build time
- **ISR:** Revalidate individual pages instead of rebuilding everything
- **CDN-level locale routing:** Instead of `/en/about` and `/fr/about`
  being separate Next.js routes, use middleware to rewrite based on
  cookie/header and serve the same route with different dictionaries

**Flash of wrong language:** If the locale is stored in a cookie and
the page is statically cached, the first request may show the wrong
language until client-side JS corrects it. PPR or server-side locale
detection in middleware prevents this.

---

**Q: How do you scale a Next.js app for high traffic?**

**How to answer:** Walk through the optimization layers from cheapest
to most complex.

**Answer:** I approach scaling in layers, starting with the optimizations
that give the biggest return for the least effort:

**1. Maximize static content (free performance):**
- Use SSG for every page that can be pre-built (marketing, docs, blog)
- Use ISR for semi-dynamic content (product pages, user profiles) so
  they're served from CDN but stay fresh
- Review `next build` output — any `λ` (dynamic) route should have a
  clear reason for being dynamic

**2. Cache aggressively:**
- Set `revalidate` on fetch calls — even 60 seconds of caching eliminates
  thousands of database queries
- Use `revalidateTag()` for targeted cache invalidation after mutations
- Configure CDN cache headers for static assets (`Cache-Control: immutable`)

**3. Optimize the client:**
- Use `next/image` with `priority` on LCP images
- Use `next/font` to self-host fonts (eliminates Google Fonts request)
- Push `'use client'` to leaf components to minimize JS bundle
- Code-split heavy routes with `dynamic(() => import(...))`

**4. Edge where possible:**
- Middleware at the edge for auth (no server round-trip for unauthorized users)
- Consider edge runtime for lightweight API routes

**5. Backend optimization:**
- Database connection pooling (PgBouncer, Prisma connection pool)
- Add database indexes for common query patterns
- Consider read replicas if reads far outnumber writes
- Use streaming SSR with Suspense so slow queries don't block fast content

**6. Monitor:**
- Track Core Web Vitals (LCP < 2.5s, INP < 200ms, CLS < 0.1)
- Use Vercel Analytics or Web Vitals reporting to catch regressions
- Set up alerts for performance budget violations

---

**Q: How do you implement RBAC in Next.js?**

**How to answer:** Show the data model, then demonstrate enforcement at
each layer (server, middleware, client).

**Answer:** Role-Based Access Control requires a permissions model and
enforcement at every layer of the application.

**1. Define the model:**
```tsx
type Role = 'admin' | 'editor' | 'viewer';
type Permission = 'posts:create' | 'posts:edit' | 'posts:delete' | 'users:manage';

const rolePermissions: Record<Role, Permission[]> = {
  admin:  ['posts:create', 'posts:edit', 'posts:delete', 'users:manage'],
  editor: ['posts:create', 'posts:edit'],
  viewer: [],
};

function hasPermission(role: Role, permission: Permission): boolean {
  return rolePermissions[role].includes(permission);
}
```

**2. Enforce in Server Components (data access):**
```tsx
async function AdminDashboard() {
  const session = await getSession();
  if (!hasPermission(session.role, 'users:manage')) {
    redirect('/unauthorized');
  }
  const users = await db.user.findMany();
  return <UserManagement users={users} />;
}
```

**3. Enforce in Server Actions (mutations):**
```tsx
'use server';
export async function deletePost(postId: string) {
  const session = await getSession();
  if (!hasPermission(session.role, 'posts:delete')) {
    throw new Error('Forbidden');
  }
  await db.post.delete({ where: { id: postId } });
}
```

**4. Enforce in middleware (route access):**
```tsx
export function middleware(request) {
  const session = getSessionFromCookie(request);
  if (request.nextUrl.pathname.startsWith('/admin') && session?.role !== 'admin') {
    return NextResponse.redirect(new URL('/unauthorized', request.url));
  }
}
```

**5. Reflect in Client Components (UX only):**
```tsx
function PostActions({ postId }) {
  const { role } = useAuth();
  return (
    <div>
      {hasPermission(role, 'posts:edit') && <EditButton postId={postId} />}
      {hasPermission(role, 'posts:delete') && <DeleteButton postId={postId} />}
    </div>
  );
}
```

**Critical rule:** Client-side checks are for **UX only** (hiding buttons
the user can't use). All real enforcement happens on the server. A
malicious user can bypass any client-side check — they can't bypass
server-side authorization.
