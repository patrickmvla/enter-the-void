# Asynchronous TypeScript — Types, Patterns, and the Runtime

## The Async Type System

TypeScript's type system maps precisely onto JavaScript's async
runtime. Every `async` function returns a `Promise<T>`. Every `await`
unwraps a `Promise<T>` into `T`. The compiler enforces correctness
at compile time; the runtime enforces scheduling at execution time.

Understanding async TypeScript means understanding both the **type
machinery** (what the compiler checks) and the **runtime behavior**
(what actually executes).

---

## Promise Types — The Foundation

### The Promise<T> Interface

```typescript
interface Promise<T> {
  then<TResult1 = T, TResult2 = never>(
    onfulfilled?: ((value: T) => TResult1 | PromiseLike<TResult1>) | null,
    onrejected?: ((reason: any) => TResult2 | PromiseLike<TResult2>) | null
  ): Promise<TResult1 | TResult2>;

  catch<TResult = never>(
    onrejected?: ((reason: any) => TResult | PromiseLike<TResult>) | null
  ): Promise<T | TResult>;

  finally(onfinally?: (() => void) | null): Promise<T>;
}
```

Key typing insights:

1. **`then` returns a new Promise** whose type depends on the
   return type of the callbacks
2. **`catch` widens the type** — `Promise<T>.catch(handler)` returns
   `Promise<T | TResult>` because the catch handler's return type
   is a valid alternative
3. **`finally` preserves the type** — the callback doesn't receive
   the value and its return type is ignored (unless it throws)

### PromiseLike<T> — The Thenable Interface

```typescript
interface PromiseLike<T> {
  then<TResult1 = T, TResult2 = never>(
    onfulfilled?: ((value: T) => TResult1 | PromiseLike<TResult1>) | null,
    onrejected?: ((reason: any) => TResult2 | PromiseLike<TResult2>) | null
  ): PromiseLike<TResult1 | TResult2>;
}
```

`PromiseLike` is the minimal interface — any object with a `.then()`
method. This is how TypeScript handles interop with non-native Promise
implementations (Bluebird, jQuery Deferred, etc.).

```typescript
// Accepts any thenable, not just native Promises:
function process(input: PromiseLike<string>): Promise<number> {
  return Promise.resolve(input).then(s => s.length);
}
```

### Awaited<T> — Recursive Promise Unwrapping

```typescript
// TypeScript 4.5+ built-in utility type:
type Awaited<T> =
  T extends null | undefined ? T :
  T extends object & { then(onfulfilled: infer F, ...args: infer _): any } ?
    F extends ((value: infer V, ...args: infer _) => any) ?
      Awaited<V> :    // Recursively unwrap
      never :
  T;

// Usage:
type A = Awaited<Promise<string>>;                    // string
type B = Awaited<Promise<Promise<number>>>;           // number (unwraps nested)
type C = Awaited<string>;                             // string (non-promise passes through)
type D = Awaited<Promise<Promise<Promise<boolean>>>>; // boolean (deep unwrap)
```

`Awaited<T>` is what `await` actually does at the type level. It
recursively unwraps nested Promises/thenables until it reaches a
non-thenable type.

---

## Async Functions — The Type Transformation

### Return Type Inference

```typescript
// TypeScript infers Promise<string>:
async function getName() {
  return "Alice";
  // Return type: Promise<string>, not string
  // The async keyword wraps the return in Promise automatically
}

// Explicit annotation must use Promise:
async function getName(): Promise<string> {
  return "Alice"; // TS accepts string, wraps it in Promise
}

// ERROR: async function cannot return non-Promise:
// async function getName(): string { ... }
// → Type 'string' is not a valid async function return type
```

### Await Type Narrowing

```typescript
async function process() {
  const response = await fetch('/api');  // Response
  const data = await response.json();    // any (json() returns Promise<any>)

  // Type-safe version:
  interface User { name: string; age: number }
  const user = await response.json() as User;
  // or with validation:
  const user = userSchema.parse(await response.json()); // zod
}
```

### The void vs undefined Return

```typescript
// These are different:
async function returnsVoid(): Promise<void> {
  // void means "I don't return a meaningful value"
  // Can have no return, return;, or return undefined;
}

async function returnsUndefined(): Promise<undefined> {
  return undefined; // Must explicitly return undefined
}

// Why it matters:
const fn: () => Promise<void> = async () => { return 42; };
// This is ALLOWED — void return type accepts any value
// (the value is simply ignored by the caller)
```

---

## Error Handling Types

### The Problem with `catch`

JavaScript's `catch` gives you `unknown` (or `any` in older TS):

```typescript
try {
  await riskyOperation();
} catch (error) {
  // error is `unknown` (TypeScript 4.4+ with useUnknownInCatchVariables)
  // error is `any` (older TS default)

  // Must narrow before using:
  if (error instanceof Error) {
    console.log(error.message); // OK
  }

  // Or assert:
  const err = error as Error;
}
```

### Typed Error Handling Patterns

```typescript
// Result type (Rust-inspired):
type Result<T, E = Error> =
  | { ok: true; value: T }
  | { ok: false; error: E };

async function safeAsync<T>(
  fn: () => Promise<T>
): Promise<Result<T>> {
  try {
    return { ok: true, value: await fn() };
  } catch (error) {
    return {
      ok: false,
      error: error instanceof Error ? error : new Error(String(error))
    };
  }
}

// Usage:
const result = await safeAsync(() => fetch('/api').then(r => r.json()));
if (result.ok) {
  console.log(result.value); // typed!
} else {
  console.error(result.error.message);
}
```

```typescript
// Tuple pattern [error, value]:
async function to<T>(promise: Promise<T>): Promise<[Error, null] | [null, T]> {
  try {
    return [null, await promise];
  } catch (error) {
    return [error instanceof Error ? error : new Error(String(error)), null];
  }
}

const [err, data] = await to(fetch('/api'));
if (err) {
  // handle error
} else {
  // data is typed
}
```

### Custom Error Classes

```typescript
class HttpError extends Error {
  constructor(
    public readonly status: number,
    public readonly statusText: string,
    message?: string
  ) {
    super(message ?? `HTTP ${status}: ${statusText}`);
    this.name = 'HttpError';
  }
}

class TimeoutError extends Error {
  constructor(public readonly ms: number) {
    super(`Operation timed out after ${ms}ms`);
    this.name = 'TimeoutError';
  }
}

class RetryExhaustedError extends Error {
  constructor(
    public readonly attempts: number,
    public readonly lastError: Error
  ) {
    super(`Failed after ${attempts} attempts: ${lastError.message}`);
    this.name = 'RetryExhaustedError';
  }
}

// Discriminated error handling:
async function fetchData(url: string) {
  try {
    return await fetchWithRetry(url);
  } catch (error) {
    if (error instanceof HttpError && error.status === 404) {
      return null; // Not found is OK
    }
    if (error instanceof TimeoutError) {
      return getCachedFallback(url);
    }
    if (error instanceof RetryExhaustedError) {
      reportToMonitoring(error);
    }
    throw error; // Re-throw unknown errors
  }
}
```

---

## Generic Async Utilities

### Type-Safe Async Wrappers

```typescript
// Timeout wrapper with proper generics:
function withTimeout<T>(
  promise: Promise<T>,
  ms: number,
  message = `Timeout after ${ms}ms`
): Promise<T> {
  return new Promise<T>((resolve, reject) => {
    const timer = setTimeout(() => reject(new TimeoutError(ms)), ms);

    promise.then(
      value => { clearTimeout(timer); resolve(value); },
      error => { clearTimeout(timer); reject(error); }
    );
  });
}

// Retry with proper generics:
async function retry<T>(
  fn: () => Promise<T>,
  options: {
    maxAttempts: number;
    delay: number;
    backoff?: 'linear' | 'exponential';
    shouldRetry?: (error: unknown) => boolean;
  }
): Promise<T> {
  const { maxAttempts, delay, backoff = 'exponential', shouldRetry } = options;
  let lastError: unknown;

  for (let attempt = 1; attempt <= maxAttempts; attempt++) {
    try {
      return await fn();
    } catch (error) {
      lastError = error;
      if (shouldRetry && !shouldRetry(error)) throw error;
      if (attempt === maxAttempts) break;

      const wait = backoff === 'exponential'
        ? delay * Math.pow(2, attempt - 1)
        : delay * attempt;
      await new Promise(r => setTimeout(r, wait));
    }
  }

  throw new RetryExhaustedError(
    maxAttempts,
    lastError instanceof Error ? lastError : new Error(String(lastError))
  );
}
```

### Typed Concurrency Pool

```typescript
async function pool<TItem, TResult>(
  items: TItem[],
  fn: (item: TItem, index: number) => Promise<TResult>,
  concurrency: number
): Promise<TResult[]> {
  const results = new Array<TResult>(items.length);
  let nextIndex = 0;

  async function worker(): Promise<void> {
    while (nextIndex < items.length) {
      const index = nextIndex++;
      results[index] = await fn(items[index], index);
    }
  }

  await Promise.all(
    Array.from({ length: Math.min(concurrency, items.length) }, worker)
  );

  return results;
}

// Fully typed:
const users: User[] = await pool(
  userIds,                                    // string[]
  (id) => fetch(`/api/users/${id}`).then(r => r.json() as Promise<User>),
  5
);
// TypeScript infers TItem = string, TResult = User
```

### Deferred Promise

```typescript
interface Deferred<T> {
  promise: Promise<T>;
  resolve: (value: T | PromiseLike<T>) => void;
  reject: (reason?: unknown) => void;
}

function deferred<T>(): Deferred<T> {
  let resolve!: (value: T | PromiseLike<T>) => void;
  let reject!: (reason?: unknown) => void;

  const promise = new Promise<T>((res, rej) => {
    resolve = res;
    reject = rej;
  });

  return { promise, resolve, reject };
}

// Usage — externally resolvable promise:
const gate = deferred<void>();

// Somewhere else:
gate.resolve(); // Unblocks all awaiters

// Waiters:
await gate.promise;
```

With `Promise.withResolvers()` (ES2024):

```typescript
const { promise, resolve, reject } = Promise.withResolvers<string>();
// Built-in — no need for the deferred helper above
```

---

## Async Generators in TypeScript

### Typing Async Generators

```typescript
// AsyncGenerator<Yield, Return, Next>
async function* paginate(url: string): AsyncGenerator<Item[], void, undefined> {
  let page = 1;
  let hasMore = true;

  while (hasMore) {
    const response = await fetch(`${url}?page=${page}`);
    const data: PaginatedResponse = await response.json();

    yield data.items; // Yield type: Item[]

    hasMore = data.hasNextPage;
    page++;
  }
  // Return type: void (implicit)
}

// Consuming:
for await (const items of paginate('/api/products')) {
  // items: Item[]
  process(items);
}
```

### AsyncIterable and AsyncIterator

```typescript
interface AsyncIterable<T> {
  [Symbol.asyncIterator](): AsyncIterator<T>;
}

interface AsyncIterator<T, TReturn = any, TNext = any> {
  next(...[value]: [] | [TNext]): Promise<IteratorResult<T, TReturn>>;
  return?(value?: TReturn): Promise<IteratorResult<T, TReturn>>;
  throw?(e?: any): Promise<IteratorResult<T, TReturn>>;
}

interface IteratorResult<T, TReturn = any> {
  done: boolean;
  value: T;  // or TReturn when done = true
}
```

### Implementing AsyncIterable

```typescript
class EventStream<T> implements AsyncIterable<T> {
  private queue: T[] = [];
  private waiting: ((value: IteratorResult<T>) => void) | null = null;
  private done = false;

  push(value: T): void {
    if (this.waiting) {
      this.waiting({ done: false, value });
      this.waiting = null;
    } else {
      this.queue.push(value);
    }
  }

  end(): void {
    this.done = true;
    if (this.waiting) {
      this.waiting({ done: true, value: undefined as any });
      this.waiting = null;
    }
  }

  [Symbol.asyncIterator](): AsyncIterator<T> {
    return {
      next: (): Promise<IteratorResult<T>> => {
        if (this.queue.length > 0) {
          return Promise.resolve({
            done: false,
            value: this.queue.shift()!,
          });
        }
        if (this.done) {
          return Promise.resolve({ done: true, value: undefined as any });
        }
        return new Promise(resolve => {
          this.waiting = resolve;
        });
      },
    };
  }
}

// Usage:
const stream = new EventStream<string>();

// Producer:
stream.push('hello');
stream.push('world');
stream.end();

// Consumer:
for await (const msg of stream) {
  console.log(msg); // 'hello', 'world'
}
```

---

## Disposable Resources — using and await using

TypeScript 5.2+ supports the Explicit Resource Management proposal:

```typescript
// Symbol.dispose for sync cleanup:
class FileHandle implements Disposable {
  [Symbol.dispose](): void {
    this.close();
  }
}

// Symbol.asyncDispose for async cleanup:
class DatabaseConnection implements AsyncDisposable {
  async [Symbol.asyncDispose](): Promise<void> {
    await this.disconnect();
  }
}

// Usage with 'using' (sync) and 'await using' (async):
async function processData() {
  await using db = await connectToDatabase();
  // db is automatically disposed when scope exits
  // Even if an exception is thrown

  using file = openFile('data.txt');
  // file is sync-disposed when scope exits

  const data = file.read();
  await db.insert(data);
  // At scope exit: db.asyncDispose() then file.dispose()
}
```

Disposal order is **reverse** declaration order (like destructors
in C++). This is critical for proper cleanup of dependent resources.

```typescript
async function transaction() {
  await using conn = await pool.getConnection();
  await using tx = await conn.beginTransaction();

  await tx.execute('INSERT ...');
  await tx.execute('UPDATE ...');

  await tx.commit();
  // Scope exit: tx disposed first (rollback if not committed),
  // then conn disposed (returned to pool)
}
```

---

## Advanced Patterns

### Type-Safe Event Emitter

```typescript
type EventMap = {
  data: [payload: Buffer];
  error: [error: Error];
  close: [];
};

class TypedEmitter<Events extends Record<string, unknown[]>> {
  private listeners = new Map<keyof Events, Set<Function>>();

  on<K extends keyof Events>(
    event: K,
    handler: (...args: Events[K]) => void
  ): this {
    if (!this.listeners.has(event)) {
      this.listeners.set(event, new Set());
    }
    this.listeners.get(event)!.add(handler);
    return this;
  }

  emit<K extends keyof Events>(event: K, ...args: Events[K]): void {
    this.listeners.get(event)?.forEach(fn => fn(...args));
  }

  // Convert event to async iterable:
  async *[Symbol.asyncIterator]<K extends keyof Events>(
    event: K
  ): AsyncGenerator<Events[K]> {
    const queue: Events[K][] = [];
    let resolve: (() => void) | null = null;

    this.on(event, ((...args: Events[K]) => {
      queue.push(args);
      resolve?.();
    }) as any);

    while (true) {
      if (queue.length === 0) {
        await new Promise<void>(r => { resolve = r; });
      }
      yield queue.shift()!;
    }
  }
}

// Usage:
const emitter = new TypedEmitter<EventMap>();
emitter.on('data', (payload) => {
  // payload: Buffer — fully typed
});
emitter.on('error', (error) => {
  // error: Error — fully typed
});
```

### Async State Machine

```typescript
type State = 'idle' | 'loading' | 'success' | 'error';

type StateTransition = {
  idle: 'loading';
  loading: 'success' | 'error';
  success: 'idle' | 'loading';
  error: 'idle' | 'loading';
};

class AsyncStateMachine<T> {
  private state: State = 'idle';
  private data: T | null = null;
  private error: Error | null = null;

  private transition(to: StateTransition[typeof this.state]): void {
    this.state = to;
  }

  async execute(fn: () => Promise<T>): Promise<T> {
    if (this.state === 'loading') {
      throw new Error('Already loading');
    }

    this.transition('loading');

    try {
      this.data = await fn();
      this.transition('success');
      return this.data;
    } catch (e) {
      this.error = e instanceof Error ? e : new Error(String(e));
      this.transition('error');
      throw this.error;
    }
  }
}
```

### Typed Middleware Chain

```typescript
type Middleware<TContext> = (
  ctx: TContext,
  next: () => Promise<void>
) => Promise<void>;

class Pipeline<TContext> {
  private middlewares: Middleware<TContext>[] = [];

  use(fn: Middleware<TContext>): this {
    this.middlewares.push(fn);
    return this;
  }

  async execute(ctx: TContext): Promise<void> {
    let index = 0;

    const next = async (): Promise<void> => {
      if (index >= this.middlewares.length) return;
      const fn = this.middlewares[index++];
      await fn(ctx, next);
    };

    await next();
  }
}

// Usage:
interface RequestContext {
  url: string;
  headers: Record<string, string>;
  body?: unknown;
  response?: { status: number; data: unknown };
}

const pipeline = new Pipeline<RequestContext>();

pipeline.use(async (ctx, next) => {
  const start = performance.now();
  await next();
  console.log(`${ctx.url}: ${performance.now() - start}ms`);
});

pipeline.use(async (ctx, next) => {
  ctx.headers['Authorization'] = `Bearer ${getToken()}`;
  await next();
});

pipeline.use(async (ctx, next) => {
  const response = await fetch(ctx.url, { headers: ctx.headers });
  ctx.response = {
    status: response.status,
    data: await response.json(),
  };
});
```

---

## Compiler Options That Affect Async

```jsonc
{
  "compilerOptions": {
    // Target affects async emit:
    "target": "ES2017",
    // ES2015: async/await compiled to __awaiter + generators
    // ES2017+: native async/await (no transform)
    // ES2018+: async iterators (for-await-of) native
    // ES2022+: top-level await

    // Unknown catch variables:
    "useUnknownInCatchVariables": true,
    // catch(e) → e is 'unknown' instead of 'any'

    // Strict promise checks:
    "strictNullChecks": true,
    // Required for proper Promise<T | null> handling

    // Top-level await:
    "module": "es2022",
    // Required for top-level await in modules
  }
}
```

### The __awaiter Transform (pre-ES2017 targets)

When targeting ES2015, TypeScript compiles `async`/`await` into
generator functions:

```typescript
// Source:
async function fetchUser(id: string): Promise<User> {
  const response = await fetch(`/api/users/${id}`);
  return response.json();
}

// Compiled (ES2015 target):
function fetchUser(id) {
  return __awaiter(this, void 0, void 0, function* () {
    const response = yield fetch(`/api/users/${id}`);
    return response.json();
  });
}
```

`__awaiter` wraps a generator and drives it by calling `.next()`
with resolved values. Each `yield` is an `await` point.

---

## Common TypeScript Async Mistakes

### Mistake 1: Forgetting to Await

```typescript
// BUG: returns Promise<boolean>, not boolean
async function isValid(data: Data): Promise<boolean> {
  return validate(data); // validate is async
}

// This comparison is always true:
if (isValid(data)) { ... }
// if (Promise) is truthy — objects are always truthy!

// Fix:
if (await isValid(data)) { ... }
```

TypeScript's `@typescript-eslint/no-floating-promises` catches this.

### Mistake 2: Async in Array Methods

```typescript
// BUG: forEach doesn't await
items.forEach(async (item) => {
  await processItem(item); // Fire-and-forget!
});
// Execution continues here before any processItem finishes

// Fix — use for...of:
for (const item of items) {
  await processItem(item); // Sequential
}

// Or Promise.all for concurrent:
await Promise.all(items.map(item => processItem(item)));
```

`forEach`, `map`, `filter`, `reduce` don't understand async
callbacks. They call the callback but ignore the returned Promise.

### Mistake 3: Redundant Async

```typescript
// UNNECESSARY: async adds overhead for no benefit
async function getUser(id: string): Promise<User> {
  return fetchUser(id); // Already returns Promise
}

// BETTER: just return the promise directly
function getUser(id: string): Promise<User> {
  return fetchUser(id);
}

// EXCEPTION: async IS needed if you want try/catch:
async function getUser(id: string): Promise<User> {
  try {
    return await fetchUser(id);
  } catch {
    return defaultUser;
  }
}
// Note: 'return await' IS needed here — without await,
// the catch block wouldn't catch fetchUser's rejection
```

### Mistake 4: Promise Constructor Antipattern

```typescript
// BAD: unnecessary Promise wrapper
function getUser(id: string): Promise<User> {
  return new Promise(async (resolve, reject) => {
    try {
      const response = await fetch(`/api/users/${id}`);
      resolve(await response.json());
    } catch (e) {
      reject(e);
    }
  });
}

// GOOD: just use async directly
async function getUser(id: string): Promise<User> {
  const response = await fetch(`/api/users/${id}`);
  return response.json();
}
```

The antipattern hides errors — if the async executor throws
synchronously, the rejection may be swallowed.

---

## Key Takeaways

| Concept | Why It Matters |
|---------|---------------|
| `Awaited<T>` recursively unwraps | Matches what `await` does at runtime — nested promises flatten |
| `PromiseLike<T>` for thenable interop | Accept any thenable, not just native Promises |
| `catch` gives `unknown` | Use type narrowing — `instanceof Error` or Result types |
| `async` always returns `Promise<T>` | Even `return "hello"` becomes `Promise<string>` |
| forEach ignores async | Use `for...of` or `Promise.all(items.map(...))` |
| `return await` in try/catch | Required — bare `return` doesn't catch rejections |
| `await using` for cleanup | Automatic resource disposal in reverse order |
| Promise.withResolvers() | Built-in deferred pattern — externally resolvable promises |
| Target affects emit | ES2017+ = native async/await, ES2015 = generator transform |
| No floating promises | Always await or explicitly handle `.catch()` |
