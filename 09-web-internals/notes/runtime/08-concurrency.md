# Concurrency in JavaScript — Models, Patterns, and Pitfalls

## What Concurrency Means in JavaScript

Concurrency is doing multiple things in overlapping time periods.
Parallelism is doing multiple things at the exact same instant.

JavaScript's main thread is **concurrent but not parallel** — it
interleaves work via the event loop but only executes one piece of
JavaScript at a time. True parallelism requires Web Workers or
Node.js worker_threads (covered in 04-workers.md).

Understanding concurrency in JavaScript means understanding:
- How the single-threaded event loop creates the illusion of
  simultaneous operations
- What "non-blocking" actually means at the implementation level
- The concurrency primitives the language provides
- How to coordinate multiple asynchronous operations correctly

---

## The Concurrency Model

### Cooperative Multitasking

JavaScript uses **cooperative multitasking** — each task runs to
completion before the next one starts. Tasks voluntarily yield
control by reaching the end of their synchronous execution or
by awaiting an asynchronous operation.

```
Preemptive (OS threads):        Cooperative (JS event loop):
┌──────┐ ┌──────┐              ┌──────────────┐
│Task A│ │Task B│              │Task A (full) │
│      │ │      │              ├──────────────┤
│  ↕   │ │  ↕   │              │Task B (full) │
│      │ │      │              ├──────────────┤
│      │ │      │              │Task A cont.  │
└──────┘ └──────┘              └──────────────┘
  OS preempts at any point      Tasks run to completion
  Race conditions possible      No mid-task interruption
```

Consequence: you cannot have a data race on JavaScript values within
a single thread. If a function is modifying an object, no other
function can observe a half-modified state. This is JavaScript's
concurrency safety guarantee.

```javascript
// This is ALWAYS safe in single-threaded JS:
let counter = 0;

async function increment() {
  const current = counter;  // Read
  counter = current + 1;    // Write
  // No other code runs between read and write
}

// Even with concurrent calls:
await Promise.all([increment(), increment(), increment()]);
// counter is ALWAYS 3 — no race condition
```

But with actual concurrency (SharedArrayBuffer + workers), races
exist — covered in 04-workers.md.

### Run-to-Completion Semantics

A function's synchronous code runs without interruption:

```javascript
let state = 'initial';

async function taskA() {
  state = 'A started';
  // No other task can see state = 'A started' and change it
  // before the next line runs
  await fetch('/api');  // ← YIELD POINT: other tasks can run now
  // After await, state may have changed!
  console.log(state); // Could be anything
  state = 'A done';
}

async function taskB() {
  state = 'B started';
  await fetch('/api');
  state = 'B done';
}

// The yield points are:
// - await
// - .then() / .catch() / .finally()
// - setTimeout / setInterval
// - requestAnimationFrame
// - Any callback registration
```

**Between yield points, your code has exclusive access to all
mutable state.** After a yield point, assume all shared state
may have changed.

---

## Concurrency Primitives

### Promise — The Fundamental Unit

A Promise represents a value that may not exist yet. It's the
foundation of all JavaScript concurrency:

```
States:
  pending  ──resolve(value)──▶  fulfilled (with value)
    │
    └──reject(reason)──▶  rejected (with reason)

Once settled (fulfilled or rejected), a Promise NEVER changes state.
```

Promise internals (V8):
```
PromiseCapability {
  promise:  the Promise object
  resolve:  function to fulfill the promise
  reject:   function to reject the promise
}

PromiseReaction {
  capability:  the dependent PromiseCapability
  type:        FULFILL or REJECT
  handler:     the .then() or .catch() callback
}
```

When a promise settles, V8 enqueues its PromiseReaction handlers
as microtasks. The handlers don't run immediately — they run at the
next microtask checkpoint.

### Promise Combinators

Four built-in combinators for coordinating multiple promises:

```
                   Short-circuit on   Short-circuit on
                   first fulfill?     first reject?        Returns
                   ───────────────    ────────────────     ─────────
Promise.all        No                 YES (fail-fast)      Array of values
Promise.allSettled No                 No                   Array of {status, value/reason}
Promise.race       YES                YES                  First settled value
Promise.any        YES                No                   First fulfilled value

                   Resolves when          Rejects when
                   ──────────────         ────────────
Promise.all        ALL fulfill            ANY rejects
Promise.allSettled ALL settle             Never (always fulfills)
Promise.race       ANY settles            ANY settles (with rejection)
Promise.any        ANY fulfills           ALL reject (AggregateError)
```

### Mutex-like Patterns (Single-Threaded)

Even in single-threaded code, you sometimes need mutual exclusion
across asynchronous operations:

```javascript
class AsyncMutex {
  #locked = false;
  #queue = [];

  async acquire() {
    if (!this.#locked) {
      this.#locked = true;
      return;
    }

    // Wait for our turn
    await new Promise(resolve => this.#queue.push(resolve));
  }

  release() {
    if (this.#queue.length > 0) {
      // Wake the next waiter
      const next = this.#queue.shift();
      // Use queueMicrotask to avoid releasing in the same microtask
      queueMicrotask(next);
    } else {
      this.#locked = false;
    }
  }

  async withLock(fn) {
    await this.acquire();
    try {
      return await fn();
    } finally {
      this.release();
    }
  }
}

// Usage:
const mutex = new AsyncMutex();

async function criticalSection() {
  await mutex.withLock(async () => {
    const data = await readFromDB();
    await writeToAPI(transform(data));
    // No other criticalSection() call runs between read and write
  });
}
```

Why you'd need this: if `readFromDB()` and `writeToAPI()` need to be
atomic (no other task should read-modify-write between them), the
`await` between them creates a yield point where another call could
interleave.

### Semaphore — Bounded Concurrency

```javascript
class Semaphore {
  #count;
  #queue = [];

  constructor(maxConcurrency) {
    this.#count = maxConcurrency;
  }

  async acquire() {
    if (this.#count > 0) {
      this.#count--;
      return;
    }
    await new Promise(resolve => this.#queue.push(resolve));
  }

  release() {
    if (this.#queue.length > 0) {
      const next = this.#queue.shift();
      queueMicrotask(next);
    } else {
      this.#count++;
    }
  }

  async withPermit(fn) {
    await this.acquire();
    try {
      return await fn();
    } finally {
      this.release();
    }
  }
}

// Limit to 5 concurrent HTTP requests:
const semaphore = new Semaphore(5);

async function fetchWithLimit(url) {
  return semaphore.withPermit(() => fetch(url));
}

const results = await Promise.all(
  urls.map(url => fetchWithLimit(url))
);
```

---

## Concurrency Patterns

### Pattern 1: Fan-Out / Fan-In

Launch N operations, collect all results:

```javascript
// Fan-out: launch all at once
const promises = urls.map(url => fetch(url).then(r => r.json()));

// Fan-in: collect results
const results = await Promise.all(promises);
```

This is the most common concurrency pattern. All operations run
concurrently (their I/O overlaps), and we wait for all to complete.

### Pattern 2: Pipeline / Waterfall

Each step depends on the previous:

```javascript
async function pipeline(input) {
  const a = await stepA(input);
  const b = await stepB(a);
  const c = await stepC(b);
  return c;
}
```

No concurrency here — each step waits for the previous. Use this
when steps have data dependencies.

### Pattern 3: Racing with Timeout

```javascript
function withTimeout<T>(promise: Promise<T>, ms: number): Promise<T> {
  const timeout = new Promise<never>((_, reject) =>
    setTimeout(() => reject(new Error(`Timeout after ${ms}ms`)), ms)
  );
  return Promise.race([promise, timeout]);
}

const data = await withTimeout(fetch('/api/slow'), 5000);
```

Note: the original fetch doesn't abort when the timeout wins —
it continues running. Use `AbortController` for true cancellation:

```javascript
async function fetchWithTimeout(url: string, ms: number) {
  const controller = new AbortController();
  const timeoutId = setTimeout(() => controller.abort(), ms);

  try {
    const response = await fetch(url, { signal: controller.signal });
    return response;
  } finally {
    clearTimeout(timeoutId);
  }
}
```

### Pattern 4: Retry with Backoff

```javascript
async function retry<T>(
  fn: () => Promise<T>,
  maxAttempts: number,
  baseDelay = 1000
): Promise<T> {
  for (let attempt = 1; attempt <= maxAttempts; attempt++) {
    try {
      return await fn();
    } catch (error) {
      if (attempt === maxAttempts) throw error;

      // Exponential backoff with jitter
      const delay = baseDelay * Math.pow(2, attempt - 1);
      const jitter = delay * 0.5 * Math.random();
      await new Promise(r => setTimeout(r, delay + jitter));
    }
  }
  throw new Error('unreachable');
}

const data = await retry(() => fetch('/api/flaky').then(r => r.json()), 3);
```

### Pattern 5: Concurrency Pool

Process N items with at most M concurrent operations:

```javascript
async function pool<T, R>(
  items: T[],
  fn: (item: T) => Promise<R>,
  concurrency: number
): Promise<R[]> {
  const results: R[] = new Array(items.length);
  let nextIndex = 0;

  async function worker() {
    while (nextIndex < items.length) {
      const index = nextIndex++;
      results[index] = await fn(items[index]);
    }
  }

  // Launch `concurrency` workers
  const workers = Array.from({ length: concurrency }, () => worker());
  await Promise.all(workers);

  return results;
}

// Process 1000 URLs, 10 at a time:
const results = await pool(urls, url => fetch(url), 10);
```

### Pattern 6: First Successful (with Fallback)

```javascript
async function firstSuccessful<T>(
  fns: (() => Promise<T>)[]
): Promise<T> {
  const errors: Error[] = [];

  for (const fn of fns) {
    try {
      return await fn();
    } catch (e) {
      errors.push(e as Error);
    }
  }

  throw new AggregateError(errors, 'All attempts failed');
}

// Try primary, then fallback, then cache:
const data = await firstSuccessful([
  () => fetch('/api/primary').then(r => r.json()),
  () => fetch('/api/fallback').then(r => r.json()),
  () => getCachedData(),
]);
```

For concurrent racing (not sequential fallback), use `Promise.any`:

```javascript
const data = await Promise.any([
  fetch('/api/us-east').then(r => r.json()),
  fetch('/api/eu-west').then(r => r.json()),
  fetch('/api/ap-south').then(r => r.json()),
]);
// Returns the first to succeed — hedged request pattern
```

### Pattern 7: Debounce and Coalesce

Coalesce multiple concurrent calls into one:

```javascript
function coalesce<T>(fn: () => Promise<T>): () => Promise<T> {
  let inflight: Promise<T> | null = null;

  return () => {
    if (inflight) return inflight;

    inflight = fn().finally(() => {
      inflight = null;
    });

    return inflight;
  };
}

const getUser = coalesce(() => fetch('/api/user').then(r => r.json()));

// These three calls result in ONE fetch:
const [a, b, c] = await Promise.all([getUser(), getUser(), getUser()]);
// a === b === c (same object)
```

---

## Concurrency Hazards

### Hazard 1: Lost Updates

```javascript
let balance = 100;

async function withdraw(amount) {
  const current = balance;     // Read: 100
  await validateWithServer();   // Yield point!
  balance = current - amount;   // Write: uses stale value
}

// Concurrent calls:
await Promise.all([
  withdraw(30),  // Reads 100, yields, writes 70
  withdraw(50),  // Reads 100, yields, writes 50 ← WRONG (should be 20)
]);
console.log(balance); // 50, not 20!
```

Fix: use the mutex pattern from above, or redesign to avoid
shared mutable state across yield points.

### Hazard 2: Unhandled Promise Rejections

```javascript
// DANGEROUS: rejection from p2 is unhandled if p1 rejects first
async function bad() {
  const p1 = fetch('/a');
  const p2 = fetch('/b');

  const r1 = await p1; // If this throws, p2's rejection is unhandled
  const r2 = await p2;
}

// SAFE: Promise.all handles both rejections
async function good() {
  const [r1, r2] = await Promise.all([fetch('/a'), fetch('/b')]);
}
```

When a Promise rejects and no handler is attached, the runtime emits
`unhandledrejection`. In Node.js, this can crash the process
(since Node 15, unhandled rejections are fatal by default).

### Hazard 3: Async Iterator Pitfalls

```javascript
// SEQUENTIAL — not concurrent:
for await (const item of asyncIterable) {
  await processItem(item);  // Waits for each before starting next
}

// To process concurrently, collect into array first:
const items = [];
for await (const item of asyncIterable) {
  items.push(item);
}
await Promise.all(items.map(processItem));

// Or use a concurrency pool if the iterable is large
```

### Hazard 4: Resource Leaks

```javascript
// LEAK: if first fetch fails, second fetch continues running
async function fetchBoth() {
  const a = fetch('/a');
  const b = fetch('/b');

  try {
    return await Promise.all([a, b]);
  } catch (e) {
    // b might still be running!
    // With AbortController:
    // controller.abort() would cancel both
    throw e;
  }
}

// PROPER cleanup with AbortController:
async function fetchBothSafe() {
  const controller = new AbortController();
  const { signal } = controller;

  try {
    return await Promise.all([
      fetch('/a', { signal }),
      fetch('/b', { signal }),
    ]);
  } catch (e) {
    controller.abort(); // Cancel all in-flight requests
    throw e;
  }
}
```

### Hazard 5: Thundering Herd

```javascript
// BAD: 1000 concurrent requests on page load
const allData = await Promise.all(
  userIds.map(id => fetch(`/api/users/${id}`))
);
// Server receives 1000 simultaneous requests — likely crashes or rate-limits

// GOOD: bounded concurrency
const allData = await pool(
  userIds,
  id => fetch(`/api/users/${id}`).then(r => r.json()),
  10  // Max 10 concurrent
);
```

---

## Cancellation

JavaScript has no built-in cancellation for Promises. The
`AbortController`/`AbortSignal` API is the standard pattern:

```javascript
const controller = new AbortController();
const { signal } = controller;

// Pass signal to cancellable operations:
const promise = fetch('/api/data', { signal });

// Cancel:
controller.abort();
// promise rejects with AbortError

// Check if aborted:
signal.aborted; // true
signal.reason;  // DOMException: The operation was aborted

// Listen for abort:
signal.addEventListener('abort', () => {
  cleanup();
});
```

### Making Custom Operations Cancellable

```javascript
async function cancellableWork(signal: AbortSignal) {
  for (let i = 0; i < 1000; i++) {
    // Check at yield points:
    signal.throwIfAborted();

    await processChunk(i);
  }
}

// Composing signals:
const timeout = AbortSignal.timeout(5000);        // Auto-aborts after 5s
const manual = new AbortController();

const combined = AbortSignal.any([timeout, manual.signal]);
// Aborts when EITHER signal fires

await fetch('/api/data', { signal: combined });
```

### AbortSignal.any and AbortSignal.timeout

```javascript
// Timeout signal (built-in):
const signal = AbortSignal.timeout(5000);
// Automatically aborts after 5 seconds
// Reason: TimeoutError (not AbortError)

// Combining signals:
const userCancel = new AbortController();
const signal = AbortSignal.any([
  AbortSignal.timeout(5000),
  userCancel.signal,
]);
// First one to fire wins
```

---

## Async Iterators and Generators

### Async Generators — Produce Values Over Time

```javascript
async function* fetchPages(baseUrl: string) {
  let page = 1;
  let hasMore = true;

  while (hasMore) {
    const response = await fetch(`${baseUrl}?page=${page}`);
    const data = await response.json();

    yield* data.items;  // Yield each item

    hasMore = data.hasNextPage;
    page++;
  }
}

// Consume:
for await (const item of fetchPages('/api/products')) {
  console.log(item);
  // Automatically fetches next page when current is exhausted
}
```

### ReadableStream as Async Iterable

```javascript
async function* streamToLines(stream: ReadableStream<Uint8Array>) {
  const reader = stream.getReader();
  const decoder = new TextDecoder();
  let buffer = '';

  try {
    while (true) {
      const { done, value } = await reader.read();
      if (done) break;

      buffer += decoder.decode(value, { stream: true });
      const lines = buffer.split('\n');
      buffer = lines.pop()!;

      for (const line of lines) {
        yield line;
      }
    }
    if (buffer) yield buffer;
  } finally {
    reader.releaseLock();
  }
}

// Usage:
const response = await fetch('/api/stream');
for await (const line of streamToLines(response.body!)) {
  processLine(line);
}
```

---

## Structured Concurrency

JavaScript doesn't have built-in structured concurrency (unlike
Go goroutines or Kotlin coroutines with scopes). But you can
approximate it:

```javascript
class TaskGroup {
  #promises: Promise<void>[] = [];
  #controller = new AbortController();
  #errors: Error[] = [];

  get signal() { return this.#controller.signal; }

  spawn(fn: (signal: AbortSignal) => Promise<void>) {
    const promise = fn(this.#controller.signal).catch(err => {
      this.#errors.push(err);
      this.#controller.abort(); // Cancel all siblings
    });
    this.#promises.push(promise);
  }

  async join() {
    await Promise.allSettled(this.#promises);
    if (this.#errors.length > 0) {
      throw new AggregateError(this.#errors);
    }
  }
}

// Usage:
const group = new TaskGroup();

group.spawn(async (signal) => {
  const data = await fetch('/api/users', { signal });
  await processUsers(await data.json());
});

group.spawn(async (signal) => {
  const data = await fetch('/api/orders', { signal });
  await processOrders(await data.json());
});

await group.join();
// If either task fails, the other is cancelled
// All tasks complete or all tasks fail — no orphans
```

---

## Key Takeaways

| Concept | Why It Matters |
|---------|---------------|
| Cooperative multitasking | No data races on JS values within a single thread |
| Run-to-completion | Between yield points, you have exclusive access to state |
| yield points = await/then/setTimeout | After any yield, assume shared state may have changed |
| Promise combinators | all/allSettled/race/any — know which one fits |
| Semaphore for bounded concurrency | Promise.all has no concurrency limit — build your own |
| AbortController for cancellation | The standard cancellation mechanism — compose with AbortSignal.any |
| Lost updates across awaits | Read-yield-write on shared state causes bugs |
| Unhandled rejections | Promise.all handles multiple rejections; bare await doesn't |
| Coalescing pattern | Deduplicate concurrent calls to the same resource |
| Structured concurrency | Cancel sibling tasks on failure — no orphaned promises |
