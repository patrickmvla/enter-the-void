# Browser Rendering Pipeline — From HTML to Pixels

## The Critical Rendering Path

When the browser receives an HTML document, it passes through a
pipeline of stages before anything appears on screen:

```
HTML bytes
  │
  ▼
Parsing         → DOM (Document Object Model) tree
  │
  │ (CSS bytes also parsed in parallel)
  ▼
Style           → CSSOM (CSS Object Model) tree
  │
  ▼
Render Tree     → DOM + CSSOM combined (only visible nodes)
  │
  ▼
Layout          → Compute geometry (position, size of every box)
  │
  ▼
Paint           → Generate draw commands (fill rect, draw text, etc.)
  │
  ▼
Composite       → Combine layers, send to GPU, display on screen
```

Every stage matters for performance. Understanding what triggers what
is the difference between 60fps and jank.

---

## Parsing — Building the DOM

### HTML Parsing

The HTML parser is a **state machine** defined by the HTML spec. It
reads bytes, decodes them to characters, tokenizes into tags, and
builds the DOM tree.

```
Bytes:  3C 68 31 3E 48 69 3C 2F 68 31 3E
        <  h  1  >  H  i  <  /  h  1  >

Tokenizer states:
  Data state → sees '<' → Tag open state
  Tag open state → sees 'h' → Tag name state
  Tag name state → sees '>' → emit StartTag(h1) → Data state
  Data state → sees 'H' → emit Character('H')
  Data state → sees 'i' → emit Character('i')
  Data state → sees '<' → Tag open state
  Tag open state → sees '/' → End tag open state
  ...→ emit EndTag(h1)

Tree construction:
  StartTag(h1) → create Element node, push to stack
  Character('Hi') → create Text node, append to h1
  EndTag(h1) → pop h1 from stack

DOM tree:
  Document
    └── html
        ├── head
        └── body
            └── h1
                └── "Hi"
```

**The parser is forgiving.** Unlike XML, HTML parsing handles errors
gracefully. Missing close tags, misnested tags, and invalid markup
are corrected by the parser following spec-defined error recovery:

```html
<p>Hello<p>World    <!-- Two <p> tags, no close tags -->

DOM result:
  body
  ├── p → "Hello"   (first <p> implicitly closed by second <p>)
  └── p → "World"
```

### Parser-Blocking Scripts

When the parser encounters a `<script>` tag:

```
Parsing HTML... → <script src="app.js"> → STOP PARSING
  │
  ▼
Fetch app.js (network)
  │
  ▼
Execute app.js (may modify DOM via document.write)
  │
  ▼
Resume parsing HTML...
```

The parser stops because the script might call `document.write()`,
which inserts HTML into the token stream at the parser's current
position. The parser can't continue until the script finishes.

**Mitigations:**

```html
<!-- async: fetch in parallel, execute when ready (blocks parser briefly) -->
<script async src="analytics.js"></script>

<!-- defer: fetch in parallel, execute AFTER parsing completes, before DOMContentLoaded -->
<script defer src="app.js"></script>

<!-- module: deferred by default -->
<script type="module" src="app.mjs"></script>
```

```
Timeline:

               HTML parsing ──────────────────────────────────►
Normal:        ──── BLOCKED ──── script executes ──── resume ──►
async:         ──────────────── script executes ── resume ─────►
               (fetch in parallel ↑)
defer:         ────────────────────────────────── parse done ── script executes
               (fetch in parallel)
```

### Speculative Parsing (Preload Scanner)

While the main parser is blocked on a script, a **preload scanner**
continues scanning the HTML for resources to fetch:

```
Main parser blocked on app.js...

Preload scanner sees:
  <link rel="stylesheet" href="style.css">  → start fetching
  <script src="vendor.js">                  → start fetching
  <img src="hero.jpg">                      → start fetching

When app.js finishes, style.css and vendor.js may already be downloaded.
```

This speculative parsing is why resources load faster than expected
even with parser-blocking scripts. The preload scanner is a critical
browser optimization — without it, resources would only start
downloading when the main parser reaches them.

---

## CSS Parsing and the CSSOM

CSS is parsed into the **CSSOM** — a tree of style rules.

```css
body { font-size: 16px; }
.container { width: 80%; margin: 0 auto; }
.container h1 { color: blue; }
```

```
CSSOM tree:
  body
    font-size: 16px
    └── .container
        width: 80%
        margin: 0 auto
        └── h1
            color: blue
```

**CSS is render-blocking.** The browser won't render anything until
the CSSOM is complete. Reason: without all CSS, the browser might
render content with wrong styles and then "flash" when the correct
styles arrive (FOUC — Flash of Unstyled Content).

```
Timeline:
  HTML parsing:     ████████████████████████████
  CSS download:     ████████░░░░░░░░░░░░░░░░░░░░
  CSSOM built:              ████░░░░░░░░░░░░░░░░
  First render:                 ████ (waits for CSSOM)
```

**Optimization**: Put critical CSS inline in `<head>`, load non-critical
CSS asynchronously:

```html
<head>
  <!-- Critical CSS inline — no network request, CSSOM built immediately -->
  <style>
    body { margin: 0; font-family: sans-serif; }
    .header { background: #333; color: white; padding: 1rem; }
  </style>

  <!-- Non-critical CSS loaded asynchronously -->
  <link rel="preload" href="full.css" as="style" onload="this.rel='stylesheet'">
</head>
```

### Selector Matching

The browser matches CSS selectors **right to left**:

```css
.sidebar .nav ul li a { color: blue; }
```

```
Matching (right to left):
  1. Find all <a> elements in the document
  2. For each <a>: does it have an ancestor <li>?
  3. For each matching <li>: does it have an ancestor <ul>?
  4. For each matching <ul>: does it have an ancestor .nav?
  5. For each matching .nav: does it have an ancestor .sidebar?

Right-to-left is efficient because most selectors DON'T match.
Starting from the rightmost (most specific) part eliminates
non-matching elements immediately. If the document has 1000 elements
but only 5 <a> tags, we only check 5 elements instead of 1000.
```

**Selector performance** (from fastest to slowest):
```
#id                  → hash lookup, O(1)
.class               → hash lookup, O(1)
tag                  → hash lookup, O(1)
.parent .child       → requires ancestor traversal
*                    → matches everything (rarely a bottleneck in practice)
[attr="value"]       → attribute check per element
:nth-child(2n+1)     → requires sibling counting
```

In modern browsers, selector matching is extremely optimized (bloom
filters for ancestor matching, hash maps for class/id). CSS selector
performance is rarely the bottleneck — layout and paint are far more
expensive.

---

## The Render Tree

The render tree combines the DOM and CSSOM. Only **visible** nodes
are included:

```
DOM tree:                    CSSOM:                    Render tree:
  html                       body {color: black}       html
  ├── head                   h1 {font-size: 24px}      └── body
  │   ├── meta              .hidden {display:none}          ├── h1 "Hello"
  │   └── script                                           └── p "World"
  └── body
      ├── h1 "Hello"
      ├── p.hidden "Secret"  ← display:none → excluded from render tree
      └── p "World"
```

**Excluded from render tree:**
- `display: none` elements (not even box generated)
- Elements inside `<head>` (meta, script, link)
- `visibility: hidden` IS in the render tree (box exists, just invisible)
- `opacity: 0` IS in the render tree (box exists, just transparent)

The distinction matters: `display: none` doesn't participate in layout
(no space reserved). `visibility: hidden` and `opacity: 0` do
participate in layout (space reserved).

---

## Layout (Reflow)

Layout computes the **exact position and size** of every box in the
render tree. This is where the CSS box model is resolved into pixel
coordinates.

### The Box Model

```
┌─────────────────────────────────────────┐
│                 margin                   │
│   ┌─────────────────────────────────┐   │
│   │             border               │   │
│   │   ┌─────────────────────────┐   │   │
│   │   │         padding          │   │   │
│   │   │   ┌─────────────────┐   │   │   │
│   │   │   │     content      │   │   │   │
│   │   │   │  (width×height)  │   │   │   │
│   │   │   └─────────────────┘   │   │   │
│   │   └─────────────────────────┘   │   │
│   └─────────────────────────────────┘   │
└─────────────────────────────────────────┘

box-sizing: content-box (default):
  Total width = content + padding + border
  width: 200px, padding: 20px, border: 1px
  → Total: 200 + 40 + 2 = 242px

box-sizing: border-box:
  Total width = width (includes padding and border)
  width: 200px, padding: 20px, border: 1px
  → Content: 200 - 40 - 2 = 158px
  → Total: 200px
```

### Layout Algorithms

**Block layout** (default for `div`, `p`, `h1`):
```
Blocks stack vertically. Each block takes the full width of its
container. Height is determined by content (or explicit height).

┌────────────────────────┐
│ Block 1 (full width)   │
├────────────────────────┤
│ Block 2 (full width)   │
├────────────────────────┤
│ Block 3 (full width)   │
└────────────────────────┘
```

**Inline layout** (default for `span`, `a`, `em`):
```
Inline elements flow horizontally, wrapping to the next line
when the container width is exceeded. Height is determined by
line-height (not content height).

┌────────────────────────┐
│ [inline1] [inline2]    │
│ [inline3] [inline4]    │
└────────────────────────┘
```

**Flexbox layout:**
```
.container { display: flex; justify-content: space-between; }

Layout algorithm:
  1. Determine available space on main axis
  2. Calculate flex basis for each item (flex-basis or width/height)
  3. Distribute free space according to flex-grow
  4. Shrink items if total exceeds container (flex-shrink)
  5. Align items on cross axis (align-items)

Single-pass algorithm (mostly): O(n) for n flex items.
More complex with flex-wrap (multiple lines, requires re-distribution).
```

**Grid layout:**
```
.container {
  display: grid;
  grid-template-columns: 1fr 2fr 1fr;
  grid-template-rows: auto 1fr auto;
}

Layout algorithm:
  1. Resolve track sizes (fr units, minmax(), auto)
  2. Place items explicitly (grid-column, grid-row)
  3. Auto-place remaining items
  4. Resolve intrinsic sizes (auto tracks sized by content)
  5. Distribute remaining space to fr tracks

More complex than flexbox: 2D placement + sizing.
Grid layout can require multiple passes for intrinsic sizing.
```

### What Triggers Layout

Layout is expensive — the browser must recalculate positions and
sizes for potentially the entire tree. These properties/actions
trigger layout:

```
Properties that trigger layout (geometry changes):
  width, height, min-width, max-width, min-height, max-height
  padding, margin, border-width
  top, right, bottom, left (positioned elements)
  font-size, font-family, font-weight (affects text metrics)
  line-height, text-align, vertical-align
  display, position, float, clear
  overflow, flex-*, grid-*

JavaScript that forces synchronous layout (layout thrashing):
  element.offsetWidth       ← forces layout to return current value
  element.offsetHeight
  element.getBoundingClientRect()
  element.scrollTop
  getComputedStyle(element)
  window.innerWidth
```

**Layout thrashing** — the most common performance mistake:

```javascript
// BAD: read-write-read-write forces layout 100 times
for (const el of elements) {
    const width = el.offsetWidth;    // READ → forces layout
    el.style.width = width * 2 + 'px'; // WRITE → invalidates layout
}
// Each READ forces the browser to recalculate layout because
// the previous WRITE invalidated it. 100 elements = 100 layouts.

// GOOD: batch reads, then batch writes
const widths = elements.map(el => el.offsetWidth); // all READs first
elements.forEach((el, i) => {
    el.style.width = widths[i] * 2 + 'px';        // all WRITEs after
});
// One layout (for the reads), then browser batches the writes
// into a single layout before the next frame.
```

---

## Paint

After layout, the browser knows where every box is. Paint generates
**display lists** — ordered sequences of draw commands.

```
Display list for a blue button with text:
  1. DrawRect(x=100, y=200, w=120, h=40, color=#3366ff, borderRadius=4)
  2. DrawText(x=130, y=225, text="Click me", font=14px sans-serif, color=#ffffff)
  3. DrawBorder(x=100, y=200, w=120, h=40, color=#2255ee, width=1)
```

### What Triggers Paint (but not Layout)

```
Properties that trigger paint only:
  color, background-color, background-image
  border-color, border-style, border-radius
  box-shadow, text-shadow
  outline, visibility
  filter (some filters are composited — see below)
```

Changing `color` doesn't change geometry, so layout is skipped. But
the affected pixels must be repainted.

### Paint Layers

The browser doesn't paint everything onto one surface. It creates
**layers** that are painted independently:

```
Page:
  ┌─────────────────────────────────┐
  │ Background layer (main content)  │
  │                                  │
  │   ┌────────────────────┐        │
  │   │ Fixed header        │ ← own layer (position: fixed)
  │   └────────────────────┘        │
  │                                  │
  │   ┌──────────┐                  │
  │   │ Video     │ ← own layer (hardware-accelerated)
  │   └──────────┘                  │
  │                                  │
  │   ┌──────────────┐             │
  │   │ Animated div  │ ← own layer (will-change: transform)
  │   └──────────────┘             │
  └─────────────────────────────────┘
```

**Why layers matter**: When only one layer changes, only that layer
needs to be repainted. The rest are reused from their cached textures.

---

## Compositing

The final stage. Layers are uploaded to the GPU as textures and
composited (combined) into the final image.

```
GPU compositing:
  Layer 1 (background) → texture on GPU
  Layer 2 (fixed header) → texture on GPU
  Layer 3 (animated div) → texture on GPU

  Frame 1: Composite all layers → display
  Frame 2: Only layer 3 changed (transform: translateX(10px))
            → re-composite layer 3 at new position
            → layers 1 and 2 reused (no repaint, no re-upload)
            → display
```

### Compositor-Only Properties

These properties can be handled entirely by the compositor (GPU)
without triggering layout or paint on the main thread:

```
transform:    translate, rotate, scale, skew
opacity:      0 to 1
filter:       blur, brightness, etc. (when on a composited layer)
```

**This is why `transform: translateX()` is faster than `left:`**

```css
/* SLOW: triggers layout + paint + composite every frame */
.animated { left: 0; transition: left 0.3s; }
.animated.moved { left: 100px; }

/* FAST: triggers composite only (GPU handles it) */
.animated { transform: translateX(0); transition: transform 0.3s; }
.animated.moved { transform: translateX(100px); }
```

With `left`, every frame requires:
1. Layout (recalculate geometry of element and potentially siblings)
2. Paint (redraw affected pixels)
3. Composite

With `transform`, every frame requires:
1. Composite only (GPU moves the texture)

At 60fps, that's 16.67ms per frame. Layout + paint can easily take
5-10ms. Composite-only takes <1ms.

### Layer Promotion

The browser promotes elements to their own compositor layer when:

```
Automatic promotion:
  - position: fixed or sticky
  - will-change: transform, opacity, filter
  - transform: translate3d(), translateZ() (forces GPU layer)
  - <video>, <canvas>, <iframe>
  - Elements with CSS animations/transitions on compositor properties
  - Overlapping a composited element (implicit promotion to preserve
    z-order — this can cause "layer explosion")

Manual hint:
  will-change: transform;    /* tells browser to pre-promote */
  transform: translateZ(0);  /* hack to force promotion (avoid) */
```

**Layer explosion** — promoting too many layers:

```
Each layer consumes GPU memory (width × height × 4 bytes per layer).
A 1920×1080 layer = ~8 MB of GPU memory.
50 unnecessary layers = 400 MB GPU memory.

Common cause: an animated element overlaps other elements.
The browser promotes overlapping elements to preserve visual ordering.

Fix: use will-change sparingly. Remove it after animation completes.
```

---

## The Frame Budget

For 60fps (the standard target), each frame must complete in
**16.67ms** (1000ms / 60 = 16.67ms).

```
One frame (16.67ms budget):

  ┌──────────┬──────┬────────┬───────┬───────────┬──────────┐
  │JavaScript│Style │ Layout │ Paint │ Composite  │  Idle    │
  │  (event  │recalc│        │       │            │          │
  │ handlers,│      │        │       │            │          │
  │  rAF)    │      │        │       │            │          │
  └──────────┴──────┴────────┴───────┴───────────┴──────────┘
  0ms                                               16.67ms

If JavaScript takes 12ms:
  Only 4.67ms left for style, layout, paint, composite.
  If those exceed 4.67ms → frame dropped → jank.

If JavaScript takes 20ms:
  Frame missed entirely. Two frames shown for 33.34ms.
  User sees a "hitch" — the animation stutters.
```

### requestAnimationFrame

Schedule work to run at the right time in the frame lifecycle:

```javascript
// BAD: setTimeout doesn't align with frame boundaries
setTimeout(() => {
    element.style.transform = `translateX(${x}px)`;
}, 16); // might fire mid-frame, or skip frames

// GOOD: requestAnimationFrame fires just before style/layout/paint
requestAnimationFrame(() => {
    element.style.transform = `translateX(${x}px)`;
});
```

```
Frame lifecycle:

  1. Input events (click, scroll, keypress)
  2. requestAnimationFrame callbacks
  3. Style recalculation
  4. Layout
  5. Paint
  6. Composite
  7. requestIdleCallback (if time remaining)
```

`requestAnimationFrame` runs BEFORE style/layout/paint. This means
your DOM changes are applied in the same frame, and the browser
batches them with the rendering step.

### requestIdleCallback

Schedule low-priority work during idle time (after the frame's
rendering work is done, if there's time left):

```javascript
requestIdleCallback((deadline) => {
    // deadline.timeRemaining() tells you how many ms you have
    while (deadline.timeRemaining() > 1 && tasks.length > 0) {
        processTask(tasks.pop());
    }
    if (tasks.length > 0) {
        requestIdleCallback(processRemainingTasks);
    }
});
```

Use for: analytics, pre-fetching, non-critical DOM updates.
Never use for: visual updates (those should use rAF).

---

## Containment and Content-Visibility

### CSS Containment

Tell the browser that an element's internals don't affect the rest
of the page, enabling layout/paint optimizations:

```css
.card {
    contain: layout;    /* layout changes inside don't affect outside */
    contain: paint;     /* nothing paints outside this element's box */
    contain: size;      /* element's size doesn't depend on children */
    contain: style;     /* counters/quotes don't escape this subtree */
    contain: strict;    /* all of the above */
    contain: content;   /* layout + paint + style (not size) */
}
```

With `contain: layout`, when something inside `.card` changes, the
browser only recalculates layout for the card's subtree, not the
entire page.

### content-visibility: auto

The most impactful optimization for long pages:

```css
.section {
    content-visibility: auto;
    contain-intrinsic-size: 0 500px;  /* estimated height for scrollbar */
}
```

`content-visibility: auto` skips rendering (style, layout, paint)
for elements that are off-screen. When the user scrolls near them,
rendering kicks in.

```
Viewport:   ┌──────────────┐
            │ Section 1     │ ← fully rendered
            │ Section 2     │ ← fully rendered
            └──────────────┘
            │ Section 3     │ ← rendered (near viewport)
            │ Section 4     │ ← SKIP (off-screen, not rendered)
            │ Section 5     │ ← SKIP
            │ ...           │
            │ Section 100   │ ← SKIP

For a page with 100 sections, only 3 are rendered at any time.
Initial render is dramatically faster (97 sections skipped).
```

`contain-intrinsic-size` gives the browser an estimated size for
skipped elements so the scrollbar height is correct.

---

## Key Takeaways

| Concept | What You Must Know |
|---------|--------------------|
| Critical rendering path: DOM → CSSOM → Render Tree → Layout → Paint → Composite | CSS is render-blocking. Scripts are parser-blocking (unless async/defer) |
| Selectors matched right-to-left | Start from rightmost (most specific), prune early. ID/class are O(1) hash lookups |
| Preload scanner fetches resources while parser is blocked on scripts | Why resources load faster than expected despite parser-blocking |
| Layout is the most expensive stage | Triggered by geometry changes (width, height, margin, padding, font-size) |
| Layout thrashing: read-write-read-write forces repeated synchronous layouts | Batch all reads first, then all writes. Or use requestAnimationFrame |
| Compositor-only properties: transform, opacity, filter | GPU handles these without main-thread layout/paint. Use transform instead of left/top |
| will-change promotes to compositor layer | Use sparingly — each layer consumes GPU memory. Remove after animation |
| 60fps = 16.67ms per frame budget | JavaScript + style + layout + paint + composite must all fit |
| requestAnimationFrame fires before style/layout/paint | Use for visual updates. setTimeout doesn't align with frame boundaries |
| content-visibility: auto skips rendering for off-screen elements | Massive win for long pages. combine with contain-intrinsic-size for scrollbar accuracy |
