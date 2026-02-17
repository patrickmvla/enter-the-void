# V8 Engine — How JavaScript Actually Executes

## What V8 Is

V8 is Google's JavaScript and WebAssembly engine. It compiles JavaScript
to machine code and executes it. Chrome, Node.js, Deno, and Cloudflare
Workers all run on V8.

V8 doesn't interpret JavaScript in the traditional sense — it uses a
multi-tier compilation pipeline that balances startup speed against
peak throughput. Understanding this pipeline explains why some JavaScript
patterns are fast and others aren't.

---

## The Compilation Pipeline

```
Source Code
    │
    ▼
┌─────────┐
│ Scanner  │  Tokenization (lazy for inner functions)
└────┬─────┘
     │
     ▼
┌─────────┐
│ Parser  │  AST generation (pre-parsing for unused functions)
└────┬─────┘
     │
     ▼
┌───────────┐
│ Ignition  │  Bytecode interpreter (baseline execution)
└────┬──────┘
     │  hot function detected
     ▼
┌────────────┐
│ Sparkplug  │  Non-optimizing compiler (fast native code)
└────┬───────┘
     │  still hot
     ▼
┌────────────┐
│ Maglev     │  Mid-tier optimizing compiler
└────┬───────┘
     │  very hot
     ▼
┌────────────┐
│ TurboFan   │  Full optimizing compiler (peak performance)
└────────────┘
```

### Scanning (Tokenization)

The scanner converts UTF-16 source text into tokens. Key optimization:
**lazy scanning**. V8 doesn't scan function bodies until they're called.

```javascript
function outer() {
  // Scanned immediately (top-level)
  function inner() {
    // NOT scanned until inner() is called
    // V8 just finds the matching } and skips
  }
}
```

This is why startup-heavy code benefits from wrapping in functions —
the scanner skips the body entirely until first invocation.

### Parsing

Two parsers:

1. **Pre-parser** — scans function body to find syntax errors and
   track variable scope, but doesn't build a full AST. Used for
   functions that aren't immediately invoked.

2. **Full parser** — builds the complete Abstract Syntax Tree.
   Only runs when a function is actually called.

```javascript
// Pre-parsed (not called at parse time):
function maybeUsed() { /* pre-parsed only */ }

// Fully parsed immediately (IIFE):
(function() { /* full parse */ })();

// Fully parsed (called at top level):
function definitelyUsed() { /* full parse */ }
definitelyUsed();
```

The parser produces an AST that Ignition then walks to generate bytecode.

**Eager vs lazy compilation** — V8 uses heuristics to decide. Functions
inside `()` are eagerly compiled (the IIFE pattern). Otherwise, V8
assumes the function might not be called and defers.

### Ignition — The Bytecode Interpreter

Ignition compiles the AST into a compact bytecode format. Each bytecode
instruction is a single byte opcode followed by operands.

Ignition uses a **register machine** (not a stack machine). It has a
set of virtual registers that map to the machine's stack frame.

```
Bytecode for: function add(a, b) { return a + b; }

   Ldar a1        ; Load register a1 (parameter b) into accumulator
   Add a0, [0]    ; Add register a0 (parameter a) to accumulator
                   ; [0] is the feedback vector slot
   Return         ; Return the accumulator value
```

Key design choices:
- **Accumulator-based** — most operations implicitly use an accumulator
  register, reducing bytecode size
- **Feedback vectors** — each operation that could benefit from type
  specialization gets a feedback slot that records the types it sees
- **Compact** — bytecodes are 1-3 bytes. A function's bytecode is
  typically 50-100x smaller than its AST

**Feedback vectors** are the bridge to optimization:

```
Feedback vector for add():
  Slot 0 (BinaryOp Add):
    - Seen: Smi + Smi → Smi    (small integer fast path)
    - State: MONOMORPHIC
```

When the same operation always sees the same types, V8 marks it
monomorphic. Two types = polymorphic. More = megamorphic. This
classification drives TurboFan's optimization decisions.

### Sparkplug — Non-Optimizing Compiler

Sparkplug is a single-pass compiler that walks Ignition bytecode and
generates machine code directly, without building an intermediate
representation. No register allocation, no optimization — just
one bytecode instruction → one machine code sequence.

Why it exists: Ignition has interpreter overhead (dispatch loop,
indirect jumps). Sparkplug eliminates that overhead with minimal
compilation cost (~1ms for a typical function).

Sparkplug uses the **same stack frame layout** as Ignition, so
deoptimization from higher tiers can fall back to Sparkplug code
without rebuilding the frame.

### Maglev — Mid-Tier Optimizing Compiler

Added in 2023. Maglev builds a simple SSA (Static Single Assignment)
graph from bytecode and applies lightweight optimizations:

- Type specialization based on feedback
- Bounds check elimination
- Simple inlining
- Dead code elimination

Maglev compiles 10-20x faster than TurboFan while capturing 70-80%
of TurboFan's performance. It fills the gap where Sparkplug is too
slow but TurboFan's compilation cost isn't justified yet.

### TurboFan — The Full Optimizing Compiler

TurboFan is V8's top-tier compiler. It builds a sea-of-nodes IR
(intermediate representation) and applies aggressive optimizations.

**Sea of nodes**: instead of a control flow graph with basic blocks,
TurboFan uses a graph where every operation is a node. Edges represent
both data flow and control flow. This representation enables
optimizations that are hard with traditional CFGs.

Optimization passes:

1. **Inlining** — small callees are merged into callers. This is
   the most impactful optimization because it enables everything else.

2. **Type specialization** — based on feedback vectors, generic
   operations become typed operations:
   ```
   GenericAdd(a, b)  →  Int32Add(a, b)  (when feedback says both are int32)
   ```

3. **Range analysis** — track possible integer ranges to eliminate
   overflow checks and bounds checks.

4. **Escape analysis** — if an object doesn't leave a function, allocate
   it on the stack instead of the heap (or eliminate it entirely).

5. **Load elimination** — remove redundant loads from the same object.

6. **Dead code elimination** — remove unreachable code.

7. **Loop peeling and unrolling** — optimize hot loops.

8. **Instruction selection** — map IR nodes to target machine instructions.

9. **Register allocation** — linear scan allocation.

10. **Code generation** — emit machine code with embedded metadata for
    GC and deoptimization.

TurboFan compilation is expensive — 10-100ms for a complex function.
This is why V8 only TurboFan-compiles genuinely hot code.

---

## Hidden Classes (Maps)

V8 stores JavaScript objects using **hidden classes**, internally called
**Maps** (not to be confused with the `Map` data structure).

JavaScript objects are dynamic — properties can be added or removed at
any time. Naively, every property access would require a dictionary
lookup. Hidden classes give objects a predictable shape that enables
fast access.

### How Hidden Classes Work

```javascript
function Point(x, y) {
  this.x = x;  // Transition: Map0 → Map1 (adds 'x' at offset 0)
  this.y = y;  // Transition: Map1 → Map2 (adds 'x' at offset 0, 'y' at offset 1)
}

let p1 = new Point(1, 2);  // Shape: Map2
let p2 = new Point(3, 4);  // Shape: Map2 (same!)
```

Hidden class structure (simplified):
```
Map2:
  - back_pointer: Map1
  - instance_size: 32 bytes
  - descriptors:
      'x' → offset 0, type: tagged
      'y' → offset 1, type: tagged
  - transitions:
      (none — this is a leaf)
```

Object memory layout:
```
┌──────────────────┐
│ Map pointer      │ → points to Map2
├──────────────────┤
│ Properties ptr   │ → empty (in-object properties)
├──────────────────┤
│ Elements ptr     │ → empty (no indexed properties)
├──────────────────┤
│ x: 1 (tagged)    │ offset 0 (in-object)
├──────────────────┤
│ y: 2 (tagged)    │ offset 1 (in-object)
└──────────────────┘
```

### Transition Chains

Hidden classes form transition trees:

```
Map0 (empty)
 │
 ├─ add 'x' → Map1 { x }
 │              │
 │              └─ add 'y' → Map2 { x, y }
 │
 └─ add 'y' → Map3 { y }
                │
                └─ add 'x' → Map4 { y, x }
```

Map2 and Map4 are **different hidden classes** even though they have
the same properties. The order of property addition matters.

```javascript
let a = {};
a.x = 1;
a.y = 2;  // Map2 { x, y }

let b = {};
b.y = 1;
b.x = 2;  // Map4 { y, x }  — DIFFERENT hidden class!
```

This is why initializing properties in a consistent order matters for
performance. Objects with different hidden classes can't share optimized
code.

### In-Object vs Out-of-Object Properties

V8 allocates a fixed number of in-object property slots when an object
is created (based on the constructor's historical behavior). Properties
that fit go directly in the object. Additional properties overflow to
a separate **properties backing store** (a dictionary or array).

In-object access: one pointer dereference (object → field).
Out-of-object access: two pointer dereferences (object → backing store → field).

### Numbered Properties (Elements)

Indexed properties (`obj[0]`, `obj[1]`) are stored in a separate
**elements backing store**. V8 chooses the storage strategy based on
the indices:

- **Packed SMI** — all elements are small integers, no holes
- **Packed Double** — all elements are doubles, no holes
- **Packed** — mixed types, no holes
- **Holey SMI/Double/Packed** — has holes (sparse)
- **Dictionary** — very sparse, switches to hash table

The elements kind **only transitions downward** (more general):

```
PACKED_SMI_ELEMENTS
    ↓
PACKED_DOUBLE_ELEMENTS
    ↓
PACKED_ELEMENTS
    ↓
HOLEY_SMI_ELEMENTS → HOLEY_DOUBLE_ELEMENTS → HOLEY_ELEMENTS
    ↓
DICTIONARY_ELEMENTS
```

Once an array becomes holey, it never goes back to packed. This is
why you should avoid creating sparse arrays:

```javascript
let a = [1, 2, 3];        // PACKED_SMI_ELEMENTS
a.push(4.5);               // PACKED_DOUBLE_ELEMENTS (can't go back to SMI)
a[100] = 5;                // HOLEY_DOUBLE_ELEMENTS (can't go back to packed)
```

---

## Inline Caching (ICs)

Inline caches are the mechanism that makes property access fast in
practice. Without ICs, every `obj.prop` would require:

1. Look up obj's hidden class
2. Search the hidden class for 'prop'
3. Find the offset
4. Load from that offset

With ICs, V8 caches the hidden class and offset at the call site:

### Monomorphic IC

```javascript
function getX(point) {
  return point.x;  // IC at this site
}

// First call: IC misses, does full lookup
// IC records: Map2 → offset 0
getX(p1);

// Second call: IC checks if p2 has Map2 → yes → direct load from offset 0
getX(p2);  // Fast path!
```

Machine code with monomorphic IC:
```asm
; Load point.x with monomorphic IC
cmp [point + kMapOffset], Map2    ; Is the map what we expect?
jne ic_miss                        ; If not, slow path
mov rax, [point + 24]             ; Direct load at known offset
```

### IC States

```
         type A     type A      type B
           │          │           │
UNINITIALIZED → MONOMORPHIC → POLYMORPHIC → MEGAMORPHIC
                   (1 type)    (2-4 types)   (5+ types)
```

- **Monomorphic**: one hidden class seen. Single comparison + direct load.
- **Polymorphic**: 2-4 hidden classes. Linear search through a small table.
- **Megamorphic**: 5+ hidden classes. Falls back to hash table lookup.

```javascript
function getX(obj) { return obj.x; }

// These make the IC polymorphic:
getX({ x: 1 });           // Shape A
getX({ x: 1, y: 2 });     // Shape B
getX({ x: 1, z: 3 });     // Shape C
getX({ x: 1, w: 4 });     // Shape D
getX({ x: 1, v: 5 });     // Shape E → now MEGAMORPHIC
```

Megamorphic ICs are significantly slower. This is why monomorphic code
(same shapes flowing through the same code paths) is important for
performance-critical code.

---

## Deoptimization

TurboFan compiles code based on **speculative assumptions** from
type feedback. When an assumption is violated at runtime, V8 must
**deoptimize** — discard the optimized machine code and fall back to
Ignition bytecode.

### How Deoptimization Works

1. TurboFan inserts **guard checks** in optimized code:
   ```
   ; Optimized for integer addition
   CheckMap r0, Map2              ; Guard: is this the expected type?
   Int32Add r0.x, r0.y           ; Speculative: assume integers
   ```

2. When a guard fails, V8:
   - Captures the current machine state (registers, stack)
   - Maps it back to the corresponding Ignition bytecode position
     using **deoptimization data** (a translation table embedded
     in the optimized code)
   - Reconstructs the Ignition stack frame
   - Continues execution in the interpreter

3. V8 records the deoptimization reason. After enough deopts, V8 may
   blacklist the function from future optimization.

### Common Deoptimization Triggers

```javascript
// 1. Type change
function add(a, b) { return a + b; }
for (let i = 0; i < 100000; i++) add(1, 2);  // Optimized for int32
add("hello", "world");  // DEOPT: wrong types

// 2. Hidden class change
function getX(p) { return p.x; }
let p = { x: 1 };
for (let i = 0; i < 100000; i++) getX(p);  // Optimized for Map_A
delete p.x;  // Changes hidden class → DEOPT

// 3. Out-of-bounds array access
function sum(arr) {
  let s = 0;
  for (let i = 0; i <= arr.length; i++) {  // Off-by-one: <= instead of <
    s += arr[i];  // arr[arr.length] is undefined → DEOPT
  }
  return s;
}

// 4. Arguments object leaks
function foo() {
  return [].slice.call(arguments);  // arguments escapes → DEOPT
}
```

### On-Stack Replacement (OSR)

What about long-running loops? V8 can't wait for the function to
return to switch between tiers. **OSR** replaces the currently
executing code mid-loop:

```javascript
function processMillionItems(arr) {
  let sum = 0;
  for (let i = 0; i < arr.length; i++) {
    sum += arr[i];  // After ~1000 iterations, V8 OSR-compiles this loop
    // Execution seamlessly switches from interpreter to optimized code
  }
  return sum;
}
```

OSR works by:
1. Identifying a back-edge (loop iteration point) where the switch can
   happen
2. Compiling the function with an entry point at that back-edge
3. Translating the current interpreter state to the optimized frame
4. Jumping to the optimized loop body

---

## Value Representation — Tagging

V8 uses **pointer tagging** to distinguish between pointers and
small integers (Smis) without extra memory:

```
64-bit value:

Smi (Small Integer):
┌────────────────────────────────────┬───┐
│        32-bit integer value        │ 0 │  LSB = 0 → Smi
└────────────────────────────────────┴───┘

Heap Pointer:
┌────────────────────────────────────┬───┐
│        Heap object address         │ 1 │  LSB = 1 → pointer
└────────────────────────────────────┴───┘
```

Smis are immediate values — no heap allocation, no garbage collection.
Arithmetic on Smis is just machine integer arithmetic (shift to remove
tag, operate, shift back).

**Pointer compression** (V8 v8.0+): In 64-bit builds, V8 uses 32-bit
compressed pointers. All heap objects live within a 4GB "cage". Only
the lower 32 bits are stored; the upper bits are reconstructed from
a base register.

```
Full pointer:    0x00007F8A_12345679  (tagged heap pointer)
Compressed:      0x12345679           (32-bit, saves 4 bytes per field)
Base register:   0x00007F8A_00000000  (constant for the isolate)
Decompress:      base | compressed    → original pointer
```

This cuts V8's heap memory usage by ~40%.

### HeapNumber

Numbers that don't fit in a Smi (doubles, integers > 2^31-1) are
allocated as **HeapNumber** objects on the heap:

```
HeapNumber:
┌──────────────┐
│ Map pointer  │  → HeapNumber map
├──────────────┤
│ float64 value│  8 bytes, IEEE 754
└──────────────┘
```

Every HeapNumber allocation costs a heap allocation + GC pressure.
This is why integer-only arithmetic (within Smi range) is faster
than floating point in V8.

---

## Garbage Collection

V8 uses a **generational** garbage collector based on the **weak
generational hypothesis**: most objects die young.

### Heap Layout

```
V8 Heap
├── Young Generation (Semi-Space)
│   ├── From-Space (active)     ~1-8 MB
│   └── To-Space (inactive)     ~1-8 MB
│
├── Old Generation
│   ├── Old Object Space         variable
│   ├── Code Space               compiled code
│   ├── Map Space                hidden classes
│   └── Large Object Space       objects > 256KB
│
└── External memory tracking (ArrayBuffers, etc.)
```

### Minor GC — Scavenger (Young Generation)

Uses **Cheney's semi-space copying algorithm**:

1. New objects are allocated in the **from-space** using a simple
   bump pointer (fastest possible allocation):
   ```
   allocate(size):
     ptr = allocation_pointer
     allocation_pointer += size
     if allocation_pointer > limit:
       trigger GC
     return ptr
   ```

2. When from-space fills up (~1-8MB), scavenge:
   - Walk all GC roots (stack, globals, handles)
   - Copy live objects from from-space → to-space
   - Update all pointers to the new locations
   - Swap from-space and to-space labels

3. Objects that survive two scavenges are **tenured** — copied to
   old generation.

Why semi-space works: allocation is O(1) (bump pointer). Collection
cost is proportional to **live** objects, not total heap. Since most
objects die young, very few need copying. A typical scavenge takes 1-5ms.

**Write barrier**: when old-generation code writes a pointer to a
young-generation object, V8 records this in a **remembered set** so
the scavenger can find cross-generation references without scanning
the entire old generation.

### Major GC — Mark-Compact (Old Generation)

For the old generation, V8 uses **mark-compact**:

**Mark phase** (tri-color marking):
```
WHITE: not yet visited (potentially garbage)
GREY:  visited, but references not yet scanned
BLACK: visited, all references scanned (definitely alive)

1. Mark roots as GREY
2. While GREY set is not empty:
     Pick a GREY object
     Mark it BLACK
     Mark all WHITE objects it references as GREY
3. All WHITE objects are garbage
```

**Compact phase**:
- Move surviving objects to eliminate fragmentation
- Update all pointers to new locations
- Reclaim freed pages

A full major GC on a large heap (100MB+) can take 100-200ms.
This is why V8 needs concurrent and incremental techniques.

### Orinoco — Concurrent and Incremental GC

V8's garbage collector (project Orinoco) minimizes pause times through
parallelism:

**Incremental marking**: instead of marking the entire heap at once,
V8 interleaves small marking steps (~1ms each) with JavaScript
execution:

```
JS ──▶ Mark(1ms) ──▶ JS ──▶ Mark(1ms) ──▶ JS ──▶ Mark(1ms) ──▶ Sweep
```

Problem: JavaScript can modify the object graph during incremental
marking. V8 uses a **write barrier** — when a BLACK object gets a
reference to a WHITE object, V8 marks the WHITE object GREY (Dijkstra's
barrier) so it won't be missed.

**Concurrent marking**: marking runs on a background thread
simultaneously with JavaScript execution on the main thread. The
write barrier ensures correctness.

**Concurrent sweeping**: reclaiming memory from dead objects happens
on background threads.

**Parallel compaction**: multiple threads compact the heap
simultaneously.

**Parallel scavenging**: the young-generation scavenger also uses
multiple threads.

Result: V8's GC pauses are typically 1-5ms for minor GC and 5-20ms
for major GC in well-behaved applications. The worst case (huge heap,
lots of live objects, fragmentation) can still cause 50-100ms pauses.

### GC-Hostile Patterns

```javascript
// 1. Excessive object allocation in hot loops
for (let i = 0; i < 1000000; i++) {
  process({ x: i, y: i * 2 });  // million short-lived objects
}
// Fix: reuse objects or use typed arrays

// 2. Closures capturing large scopes
function processData(hugeArray) {
  return function() {
    // This closure keeps hugeArray alive even if it only needs one value
    return hugeArray.length;
  };
}
// Fix: extract needed values before creating closure

// 3. Growing arrays past their backing store
let arr = [];
for (let i = 0; i < 1000000; i++) {
  arr.push(i);  // Multiple reallocations + copies
}
// Fix: pre-allocate with new Array(1000000) or use typed arrays

// 4. WeakRef/FinalizationRegistry abuse
// WeakRefs keep checking if targets are collected, adding GC overhead
```

---

## Optimizing for V8 — What Actually Matters

### Function Monomorphism

```javascript
// GOOD: monomorphic — one shape flows through
class Point {
  constructor(x, y) {
    this.x = x;
    this.y = y;
  }
}
function distance(p) { return Math.sqrt(p.x ** 2 + p.y ** 2); }
points.forEach(p => distance(p));  // All same hidden class

// BAD: megamorphic — many shapes
function distance(p) { return Math.sqrt(p.x ** 2 + p.y ** 2); }
distance({ x: 1, y: 2 });
distance({ x: 1, y: 2, z: 3 });
distance({ y: 2, x: 1 });  // Different property order!
```

### Avoid Type Changes

```javascript
// BAD: changes type mid-stream
let x = 42;        // Smi
x = 42.5;          // HeapNumber — type change!
x = "forty-two";   // String — another type change!

// GOOD: consistent types
let count = 0;     // Always Smi
count++;
count += 10;
```

### Use Typed Arrays for Numeric Work

```javascript
// Regular arrays: tagged values, type checks, bounds checks
let arr = [1.0, 2.0, 3.0];  // PACKED_DOUBLE_ELEMENTS

// Typed arrays: raw doubles, no tagging, no type checks
let arr = new Float64Array([1.0, 2.0, 3.0]);
// V8 generates tight machine code — comparable to C
```

### Avoid Arguments Object

```javascript
// BAD: arguments prevents optimization
function sum() {
  let total = 0;
  for (let i = 0; i < arguments.length; i++) {
    total += arguments[i];
  }
  return total;
}

// GOOD: rest parameters
function sum(...args) {
  let total = 0;
  for (let i = 0; i < args.length; i++) {
    total += args[i];
  }
  return total;
}
```

---

## V8 Isolates and Contexts

An **Isolate** is an independent instance of the V8 engine with its own
heap, GC, and compilation pipeline. Each Isolate is single-threaded —
JavaScript runs on exactly one thread per Isolate.

```
Process
├── Isolate A (Thread 1)
│   ├── Context 1 (tab/page A)
│   └── Context 2 (iframe in A)
│
└── Isolate B (Thread 2)
    └── Context 3 (tab B)
```

A **Context** is a JavaScript execution environment within an Isolate.
Each Context has its own global object, builtins, and scope. Contexts
within an Isolate share the heap (which enables cross-context references
for same-origin iframes).

**Cloudflare Workers** use V8 Isolates for multi-tenant isolation.
Each Worker gets its own Isolate with startup time ~5ms (vs ~200ms
for a full container). The Isolate provides memory isolation without
process-level overhead.

---

## Profiling V8

### Chrome DevTools

```javascript
// CPU profiling — shows time spent in each function
console.profile('my profile');
// ... code to profile ...
console.profileEnd('my profile');
```

### V8 Flags (Node.js)

```bash
# Print optimized/deoptimized functions
node --trace-opt --trace-deopt script.js

# Print bytecode for a specific function
node --print-bytecode --print-bytecode-filter=functionName script.js

# Print generated machine code
node --print-opt-code --print-opt-code-filter=functionName script.js

# Trace inline cache state changes
node --trace-ic script.js

# Trace GC events
node --trace-gc script.js
# Output: [12345:0x7f8b0c] 12.3 ms: Scavenge 4.2 (8.0) -> 2.1 (8.0) MB

# Detailed GC statistics
node --trace-gc --trace-gc-verbose script.js

# Print hidden class transitions
node --trace-maps script.js
```

### Reading --trace-opt Output

```
[marking 0x1234 <JSFunction add> for optimization]
[compiling method 0x1234 <JSFunction add> using TurboFan]
[completed optimizing 0x1234 <JSFunction add>]

# Or if deoptimized:
[deoptimizing (DEOPT eager): begin ... reason: wrong map]
```

---

## V8 vs Other Engines

| Feature | V8 (Chrome) | SpiderMonkey (Firefox) | JavaScriptCore (Safari) |
|---------|-------------|----------------------|----------------------|
| Baseline | Ignition (interpreter) | Baseline (compiler) | LLInt (interpreter) |
| Mid-tier | Sparkplug + Maglev | — | Baseline JIT |
| Top-tier | TurboFan | IonMonkey (Warp) | DFG → FTL (B3/Air) |
| GC | Generational + incremental | Generational + incremental | Generational + incremental |
| Unique | Pointer compression, OSR | Nursery strings | Concurrent JIT |

All three engines have converged on similar architectures: multi-tier
compilation, type feedback, generational GC. The differences are in
implementation details and heuristics.

---

## Key Takeaways

| Concept | Why It Matters |
|---------|---------------|
| Multi-tier compilation | Startup speed (interpreter) vs peak throughput (TurboFan) |
| Hidden classes | Property access is O(1) when shapes are consistent |
| Inline caching | Monomorphic sites are 10-100x faster than megamorphic |
| Type feedback | Speculation + deoptimization = fast common case, correct edge case |
| Smi tagging | Integer arithmetic avoids heap allocation entirely |
| Pointer compression | 40% heap reduction in 64-bit builds |
| Generational GC | Young gen bump allocation is essentially free |
| Concurrent marking | GC pauses reduced from 100ms+ to 5-20ms |
| Escape analysis | Hot objects may never touch the heap |
| Deoptimization | Wrong assumptions → fall back → re-optimize correctly |
