# Next.js Rendering Strategies & Optimization

## Static vs Dynamic Rendering

Next.js decides at **build time** whether a route is static or dynamic:

```
Static (default):
  Route rendered at build time → cached as HTML → served from CDN
  Fastest possible response. No server computation per request.

Dynamic:
  Route rendered on every request → fresh data every time
  Required when the response depends on request-time information.
```

### What makes a route dynamic?

A route becomes dynamic when it uses:

```
cookies()               — reads request cookies
headers()               — reads request headers
searchParams            — reads URL query params
fetch(..., { cache: 'no-store' })  — uncached fetch
Connection, after       — request-time APIs

Or when you explicitly set:
export const dynamic = 'force-dynamic';
```

### Route segment config

```tsx
// Fine-grained control per page/layout
export const dynamic = 'auto'           // default: let Next.js decide
export const dynamic = 'force-dynamic'  // always SSR
export const dynamic = 'force-static'   // always SSG
export const dynamic = 'error'          // error if dynamic APIs used

export const revalidate = 60;           // ISR: revalidate every 60s
export const revalidate = 0;            // equivalent to force-dynamic
export const revalidate = false;        // cache indefinitely (default)

export const runtime = 'nodejs';        // default Node.js runtime
export const runtime = 'edge';          // edge runtime (lighter, faster cold start)
```

---

## Partial Prerendering (PPR)

The future of Next.js rendering — combines static and dynamic in ONE page:

```
┌─────────────────────────────────────┐
│  Header (static) ✓ cached           │
├─────────┬───────────────────────────┤
│         │                           │
│ Sidebar │  Main Content (static)    │
│ (static)│  ✓ cached                 │
│         │                           │
│         ├───────────────────────────┤
│         │  ┌─────────────────────┐  │
│         │  │ User Cart (dynamic) │  │
│         │  │ Suspense boundary   │  │
│         │  │ → streams in later  │  │
│         │  └─────────────────────┘  │
│         │                           │
└─────────┴───────────────────────────┘
```

The static shell is served instantly from the CDN. Dynamic parts
stream in behind Suspense boundaries. Best of both worlds.

```tsx
import { Suspense } from 'react';

export default function ProductPage({ params }) {
  return (
    <div>
      {/* Static — cached at build time */}
      <ProductDetails id={params.id} />
      <ProductReviews id={params.id} />

      {/* Dynamic — streams in per-request */}
      <Suspense fallback={<CartSkeleton />}>
        <UserCart />  {/* uses cookies() → dynamic */}
      </Suspense>
    </div>
  );
}
```

---

## Image Optimization

```tsx
import Image from 'next/image';

<Image
  src="/hero.jpg"
  alt="Hero"
  width={1200}
  height={600}
  priority          // preload (use for above-the-fold images)
  placeholder="blur" // show blur while loading (requires blurDataURL or static import)
/>
```

What next/image does:
- **Resizes** to the exact dimensions needed
- **Converts** to modern formats (WebP, AVIF)
- **Lazy loads** by default (below fold)
- **Prevents CLS** (reserves space with width/height)
- **Serves from CDN** with proper cache headers

### Responsive images

```tsx
<Image
  src="/hero.jpg"
  alt="Hero"
  fill                     // fills parent container
  sizes="(max-width: 768px) 100vw, 50vw"  // hints for srcset
  style={{ objectFit: 'cover' }}
/>
```

---

## Font Optimization

```tsx
// app/layout.tsx
import { Inter } from 'next/font/google';

const inter = Inter({
  subsets: ['latin'],
  display: 'swap',    // prevent FOIT (flash of invisible text)
});

export default function RootLayout({ children }) {
  return (
    <html className={inter.className}>
      <body>{children}</body>
    </html>
  );
}
```

Next.js:
- Downloads fonts at build time (no external requests)
- Self-hosts them (no Google Fonts request in production)
- Automatically applies `size-adjust` to prevent CLS

---

## Metadata & SEO

```tsx
// Static metadata
export const metadata = {
  title: 'My App',
  description: 'A description of my app',
  openGraph: {
    title: 'My App',
    description: 'A description of my app',
    images: ['/og-image.jpg'],
  },
};

// Dynamic metadata
export async function generateMetadata({ params }) {
  const post = await getPost(params.slug);
  return {
    title: post.title,
    description: post.excerpt,
  };
}
```

---

## Bundle Optimization

### Analyzing the bundle

```bash
ANALYZE=true next build
```

### Tree shaking

Next.js tree-shakes automatically, but watch for:

```tsx
// BAD: imports entire library
import { format } from 'date-fns';

// GOOD: imports only what's needed (if tree-shaking isn't working)
import format from 'date-fns/format';
```

### Dynamic imports

```tsx
// Component only loads when rendered
const HeavyChart = dynamic(() => import('./HeavyChart'), {
  loading: () => <ChartSkeleton />,
  ssr: false,  // skip server rendering (for browser-only libs)
});
```

### Package import optimization

```js
// next.config.js
module.exports = {
  experimental: {
    optimizePackageImports: ['lucide-react', '@heroicons/react'],
  },
};
```

---

## Edge vs Node.js Runtime

```
Node.js Runtime (default):
  ✓ Full Node.js APIs
  ✓ Database drivers, file system, streams
  ✓ No code size limits
  ✗ Slower cold starts
  ✗ Runs in one region (unless multi-region deploy)

Edge Runtime:
  ✓ Fast cold starts (~0ms)
  ✓ Runs close to the user (global edge network)
  ✓ Lower cost
  ✗ Limited APIs (no fs, limited crypto, etc.)
  ✗ 4MB code size limit
  ✗ Limited npm package compatibility
```

```tsx
// Use edge for lightweight, latency-sensitive pages
export const runtime = 'edge';
```

---

## Common Performance Mistakes

### 1. Fetching on the client when you could fetch on the server

```tsx
// BAD: useEffect + fetch in a Client Component
'use client';
function Posts() {
  const [posts, setPosts] = useState([]);
  useEffect(() => {
    fetch('/api/posts').then(r => r.json()).then(setPosts);
  }, []);
}

// GOOD: fetch in a Server Component (no client JS, no waterfall)
async function Posts() {
  const posts = await db.post.findMany();
  return <PostList posts={posts} />;
}
```

### 2. Making everything a Client Component

```tsx
// BAD: 'use client' at the top of every file
// → entire tree ships as JS to the client

// GOOD: push 'use client' as low as possible
// Server Component (no directive)
function ProductPage({ id }) {
  const product = await getProduct(id);
  return (
    <div>
      <h1>{product.name}</h1>              {/* server — 0 JS */}
      <p>{product.description}</p>          {/* server — 0 JS */}
      <AddToCartButton productId={id} />    {/* client — only this ships JS */}
    </div>
  );
}
```

### 3. Not using Suspense boundaries

```tsx
// BAD: entire page waits for slowest fetch
async function Page() {
  const [user, posts, analytics] = await Promise.all([
    getUser(),          // 100ms
    getPosts(),         // 200ms
    getAnalytics(),     // 2000ms  ← blocks entire page!
  ]);
}

// GOOD: slow data streams in behind Suspense
function Page() {
  return (
    <div>
      <Suspense fallback={<UserSkeleton />}>
        <UserSection />       {/* appears at 100ms */}
      </Suspense>
      <Suspense fallback={<PostsSkeleton />}>
        <PostsSection />      {/* appears at 200ms */}
      </Suspense>
      <Suspense fallback={<AnalyticsSkeleton />}>
        <AnalyticsSection />  {/* appears at 2000ms */}
      </Suspense>
    </div>
  );
}
```

---

## Interview Questions

---

**Q: How does Next.js decide if a route is static or dynamic?**

**How to answer:** Explain the default (static), list what triggers
dynamic, and show how to override.

**Answer:**

**What most candidates say:** "cookies() and headers() make a route
dynamic." Correct but incomplete.

**What a senior candidate says:** Next.js defaults to static rendering
at build time. A route becomes dynamic only when it uses request-time APIs:

- `cookies()`, `headers()`, `searchParams` — read request data
- `fetch(url, { cache: 'no-store' })` — explicitly uncached
- `connection()`, `after()` — request-time APIs

**The critical gotcha:** A single `cookies()` call makes the **entire
route** dynamic — not just the component that calls it. If your product
page reads a theme preference from cookies, the entire page (product
details, reviews, recommendations) becomes dynamic. Even if `cookies()`
is in a deeply nested component, the route-level rendering strategy flips.

```tsx
// This entire page is dynamic because ONE component reads cookies:
async function ProductPage() {
  return (
    <div>
      <ProductDetails />   {/* could be static... but it's not */}
      <ProductReviews />   {/* could be static... but it's not */}
      <ThemeToggle />      {/* reads cookies() — makes EVERYTHING dynamic */}
    </div>
  );
}
```

**The fix:** Push the cookies() call into a Client Component (which
doesn't affect the server rendering strategy) or use Partial Prerendering
(PPR) with a Suspense boundary around the dynamic part.

**Use `dynamic = 'error'`** during development to catch accidental dynamic
API usage. If someone adds `cookies()` to a page that should be static,
the build will fail instead of silently degrading performance.

**How to check:** `next build` output: `○` = static, `λ` = dynamic,
`●` = ISR. Every `λ` should have a documented justification. Static routes
= 0ms server time + CDN edge. Dynamic routes = database query latency +
server render time on every single request.

---

**Q: What is Partial Prerendering?**

**How to answer:** Explain the problem it solves (all-or-nothing
rendering), how it works, and the developer experience.

**Answer:** Partial Prerendering (PPR) is an experimental rendering
strategy that lets a **single route** have both static and dynamic parts.
Before PPR, a route was either entirely static or entirely dynamic — one
`cookies()` call made the whole page dynamic.

**How it works:**

1. At build time, Next.js renders the route and captures everything
   **above** Suspense boundaries as the **static shell** — this gets
   cached on the CDN.

2. Dynamic parts (inside Suspense boundaries that use `cookies()`,
   `headers()`, etc.) are rendered per-request and **streamed in**.

```tsx
export default function ProductPage({ params }) {
  return (
    <div>
      {/* STATIC: rendered at build time, served from CDN */}
      <ProductDetails id={params.id} />
      <ProductReviews id={params.id} />

      {/* DYNAMIC: rendered per-request, streams in */}
      <Suspense fallback={<CartSkeleton />}>
        <UserCart />  {/* uses cookies() — dynamic */}
      </Suspense>
    </div>
  );
}
```

**What the user sees:**
1. Instant static shell from CDN (product info, reviews)
2. Cart skeleton in the Suspense fallback
3. Cart streams in a moment later with personalized data

**Why it matters:** You get CDN-speed for 90% of the page while still
having personalized, request-time content where needed. Before PPR,
the entire page would be dynamic (and slower) just because one small
section needed cookies.

**The nuance most candidates miss:** PPR sounds perfect until you consider
edge cases. If the dynamic part (user cart) takes 3 seconds to fetch,
the user sees the static shell instantly... with a skeleton that stays
for 3 seconds. Is that better than a fully dynamic page that takes 3
seconds? It depends — if the static shell is useful on its own (product
info, reviews), yes. If it's just a header and an empty container, the
skeleton is just a fancier loading spinner.

**Production readiness:** PPR is experimental. It requires careful
Suspense boundary placement — the boundary position determines what's
static vs dynamic. A misplaced boundary can accidentally make large
sections of your page dynamic or static.

---

**Q: How do you optimize images in Next.js?**

**How to answer:** Explain what next/image does automatically, then
cover the key props and when to use each.

**Answer:** The `next/image` component is a drop-in replacement for
`<img>` that handles optimization automatically:

**What it does:**
- **Resizes** to the exact dimensions needed for each device (generates
  multiple sizes via srcset)
- **Converts** to modern formats (WebP, AVIF) for browsers that support them
- **Lazy loads** by default — images below the fold only load when they
  enter the viewport
- **Prevents CLS** (Cumulative Layout Shift) — the width/height reserve
  space before the image loads, so the layout doesn't jump
- **Caches** on the server — resized images are generated once and cached

**Key props:**

```tsx
// Fixed size image:
<Image src="/hero.jpg" alt="Hero" width={800} height={400} />

// Fill parent container:
<Image src="/hero.jpg" alt="Hero" fill sizes="100vw"
       style={{ objectFit: 'cover' }} />

// Above-the-fold (disable lazy loading, add preload):
<Image src="/hero.jpg" alt="Hero" width={800} height={400} priority />

// Blur placeholder while loading:
<Image src={importedImage} alt="Hero" placeholder="blur" />
```

**Important:** Always set `priority` on the **Largest Contentful Paint
(LCP) image** — typically the hero image. This preloads it in the HTML
head, significantly improving LCP scores. Never lazy-load the LCP image.

The `sizes` prop is critical for responsive images — it tells the browser
which image size to request based on the viewport. Without it, the
browser may download an image larger than needed.

---

**Q: When would you use the edge runtime?**

**How to answer:** Compare edge vs Node.js, give specific use cases for
edge, and list what doesn't work on edge.

**Answer:** The edge runtime runs your code on a globally distributed
network close to the user, with very fast cold starts but limited APIs.

**Use the edge runtime when:**
- **Middleware** — auth redirects, locale detection, A/B testing. These
  run on every request and need to be fast. Edge cold start is ~0ms vs
  ~250ms for Node.js.
- **Lightweight API routes** — returning cached data, proxying requests,
  simple transforms.
- **Global latency matters** — if your users are worldwide and the
  computation is light (no database), edge serves them from the nearest
  data center.

```tsx
export const runtime = 'edge';
```

**Don't use edge when:**
- You need **database access** — most database drivers require TCP
  connections that aren't available on edge (though some, like
  @vercel/postgres and Neon, have HTTP adapters)
- You need **Node.js APIs** — `fs`, `path`, `child_process`,
  `crypto.createHash` (only Web Crypto is available)
- Your code is **large** — edge has a 4MB code size limit
- You need **long-running computation** — edge has CPU time limits

**The killer insight:** Edge is a **footgun for most developers**. It
looks amazing on paper (0ms cold start, global distribution), but here's
the reality: if your Route Handler queries a Postgres database in
us-east-1 and your user is in Tokyo, you run the handler in Tokyo...
then wait 200ms for the database round-trip to us-east-1. The edge saved
you 5ms of cold start latency but you're still bottlenecked by the
database.

Edge only helps when:
1. The computation doesn't need a traditional database (uses KV storage, or is pure compute)
2. The response can be computed close to the user (locale redirects, A/B routing)
3. You need global low-latency (middleware auth checks)

**In practice:** Middleware always runs on edge. For route handlers and
pages, stick with the Node.js runtime (default). 99% of apps don't need
edge for pages. If your database is single-region, edge routing just adds
complexity without reducing latency.

---

**Q: What's the biggest performance mistake in Next.js apps?**

**How to answer:** Name the mistake, explain why it's costly, and show
the fix with a concrete example.

**Answer:** The biggest mistake is **making too many Client Components**.
Developers coming from client-side React instinctively add `'use client'`
to every file. This defeats the core benefit of the App Router.

**Why it's costly:** When you mark a component with `'use client'`, its
code — plus all its imports — gets shipped to the browser as JavaScript.
If your page component is a Client Component, the entire page's JS
(including every library it imports) goes into the client bundle.

```tsx
// BAD: entire page is a Client Component
'use client';
import { marked } from 'marked';  // 50KB library → shipped to client
import { formatDate } from 'date-fns';  // 30KB → shipped to client

export default function BlogPost({ post }) {
  const [liked, setLiked] = useState(false);  // only reason for 'use client'
  return (
    <article>
      <h1>{post.title}</h1>
      <time>{formatDate(post.date, 'PPP')}</time>
      <div dangerouslySetInnerHTML={{ __html: marked(post.content) }} />
      <button onClick={() => setLiked(!liked)}>♥</button>
    </article>
  );
}
// Result: 80KB+ shipped to client
```

```tsx
// GOOD: push 'use client' to the leaf
// BlogPost.tsx — Server Component (default, no directive)
import { marked } from 'marked';       // 0 bytes to client
import { formatDate } from 'date-fns'; // 0 bytes to client

export default async function BlogPost({ params }) {
  const post = await getPost(params.slug);
  return (
    <article>
      <h1>{post.title}</h1>
      <time>{formatDate(post.date, 'PPP')}</time>
      <div dangerouslySetInnerHTML={{ __html: marked(post.content) }} />
      <LikeButton postId={post.id} />  {/* only this is client */}
    </article>
  );
}

// LikeButton.tsx — Client Component (tiny)
'use client';
export function LikeButton({ postId }) {
  const [liked, setLiked] = useState(false);
  return <button onClick={() => setLiked(!liked)}>♥</button>;
}
// Result: ~1KB shipped to client
```

**The principle:** Push `'use client'` as **deep into the tree as
possible**. Only the interactive leaves need to be Client Components.
Everything above — data fetching, layout, content rendering — should
be Server Components with zero client JS.
