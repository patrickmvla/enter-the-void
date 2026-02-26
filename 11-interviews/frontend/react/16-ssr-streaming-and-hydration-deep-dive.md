# SSR, Streaming & Hydration — The Deep Dive

This file goes **below the surface** of server rendering. The basics (SSR vs
SSG vs ISR, what hydration is, what RSCs are) are covered in
`05-server-components-and-ssr.md` and `nextjs/03-rendering-strategies.md`.
This file covers the **mechanics** — how streaming actually works, what the
RSC payload looks like, the full rendering pipeline, hydration internals,
and advanced patterns that separate production architects from tutorial
followers.

---

## The Rendering Pipeline — What Actually Happens

Most people think of SSR as "render HTML on the server." The reality is a
**multi-phase pipeline** with distinct bottlenecks at each stage.

### Traditional SSR (React 17 and earlier)

```
Request arrives
  │
  ▼
renderToString(React tree)          ← BLOCKING: entire tree must render
  │                                    before ANY bytes are sent
  ▼
Complete HTML string in memory
  │
  ▼
Send entire response                ← Single chunk, all or nothing
  │
  ▼
Browser parses HTML, paints
  │
  ▼
JS bundle downloads (parallel)
  │
  ▼
ReactDOM.hydrate()                  ← BLOCKING: entire tree hydrates
  │                                    as one synchronous pass
  ▼
App is interactive
```

**The problem:** Two massive blocking points. The server can't send anything
until the *slowest* data source resolves. The client can't be interactive
until the *entire* tree hydrates. If you have 50 components and one needs
a 3-second database query, all 50 wait.

### Streaming SSR (React 18+)

```
Request arrives
  │
  ▼
renderToPipeableStream(React tree)
  │
  ├─ Shell HTML sent immediately    ← browser starts painting
  │
  ├─ <Suspense> boundaries:
  │   ├─ Fallback HTML sent inline
  │   │
  │   └─ When data resolves:
  │       ├─ Replacement HTML sent as <template> + inline <script>
  │       └─ Browser swaps fallback with real content (no React needed)
  │
  ▼
Stream completes
  │
  ▼
JS bundle downloads (parallel with stream)
  │
  ▼
hydrateRoot() with selective hydration
  │
  ├─ Each Suspense boundary hydrates independently
  ├─ User interaction triggers priority hydration
  └─ Non-visible boundaries hydrate last
  │
  ▼
App is fully interactive
```

---

## How Streaming Actually Works — The Wire Format

When React streams HTML, it's not magic. Here's what goes over the wire:

### Phase 1: Shell + Suspense Fallbacks

```html
<!DOCTYPE html>
<html>
<head>...</head>
<body>
  <div id="root">
    <header>My App</header>

    <main>
      <h1>Product: Widget Pro</h1>
      <p>$49.99</p>

      <!--$?-->
      <template id="B:0"></template>
      <div class="skeleton">Loading reviews...</div>
      <!--/$-->

      <!--$?-->
      <template id="B:1"></template>
      <div class="skeleton">Loading cart...</div>
      <!--/$-->
    </main>
  </div>
```

The `<!--$?-->` comments are **Suspense boundary markers**. The `<template>`
tags are **replacement targets**. This HTML is sent immediately — the browser
starts painting the shell and skeletons right away.

### Phase 2: Streamed Replacements

As each Suspense boundary resolves, React streams a chunk:

```html
  <div hidden id="S:0">
    <section class="reviews">
      <h2>Reviews (47)</h2>
      <div class="review">Great product! ★★★★★</div>
      ...
    </section>
  </div>
  <script>
    // This is React's "swap" script — runs immediately
    // Finds B:0 template, replaces fallback with S:0 content
    $RC("B:0", "S:0")
  </script>
```

**Key insight:** The `$RC` (Replace Content) function is a tiny inline script
that React injects. It doesn't need the React runtime to be loaded yet. It
works with raw DOM manipulation:

1. Find the `<template id="B:0">` marker
2. Find the `<div hidden id="S:0">` replacement content
3. Remove the fallback (skeleton)
4. Insert the replacement content
5. Remove the hidden div and template

This is why streaming works **before React JS even loads** — the swap is
pure DOM manipulation via inline scripts.

### Phase 3: The stream closes

```html
  </body>
</html>
```

At this point, the browser has the complete HTML with all content swapped
in. React JS can load and hydrate at its own pace.

---

## RSC Payload Format — What Server Components Actually Send

Server Components don't send HTML. They send an **RSC payload** — a compact
serialized representation of the React tree.

### The format (simplified)

```
0:["$","div",null,{"className":"page","children":[
  ["$","h1",null,{"children":"Hello World"}],
  ["$","$Ld","user-section",{"fallback":["$","div",null,{"className":"skeleton"}]}],
  ["$","$1","like-button",{"postId":"abc123"}]
]}]

// Streamed later as chunks:
d:"user-section":["$","section",null,{"children":[
  ["$","p",null,{"children":"Welcome, Alice"}]
]}]
```

### What each part means:

```
"$"              → React element marker
"div"            → Element type (or component reference)
null             → Key
{...}            → Props

"$Ld"            → Suspense boundary (lazy, with data)
"$1"             → Client Component reference (by module ID)
```

### Why this matters in interviews:

1. **It's not HTML** — RSC payload is a tree description, not markup. React
   uses it to construct the virtual DOM on the client, which is then
   reconciled. This means Server Components participate in React's
   reconciliation — they're not just static HTML blobs.

2. **Client Components are references** — The payload contains `"$1"` (a
   module reference), not the component code. The client already has the
   Client Component code in its JS bundle. The payload just says "render
   component $1 here with these props."

3. **Streaming is per-Suspense-boundary** — Each boundary gets its own
   chunk in the RSC payload stream. This maps directly to the HTML streaming
   we saw above.

4. **Size implications** — RSC payload can be larger than equivalent HTML
   for simple content (because it includes the tree structure). But it's
   smaller for interactive content (because it references Client Components
   by ID instead of inlining their rendered output).

---

## Hydration Internals — What React Actually Does

### The reconciliation walk

When `hydrateRoot()` is called:

```
1. React creates a Fiber tree from your component tree (same as a render)
2. Instead of creating DOM nodes, it WALKS the existing DOM
3. For each Fiber node:
   a. Find the corresponding DOM node
   b. Verify it matches (tag name, key attributes)
   c. Attach event listeners
   d. Set up refs
   e. Initialize state and effects
4. If mismatch found → throw hydration error (dev) or silently recreate (prod)
```

### Why hydration is expensive

Hydration does ALMOST as much work as a full render:

```
Full render:
  1. Create Fiber tree          ✓
  2. Execute all components     ✓ (including Server Component results)
  3. Create DOM nodes           ✓
  4. Attach event listeners     ✓
  5. Set up refs                ✓

Hydration:
  1. Create Fiber tree          ✓ (same)
  2. Execute all components     ✓ (same — Client Components re-execute!)
  3. Walk existing DOM          ✓ (slightly cheaper than creating)
  4. Attach event listeners     ✓ (same)
  5. Set up refs                ✓ (same)
```

The only savings is step 3 — walking the DOM is cheaper than creating it.
But every Client Component still executes. Every useState initializer runs.
Every useEffect is scheduled. **Hydration is essentially a full render
that reuses DOM nodes instead of creating them.**

This is why "just SSR it" isn't a magic performance bullet. You still pay
the JavaScript execution cost.

### Selective Hydration — The React 18+ breakthrough

```
Traditional hydration:
  ┌─────────────────────────────────┐
  │  Entire page hydrates as one    │
  │  synchronous, blocking pass     │
  │  (2-5 seconds for large pages)  │
  └─────────────────────────────────┘

Selective hydration:
  ┌──────────┐ ┌──────────┐ ┌──────────┐
  │ Header   │ │ Content  │ │ Sidebar  │
  │ hydrated │ │ hydrated │ │ hydrated │
  │ at 100ms │ │ at 200ms │ │ at 500ms │
  └──────────┘ └──────────┘ └──────────┘
       Each Suspense boundary = independent hydration unit
```

**How it works:**

1. Each `<Suspense>` boundary creates an independent hydration unit
2. React hydrates boundaries in **priority order**:
   - Visible boundaries first (Intersection Observer)
   - Interacted-with boundaries get **highest priority**
3. If a user clicks a not-yet-hydrated boundary:
   - React **synchronously hydrates that boundary** immediately
   - The click event is **replayed** after hydration completes
   - The user doesn't notice the delay (sub-100ms for most components)

```tsx
// Each Suspense = independent hydration unit
function Page() {
  return (
    <>
      <Suspense fallback={<HeaderSkeleton />}>
        <Header />          {/* hydrates first — visible, above fold */}
      </Suspense>

      <Suspense fallback={<ContentSkeleton />}>
        <MainContent />     {/* hydrates second */}
      </Suspense>

      <Suspense fallback={<SidebarSkeleton />}>
        <Sidebar />         {/* hydrates last — below fold, less critical */}
      </Suspense>

      <Suspense fallback={null}>
        <Analytics />       {/* hydrates LAST — not visible, no interaction */}
      </Suspense>
    </>
  );
}
```

### Event Replay — The subtle genius

```
Timeline:
  0ms    — HTML arrives, user sees page
  100ms  — Header hydrated
  150ms  — User clicks "Buy Now" button (in not-yet-hydrated Content)
  150ms  — React captures the click event
  151ms  — React STOPS hydrating Sidebar
  151ms  — React starts hydrating Content (priority escalation)
  180ms  — Content hydrated
  180ms  — Click event is REPLAYED
  180ms  — onClick handler fires, user sees response

Total perceived delay: 30ms (from click to response)
Without selective hydration: 2000ms+ (entire page must hydrate first)
```

---

## Progressive Rendering Patterns

### Pattern 1: Shell-First with Streaming

The most common pattern. Static shell loads instantly, dynamic content
streams in.

```tsx
// app/dashboard/page.tsx
import { Suspense } from 'react';

export default function Dashboard() {
  return (
    <div className="dashboard">
      {/* Shell — renders instantly (static) */}
      <DashboardHeader />
      <NavigationSidebar />

      {/* Dynamic sections stream in independently */}
      <div className="dashboard-grid">
        <Suspense fallback={<MetricsSkeleton />}>
          <MetricsPanel />      {/* 200ms — fast DB query */}
        </Suspense>

        <Suspense fallback={<ChartSkeleton />}>
          <RevenueChart />      {/* 800ms — aggregation query */}
        </Suspense>

        <Suspense fallback={<TableSkeleton />}>
          <RecentOrders />      {/* 1200ms — joins multiple tables */}
        </Suspense>
      </div>
    </div>
  );
}
```

**Why this works:** The user sees the dashboard layout immediately. Metrics
appear first (fastest query). Charts appear next. The order table streams in
last. Each section is independently useful — the user can read metrics while
charts are still loading.

### Pattern 2: Nested Suspense for Progressive Detail

```tsx
function ProductPage({ params }) {
  return (
    <div>
      {/* Level 1: Product info (fast, cached) */}
      <Suspense fallback={<PageSkeleton />}>
        <ProductDetails id={params.id} />

        {/* Level 2: Reviews (moderate, paginated) */}
        <Suspense fallback={<ReviewsSkeleton />}>
          <ProductReviews id={params.id} />

          {/* Level 3: Related products (slow, ML recommendation) */}
          <Suspense fallback={<RelatedSkeleton />}>
            <RelatedProducts id={params.id} />
          </Suspense>
        </Suspense>
      </Suspense>
    </div>
  );
}
```

**The cascade:** Outer boundaries resolve first. Inner boundaries resolve
after. This creates a natural "progressive detail" effect — the most
important content appears first, supplementary content fills in.

### Pattern 3: Parallel Data + Serial UI

Start ALL data fetches immediately, but present results as they resolve:

```tsx
// This is WRONG — creates a waterfall:
async function Dashboard() {
  const user = await getUser();           // 100ms
  const orders = await getOrders(user.id); // 200ms (waits for user)
  const analytics = await getAnalytics();  // 500ms (waits for orders!)
  // Total: 800ms sequential
}

// This is BETTER — parallel with Promise.all:
async function Dashboard() {
  const [user, analytics] = await Promise.all([
    getUser(),         // 100ms ─┐
    getAnalytics(),    // 500ms ─┤── parallel
  ]);                  //        ─┘
  const orders = await getOrders(user.id);  // 200ms (depends on user)
  // Total: 700ms (500ms + 200ms)
}

// This is BEST — parallel data, streaming UI:
function Dashboard() {
  return (
    <>
      <Suspense fallback={<UserSkeleton />}>
        <UserSection />       {/* starts getUser() immediately */}
      </Suspense>

      <Suspense fallback={<AnalyticsSkeleton />}>
        <AnalyticsSection />  {/* starts getAnalytics() immediately */}
      </Suspense>

      <Suspense fallback={<OrdersSkeleton />}>
        <OrdersSection />     {/* starts getOrders() immediately */}
      </Suspense>
    </>
  );
  // Each component fetches independently, streams when ready
  // User section appears at 100ms, Analytics at 500ms, etc.
}
```

**The key insight:** When you move data fetching INTO individual Server
Components behind Suspense boundaries, React can start all fetches in
parallel automatically. You don't need `Promise.all` — the streaming
architecture parallelizes for you.

### Pattern 4: Preloading Data for Dependent Components

When one component's data depends on another's:

```tsx
// lib/data.ts
import { cache } from 'react';

// React.cache deduplicates within a single request
export const getUser = cache(async () => {
  return await db.user.findUnique({ where: { id: getCurrentUserId() } });
});

export const getOrders = cache(async (userId: string) => {
  return await db.order.findMany({ where: { userId } });
});

// PRELOAD pattern — start the fetch, don't await it
export const preloadOrders = (userId: string) => {
  void getOrders(userId);  // starts fetch, result is cached
};

// Component that uses it:
async function UserSection() {
  const user = await getUser();

  // Start fetching orders NOW — don't wait for OrdersSection to render
  preloadOrders(user.id);

  return <UserCard user={user} />;
}

async function OrdersSection() {
  const user = await getUser();    // cache hit — already resolved
  const orders = await getOrders(user.id);  // cache hit — preloaded!
  return <OrderList orders={orders} />;
}
```

**Why this matters:** Without preloading, `OrdersSection` would start
fetching orders only after it renders (waterfall). With preloading,
`UserSection` kicks off the orders fetch as soon as it has the userId,
and `OrdersSection` gets a cache hit.

---

## renderToPipeableStream — The API

```tsx
import { renderToPipeableStream } from 'react-dom/server';

// Express example (non-Next.js)
app.get('*', (req, res) => {
  const { pipe, abort } = renderToPipeableStream(
    <App url={req.url} />,
    {
      // Called when the shell is ready (everything outside Suspense)
      onShellReady() {
        res.statusCode = 200;
        res.setHeader('Content-Type', 'text/html');
        pipe(res);  // Start streaming
      },

      // Called if the shell fails to render
      onShellError(error) {
        res.statusCode = 500;
        res.send('<h1>Something went wrong</h1>');
      },

      // Called when ALL content (including Suspense) has resolved
      onAllReady() {
        // For crawlers/bots: wait for everything before sending
        // (they can't process streamed chunks)
      },

      // Called for any error during rendering
      onError(error) {
        console.error(error);
        // Note: you can't change status code here — headers already sent
      },
    }
  );

  // Safety timeout — abort if rendering takes too long
  setTimeout(() => abort(), 10000);
});
```

### Critical nuances:

**1. Status code is set in `onShellReady`:**

Once you call `pipe(res)`, the HTTP headers are sent. If a Suspense
boundary later errors, you can't change the status code — it's already
200. This is why error boundaries inside Suspense are critical.

**2. Bot detection for crawlers:**

Search engine bots can't process streaming HTML. You need to use
`onAllReady` instead of `onShellReady` for crawlers:

```tsx
onShellReady() {
  if (isCrawler(req.headers['user-agent'])) {
    return;  // Don't stream for bots — wait for onAllReady
  }
  res.statusCode = 200;
  pipe(res);
},
onAllReady() {
  if (isCrawler(req.headers['user-agent'])) {
    res.statusCode = 200;
    pipe(res);  // Send complete HTML for bots
  }
}
```

**3. Abort for timeouts:**

The `abort()` function stops rendering and sends whatever is ready. Any
unresolved Suspense boundaries are sent with their fallback HTML.

---

## Hydration Mismatch Deep Dive — Beyond the Basics

### The subtle mismatches that don't throw errors

React's mismatch detection isn't perfect. Some mismatches are **silent**:

```tsx
// Silent mismatch: attribute differences
// Server: <div class="dark">
// Client: <div class="light">
// React WON'T warn about className mismatches in some cases

// Silent mismatch: whitespace differences
// Server: <p>Hello World</p>
// Client: <p>Hello  World</p>  (extra space)
// React ignores whitespace differences

// LOUD mismatch: text content
// Server: <p>Hello</p>
// Client: <p>World</p>
// React WILL throw: "Text content did not match"

// LOUD mismatch: element type
// Server: <div>content</div>
// Client: <span>content</span>
// React WILL throw: "Expected div but found span"
```

### The suppressHydrationWarning escape hatch

```tsx
// For intentional mismatches (timestamps, random IDs):
<time suppressHydrationWarning>
  {new Date().toLocaleTimeString()}
</time>
```

**Warning:** This suppresses the warning, not the mismatch. React will
still use the client-rendered content, which means a **visual flash** —
the server-rendered time shows briefly, then React replaces it with the
client-rendered time. For timestamps, this is acceptable. For layout-
affecting content, it causes CLS.

### The useId pattern for stable identifiers

```tsx
// WRONG: generates different IDs on server vs client
function Input({ label }) {
  const id = `input-${Math.random().toString(36).slice(2)}`;
  return (
    <>
      <label htmlFor={id}>{label}</label>
      <input id={id} />
    </>
  );
}

// RIGHT: useId generates consistent IDs across server and client
function Input({ label }) {
  const id = useId();
  return (
    <>
      <label htmlFor={id}>{label}</label>
      <input id={id} />
    </>
  );
}
// Server: id = ":R1:"
// Client: id = ":R1:"  ← same, determined by position in tree
```

`useId` generates IDs based on the component's **position in the tree**,
which is the same on server and client. This is why the hook exists — not
for convenience, but for hydration correctness.

---

## Advanced: React Float — Resource Loading During Streaming

React 19 introduces `float` APIs for preloading resources during streaming:

```tsx
import { preload, preinit, prefetchDNS, preconnect } from 'react-dom';

async function ProductPage({ id }) {
  // These FLOAT up to the <head> — even from deeply nested components
  prefetchDNS('https://cdn.example.com');
  preconnect('https://api.example.com');
  preload('/fonts/inter.woff2', { as: 'font', type: 'font/woff2' });
  preinit('/lib/analytics.js', { as: 'script' });

  const product = await getProduct(id);
  return <ProductDetails product={product} />;
}
```

**Why this matters for streaming:** In traditional SSR, you put `<link>`
tags in the `<head>`. But with streaming, the `<head>` is sent in the shell
— BEFORE Suspense boundaries resolve. If a component inside a Suspense
boundary needs a font or script, it's too late to add it to the `<head>`.

Float solves this: resources called with `preload`/`preinit` are
**hoisted to the `<head>`** regardless of where in the tree they're called,
and they're sent as early as possible in the stream.

---

## Common Production Pitfalls

### 1. The Streaming + Error Status Code Problem

```
Timeline:
  0ms    — Request arrives
  10ms   — Shell renders, status 200 sent, headers flushed
  500ms  — Suspense boundary tries to render
  500ms  — Database throws error
  500ms  — Can't send 500 status — already sent 200!

Result: User gets a 200 OK response with an error message inside.
        Monitoring tools think it succeeded.
```

**Fix:** Use error boundaries inside Suspense. Log the error server-side.
Don't rely on HTTP status codes for streaming responses.

```tsx
<Suspense fallback={<Skeleton />}>
  <ErrorBoundary fallback={<ErrorMessage />}>
    <DataComponent />
  </ErrorBoundary>
</Suspense>
```

### 2. Streaming + CDN Buffering

Many CDNs (Cloudflare, Fastly) **buffer streaming responses** by default.
Your carefully crafted streaming response arrives at the user as one
big chunk — defeating the purpose entirely.

```
Server:  chunk1 ... 500ms ... chunk2 ... 800ms ... chunk3
CDN:     buffers all chunks → sends as single response
User:    waits 1300ms → gets everything at once
```

**Fix:** Set appropriate headers:

```tsx
res.setHeader('X-Accel-Buffering', 'no');       // Nginx
res.setHeader('Cache-Control', 'no-transform'); // CDN passthrough
// Or configure your CDN to pass through chunked Transfer-Encoding
```

### 3. Suspense Boundary Granularity

**Too few boundaries:** The whole page waits for the slowest component.

```tsx
// BAD: one boundary for everything
<Suspense fallback={<PageSkeleton />}>
  <FastComponent />       {/* ready at 50ms, but waits */}
  <MediumComponent />     {/* ready at 200ms, but waits */}
  <SlowComponent />       {/* ready at 2000ms — blocks all */}
</Suspense>
// User waits 2000ms to see anything
```

**Too many boundaries:** Skeleton soup — the page is full of independent
loading states that pop in at different times. Feels janky, not progressive.

```tsx
// BAD: skeleton soup
<Suspense fallback={<TitleSkeleton />}>
  <Title />
</Suspense>
<Suspense fallback={<DescriptionSkeleton />}>
  <Description />
</Suspense>
<Suspense fallback={<PriceSkeleton />}>
  <Price />
</Suspense>
// 15 skeletons all resolving at slightly different times = chaos
```

**Right approach:** Group by data source and visual coherence.

```tsx
// GOOD: grouped by data dependency + visual region
<Suspense fallback={<ProductSkeleton />}>
  {/* All from the same DB query — resolve together */}
  <Title />
  <Description />
  <Price />
</Suspense>

<Suspense fallback={<ReviewsSkeleton />}>
  {/* Separate data source, separate visual section */}
  <Reviews />
</Suspense>

<Suspense fallback={<RecommendationsSkeleton />}>
  {/* ML service, slowest — isolated so it doesn't block anything */}
  <Recommendations />
</Suspense>
```

### 4. Memory Pressure from Concurrent Streams

Each active streaming request holds state in memory until all Suspense
boundaries resolve:

```
100 concurrent requests × 10 Suspense boundaries × ~50KB state each
= ~50MB memory for streaming state alone

If a boundary takes 10 seconds (slow API):
  100 requests × 10 seconds = 1000 request-seconds of memory held
```

**Fix:**
- Set aggressive timeouts with `abort()`
- Monitor stream completion times
- Use connection pooling for downstream services
- Consider switching to full SSR (non-streaming) for routes with
  consistently slow data sources

### 5. The Double-Data Problem

When using RSC with client-side data fetching (TanStack Query), you can
accidentally send the same data twice:

```tsx
// Server Component
async function UserPage() {
  const user = await getUser();  // Fetched on server
  return (
    <div>
      <h1>{user.name}</h1>       {/* In the HTML */}
      <UserProfile userId={user.id} />  {/* Client Component */}
    </div>
  );
}

// Client Component
'use client';
function UserProfile({ userId }) {
  const { data: user } = useQuery({
    queryKey: ['user', userId],
    queryFn: () => fetchUser(userId),  // Fetches AGAIN on client!
  });
  // ...
}
```

**Fix:** Seed the client cache from server data:

```tsx
// Server Component
async function UserPage() {
  const user = await getUser();
  return (
    <HydrationBoundary state={dehydrate(queryClient)}>
      <UserProfile userId={user.id} initialData={user} />
    </HydrationBoundary>
  );
}

// Or pass initialData:
'use client';
function UserProfile({ userId, initialData }) {
  const { data: user } = useQuery({
    queryKey: ['user', userId],
    queryFn: () => fetchUser(userId),
    initialData,           // Use server data, don't refetch
    staleTime: 60 * 1000, // Don't refetch for 60 seconds
  });
}
```

---

## Interview Questions

---

**Q: Explain how React streaming SSR works under the hood.**

**How to answer:** Don't just say "React streams HTML." Walk through the
actual wire format — what goes over HTTP, how Suspense fallbacks are
replaced, and why it works before React JS loads.

**Answer:**

**What most candidates say:** "React 18 can stream HTML using Suspense.
Slow parts load later." This is surface-level.

**What a senior candidate says:** Streaming SSR is a multi-phase protocol
over a single HTTP response using chunked Transfer-Encoding:

**Phase 1 — Shell:** React renders everything outside Suspense boundaries
into HTML and sends it immediately. Suspense boundaries are sent as
**marker comments** (`<!--$?-->`) with a `<template>` tag and the fallback
HTML (skeleton). The browser starts painting this immediately.

**Phase 2 — Streamed replacements:** As each Suspense boundary's data
resolves, React appends to the HTTP response:
1. A `<div hidden>` containing the rendered HTML
2. An inline `<script>` that calls `$RC()` (Replace Content)

The `$RC` function is a tiny DOM manipulation script — it finds the
template marker, removes the fallback, inserts the real content. This
happens **without the React runtime loaded**. It's pure DOM manipulation.

**Phase 3 — Stream closes:** The HTTP response ends. All content has been
delivered.

**Phase 4 — Hydration:** React JS loads (it was downloading in parallel)
and calls `hydrateRoot()`. With selective hydration, each Suspense boundary
hydrates independently. Visible and interacted-with boundaries are
prioritized.

**The critical insight:** The shell, the Suspense fallbacks, and the
streamed replacements are all part of the SAME HTTP response. There's no
second request. The browser receives a continuous stream of HTML chunks
and processes each one as it arrives.

**Why this matters for architecture:** You can't change HTTP status codes
after the shell is sent. If a Suspense boundary errors, you've already
sent `200 OK`. You also need CDNs configured to not buffer the stream,
or the entire optimization is defeated.

---

**Q: What is selective hydration and how does event replay work?**

**How to answer:** Explain the mechanism, not just the concept. What
determines hydration order? What happens to user events?

**Answer:**

**What most candidates say:** "React 18 can hydrate parts of the page
independently."

**What a senior candidate says:** Selective hydration is React 18's
strategy for breaking hydration into **independent units** aligned with
Suspense boundaries.

**Hydration priority order:**
1. Boundaries whose HTML has arrived (from streaming)
2. Boundaries that are visible (via Intersection Observer heuristics)
3. Boundaries that the user has interacted with → **highest priority**

**Event replay mechanism:** When a user clicks on a not-yet-hydrated
element inside a Suspense boundary:

1. React captures the event (it has a root event listener on the document)
2. React identifies which Suspense boundary contains the target element
3. React **pauses current hydration work** (even mid-boundary)
4. React **synchronously hydrates** the clicked boundary
5. Once hydrated, React **replays the captured event** against the now-
   interactive component
6. The event handler fires as if it ran naturally

Perceived delay: typically under 100ms. Without selective hydration, the
user would wait for the ENTIRE page to hydrate (potentially 2-5 seconds)
before any click handler could fire.

**The subtle limitation:** Event replay only works for discrete events
(clicks, key presses). Continuous events like `mousemove` or `scroll`
are NOT replayed — they're lost. This is fine because continuous events
don't have meaningful individual instances, but it means hover states
won't trigger until hydration completes.

**Architectural implication:** Every `<Suspense>` boundary is a hydration
unit. Place boundaries around interactive sections that users are likely
to interact with early. Don't wrap the entire page in a single Suspense —
you lose the benefits of selective hydration.

---

**Q: What problems does streaming SSR solve that traditional SSR doesn't?**

**How to answer:** Name specific problems with `renderToString`, and
explain how streaming addresses each one.

**Answer:**

`renderToString` (traditional SSR) has three fundamental problems that
streaming solves:

**Problem 1: All-or-nothing rendering.**
With `renderToString`, the ENTIRE React tree must render before any bytes
are sent. If one component does a 3-second API call, the user waits 3
seconds for a blank page.

Streaming: The shell is sent immediately. Slow components stream in
behind Suspense boundaries. The user sees useful content within
milliseconds.

**Problem 2: All-or-nothing hydration.**
With traditional `ReactDOM.hydrate`, the ENTIRE tree hydrates as one
synchronous, blocking operation. During hydration (2-5 seconds for large
pages), the page looks interactive but isn't — clicks are silently dropped.

Streaming + selective hydration: Each Suspense boundary hydrates
independently. User interactions trigger priority hydration with event
replay. The uncanny valley is reduced from page-level to component-level.

**Problem 3: Waterfall data fetching.**
With `renderToString`, you must gather ALL data before rendering (no async
components). This forces either a waterfall or a complex `Promise.all` at
the route level.

Streaming: Each Server Component can be async independently. React starts
rendering the tree, hits a Suspense boundary with an async component,
sends the fallback, continues rendering other branches, and streams in
results as they resolve. Data fetching is naturally parallelized by the
component tree structure.

**Problem 4 (bonus): TTFB (Time to First Byte).**
`renderToString` TTFB = data fetching + full render time. For a complex
page, this can be 1-3 seconds.

Streaming TTFB = shell render time (no data fetching needed for shell).
Typically under 50ms. The rest streams in progressively.

---

**Q: How should you decide where to place Suspense boundaries?**

**How to answer:** This is an architecture question. Give a framework for
thinking about it, not just "wrap slow things."

**Answer:** Suspense boundary placement is an **architecture decision**
that affects three things: perceived performance, user experience, and
hydration granularity.

**The framework:**

1. **Group by data source.** Components that share the same data fetch
   should share a Suspense boundary. If title, description, and price all
   come from the same query, wrapping them in one boundary means they
   appear together (no partial product card).

2. **Isolate by latency.** Components with significantly different fetch
   times should be in separate boundaries. A 50ms product query and a
   2-second ML recommendation should never share a boundary — the fast
   one would wait for the slow one.

3. **Align with visual regions.** Boundaries should map to coherent UI
   sections. A boundary around a card's title but not its image creates
   a jarring pop-in. A boundary around the entire card is more natural.

4. **Consider hydration impact.** Every Suspense boundary is a selective
   hydration unit. Put boundaries around sections users interact with
   early (navigation, search, primary CTA). Low-priority sections
   (footer, analytics) can share a boundary or be in a lower-priority one.

**Anti-patterns:**
- **One giant boundary** — defeats the purpose of streaming entirely
- **Per-component boundaries** — skeleton soup, janky pop-in everywhere
- **Boundary inside a loading sequence** — if components A, B, C must
  appear together for UX coherence, one boundary around all three, not
  three separate boundaries

**The test:** Ask "if this section loaded 2 seconds after the rest, would
the user care?" If yes, it needs its own boundary (so it doesn't block
others). If no, it can share a boundary with related content.

---

**Q: What are the gotchas with streaming SSR in production?**

**How to answer:** These are the things that work perfectly in development
but break in production. Real-world deployment issues.

**Answer:**

**1. CDN buffering kills streaming.** Most CDNs and reverse proxies
(Nginx, Cloudflare, Fastly) buffer response bodies by default. Your
streaming response gets collected into one chunk and delivered at once.
Fix: Set `X-Accel-Buffering: no` for Nginx, configure CDN passthrough
for chunked encoding. Verify with `curl --no-buffer` that you actually
receive chunks progressively.

**2. HTTP status codes are committed early.** Once the shell is streamed,
you've sent `200 OK`. If a Suspense boundary later errors (database down,
API timeout), you can't change the status code. Monitoring tools that only
check HTTP status see "200 OK" for a page showing an error message. Fix:
Use error boundaries inside Suspense. Report errors to monitoring from
the error boundary, not from HTTP status.

**3. Memory pressure from long-running streams.** Each in-flight streaming
response holds React rendering state in memory. If you have 1000 concurrent
requests and a Suspense boundary takes 10 seconds (slow API), you're
holding 1000 response objects for 10 seconds. This is a memory leak vector.
Fix: Use `abort()` with aggressive timeouts (5-10 seconds max). If a
boundary can't resolve in time, the fallback HTML is sent as final content.

**4. SEO and bot handling.** Search engine crawlers (Googlebot) can handle
streaming, but many other bots can't. Social media scrapers (Facebook,
Twitter) often read only the initial HTML. If critical content (product
name, description, price) is behind a Suspense boundary and the bot
doesn't process the stream, it sees skeletons. Fix: Use `onAllReady`
for bot traffic (detected via User-Agent). Or ensure critical SEO content
is in the shell, not behind Suspense.

**5. Head management during streaming.** With traditional SSR, you render
the entire page, then set `<title>`, `<meta>` tags in the `<head>`. With
streaming, the `<head>` is sent in the shell — before Suspense boundaries
resolve. If the page title depends on data inside a Suspense boundary,
you can't set it. Fix: Use React Float (`preload`, `preinit`) which can
hoist resources to `<head>` during streaming. For metadata, fetch the
title data before the shell renders (outside Suspense).
