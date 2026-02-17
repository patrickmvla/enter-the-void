# The Event Loop — How JavaScript Schedules Work

## The Fundamental Constraint

JavaScript is **single-threaded**. One call stack, one thread of
execution. Yet it handles thousands of concurrent I/O operations,
animations, user input, and timers. The event loop is the mechanism
that makes this possible.

The event loop is not part of JavaScript itself — it's provided by
the **host environment** (browser or Node.js). The JavaScript spec
defines the language; the HTML spec and libuv define the event loops.

---

## Browser Event Loop

The HTML specification defines the event loop. It's a loop that runs
continuously, processing tasks from multiple queues.

### Core Components

```
┌──────────────────────────────────────────────────────┐
│                   Browser Process                     │
│                                                       │
│  ┌─────────────────────────────────────────────────┐  │
│  │              Main Thread                         │  │
│  │                                                  │  │
│  │  ┌──────────┐    ┌──────────────┐               │  │
│  │  │Call Stack │    │Microtask     │               │  │
│  │  │          │    │Queue         │               │  │
│  │  │ fn_c()   │    │ Promise.then │               │  │
│  │  │ fn_b()   │    │ queueMicro.. │               │  │
│  │  │ fn_a()   │    │ MutObserver  │               │  │
│  │  └──────────┘    └──────────────┘               │  │
│  │                                                  │  │
│  │  ┌──────────────┐  ┌──────────────┐             │  │
│  │  │Task Queue    │  │rAF Callbacks │             │  │
│  │  │(macrotask)   │  │              │             │  │
│  │  │ setTimeout   │  │ animation cb │             │  │
│  │  │ setInterval  │  │ animation cb │             │  │
│  │  │ MessagePort  │  │              │             │  │
│  │  │ I/O callback │  └──────────────┘             │  │
│  │  │ UI events    │                               │  │
│  │  └──────────────┘                               │  │
│  │                                                  │  │
│  └─────────────────────────────────────────────────┘  │
│                                                       │
└──────────────────────────────────────────────────────┘
```

### The Loop Algorithm

Per the HTML spec, one iteration of the event loop:

```
1. Pick the oldest task from a task queue that has a runnable task.
   (The browser chooses WHICH queue — not necessarily FIFO across
   all queues. This allows prioritization.)

2. Execute that task to completion.
   - This means: run until the call stack is empty.
   - During execution, any microtasks queued are NOT run yet.

3. Perform a microtask checkpoint:
   a. While the microtask queue is not empty:
      - Dequeue the oldest microtask
      - Execute it to completion
      - If that microtask queues more microtasks, they run too
        (the queue is drained completely)

4. If this is a rendering opportunity (typically ~60Hz / 16.67ms):
   a. Run requestAnimationFrame callbacks (all of them, in order)
   b. Run the rendering pipeline:
      - Style recalculation
      - Layout
      - Paint
      - Composite

5. If idle time remains before the next frame:
   a. Run requestIdleCallback callbacks

6. Go to step 1.
```

### Task Queues (Macrotasks)

Each event loop has **multiple** task queues. The spec doesn't mandate
a single queue — it mandates that the browser can prioritize:

```
Task Sources → Task Queues:
  - User interaction events (click, keydown)     → High priority queue
  - Timer callbacks (setTimeout, setInterval)     → Normal priority queue
  - MessageChannel / postMessage                  → Normal priority queue
  - I/O completion (fetch response, IndexedDB)    → Normal priority queue
  - DOM manipulation events                       → Normal priority queue
```

**Critical**: only ONE task runs per loop iteration before the
microtask checkpoint. This is why a long task blocks everything.

```javascript
// This blocks the event loop for ~1 second:
button.addEventListener('click', () => {
  const start = Date.now();
  while (Date.now() - start < 1000) {} // Busy wait
  // No other tasks, microtasks, or rendering can happen during this
});
```

### Microtask Queue

Microtasks are processed AFTER each task, and the queue is drained
completely — including microtasks added during microtask processing.

Microtask sources:
- `Promise.then()`, `Promise.catch()`, `Promise.finally()`
- `queueMicrotask()`
- `MutationObserver` callbacks
- `async`/`await` continuations (after each `await`)

```javascript
console.log('1: script start');

setTimeout(() => console.log('2: setTimeout'), 0);

Promise.resolve().then(() => {
  console.log('3: promise 1');
  Promise.resolve().then(() => console.log('4: nested promise'));
});

queueMicrotask(() => console.log('5: microtask'));

console.log('6: script end');

// Output:
// 1: script start
// 6: script end
// 3: promise 1      (microtask)
// 5: microtask       (microtask)
// 4: nested promise  (microtask queued by microtask — runs before setTimeout!)
// 2: setTimeout      (next task)
```

**Microtask starvation**: since microtasks drain completely, you can
starve the event loop:

```javascript
function infinite() {
  Promise.resolve().then(infinite);  // Infinitely queues microtasks
}
infinite();
// The page is now PERMANENTLY frozen.
// No tasks, no rendering, no user input will ever run.
// This is worse than an infinite while loop (at least that can be killed).
```

### requestAnimationFrame (rAF)

rAF callbacks run once per frame, **before** the rendering steps.
They are NOT microtasks or macrotasks — they have their own queue
that's processed at a specific point in the loop.

```javascript
// rAF runs before paint, after microtasks
requestAnimationFrame(() => {
  element.style.transform = 'translateX(100px)';
  // This change will be painted in this frame
});
```

Key behaviors:

1. **rAF callbacks run all at once** — if 10 rAF callbacks are queued,
   all 10 run before paint. New rAF callbacks registered during
   execution are deferred to the **next** frame.

2. **rAF doesn't run if the tab is hidden** — the browser throttles
   to save resources. `setTimeout` continues (throttled to 1Hz).

3. **rAF timing is aligned to the display refresh rate** — typically
   60Hz (16.67ms) but can be 120Hz, 144Hz, or variable refresh rate.

```javascript
// Animation loop pattern
function animate(timestamp) {
  // timestamp = performance.now() at the start of the frame
  const elapsed = timestamp - lastTimestamp;

  // Update state based on elapsed time, not frame count
  position += velocity * (elapsed / 1000);

  element.style.transform = `translateX(${position}px)`;

  lastTimestamp = timestamp;
  requestAnimationFrame(animate);
}
requestAnimationFrame(animate);
```

### requestIdleCallback

Runs when the browser has idle time in a frame — after rendering,
before the next frame starts:

```
│ Task │ Microtasks │ rAF │ Style │ Layout │ Paint │ Idle │ ... │
0ms                                                        16.67ms
```

```javascript
requestIdleCallback((deadline) => {
  // deadline.timeRemaining() tells you how much idle time is left
  while (deadline.timeRemaining() > 0 && workQueue.length > 0) {
    processItem(workQueue.shift());
  }
  if (workQueue.length > 0) {
    requestIdleCallback(processWork);  // Continue next idle period
  }
}, { timeout: 2000 });  // Force execution after 2 seconds if no idle time
```

**`timeRemaining()`** returns milliseconds until the next frame deadline
(capped at 50ms). This is an estimate — the browser might need the time
back if a high-priority event arrives.

---

## Rendering and the Event Loop

### When Does Rendering Happen?

The browser does NOT render after every task. Rendering is opportunistic:

- Target: once per display refresh (60Hz = every 16.67ms)
- Skip if: nothing visual changed (no style/layout invalidation)
- Skip if: tab is hidden/occluded
- Throttle if: system is under load

```javascript
// This does NOT cause 100 repaints:
for (let i = 0; i < 100; i++) {
  element.style.left = i + 'px';
}
// Only the final value (99px) is painted.
// The rendering step happens AFTER the task completes.
// All intermediate values are overwritten before paint.
```

### Forced Synchronous Layout

Certain JavaScript operations force the browser to calculate layout
synchronously (mid-task), because the return value depends on layout:

```javascript
// Normal flow: style → layout → paint (batched, async)

// Forced sync layout:
element.style.width = '100px';    // Invalidates layout
let height = element.offsetHeight; // Forces layout NOW to read geometry
element.style.width = '200px';    // Invalidates layout again
let height2 = element.offsetHeight; // Forces layout AGAIN

// Properties that force layout:
// offsetTop/Left/Width/Height
// clientTop/Left/Width/Height
// scrollTop/Left/Width/Height
// getComputedStyle()
// getBoundingClientRect()
```

### Layout Thrashing

Reading and writing layout properties alternately forces multiple
synchronous layouts per frame:

```javascript
// TERRIBLE: N forced layouts
const elements = document.querySelectorAll('.item');
for (const el of elements) {
  const width = el.offsetWidth;     // READ → forces layout
  el.style.width = width * 2 + 'px'; // WRITE → invalidates layout
  // Next iteration's READ forces layout again
}

// GOOD: batch reads, then batch writes
const widths = [];
for (const el of elements) {
  widths.push(el.offsetWidth);  // All READs together → one layout
}
for (let i = 0; i < elements.length; i++) {
  elements[i].style.width = widths[i] * 2 + 'px';  // All WRITEs together
}
// → Only one layout needed
```

---

## setTimeout and setInterval — The Details

### setTimeout(fn, 0) Is Not Actually 0ms

```javascript
setTimeout(() => console.log('fired'), 0);
```

Minimum delay: **4ms** after 5 nested setTimeout calls (HTML spec).
Browser implementation varies — Chrome clamps to 1ms for non-nested
calls but 4ms after nesting depth 5.

```javascript
// Demonstrating the 4ms clamp:
function nestTimeout(depth) {
  const start = performance.now();
  setTimeout(() => {
    console.log(`Depth ${depth}: ${performance.now() - start}ms`);
    if (depth < 10) nestTimeout(depth + 1);
  }, 0);
}
nestTimeout(0);
// Depth 0: ~1ms
// Depth 1: ~1ms
// ...
// Depth 5: ~4ms  (clamped!)
// Depth 6: ~4ms
```

**Background tabs**: timers are throttled to 1000ms minimum (1Hz)
to save battery.

### setInterval Drift

`setInterval` doesn't guarantee exact intervals — if the callback
takes longer than the interval, calls queue up:

```
setInterval(heavyWork, 100)

Time: 0    100   200   300   400   500
      |--heavyWork(120ms)--|
                           |--heavyWork(120ms)--|
                                                |--heavyWork...
      ↑                    ↑
    start               fires immediately (was queued at 100ms)
```

The browser prevents interval callbacks from stacking — at most one
pending callback per interval. But drift accumulates.

**Self-correcting timer pattern**:
```javascript
function preciseInterval(callback, interval) {
  let expected = performance.now() + interval;

  function tick() {
    const drift = performance.now() - expected;
    callback();
    expected += interval;
    setTimeout(tick, Math.max(0, interval - drift));
  }

  setTimeout(tick, interval);
}
```

---

## Node.js Event Loop

Node.js uses **libuv**, which implements a different event loop with
explicit phases. Unlike the browser's spec-defined loop, Node's loop
has a fixed phase order.

### The Phases

```
   ┌───────────────────────────┐
┌─▶│         timers            │  setTimeout, setInterval
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐
│  │     pending callbacks     │  I/O callbacks deferred from previous loop
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐
│  │       idle, prepare       │  internal use only
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐
│  │          poll              │  Retrieve new I/O events; execute I/O
│  │                           │  callbacks. Node may block here.
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐
│  │          check            │  setImmediate callbacks
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐
│  │     close callbacks       │  socket.on('close', ...) etc.
│  └─────────────┬─────────────┘
│                │
└────────────────┘
```

### Phase Details

**timers**: executes callbacks for `setTimeout` and `setInterval`
whose threshold has elapsed. Note: the timer threshold is a
**minimum** delay, not exact. If the poll phase takes 50ms and a
timer is set for 10ms, it fires at 50ms.

**pending callbacks**: executes I/O callbacks deferred to the next
loop iteration (e.g., TCP errors like ECONNREFUSED on some systems).

**poll**: the most important phase. Two functions:
1. Calculate how long it should block and wait for I/O
2. Process events in the poll queue

If the poll queue is not empty, iterate through the queue executing
callbacks synchronously until the queue is empty or the hard limit
is reached.

If the poll queue IS empty:
- If `setImmediate` is scheduled → move to check phase
- If timers are scheduled → loop back to timers phase
- Otherwise → **block here** waiting for new I/O events

**check**: `setImmediate()` callbacks. Always runs after poll.

**close callbacks**: `socket.on('close', ...)` and similar cleanup.

### Microtasks in Node.js

Between each phase (and between each callback within a phase), Node
drains:
1. `process.nextTick()` queue (always first)
2. Promise microtask queue (always second)

```javascript
setImmediate(() => console.log('1: setImmediate'));
setTimeout(() => console.log('2: setTimeout'), 0);
process.nextTick(() => console.log('3: nextTick'));
Promise.resolve().then(() => console.log('4: promise'));
console.log('5: sync');

// Output:
// 5: sync
// 3: nextTick     (microtask — runs between sync code and first phase)
// 4: promise       (microtask — runs after all nextTicks)
// 2: setTimeout    (timers phase — may vary with setImmediate)
// 1: setImmediate  (check phase — may vary with setTimeout)
```

**setTimeout vs setImmediate order**: when called from the main module
(not inside an I/O callback), the order is **non-deterministic**:

```javascript
// Order depends on process performance at startup
setTimeout(() => console.log('timeout'), 0);
setImmediate(() => console.log('immediate'));
// Could be either order!

// But inside an I/O callback, setImmediate always fires first:
const fs = require('fs');
fs.readFile(__filename, () => {
  setTimeout(() => console.log('timeout'), 0);
  setImmediate(() => console.log('immediate'));
  // Always: immediate, then timeout
  // Because we're in the poll phase → check phase (setImmediate)
  // runs before looping back to timers phase (setTimeout)
});
```

### process.nextTick()

`nextTick` is NOT part of the event loop phases. It runs between
every phase transition and after every callback:

```javascript
process.nextTick(() => {
  process.nextTick(() => {
    process.nextTick(() => {
      console.log('nested nextTicks');
    });
  });
});
// All three run before ANY I/O, timer, or setImmediate
```

**Starvation risk**: like browser microtasks, recursive `nextTick`
starves the event loop:

```javascript
function recursive() {
  process.nextTick(recursive);  // I/O and timers NEVER fire
}
recursive();
```

This is why `setImmediate` is preferred for breaking up work — it
yields to the event loop between iterations.

### process.nextTick vs setImmediate vs Promise

```
process.nextTick()  — fires BEFORE I/O, between every phase
Promise.then()      — fires after all nextTicks, before I/O
setImmediate()      — fires in the check phase (after I/O poll)
setTimeout(fn, 0)   — fires in the timers phase (next iteration)
```

Order within a callback:
```javascript
fs.readFile('file', () => {
  process.nextTick(() => console.log('1: nextTick'));
  Promise.resolve().then(() => console.log('2: promise'));
  setImmediate(() => console.log('3: setImmediate'));
  setTimeout(() => console.log('4: setTimeout'), 0);
});
// 1: nextTick
// 2: promise
// 3: setImmediate
// 4: setTimeout
```

---

## Long Tasks and Yielding

### What Makes a Task "Long"?

A **Long Task** is any task that takes >50ms. During a long task:
- No rendering
- No user input processing
- No other callbacks
- The page appears frozen

The **Interaction to Next Paint (INP)** metric measures how long
user interactions are blocked. The target is <200ms.

### Strategies for Yielding

**1. setTimeout yielding** — break work into chunks:
```javascript
async function processLargeArray(items) {
  const CHUNK_SIZE = 100;

  for (let i = 0; i < items.length; i += CHUNK_SIZE) {
    const chunk = items.slice(i, i + CHUNK_SIZE);
    processChunk(chunk);

    // Yield to the event loop
    await new Promise(resolve => setTimeout(resolve, 0));
  }
}
```

**2. scheduler.yield()** — the modern API (Chrome 129+):
```javascript
async function processItems(items) {
  for (const item of items) {
    processItem(item);

    if (shouldYield()) {
      await scheduler.yield();
      // Resumes with HIGH priority (unlike setTimeout which is low)
    }
  }
}
```

The key advantage of `scheduler.yield()` over `setTimeout`: after
yielding, the continuation runs at **user-visible** priority, meaning
it resumes before other lower-priority tasks. `setTimeout` puts the
continuation at the back of the normal task queue.

**3. scheduler.postTask()** — priority-based scheduling:
```javascript
// Three priority levels:
scheduler.postTask(() => criticalWork(), { priority: 'user-blocking' });
scheduler.postTask(() => normalWork(),   { priority: 'user-visible' });
scheduler.postTask(() => bgWork(),       { priority: 'background' });
```

**4. isInputPending()** — check if user input is waiting:
```javascript
function processItems(deadline) {
  while (items.length > 0) {
    if (navigator.scheduling?.isInputPending()) {
      // User input is waiting — yield immediately
      setTimeout(processItems, 0);
      return;
    }
    processItem(items.pop());
  }
}
```

**5. Web Workers** — move computation off the main thread entirely
(covered in 04-workers.md).

---

## Async/Await and the Event Loop

`async`/`await` is syntactic sugar over Promises, which means it
interacts with the **microtask** queue:

```javascript
async function foo() {
  console.log('A');
  await bar();         // Yields here — schedules continuation as microtask
  console.log('B');    // Runs as microtask
}

async function bar() {
  console.log('C');
}

console.log('D');
foo();
console.log('E');

// Output:
// D         (sync)
// A         (sync — foo() runs synchronously until first await)
// C         (sync — bar() runs synchronously)
// E         (sync — execution returns to caller)
// B         (microtask — continuation after await)
```

### await desugaring

```javascript
async function example() {
  const x = await somePromise;
  doSomething(x);
}

// Roughly equivalent to:
function example() {
  return somePromise.then(x => {
    doSomething(x);
  });
}
```

Each `await` creates a microtask checkpoint. Multiple awaits in
sequence are multiple microtask transitions:

```javascript
async function multiAwait() {
  const a = await fetch('/a');    // microtask 1
  const b = await fetch('/b');    // microtask 2
  const c = await fetch('/c');    // microtask 3
  return [a, b, c];
}

// This is sequential. For parallel:
async function parallel() {
  const [a, b, c] = await Promise.all([
    fetch('/a'),
    fetch('/b'),
    fetch('/c'),
  ]);
  return [a, b, c];
}
```

---

## MessageChannel and postMessage

`MessageChannel` creates a pair of ports for posting messages. Messages
are delivered as **tasks** (macrotasks), not microtasks:

```javascript
const channel = new MessageChannel();
const port = channel.port2;

port.onmessage = () => console.log('message');
channel.port1.postMessage(null);

Promise.resolve().then(() => console.log('microtask'));
console.log('sync');

// Output:
// sync
// microtask    (microtask before macrotask)
// message      (macrotask from MessageChannel)
```

React's scheduler uses `MessageChannel` for yielding because it fires
sooner than `setTimeout` (no 4ms clamp) but still yields to the
event loop (unlike microtasks).

---

## queueMicrotask vs Promise.resolve().then

Both queue microtasks, but with subtle differences:

```javascript
// These are functionally equivalent for scheduling:
queueMicrotask(() => console.log('a'));
Promise.resolve().then(() => console.log('b'));

// But Promise.resolve().then() creates a Promise object + resolves it.
// queueMicrotask() directly queues a microtask — slightly less overhead.

// Use queueMicrotask() when you just need to defer work to the microtask
// queue and don't need Promise chaining.
```

---

## MutationObserver and Microtasks

`MutationObserver` batches DOM mutations and fires a callback as a
microtask:

```javascript
const observer = new MutationObserver(mutations => {
  console.log(`${mutations.length} mutations`);
});

observer.observe(element, { childList: true, subtree: true });

// These three mutations are batched into ONE callback:
element.appendChild(child1);
element.appendChild(child2);
element.appendChild(child3);

// At the microtask checkpoint: "3 mutations"
```

This is more efficient than individual event listeners because the
callback fires once per microtask checkpoint, not once per mutation.

---

## Event Dispatch and the Loop

User events (click, keydown) are dispatched as tasks:

```javascript
button.addEventListener('click', () => {
  console.log('1: click handler');
  Promise.resolve().then(() => console.log('2: microtask'));
});

// User clicks button:
// 1: click handler  (task)
// 2: microtask      (microtask checkpoint after handler)
```

But `element.click()` (programmatic click) is **synchronous** — it
runs the handler inline, not as a separate task:

```javascript
console.log('A');
button.click();       // Synchronous! Handler runs here.
console.log('B');
// A, 1: click handler, B, 2: microtask
// (microtask runs during the microtask checkpoint after the current task)

// vs user click:
// A, B, 1: click handler, 2: microtask
```

This difference matters for event handler ordering and microtask
timing.

---

## Key Takeaways

| Concept | Why It Matters |
|---------|---------------|
| Single task per iteration | One long task blocks everything — keep tasks <50ms |
| Microtasks drain fully | Recursive microtasks permanently freeze the page |
| rAF before paint | Animation updates go in rAF, not setTimeout |
| setTimeout ≠ 0ms | 4ms minimum after nesting depth 5 |
| Node phases are fixed | poll → check → timers is the guaranteed order |
| nextTick before Promise | nextTick fires before Promise microtasks in Node |
| setImmediate yields | Unlike nextTick, it lets I/O and timers interleave |
| Layout thrashing | Interleaved read/write forces synchronous layout |
| scheduler.yield() | Modern yielding API — resumes at high priority |
| MessageChannel for scheduling | No 4ms clamp, yields to event loop (React uses this) |
