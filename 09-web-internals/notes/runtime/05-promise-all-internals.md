# Promise.all Internals — The God-Complex Deep Dive

Everything about `Promise.all`: the ECMA-262 specification algorithm,
V8 engine implementation, microtask scheduling, memory characteristics,
TypeScript typing, comparisons with sibling combinators, performance
analysis, concurrency patterns, edge cases, and antipatterns.

---

## Table of Contents

1. [ECMA-262 Specification Algorithm](#1-ecma-262-specification-algorithm)
2. [Promise.all vs allSettled vs race vs any](#2-promiseall-vs-allsettled-vs-race-vs-any)
3. [Microtask Scheduling](#3-microtask-scheduling)
4. [Memory Characteristics](#4-memory-characteristics)
5. [Iterables vs Arrays](#5-iterables-vs-arrays)
6. [Error Handling Edge Cases](#6-error-handling-edge-cases)
7. [Performance Analysis](#7-performance-analysis)
8. [Concurrency Control](#8-concurrency-control)
9. [Event Loop Interaction](#9-event-loop-interaction)
10. [Common Bugs and Antipatterns](#10-common-bugs-and-antipatterns)
11. [TypeScript Typing](#11-typescript-typing)
12. [V8 Engine Implementation Details](#12-v8-engine-implementation-details)

---

## 1. ECMA-262 Specification Algorithm

Reference: ECMA-262, Section 27.2.4.1 (Promise.all)

### 1.1 The Entry Point: Promise.all ( iterable )

The spec defines `Promise.all` as a method on the Promise constructor.
Here is the algorithm, step by step:

```
Promise.all ( iterable )

1. Let C be the this value.
   // C is the constructor — normally Promise, but could be a subclass.

2. Let promiseCapability be ? NewPromiseCapability(C).
   // Creates a new promise + its resolve/reject functions.
   // This is the "outer" promise that Promise.all returns.

3. Let promiseResolve be ? GetPromiseResolve(C).
   // Gets C.resolve — used to wrap each iterable element.
   // This is Promise.resolve, used to coerce values into promises.

4. Let iteratorRecord be GetIterator(iterable, sync).
   // Gets the [Symbol.iterator]() of the iterable.

5. IfAbruptRejectPromise(iteratorRecord, promiseCapability).
   // If getting the iterator throws, reject the outer promise.

6. Let result be PerformPromiseAll(iteratorRecord, C, promiseCapability,
                                    promiseResolve).
   // The core algorithm.

7. If result is an abrupt completion, then
   a. If iteratorRecord.[[Done]] is false, let result be
      Completion(IteratorClose(iteratorRecord, result)).
   b. IfAbruptRejectPromise(result, promiseCapability).

8. Return ? result.
```

### 1.2 GetPromiseResolve ( C )

```
GetPromiseResolve ( promiseConstructor )

1. Let promiseResolve be ? Get(promiseConstructor, "resolve").
2. If IsCallable(promiseResolve) is false, throw a TypeError.
3. Return promiseResolve.
```

This is why monkey-patching `Promise.resolve` affects `Promise.all`.
The spec explicitly looks up `.resolve` on the constructor.

### 1.3 NewPromiseCapability ( C )

This is fundamental to all Promise combinators:

```
NewPromiseCapability ( C )

1. If IsConstructor(C) is false, throw a TypeError.
2. Let promiseCapability be the PromiseCapability Record
   { [[Promise]]: undefined, [[Resolve]]: undefined, [[Reject]]: undefined }.
3. Let executorClosure be a new Abstract Closure with parameters
   (resolve, reject) that captures promiseCapability.
4. Let executor be CreateBuiltinFunction(executorClosure, ...).
5. Let promise be ? Construct(C, << executor >>).
   // This calls new C(executor), which gives us resolve and reject.
6. If IsCallable(promiseCapability.[[Resolve]]) is false, throw TypeError.
7. If IsCallable(promiseCapability.[[Reject]]) is false, throw TypeError.
8. Return promiseCapability.
```

A **PromiseCapability Record** has three fields:
- `[[Promise]]` — the promise object itself
- `[[Resolve]]` — the resolve function
- `[[Reject]]` — the reject function

This is the "deferred" pattern. The outer code holds the resolve/reject
handles while returning the promise to the caller.

### 1.4 PerformPromiseAll — The Core Algorithm

This is where the real work happens:

```
PerformPromiseAll ( iteratorRecord, constructor, resultCapability,
                    promiseResolve )

1.  Let values be a new empty List.
    // This will hold the results.

2.  Let remainingElementsCount be the Record { [[Value]]: 1 }.
    // Starts at 1, not 0. This is the "completion counter."
    // The initial 1 accounts for the iteration itself.

3.  Let index be 0.

4.  Repeat,
    a. Let next be Completion(IteratorStep(iteratorRecord)).
    b. If next is an abrupt completion, set iteratorRecord.[[Done]] to true.
    c. ReturnIfAbrupt(next).
    d. If next is false (iterator exhausted), then
       i.   Set iteratorRecord.[[Done]] to true.
       ii.  Set remainingElementsCount.[[Value]] to
            remainingElementsCount.[[Value]] - 1.
       iii. If remainingElementsCount.[[Value]] is 0, then
            1. Let valuesArray be CreateArrayFromList(values).
            2. Perform ? Call(resultCapability.[[Resolve]], undefined,
                              << valuesArray >>).
       iv.  Return resultCapability.[[Promise]].

    e. Let nextValue be Completion(IteratorValue(next)).
    f. If nextValue is an abrupt completion, set
       iteratorRecord.[[Done]] to true.
    g. ReturnIfAbrupt(nextValue).

    h. Append undefined to values.
       // Pre-allocate the slot. The actual value comes later.

    i. Let nextPromise be ? Call(promiseResolve, constructor,
                                  << nextValue >>).
       // This is Promise.resolve(nextValue) — wraps in a promise.

    j. Let steps be the algorithm steps defined in
       "Promise.all Resolve Element Functions."

    k. Let length be the number of non-optional parameters of
       the function object to be created in the next step.

    l. Let onFulfilled be CreateBuiltinFunction(steps, length, "",
       << [[AlreadyCalled]], [[Index]], [[Values]],
          [[Capability]], [[RemainingElements]] >>).

    m. Set onFulfilled.[[AlreadyCalled]] to false.

    n. Set onFulfilled.[[Index]] to index.

    o. Set onFulfilled.[[Values]] to values.

    p. Set onFulfilled.[[Capability]] to resultCapability.

    q. Set onFulfilled.[[RemainingElements]] to remainingElementsCount.
       // Note: this is a REFERENCE to the shared counter, not a copy.

    r. Set remainingElementsCount.[[Value]] to
       remainingElementsCount.[[Value]] + 1.
       // Increment for each promise we attach to.

    s. Perform ? Invoke(nextPromise, "then",
                         << onFulfilled, resultCapability.[[Reject]] >>).
       // nextPromise.then(onFulfilled, reject)
       // Note: ALL promises share the SAME reject function.
       // First rejection wins.

    t. Set index to index + 1.
```

### 1.5 Promise.all Resolve Element Functions

Each element gets its own resolve handler. Here is the algorithm:

```
Promise.all Resolve Element Functions

A function F with parameter (x):

1. Let F be the active function object.
2. Let alreadyCalled be F.[[AlreadyCalled]].
3. If alreadyCalled is true, return undefined.
   // Guard against double-resolution.
4. Set F.[[AlreadyCalled]] to true.
5. Let index be F.[[Index]].
6. Let values be F.[[Values]].
7. Let promiseCapability be F.[[Capability]].
8. Let remainingElementsCount be F.[[RemainingElements]].
9. Set values[index] to x.
   // Place the resolved value at the correct index.
   // This is why results maintain input order regardless of
   // resolution order.
10. Set remainingElementsCount.[[Value]] to
    remainingElementsCount.[[Value]] - 1.
11. If remainingElementsCount.[[Value]] is 0, then
    a. Let valuesArray be CreateArrayFromList(values).
    b. Return ? Call(promiseCapability.[[Resolve]], undefined,
                      << valuesArray >>).
    // All promises resolved — resolve the outer promise.
12. Return undefined.
```

### 1.6 Critical Design Details from the Spec

**The remainingElementsCount starts at 1, not 0:**
This prevents a race condition. If all promises resolve synchronously
during iteration, the counter would hit 0 mid-loop. By starting at 1
and decrementing after the loop completes (step 4.d.ii), the spec
ensures the outer promise is only resolved after ALL elements have
been processed.

```javascript
// Without the +1 trick, this would break:
Promise.all([
  Promise.resolve(1),  // Resolves synchronously during iteration
  Promise.resolve(2),  // Resolves synchronously during iteration
]);
// Counter: 0 + 1 (first) + 1 (second) = 2
// First resolves: 2 - 1 = 1
// Second resolves: 1 - 1 = 0 → would try to resolve mid-iteration!
// With the +1 trick:
// Counter starts at 1: 1 + 1 + 1 = 3
// First resolves: 3 - 1 = 2
// Second resolves: 2 - 1 = 1
// Loop ends: 1 - 1 = 0 → NOW resolve the outer promise.
```

**Each resolve element gets its own [[AlreadyCalled]] flag:**
This prevents any single element's resolution handler from being called
twice. Even if a thenable calls its resolve callback multiple times,
only the first invocation has effect.

**All elements share the SAME reject function:**
The spec passes `resultCapability.[[Reject]]` directly to every
`.then()` call. The first rejection immediately rejects the outer
promise. Subsequent rejections are ignored because the outer promise
is already settled (promises can only be settled once).

**Order preservation via [[Index]]:**
Each resolve element function captures its `index`. When it fires,
it writes to `values[index]`. This guarantees the output array matches
the input order, regardless of which promise resolves first.

---

## 2. Promise.all vs allSettled vs race vs any

### 2.1 Behavioral Comparison

```
                    ┌─────────────┬─────────────┬─────────────┬─────────────┐
                    │ Promise.all │  allSettled  │ Promise.race│ Promise.any │
┌───────────────────┼─────────────┼─────────────┼─────────────┼─────────────┤
│ Resolves when     │ ALL fulfill │ ALL settle  │ FIRST       │ FIRST       │
│                   │             │             │ settles     │ fulfills    │
├───────────────────┼─────────────┼─────────────┼─────────────┼─────────────┤
│ Rejects when      │ ANY rejects │ NEVER       │ FIRST       │ ALL reject  │
│                   │ (fail-fast) │ (always     │ settles     │             │
│                   │             │  fulfills)  │ (if reject) │             │
├───────────────────┼─────────────┼─────────────┼─────────────┼─────────────┤
│ Result type       │ T[]         │ PromiseSet- │ T           │ T           │
│                   │             │ tledResult[]│             │             │
├───────────────────┼─────────────┼─────────────┼─────────────┼─────────────┤
│ Empty iterable    │ Resolves [] │ Resolves [] │ Hangs       │ Rejects     │
│                   │             │             │ forever     │ AggregateErr│
├───────────────────┼─────────────┼─────────────┼─────────────┼─────────────┤
│ Spec section      │ 27.2.4.1   │ 27.2.4.2   │ 27.2.4.3   │ 27.2.4.4   │
└───────────────────┴─────────────┴─────────────┴─────────────┴─────────────┘
```

### 2.2 Internal Algorithmic Differences

**Promise.all — PerformPromiseAll:**
- Creates one resolve handler per element, shares one reject handler.
- Resolve handler: writes value to `values[index]`, decrements counter.
- Reject handler: immediately rejects the outer promise. First rejection
  wins; subsequent rejections are no-ops (promise already settled).

**Promise.allSettled — PerformPromiseAllSettled:**
- Creates BOTH a resolve handler AND a reject handler per element.
- Resolve handler: writes `{ status: "fulfilled", value: x }` to
  `values[index]`, decrements counter.
- Reject handler: writes `{ status: "rejected", reason: x }` to
  `values[index]`, decrements counter.
- The outer promise NEVER rejects (unless the iterator itself throws).
- Both handlers use the same `[[AlreadyCalled]]` flag per element.

```javascript
// allSettled resolve element:
//   Set values[index] to { status: "fulfilled", value: x }
//   remainingElementsCount - 1
//   If 0, resolve outer promise

// allSettled reject element:
//   Set values[index] to { status: "rejected", reason: x }
//   remainingElementsCount - 1
//   If 0, resolve outer promise  ← resolves, never rejects!
```

**Promise.race — PerformPromiseRace:**
The simplest combinator. The algorithm is radically different:

```
PerformPromiseRace ( iteratorRecord, constructor, resultCapability,
                     promiseResolve )

1. Repeat,
   a. Let next be IteratorStep(iteratorRecord).
   b. If next is false, return resultCapability.[[Promise]].
   c. Let nextValue be IteratorValue(next).
   d. Let nextPromise be Call(promiseResolve, constructor, << nextValue >>).
   e. Perform Invoke(nextPromise, "then",
        << resultCapability.[[Resolve]], resultCapability.[[Reject]] >>).
```

That is it. No counter, no values array, no per-element handlers.
Every promise in the iterable gets the SAME resolve AND reject
functions. The first one to settle wins. All subsequent settlements
are no-ops because the outer promise is already settled.

**Promise.any — PerformPromiseAny:**
The mirror image of Promise.all:

- Creates one reject handler per element, shares one resolve handler.
- Reject handler: writes error to `errors[index]`, decrements counter.
- Resolve handler: immediately resolves the outer promise.
- If ALL reject: rejects with `AggregateError(errors)`.

```javascript
// Promise.any is the dual of Promise.all:
// Promise.all:  all-fulfill → resolve,  any-reject → reject
// Promise.any:  any-fulfill → resolve,  all-reject → reject(AggregateError)
```

### 2.3 Memory Allocation Differences

```
Promise.all:
  - 1 values array (N slots)
  - N resolve element functions (each with captured state)
  - 1 shared reject function
  - 1 remainingElementsCount record

Promise.allSettled:
  - 1 values array (N slots)
  - N resolve element functions
  - N reject element functions  ← 2x the handlers vs Promise.all
  - 1 remainingElementsCount record

Promise.race:
  - 0 arrays, 0 per-element functions
  - Just reuses the outer promise's resolve/reject
  - Minimal memory overhead

Promise.any:
  - 1 errors array (N slots)
  - N reject element functions
  - 1 shared resolve function
  - 1 remainingElementsCount record
```

### 2.4 Edge Case: Empty Iterable

```javascript
Promise.all([]);
// Resolves immediately with [].
// remainingElementsCount starts at 1, loop never runs,
// decrements to 0 after loop, resolves.

Promise.allSettled([]);
// Resolves immediately with [].
// Same mechanism as Promise.all.

Promise.race([]);
// Returns a FOREVER-PENDING promise!
// The loop never runs, so nothing ever calls resolve or reject.
// The returned promise never settles. It is permanently pending.

Promise.any([]);
// Rejects immediately with AggregateError([]).
// remainingElementsCount starts at 1, loop never runs,
// decrements to 0, rejects with empty AggregateError.
```

---

## 3. Microtask Scheduling

### 3.1 How Many Microtasks Does Promise.all Create?

For N input promises, `Promise.all` generates the following microtasks:

**Phase 1 — Setup (synchronous, no microtasks):**
The iteration over the input array, calling `Promise.resolve()` on each
element, and attaching `.then()` handlers all happen synchronously during
the `Promise.all()` call.

**Phase 2 — Element Resolution (N microtasks minimum):**
When each input promise resolves, its `.then()` handler (the resolve
element function) is enqueued as a microtask. That is N microtasks
for N promises.

However, `Promise.resolve(alreadyResolvedPromise)` has an optimization
in the spec. If the value is already a promise with the same constructor,
`Promise.resolve` returns it directly (no wrapping). But `.then()` still
creates a PromiseReaction that fires as a microtask.

**Phase 3 — Outer Promise Resolution (1-2 additional microtasks):**
When the last resolve element fires and decrements the counter to 0,
it calls the outer promise's resolve function. Resolving the outer
promise enqueues reactions (any `.then()` handlers chained on the
`Promise.all()` result) as microtasks.

**Total microtask count for N already-resolved promises:**

```
N (resolve element handlers) + 1 (outer promise resolve reaction)
= N + 1 microtasks minimum
```

But there is a subtlety. For promises that are not yet resolved when
`Promise.all` is called, the microtask scheduling is:

```
For each input promise that resolves:
  Microtask 1: PromiseReaction for the input promise's .then()
               → runs the resolve element function
               → if last element, calls outer resolve

When outer promise resolves:
  Microtask 2: PromiseReaction for any .then() chained on Promise.all()

When chaining with await:
  Microtask 3: async function continuation
```

### 3.2 Scheduling Order with Already-Resolved Promises

```javascript
console.log('1: sync start');

const p = Promise.all([
  Promise.resolve('a'),
  Promise.resolve('b'),
  Promise.resolve('c'),
]);

p.then(values => console.log('4: all resolved', values));

Promise.resolve().then(() => console.log('2: independent microtask'));
Promise.resolve().then(() => console.log('3: another microtask'));

console.log('5: sync end');
```

Execution order:
```
1: sync start                   ← synchronous
5: sync end                     ← synchronous
2: independent microtask        ← microtask (queued during sync)
3: another microtask            ← microtask (queued during sync)

At this point the microtask queue also has:
  - resolve element for 'a' (queued by .then on Promise.resolve('a'))
  - resolve element for 'b'
  - resolve element for 'c'

These were queued DURING the synchronous Promise.all() call,
because Promise.resolve('a') is already resolved, so .then()
immediately enqueues the reaction.

After the independent microtasks drain, the resolve elements fire:
  - resolve element 'a': values[0] = 'a', counter 3→2
  - resolve element 'b': values[1] = 'b', counter 2→1
  - resolve element 'c': values[2] = 'c', counter 1→0 → resolve outer

This enqueues the outer promise's reaction:
4: all resolved ['a', 'b', 'c'] ← microtask (from outer resolve)
```

Wait — the order of 2/3 versus the resolve elements depends on
queueing order. Let me be precise:

During the synchronous `Promise.all([...])` call:
1. `Promise.resolve('a').then(resolveElement0, reject)` — since the
   promise is already resolved, this enqueues resolveElement0.
2. `Promise.resolve('b').then(resolveElement1, reject)` — enqueues.
3. `Promise.resolve('c').then(resolveElement2, reject)` — enqueues.

Then after `Promise.all` returns:
4. `Promise.resolve().then(...)` — enqueues the '2: independent' callback.
5. `Promise.resolve().then(...)` — enqueues the '3: another' callback.

The microtask queue at the end of synchronous execution:
```
[resolveElement0, resolveElement1, resolveElement2,
 independentMicrotask, anotherMicrotask]
```

So the actual output is:
```
1: sync start
5: sync end
(resolveElement0 fires — no visible output)
(resolveElement1 fires — no visible output)
(resolveElement2 fires — counter hits 0, outer resolves,
 enqueues the .then callback on p)
2: independent microtask
3: another microtask
4: all resolved ['a', 'b', 'c']
```

The resolve elements fire FIRST because they were enqueued first.
The `.then(values => ...)` callback fires LAST because it was only
enqueued when the outer promise resolved (which happened during
resolveElement2).

### 3.3 Scheduling with Async Promises

```javascript
const slow = new Promise(resolve => setTimeout(() => resolve('slow'), 100));
const fast = new Promise(resolve => setTimeout(() => resolve('fast'), 50));

Promise.all([slow, fast]).then(([s, f]) => {
  console.log(s, f);  // 'slow', 'fast' — order preserved!
});
```

Timeline:
```
t=0ms:   Promise.all called. Both promises are pending.
         .then(resolveElement, reject) attached to each.
         No microtasks enqueued yet (promises aren't resolved).

t=50ms:  fast resolves. This enqueues resolveElement1 as a microtask.
         Microtask fires: values[1] = 'fast', counter 2→1.
         Counter is not 0, so outer promise stays pending.

t=100ms: slow resolves. This enqueues resolveElement0 as a microtask.
         Microtask fires: values[0] = 'slow', counter 1→0.
         Counter is 0! Outer promise resolves with ['slow', 'fast'].
         .then() callback enqueued as microtask.
         .then() fires: logs 'slow', 'fast'.
```

### 3.4 Microtask Count for Rejection

```javascript
Promise.all([
  Promise.resolve('a'),
  Promise.reject('error'),
  Promise.resolve('c'),
]).catch(err => console.log(err));
```

Microtask breakdown:
```
During synchronous Promise.all():
  - .then(resolveElement0, reject) on resolved promise → enqueue resolveElement0
  - .then(resolveElement1, reject) on rejected promise → enqueue reject
  - .then(resolveElement2, reject) on resolved promise → enqueue resolveElement2

Microtask queue: [resolveElement0, reject, resolveElement2]

resolveElement0 fires: values[0] = 'a', counter 3→2
reject fires: outer promise rejects with 'error'
  → enqueues .catch() callback
resolveElement2 fires: values[2] = 'c', counter 2→1
  (but outer promise already settled — this is wasted work)
.catch fires: logs 'error'
```

Key insight: **even after rejection, the remaining resolve element
functions still fire**. They just have no effect because the outer
promise is already settled. This is wasted microtask work. The spec
does NOT short-circuit iteration or handler execution on rejection.

---

## 4. Memory Characteristics

### 4.1 What Promise.all Holds in Memory

During execution, `Promise.all` retains:

```
1. The outer promise (PromiseCapability record)
   - The Promise object
   - The resolve function (closure)
   - The reject function (closure)

2. The values array (List)
   - Pre-allocated with N undefined slots
   - Filled as promises resolve
   - Held until outer promise resolves

3. The remainingElementsCount record
   - A mutable { [[Value]]: number } shared by all handlers

4. N resolve element functions, each capturing:
   - [[AlreadyCalled]]: boolean
   - [[Index]]: integer
   - [[Values]]: reference to the shared values array
   - [[Capability]]: reference to the outer promise capability
   - [[RemainingElements]]: reference to the shared counter

5. Each input promise retains its PromiseReaction records
   (the .then handlers) until it settles
```

### 4.2 Memory Per Element

Each element of Promise.all costs approximately:

```
Per promise:
  - 1 resolve element function object: ~64-128 bytes (V8)
    (function object + internal slots for captured state)
  - 1 PromiseReaction record on the input promise: ~48 bytes
  - 1 slot in the values array: 8 bytes (tagged pointer)

Total per element: ~120-184 bytes
```

For 1 million promises:
```
~120-184 MB just for the Promise.all machinery
+ the promises themselves (each ~80-120 bytes in V8)
+ whatever values the promises resolve with
```

### 4.3 When Memory Is Released

```
Timeline:
  1. Promise.all([p1, p2, p3]) called
     → values array allocated, resolve element functions created

  2. p1 resolves
     → resolveElement0 fires, sets values[0]
     → resolveElement0 is now GC-eligible (if no other references)
     → p1's PromiseReaction is cleared
     → BUT p1's resolved value is now held in values array

  3. p2 resolves (same as above)

  4. p3 resolves → counter hits 0
     → Outer promise resolves with values array
     → The values array is now owned by the .then() handler
     → All resolve element functions are GC-eligible
     → The remainingElementsCount record is GC-eligible

  5. .then() handler runs
     → User code receives the values array
     → If user drops reference, everything is GC-eligible
```

Critical detail: **the values array holds ALL resolved values
simultaneously in memory**. If you have 1000 promises each
resolving to a 1MB buffer, Promise.all holds 1GB in memory when
the last promise resolves.

### 4.4 Memory Comparison

```javascript
// Promise.all: holds ALL results simultaneously
const results = await Promise.all(urls.map(url => fetch(url)));
// All responses in memory at once

// Sequential: can process and discard one at a time
for (const url of urls) {
  const result = await fetch(url);
  processAndDiscard(result);
}
// Only one response in memory at a time

// Stream processing: never holds more than the batch
async function* batchFetch(urls, batchSize) {
  for (let i = 0; i < urls.length; i += batchSize) {
    const batch = urls.slice(i, i + batchSize);
    yield await Promise.all(batch.map(url => fetch(url)));
  }
}
// Only batchSize responses in memory at a time
```

### 4.5 V8's Internal Promise Object Size

In V8, a Promise object on the heap:

```
JSPromise object layout (V8):
┌────────────────────────┐
│ Map pointer            │  8 bytes (compressed: 4)
├────────────────────────┤
│ Properties pointer     │  8 bytes (compressed: 4)
├────────────────────────┤
│ Elements pointer       │  8 bytes (compressed: 4)
├────────────────────────┤
│ Reactions or result    │  8 bytes (compressed: 4)
├────────────────────────┤
│ Flags                  │  4 bytes
│  - status (2 bits)     │  pending/fulfilled/rejected
│  - has_handler (1 bit) │  for unhandled rejection tracking
│  - handled_hint (1 bit)│
│  - async_task_id       │
└────────────────────────┘
Total: ~40 bytes (compressed pointers) to ~52 bytes
```

A PromiseReaction (the internal record for a `.then()` handler):

```
PromiseReaction layout (V8):
┌────────────────────────┐
│ Map pointer            │  4 bytes (compressed)
├────────────────────────┤
│ Next reaction          │  4 bytes (linked list for multiple .then())
├────────────────────────┤
│ Reject handler         │  4 bytes
├────────────────────────┤
│ Fulfill handler        │  4 bytes
├────────────────────────┤
│ Promise or capability  │  4 bytes
└────────────────────────┘
Total: ~20-24 bytes (compressed)
```

---

## 5. Iterables vs Arrays

### 5.1 How the Spec Handles Iterables

Promise.all does NOT require an array. It requires an **iterable** —
any object with a `[Symbol.iterator]()` method. The spec uses
`GetIterator(iterable, sync)` which calls `iterable[Symbol.iterator]()`.

```javascript
// All of these work:

// Array
Promise.all([p1, p2, p3]);

// Set
Promise.all(new Set([p1, p2, p3]));

// Generator
function* gen() {
  yield p1;
  yield p2;
  yield p3;
}
Promise.all(gen());

// Map.values()
const map = new Map([['a', p1], ['b', p2]]);
Promise.all(map.values());

// Custom iterable
const iterable = {
  [Symbol.iterator]() {
    let i = 0;
    const promises = [p1, p2, p3];
    return {
      next() {
        if (i < promises.length) {
          return { value: promises[i++], done: false };
        }
        return { done: true };
      }
    };
  }
};
Promise.all(iterable);
```

### 5.2 Lazy Iteration

A critical implication: the iterable is consumed **eagerly and
synchronously** during the `Promise.all()` call. The spec iterates
the entire iterable in step 4's Repeat loop before returning.

```javascript
function* lazyPromises() {
  console.log('yielding 1');
  yield fetch('/api/1');
  console.log('yielding 2');
  yield fetch('/api/2');
  console.log('yielding 3');
  yield fetch('/api/3');
}

// ALL three fetches are initiated synchronously:
Promise.all(lazyPromises());
// Output (synchronous):
// yielding 1
// yielding 2
// yielding 3
// All three fetch() calls have already been made.
```

This means you cannot use a generator to lazily produce promises for
concurrency control. Promise.all consumes everything immediately.

### 5.3 Async Iterables

**Promise.all does NOT support async iterables.** It uses
`GetIterator(iterable, sync)`, not `GetIterator(iterable, async)`.

```javascript
async function* asyncGen() {
  yield await fetch('/api/1');
  yield await fetch('/api/2');
}

// This does NOT work as expected:
Promise.all(asyncGen());
// asyncGen() returns an AsyncGenerator object.
// AsyncGenerator has [Symbol.asyncIterator], NOT [Symbol.iterator].
// GetIterator with sync hint will fail or fall back to @@iterator.

// What actually happens: asyncGen() does have [Symbol.iterator]
// (inherited from %AsyncGeneratorPrototype%), but calling .next()
// returns { value: Promise, done: false } — the values are promises.
// Actually, AsyncGenerator does NOT have @@iterator, so this throws:
// TypeError: object is not iterable
```

If you need to work with async iterables, you must collect them first:

```javascript
async function collectAsync(asyncIterable) {
  const results = [];
  for await (const item of asyncIterable) {
    results.push(item);
  }
  return results;
}

// Or use Array.fromAsync (ES2024):
const results = await Array.fromAsync(asyncGen());
```

### 5.4 Iterator Protocol Error Handling

The spec is careful about iterator errors:

```javascript
const badIterable = {
  [Symbol.iterator]() {
    return {
      next() {
        throw new Error('iterator exploded');
      }
    };
  }
};

Promise.all(badIterable).catch(err => {
  console.log(err.message);  // 'iterator exploded'
  // The outer promise is rejected.
  // The spec wraps iterator errors via IfAbruptRejectPromise.
});
```

If the iterator throws AFTER some promises have been processed:

```javascript
let count = 0;
const partialBadIterable = {
  [Symbol.iterator]() {
    return {
      next() {
        count++;
        if (count === 3) throw new Error('died at 3');
        return { value: Promise.resolve(count), done: false };
      }
    };
  }
};

Promise.all(partialBadIterable).catch(err => {
  console.log(err.message);  // 'died at 3'
  // Promises 1 and 2 were already attached with .then() handlers.
  // Those handlers will still fire as microtasks, but the outer
  // promise is already rejected, so they have no effect.
});
```

The spec also handles the case where `IteratorClose` itself throws
(step 7 of Promise.all). If the iterator has a `.return()` method
and calling it throws, that error is used to reject the outer promise.

---

## 6. Error Handling Edge Cases

### 6.1 Empty Array

```javascript
const result = await Promise.all([]);
console.log(result);  // []
```

Per the spec: the loop body never executes. After the loop,
`remainingElementsCount` is decremented from 1 to 0, which triggers
the resolve with an empty values array. This happens synchronously
during the Promise.all() call, but the resolution itself is
asynchronous (the outer promise's `.then()` handlers fire as
microtasks).

### 6.2 Non-Promise Values

```javascript
const result = await Promise.all([1, 'hello', true, null, undefined]);
console.log(result);  // [1, 'hello', true, null, undefined]
```

Each value goes through `Promise.resolve(value)` (step 4.i in the
spec). For non-thenable values, `Promise.resolve(x)` returns a
new already-resolved promise with value `x`. This means each
non-promise value gets wrapped in a promise, which then immediately
enqueues its `.then()` handler.

### 6.3 Thenables (Non-Promise Objects with .then)

```javascript
const thenable = {
  then(resolve, reject) {
    setTimeout(() => resolve('thenable value'), 100);
  }
};

const result = await Promise.all([thenable, Promise.resolve('real promise')]);
console.log(result);  // ['thenable value', 'real promise']
```

The thenable is processed through `Promise.resolve(thenable)`:
1. `Promise.resolve` detects that `thenable` has a `.then` method.
2. It creates a new promise and calls `thenable.then(resolve, reject)`.
3. This `.then()` call is scheduled as a **microtask** (per the spec's
   NewPromiseResolveThenableJob).

Important: the thenable's `.then()` is NOT called synchronously.
It is wrapped in a PromiseResolveThenableJob and enqueued on the
microtask queue. This prevents thenables from synchronously resolving
or rejecting in unexpected ways.

```javascript
// Dangerous thenable that tries to be synchronous:
const evilThenable = {
  then(resolve) {
    resolve('sync!');
    // Even though this calls resolve synchronously,
    // the resolution is deferred to a microtask by the spec.
  }
};

Promise.all([evilThenable]).then(result => {
  console.log(result);  // ['sync!'] — but arrived via microtask
});
```

### 6.4 Multiple Rejections — Which Error Wins?

```javascript
Promise.all([
  new Promise((_, reject) => setTimeout(() => reject('error-A'), 100)),
  new Promise((_, reject) => setTimeout(() => reject('error-B'), 50)),
  new Promise((_, reject) => setTimeout(() => reject('error-C'), 150)),
]).catch(err => {
  console.log(err);  // 'error-B' — first to reject wins
});
```

The spec shares a SINGLE reject function across all elements.
A promise can only be settled once. The first call to `reject`
transitions the outer promise from pending to rejected. All
subsequent calls to `reject` are no-ops.

**For already-rejected promises:**

```javascript
Promise.all([
  Promise.reject('first'),
  Promise.reject('second'),
  Promise.reject('third'),
]).catch(err => {
  console.log(err);  // 'first'
});
```

The iteration processes them left-to-right. For each rejected promise,
`.then(resolveElement, reject)` is called. Since the promises are
already rejected, the reject reaction is immediately enqueued. The
first rejection microtask to fire wins.

Since they are enqueued in order during iteration, 'first' is
enqueued first and therefore fires first.

### 6.5 Mixed Resolved and Rejected

```javascript
Promise.all([
  Promise.resolve('a'),
  Promise.reject('error'),
  Promise.resolve('c'),
]).catch(err => {
  console.log(err);  // 'error'
});
```

Timeline:
```
Synchronous: iterate all three, enqueue handlers:
  - resolveElement0 (for 'a')
  - reject (for 'error')
  - resolveElement2 (for 'c')

Microtask 1: resolveElement0 fires → values[0] = 'a', counter 3→2
Microtask 2: reject fires → outer promise rejected with 'error'
  → .catch() handler enqueued
Microtask 3: resolveElement2 fires → values[2] = 'c', counter 2→1
  → but outer promise already settled, this does nothing useful
Microtask 4: .catch() fires → logs 'error'
```

### 6.6 Rejection After All Others Resolve

```javascript
Promise.all([
  Promise.resolve('a'),
  Promise.resolve('b'),
  new Promise((_, reject) => setTimeout(() => reject('late error'), 1000)),
]).catch(err => {
  console.log(err);  // 'late error' — after 1 second
});
```

Even though 'a' and 'b' resolve immediately, the outer promise
waits. The reject still wins because the counter never reaches 0
(it goes 3→2→1 but never hits 0 because the third promise hasn't
resolved — and it rejects instead).

### 6.7 Promise That Resolves to Another Promise

```javascript
const inner = new Promise(resolve => setTimeout(() => resolve('deep'), 100));
const outer = Promise.resolve(inner);
// outer is NOT resolved with inner as its value.
// outer FOLLOWS inner — it will resolve to 'deep' after 100ms.

Promise.all([outer]).then(([val]) => {
  console.log(val);  // 'deep', not a Promise object
});
```

When `Promise.resolve(value)` receives a thenable, it recursively
unwraps. The spec's promise resolution procedure (ResolvePromise)
checks if the resolution value is a thenable and, if so, enqueues
a PromiseResolveThenableJob. This is recursive until a non-thenable
value is reached.

### 6.8 Promise.resolve Shortcut Optimization

```javascript
const p = Promise.resolve(42);
const q = Promise.resolve(p);
// q === p  (same object!)
// Spec 27.2.4.7 step 2: if value is a promise whose constructor
// matches the receiver, return value directly.
```

This means `Promise.all([existingPromise])` does NOT create a
wrapper promise around `existingPromise`. The `.then()` is called
directly on the original promise.

### 6.9 Throwing in a .then Handler

```javascript
Promise.all([
  Promise.resolve(1).then(v => { throw new Error('boom'); }),
  Promise.resolve(2),
]).catch(err => {
  console.log(err.message);  // 'boom'
  // The thrown error rejects the promise returned by .then(),
  // which is what Promise.all sees as its first element.
});
```

### 6.10 Non-Iterable Argument

```javascript
Promise.all(42).catch(err => {
  console.log(err);  // TypeError: number 42 is not iterable
  // The GetIterator call throws, which rejects the outer promise.
});

Promise.all(null).catch(err => {
  console.log(err);  // TypeError: Cannot read properties of null
});
```

---

## 7. Performance Analysis

### 7.1 Promise.all Overhead vs Manual Chaining

Promise.all has overhead compared to manually tracking promises:

```javascript
// Promise.all creates:
// - 1 outer promise + capability
// - N resolve element function objects
// - 1 values array
// - N .then() calls (each creating a PromiseReaction)
// Total: O(N) memory, O(N) function allocations

// Manual equivalent:
function manualAll(promises) {
  return new Promise((resolve, reject) => {
    const results = new Array(promises.length);
    let remaining = promises.length;
    if (remaining === 0) return resolve(results);

    promises.forEach((p, i) => {
      Promise.resolve(p).then(
        value => {
          results[i] = value;
          if (--remaining === 0) resolve(results);
        },
        reject
      );
    });
  });
}
// This is essentially what Promise.all does, but the spec-mandated
// version has additional safety (AlreadyCalled guards, proper
// iterator protocol handling, IfAbruptRejectPromise wrapping).
```

The overhead of Promise.all compared to a manual approach is minimal:
a few extra per-element function objects for the AlreadyCalled guard.

### 7.2 Scaling Characteristics

```
N promises  | Time overhead      | Memory overhead
------------|--------------------|-----------------
10          | negligible (~0.1ms)| ~2KB
100         | negligible (~0.5ms)| ~18KB
1,000       | ~2-5ms             | ~180KB
10,000      | ~20-50ms           | ~1.8MB
100,000     | ~200-500ms         | ~18MB
1,000,000   | ~2-5 seconds       | ~180MB
```

The bottleneck is function object creation and microtask processing.
Each resolve element function is a distinct object with its own
captured slots. V8 can optimize these with hidden classes (all
resolve element functions share the same shape), but they still
require heap allocation.

### 7.3 Already-Resolved vs Pending Promises

```javascript
// Already resolved — fastest case
// All handlers fire in the same microtask checkpoint
console.time('all-resolved');
await Promise.all(Array.from({ length: 10000 }, () => Promise.resolve(1)));
console.timeEnd('all-resolved');
// ~2-5ms (mostly microtask processing)

// Pending promises — adds event loop latency
console.time('all-pending');
await Promise.all(Array.from({ length: 10000 }, () =>
  new Promise(resolve => setTimeout(resolve, 0))
));
console.timeEnd('all-pending');
// ~10-50ms (setTimeout overhead + microtask processing)
```

### 7.4 Promise.all vs for-await Sequential

```javascript
// Parallel with Promise.all
async function parallel(urls) {
  const results = await Promise.all(urls.map(url => fetch(url)));
  return results;
}
// Time: max(individual fetch times) + overhead
// For 10 requests of 100ms each: ~100ms

// Sequential with for-await
async function sequential(urls) {
  const results = [];
  for (const url of urls) {
    results.push(await fetch(url));
  }
  return results;
}
// Time: sum(individual fetch times)
// For 10 requests of 100ms each: ~1000ms
```

### 7.5 V8-Specific Optimizations

V8 has special fast paths for Promise.all:

**1. Built-in implementation (not user-land JS):**
Promise.all is implemented in V8's C++ / Torque / CSA (CodeStubAssembler)
builtins, not in JavaScript. This avoids the overhead of parsing and
optimizing JavaScript. The `PerformPromiseAll` function is a native
builtin that directly manipulates V8's internal promise structures.

**2. Fast path for arrays of native promises:**
When the input is a plain array and all elements are native Promise
objects (not subclasses, not thenables), V8 can skip some of the
spec's generic iterable/constructor checks.

**3. Promise.resolve fast path:**
When calling `Promise.resolve(value)` where `value` is already a
native Promise with the correct constructor, V8 returns it directly
without creating a wrapper.

**4. Microtask batching:**
V8's microtask queue is a simple FIFO queue implemented as a resizable
array. Enqueuing and dequeuing are O(1) amortized. The microtask
checkpoint drains the entire queue without returning to the event
loop, which means no event loop overhead between resolve element
function executions.

---

## 8. Concurrency Control

### 8.1 Why Promise.all Has No Concurrency Limit

Promise.all initiates ALL promises immediately during iteration.
There is no built-in mechanism to limit how many run concurrently.

The reason is philosophical and architectural:
- Promises represent **already-started** computations.
- By the time you pass a promise to Promise.all, the work has
  already begun.
- Concurrency control must happen BEFORE promise creation, not after.

```javascript
// This does NOT control concurrency:
const promises = urls.map(url => fetch(url));  // All 1000 fetches started!
const results = await Promise.all(promises);   // Just waits for all.
// Promise.all didn't start anything — the .map() did.
```

### 8.2 Concurrency Pool Pattern

```javascript
async function promisePool(tasks, concurrency) {
  const results = [];
  const executing = new Set();

  for (const [index, task] of tasks.entries()) {
    const promise = task().then(result => {
      executing.delete(promise);
      results[index] = result;
    });
    executing.add(promise);

    if (executing.size >= concurrency) {
      await Promise.race(executing);
    }
  }

  await Promise.all(executing);
  return results;
}

// Usage:
const tasks = urls.map(url => () => fetch(url));  // Thunks, not promises!
const results = await promisePool(tasks, 5);       // Max 5 concurrent
```

### 8.3 Semaphore Pattern

```javascript
class Semaphore {
  constructor(max) {
    this.max = max;
    this.count = 0;
    this.queue = [];
  }

  async acquire() {
    if (this.count < this.max) {
      this.count++;
      return;
    }
    await new Promise(resolve => this.queue.push(resolve));
  }

  release() {
    this.count--;
    if (this.queue.length > 0) {
      this.count++;
      const next = this.queue.shift();
      next();
    }
  }
}

// Usage:
const semaphore = new Semaphore(5);
const results = await Promise.all(urls.map(async url => {
  await semaphore.acquire();
  try {
    return await fetch(url);
  } finally {
    semaphore.release();
  }
}));
```

### 8.4 Batch Pattern

```javascript
async function batchAll(items, batchSize, fn) {
  const results = [];
  for (let i = 0; i < items.length; i += batchSize) {
    const batch = items.slice(i, i + batchSize);
    const batchResults = await Promise.all(batch.map(fn));
    results.push(...batchResults);
  }
  return results;
}

// Usage:
const results = await batchAll(urls, 10, url => fetch(url));
// Processes 10 at a time, waits for all 10, then next 10.
// Downside: if 9/10 finish in 10ms but 1 takes 5s, the
// next batch waits the full 5s. The pool pattern is better.
```

### 8.5 p-limit / p-map Pattern (npm ecosystem)

The popular `p-limit` package implements the semaphore pattern:

```javascript
// Conceptual implementation of p-limit:
function pLimit(concurrency) {
  const queue = [];
  let activeCount = 0;

  const next = () => {
    activeCount--;
    if (queue.length > 0) {
      queue.shift()();
    }
  };

  const run = async (fn, resolve, args) => {
    activeCount++;
    const result = (async () => fn(...args))();
    resolve(result);
    try {
      await result;
    } catch {}
    next();
  };

  const enqueue = (fn, resolve, args) => {
    queue.push(run.bind(undefined, fn, resolve, args));
    (async () => {
      // This is needed because the queue is processed
      // asynchronously. Wait a microtick to ensure
      // activeCount is updated before checking.
      await Promise.resolve();
      if (activeCount < concurrency && queue.length > 0) {
        queue.shift()();
      }
    })();
  };

  const generator = (fn, ...args) =>
    new Promise(resolve => {
      enqueue(fn, resolve, args);
    });

  generator.activeCount = () => activeCount;
  generator.pendingCount = () => queue.length;

  return generator;
}

// Usage:
const limit = pLimit(5);
const results = await Promise.all(
  urls.map(url => limit(() => fetch(url)))
);
```

---

## 9. Event Loop Interaction

### 9.1 Promise.all with All-Synchronous Resolution

When all input promises are already resolved, Promise.all does NOT
resolve synchronously. The resolve element handlers are microtasks:

```javascript
console.log('sync start');

const p = Promise.all([
  Promise.resolve(1),
  Promise.resolve(2),
  Promise.resolve(3),
]);

console.log('sync end');
console.log('p is still pending here:', p);

// Even though all inputs are resolved, p does not resolve
// until the microtask checkpoint. This is because .then()
// on an already-resolved promise enqueues the reaction as
// a microtask rather than calling it synchronously.

p.then(values => {
  console.log('resolved:', values);
  // This fires during the microtask checkpoint after
  // the current synchronous execution completes.
});
```

### 9.2 Microtask Checkpoint Behavior

```javascript
// How many microtask checkpoints does this cross?

async function example() {
  // Checkpoint 1: N microtasks (resolve elements)
  // Checkpoint 2: 1 microtask (outer promise resolves)
  // Checkpoint 3: 1 microtask (await continuation)
  const results = await Promise.all([
    Promise.resolve('a'),
    Promise.resolve('b'),
    Promise.resolve('c'),
  ]);
  // Total: N + 2 microtasks between call and continuation

  console.log(results);
}
```

But "microtask checkpoint" is slightly misleading. The microtask
queue is drained completely at each checkpoint. So all N + 2
microtasks fire within a single checkpoint, not across multiple:

```
Call stack: example() → Promise.all() → returns
            ↓
Microtask checkpoint begins:
  [resolveElement0] → fires, enqueues nothing
  [resolveElement1] → fires, enqueues nothing
  [resolveElement2] → fires, counter=0, resolves outer
    → enqueues outer.then reaction (the await continuation)
  [await continuation] → fires, resumes example()
    → console.log(results)
Microtask checkpoint ends.
```

All of this happens in ONE microtask checkpoint. The await
continuation fires in the same drain cycle because it was enqueued
during the drain.

### 9.3 Interaction with setTimeout

```javascript
Promise.all([
  new Promise(resolve => setTimeout(() => resolve('a'), 0)),
  new Promise(resolve => setTimeout(() => resolve('b'), 0)),
]).then(values => console.log(values));

console.log('sync');

// Output:
// sync
// (microtask checkpoint — nothing from Promise.all yet)
// (next event loop iteration — setTimeout callbacks fire)
// (resolve 'a' → microtask: resolveElement0)
// (resolve 'b' → microtask: resolveElement1)
// (microtask checkpoint: resolveElement0, resolveElement1 fire)
// (resolveElement1 triggers outer resolve → microtask)
// (outer .then fires)
// ['a', 'b']
```

Each setTimeout callback is a separate macrotask. But browsers
may batch setTimeout(fn, 0) callbacks in the same macrotask.
The exact behavior depends on the engine implementation.

In practice, the two setTimeout(fn, 0) callbacks typically fire
in the same macrotask (since they have the same delay), so both
resolve elements fire in the same microtask checkpoint.

### 9.4 Starvation Scenarios

```javascript
// Can Promise.all cause microtask starvation?
// Yes, if the input contains enough already-resolved promises:

const millions = Array.from({ length: 10_000_000 },
  () => Promise.resolve(1));

Promise.all(millions).then(values => {
  console.log('done', values.length);
});

// During the microtask checkpoint:
// 10 million resolve element functions fire sequentially.
// The main thread is blocked for the entire duration.
// No rendering, no user input, no macrotasks.
// This could take several seconds.
```

This is technically microtask starvation of macrotasks. The microtask
checkpoint won't complete until all 10 million resolve elements fire.

### 9.5 Promise.all in Node.js Phases

```javascript
const fs = require('fs');

fs.readFile('file.txt', () => {
  // We're in the poll phase callback

  Promise.all([
    Promise.resolve('a'),
    Promise.resolve('b'),
  ]).then(values => {
    console.log('Promise.all resolved:', values);
    // This fires during the microtask checkpoint
    // BETWEEN this I/O callback and the next one
    // (or between poll phase and check phase).
  });

  setImmediate(() => {
    console.log('setImmediate');
    // This fires in the check phase, AFTER the microtask
    // checkpoint where Promise.all resolves.
  });
});
```

---

## 10. Common Bugs and Antipatterns

### 10.1 Fail-Fast Surprise

The most common source of confusion:

```javascript
// Developer expects all results, gets an error instead:
try {
  const [users, posts, comments] = await Promise.all([
    fetchUsers(),     // succeeds
    fetchPosts(),     // FAILS
    fetchComments(),  // succeeds
  ]);
} catch (err) {
  // Only get the error from fetchPosts.
  // No access to the successful results from fetchUsers/fetchComments.
  // Those results are lost forever.
}
```

Fix: use `Promise.allSettled` when you need partial results:

```javascript
const results = await Promise.allSettled([
  fetchUsers(),
  fetchPosts(),
  fetchComments(),
]);

const users = results[0].status === 'fulfilled' ? results[0].value : [];
const posts = results[1].status === 'fulfilled' ? results[1].value : [];
const comments = results[2].status === 'fulfilled' ? results[2].value : [];
```

### 10.2 Unhandled Rejection from Remaining Promises

```javascript
// BUG: the second rejection is unhandled!
async function broken() {
  try {
    await Promise.all([
      Promise.reject('error1'),
      Promise.reject('error2'),  // This rejection is NOT caught!
    ]);
  } catch (err) {
    console.log(err);  // 'error1'
    // 'error2' triggers an UnhandledPromiseRejection warning/error
    // in some environments.
  }
}
```

Wait — this is actually NOT true in practice. Let me be precise.

Promise.all attaches a `.then(resolveElement, reject)` to each input
promise. The `reject` is the outer promise's reject function. When
the second promise rejects, it calls `reject('error2')`, but the
outer promise is already rejected (by 'error1'), so the call is a
no-op. The rejection IS handled because `.then()` was called on the
promise, giving it a rejection handler.

The actual unhandled rejection bug occurs when you create promises
that NEVER get `.then()` called on them:

```javascript
// ACTUAL unhandled rejection bug:
const p1 = fetch('/api/1');
const p2 = fetch('/api/2');

// If this throws before setting up Promise.all:
doSomethingSynchronous();  // throws!

// p1 and p2 are now dangling with no rejection handlers.
// If either fetch fails, you get UnhandledPromiseRejection.

// FIX: create promises inside Promise.all:
const [r1, r2] = await Promise.all([
  fetch('/api/1'),
  fetch('/api/2'),
]);
```

### 10.3 Sequential Disguised as Parallel

```javascript
// BUG: this is sequential, not parallel!
async function notParallel() {
  const a = await fetch('/api/1');  // wait for this...
  const b = await fetch('/api/2');  // THEN start this
  return [a, b];
}

// FIX: start all promises before awaiting
async function parallel() {
  const [a, b] = await Promise.all([
    fetch('/api/1'),
    fetch('/api/2'),
  ]);
  return [a, b];
}

// ALSO PARALLEL (but less idiomatic):
async function alsoParallel() {
  const pA = fetch('/api/1');  // start immediately
  const pB = fetch('/api/2');  // start immediately
  const a = await pA;          // now wait
  const b = await pB;          // already done or almost done
  return [a, b];
}
```

### 10.4 Promise.all Inside a Loop

```javascript
// BUG: sequential batches (each batch waits for the previous)
for (const batch of batches) {
  const results = await Promise.all(batch.map(item => process(item)));
  // Next iteration waits for this batch to finish.
}

// If you want ALL batches in parallel:
const allResults = await Promise.all(
  batches.map(batch =>
    Promise.all(batch.map(item => process(item)))
  )
);
// This is Promise.all of Promise.all — nested parallelism.
```

### 10.5 Ignoring the Return Value

```javascript
// BUG: fire-and-forget with no error handling
function doStuff() {
  Promise.all([
    saveToDatabase(data),
    sendNotification(user),
    updateCache(key),
  ]);
  // No await, no .catch() — if any fails, UnhandledPromiseRejection
}

// FIX:
async function doStuff() {
  await Promise.all([
    saveToDatabase(data),
    sendNotification(user),
    updateCache(key),
  ]);
}
// or at minimum:
function doStuff() {
  Promise.all([...]).catch(err => {
    logger.error('Background task failed:', err);
  });
}
```

### 10.6 Mutating Shared State

```javascript
// BUG: race condition with shared mutable state
let counter = 0;

await Promise.all(items.map(async item => {
  const current = counter;
  await processAsync(item);
  counter = current + 1;  // Race! Multiple callbacks read the same 'current'
}));

console.log(counter);  // Likely 1, not items.length

// FIX: use atomic operations or avoid shared state
const results = await Promise.all(items.map(item => processAsync(item)));
const counter = results.length;
```

### 10.7 Promise.all with Side-Effect Ordering

```javascript
// BUG: assuming side effects happen in order
await Promise.all([
  updateStep1(),  // might finish last
  updateStep2(),  // might finish first
  updateStep3(),  // might finish second
]);
// If these have ordering dependencies, use sequential await instead.

// The RESULTS are ordered, but the EXECUTION is not.
```

### 10.8 Memory Bomb

```javascript
// BUG: loading everything into memory at once
const results = await Promise.all(
  millionUrls.map(url => fetch(url).then(r => r.arrayBuffer()))
);
// OOM! All array buffers in memory simultaneously.

// FIX: use streaming or batching (see Section 8)
```

### 10.9 Nested Promise.all with Error Propagation

```javascript
// Subtle: inner Promise.all rejection propagates to outer
try {
  await Promise.all([
    Promise.all([
      fetch('/a'),
      fetch('/b'),  // If this fails...
    ]),
    Promise.all([
      fetch('/c'),
      fetch('/d'),
    ]),
  ]);
} catch (err) {
  // err is from fetch('/b')
  // The inner Promise.all rejected, which rejected the outer Promise.all
  // fetch('/c') and fetch('/d') might still be in-flight!
  // Their results are discarded. Their side effects still happen.
}
```

### 10.10 Type Narrowing Pitfall

```javascript
// BUG: TypeScript thinks all results are the same type
const results = await Promise.all([
  fetchNumber(),   // Promise<number>
  fetchString(),   // Promise<string>
  fetchBoolean(),  // Promise<boolean>
]);
// results: [number, string, boolean] — TypeScript handles this correctly
// with its tuple overloads (see Section 11).

// BUT: if you use .map(), you lose the tuple type:
const fns = [fetchNumber, fetchString, fetchBoolean];
const results = await Promise.all(fns.map(fn => fn()));
// results: (number | string | boolean)[] — lost the per-index type!
```

---

## 11. TypeScript Typing

### 11.1 Promise.all Overloads

TypeScript's `lib.es2015.promise.d.ts` defines Promise.all with
multiple overloads to handle heterogeneous arrays as tuples:

```typescript
interface PromiseConstructor {
  all<T extends readonly unknown[] | []>(
    values: T
  ): Promise<{ -readonly [P in keyof T]: Awaited<T[P]> }>;
}
```

This single mapped-type signature (in newer TS versions) replaces
what used to be many overloads. In older TypeScript (pre-4.5), there
were explicit overloads for up to 10 elements:

```typescript
// Old-style overloads (pre TypeScript 4.5):
all<T1, T2>(values: readonly [T1 | PromiseLike<T1>, T2 | PromiseLike<T2>]):
  Promise<[T1, T2]>;
all<T1, T2, T3>(values: readonly [
  T1 | PromiseLike<T1>,
  T2 | PromiseLike<T2>,
  T3 | PromiseLike<T3>
]): Promise<[T1, T2, T3]>;
// ... up to T10
```

### 11.2 The Awaited<T> Utility Type

`Awaited<T>` was introduced in TypeScript 4.5 to recursively unwrap
Promise-like types:

```typescript
// Definition (simplified):
type Awaited<T> =
  T extends null | undefined ? T :
  T extends object & { then(onfulfilled: infer F, ...args: infer _): any } ?
    F extends ((value: infer V, ...args: infer _) => any) ?
      Awaited<V> :   // Recursive unwrap
      never :
  T;                 // Non-thenable: return as-is

// Examples:
type A = Awaited<Promise<string>>;          // string
type B = Awaited<Promise<Promise<number>>>; // number (recursive!)
type C = Awaited<string>;                   // string (non-promise passthrough)
type D = Awaited<boolean | Promise<number>>; // boolean | number
```

### 11.3 Heterogeneous Arrays

```typescript
// TypeScript correctly infers tuple types:
const result = await Promise.all([
  fetch('/users'),        // Promise<Response>
  Promise.resolve(42),    // Promise<number>
  'hello',                // string (non-promise)
]);
// typeof result: [Response, number, string]

// Each element gets its own type, preserving index position.
```

The key insight: TypeScript uses the `readonly [T1, T2, T3]` tuple
type for the input, and the mapped type `{ [P in keyof T]: Awaited<T[P]> }`
to transform each element individually.

### 11.4 Array vs Tuple Inference

```typescript
// TUPLE — TypeScript infers per-element types:
const tuple = await Promise.all([
  fetchUser(1),          // Promise<User>
  fetchPosts(1),         // Promise<Post[]>
]);
// typeof tuple: [User, Post[]]

// ARRAY — TypeScript infers union type:
const promises = [fetchUser(1), fetchPosts(1)];
// typeof promises: (Promise<User> | Promise<Post[]>)[]
const array = await Promise.all(promises);
// typeof array: (User | Post[])[]  — lost the tuple type!

// FIX: use 'as const' or explicit tuple type:
const promises = [fetchUser(1), fetchPosts(1)] as const;
const result = await Promise.all(promises);
// typeof result: [User, Post[]]
```

### 11.5 Typing Promise.allSettled

```typescript
interface PromiseFulfilledResult<T> {
  status: 'fulfilled';
  value: T;
}

interface PromiseRejectedResult {
  status: 'rejected';
  reason: any;
}

type PromiseSettledResult<T> =
  | PromiseFulfilledResult<T>
  | PromiseRejectedResult;

// Promise.allSettled returns:
// Promise<PromiseSettledResult<T>[]>

// Usage with type guards:
const results = await Promise.allSettled([
  fetchUser(1),
  fetchPosts(1),
]);

for (const result of results) {
  if (result.status === 'fulfilled') {
    // result.value is typed correctly
    console.log(result.value);
  } else {
    // result.reason is 'any' — TypeScript can't type rejection reasons
    console.error(result.reason);
  }
}
```

### 11.6 Generic Wrapper Functions

```typescript
// Wrapping Promise.all while preserving types is tricky:

// BAD: loses tuple type
function myAll<T>(promises: Promise<T>[]): Promise<T[]> {
  return Promise.all(promises);
}

// GOOD: preserves tuple type
function myAll<T extends readonly unknown[]>(
  promises: [...T]
): Promise<{ -readonly [P in keyof T]: Awaited<T[P]> }> {
  return Promise.all(promises) as any;
}

// The [...T] spread is crucial — it forces TypeScript to infer T
// as a tuple type rather than an array type.
```

### 11.7 Promise.all with Map/Filter

```typescript
// Type-safe parallel mapping:
async function parallelMap<T, U>(
  items: T[],
  fn: (item: T) => Promise<U>
): Promise<U[]> {
  return Promise.all(items.map(fn));
}

// Type-safe parallel filter:
async function parallelFilter<T>(
  items: T[],
  predicate: (item: T) => Promise<boolean>
): Promise<T[]> {
  const results = await Promise.all(
    items.map(async item => ({
      item,
      keep: await predicate(item),
    }))
  );
  return results.filter(r => r.keep).map(r => r.item);
}
```

---

## 12. V8 Engine Implementation Details

### 12.1 V8's Promise Internal Structure

In V8's C++ source (src/objects/js-promise.h), a JSPromise has:

```cpp
// V8 internal: JSPromise
class JSPromise : public JSObjectWithEmbedderSlots {
 public:
  // The result (value if fulfilled, reason if rejected)
  // OR: the list of PromiseReaction objects if pending
  DECL_ACCESSORS(reactions_or_result, Tagged<Object>)

  // Status flags packed into a single Smi
  DECL_INT_ACCESSORS(flags)

  // Status: kPending (0), kFulfilled (1), kRejected (2)
  inline Promise::PromiseState status() const;

  // Whether the promise has a handler (.then was called)
  inline bool has_handler() const;

  // Whether the promise was marked as handled
  inline bool handled_hint() const;
};
```

When a promise is **pending**, `reactions_or_result` holds a linked
list of `PromiseReaction` objects. When the promise **settles**,
this field is overwritten with the result value, and the reactions
are moved to the microtask queue.

### 12.2 PromiseReaction Records in V8

```cpp
// V8 internal: PromiseReaction
// Corresponds to the spec's PromiseReaction Record
class PromiseReaction : public Struct {
 public:
  DECL_ACCESSORS(next, Tagged<Object>)           // Linked list
  DECL_ACCESSORS(reject_handler, Tagged<Object>)  // Callable or Undefined
  DECL_ACCESSORS(fulfill_handler, Tagged<Object>) // Callable or Undefined
  DECL_ACCESSORS(promise_or_capability, Tagged<Object>)
};
```

The `next` field forms a singly-linked list. When you call `.then()`
on a pending promise, a new PromiseReaction is prepended to the list.
When the promise settles, the list is reversed (to maintain FIFO order)
and each reaction is enqueued as a microtask.

### 12.3 Resolving Functions — CreateResolvingFunctions

The spec's CreateResolvingFunctions creates a resolve and reject
function pair, each with an `[[AlreadyResolved]]` flag that is
shared between them:

```
CreateResolvingFunctions ( promise )

1. Let alreadyResolved be the Record { [[Value]]: false }.
2. Let resolve be a new built-in function (Promise Resolve Function).
   resolve.[[Promise]] = promise
   resolve.[[AlreadyResolved]] = alreadyResolved
3. Let reject be a new built-in function (Promise Reject Function).
   reject.[[Promise]] = promise
   reject.[[AlreadyResolved]] = alreadyResolved
4. Return the Record { [[Resolve]]: resolve, [[Reject]]: reject }.
```

The shared `alreadyResolved` record ensures that once either resolve
or reject is called, the other becomes a no-op. This is different from
Promise.all's per-element `[[AlreadyCalled]]` flag, which only guards
individual resolve element functions.

### 12.4 Promise Resolution Procedure

When `resolve(x)` is called (the Promise Resolve Function):

```
Promise Resolve Function ( resolution )

1. Let F be the active function object.
2. Let promise be F.[[Promise]].
3. Let alreadyResolved be F.[[AlreadyResolved]].
4. If alreadyResolved.[[Value]] is true, return undefined.
5. Set alreadyResolved.[[Value]] to true.
6. If SameValue(resolution, promise), then
   a. Let selfResolutionError be a new TypeError.
   b. Perform RejectPromise(promise, selfResolutionError).
   c. Return undefined.
   // A promise cannot resolve to itself!
7. If resolution is not an Object, then
   a. Perform FulfillPromise(promise, resolution).
   b. Return undefined.
8. Let then be Get(resolution, "then").
9. If then is an abrupt completion, then
   a. Perform RejectPromise(promise, then.[[Value]]).
   b. Return undefined.
10. Let thenAction be then.[[Value]].
11. If IsCallable(thenAction) is false, then
    a. Perform FulfillPromise(promise, resolution).
    b. Return undefined.
12. Let thenJobCallback be HostMakeJobCallback(thenAction).
13. Let job be NewPromiseResolveThenableJob(promise, resolution,
                                             thenJobCallback).
14. Perform HostEnqueuePromiseJob(job.[[Job]], job.[[Realm]]).
15. Return undefined.
```

Key points:
- Step 6: resolving a promise with itself is a TypeError.
- Steps 7-11: non-objects and objects without .then are direct fulfillment.
- Step 12-14: thenables get a PromiseResolveThenableJob, which is a
  microtask that calls `thenable.then(resolve, reject)`.

### 12.5 FulfillPromise and RejectPromise

```
FulfillPromise ( promise, value )

1. Assert: promise.[[PromiseState]] is pending.
2. Let reactions be promise.[[PromiseFulfillReactions]].
3. Set promise.[[PromiseResult]] to value.
4. Set promise.[[PromiseFulfillReactions]] to undefined.
5. Set promise.[[PromiseRejectReactions]] to undefined.
6. Set promise.[[PromiseState]] to fulfilled.
7. Perform TriggerPromiseReactions(reactions, value).
```

In V8's implementation, steps 2-6 translate to:
- Read the reactions_or_result field (which is the reaction list).
- Overwrite reactions_or_result with the value.
- Update the status bits in the flags field.
- Call TriggerPromiseReactions.

### 12.6 TriggerPromiseReactions

```
TriggerPromiseReactions ( reactions, argument )

1. For each element reaction of reactions, do
   a. Let job be NewPromiseReactionJob(reaction, argument).
   b. Perform HostEnqueuePromiseJob(job.[[Job]], job.[[Realm]]).
2. Return undefined.
```

Each reaction becomes a **PromiseReactionJob** on the microtask queue.
The job, when executed, calls the appropriate handler (fulfill or reject)
and resolves/rejects the downstream promise.

### 12.7 PromiseReactionJob

```
NewPromiseReactionJob ( reaction, argument )

Returns a job that:
1. Let handler be reaction.[[Handler]].
2. If handler is empty (no handler — bare .then()):
   a. If reaction.[[Type]] is Fulfill, let handlerResult be
      NormalCompletion(argument).
   b. If reaction.[[Type]] is Reject, let handlerResult be
      ThrowCompletion(argument).
3. Else, let handlerResult be Completion(HostCallJobCallback(handler,
   undefined, << argument >>)).
   // Call the actual handler function with the value/reason.
4. If reaction.[[PromiseOrCapability]] is a PromiseCapability Record:
   a. If handlerResult is an abrupt completion:
      Call(capability.[[Reject]], undefined, << handlerResult.[[Value]] >>).
   b. Else:
      Call(capability.[[Resolve]], undefined, << handlerResult.[[Value]] >>).
```

This is how promise chaining works at the engine level. Each `.then()`
creates a new promise + capability. The reaction job calls the handler,
then resolves or rejects the downstream promise based on the result.

### 12.8 V8's Torque/CSA Implementation of Promise.all

V8 implements Promise.all in Torque (V8's internal language that
compiles to CodeStubAssembler). The key files in V8's source:

```
src/builtins/promise-all.tq
src/builtins/promise-all-element-closure.tq
src/builtins/promise-abstract-operations.tq
src/builtins/promise-resolve.tq
```

The Torque implementation follows the spec closely but with engine-
specific optimizations:

**Fast path check**: Before entering the generic algorithm, V8 checks:
1. Is the receiver the native Promise constructor (not a subclass)?
2. Is `Promise.resolve` the original unmodified function?
3. Is the iterable a JSArray with the original Array iterator?
4. Is `Promise.prototype.then` the original unmodified function?

If ALL checks pass, V8 uses a fast path that skips the generic
iterator protocol, GetPromiseResolve, and some of the capability
checks. This is significantly faster for the common case.

**Resolve element function optimization**: V8 doesn't create a
full JSFunction for each resolve element. Instead, it uses a
special internal closure type (PromiseAllResolveElementClosure)
that stores the index, values array, and counter in internal
slots rather than as captured JavaScript variables. This reduces
per-element allocation overhead.

### 12.9 The PromiseAllResolveElementClosure

In V8, the resolve element functions are not regular JavaScript
function objects. They are instances of a special internal type:

```cpp
// V8 creates a specialized closure for each element:
// Stored fields (internal slots, not JS properties):
//   - values_array: FixedArray
//   - index: Smi
//   - remaining_elements: Smi (in a Cell for mutability)
//   - capability: PromiseCapability
//   - already_called: Boolean (in a Cell for mutability)
```

This is more memory-efficient than a regular closure because:
1. No captured scope chain to walk.
2. Internal slots are fixed-layout (no dictionary overhead).
3. All resolve element functions share the same Map (hidden class).

### 12.10 The Microtask Queue in V8

V8's microtask queue (`src/execution/microtask-queue.cc`):

```cpp
class MicrotaskQueue final {
 public:
  // Ring buffer implementation
  std::unique_ptr<Address[]> ring_buffer_;
  int capacity_;
  int size_;
  int start_;

  // Enqueue a microtask
  void EnqueueMicrotask(Tagged<Microtask> microtask);

  // Run all microtasks until queue is empty
  int RunMicrotasks(Isolate* isolate);
};
```

The microtask queue is a simple ring buffer (not a linked list).
`RunMicrotasks` loops until the queue is empty, handling the case
where microtasks enqueue more microtasks:

```cpp
int MicrotaskQueue::RunMicrotasks(Isolate* isolate) {
  int processed = 0;
  while (size_ > 0) {
    Tagged<Microtask> task = ring_buffer_[start_];
    start_ = (start_ + 1) % capacity_;
    size_--;

    // Execute the microtask
    RunSingleMicrotask(isolate, task);
    processed++;
  }
  return processed;
}
```

For Promise.all with N already-resolved promises, this means N
microtasks are enqueued during the synchronous `Promise.all()` call,
and then all N are drained in the next microtask checkpoint. The
ring buffer may need to resize if N is large.

### 12.11 Promise Hooks and Async Stack Traces

V8 supports promise lifecycle hooks for debugging:

```cpp
// V8 promise hooks (used by Node.js async_hooks):
enum class PromiseHookType {
  kInit,      // Promise created
  kResolve,   // Promise resolved
  kBefore,    // Before .then handler runs
  kAfter      // After .then handler runs
};
```

When async stack traces are enabled (Chrome DevTools, Node.js
`--async-stack-traces`), V8 captures a stack trace when each
promise is created and attaches it to the promise. This adds
overhead to every Promise.all element.

With `--async-stack-traces` (default in Node 12+):
- Each promise stores a reference to its "parent" promise.
- When an error occurs, V8 walks the promise chain to reconstruct
  the async call stack.
- This means Promise.all's resolve element functions have traceable
  async provenance.

### 12.12 Promise.all and V8's Optimization Tiers

Promise.all's behavior across V8's compilation tiers:

**Ignition (interpreter)**:
The built-in functions (Promise.all, resolve element functions) are
NOT interpreted. They are C++ builtins or Torque-generated code.
The user's `.then()` callback IS interpreted by Ignition on first
call.

**TurboFan (optimizing compiler)**:
V8 can inline simple `.then()` callbacks when they are monomorphic.
For Promise.all, the resolve element functions are internal builtins
and are not subject to TurboFan optimization. However, the user's
callback passed to `.then()` on the Promise.all result CAN be
optimized by TurboFan.

**Practical impact**: the Promise.all machinery itself is already
optimized native code. The performance-sensitive part is the user's
handlers, not the Promise.all infrastructure.

---

## Summary: The Complete Promise.all Mental Model

```
Promise.all(iterable)
  │
  ├─ Creates outer PromiseCapability (promise + resolve + reject)
  │
  ├─ Gets constructor.resolve (for wrapping values)
  │
  ├─ Initializes: values = [], counter = 1, index = 0
  │
  ├─ Synchronously iterates the iterable:
  │   │
  │   ├─ For each element:
  │   │   ├─ Append undefined to values (pre-allocate slot)
  │   │   ├─ Wrap with Promise.resolve(element) → nextPromise
  │   │   ├─ Create resolveElementN function (captures index, values,
  │   │   │   counter, capability, alreadyCalled flag)
  │   │   ├─ counter++
  │   │   └─ nextPromise.then(resolveElementN, outerReject)
  │   │       ├─ If nextPromise already settled:
  │   │       │   → enqueue reaction as microtask immediately
  │   │       └─ If nextPromise pending:
  │   │           → attach reaction, fires when promise settles
  │   │
  │   └─ After iteration: counter--
  │       └─ If counter === 0 → resolve outer with values array
  │          (only happens if iterable was empty)
  │
  ├─ Returns outer promise (still pending unless empty iterable)
  │
  └─ Later, as each input promise resolves:
      │
      ├─ resolveElementN fires (microtask):
      │   ├─ Check alreadyCalled → bail if true
      │   ├─ Set values[N] = resolved value
      │   ├─ counter--
      │   └─ If counter === 0 → resolve outer with values array
      │
      └─ If any input promise rejects:
          └─ outerReject fires (microtask):
              └─ Outer promise rejects (first rejection wins)
                 (subsequent rejections are no-ops)


Microtask count (N already-resolved inputs):
  - N resolve element microtasks
  - 1 outer promise resolve microtask
  - 1 .then()/.catch()/await continuation microtask
  = N + 2 total microtasks

Memory footprint:
  - 1 outer promise (~40 bytes)
  - 1 values array (N * 8 bytes)
  - N resolve element closures (~100 bytes each)
  - N PromiseReaction records (~24 bytes each)
  - Shared counter + shared reject function
  ≈ 132*N + 50 bytes overhead

Key invariants:
  1. Output order ALWAYS matches input order
  2. First rejection wins (fail-fast)
  3. No concurrency control (all started immediately)
  4. Works with any iterable, not just arrays
  5. Empty iterable resolves immediately with []
  6. Non-promise values are wrapped via Promise.resolve()
  7. All resolve element functions still fire even after rejection
  8. The outer promise is a brand new promise (not one of the inputs)
```

---

## References

- ECMA-262, 15th Edition (ES2024), Sections 27.2.4.1 through 27.2.4.7
- ECMA-262, Section 27.2.1 (Promise Abstract Operations)
- ECMA-262, Section 27.2.2 (Promise Jobs)
- V8 source: `src/builtins/promise-all.tq`
- V8 source: `src/objects/js-promise.h`, `src/objects/js-promise-inl.h`
- V8 source: `src/execution/microtask-queue.cc`
- V8 blog: "Fast async functions and promises" (2018)
- V8 blog: "Understanding the ECMAScript spec" (2020)
- HTML specification, Section 8.1.7 (Event loops)
- Node.js documentation: "The Node.js Event Loop, Timers, and process.nextTick()"
- TypeScript `lib.es2015.promise.d.ts` and `lib.es2018.promise.d.ts`
- TypeScript 4.5 release notes (Awaited<T> type)
