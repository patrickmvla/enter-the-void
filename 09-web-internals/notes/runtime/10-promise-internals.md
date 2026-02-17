# Promise Internals — Specification, Engine Implementation, and Memory Model

## The Promise State Machine

Every Promise is a finite state machine with exactly three states and
two possible transitions. Once a promise leaves the `pending` state,
it can never transition again — this is the **immutability guarantee**.

```
                    FulfillPromise(promise, value)
            ┌─────────────────────────────────────────┐
            │                                         ▼
        ┌───────┐                              ┌───────────┐
        │PENDING│                              │ FULFILLED │
        └───────┘                              └───────────┘
            │
            │       RejectPromise(promise, reason)
            └─────────────────────────────────────────┐
                                                      ▼
                                               ┌──────────┐
                                               │ REJECTED │
                                               └──────────┘

No transitions from FULFILLED or REJECTED — ever.
```

The ECMA-262 specification defines five internal slots on every
Promise object:

| Internal Slot                | Type                      | Description                                        |
|------------------------------|---------------------------|----------------------------------------------------|
| `[[PromiseState]]`           | `"pending"` / `"fulfilled"` / `"rejected"` | Current state                |
| `[[PromiseResult]]`          | any                       | Resolution value or rejection reason               |
| `[[PromiseFulfillReactions]]`| List of PromiseReaction   | Handlers waiting for fulfillment (cleared on settle)|
| `[[PromiseRejectReactions]]` | List of PromiseReaction   | Handlers waiting for rejection (cleared on settle)  |
| `[[PromiseIsHandled]]`       | Boolean                   | Has a handler been attached? (unhandled tracking)   |

---

## The Spec's Abstract Operations

### PromiseCapability Records

A PromiseCapability bundles a promise with the power to resolve or
reject it:

```
PromiseCapability Record {
  [[Promise]]: Object    — the Promise object itself
  [[Resolve]]: Function  — the resolve function
  [[Reject]]:  Function  — the reject function
}
```

**NewPromiseCapability(C)** creates one:

```
NewPromiseCapability(C):
1. If IsConstructor(C) is false, throw TypeError.
2. Let promiseCapability = { [[Promise]]: undefined,
                             [[Resolve]]: undefined,
                             [[Reject]]:  undefined }.
3. Let executorClosure be a closure that:
   a. Sets promiseCapability.[[Resolve]] = resolve argument
   b. Sets promiseCapability.[[Reject]]  = reject argument
4. Let promise = ? Construct(C, << executorClosure >>).
5. If promiseCapability.[[Resolve]] is not callable, throw TypeError.
6. If promiseCapability.[[Reject]] is not callable, throw TypeError.
7. Set promiseCapability.[[Promise]] = promise.
8. Return promiseCapability.
```

This is the spec-level abstraction that `Promise.withResolvers()`
now exposes to userland (ES2024). Before that, the only way to get
the resolve/reject handles outside the executor was the ugly pattern:

```javascript
// The deferred anti-pattern that Promise.withResolvers replaces
let resolve, reject;
const promise = new Promise((res, rej) => {
  resolve = res;
  reject = rej;
});
```

### PromiseReaction Records

When you call `.then(onFulfilled, onRejected)`, the spec creates
PromiseReaction records — these are the pending callbacks waiting
for the promise to settle:

```
PromiseReaction Record {
  [[Capability]]: PromiseCapability | undefined
  [[Type]]:       "Fulfill" | "Reject"
  [[Handler]]:    callable | empty
}
```

- `[[Capability]]` — the PromiseCapability of the promise that
  `.then()` returned (so when the handler runs, its return value
  can resolve/reject the chained promise)
- `[[Type]]` — which transition triggers this reaction
- `[[Handler]]` — the actual callback (`onFulfilled` or `onRejected`),
  or `empty` if the argument was null/undefined (passthrough)

### The Critical Resolve Algorithm

The resolve function you get in `new Promise((resolve, reject) => ...)`
follows this algorithm — it's the heart of promise resolution:

```
Promise Resolve Functions (resolution):

1. Let promise = F.[[Promise]].
2. Let alreadyResolved = F.[[AlreadyResolved]].
3. If alreadyResolved.[[Value]] is true, return undefined.
   // First-call-wins: subsequent calls are no-ops
4. Set alreadyResolved.[[Value]] = true.

5. If SameValue(resolution, promise) is true:
   a. Let selfResolutionError = new TypeError("self-resolution").
   b. Perform RejectPromise(promise, selfResolutionError).
   c. Return.
   // A promise cannot resolve to itself — infinite chain

6. If resolution is not an Object:
   a. Perform FulfillPromise(promise, resolution).
   b. Return.
   // Primitives fulfill immediately — no thenable check

7. Let then = Get(resolution, "then").
   // This is a PROPERTY ACCESS — it can throw, it can be a getter
8. If then is an abrupt completion:
   a. Perform RejectPromise(promise, then.[[Value]]).
   b. Return.
   // If accessing .then throws, the promise rejects

9. Let thenAction = then.[[Value]].
10. If IsCallable(thenAction) is false:
    a. Perform FulfillPromise(promise, resolution).
    b. Return.
    // Object with no callable .then — treat as plain value

11. Let thenJobCallback = HostMakeJobCallback(thenAction).
12. Let job = NewPromiseResolveThenableJob(promise, resolution, thenJobCallback).
13. Perform HostEnqueuePromiseJob(job.[[Job]], job.[[Realm]]).
14. Return.
    // Object WITH callable .then — schedule async assimilation
```

**Step 7 is remarkable.** The spec reads `.then` exactly once and
caches it. This matters for adversarial thenables:

```javascript
let readCount = 0;
const evil = Object.defineProperty({}, 'then', {
  get() {
    readCount++;
    if (readCount === 1) return (resolve) => resolve(42);
    return undefined; // different on second access
  }
});

const p = Promise.resolve(evil);
// .then is read ONCE, cached as the callable,
// and passed to NewPromiseResolveThenableJob
```

**Steps 5-6 establish the recursive unwrapping semantics:**
- Resolving with a primitive → immediate fulfillment
- Resolving with a non-thenable object → immediate fulfillment
- Resolving with a thenable → async unwrapping via PromiseResolveThenableJob
- Resolving with itself → TypeError rejection

---

## The Two Microtask Job Types

The spec defines two distinct types of promise-related microtasks.
Understanding the difference is essential for predicting execution order.

### PromiseReactionJob

This is the "normal" microtask — it runs a `.then()` handler:

```
NewPromiseReactionJob(reaction, argument):

Creates a job that, when executed:
1. Let handler = reaction.[[Handler]].
2. If handler is empty:
   a. If reaction.[[Type]] is "Fulfill":
      Let handlerResult = NormalCompletion(argument).
   b. Else (reaction.[[Type]] is "Reject"):
      Let handlerResult = ThrowCompletion(argument).
3. Else:
   Let handlerResult = Completion(Call(handler, undefined, << argument >>)).
   // Actually run the callback

4. If reaction.[[Capability]] is undefined, return empty.
   // Internal reactions with no chained promise

5. Let capability = reaction.[[Capability]].
6. If handlerResult is an abrupt completion:
   Perform ? Call(capability.[[Reject]], undefined,
                  << handlerResult.[[Value]] >>).
7. Else:
   Perform ? Call(capability.[[Resolve]], undefined,
                  << handlerResult.[[Value]] >>).
   // Resolve/reject the chained promise with the handler's result
```

When the handler is `empty` (you called `.then(null, null)` or
`.then()` with missing arguments), the value/reason passes straight
through — this is how `.catch()` and `.finally()` work as sugar.

### PromiseResolveThenableJob

This is the "thenable assimilation" microtask — it fires when a
promise is resolved with a thenable:

```
NewPromiseResolveThenableJob(promiseToResolve, thenable, then):

Creates a job that, when executed:
1. Let resolvingFunctions = CreateResolvingFunctions(promiseToResolve).
2. Let thenCallResult = Completion(
     Call(then, thenable, << resolvingFunctions.[[Resolve]],
                             resolvingFunctions.[[Reject]] >>)
   ).
   // Call thenable.then(resolve, reject) to subscribe
3. If thenCallResult is an abrupt completion:
   Perform ? Call(resolvingFunctions.[[Reject]], undefined,
                  << thenCallResult.[[Value]] >>).
   // If .then() throws, reject the promise
```

**This is the source of the extra microtask tick when resolving a
promise with another promise.** The resolution doesn't happen
synchronously — it's deferred to a PromiseResolveThenableJob
microtask, which then calls `.then()` on the inner promise, which
schedules another microtask for when the inner promise settles.

---

## Microtask Counting — Predicting Execution Order

### Pattern 1: Simple `.then()` chain

```javascript
Promise.resolve(42)
  .then(v => v + 1)    // 1 microtask (PromiseReactionJob)
  .then(v => v + 1)    // 1 microtask
  .then(v => v + 1);   // 1 microtask
// Total: 3 microtasks, executed sequentially
```

### Pattern 2: Resolving with a native Promise

```javascript
const p1 = Promise.resolve(42);
const p2 = new Promise(resolve => resolve(p1));
p2.then(v => console.log(v));

// resolve(p1) detects p1 has a callable .then
// → PromiseResolveThenableJob enqueued           (microtask 1)
// → That job calls p1.then(resolve2, reject2)
// → p1 is already fulfilled, so a PromiseReactionJob is enqueued
//                                                 (microtask 2)
// → That job fulfills p2 with 42
// → p2's .then handler is enqueued               (microtask 3)
// Total: 3 microtasks
```

### Pattern 3: `await` on a native Promise (V8 optimized)

```javascript
async function foo() {
  const v = await Promise.resolve(42);
  // Post-2018 optimization: V8 uses PromiseResolve to detect
  // that the value is already a native Promise.
  // Skips PromiseResolveThenableJob entirely.
  // 1 microtask to resume.
  return v;
}
foo().then(v => console.log(v));
// foo() resumes: 1 microtask
// .then handler: 1 microtask
// Total: 2 microtasks
```

### Pattern 4: `await` on a thenable (NOT optimized)

```javascript
async function bar() {
  const v = await { then(resolve) { resolve(42); } };
  return v;
}
bar().then(v => console.log(v));

// The thenable is NOT a native Promise
// → PromiseResolve wraps it, triggering PromiseResolveThenableJob
//                                                 (microtask 1)
// → That job calls thenable.then(resolve), which resolves immediately
// → But the resolve triggers FulfillPromise, scheduling the
//   continuation reaction                         (microtask 2)
// → bar() resumes, returns v
// → .then handler fires                          (microtask 3)
// Total: 3 microtasks (1 extra vs native Promise)
```

### Pattern 5: `queueMicrotask` vs `Promise.resolve().then()`

```javascript
queueMicrotask(() => console.log('a'));
Promise.resolve().then(() => console.log('b'));

// Both enqueue exactly 1 microtask
// But Promise.resolve().then() also allocates a new Promise
// (the return value of .then()) that nobody uses
// queueMicrotask is cheaper — no Promise allocation
```

### The Observable Difference: Thenables vs Promises

```javascript
async function fast() {
  await Promise.resolve();
  console.log('fast');
}

async function slow() {
  await { then(r) { r(); } };
  console.log('slow');
}

fast();
slow();
// Output: "fast" then "slow"
// fast() resumes in 1 microtick, slow() in 2 microticks
```

This ordering difference is **observable** and has caused real bugs
in code that assumes all awaited values have identical timing.

---

## V8 Promise Implementation

### The JSPromise Object Layout

In V8's C++ codebase, a Promise is represented as a `JSPromise`
object with a highly compressed layout:

```
JSPromise (V8 internal, pointer-compressed, 64-bit):
┌─────────────────────────────────┐
│ Map pointer          (4 bytes)  │  // Hidden class / shape
├─────────────────────────────────┤
│ Properties pointer   (4 bytes)  │  // Usually empty_fixed_array
├─────────────────────────────────┤
│ Elements pointer     (4 bytes)  │  // Usually empty_fixed_array
├─────────────────────────────────┤
│ reactions_or_result  (4 bytes)  │  // Overloaded field:
│                                 │  //   pending: PromiseReaction linked list
│                                 │  //   settled: the result value
├─────────────────────────────────┤
│ flags                (4 bytes)  │  // Bitfield:
│   bits 0-1: status             │  //   0=pending, 1=fulfilled, 2=rejected
│   bit 2:    has_handler        │  //   [[PromiseIsHandled]]
│   bit 3:    handled_hint       │  //   DevTools tracking
│   bit 4:    async_task_id      │  //   async stack trace linkage
└─────────────────────────────────┘
Total: ~20 bytes (compressed pointers)
      ~40 bytes (uncompressed, 64-bit)
```

The `reactions_or_result` field is a **tagged union** — when the
promise is pending, it points to a linked list of PromiseReaction
objects. When the promise settles, those reactions are triggered and
the field is overwritten with the result value.

### PromiseReaction Objects

```
PromiseReaction (V8 internal):
┌─────────────────────────────────┐
│ Map pointer          (4 bytes)  │
├─────────────────────────────────┤
│ next                 (4 bytes)  │  // Linked list → next reaction
├─────────────────────────────────┤
│ reject_handler       (4 bytes)  │  // The onRejected callback
├─────────────────────────────────┤
│ fulfill_handler      (4 bytes)  │  // The onFulfilled callback
├─────────────────────────────────┤
│ promise_or_capability(4 bytes)  │  // The chained promise
├─────────────────────────────────┤
│ continuation_preserved_embedder_data │
└─────────────────────────────────┘
Total: ~24 bytes (compressed)
```

Reactions form a **singly-linked list** off the promise. When the
promise settles, V8 walks this list, creates a PromiseReactionJob
microtask for each reaction, enqueues them all, then overwrites the
field with the result value.

### V8's Fast Paths

V8 implements several fast paths to avoid the full spec algorithm
overhead for common cases:

**1. Promise.resolve(value) identity shortcut:**
```javascript
const p = Promise.resolve(42);
Promise.resolve(p) === p; // true — same object returned
```

When `Promise.resolve` receives a value whose constructor is `%Promise%`
(not a subclass), it returns the value directly without wrapping. V8
checks `value.map === native_promise_map` — a single pointer comparison.

**2. .then() fast path for native Promises:**
V8 checks that:
- The receiver is a native Promise (not subclassed)
- `Promise.prototype.then` hasn't been modified
- The promise's `constructor` property is `Promise`

If all checks pass, V8 bypasses `NewPromiseCapability` (which would
call `new Promise(executor)`) and directly creates the result promise
and reaction objects using internal C++ constructors.

**3. await fast path (PromiseResolve):**
When `await` receives a native Promise, V8 skips creating a wrapper
promise and directly calls `PerformPromiseThen` on the awaited value.
This is the optimization that reduced `await` from 3 to 1 microticks.

**4. Promise.all fast path:**
V8 checks that the input is an Array (not a generic iterable), the
Array's iterator hasn't been modified, and `Promise.resolve` is the
built-in. If so, it uses an optimized C++ loop instead of the generic
iterable protocol.

---

## The Promise.race Memory Leak

One of the most important and subtle memory issues with promises:

```javascript
const neverSettles = new Promise(() => {});

async function pollForever() {
  while (true) {
    const result = await Promise.race([
      fetchData(),       // resolves in ~100ms
      neverSettles       // never resolves
    ]);
    process(result);
  }
}
```

**Why it leaks:**

Each `Promise.race()` call does `.then(resolve, reject)` on every
input promise. This creates a PromiseReaction record on each input.
When `fetchData()` wins the race, the `Promise.race` result settles
and its reactions are cleared.

But `neverSettles` is still pending. Its reaction list **never gets
cleared** — PromiseReaction lists are only cleared by `FulfillPromise`
or `RejectPromise`. Each loop iteration adds a new PromiseReaction
to `neverSettles`:

```
After 1 iteration:
  neverSettles.reactions → [Reaction₁]

After 100 iterations:
  neverSettles.reactions → [Reaction₁] → [Reaction₂] → ... → [Reaction₁₀₀]

After 10,000 iterations:
  neverSettles.reactions → [Reaction₁] → ... → [Reaction₁₀₀₀₀]
  Memory: ~24 bytes × 10,000 = ~240 KB just for reactions
  Plus closures, plus the resolve/reject functions, plus the
  result promises: ~132 bytes × 10,000 = ~1.3 MB total
```

Each reaction holds references to:
- The `Promise.race` result promise's resolve/reject functions
- The resolve/reject closures capture scope variables
- The result promise itself

None of these can be garbage collected because `neverSettles` is
reachable and holds them via its reaction list.

**The fix — use fresh promises per iteration:**

```javascript
async function pollForever() {
  while (true) {
    const controller = new AbortController();
    const timeout = new Promise((_, reject) => {
      const timer = setTimeout(() => reject(new Error('timeout')), 5000);
      controller.signal.addEventListener('abort', () => clearTimeout(timer));
    });

    try {
      const result = await Promise.race([
        fetchData(),
        timeout
      ]);
      process(result);
    } finally {
      controller.abort(); // clean up
    }
    // Both promises in the race are now settleable/settled
    // No reaction accumulation on a single pending promise
  }
}
```

The key insight: **never reuse a permanently-pending promise as a
participant in `Promise.race()` across iterations.**

---

## Unhandled Rejection Tracking

### The Specification Hook

When a promise is rejected with no handler, the spec calls:

```
HostPromiseRejectionTracker(promise, operation):
  operation = "reject"  → promise was rejected, no handler yet
  operation = "handle"  → handler was attached to a previously-unhandled rejection
```

The spec deliberately leaves the **behavior** of this hook to the
host environment (browser, Node.js, Deno). Different hosts implement
it differently.

### Browser Implementation (Chromium)

Chromium uses a **microtask-checkpoint-based** approach:

```
Timeline:
┌─────────────────────────────────────────────────────┐
│ Task: run script                                     │
│  ① Promise.reject(err)                               │
│     → HostPromiseRejectionTracker(p, "reject")       │
│     → p added to "pending unhandled rejections" set  │
│                                                      │
│  ② Promise.resolve().then(() => {                    │
│       p.catch(() => {});                             │
│       → HostPromiseRejectionTracker(p, "handle")     │
│       → p removed from pending set                   │
│     })                                               │
│                                                      │
│ [microtask checkpoint]                               │
│  ③ Drain microtask queue                             │
│     - ② runs, p gets a handler                       │
│                                                      │
│ [end of microtask checkpoint]                        │
│  ④ Check pending set — p was removed → no event      │
└─────────────────────────────────────────────────────┘
```

You have until the **end of the current microtask queue drain** to
attach a handler:

```javascript
// This is FINE — handler attached in same microtask drain:
const p = Promise.reject(new Error('oops'));
Promise.resolve().then(() => {
  p.catch(() => {}); // still in time
});
// No unhandledrejection event fires

// This is TOO LATE — handler attached in next task:
const p2 = Promise.reject(new Error('oops'));
setTimeout(() => {
  p2.catch(() => {}); // after microtask checkpoint
}, 0);
// unhandledrejection fires, then rejectionhandled fires
```

The events:

```javascript
window.addEventListener('unhandledrejection', (event) => {
  event.promise;          // the rejected Promise
  event.reason;           // the rejection reason
  event.preventDefault(); // prevents console error
});

window.addEventListener('rejectionhandled', (event) => {
  // Fires when a previously-unhandled rejection gets handled
  event.promise;
  event.reason;
});
```

### Node.js Implementation

Node.js uses a **tick-based** approach:

```javascript
process.on('unhandledRejection', (reason, promise) => {
  // Note: argument order is (reason, promise), not an event object
  console.error('Unhandled rejection:', reason);
});

process.on('rejectionHandled', (promise) => {
  // Only gets the promise, not the reason
});
```

Key differences from browsers:

1. **Detection timing**: Node.js checks after the next event loop
   tick (not microtask checkpoint). This gives slightly more time.

2. **Default behavior changed in Node.js 15+**: Unhandled rejections
   now **terminate the process** by default (`--unhandled-rejections=throw`).
   Before Node.js 15, it was just a deprecation warning.

3. **The `--unhandled-rejections` flag** controls behavior:
   - `throw` (default since v15): throw as uncaught exception
   - `warn`: print warning (old behavior)
   - `strict`: throw even if listener is attached
   - `none`: silence everything

### The `[[PromiseIsHandled]]` Flag

This internal slot drives the tracking. It's set to `true` when:

- `.then()` or `.catch()` is called on the promise
- `await` is used on the promise (since `await` calls `.then()`)
- The promise is passed to `Promise.all()`, `.race()`, etc.

**The subtle timing bug:**

```javascript
async function dangerous() {
  const p = riskyOperation(); // returns rejected promise
  // p.[[PromiseIsHandled]] is false here

  await delay(1000); // 1 second passes
  // During this time, the microtask checkpoint fires
  // unhandledRejection detected for p!

  return await p; // we intended to handle it here, but too late
}
```

**The fix — always attach handlers synchronously:**

```javascript
async function safe() {
  const p = riskyOperation();
  // Immediately make it "handled" by attaching .catch()
  // but re-throw so the original rejection still propagates
  const handled = p.catch(() => {}); // mark as handled

  await delay(1000);

  return await p; // now safe — p was already marked handled
}

// Or better — restructure the code:
async function better() {
  await delay(1000);
  return await riskyOperation(); // await immediately
}
```

---

## Promises/A+ vs ECMA-262

The Promises/A+ specification (2013) was the community standard that
informed ES2015 Promises. They are related but not identical.

### What Promises/A+ Defines

Promises/A+ is **minimal** — it only specifies:
- The `.then()` method
- The Promise Resolution Procedure
- State transitions (pending → fulfilled/rejected)

It does **NOT** define:
- `Promise.resolve()` / `Promise.reject()`
- `Promise.all()` / `Promise.race()` / `Promise.any()` / `Promise.allSettled()`
- The constructor `new Promise(executor)`
- `.catch()` or `.finally()`
- `Promise.withResolvers()`

### The Key Difference: Thenable Assimilation Timing

**Promises/A+, §2.3.2**: If `x` is a promise, adopt its state.
This is underspecified regarding *when* the adoption happens.

**ECMA-262**: Always uses `PromiseResolveThenableJob` — a microtask.
This means resolving a promise with a thenable (including another
Promise) always costs one extra microtask tick.

```javascript
const p1 = Promise.resolve(42);
const p2 = new Promise(resolve => resolve(p1));

// ECMA-262 guarantees:
// p2 does NOT settle synchronously.
// A PromiseResolveThenableJob is enqueued.
// p2 settles in the next microtask.
```

This extra tick is deliberate — it prevents thenables from running
synchronously during promise resolution, which could observe
intermediate states or cause re-entrancy bugs.

### The `.then` Property: Read Once, Cache

Both specs mandate that `.then` is read exactly once from the
resolution value:

**Promises/A+, §2.3.3.1**: `Let then = x.then`
**ECMA-262, Step 7**: `Let then = Get(resolution, "then")`

The value is cached for the PromiseResolveThenableJob. If `.then`
is a getter with side effects, it runs exactly once:

```javascript
let calls = 0;
const tricky = {
  get then() {
    calls++;
    return (resolve) => resolve(42);
  }
};

Promise.resolve(tricky);
// calls === 1, guaranteed by both specs
```

### Subclassing

ECMA-262 supports Promise subclassing through `Symbol.species` and
`NewPromiseCapability(C)`. Promises/A+ has no notion of subclassing.

In practice, most engines **de-optimize** when they encounter
subclassed promises. V8 falls off all fast paths when it detects
a non-native Promise constructor:

```javascript
class MyPromise extends Promise {}

const p = MyPromise.resolve(42);
// V8 uses the slow path:
// - Calls new MyPromise(executor) via NewPromiseCapability
// - Cannot use internal C++ constructors
// - Cannot use the PromiseResolve fast path
// - ~3-5x slower than native Promise.resolve(42)
```

---

## Promise.withResolvers() — ES2024

### The Spec

```
Promise.withResolvers():
1. Let C be the this value.
2. Let promiseCapability = ? NewPromiseCapability(C).
3. Let obj = OrdinaryObjectCreate(%Object.prototype%).
4. Perform ! CreateDataPropertyOrThrow(obj, "promise",
     promiseCapability.[[Promise]]).
5. Perform ! CreateDataPropertyOrThrow(obj, "resolve",
     promiseCapability.[[Resolve]]).
6. Perform ! CreateDataPropertyOrThrow(obj, "reject",
     promiseCapability.[[Reject]]).
7. Return obj.
```

It's literally `NewPromiseCapability` exposed to userland:

```javascript
const { promise, resolve, reject } = Promise.withResolvers();

// Use resolve/reject from anywhere:
setTimeout(() => resolve('done'), 1000);
await promise; // 'done'
```

### When to Use It

```javascript
// 1. Adapting callback APIs:
function readFile(path) {
  const { promise, resolve, reject } = Promise.withResolvers();
  fs.readFile(path, (err, data) => err ? reject(err) : resolve(data));
  return promise;
}

// 2. Event-based resolution:
function waitForEvent(target, event) {
  const { promise, resolve } = Promise.withResolvers();
  target.addEventListener(event, resolve, { once: true });
  return promise;
}

// 3. Building async queues:
function createAsyncQueue() {
  let current = Promise.withResolvers();
  return {
    push(value) {
      const prev = current;
      current = Promise.withResolvers();
      prev.resolve({ value, done: false });
    },
    [Symbol.asyncIterator]() {
      return { next: () => current.promise };
    }
  };
}
```

### The Polyfill

```javascript
if (!Promise.withResolvers) {
  Promise.withResolvers = function() {
    let resolve, reject;
    const promise = new this((res, rej) => {
      resolve = res;
      reject = rej;
    });
    return { promise, resolve, reject };
  };
}
// Note: uses `this` to respect subclassing
```

### Implementation Status (as of 2024)

| Engine        | Version     |
|---------------|-------------|
| V8            | Chrome 119+ |
| SpiderMonkey  | Firefox 121+|
| JavaScriptCore| Safari 17.2+|
| Node.js       | 22+         |

---

## Promise Memory Model

### Allocation Costs

Per-promise memory in V8 (pointer-compressed, 64-bit):

| Component           | Bytes | Notes                                |
|---------------------|-------|--------------------------------------|
| JSPromise object    | ~20   | Map + properties + elements + result + flags |
| PromiseReaction     | ~24   | Per `.then()` call while pending     |
| Resolve function    | ~32   | JSFunction + closure context         |
| Reject function     | ~32   | JSFunction + closure context         |
| **Total per new Promise()** | **~104** | Resolve + reject are only created for `new Promise()` |

`Promise.resolve(value)` on a non-thenable creates just the JSPromise
(~20 bytes) — no resolve/reject functions needed since it's already
settled.

### Chain Memory

```javascript
// Each .then() adds:
// - 1 PromiseReaction (~24 bytes) on the source promise
// - 1 new Promise (~20 bytes) as the return value
// - The callback closure (variable, typically ~32-64 bytes)
// Total: ~80-108 bytes per .then() link

const chain = Promise.resolve(0);
for (let i = 0; i < 100_000; i++) {
  chain = chain.then(v => v + 1);
}
// Peak memory: ~100,000 × ~100 bytes = ~10 MB
// As the chain resolves from front to back, earlier promises become
// collectible — but peak memory is the full chain
```

### Promise.all Memory

```javascript
Promise.all(arrayOfNPromises);

// Creates:
// - 1 outer result promise (~20 bytes)
// - 1 values array (8 bytes × N)
// - N PromiseReaction records (~24 bytes each) on input promises
// - N+1 resolve element functions (~32 bytes each, sharing a closure)
// - 1 reject function (shared across all inputs) (~32 bytes)
// - 1 remainingElements counter (~16 bytes)
//
// Total overhead: ~64N + ~68 bytes
// Plus the input promises themselves: ~20N bytes (if pre-existing)
// Grand total: ~84N + ~68 bytes
```

For 10,000 promises: ~84 × 10,000 + 68 ≈ **840 KB** of overhead,
not counting the resolution values themselves.

### GC Behavior

Promises are typically **short-lived** — created, resolved, consumed
within a single task. All major engines allocate them in the nursery
(young generation):

- **V8 (Orinoco)**: Promises in the nursery are collected by the
  Scavenger (Cheney's semi-space copying collector). Promotion to
  old space only happens if the promise survives 2+ GC cycles.

- **SpiderMonkey**: Nursery allocation with the generational GC.
  Promises rarely survive to tenured heap.

- **JavaScriptCore (Riptide)**: Concurrent generational GC. Short-lived
  promises benefit from concurrent collection — no main-thread pause.

**The worst case for GC**: A large `Promise.all` with thousands of
input promises — all values must be held simultaneously in the values
array until every promise resolves. The values array and all results
are promoted to old space if the operation takes longer than a GC cycle.

---

## Edge Cases and Surprises

### The Executor Runs Synchronously

```javascript
console.log('before');
new Promise(() => console.log('executor'));
console.log('after');
// Output: before, executor, after
```

Everything inside the executor runs synchronously in the `Promise`
constructor. Only `.then()` handlers run asynchronously.

### `.then()` Always Returns a New Promise

```javascript
const p = Promise.resolve(42);
p.then() === p.then(); // false — each call creates a new Promise
```

Even `Promise.resolve(p) === p` (identity shortcut) does NOT mean
`p.then() === p` — `.then()` always allocates.

### Throwing in an Executor Rejects

```javascript
const p = new Promise(() => { throw new Error('boom'); });
// p is rejected with Error('boom')
// The constructor catches the throw
```

### `finally()` Is Value-Transparent

```javascript
Promise.resolve(42)
  .finally(() => 99)      // 99 is IGNORED
  .then(v => console.log(v)); // 42

Promise.resolve(42)
  .finally(() => { throw new Error('x'); }) // overrides
  .catch(e => console.log(e.message));      // 'x'
```

`finally` passes through the original value unless the callback throws
or returns a rejected promise.

### Empty Iterable Behavior Is Asymmetric

```javascript
Promise.all([]);         // resolves immediately with []
Promise.allSettled([]);  // resolves immediately with []
Promise.any([]);         // rejects immediately with AggregateError (empty .errors)
Promise.race([]);        // FOREVER PENDING — never settles
```

The mathematical reasoning:
- `all([])`: Are all zero promises fulfilled? Vacuously true → resolve.
- `any([])`: Is any promise fulfilled? No. All rejected? Vacuously true → reject.
- `race([])`: First to settle? None ever will → pending forever.

### Promise.resolve Is Realm-Specific

```javascript
// In an iframe:
const iframePromise = iframe.contentWindow.Promise.resolve(42);

// In the parent:
Promise.resolve(iframePromise) === iframePromise;
// false! Different Promise constructor (different realm)
// A wrapping PromiseResolveThenableJob is created
```

Cross-realm promises are thenables from each other's perspective.

---

## Key Takeaways

1. **Two microtask types** — PromiseReactionJob (runs `.then` handlers)
   and PromiseResolveThenableJob (assimilates thenables). The second
   type is the source of "extra ticks."

2. **The resolve algorithm reads `.then` once** — this matters for
   adversarial thenables and security.

3. **V8 uses tagged unions and bitfields** — a JSPromise is ~20 bytes
   compressed, with reactions as a linked list overwritten by the
   result on settlement.

4. **Promise.race leaks memory** when used with permanently-pending
   promises. Reactions accumulate and can never be collected.

5. **Unhandled rejection tracking differs** — browsers check at
   microtask checkpoint, Node.js at the next event loop tick.
   Always attach handlers synchronously.

6. **Promises/A+ only defines `.then()`** — everything else
   (constructors, combinators, `catch`, `finally`) is ECMA-262.

7. **Promise.withResolvers()** (ES2024) finally exposes the spec's
   `NewPromiseCapability` to userland. Use it instead of the
   deferred anti-pattern.

8. **Promise memory is dominated by reactions and closures** — the
   Promise object itself is cheap (~20 bytes), but each `.then()`
   adds ~80-108 bytes of overhead.
