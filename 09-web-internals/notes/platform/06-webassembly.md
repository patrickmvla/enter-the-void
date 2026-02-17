# WebAssembly — A Compilation Target for the Web

## What WebAssembly Is

WebAssembly (Wasm) is a binary instruction format for a stack-based
virtual machine. It's designed as a portable compilation target for
languages like C, C++, Rust, Go, and others — enabling near-native
execution speed in the browser.

Wasm is NOT a replacement for JavaScript. It's a complement:
- JavaScript: DOM manipulation, UI logic, glue code, quick scripting
- Wasm: compute-heavy workloads, codecs, game engines, crypto,
  image processing, scientific simulation

Key properties:
- **Binary format** — compact, fast to decode (~20x faster than JS parse)
- **Typed** — all operations have known types, no runtime type checks
- **Sandboxed** — runs in the same sandbox as JavaScript, no extra
  access to system resources
- **Deterministic** — same inputs produce same outputs (excepting NaN
  bit patterns and resource limits)

---

## Binary Format

A Wasm module is a sequence of bytes starting with the magic number
and version:

```
Byte offset  Content
───────────  ────────────────────
0x00         0x00 0x61 0x73 0x6D    Magic: \0asm
0x04         0x01 0x00 0x00 0x00    Version: 1

0x08         Section ID (1 byte)
0x09         Section length (LEB128)
0x0A+        Section payload
...          More sections
```

### Section Types

```
ID  Name         Purpose
──  ──────       ───────
 0  Custom       Debug info, names, source maps
 1  Type         Function signatures (param types → return types)
 2  Import       Imported functions, tables, memories, globals
 3  Function     Maps function index → type index
 4  Table        Indirect function call tables (funcref)
 5  Memory       Linear memory declarations
 6  Global       Global variable declarations
 7  Export       Functions, tables, memories, globals exposed to JS
 8  Start        Entry point function (runs on instantiation)
 9  Element      Initialize table contents
10  Code         Function bodies (bytecode)
11  Data         Initialize linear memory contents
12  DataCount    Count of data segments (for validation)
```

Sections must appear in order by ID (with Custom sections interleaved
anywhere). This ordering enables single-pass validation and streaming
compilation.

### LEB128 Encoding

Wasm uses LEB128 (Little Endian Base 128) for variable-length integers:

```
Value: 624485
Binary: 10011000 01100001 00000101

LEB128 encoding (7 bits per byte, high bit = continuation):
  0xE5 = 1 1100101    ← 7 bits of value, continuation bit = 1
  0x8E = 1 0001110    ← 7 bits of value, continuation bit = 1
  0x26 = 0 0100110    ← 7 bits of value, continuation bit = 0 (final)

Decode: 0100101 0001110 0100110 → 624485
```

LEB128 compresses small values (0-127 = 1 byte) while supporting
arbitrarily large integers.

---

## Type System

Wasm has four value types:

```
Type     Size    Description
───────  ──────  ──────────────────
i32      32-bit  Integer (also used for booleans, addresses)
i64      64-bit  Integer
f32      32-bit  IEEE 754 float
f64      64-bit  IEEE 754 double

v128     128-bit SIMD vector (Wasm SIMD proposal, shipped)
funcref           Function reference (for indirect calls)
externref         Opaque reference to host (JS) value
```

Wasm is **strongly typed** — every instruction specifies exact types.
There's no implicit coercion. An `i32.add` takes two i32s and produces
an i32. Period.

### Function Signatures

```wasm
;; Type section declares signatures:
(type $add_type (func (param i32 i32) (result i32)))

;; Function uses that type:
(func $add (type $add_type)
  local.get 0    ;; Push first param
  local.get 1    ;; Push second param
  i32.add        ;; Pop two i32s, push i32 result
)
```

---

## Instruction Set — Stack Machine

Wasm uses a **stack machine**. Instructions push and pop values from
an implicit operand stack:

```wasm
;; Compute (a + b) * c where a=3, b=4, c=5

i32.const 3      ;; Stack: [3]
i32.const 4      ;; Stack: [3, 4]
i32.add          ;; Stack: [7]
i32.const 5      ;; Stack: [7, 5]
i32.mul          ;; Stack: [35]
```

### Instruction Categories

**Numeric operations**:
```
i32.add, i32.sub, i32.mul, i32.div_s, i32.div_u
i32.rem_s, i32.rem_u (modulo)
i32.and, i32.or, i32.xor
i32.shl, i32.shr_s, i32.shr_u
i32.rotl, i32.rotr
i32.clz, i32.ctz, i32.popcnt
i32.eqz, i32.eq, i32.ne, i32.lt_s, i32.lt_u, ...

f64.add, f64.sub, f64.mul, f64.div
f64.sqrt, f64.ceil, f64.floor, f64.trunc, f64.nearest
f64.min, f64.max, f64.copysign
f64.abs, f64.neg
```

**Control flow** — structured, not goto:
```wasm
;; Block (label for br)
(block $label
  br $label        ;; Break out of block
)

;; Loop (label targets loop start)
(loop $loop
  ;; ... body ...
  br $loop          ;; Jump back to loop start
)

;; If/else
(if (i32.eqz (local.get $x))
  (then ...)
  (else ...)
)

;; Branch table (switch/case)
(br_table $case0 $case1 $case2 $default
  (local.get $index)
)
```

**No goto**: Wasm only has structured control flow (blocks, loops, if).
This is deliberate — it enables O(1) validation and efficient
compilation. Every branch target must be a lexically enclosing block.

**Memory operations**:
```wasm
i32.load offset=0 align=2    ;; Load i32 from memory[addr + offset]
i32.store offset=0 align=2   ;; Store i32 to memory[addr + offset]
i32.load8_s                   ;; Load byte, sign-extend to i32
i32.load8_u                   ;; Load byte, zero-extend to i32
i32.load16_s                  ;; Load 16 bits, sign-extend
memory.size                   ;; Current memory size in pages
memory.grow                   ;; Grow memory (returns old size, or -1)
```

---

## Linear Memory Model

Wasm memory is a **contiguous, byte-addressable, resizable array of
bytes**. It starts at index 0 and grows in units of 64KB **pages**.

```
Memory layout:
┌──────────────────────────────────────────────────┐
│ 0x00000                                          │
│ ┌──────────────┐                                 │
│ │ Static data  │  (initialized by Data section)  │
│ ├──────────────┤                                 │
│ │ Stack        │  (grows downward, managed by    │
│ │   ↓          │   compiled code, not Wasm VM)   │
│ ├──────────────┤                                 │
│ │ Heap         │  (grows upward via malloc/free  │
│ │   ↑          │   from the language runtime)    │
│ ├──────────────┤                                 │
│ │ Unused       │                                 │
│ └──────────────┘                                 │
│                                     0xNNNNN      │
└──────────────────────────────────────────────────┘
```

Key properties:
- **Bounds-checked**: every memory access is checked against the current
  memory size. Out-of-bounds = trap (not UB like C).
- **Isolated**: Wasm memory is a separate ArrayBuffer. The Wasm module
  can't access JavaScript's heap, the DOM, or system memory.
- **Growable**: `memory.grow` adds pages. Maximum can be specified at
  declaration (`(memory 1 256)` = initial 1 page, max 256 pages = 16MB).

### Memory from JavaScript

```javascript
// Create memory:
const memory = new WebAssembly.Memory({
  initial: 1,     // 1 page = 64KB
  maximum: 100,   // Max 100 pages = 6.4MB
  shared: false,   // true for SharedArrayBuffer (requires COOP/COEP)
});

// Access as typed array:
const view = new Uint8Array(memory.buffer);
view[0] = 42;

// Grow:
const oldPageCount = memory.grow(10);  // Add 10 pages
// IMPORTANT: memory.buffer may be detached after grow()
// Must re-create views: view = new Uint8Array(memory.buffer);
```

### String Passing

Wasm has no string type. Strings are bytes in linear memory:

```javascript
// JS → Wasm: write string into Wasm memory
function writeString(instance, str) {
  const memory = new Uint8Array(instance.exports.memory.buffer);
  const encoder = new TextEncoder();
  const encoded = encoder.encode(str);
  const ptr = instance.exports.malloc(encoded.length + 1);
  memory.set(encoded, ptr);
  memory[ptr + encoded.length] = 0;  // null terminator
  return ptr;
}

// Wasm → JS: read string from Wasm memory
function readString(instance, ptr) {
  const memory = new Uint8Array(instance.exports.memory.buffer);
  let end = ptr;
  while (memory[end] !== 0) end++;
  return new TextDecoder().decode(memory.slice(ptr, end));
}
```

This is the fundamental challenge of Wasm interop: complex data types
must be serialized/deserialized through linear memory.

---

## Instantiation and JavaScript Interop

### Loading a Module

```javascript
// Method 1: Fetch + instantiate (non-streaming)
const response = await fetch('module.wasm');
const bytes = await response.arrayBuffer();
const { instance, module } = await WebAssembly.instantiate(bytes, imports);

// Method 2: Streaming compilation (preferred — compiles while downloading)
const { instance, module } = await WebAssembly.instantiateStreaming(
  fetch('module.wasm'),
  imports
);
```

Streaming compilation is faster because the compiler processes bytes
as they arrive over the network, overlapping download and compilation.

### Import/Export

```javascript
// Import object — what JS provides to Wasm:
const imports = {
  env: {
    // Imported function:
    log: (value) => console.log('Wasm says:', value),

    // Imported memory:
    memory: new WebAssembly.Memory({ initial: 1 }),

    // Imported global:
    baseValue: new WebAssembly.Global({ value: 'i32', mutable: false }, 42),

    // Imported table:
    table: new WebAssembly.Table({ initial: 1, element: 'anyfunc' }),
  },
};

const { instance } = await WebAssembly.instantiateStreaming(
  fetch('module.wasm'),
  imports,
);

// Call exported function:
const result = instance.exports.add(3, 4);  // 7

// Access exported memory:
const view = new Uint8Array(instance.exports.memory.buffer);

// Access exported global:
const g = instance.exports.someGlobal;
console.log(g.value);
```

### Calling Conventions — The Cost of JS ↔ Wasm Calls

Crossing the JS/Wasm boundary has overhead:

```
Wasm → Wasm call:   ~1ns  (direct call, no marshaling)
JS → Wasm call:     ~10-50ns  (type checking, stack switching)
Wasm → JS call:     ~50-100ns  (exit sandbox, enter JS, re-enter sandbox)
JS → JS call:       ~5-20ns  (depends on optimization tier)
```

For tight loops calling between JS and Wasm, this overhead adds up.
Minimize boundary crossings — batch work inside Wasm.

---

## Compilation Internals

### Validation

Before compilation, the engine validates the module in a single pass:
- Type checking (every instruction's inputs/outputs match)
- Structured control flow (every block/loop is properly nested)
- Memory access alignment
- Function and type index bounds

Validation is O(n) in module size and O(1) in stack depth — the
structured control flow guarantee makes this possible.

### Baseline Compilation

Engines compile Wasm to native code in a single pass:

```
V8 (Liftoff):
  Wasm bytecode → machine code (no IR, no optimization)
  Speed: ~10-20MB/s of Wasm bytecode
  Purpose: fast startup, acceptable performance

SpiderMonkey (Baseline):
  Similar single-pass approach

JSC (BBQ — Build Bytecodes Quickly):
  Similar single-pass approach
```

### Optimizing Compilation (Background)

In parallel with baseline execution, the engine optimizes hot functions:

```
V8 (TurboFan for Wasm):
  Wasm bytecode → TurboFan IR → optimized machine code
  Speed: ~3-5MB/s (slower but better code)
  Optimizations: register allocation, instruction selection,
                 dead code elimination, constant folding

SpiderMonkey (IonMonkey/Cranelift):
  Wasm → MIR → optimized machine code

JSC (FTL via B3):
  Wasm → B3 IR → Air → machine code
```

Unlike JavaScript, Wasm doesn't need **type speculation** — all types
are known statically. This makes Wasm optimization simpler and more
predictable than JS optimization.

### Streaming Compilation

```
Network:    [chunk1] [chunk2] [chunk3] [chunk4] [chunk5]
Compile:         [c1]    [c2]    [c3]    [c4]    [c5]
Optimize:              [o1]         [o2]         [o3]
```

V8 compiles Wasm functions as soon as their bytes arrive. By the time
the last byte downloads, most of the module is already compiled.
This means large Wasm modules can start executing almost immediately
after download completes.

### Caching

V8 caches compiled Wasm modules in the HTTP cache. On subsequent
loads, it loads the already-compiled native code instead of
recompiling. This makes second-load startup nearly instant.

```
First load:  Download (200ms) + Compile (100ms) = 300ms
Second load: Load cached native code (5ms) = 5ms
```

---

## SIMD (Single Instruction, Multiple Data)

Wasm SIMD operates on 128-bit vectors, processing multiple values
in a single instruction:

```wasm
;; Add four f32 values at once:
v128.const f32x4 1.0 2.0 3.0 4.0
v128.const f32x4 5.0 6.0 7.0 8.0
f32x4.add
;; Result: f32x4 [6.0, 8.0, 10.0, 12.0]
```

SIMD types:
```
i8x16  — 16 × 8-bit integers
i16x8  — 8 × 16-bit integers
i32x4  — 4 × 32-bit integers
i64x2  — 2 × 64-bit integers
f32x4  — 4 × 32-bit floats
f64x2  — 2 × 64-bit doubles
```

SIMD maps to hardware instructions:
- x86: SSE/SSE2/SSE4.1/AVX
- ARM: NEON
- Wasm SIMD is a portable abstraction over these

Use cases: image processing, audio DSP, physics simulation, matrix
operations — anywhere data-parallel computation applies.

---

## WASI — WebAssembly System Interface

WASI standardizes how Wasm modules interact with the operating system
outside the browser. It provides POSIX-like capabilities:

```
WASI Capability          What It Provides
─────────────────        ────────────────
fd_read, fd_write        File I/O
path_open, path_create   Filesystem access
clock_time_get           System clock
random_get               Cryptographic random
args_get                 Command-line arguments
environ_get              Environment variables
sock_accept              Network sockets (preview)
```

### Capability-Based Security

WASI uses a **capability model** — a module can only access resources
it's explicitly granted:

```javascript
// Runtime grants specific directory access:
const wasi = new WASI({
  args: ['myapp', '--verbose'],
  env: { HOME: '/home/user' },
  preopens: {
    '/data': '/real/path/to/data',  // Module sees /data, maps to real path
    // Module CANNOT access any other filesystem path
  },
});
```

This is fundamentally different from POSIX, where a process can access
anything the user can. WASI modules are sandboxed by default and must
be explicitly granted each capability.

### Running Wasm Outside the Browser

```bash
# Wasmtime (Bytecode Alliance reference runtime):
wasmtime run --dir=/data module.wasm

# Wasmer:
wasmer run module.wasm --dir /data

# Node.js (experimental):
node --experimental-wasi-unstable-preview1 run.js
```

WASI enables Wasm as a universal binary format — write once, run
anywhere (browser, server, edge, embedded).

---

## When Wasm Beats JavaScript

### Where Wasm Wins

1. **Compute-bound work**: number crunching, codecs, simulation
   - V8's TurboFan generates excellent code for JS, but Wasm avoids
     the speculation/deoptimization overhead entirely
   - Typical speedup: 1.5-5x for compute-heavy code

2. **Predictable performance**: no GC pauses, no deoptimization,
   no JIT warmup
   - Wasm performance is consistent from the first call
   - Critical for real-time audio, games, video processing

3. **Existing codebases**: porting C/C++/Rust libraries
   - SQLite, ffmpeg, OpenCV, game engines
   - Rewrite in JS would take years; compile to Wasm takes days

4. **Large binary data processing**: image/video/audio manipulation
   - Wasm's linear memory avoids JS object overhead
   - Direct byte manipulation without TypedArray wrapper overhead

### Where JavaScript Wins

1. **DOM manipulation**: Wasm can't touch the DOM directly, must call
   through JS. DOM-heavy code should stay in JS.

2. **Small, irregular tasks**: the JS/Wasm call overhead makes small
   frequent calls expensive.

3. **String-heavy processing**: every string crossing the boundary
   must be copied through linear memory. If your workload is mostly
   string manipulation, JS is often faster.

4. **Startup time for small modules**: JS parses and runs immediately.
   Wasm needs compilation (even if fast). For tiny scripts, the
   compilation overhead isn't worth it.

### The Real-World Pattern

```
Typical Wasm-accelerated app:

  JavaScript                    WebAssembly
  ──────────                    ──────────────
  UI framework (React, etc.)    Image processing
  DOM manipulation              Video codec
  Event handling                Physics engine
  API calls                     Cryptography
  Routing                       Data compression
  State management              Audio DSP
       │                              │
       └──── JS ↔ Wasm boundary ──────┘
             (minimize crossings)
```

---

## Memory Management in Wasm

Wasm itself has no built-in allocator. The compiled language's runtime
provides memory management:

**C/Rust compiled to Wasm**: includes `malloc`/`free` (or Rust's
allocator) compiled into the Wasm module. The allocator manages the
linear memory heap.

**GC languages compiled to Wasm**: must include their GC in the Wasm
module. This is why Go and Java Wasm binaries are large — they include
their entire runtime.

**Wasm GC proposal** (shipped in Chrome 119, Firefox 120):
Lets Wasm use the engine's built-in GC for struct/array types:

```wasm
;; Wasm GC: engine-managed struct
(type $point (struct (field $x f64) (field $y f64)))
(func $newPoint (param $x f64) (param $y f64) (result (ref $point))
  struct.new $point (local.get $x) (local.get $y)
)
```

This enables compact Wasm binaries for GC languages (Kotlin, Dart,
Java) without bundling a custom GC.

---

## Emerging Proposals

| Proposal | Status | What It Enables |
|----------|--------|----------------|
| Exception handling | Shipped | try/catch without JS interop overhead |
| Tail calls | Shipped | Functional language efficiency |
| GC | Shipped | Engine-managed GC objects |
| Threads | Shipped | SharedArrayBuffer + atomics for Wasm |
| SIMD | Shipped | 128-bit vector operations |
| Relaxed SIMD | Shipped | FMA, reciprocal approximations |
| Memory64 | Phase 4 | >4GB address space |
| Component Model | Phase 2 | Module composition + interface types |
| Stack switching | Phase 2 | Coroutines, green threads, async |

The **Component Model** is the most significant upcoming change — it
defines how Wasm modules compose with each other and with host
environments through typed interfaces, solving the "everything is
bytes in linear memory" problem.

---

## Key Takeaways

| Concept | Why It Matters |
|---------|---------------|
| Binary format | 20x faster to decode than JS parse |
| Streaming compilation | Compile while downloading — near-instant start |
| Linear memory | Sandboxed byte array — safe, bounds-checked |
| No GC in core Wasm | Predictable performance, no pauses |
| Strong typing | No speculation needed — compiler knows all types |
| SIMD | 4x throughput for data-parallel workloads |
| JS/Wasm call overhead | Minimize boundary crossings for performance |
| WASI | Portable system interface — run Wasm anywhere |
| Capability security | Deny-by-default — explicit resource grants |
| Wasm GC proposal | Enables GC languages without bundled runtimes |
| Not a JS replacement | Best as a complement — compute in Wasm, UI in JS |
