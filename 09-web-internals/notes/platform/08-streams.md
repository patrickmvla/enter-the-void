# WHATWG Streams — Backpressure, Piping, and Async Iteration

## The Core Problem

Data arrives in chunks — network responses, file reads, sensor data,
WebSocket messages. You need to process these chunks as they arrive,
not buffer everything into memory first. A 2GB file download should
use kilobytes of memory, not gigabytes.

The WHATWG Streams API provides the abstraction: data flows from a
**source** through optional **transforms** to a **sink**, with
**backpressure** ensuring fast producers don't overwhelm slow
consumers.

---

## The Three Stream Types

### ReadableStream — A Source of Data

A ReadableStream produces chunks on demand:

```javascript
const stream = new ReadableStream({
  start(controller) {
    // Called once when the stream is constructed.
    // Use for setup. Can return a promise.
    controller.enqueue('chunk 1');
    controller.enqueue('chunk 2');
    controller.close();
  },

  pull(controller) {
    // Called when the internal queue has space.
    // This IS the backpressure mechanism on the source side.
    // Return a promise to delay the next pull.
    return fetch('/next-chunk').then(res => {
      controller.enqueue(res.body);
    });
  },

  cancel(reason) {
    // Called when the consumer cancels the stream.
    // Clean up resources.
    console.log('Stream cancelled:', reason);
  }
}, new CountQueuingStrategy({ highWaterMark: 10 }));
```

The `controller` provides:
- `controller.enqueue(chunk)` — push data into the stream's queue
- `controller.close()` — signal no more data
- `controller.error(e)` — signal an error
- `controller.desiredSize` — how much room is in the queue
  (negative = over the high-water mark)

### WritableStream — A Sink for Data

A WritableStream consumes chunks:

```javascript
const stream = new WritableStream({
  start(controller) {
    // Setup. Can return a promise.
  },

  write(chunk, controller) {
    // Called for each chunk. RETURN A PROMISE for backpressure.
    // The next chunk won't be written until this promise resolves.
    return db.insert(chunk);
  },

  close() {
    // All chunks written. Finalize.
    return db.commit();
  },

  abort(reason) {
    // Stream was aborted. Clean up.
    return db.rollback();
  }
});
```

The key: `write()` returns a promise. If writing is slow, the promise
stays pending, and the stream applies backpressure upstream.

### TransformStream — A Transform in the Middle

A TransformStream has both a writable side (input) and a readable
side (output):

```javascript
const { readable, writable } = new TransformStream({
  start(controller) {
    // Setup.
  },

  transform(chunk, controller) {
    // Process each input chunk, enqueue zero or more output chunks.
    controller.enqueue(chunk.toUpperCase());
    // Can also return a promise for async transforms.
  },

  flush(controller) {
    // Called after all input chunks processed.
    // Enqueue any remaining output.
    controller.enqueue('[END]');
  }
});
```

TransformStreams are composable — you chain them via `pipeThrough`:

```javascript
response.body
  .pipeThrough(new DecompressionStream('gzip'))
  .pipeThrough(new TextDecoderStream())
  .pipeThrough(jsonParseTransform)
  .pipeTo(writableStream);
```

---

## Backpressure — The Critical Mechanism

Backpressure is how a slow consumer signals a fast producer to
slow down. Without it, fast producers overwhelm slow consumers,
filling memory until the process crashes.

### The Internal Queue

Every ReadableStream has an internal queue with a **high-water mark**:

```
                    highWaterMark = 3
                    ┌────────────────┐
Source → enqueue → │ [c1] [c2] [c3] │ → read → Consumer
                    └────────────────┘
                    desiredSize = 0 → stop pulling

Source → enqueue → │ [c1] [c2] [c3] [c4] │
                    desiredSize = -1 → OVER high-water mark
                    pull() NOT called until queue drains
```

The `desiredSize` property tells the source how much room remains:
- `desiredSize > 0` — room in the queue, `pull()` will be called
- `desiredSize <= 0` — queue full, `pull()` won't be called
- `desiredSize === null` — stream is closed or errored

### Queuing Strategies

Two built-in strategies control how "size" is measured:

**CountQueuingStrategy**: Each chunk counts as 1 unit.
```javascript
new CountQueuingStrategy({ highWaterMark: 10 })
// Queue holds up to 10 chunks regardless of their size
```

**ByteLengthQueuingStrategy**: Each chunk's size is its byte length.
```javascript
new ByteLengthQueuingStrategy({ highWaterMark: 1024 * 64 })
// Queue holds up to 64 KB of data
```

You can also provide a custom `size()` function:

```javascript
new ReadableStream(source, {
  highWaterMark: 100,
  size(chunk) {
    return chunk.length; // or any numeric metric
  }
});
```

### How Backpressure Flows Through a Pipe

```
ReadableStream ──pipeTo──► WritableStream

1. pipe reads from ReadableStream
2. pipe writes to WritableStream
3. If writer.write() returns a pending promise (sink is slow):
   a. pipe stops reading from ReadableStream
   b. ReadableStream's internal queue fills up
   c. pull() stops being called
   d. Source stops producing

4. When writer's promise resolves:
   a. pipe resumes reading
   b. ReadableStream's queue drains
   c. desiredSize goes positive
   d. pull() is called again
   e. Source resumes producing
```

The beauty: backpressure propagates **automatically** through the
entire pipeline. No manual flow control needed.

### Manual Backpressure-Aware Writing

When you're not using `pipeTo`, you handle backpressure via
`writer.ready`:

```javascript
const writer = writableStream.getWriter();

for (const chunk of chunks) {
  // Wait for backpressure to clear before writing
  await writer.ready;
  writer.write(chunk); // note: NOT awaiting the write itself
}

// Wait for all writes to complete
await writer.close();
```

`writer.ready` is a promise that resolves when the writable stream's
internal queue has room. If the queue is full, `writer.ready` stays
pending until space opens.

---

## Reading from Streams

### Reader Mode (Exclusive Lock)

```javascript
const reader = stream.getReader();

try {
  while (true) {
    const { value, done } = await reader.read();
    if (done) break;
    process(value);
  }
} finally {
  reader.releaseLock(); // MUST release or stream is locked forever
}
```

`getReader()` **locks** the stream — no other consumer can read
from it until `releaseLock()` is called. This prevents interleaved
reads from corrupting chunk boundaries.

### BYOB Readers (Bring Your Own Buffer)

For byte-oriented streams, BYOB readers let you control allocation:

```javascript
const reader = stream.getReader({ mode: 'byob' });

const buffer = new ArrayBuffer(1024);
let view = new Uint8Array(buffer);

while (true) {
  const { value, done } = await reader.read(view);
  if (done) break;

  // value is a NEW view into the same (or transferred) buffer
  // The old view is detached — you can't use it anymore
  process(value);

  // Create a new view for the next read
  view = new Uint8Array(value.buffer);
}
```

BYOB readers avoid allocating new buffers for each chunk — critical
for high-throughput byte processing.

### Async Iteration (for-await-of)

ReadableStreams implement `Symbol.asyncIterator`:

```javascript
for await (const chunk of stream) {
  process(chunk);
}
```

This is sugar for the reader pattern above. When the loop exits
(normally or via break/throw), the reader is released and the
stream is cancelled.

**Implementation detail**: The async iterator acquires a reader via
`getReader()`, calls `reader.read()` for each `.next()`, and calls
`reader.releaseLock()` + `stream.cancel()` for `return()`.

---

## Piping — Connecting Streams

### pipeTo (Readable → Writable)

```javascript
await readableStream.pipeTo(writableStream);
// Returns a promise that resolves when piping completes
```

Options:
```javascript
await readableStream.pipeTo(writableStream, {
  preventClose: false,  // true: don't close writable when readable ends
  preventAbort: false,  // true: don't abort writable on readable error
  preventCancel: false, // true: don't cancel readable on writable error
  signal: abortController.signal // abort the pipe externally
});
```

`pipeTo` handles:
- Backpressure (reads only when writable has room)
- Error propagation (readable error → abort writable, writable error
  → cancel readable)
- Completion (readable closes → close writable)

### pipeThrough (Readable → Transform → Readable)

```javascript
const transformed = readableStream.pipeThrough(transformStream);
// Returns the readable side of the transform
```

Under the hood, `pipeThrough` does:
1. `readableStream.pipeTo(transformStream.writable)`
2. Returns `transformStream.readable`

The returned readable can be piped further or read directly.

### Building Pipelines

```javascript
// Decompress → Decode → Parse → Process
const objects = response.body
  .pipeThrough(new DecompressionStream('gzip'))
  .pipeThrough(new TextDecoderStream())
  .pipeThrough(new TransformStream({
    transform(chunk, controller) {
      // Parse newline-delimited JSON
      for (const line of chunk.split('\n')) {
        if (line.trim()) {
          controller.enqueue(JSON.parse(line));
        }
      }
    }
  }));

// Consume the pipeline
for await (const obj of objects) {
  await processObject(obj);
}
```

Backpressure flows backward through the entire pipeline. If
`processObject` is slow, it stops reading from `objects`, which
applies backpressure to the JSON parser, which applies backpressure
to the text decoder, which applies backpressure to the decompressor,
which applies backpressure to the network fetch.

---

## Built-In Transform Streams

### TextEncoderStream / TextDecoderStream

```javascript
// Encode strings to UTF-8 bytes
const encoded = stringStream.pipeThrough(new TextEncoderStream());

// Decode UTF-8 bytes to strings
const decoded = byteStream.pipeThrough(new TextDecoderStream('utf-8'));
// Handles multi-byte characters that span chunk boundaries correctly
```

### CompressionStream / DecompressionStream

```javascript
// Compress
const compressed = readable.pipeThrough(
  new CompressionStream('gzip') // or 'deflate', 'deflate-raw'
);

// Decompress
const decompressed = compressed.pipeThrough(
  new DecompressionStream('gzip')
);
```

These use the browser's native compression implementations —
significantly faster than JavaScript-based alternatives.

---

## Tee — Splitting a Stream

```javascript
const [stream1, stream2] = readableStream.tee();
// Both streams independently produce the same data
```

**Implementation**: Tee creates a single reader on the original
stream and two new ReadableStreams. Each read from the original is
enqueued into both branches.

**The memory pitfall**: If one consumer is faster than the other,
the slower consumer's branch buffers chunks waiting to be read:

```
Original:  chunk1, chunk2, chunk3, chunk4, chunk5, ...
                                            ↑ fast consumer is here
Branch 1 (fast):  [empty — all consumed]
Branch 2 (slow):  [chunk2, chunk3, chunk4]  ← buffering!
                    ↑ slow consumer is here
```

The buffered chunks cannot be garbage collected until the slow
consumer reads them. For large streams with a slow consumer, this
defeats the purpose of streaming.

**Fix**: If you don't need both consumers to proceed independently,
consider a TransformStream that multicasts instead:

```javascript
function multicast(readable, sinks) {
  const reader = readable.getReader();
  return (async () => {
    while (true) {
      const { value, done } = await reader.read();
      if (done) {
        for (const sink of sinks) sink.close();
        break;
      }
      for (const sink of sinks) sink.enqueue(structuredClone(value));
    }
  })();
}
```

---

## Streaming Fetch Patterns

### Streaming a Response Body

```javascript
const response = await fetch('/large-file');

// response.body is a ReadableStream<Uint8Array>
const reader = response.body.getReader();
let receivedBytes = 0;

while (true) {
  const { value, done } = await reader.read();
  if (done) break;
  receivedBytes += value.byteLength;
  updateProgress(receivedBytes / response.headers.get('Content-Length'));
  appendToDocument(value);
}
```

### Streaming a Request Body (Upload)

```javascript
const stream = new ReadableStream({
  async pull(controller) {
    const chunk = await getNextChunk();
    if (chunk === null) {
      controller.close();
    } else {
      controller.enqueue(chunk);
    }
  }
});

await fetch('/upload', {
  method: 'POST',
  body: stream, // ReadableStream as request body
  duplex: 'half', // required for streaming bodies
});
```

**The `duplex: 'half'` requirement**: Streaming uploads require
half-duplex mode. The browser sends the request body before reading
the response. Full-duplex (send and receive simultaneously) is not
supported by fetch.

### Server-Sent Events via Streams

```javascript
async function* streamSSE(url) {
  const response = await fetch(url);
  const reader = response.body
    .pipeThrough(new TextDecoderStream())
    .getReader();

  let buffer = '';
  while (true) {
    const { value, done } = await reader.read();
    if (done) break;

    buffer += value;
    const events = buffer.split('\n\n');
    buffer = events.pop(); // incomplete event goes back to buffer

    for (const event of events) {
      if (event.trim()) {
        const data = event
          .split('\n')
          .filter(line => line.startsWith('data: '))
          .map(line => line.slice(6))
          .join('\n');
        yield JSON.parse(data);
      }
    }
  }
}

// Usage:
for await (const event of streamSSE('/api/events')) {
  handleEvent(event);
}
```

### NDJSON Streaming

```javascript
function ndJsonStream(url) {
  return new ReadableStream({
    async start(controller) {
      const response = await fetch(url);
      const reader = response.body
        .pipeThrough(new TextDecoderStream())
        .getReader();

      let buffer = '';
      while (true) {
        const { value, done } = await reader.read();
        if (done) break;
        buffer += value;

        const lines = buffer.split('\n');
        buffer = lines.pop(); // keep incomplete line

        for (const line of lines) {
          if (line.trim()) {
            controller.enqueue(JSON.parse(line));
          }
        }
      }

      if (buffer.trim()) {
        controller.enqueue(JSON.parse(buffer));
      }

      controller.close();
    }
  });
}
```

---

## Node.js Streams vs WHATWG Streams

### The Two APIs

Node.js has **two** stream APIs:

1. **Node.js streams** (`stream.Readable`, `stream.Writable`) —
   the original API, event-based, Node-specific.
2. **WHATWG Streams** (`ReadableStream`, `WritableStream`) —
   the web standard, promise-based, cross-platform.

Node.js 16.5+ supports WHATWG Streams natively.

### Key Differences

| Feature           | Node.js Streams          | WHATWG Streams            |
|-------------------|--------------------------|---------------------------|
| Backpressure      | `pause()`/`resume()`, `highWaterMark` | `desiredSize`, `pull()` |
| Error handling    | `'error'` event          | Promise rejection         |
| Piping            | `.pipe(dest)`            | `.pipeTo(dest)`           |
| Composition       | `.pipe(transform).pipe(dest)` | `.pipeThrough(t).pipeTo(dest)` |
| Async iteration   | `for await (const x of readable)` | Same                 |
| BYOB reading      | Not supported            | `reader.read(view)`       |
| Cancellation      | `stream.destroy()`       | `stream.cancel()`         |
| Locking           | Not applicable           | Reader/writer locks       |
| Availability      | Node.js only             | Browsers + Node.js + Deno |

### Conversion

```javascript
// Node.js Readable → WHATWG ReadableStream
const { Readable } = require('stream');
const nodeStream = fs.createReadStream('file.txt');
const webStream = Readable.toWeb(nodeStream);

// WHATWG ReadableStream → Node.js Readable
const nodeStream2 = Readable.fromWeb(webStream);

// Similarly for writable:
const { Writable } = require('stream');
const webWritable = Writable.toWeb(nodeWritable);
const nodeWritable2 = Writable.fromWeb(webWritable);
```

---

## Custom Transform Patterns

### Chunking Transform

Split a stream into fixed-size chunks:

```javascript
function chunkTransform(chunkSize) {
  let buffer = new Uint8Array(0);

  return new TransformStream({
    transform(chunk, controller) {
      // Accumulate bytes
      const combined = new Uint8Array(buffer.length + chunk.length);
      combined.set(buffer);
      combined.set(chunk, buffer.length);
      buffer = combined;

      // Emit full chunks
      while (buffer.length >= chunkSize) {
        controller.enqueue(buffer.slice(0, chunkSize));
        buffer = buffer.slice(chunkSize);
      }
    },

    flush(controller) {
      // Emit remaining bytes
      if (buffer.length > 0) {
        controller.enqueue(buffer);
      }
    }
  });
}

// Usage: split into 64KB chunks
response.body
  .pipeThrough(chunkTransform(64 * 1024))
  .pipeTo(dest);
```

### Throttle Transform

Rate-limit data flow:

```javascript
function throttleTransform(bytesPerSecond) {
  let lastEmit = 0;

  return new TransformStream({
    async transform(chunk, controller) {
      const now = performance.now();
      const expectedDelay = (chunk.byteLength / bytesPerSecond) * 1000;
      const elapsed = now - lastEmit;

      if (elapsed < expectedDelay) {
        await new Promise(r => setTimeout(r, expectedDelay - elapsed));
      }

      controller.enqueue(chunk);
      lastEmit = performance.now();
    }
  });
}
```

### Filter Transform

Skip chunks that don't match a predicate:

```javascript
function filterTransform(predicate) {
  return new TransformStream({
    transform(chunk, controller) {
      if (predicate(chunk)) {
        controller.enqueue(chunk);
      }
      // Dropping chunks that don't match — no enqueue
    }
  });
}

// Filter out empty lines from a text stream:
textStream.pipeThrough(filterTransform(line => line.trim().length > 0));
```

### Batch Transform

Collect N items into arrays:

```javascript
function batchTransform(batchSize) {
  let batch = [];

  return new TransformStream({
    transform(chunk, controller) {
      batch.push(chunk);
      if (batch.length >= batchSize) {
        controller.enqueue(batch);
        batch = [];
      }
    },

    flush(controller) {
      if (batch.length > 0) {
        controller.enqueue(batch);
      }
    }
  });
}
```

---

## Error Handling in Streams

### Error Propagation in Pipes

```javascript
const readable = new ReadableStream({
  pull(controller) {
    // Source error
    controller.error(new Error('source failed'));
  }
});

const writable = new WritableStream({
  write(chunk) {
    // Sink error
    throw new Error('write failed');
  }
});

try {
  await readable.pipeTo(writable);
} catch (e) {
  // e is whichever error occurred first
  console.error(e.message);
}
```

Default error propagation with `pipeTo`:
- **Readable errors** → abort the writable (unless `preventAbort`)
- **Writable errors** → cancel the readable (unless `preventCancel`)
- **Readable closes** → close the writable (unless `preventClose`)

### Error in a Transform

```javascript
const transform = new TransformStream({
  transform(chunk, controller) {
    if (chunk === 'bad') {
      controller.error(new Error('bad chunk'));
      return;
    }
    controller.enqueue(chunk);
  }
});
```

When the transform errors:
- Its writable side becomes errored → upstream gets backpressure
  error → upstream cancels
- Its readable side becomes errored → downstream gets error →
  downstream cleans up

### Aborting a Pipe

```javascript
const controller = new AbortController();

readable.pipeTo(writable, { signal: controller.signal })
  .catch(e => {
    if (e.name === 'AbortError') {
      console.log('Pipe aborted');
    }
  });

// Abort the pipe externally:
controller.abort();
// → readable is cancelled, writable is aborted
```

---

## Performance Characteristics

### Chunk Size and Throughput

Chunk size has a dramatic effect on throughput:

```
Chunk size    Chunks/sec    Throughput     Overhead
64 bytes      500,000       32 MB/s        High (per-chunk cost dominates)
1 KB          200,000       200 MB/s       Moderate
64 KB         10,000        640 MB/s       Low
1 MB          1,000         1 GB/s         Minimal
```

Each chunk incurs overhead:
- Promise allocation for async read/write (~175ns)
- Queue enqueue/dequeue operations
- Potential GC pressure from short-lived Uint8Arrays

For maximum throughput, use larger chunks (64KB-1MB). For minimum
latency, use smaller chunks.

### Memory Usage

Streams memory = internal queue size × chunk size:

```javascript
// highWaterMark: 10, average chunk: 64KB
// Peak memory: 10 × 64KB = 640KB per stream in the pipeline

// A 4-stage pipeline:
// readable (640KB) + transform1 (640KB) + transform2 (640KB) + writable (640KB)
// Peak: ~2.5MB for streaming a multi-GB file
```

Compare to buffering everything: a 2GB file would need 2GB of memory.
Streams keep it constant regardless of total data size.

### Zero-Copy with Transferable Streams

```javascript
// Transfer a ReadableStream to a worker (zero-copy):
const stream = new ReadableStream(/* ... */);
worker.postMessage({ stream }, [stream]);
// The stream is TRANSFERRED — the main thread can no longer use it

// In the worker:
self.onmessage = async (event) => {
  const stream = event.data.stream;
  for await (const chunk of stream) {
    processInWorker(chunk);
  }
};
```

The underlying data isn't copied — the ownership is transferred.

---

## Key Takeaways

1. **Three stream types**: ReadableStream (source), WritableStream
   (sink), TransformStream (transform). Composable via
   `pipeThrough` and `pipeTo`.

2. **Backpressure is automatic** through pipes — slow consumers
   naturally slow down fast producers via `desiredSize` and `pull()`.

3. **Queuing strategies** control the high-water mark — use
   `CountQueuingStrategy` for object streams, `ByteLengthQueuingStrategy`
   for byte streams.

4. **Async iteration** works natively — `for await (const chunk of stream)`
   is the simplest way to consume a ReadableStream.

5. **`tee()` has a memory trap** — if one consumer is slower, the
   faster consumer's data buffers in the slower branch.

6. **Chunk size matters** — larger chunks = higher throughput but
   higher latency. 64KB is a good default for byte streams.

7. **Error propagation through pipes** follows the principle of
   least surprise — errors flow in both directions, resources are
   cleaned up automatically.

8. **Streams are transferable** — you can move a stream to a worker
   via `postMessage` without copying the data.
