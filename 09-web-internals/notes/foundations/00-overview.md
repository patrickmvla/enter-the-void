# Web Internals — How Browsers and Runtimes Actually Work

## The Core Problem

You write HTML, CSS, and JavaScript. The browser turns that into
pixels on screen and interactive behavior. Between your code and
the pixels is a staggeringly complex pipeline — parsing, style
computation, layout, painting, compositing, JIT compilation, garbage
collection, event scheduling, and more.

Understanding this machinery at the internals level means you can
write code that works WITH the browser instead of against it. You
know why a CSS property triggers layout but not paint. Why a long
JavaScript task makes the page unresponsive. Why moving an animation
to a compositor layer eliminates jank. Why `requestAnimationFrame`
exists and when to use it.

---

## The Topics

```
Start here
│
├── foundations/
│   ├── 01-rendering-pipeline.md
│   │   The critical rendering path: HTML→DOM, CSS→CSSOM, render
│   │   tree, layout, paint, composite. GPU compositing and layers.
│   │   Frame budgets (16.67ms), reflow vs repaint, what triggers
│   │   each. Content visibility and containment.
│   │
│   └── 02-v8-engine.md
│       V8's compilation pipeline: parsing → Ignition bytecode →
│       TurboFan optimized machine code. Hidden classes and inline
│       caching. Deoptimization. Generational garbage collection
│       (Scavenger, Mark-Compact, Orinoco concurrent GC).
│
├── runtime/
│   ├── 03-event-loop.md
│   │   Browser event loop: call stack, task queue, microtask queue,
│   │   requestAnimationFrame, rendering steps. Node.js event loop:
│   │   libuv phases. process.nextTick vs setImmediate vs Promises.
│   │   Blocking, yielding, and scheduling strategies.
│   │
│   ├── 04-workers.md
│   │   Web Workers (dedicated, shared), message passing, structured
│   │   clone, transferable objects. SharedArrayBuffer + Atomics.
│   │   Spectre mitigations (COOP/COEP). OffscreenCanvas.
│   │
│   ├── 05-promise-all-internals.md
│   │   ECMA-262 §27.2.4.1 algorithm step-by-step. V8's JSPromise
│   │   C++ layout, PromiseReaction records, Torque fast paths.
│   │   Microtask scheduling analysis (N+2 ticks), memory footprint
│   │   (~132*N bytes). Iterable protocol edge cases. Concurrency
│   │   control patterns. 10+ common bugs and antipatterns.
│   │   TypeScript Awaited<T> and tuple overloads.
│   │
│   ├── 08-concurrency.md
│   │   Cooperative multitasking model and run-to-completion.
│   │   Promise combinators (all/allSettled/race/any). AsyncMutex,
│   │   Semaphore, concurrency pool implementations. 7 patterns
│   │   (fan-out, pipeline, racing, retry, pool, first-successful,
│   │   coalesce). Cancellation with AbortController/AbortSignal.
│   │   Async generators and structured concurrency (TaskGroup).
│   │
│   ├── 09-async-typescript.md
│   │   Promise<T>, PromiseLike<T>, Awaited<T> type internals.
│   │   Async function return type transformation. Error handling
│   │   patterns (Result type, tuple, custom error classes).
│   │   Generic async utilities (withTimeout, retry, pool).
│   │   Deferred, Promise.withResolvers, await using (Explicit
│   │   Resource Management). Async state machines, middleware
│   │   pipelines, typed event emitters. __awaiter transform.
│   │
│   ├── 10-promise-internals.md
│   │   ECMA-262 Promise abstract operations: PromiseCapability,
│   │   PromiseReaction records, the resolve algorithm (14 steps).
│   │   Two microtask job types (PromiseReactionJob vs
│   │   PromiseResolveThenableJob). V8 JSPromise layout (~20 bytes
│   │   compressed), fast paths, inline caching. Promise.race
│   │   memory leak (reaction accumulation). Unhandled rejection
│   │   tracking (browser vs Node.js). Promises/A+ vs ECMA-262.
│   │   Promise.withResolvers (ES2024). Memory model (~104 bytes
│   │   per new Promise()).
│   │
│   └── 11-async-runtime.md
│       async/await desugaring: __awaiter state machine, generator-
│       based compilation, Babel regenerator-runtime. V8's zero-cost
│       optimization (3→1 microticks), PromiseResolve spec change.
│       Zero-cost async stack traces (lazy materialization). Top-
│       level await (module evaluation, deadlock risks). AsyncContext
│       and Node.js AsyncLocalStorage. Engine-specific JIT: V8
│       TurboFan, SpiderMonkey Warp, JSC FTL. await as optimization
│       barrier. Scheduler patterns: postTask, yield, React's
│       MessageChannel + min-heap scheduler.
│
└── platform/
    ├── 05-service-workers.md
    │   Lifecycle (install, activate, fetch). Caching strategies
    │   (cache-first, network-first, stale-while-revalidate).
    │   Background sync, push notifications, navigation preload.
    │
    ├── 06-webassembly.md
    │   Binary format and module structure. Linear memory model.
    │   Value types. Instantiation and JS interop. Streaming
    │   compilation. WASI and the system interface. Use cases
    │   and when Wasm beats JavaScript.
    │
    ├── 07-json.md
    │   RFC 8259 grammar (ABNF production rules). Complete
    │   recursive-descent parser implementation. V8 JSON.parse
    │   fast path internals. Surrogate pair math for Unicode
    │   escapes. Security (prototype pollution, JSON injection).
    │   Streaming/NDJSON. Binary alternatives (MessagePack, CBOR,
    │   BSON, protobuf) with byte-level format analysis. JSON
    │   Schema, JSON Pointer, JSON Patch (RFC 6901/6902).
    │
    └── 08-streams.md
        WHATWG Streams API: ReadableStream, WritableStream,
        TransformStream. Backpressure via desiredSize, pull(),
        and queuing strategies. Piping (pipeTo, pipeThrough) with
        automatic backpressure propagation. Built-in transforms
        (TextDecoder/Encoder, Compression/Decompression). BYOB
        readers, stream teeing (memory pitfall). Streaming fetch
        patterns: response streaming, upload streams, SSE, NDJSON.
        Node.js streams vs WHATWG streams. Custom transform
        patterns (chunking, throttle, filter, batch). Error
        propagation and AbortController integration.
```
