# Data Fetching & TanStack Query — Server State Mastery

## The Evolution of Data Fetching in React

```
Era 1: componentDidMount + fetch (class components)
  → manual loading/error state, no caching, no dedup

Era 2: useEffect + useState (hooks)
  → same problems but with hooks, plus stale closure bugs

Era 3: SWR / TanStack Query (server state libraries)
  → caching, dedup, background refresh, optimistic updates

Era 4: React Server Components (Next.js App Router)
  → fetch on the server, no client JS, streaming
  → TanStack Query still needed for client-side mutations/polling
```

---

## Why useState + useEffect Is Broken for Data Fetching

```tsx
// The "standard" pattern everyone writes. It's broken in 6 ways.
function UserProfile({ userId }) {
  const [user, setUser] = useState(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState(null);

  useEffect(() => {
    let cancelled = false;
    setLoading(true);
    setError(null);

    fetch(`/api/users/${userId}`)
      .then(r => {
        if (!r.ok) throw new Error('Failed');
        return r.json();
      })
      .then(data => {
        if (!cancelled) { setUser(data); setLoading(false); }
      })
      .catch(err => {
        if (!cancelled) { setError(err); setLoading(false); }
      });

    return () => { cancelled = true; };
  }, [userId]);
}
```

**What's broken:**

1. **No caching** — navigate away and back = refetch from scratch.
   The user sees a loading spinner every time.

2. **No deduplication** — 3 components need the same user? 3 fetches.
   3 loading spinners. 3 independent error states.

3. **No background refresh** — data goes stale while the user sits on
   the page. Another user changes the data. This user sees old data
   until they manually refresh.

4. **No retry** — network blip? Show error forever. User must reload
   the page.

5. **Race condition** — if userId changes quickly (A → B → A), the
   response for B might arrive after A's second response, showing
   wrong data. The `cancelled` flag helps but doesn't prevent the
   fetch itself.

6. **Memory leak potential** — if the component unmounts during fetch,
   `cancelled` prevents setState, but the promise is still pending.
   In React 18 Strict Mode, the effect runs twice on mount.

---

## TanStack Query — The Complete Mental Model

### Core concepts

```
┌─────────────────────────────────────────────┐
│                Query Client                 │
│  (global cache, manages all queries)        │
│                                             │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  │
│  │ ['users'] │  │['user',1]│  │['posts'] │  │
│  │           │  │          │  │          │  │
│  │ data: []  │  │ data: {} │  │ data: [] │  │
│  │ stale: T  │  │ stale: F │  │ stale: T │  │
│  │ fetching: │  │ fetching:│  │ fetching:│  │
│  │   false   │  │  false   │  │  true    │  │
│  └──────────┘  └──────────┘  └──────────┘  │
└─────────────────────────────────────────────┘
```

**Query Key** — the cache key. Determines which cache entry to read/write.
Must be serializable. Hierarchical by convention:

```tsx
['users']                    // all users
['users', { role: 'admin' }] // filtered users
['users', userId]            // single user
['users', userId, 'posts']   // user's posts
```

**Staleness** — controlled by `staleTime`. After staleTime, data is
considered stale. Stale data is still shown (instant) but a background
refetch is triggered.

**Garbage collection** — controlled by `gcTime` (formerly `cacheTime`).
After this duration with no active subscribers, the cache entry is deleted.

```
staleTime: 5min, gcTime: 10min

t=0:   fetch → cache → fresh
t=5:   stale → show cached, refetch in background
t=10:  no subscribers for 10min → garbage collected
t=11:  new subscriber → full fetch (no cache)
```

### useQuery — reads

```tsx
function UserProfile({ userId }) {
  const {
    data,              // the cached data (undefined until first fetch)
    isLoading,         // true on first fetch (no cached data)
    isFetching,        // true during any fetch (including background)
    isError,           // true if last fetch failed
    error,             // the error object
    isSuccess,         // true if data is available
    isStale,           // true if data is past staleTime
    refetch,           // manually trigger a refetch
    dataUpdatedAt,     // timestamp of last successful fetch
  } = useQuery({
    queryKey: ['users', userId],
    queryFn: () => fetchUser(userId),
    staleTime: 5 * 60 * 1000,       // 5 minutes
    gcTime: 10 * 60 * 1000,          // 10 minutes
    retry: 3,                         // retry failed requests 3 times
    retryDelay: (attempt) => Math.min(1000 * 2 ** attempt, 30000),
    refetchOnWindowFocus: true,       // refetch when tab regains focus
    refetchOnReconnect: true,         // refetch when network reconnects
    enabled: !!userId,                // only fetch when userId exists
  });

  if (isLoading) return <Skeleton />;
  if (isError) return <ErrorDisplay error={error} retry={refetch} />;
  return <Profile user={data} />;
}
```

### isLoading vs isFetching

```
isLoading: first fetch ever (no cached data exists)
  → show skeleton/spinner

isFetching: ANY fetch (including background refetch)
  → show subtle indicator (spinner in corner, not full skeleton)

Scenario: user visits page, navigates away, returns
  Visit 1: isLoading=true, isFetching=true → skeleton
  Visit 2: isLoading=false, isFetching=true → show cached data + subtle spinner
```

This distinction is critical for UX. On return visits, show cached
data immediately (feels instant) with a subtle background refresh
indicator — NOT a full loading skeleton.

---

### useMutation — writes

```tsx
function CreatePost() {
  const queryClient = useQueryClient();

  const mutation = useMutation({
    mutationFn: (newPost: CreatePostInput) => createPost(newPost),

    // Optimistic update
    onMutate: async (newPost) => {
      // Cancel outgoing refetches (so they don't overwrite optimistic update)
      await queryClient.cancelQueries({ queryKey: ['posts'] });

      // Snapshot current cache
      const previousPosts = queryClient.getQueryData(['posts']);

      // Optimistically update cache
      queryClient.setQueryData(['posts'], (old: Post[]) => [
        { ...newPost, id: 'temp-' + Date.now(), pending: true },
        ...old,
      ]);

      // Return snapshot for rollback
      return { previousPosts };
    },

    onError: (err, newPost, context) => {
      // Rollback to snapshot
      queryClient.setQueryData(['posts'], context?.previousPosts);
    },

    onSettled: () => {
      // Refetch to ensure server state is synced
      queryClient.invalidateQueries({ queryKey: ['posts'] });
    },
  });

  return (
    <form onSubmit={(e) => {
      e.preventDefault();
      mutation.mutate({ title, content });
    }}>
      {mutation.isError && <p>Error: {mutation.error.message}</p>}
      <button disabled={mutation.isPending}>
        {mutation.isPending ? 'Creating...' : 'Create Post'}
      </button>
    </form>
  );
}
```

### Cache Invalidation Strategies

```tsx
// 1. Invalidate by exact key — refetches this specific query
queryClient.invalidateQueries({ queryKey: ['users', userId] });

// 2. Invalidate by prefix — refetches ALL user-related queries
queryClient.invalidateQueries({ queryKey: ['users'] });

// 3. Set data directly (skip refetch, use when mutation returns new data)
queryClient.setQueryData(['users', userId], updatedUser);

// 4. Remove from cache entirely (next read triggers full fetch)
queryClient.removeQueries({ queryKey: ['users', userId] });
```

**The production insight:** After a mutation, use `invalidateQueries` (not
`setQueryData`) unless the mutation response contains the complete updated
data. `invalidateQueries` marks queries as stale and refetches them,
ensuring the cache matches the server. `setQueryData` is faster but risks
cache/server divergence if your mutation response is incomplete.

---

## Pagination Patterns

### Offset-based pagination

```tsx
function PaginatedPosts() {
  const [page, setPage] = useState(1);

  const { data, isLoading, isPlaceholderData } = useQuery({
    queryKey: ['posts', { page }],
    queryFn: () => fetchPosts({ page, limit: 20 }),
    placeholderData: keepPreviousData,  // show old page while loading new
  });

  return (
    <div>
      {data?.posts.map(post => <PostCard key={post.id} post={post} />)}

      <div style={{ opacity: isPlaceholderData ? 0.5 : 1 }}>
        <button
          onClick={() => setPage(p => p - 1)}
          disabled={page === 1}
        >
          Previous
        </button>
        <span>Page {page}</span>
        <button
          onClick={() => setPage(p => p + 1)}
          disabled={!data?.hasNextPage}
        >
          Next
        </button>
      </div>
    </div>
  );
}
```

### Cursor-based infinite scroll

```tsx
function InfinitePosts() {
  const {
    data,
    fetchNextPage,
    hasNextPage,
    isFetchingNextPage,
    isLoading,
  } = useInfiniteQuery({
    queryKey: ['posts', 'infinite'],
    queryFn: ({ pageParam }) =>
      fetchPosts({ cursor: pageParam, limit: 20 }),
    initialPageParam: undefined as string | undefined,
    getNextPageParam: (lastPage) => lastPage.nextCursor,
  });

  // Intersection Observer for auto-loading
  const observerRef = useRef<IntersectionObserver>();
  const loadMoreRef = useCallback((node: HTMLDivElement | null) => {
    if (observerRef.current) observerRef.current.disconnect();
    observerRef.current = new IntersectionObserver(entries => {
      if (entries[0].isIntersecting && hasNextPage && !isFetchingNextPage) {
        fetchNextPage();
      }
    });
    if (node) observerRef.current.observe(node);
  }, [hasNextPage, isFetchingNextPage, fetchNextPage]);

  const allPosts = data?.pages.flatMap(page => page.posts) ?? [];

  return (
    <div>
      {allPosts.map(post => <PostCard key={post.id} post={post} />)}
      <div ref={loadMoreRef}>
        {isFetchingNextPage ? <Spinner /> : hasNextPage ? 'Scroll for more' : 'No more posts'}
      </div>
    </div>
  );
}
```

**Cursor vs offset:** Cursor-based pagination is more reliable for
real-time data. With offset pagination, if a new item is added while
the user is on page 2, page 3 will show a duplicate of page 2's last
item. Cursor-based: "give me items after this specific item" — no
duplication possible.

---

## Prefetching & Optimistic Patterns

### Prefetch on hover

```tsx
function PostLink({ postId, title }) {
  const queryClient = useQueryClient();

  const prefetch = () => {
    queryClient.prefetchQuery({
      queryKey: ['posts', postId],
      queryFn: () => fetchPost(postId),
      staleTime: 60 * 1000,  // don't re-prefetch within 1 minute
    });
  };

  return (
    <Link
      href={`/posts/${postId}`}
      onMouseEnter={prefetch}
      onFocus={prefetch}
    >
      {title}
    </Link>
  );
}
```

### Seed cache from list data

```tsx
// When you fetch a list, seed individual caches
const { data: users } = useQuery({
  queryKey: ['users'],
  queryFn: async () => {
    const users = await fetchUsers();
    // Seed individual user caches — navigating to /user/123 is instant
    users.forEach(user => {
      queryClient.setQueryData(['users', user.id], user);
    });
    return users;
  },
});
```

---

## TanStack Query + Server Components

### The coexistence pattern

```
Server Components: initial data fetch (SSR, no client JS)
TanStack Query:    client-side mutations, polling, optimistic updates

Page load:
  Server Component fetches data → HTML streams to client
  Client hydrates → TanStack Query takes over for interactivity
```

```tsx
// Server Component — fetches initial data
async function PostsPage() {
  const posts = await db.post.findMany({ take: 20 });
  return <PostsList initialPosts={posts} />;
}

// Client Component — TanStack Query for interactivity
'use client';
function PostsList({ initialPosts }) {
  const { data: posts } = useQuery({
    queryKey: ['posts'],
    queryFn: fetchPosts,
    initialData: initialPosts,  // skip first fetch, use server data
    staleTime: 60 * 1000,       // don't refetch for 1 minute
  });

  const deleteMutation = useMutation({
    mutationFn: deletePost,
    onSuccess: () => queryClient.invalidateQueries({ queryKey: ['posts'] }),
  });

  return posts.map(post => (
    <PostCard key={post.id} post={post} onDelete={deleteMutation.mutate} />
  ));
}
```

### Hydration pattern (preferred)

```tsx
// layout.tsx or page.tsx — Server Component
import { dehydrate, HydrationBoundary, QueryClient } from '@tanstack/react-query';

async function PostsPage() {
  const queryClient = new QueryClient();

  await queryClient.prefetchQuery({
    queryKey: ['posts'],
    queryFn: fetchPosts,
  });

  return (
    <HydrationBoundary state={dehydrate(queryClient)}>
      <PostsList />  {/* Client Component — reads from hydrated cache */}
    </HydrationBoundary>
  );
}

// PostsList.tsx — Client Component
'use client';
function PostsList() {
  // This reads from the hydrated cache — no loading state on first render
  const { data: posts } = useQuery({
    queryKey: ['posts'],
    queryFn: fetchPosts,
  });
  // ...
}
```

**Why hydration > initialData:** With `initialData`, the data lives outside
TanStack Query until hydration. With `HydrationBoundary`, the data is
pre-loaded into TQ's cache, so `dataUpdatedAt` is set correctly, staleTime
works from the server fetch time, and background refetches are properly
scheduled.

---

## Interview Questions

---

**Q: What problems does TanStack Query solve that you can't easily solve
with useState + useEffect?**

**How to answer:** Don't just list features. Describe the bugs that
occur WITHOUT TanStack Query.

**Answer:**

**What most candidates say:** "Caching, deduplication, and automatic
refetching." This is a feature list, not understanding.

**What a senior candidate says:** TanStack Query solves an entire class
of bugs that are almost impossible to avoid with manual data fetching:

1. **Stale data after navigation** — user views a list, opens a detail
   page, goes back. With useState: the list refetches from scratch
   (loading spinner). With TQ: cached data shown instantly, background
   refresh if stale.

2. **Duplicate requests** — three components need user data. With
   useState: three independent fetches, three loading states, three
   error states. If one fails and two succeed, you have inconsistent UI.
   With TQ: one fetch, shared cache, consistent state.

3. **Background staleness** — user leaves a tab open for an hour. The
   data is now stale. With useState: nothing happens, user sees old data.
   With TQ: refetch on window focus, refetch on reconnect, configurable
   polling.

4. **Race conditions** — user types quickly in a search, triggering
   fetch A then fetch B. B resolves first, then A overwrites with old
   data. With TQ: query keys handle this automatically. When the key
   changes, TQ cancels the old query's state update.

5. **Optimistic update rollback** — user deletes an item, you remove
   it from the UI instantly. Server fails. With useState: complex manual
   rollback logic. With TQ: `onMutate` snapshots, `onError` rollbacks,
   `onSettled` refetches — a declarative pattern.

The fundamental insight: **server state is a fundamentally different
problem than UI state**. It requires caching, synchronization, and
invalidation — problems that have been solved by libraries but are
almost impossible to solve correctly ad-hoc.

---

**Q: How do you handle loading states in TanStack Query?**

**How to answer:** Distinguish between first-load and background
refetch. Show the UX difference.

**Answer:**

**What most candidates say:** "Check isLoading and show a spinner."

**What a senior candidate says:** There are two types of loading, and
confusing them destroys UX:

**`isLoading`** — no cached data exists. First visit ever. Show a full
skeleton/spinner. The user has nothing to look at.

**`isFetching`** — a fetch is in progress (could be background refetch).
Cached data IS available. Show the cached data with a subtle loading
indicator (spinner in the corner, dimmed content). Do NOT replace the
content with a skeleton.

```tsx
if (isLoading) return <FullSkeleton />;        // first load
return (
  <div style={{ opacity: isFetching ? 0.7 : 1 }}>  {/* background fetch */}
    <UserProfile user={data} />
    {isFetching && <SmallSpinner />}           {/* subtle indicator */}
  </div>
);
```

**`placeholderData: keepPreviousData`** — for pagination. When changing
pages, shows the current page at reduced opacity while the next page
loads. The user never sees a blank loading state between pages.

**The production insight:** Users hate loading spinners on return visits.
If they saw the data 30 seconds ago and come back, show it instantly
from cache. The background refetch ensures freshness without the user
ever seeing a spinner. This is the "stale-while-revalidate" pattern —
the same strategy CDNs use.

---

**Q: When would you still use TanStack Query in a Next.js App Router
project that uses Server Components?**

**How to answer:** Show that RSCs handle reads but TQ handles the
interactive parts.

**Answer:**

**What most candidates say:** "Server Components handle data fetching,
so you don't need TanStack Query." This is only half right.

**What a senior candidate says:** Server Components are excellent for
**initial data fetching** — zero client JS, streaming, close to the
database. But they can't handle:

1. **Client-side mutations with optimistic UI** — deleting a post,
   adding a comment, toggling a like. You need instant UI feedback with
   rollback on failure. Server Actions handle the mutation, but TQ
   handles the cache update and rollback.

2. **Polling / real-time updates** — TQ's `refetchInterval` polls every
   N seconds. Server Components only run on navigation.

3. **Infinite scroll / pagination on the client** — after the initial
   page loads, subsequent pages are client-driven. TQ's
   `useInfiniteQuery` handles cursor management, page caching, and
   loading states.

4. **Cross-component cache sharing** — multiple Client Components
   that need the same data. TQ deduplicates and shares the cache.
   Without it, each component would need to receive data via props from
   a Server Component parent.

5. **Offline support** — TQ's `persistQueryClient` saves the cache to
   localStorage. The app works offline with cached data and syncs when
   reconnected.

**The pattern:** Use Server Components for the initial render (no loading
state, SSR, SEO). Use TanStack Query for anything that needs to change
after the initial render (mutations, pagination, polling, optimistic
updates). Pass the server-fetched data as `initialData` or use
`HydrationBoundary` so TQ doesn't refetch what the server already
fetched.

---

**Q: Explain staleTime vs gcTime in TanStack Query.**

**How to answer:** Define both, then show the lifecycle of a cache entry
through time.

**Answer:**

**What most candidates say:** "staleTime is how long data is fresh,
gcTime is how long it stays in cache."

**What a senior candidate says:** These control two different lifecycle
phases of cached data:

**`staleTime`** (default: 0) — how long after a fetch the data is
considered "fresh." Fresh data is never automatically refetched. Set this
based on how frequently your data changes:

```
staleTime: 0        → always stale, refetch on every mount/focus (real-time data)
staleTime: 5min     → fresh for 5 minutes, then background refetch (most APIs)
staleTime: Infinity → never stale, only refetch on manual invalidation (static config)
```

**`gcTime`** (default: 5min) — how long the cache entry survives after
the last subscriber unmounts. When no component is subscribed to a query,
the cache entry starts a gcTime countdown. If a new subscriber appears
before the timer expires, the cached data is immediately available.

```
Timeline:
t=0:    Component mounts → fetch → cache data ← staleTime starts
t=1min: Data goes stale (staleTime: 1min)
t=2min: Component unmounts ← gcTime starts (5min countdown)
t=3min: New component mounts → cache hit! Data shown instantly
        (stale, so background refetch triggered)
t=7min: Component unmounts again ← gcTime restarts
t=12min: gcTime expires → cache entry garbage collected
t=13min: New component mounts → full fetch (no cache)
```

**The most common misconfiguration:** Setting `staleTime: Infinity` but
`gcTime: 5min`. The data is "never stale" but gets garbage collected after
5 minutes without subscribers. The user navigates away for 6 minutes,
comes back, and sees a loading spinner. If you want permanent caching,
set BOTH to Infinity (and use manual `invalidateQueries` after mutations).

**The production default I use:**
```tsx
const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      staleTime: 60 * 1000,       // 1 minute — good balance
      gcTime: 5 * 60 * 1000,      // 5 minutes
      retry: 2,
      refetchOnWindowFocus: true,
    },
  },
});
```
