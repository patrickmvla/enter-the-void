# Web Workers — True Parallelism in the Browser

## Why Workers Exist

The main thread handles JavaScript execution, DOM updates, layout,
paint, compositing, and user input. That's too much for one thread.
Any computation-heavy work blocks everything else.

Web Workers give JavaScript **real OS threads**. Worker threads run
JavaScript in parallel with the main thread — separate call stacks,
separate event loops, separate heaps. No shared memory by default.

---

## Dedicated Workers

A dedicated worker belongs to a single page/script.

### Lifecycle

```javascript
// main.js
const worker = new Worker('worker.js');

worker.postMessage({ type: 'process', data: largeArray });

worker.onmessage = (event) => {
  console.log('Result:', event.data);
};

worker.onerror = (event) => {
  console.error('Worker error:', event.message);
};

// When done:
worker.terminate();
```

```javascript
// worker.js
self.onmessage = (event) => {
  const { type, data } = event.data;

  if (type === 'process') {
    const result = heavyComputation(data);
    self.postMessage(result);
  }
};

// Worker can also terminate itself:
self.close();
```

### What Workers CAN'T Access

Workers run in a **restricted global scope** (`WorkerGlobalScope`):

```
Main Thread (Window)          Worker Thread (WorkerGlobalScope)
─────────────────────         ──────────────────────────────────
✓ document / DOM              ✗ NO DOM access
✓ window                      ✗ NO window
✓ localStorage                ✗ NO localStorage
✓ sessionStorage              ✗ NO sessionStorage
✓ alert/confirm/prompt        ✗ NO UI dialogs
✓ requestAnimationFrame       ✗ NO rAF
                              ✓ fetch / XMLHttpRequest
                              ✓ IndexedDB
                              ✓ WebSocket
                              ✓ setTimeout / setInterval
                              ✓ crypto
                              ✓ performance
                              ✓ importScripts()
                              ✓ navigator (subset)
                              ✓ Cache API
```

---

## Message Passing — Structured Clone

Workers communicate via `postMessage`. Data is copied using the
**structured clone algorithm**, not JSON serialization.

### What Structured Clone Handles

```
JSON.parse/stringify          Structured Clone
──────────────────           ──────────────
✓ Objects, Arrays            ✓ Objects, Arrays
✓ Strings, Numbers           ✓ Strings, Numbers
✓ Booleans, null             ✓ Booleans, null
✗ undefined                  ✓ undefined
✗ Date (becomes string)      ✓ Date (preserves as Date)
✗ RegExp                     ✓ RegExp
✗ Map/Set                    ✓ Map/Set
✗ ArrayBuffer/TypedArrays    ✓ ArrayBuffer/TypedArrays
✗ Blob/File                  ✓ Blob/File
✗ ImageBitmap                ✓ ImageBitmap
✗ Circular references        ✓ Circular references
✗ Error objects              ✓ Error objects

Neither can clone:
✗ Functions
✗ DOM nodes
✗ Symbols
✗ WeakMap/WeakSet
```

### The Copy Problem

Structured clone **copies** data. For a 100MB ArrayBuffer:

```javascript
// This copies 100MB of data:
const buffer = new ArrayBuffer(100 * 1024 * 1024);
worker.postMessage(buffer);
// Main thread: still has buffer (100MB)
// Worker: has a copy (100MB)
// Total memory: 200MB
// Time: ~50-100ms (memcpy)
```

For large data, copying is unacceptable.

---

## Transferable Objects

Transferable objects are **moved**, not copied. After transfer, the
original reference becomes unusable (zero bytes, neutered).

```javascript
const buffer = new ArrayBuffer(100 * 1024 * 1024);

// Transfer (second argument is the transfer list):
worker.postMessage(buffer, [buffer]);

// Now:
console.log(buffer.byteLength);  // 0 — buffer has been neutered!
// Worker: has the buffer (100MB)
// Total memory: 100MB
// Time: ~0ms (just pointer move)
```

Transferable types:
- `ArrayBuffer`
- `MessagePort`
- `ReadableStream`
- `WritableStream`
- `TransformStream`
- `ImageBitmap`
- `OffscreenCanvas`

### How Transfer Works Internally

The browser doesn't copy memory. It:
1. Marks the source ArrayBuffer as neutered (byteLength → 0)
2. Moves the underlying memory page mappings to the worker's address
   space
3. Creates a new ArrayBuffer in the worker pointing to the same
   physical memory

This is O(1) regardless of buffer size — it's a pointer/mapping
transfer, not a data copy.

### Transfer + Message Pattern

You can mix transferred and cloned data:

```javascript
const buffer = new ArrayBuffer(1024);
const metadata = { width: 100, height: 100, format: 'rgba' };

// metadata is cloned, buffer is transferred:
worker.postMessage(
  { metadata, pixels: buffer },
  [buffer]  // Transfer list: only buffer is transferred
);
```

---

## SharedArrayBuffer and Atomics

`SharedArrayBuffer` (SAB) enables **true shared memory** between
threads — no copying, no transfer. Multiple threads read/write the
same memory simultaneously.

### Basic Usage

```javascript
// main.js
const sab = new SharedArrayBuffer(1024);
const view = new Int32Array(sab);

view[0] = 42;
worker.postMessage(sab);  // NOT transferred — shared!

// worker.js
self.onmessage = (event) => {
  const sab = event.data;
  const view = new Int32Array(sab);

  console.log(view[0]);  // 42 — same memory!
  view[0] = 100;         // Main thread sees this change too
};
```

### The Concurrency Problem

Shared memory without synchronization is a race condition minefield:

```javascript
// Thread A:                  Thread B:
view[0] = 1;                 view[0] = 2;
// What is view[0]? Could be 1 or 2.

// Worse — non-atomic read/write of 64-bit values on 32-bit systems:
// A 64-bit write is TWO 32-bit writes. Thread B might read
// the high 32 bits from the old value and low 32 bits from the new.
```

### Atomics API

`Atomics` provides atomic operations on SharedArrayBuffer:

```javascript
const sab = new SharedArrayBuffer(4);
const view = new Int32Array(sab);

// Atomic read/write
Atomics.store(view, 0, 42);        // Atomic write
const val = Atomics.load(view, 0);  // Atomic read

// Atomic read-modify-write
Atomics.add(view, 0, 5);           // Atomically: view[0] += 5
Atomics.sub(view, 0, 3);           // Atomically: view[0] -= 3
Atomics.and(view, 0, 0xFF);        // Atomically: view[0] &= 0xFF
Atomics.or(view, 0, 0x80);         // Atomically: view[0] |= 0x80
Atomics.xor(view, 0, 0xFF);        // Atomically: view[0] ^= 0xFF

// Compare-and-swap (CAS)
Atomics.compareExchange(view, 0, expected, newValue);
// If view[0] === expected: set to newValue, return expected
// If view[0] !== expected: return current value (no change)

// Exchange
Atomics.exchange(view, 0, newValue);
// Set view[0] = newValue, return old value
```

### Wait/Notify (Futex-like)

`Atomics.wait()` and `Atomics.notify()` implement a futex — a
mechanism where a thread can sleep until another thread wakes it:

```javascript
// Worker (waiting thread):
const view = new Int32Array(sab);
Atomics.wait(view, 0, 0);  // Sleep while view[0] === 0
// Blocked here until notified...
console.log('Woken up! Value:', Atomics.load(view, 0));

// Main thread (NOT allowed to use Atomics.wait):
const view = new Int32Array(sab);
Atomics.store(view, 0, 1);
Atomics.notify(view, 0, 1);  // Wake 1 waiting thread
```

**Main thread cannot call `Atomics.wait()`** — it would block the
event loop. Only workers can wait. Use `Atomics.waitAsync()` on the
main thread instead (returns a Promise).

### Building a Mutex with Atomics

```javascript
const UNLOCKED = 0;
const LOCKED = 1;

class Mutex {
  constructor(sab, offset = 0) {
    this.view = new Int32Array(sab, offset, 1);
  }

  lock() {
    while (true) {
      // Try to acquire: CAS from UNLOCKED to LOCKED
      if (Atomics.compareExchange(this.view, 0, UNLOCKED, LOCKED) === UNLOCKED) {
        return; // Acquired!
      }
      // Spin failed — wait for notification
      Atomics.wait(this.view, 0, LOCKED);
    }
  }

  unlock() {
    Atomics.store(this.view, 0, UNLOCKED);
    Atomics.notify(this.view, 0, 1); // Wake one waiter
  }
}

// Usage in worker:
const mutex = new Mutex(sharedBuffer, 0);
mutex.lock();
// Critical section — safe to modify shared data
sharedArray[1] = computeResult();
mutex.unlock();
```

### Building a Ring Buffer with SharedArrayBuffer

```javascript
// Shared memory layout:
// [0]: write index (atomic)
// [1]: read index (atomic)
// [2..N+1]: ring buffer data

class SharedRingBuffer {
  constructor(sab, capacity) {
    this.meta = new Int32Array(sab, 0, 2);
    this.data = new Float64Array(sab, 8, capacity);
    this.capacity = capacity;
  }

  push(value) {
    const writeIdx = Atomics.load(this.meta, 0);
    const readIdx = Atomics.load(this.meta, 1);
    const nextWrite = (writeIdx + 1) % this.capacity;

    if (nextWrite === readIdx) return false; // Full

    this.data[writeIdx] = value;
    Atomics.store(this.meta, 0, nextWrite);
    Atomics.notify(this.meta, 0, 1); // Wake consumer
    return true;
  }

  pop() {
    const readIdx = Atomics.load(this.meta, 1);
    const writeIdx = Atomics.load(this.meta, 0);

    if (readIdx === writeIdx) {
      Atomics.wait(this.meta, 0, writeIdx); // Wait for data
      return this.pop(); // Retry
    }

    const value = this.data[readIdx];
    Atomics.store(this.meta, 1, (readIdx + 1) % this.capacity);
    return value;
  }
}
```

---

## Spectre Mitigations — COOP/COEP

In 2018, the Spectre CPU vulnerability showed that high-resolution
timers (like `SharedArrayBuffer` used as a timer) could leak data
across security boundaries. Browsers disabled SAB entirely, then
re-enabled it behind security requirements.

### The Attack

```javascript
// SharedArrayBuffer as high-resolution timer:
// Worker increments shared counter in a tight loop
const sab = new SharedArrayBuffer(4);
const view = new Int32Array(sab);

// Worker:
while (true) { Atomics.add(view, 0, 1); }

// Main thread uses view[0] as a nanosecond-precision timer
// to measure cache timing side-channels (Spectre)
```

### The Fix — Cross-Origin Isolation

To use `SharedArrayBuffer`, your page must be **cross-origin isolated**:

```http
Cross-Origin-Opener-Policy: same-origin
Cross-Origin-Embedder-Policy: require-corp
```

**COOP** (`Cross-Origin-Opener-Policy: same-origin`):
- Isolates your browsing context group
- Prevents cross-origin windows from getting a reference to your window
- `window.opener` becomes `null` for cross-origin navigations

**COEP** (`Cross-Origin-Embedder-Policy: require-corp`):
- Every sub-resource must explicitly opt in to being loaded
- Resources need `Cross-Origin-Resource-Policy: cross-origin` header
- Or be loaded with `crossorigin` attribute

```javascript
// Check if cross-origin isolated:
if (self.crossOriginIsolated) {
  // SharedArrayBuffer is available
  const sab = new SharedArrayBuffer(1024);
}
```

Why this helps: cross-origin isolation ensures no cross-origin
resources are loaded without explicit consent, preventing Spectre
from reading their data through timing side-channels.

---

## Shared Workers

A Shared Worker is shared among all pages from the same origin:

```javascript
// page1.js and page2.js both:
const shared = new SharedWorker('shared-worker.js');
shared.port.start();
shared.port.postMessage('hello from page');
shared.port.onmessage = (e) => console.log(e.data);
```

```javascript
// shared-worker.js
const connections = [];

self.onconnect = (event) => {
  const port = event.ports[0];
  connections.push(port);

  port.onmessage = (e) => {
    // Broadcast to all connected pages:
    for (const conn of connections) {
      conn.postMessage(`Broadcast: ${e.data}`);
    }
  };

  port.start();
};
```

Use cases:
- Shared WebSocket connection across tabs
- Shared state/cache across tabs
- Reducing redundant network requests

Shared Workers are killed when the last connected page closes.

---

## Worker Pools

Creating workers is expensive (~5-20ms each). For repeated tasks,
maintain a pool:

```javascript
class WorkerPool {
  constructor(workerScript, poolSize = navigator.hardwareConcurrency) {
    this.workers = [];
    this.queue = [];
    this.available = [];

    for (let i = 0; i < poolSize; i++) {
      const worker = new Worker(workerScript);
      this.workers.push(worker);
      this.available.push(worker);
    }
  }

  execute(data) {
    return new Promise((resolve, reject) => {
      const task = { data, resolve, reject };

      const worker = this.available.pop();
      if (worker) {
        this._runTask(worker, task);
      } else {
        this.queue.push(task);
      }
    });
  }

  _runTask(worker, task) {
    worker.onmessage = (e) => {
      task.resolve(e.data);
      this._returnWorker(worker);
    };
    worker.onerror = (e) => {
      task.reject(e);
      this._returnWorker(worker);
    };
    worker.postMessage(task.data);
  }

  _returnWorker(worker) {
    const nextTask = this.queue.shift();
    if (nextTask) {
      this._runTask(worker, nextTask);
    } else {
      this.available.push(worker);
    }
  }

  terminate() {
    this.workers.forEach(w => w.terminate());
  }
}

// Usage:
const pool = new WorkerPool('compute.js', 4);
const results = await Promise.all(
  chunks.map(chunk => pool.execute(chunk))
);
```

---

## OffscreenCanvas

`OffscreenCanvas` moves canvas rendering off the main thread:

```javascript
// main.js
const canvas = document.getElementById('canvas');
const offscreen = canvas.transferControlToOffscreen();

const worker = new Worker('render-worker.js');
worker.postMessage({ canvas: offscreen }, [offscreen]);
```

```javascript
// render-worker.js
self.onmessage = (event) => {
  const canvas = event.data.canvas;
  const ctx = canvas.getContext('2d');

  function render() {
    ctx.clearRect(0, 0, canvas.width, canvas.height);
    // Heavy rendering work — doesn't block main thread
    drawComplexScene(ctx);

    // requestAnimationFrame works in workers with OffscreenCanvas!
    requestAnimationFrame(render);
  }
  requestAnimationFrame(render);
};
```

WebGL also works with OffscreenCanvas — game engines use this to
run the entire render loop in a worker.

---

## Module Workers

Workers can use ES module syntax:

```javascript
// main.js
const worker = new Worker('worker.js', { type: 'module' });

// worker.js
import { heavyCompute } from './compute.js';
import { format } from './format.js';

self.onmessage = (event) => {
  const result = heavyCompute(event.data);
  self.postMessage(format(result));
};
```

Benefits:
- Static imports (the browser can optimize loading)
- `import.meta.url` for relative paths
- Top-level await
- Strict mode by default

---

## Worklets

Worklets are lightweight workers designed for specific rendering
pipeline stages. They run on the **rendering thread**, not a worker
thread:

**AudioWorklet** — custom audio processing:
```javascript
// Runs on the audio rendering thread (not main, not worker)
class NoiseProcessor extends AudioWorkletProcessor {
  process(inputs, outputs) {
    const output = outputs[0][0];
    for (let i = 0; i < output.length; i++) {
      output[i] = Math.random() * 2 - 1;
    }
    return true; // Keep processing
  }
}
registerProcessor('noise', NoiseProcessor);
```

**PaintWorklet (CSS Houdini)** — custom CSS paint:
```javascript
class CheckerboardPainter {
  paint(ctx, geom, properties) {
    const size = 32;
    for (let y = 0; y < geom.height; y += size) {
      for (let x = 0; x < geom.width; x += size) {
        if ((x / size + y / size) % 2 === 0) {
          ctx.fillRect(x, y, size, size);
        }
      }
    }
  }
}
registerPaint('checkerboard', CheckerboardPainter);
```

```css
.element {
  background-image: paint(checkerboard);
}
```

**AnimationWorklet** — animations synchronized with the compositor:
```javascript
class ScrollTimeline extends AnimationWorkletGlobalScope {
  animate(currentTime, effect) {
    effect.localTime = currentTime * 0.5; // Half-speed scroll effect
  }
}
```

---

## Performance Characteristics

| Operation | Cost |
|-----------|------|
| Worker creation | 5-20ms |
| postMessage (small object) | ~0.1ms |
| postMessage (1MB clone) | ~5-10ms |
| postMessage (100MB clone) | ~50-100ms |
| Transfer (any size) | ~0.01ms |
| SharedArrayBuffer access | Same as regular memory |
| Atomics.wait wakeup | ~0.01-0.1ms |

Rules of thumb:
- Use workers for tasks >16ms (otherwise the overhead isn't worth it)
- Transfer large buffers, don't clone them
- Use SharedArrayBuffer for frequently shared state
- Pool workers — don't create/destroy per task
- `navigator.hardwareConcurrency` tells you available cores

---

## Key Takeaways

| Concept | Why It Matters |
|---------|---------------|
| Workers are real OS threads | True parallelism, not just concurrency |
| No shared state by default | Eliminates race conditions for message passing |
| Structured clone copies | Data is copied on postMessage — watch for large payloads |
| Transferable objects | O(1) transfer of ArrayBuffers — no copy |
| SharedArrayBuffer | True shared memory when you need it |
| Atomics for synchronization | CAS, wait/notify — build locks and queues |
| COOP/COEP required for SAB | Spectre mitigation — cross-origin isolation |
| OffscreenCanvas | Move rendering off main thread |
| Worker pools | Amortize creation cost across many tasks |
| Worklets | Lightweight, pipeline-specific (audio, paint, animation) |
