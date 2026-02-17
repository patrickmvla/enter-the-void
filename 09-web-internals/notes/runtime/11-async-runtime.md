# Async/Await Runtime Internals — Desugaring, Engine Optimization, and Scheduling

## What Async/Await Actually Compiles To

An `async` function is syntactic sugar over a **resumable state
machine** built on promises. The sugar hides enormous complexity.
Understanding what happens underneath means understanding the
compilation pipeline, the spec's Await() operation, and the engine
optimizations that make it fast.

---

## The Specification: Await()

The ECMA-262 `Await(value)` abstract operation (§27.7.5.3):

```
Await(value):
1.  Let asyncContext = the running execution context.
2.  Let promise = ? PromiseResolve(%Promise%, value).
    // If value is a native Promise → return it directly (fast path)
    // If value is a thenable → wrap in a new Promise
    // If value is not a thenable → Promise.resolve(value)

3.  Let fulfilledClosure = a closure(v) that:
    a. Suspend the running context.
    b. Push asyncContext back onto the execution context stack.
    c. Resume asyncContext with NormalCompletion(v).

4.  Let rejectedClosure = a closure(reason) that:
    a. Suspend the running context.
    b. Push asyncContext back onto the execution context stack.
    c. Resume asyncContext with ThrowCompletion(reason).

5.  Let onFulfilled = CreateBuiltinFunction(fulfilledClosure).
6.  Let onRejected = CreateBuiltinFunction(rejectedClosure).
7.  Perform PerformPromiseThen(promise, onFulfilled, onRejected).
8.  Remove asyncContext from the execution context stack.
9.  Return.
    // The async function is now suspended.
```

Step 2 is where the magic happens — `PromiseResolve` is the operation
that V8 optimized to eliminate extra microticks (see below).

---

## Desugaring: The State Machine

### What the Compiler Generates

Consider this async function:

```javascript
async function fetchAndProcess(url) {
  console.log('start');
  const response = await fetch(url);
  const data = await response.json();
  console.log('done');
  return data;
}
```

Conceptually, this becomes a state machine with states at each
`await` boundary:

```
State Machine:
┌─────────────────┐
│ State 0 (entry)  │ → console.log('start')
│                  │ → initiate fetch(url)
│                  │ → SUSPEND at await #1
├─────────────────┤
│ State 1          │ → receive response
│                  │ → initiate response.json()
│                  │ → SUSPEND at await #2
├─────────────────┤
│ State 2          │ → receive data
│                  │ → console.log('done')
│                  │ → RETURN data
└─────────────────┘
```

### TypeScript's `__awaiter` Helper

When targeting ES5/ES2015, TypeScript emits a `__awaiter` wrapper
that turns the generator-based state machine into a promise chain:

```javascript
var __awaiter = function(thisArg, _arguments, P, generator) {
  function adopt(value) {
    return value instanceof P ? value : new P(function(resolve) { resolve(value); });
  }
  return new (P || (P = Promise))(function(resolve, reject) {
    function fulfilled(value) {
      try { step(generator.next(value)); } catch(e) { reject(e); }
    }
    function rejected(value) {
      try { step(generator["throw"](value)); } catch(e) { reject(e); }
    }
    function step(result) {
      result.done
        ? resolve(result.value)
        : adopt(result.value).then(fulfilled, rejected);
    }
    step((generator = generator.apply(thisArg, _arguments || [])).next());
  });
};
```

The compiled output:

```javascript
function fetchAndProcess(url) {
  return __awaiter(this, void 0, void 0, function* () {
    console.log('start');
    const response = yield fetch(url);     // await → yield
    const data = yield response.json();    // await → yield
    console.log('done');
    return data;
  });
}
```

How `__awaiter` works step by step:

1. Creates a new Promise as the return value (the implicit promise).
2. Creates a generator from the inner `function*`.
3. `step()` is the **trampoline** — it calls `generator.next(value)`
   to advance the state machine.
4. If `result.done` is true, resolves the implicit promise with
   `result.value`.
5. If `result.done` is false, wraps the yielded value in a promise
   via `adopt()`, then chains `.then(fulfilled, rejected)`.
6. `fulfilled` and `rejected` call `step()` again with the next/throw
   result, creating the resumption cycle.

### When Targeting ES5 (No Generators)

TypeScript emits an additional `__generator` helper that compiles
the generator into an explicit switch-based state machine:

```javascript
function __generator(thisArg, body) {
  var _ = { label: 0, sent: function() { /*...*/ }, trys: [], ops: [] };
  // ...
  function step(op) {
    // Giant switch(_.label) implementing the state machine
    // _.label = program counter
    // _.trys = stack of try/catch/finally frames
    // _.sent() = value sent into generator
  }
}
```

The `_.label` field tracks which state the machine is in. Each
`case` in the switch corresponds to a suspension point. The
`_.trys` array tracks active try/catch/finally blocks as tuples
of `[tryLoc, catchLoc, finallyLoc, afterLoc]`.

### Babel's `regenerator-runtime`

Babel uses a similar approach but via the `regenerator` package:

```javascript
function _asyncToGenerator(fn) {
  return function() {
    var self = this, args = arguments;
    return new Promise(function(resolve, reject) {
      var gen = fn.apply(self, args);
      function _next(value) {
        asyncGeneratorStep(gen, resolve, reject, _next, _throw, "next", value);
      }
      function _throw(err) {
        asyncGeneratorStep(gen, resolve, reject, _next, _throw, "throw", err);
      }
      _next(undefined);
    });
  };
}
```

**Important**: Transpiled async functions have **different microtask
timing** than native async/await. The generator-based desugaring
creates more intermediate promises, scheduling more microtasks.
Code that depends on precise microtask ordering can behave differently
between transpiled and native execution.

---

## V8's Zero-Cost Async Optimization

### The Problem: 3 Extra Microticks (Pre-V8 7.2)

Before the November 2018 optimization, V8's `await` desugared to:

```javascript
// Internal V8 pseudocode for "await somePromise":
resumable function foo(somePromise) {
  implicit_promise = createPromise();

  // Step 1: Create a throwaway wrapper promise
  throwaway = createPromise();
  resolvePromise(throwaway, somePromise);
  // → Enqueues PromiseResolveThenableJob (microtask 1)

  // Step 2: Chain off the throwaway
  performPromiseThen(throwaway,
    (v) => resume(foo, v),
    (e) => throw(foo, e)
  );
  // When the PromiseResolveThenableJob runs, it calls
  // somePromise.then(resolve, reject) on the throwaway
  // → Enqueues PromiseReactionJob (microtask 2)

  // When somePromise settles, the reaction fires
  // → Enqueues resume microtask (microtask 3)

  suspend(foo, implicit_promise);
}
```

Each `await` cost **3 microtask ticks**. For a function with 5 awaits,
that's 15 microtasks just for the suspend/resume overhead.

### The Spec Change: PromiseResolve (ES2020)

V8 engineers (Maya Lerner, Benedikt Meurer) proposed a spec change
to TC39 (PR #1250 to ecma262). The new `PromiseResolve` operation:

```
PromiseResolve(C, x):
1. If IsPromise(x) is true:
   a. Let xConstructor = ? Get(x, "constructor").
   b. If SameValue(xConstructor, C) is true, return x.
      // Native Promise from same constructor → return as-is
2. Let promiseCapability = ? NewPromiseCapability(C).
3. Perform ? Call(promiseCapability.[[Resolve]], undefined, << x >>).
4. Return promiseCapability.[[Promise]].
```

The key: if the awaited value is already a native Promise, skip
wrapping entirely. No throwaway promise. No PromiseResolveThenableJob.

### After Optimization: 1 Microtick

```javascript
// Optimized V8 pseudocode for "await somePromise":
resumable function foo(somePromise) {
  implicit_promise = createPromise();

  // PromiseResolve detects somePromise is native — returns it directly
  promise = promiseResolve(somePromise); // === somePromise

  // Directly attach resume/throw handlers
  performPromiseThen(promise,
    (v) => resume(foo, v),
    (e) => throw(foo, e)
  );

  suspend(foo, implicit_promise);
  // Resume happens on the next microtick when somePromise settles
  // Total: 1 microtask
}
```

### Performance Impact

From the V8 blog (Doxbee benchmark):

```
Before optimization (V8 < 7.2):
  async/await: 4,063 ops/sec
  hand-written promises: 6,436 ops/sec

After optimization (V8 ≥ 7.2):
  async/await: 10,830 ops/sec (65% improvement!)
  hand-written promises: 10,922 ops/sec

async/await became as fast as hand-written promise chains.
```

### Throwaway Promise Elision

V8 went further — even when `PromiseResolve` can't return the value
directly (e.g., the value is a thenable but not a native Promise),
V8 detected that the "throwaway" wrapper promise is never exposed
to user code. It eliminated the allocation entirely, saving ~20 bytes
per `await` on non-native thenables.

---

## Zero-Cost Async Stack Traces

### The Problem

Before V8 7.2, async stack traces were **expensive**. V8 had to
capture the full call stack at every `await` point to chain the
frames together. This added overhead to every async function even
when no errors occurred.

### The Solution: Lazy Materialization

V8's approach (on by default since V8 7.3, `--async-stack-traces`):

1. When an async function suspends at `await`, V8 stores a reference
   to the **implicit promise** on the generator object. This is
   almost free — just a pointer write.

2. The implicit promise's `context` field already points to the
   async function's closure. No extra stack capture needed.

3. When `Error.stack` is accessed (or a stack trace is generated),
   V8 walks the chain:
   ```
   Current synchronous stack frames
   → implicit_promise of current async function
     → generator object → closure → call site info
       → implicit_promise of caller async function
         → ... (repeat up the async chain)
   ```

4. The stack trace string is formatted **lazily** — only when
   `.stack` is actually read. `Error.stack` is a **getter** in V8.

**Cost breakdown:**
- Normal execution (no errors): ~0 overhead. Just a pointer store.
- Error path: Walk the promise chain + format the string. Happens
  only when needed.

### Why `return await` Matters for Stack Traces

```javascript
async function withReturnAwait() {
  return await riskyOperation();
  // This function appears in the async stack trace
  // because it's suspended at the await when the error occurs
}

async function withoutReturnAwait() {
  return riskyOperation();
  // This function DOES NOT appear in the stack trace
  // because it has already returned — no active await
}
```

The ESLint rule `no-return-await` was **deprecated in v8.46** because
removing `return await` was harmful for debugging. Inside `try/catch`,
`return await` is also required for the catch block to fire:

```javascript
async function safe() {
  try {
    return await riskyOperation(); // MUST await to catch
  } catch (e) {
    return fallback;
  }
}

async function broken() {
  try {
    return riskyOperation(); // catch NEVER fires for rejections
  } catch (e) {
    return fallback; // dead code for async errors
  }
}
```

---

## Top-Level Await

### What It Is

ES2022 allows `await` at the top level of ES modules:

```javascript
// config.mjs
const response = await fetch('/api/config');
export const config = await response.json();
```

This is NOT available in:
- Scripts (non-module `<script>` tags without `type="module"`)
- CommonJS modules (`require()`)
- Node.js REPL (the REPL wraps input in an async function — a
  different mechanism that *looks like* TLA but isn't)

### Module Evaluation Order

Without TLA, modules evaluate synchronously in post-order (all
dependencies before the dependent module). TLA changes this:

```
Module graph:
    main.mjs
   /        \
a.mjs      b.mjs (has TLA)
```

Evaluation order:
1. `a.mjs` evaluates synchronously (no TLA) → completes.
2. `b.mjs` starts evaluating, hits `await` → suspends.
3. `main.mjs` cannot evaluate yet (depends on `b.mjs`).
4. The event loop runs. `b.mjs`'s awaited promise settles.
5. `b.mjs` resumes and completes.
6. `main.mjs` evaluates.

**The key insight**: sibling modules that don't depend on the
suspended module can evaluate concurrently:

```javascript
// a.mjs — no TLA
console.log('a start');
export const a = 1;
console.log('a end');

// b.mjs — has TLA
console.log('b start');
const data = await fetch('/api');
console.log('b end');
export const b = await data.json();

// main.mjs — depends on both
import { a } from './a.mjs';
import { b } from './b.mjs';
console.log('main', a, b);

// Output:
// "a start"
// "a end"
// "b start"
// ... (network delay) ...
// "b end"
// "main 1 {...}"
```

`a.mjs` completes immediately. `b.mjs` suspends. `main.mjs` waits
for both.

### The Variant: Maximum Parallelism

TC39 chose **Variant A** from three proposals:

- **Variant A (chosen)**: Siblings execute as soon as their
  dependencies are met. Maximum parallelism.
- **Variant B**: All siblings of an awaiting module are blocked.
- **Variant C**: Only direct siblings blocked.

Variant A matches the mental model of `Promise.all` for independent
branches of the dependency graph.

### Deadlock Risk with Circular Dependencies

```javascript
// x.mjs
import { y } from './y.mjs';
export const x = await someAsyncOp();

// y.mjs
import { x } from './x.mjs';
export const y = await anotherAsyncOp();
```

If `someAsyncOp()` depends on `y` being initialized, and
`anotherAsyncOp()` depends on `x` being initialized, the modules
will **deadlock** — each is suspended waiting for something that
depends on the other's completion.

There's no timeout, no error — just eternal suspension. The module
graph evaluation stalls silently.

**Even without TLA, circular dependencies are risky:**

```javascript
// x.mjs
import { y } from './y.mjs';
export const x = await Promise.resolve('x');

// y.mjs
import { x } from './x.mjs';
console.log(x); // undefined — x hasn't been initialized yet
export const y = 'y';
```

The first module in the cycle to evaluate sees `undefined` for
circular imports. TLA widens this window.

---

## AsyncContext and AsyncLocalStorage

### The Problem: Context Propagation

In synchronous code, you can use closures or module-scoped variables
to propagate context. In async code, this breaks:

```javascript
let currentRequestId = null;

async function handleRequest(id) {
  currentRequestId = id;
  await doSomething();
  // Another request may have overwritten currentRequestId!
  console.log(currentRequestId); // might not be `id` anymore
}

// Concurrent requests:
handleRequest('req-1');
handleRequest('req-2');
// currentRequestId is 'req-2' for both after the awaits
```

### Node.js AsyncLocalStorage

Available since Node.js 13.10, stable in v16:

```javascript
const { AsyncLocalStorage } = require('async_hooks');
const als = new AsyncLocalStorage();

function handleRequest(id) {
  als.run({ requestId: id }, async () => {
    await doSomething();
    console.log(als.getStore()); // { requestId: id } — correct!
  });
}

// Concurrent requests each see their own store:
handleRequest('req-1');
handleRequest('req-2');
// Each async chain sees its own { requestId: ... }
```

### How Context Propagation Works

AsyncLocalStorage is built on `async_hooks`, which tracks async
resource lifetimes:

```
Timeline:
┌────────────────────────────────────────────────┐
│ als.run({ requestId: 'req-1' }, callback)       │
│                                                  │
│  callback() starts                               │
│    → als.getStore() = { requestId: 'req-1' }     │
│                                                  │
│    await fetch('/api')                           │
│    │                                             │
│    │ V8 creates a PromiseReaction                │
│    │ async_hooks.init() fires:                   │
│    │   triggerAsyncId = current execution ID     │
│    │   store is PROPAGATED to the new async      │
│    │   resource via the init hook                │
│    │                                             │
│    └─── ... fetch completes ...                  │
│                                                  │
│    async_hooks.before() fires:                   │
│      restore the store for this async resource   │
│    → als.getStore() = { requestId: 'req-1' }     │
│    async_hooks.after() fires:                    │
│      restore previous store                      │
└────────────────────────────────────────────────┘
```

The four `async_hooks` callbacks:
- **init**: New async resource created → propagate store from parent
- **before**: Async callback about to execute → set active store
- **after**: Async callback finished → restore previous store
- **destroy**: Async resource GC'd → cleanup

### Performance

Early `async_hooks` had ~15-30% overhead because it hooked into
**every** async operation (timers, I/O, DNS, etc.). Node.js 16.4+
introduced a `PromiseHook` implementation that only hooks into
Promises, reducing overhead to ~3-5% for typical workloads.

Node.js 20+ uses V8's internal context tracking, further reducing
overhead.

### TC39 AsyncContext Proposal (Stage 2)

The proposal introduces `AsyncContext.Variable` and `AsyncContext.Snapshot`:

```javascript
const requestId = new AsyncContext.Variable({ defaultValue: 'none' });

requestId.run('req-1', async () => {
  await doSomething();
  console.log(requestId.get()); // 'req-1' — preserved across await

  requestId.run('req-2', () => {
    console.log(requestId.get()); // 'req-2' — nested override
  });

  console.log(requestId.get()); // 'req-1' — back to outer
});
```

The key semantic: every Promise captures a snapshot of all
`AsyncContext.Variable` values at creation time. When handlers run,
they restore that snapshot. This means:

```javascript
const ctx = new AsyncContext.Variable();

ctx.run('outer', () => {
  const p = Promise.resolve(42); // snapshot: ctx='outer'

  ctx.run('inner', () => {
    p.then(v => {
      console.log(ctx.get()); // 'outer' — creation context
      // NOT 'inner' (where .then was called)
    });
  });
});
```

Context flows from promise **creation**, not from `.then()` registration.

---

## Engine-Specific Async Optimizations

### V8 (Chrome, Node.js, Deno, Edge)

**1. TurboFan async function compilation:**
V8 compiles async functions as a single compilation unit with
explicit suspend/resume points. The bytecode compiler emits
`SuspendGenerator` and `ResumeGenerator` bytecodes at each `await`.
Local variables are stored in the generator object's register file
across suspension points.

**2. Promise hook overhead:**
V8's promise hooks (used by Node.js `async_hooks`) added ~30% overhead
pre-V8 7.4. The "zero-cost async hooks" optimization patched hooks
in lazily — they have **zero cost when not enabled**.

**3. Inline caching for `.then()`:**
TurboFan uses inline caches for `.then()` dispatch. When the same
promise shape is seen repeatedly, V8 caches the lookup and makes
`.then()` calls as fast as regular method calls.

**4. Promise allocation:**
V8 allocates promises in the nursery (young generation). Since most
promises are short-lived, they're collected by the Scavenger without
ever being promoted to old space. Cost: ~175 nanoseconds per promise.

### SpiderMonkey (Firefox)

**1. Warp compiler (replaced IonMonkey frontend, Firefox 83+):**
Uses CacheIR-based optimization that specializes on observed types.
For async functions, Warp specializes the state transitions at resume
points, avoiding generic dispatch.

**2. Generator representation:**
SpiderMonkey represents async function state machines as generator
objects internally. The Warp compiler optimizes the switch dispatch
for state transitions.

**3. Promise nursery allocation:**
SpiderMonkey's generational GC allocates promises in the nursery.
Short-lived promises are collected cheaply without promotion.

**4. Microtask queue:**
SpiderMonkey implements the microtask queue as a C++-level circular
buffer — O(1) enqueue/dequeue, integrated directly into the event
loop at the C++ level.

### JavaScriptCore (Safari, Bun)

**1. Four-tier JIT: LLInt → Baseline → DFG → FTL:**
Async functions are optimized at the DFG and FTL tiers. FTL (Faster
Than Light) uses B3, a backend that generates highly optimized
machine code.

**2. Structure transition model for promises:**
JSC allocates promises using structure transitions. Common patterns
like `.then().catch()` get specialized structures, reducing property
lookup overhead.

**3. for-await-of optimization:**
JSC has specific DFG/FTL optimizations that avoid materializing
intermediate `{ value, done }` result objects when the compiler can
prove they're only destructured.

**4. Riptide GC:**
JSC's concurrent generational GC collects short-lived promises
without pausing the main thread.

### Cross-Engine: `await` as JIT Optimization Barrier

**This is a critical performance insight that applies to ALL engines.**

```javascript
async function example() {
  const a = computeA();    // JIT can optimize this
  const b = computeB(a);   // JIT can optimize this WITH context from a
  await somePromise;        // <<< OPTIMIZATION BARRIER >>>
  const c = computeC(b);   // JIT starts fresh — no cross-await optimization
  const d = computeD(c);   // JIT can optimize this WITH context from c
}
```

**Why `await` is a barrier:**

1. `await` suspends the function and returns control to the event
   loop. The JIT treats each section between awaits as a semi-
   independent unit.

2. Local variables are stored in the generator object's register
   file across await boundaries — NOT in CPU registers. After
   resumption, values must be reloaded.

3. Cross-await optimizations like hoisting, common subexpression
   elimination, and register allocation across the boundary are
   generally not performed.

4. Each "state" after a resume point may have different type
   feedback than the corresponding state before suspension.

**Practical implication — batch synchronous work:**

```javascript
// Worse: JIT can't optimize across barriers
for (const item of items) {
  const processed = heavyComputation(item);  // JIT optimizes
  await saveToDb(processed);                  // barrier
  const verified = verify(processed);         // JIT restarts
  await logResult(verified);                  // barrier
}

// Better: Synchronous work batched for JIT optimization
const processed = items.map(heavyComputation);  // hot loop, JIT-friendly
for (let i = 0; i < processed.length; i++) {
  const verified = verify(processed[i]);
  await saveToDb(processed[i]);
  await logResult(verified);
}
```

---

## Scheduling Patterns

### Why Promises Can't Yield to Rendering

Microtasks (promise callbacks) drain **completely** before the
browser can render. If you use promises for scheduling work, you
block the main thread:

```javascript
// This blocks rendering until ALL items are processed:
async function processAll(items) {
  for (const item of items) {
    await process(item); // microtask — no rendering between iterations
  }
}
```

The microtask queue drains fully during a single microtask checkpoint.
No rendering, no input handling, no macrotasks execute until the
queue is empty.

### scheduler.postTask() — Prioritized Task Scheduling API

Chrome 94+ provides a priority-aware task scheduler:

```javascript
// Three priority levels:
// 'user-blocking' — highest, for input response
// 'user-visible'  — default, for visible updates
// 'background'    — lowest, for non-urgent work

const result = await scheduler.postTask(() => {
  return expensiveComputation();
}, { priority: 'background' });

// With AbortSignal for cancellation:
const controller = new AbortController();
scheduler.postTask(work, {
  priority: 'background',
  signal: controller.signal
});
controller.abort(); // cancels if not yet started

// Dynamic priority via TaskController:
const taskController = new TaskController({ priority: 'background' });
scheduler.postTask(work, { signal: taskController.signal });
// Promote priority dynamically:
taskController.setPriority('user-blocking');
```

Each `postTask` call creates a **macrotask** at the specified priority.
This means rendering and input handling can interleave between tasks.

### scheduler.yield() — Cooperative Yielding

Chrome 115+ provides explicit yielding:

```javascript
async function processLargeList(items) {
  for (const item of items) {
    processItem(item);
    await scheduler.yield(); // yields to browser
  }
}
```

Why `scheduler.yield()` is better than `setTimeout(0)`:

1. **No 4ms clamping**: `setTimeout` clamps to 4ms after 5 levels
   of nesting. `scheduler.yield()` has no minimum delay.

2. **Priority preservation**: The continuation runs at the same
   priority as the yielding task, not at the default priority.

3. **Semantic clarity**: Explicitly communicates "I'm yielding."

### React's Scheduler: MessageChannel + Min-Heap

React's scheduler avoids promises entirely for scheduling because
microtasks can't yield to rendering. Instead:

**1. MessageChannel for zero-delay macrotasks:**

```javascript
const channel = new MessageChannel();
const port = channel.port2;
channel.port1.onmessage = performWorkUntilDeadline;

function requestHostCallback(callback) {
  scheduledHostCallback = callback;
  port.postMessage(null); // macrotask with ~0ms delay
}
```

`MessageChannel.onmessage` fires as a macrotask with effectively
zero delay — unlike `setTimeout(fn, 0)` which has a minimum 1ms
delay and 4ms clamping after nesting.

**2. Priority levels and expiration:**

| Priority       | Timeout   | Description                    |
|----------------|-----------|--------------------------------|
| ImmediatePriority | -1 ms  | Already expired — run now      |
| UserBlockingPriority | 250 ms | User interactions            |
| NormalPriority   | 5000 ms  | Normal updates                 |
| LowPriority      | 10000 ms | Low-priority updates           |
| IdlePriority     | maxSafe  | Never expires — true idle work |

**3. Min-heap task queue:**

Tasks are stored in a min-heap sorted by `expirationTime`
(`startTime + timeout`). This gives:
- O(log n) insertion
- O(1) peek (highest priority task)
- O(log n) extraction

**4. Time slicing (5ms chunks):**

```javascript
function workLoop(deadline) {
  while (currentTask !== null && !shouldYield()) {
    // Perform one unit of work
    const continuationCallback = currentTask.callback(didTimeout);
    if (typeof continuationCallback === 'function') {
      currentTask.callback = continuationCallback;
    } else {
      pop(taskQueue);
    }
    currentTask = peek(taskQueue);
  }
}

function shouldYield() {
  return getCurrentTime() >= deadline; // 5ms budget
}
```

React performs work in 5ms chunks, then yields back to the browser
via `MessageChannel`. This is what enables concurrent features —
Suspense, transitions, `useTransition()`.

**5. Why not requestIdleCallback?**

React tried `requestIdleCallback` (rIC) and abandoned it:
- Safari doesn't support it
- Chrome's implementation fires inconsistently (sometimes >50ms delay)
- rIC only fires when the main thread is idle — React needs to run
  during busy periods too (just yield often enough)

### The Scheduling Hierarchy

```
Priority (high → low):

Microtasks (Promise.then, queueMicrotask)
  → Drain completely, block everything
  → Use for: dependent async continuations

scheduler.postTask('user-blocking')
  → High-priority macrotask
  → Use for: responding to user input

requestAnimationFrame
  → Before next paint
  → Use for: visual updates, animations

scheduler.postTask('user-visible')
  → Normal macrotask
  → Use for: general work

scheduler.postTask('background')
  → Low-priority macrotask
  → Use for: prefetching, analytics

requestIdleCallback
  → Only when idle
  → Use for: truly non-urgent work

setTimeout(..., 0)
  → Macrotask with clamping
  → Use for: legacy compatibility
```

---

## Async Iteration Protocol Deep Dive

### for-await-of Desugaring

```javascript
for await (const item of asyncIterable) {
  process(item);
}
```

Desugars to:

```javascript
const iterator = asyncIterable[Symbol.asyncIterator]();
try {
  while (true) {
    const result = await iterator.next();
    if (result.done) break;
    const item = result.value;
    process(item);
  }
} finally {
  if (iterator.return) {
    await iterator.return(); // cleanup on early exit
  }
}
```

**Key details:**
- `iterator.next()` is called and the result is `await`ed even if
  the iterator returns synchronous values.
- `iterator.return()` fires on **any** early exit: `break`, `throw`,
  `return`. This is critical for resource cleanup.
- If `return()` itself throws, that error **replaces** any error
  from the loop body.

### Async-from-Sync Iterator Wrapper

When `for await...of` receives a **synchronous** iterable, the spec
creates an `AsyncFromSyncIterator` wrapper:

```javascript
// This works — sync iterables auto-wrapped:
for await (const x of [1, 2, 3]) {
  console.log(x); // 1, 2, 3
}

// The wrapper does Promise.resolve() on each value:
for await (const x of [Promise.resolve(1), Promise.resolve(2)]) {
  console.log(x); // 1, 2 — promises are UNWRAPPED
}
```

### AsyncGenerator State Machine

Async generators have **four states**:

```
┌─────────────────┐
│ suspendedStart   │ ← created but .next() not yet called
│                  │
│  .next() ──────────► executing ──────┐
│                  │                    │
│                  │  yield ◄───────────┤
│                  │  ↓                 │
│ suspendedYield   │ ← paused at yield │
│                  │                    │
│  .next() ──────────► executing       │
│                  │                    │
│  return ◄─────────────────────────────┘
│  throw  ◄─────────────────────────────┘
│  ↓
│ completed        │ ← terminal state
└─────────────────┘
```

**The internal queue**: If you call `.next()` multiple times before
the previous one resolves, they queue up:

```javascript
async function* gen() {
  yield await fetch('/a');
  yield await fetch('/b');
}

const g = gen();
const p1 = g.next(); // starts execution → suspends at first await
const p2 = g.next(); // QUEUED — will resume after first yield
const p3 = g.next(); // QUEUED — will resume after second yield
```

This queue is necessary because the generator is single-threaded.

### ReadableStream as Async Iterable

```javascript
const response = await fetch('/large-file');
for await (const chunk of response.body) {
  processChunk(chunk);
}
```

Under the hood:
1. `response.body[Symbol.asyncIterator]()` acquires a reader via
   `stream.getReader()`.
2. Each `.next()` call does `reader.read()`.
3. `return()` calls `reader.releaseLock()` and cancels the stream.

---

## Key Takeaways

1. **async/await compiles to state machines** — TypeScript uses
   `__awaiter` + generators, Babel uses `regenerator-runtime`.
   Native engines use bytecode-level suspend/resume.

2. **V8 reduced await overhead from 3 to 1 microticks** — via the
   `PromiseResolve` spec change (ES2020) and throwaway promise
   elision. async/await is now as fast as hand-written promises.

3. **Async stack traces are zero-cost** — V8 stores a pointer at
   each await, formats the trace lazily only on error. Use
   `return await` in try/catch for correct error handling.

4. **Top-level await modifies module evaluation** — siblings proceed
   in parallel, but circular dependencies can deadlock silently.

5. **AsyncLocalStorage propagates through promises** — built on
   async_hooks (or the proposed AsyncContext). Context flows from
   promise creation, not `.then()` registration.

6. **`await` is a JIT optimization barrier** — all engines. Batch
   synchronous computation between await points for best JIT
   performance.

7. **Promises can't yield to rendering** — microtasks drain
   completely. Use `scheduler.postTask()`, `scheduler.yield()`,
   or `MessageChannel` for interruptible work.

8. **React uses MessageChannel + min-heap** — not promises, not
   `requestIdleCallback`. 5ms time slices with priority scheduling.
