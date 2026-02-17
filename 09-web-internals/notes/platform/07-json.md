# JSON — The Format, The Parser, The Internals

## What JSON Actually Is

JSON (JavaScript Object Notation) is a text-based data interchange
format defined by RFC 8259. Despite the name, JSON is language-
independent — it's used everywhere: APIs, config files, databases,
message queues, log formats.

JSON is a **subset of JavaScript literal syntax**, but with critical
differences. Understanding the exact grammar, parsing internals,
performance characteristics, and edge cases is what separates
knowing JSON from mastering it.

---

## The Grammar — RFC 8259

The entire JSON grammar fits on one page. That's the point — JSON
is deliberately minimal.

### The Six Value Types

```
JSON-value = object / array / string / number / "true" / "false" / "null"
```

That's it. No dates, no binary, no undefined, no comments, no
trailing commas, no single quotes. Every extension people want
(JSON5, JSONC) exists because the spec intentionally excluded them.

### Formal Grammar (ABNF)

```
JSON-text = ws value ws

value = object / array / number / string
      / "true" / "false" / "null"

object = "{" ws "}" / "{" members "}"
members = member *( "," member )
member = ws string ws ":" value

array = "[" ws "]" / "[" elements "]"
elements = value *( ws "," ws value )

number = [ "-" ] int [ frac ] [ exp ]
int = "0" / ( digit1-9 *digit )
frac = "." 1*digit
exp = ("e" / "E") [ "+" / "-" ] 1*digit

string = quotation-mark *char quotation-mark
char = unescaped / escaped
escaped = "\" ( "\"" / "\\" / "/" / "b" / "f" / "n" / "r" / "t"
         / "u" 4HEXDIG )
unescaped = %x20-21 / %x23-5B / %x5D-10FFFF

ws = *( %x20 / %x09 / %x0A / %x0D )  ; space, tab, LF, CR
```

### What the Grammar Forbids

```javascript
// All of these are INVALID JSON:

{ key: "value" }           // Keys MUST be quoted strings
{ "key": 'value' }         // Single quotes not allowed
{ "key": undefined }       // undefined is not a JSON value
{ "key": NaN }             // NaN is not a JSON value
{ "key": Infinity }        // Infinity is not a JSON value
{ "key": 0x1F }            // Hex literals not allowed
{ "key": 0777 }            // Octal literals not allowed
{ "key": +1 }              // Leading + not allowed (- is)
{ "key": .5 }              // Leading decimal point not allowed
{ "key": 1. }              // Trailing decimal point not allowed
[1, 2, 3,]                 // Trailing commas not allowed
// comment                  // Comments not allowed
{ "a": 1, "a": 2 }        // Duplicate keys — technically valid per RFC,
                           // but behavior is "implementation-defined"
```

### Number Precision

JSON numbers are text. The spec says nothing about precision. But
in practice, JSON is consumed by IEEE 754 double-precision float
parsers, which means:

```
Maximum safe integer:  2^53 - 1 = 9007199254740991
Minimum safe integer: -(2^53 - 1)

Numbers outside this range lose precision:
  JSON:   9007199254740993
  Parsed: 9007199254740992  (rounded to nearest representable double)

This is why APIs return large IDs as strings:
  { "id": "9007199254740993" }   // Safe
  { "id": 9007199254740993 }     // Precision loss!
```

Twitter's API hit this — tweet IDs exceeded 2^53 around 2010.
They added `id_str` fields as a workaround that persists to this day.

### String Encoding

JSON text MUST be encoded in UTF-8 (RFC 8259 §8.1). Earlier specs
(RFC 4627) allowed UTF-16 and UTF-32, but RFC 8259 mandates UTF-8.

Strings in JSON use Unicode escape sequences for characters outside
the basic set:

```
"Hello"              → Hello
"Hello\nWorld"       → Hello<newline>World
"Price: \u20AC100"   → Price: €100
"\uD83D\uDE00"       → 😀 (surrogate pair for U+1F600)
```

**Surrogate pairs**: characters above U+FFFF (like emoji) require
two `\uXXXX` escapes in JSON — a high surrogate (D800-DBFF) followed
by a low surrogate (DC00-DFFF):

```
U+1F600 (😀):
  High surrogate: D83D
  Low surrogate:  DE00
  JSON: "\uD83D\uDE00"

Formula:
  codepoint = 0x10000 + (high - 0xD800) × 0x400 + (low - 0xDC00)
  0x1F600  = 0x10000 + (0xD83D - 0xD800) × 0x400 + (0xDE00 - 0xDC00)
           = 0x10000 + 0x3D × 0x400 + 0x200
           = 0x10000 + 0xF400 + 0x200
           = 0x1F600 ✓
```

**Lone surrogates** (a high without a low, or vice versa) are
technically valid in JSON strings per the grammar but produce invalid
Unicode. Well-formed JSON generators should never produce them.
`JSON.parse` in V8 accepts them (creates a string with unpaired
surrogates), but this can cause issues downstream.

### Control Characters

Characters U+0000 through U+001F MUST be escaped. You cannot embed
a raw newline, tab, or null byte in a JSON string:

```
Valid:    { "msg": "line1\nline2" }     (escaped newline)
Invalid:  { "msg": "line1
line2" }                                 (raw newline in string)
```

---

## Parsing JSON — From Bytes to Values

### The Parser Architecture

A JSON parser is a **recursive descent parser** — one of the simplest
parser architectures. The grammar is LL(1): you can decide which
production to use by looking at the next character.

```
                JSON bytes (UTF-8)
                     │
                     ▼
              ┌─────────────┐
              │  Tokenizer   │  Reads characters, produces tokens
              │  (Scanner)   │  STRING, NUMBER, TRUE, FALSE, NULL,
              └──────┬───────┘  LBRACE, RBRACE, LBRACKET, RBRACKET,
                     │          COLON, COMMA
                     ▼
              ┌─────────────┐
              │  Parser      │  Recursive descent over token stream
              │              │  Builds in-memory value tree
              └──────┬───────┘
                     │
                     ▼
              Native values (objects, arrays, strings, numbers)
```

### Implementing a JSON Parser

A complete, spec-compliant JSON parser from scratch:

```javascript
function parseJSON(input) {
  let pos = 0;

  function error(msg) {
    throw new SyntaxError(`${msg} at position ${pos}`);
  }

  function peek() { return input[pos]; }
  function advance() { return input[pos++]; }

  function skipWhitespace() {
    while (pos < input.length) {
      const ch = input[pos];
      if (ch === ' ' || ch === '\t' || ch === '\n' || ch === '\r') {
        pos++;
      } else {
        break;
      }
    }
  }

  function expect(ch) {
    if (advance() !== ch) error(`Expected '${ch}'`);
  }

  // ---- Value dispatch ----
  function parseValue() {
    skipWhitespace();
    const ch = peek();
    let result;

    if (ch === '"')      result = parseString();
    else if (ch === '{') result = parseObject();
    else if (ch === '[') result = parseArray();
    else if (ch === 't') result = parseLiteral('true', true);
    else if (ch === 'f') result = parseLiteral('false', false);
    else if (ch === 'n') result = parseLiteral('null', null);
    else if (ch === '-' || (ch >= '0' && ch <= '9')) result = parseNumber();
    else error(`Unexpected character '${ch}'`);

    skipWhitespace();
    return result;
  }

  // ---- Literals ----
  function parseLiteral(text, value) {
    for (let i = 0; i < text.length; i++) {
      if (advance() !== text[i]) error(`Expected '${text}'`);
    }
    return value;
  }

  // ---- Strings ----
  function parseString() {
    expect('"');
    let result = '';

    while (pos < input.length) {
      let ch = advance();

      if (ch === '"') return result;

      if (ch === '\\') {
        ch = advance();
        switch (ch) {
          case '"':  result += '"'; break;
          case '\\': result += '\\'; break;
          case '/':  result += '/'; break;
          case 'b':  result += '\b'; break;
          case 'f':  result += '\f'; break;
          case 'n':  result += '\n'; break;
          case 'r':  result += '\r'; break;
          case 't':  result += '\t'; break;
          case 'u': {
            const hex = input.slice(pos, pos + 4);
            pos += 4;
            const codepoint = parseInt(hex, 16);
            if (isNaN(codepoint)) error('Invalid unicode escape');

            // Handle surrogate pairs
            if (codepoint >= 0xD800 && codepoint <= 0xDBFF) {
              // High surrogate — expect low surrogate
              if (advance() !== '\\' || advance() !== 'u') {
                error('Expected low surrogate');
              }
              const lowHex = input.slice(pos, pos + 4);
              pos += 4;
              const low = parseInt(lowHex, 16);
              if (low < 0xDC00 || low > 0xDFFF) {
                error('Invalid low surrogate');
              }
              const full = 0x10000 + (codepoint - 0xD800) * 0x400 + (low - 0xDC00);
              result += String.fromCodePoint(full);
            } else {
              result += String.fromCharCode(codepoint);
            }
            break;
          }
          default: error(`Invalid escape '\\${ch}'`);
        }
      } else if (ch.charCodeAt(0) < 0x20) {
        error('Unescaped control character');
      } else {
        result += ch;
      }
    }
    error('Unterminated string');
  }

  // ---- Numbers ----
  function parseNumber() {
    const start = pos;

    if (peek() === '-') advance();

    // Integer part
    if (peek() === '0') {
      advance();
      // After leading 0, must not be followed by digit (no 077)
    } else if (peek() >= '1' && peek() <= '9') {
      advance();
      while (pos < input.length && peek() >= '0' && peek() <= '9') advance();
    } else {
      error('Expected digit');
    }

    // Fraction
    if (peek() === '.') {
      advance();
      if (!(peek() >= '0' && peek() <= '9')) error('Expected digit after decimal');
      while (pos < input.length && peek() >= '0' && peek() <= '9') advance();
    }

    // Exponent
    if (peek() === 'e' || peek() === 'E') {
      advance();
      if (peek() === '+' || peek() === '-') advance();
      if (!(peek() >= '0' && peek() <= '9')) error('Expected digit in exponent');
      while (pos < input.length && peek() >= '0' && peek() <= '9') advance();
    }

    const numStr = input.slice(start, pos);
    const num = Number(numStr);

    if (!isFinite(num)) error('Number out of range');
    return num;
  }

  // ---- Objects ----
  function parseObject() {
    expect('{');
    skipWhitespace();

    const obj = {};

    if (peek() === '}') { advance(); return obj; }

    while (true) {
      skipWhitespace();
      if (peek() !== '"') error('Expected string key');
      const key = parseString();

      skipWhitespace();
      expect(':');

      const value = parseValue();
      obj[key] = value;

      skipWhitespace();
      if (peek() === '}') { advance(); return obj; }
      expect(',');
    }
  }

  // ---- Arrays ----
  function parseArray() {
    expect('[');
    skipWhitespace();

    const arr = [];

    if (peek() === ']') { advance(); return arr; }

    while (true) {
      arr.push(parseValue());

      skipWhitespace();
      if (peek() === ']') { advance(); return arr; }
      expect(',');
    }
  }

  // ---- Entry point ----
  const result = parseValue();
  skipWhitespace();
  if (pos < input.length) error('Unexpected data after JSON value');
  return result;
}
```

This is ~150 lines for a complete, spec-compliant parser. The
simplicity of JSON's grammar is its greatest engineering achievement.

---

## V8's JSON.parse — How It Actually Works

V8 doesn't use a simple recursive descent parser. It has a
**highly optimized, template-specialized** JSON parser.

### The Fast Path

```
1. Scan for structure:
   V8 quickly scans the input to determine:
   - Is it a flat object (no nesting)?
   - Are all keys short ASCII strings?
   - Are all values simple types?

2. If the input is "simple enough":
   - Pre-compute the hidden class (Map) for the result object
   - Allocate the object with the right shape up front
   - Parse values directly into the pre-shaped object
   - Skip intermediate allocations

3. If the input is complex:
   - Fall back to general recursive parsing
   - Construct values bottom-up
```

### Why JSON.parse Can Beat Object Literals

```javascript
// This is SLOWER for large objects:
const data = { name: "Alice", age: 30, city: "NYC", ... };
// V8 must: parse JS, build AST, compile bytecode, execute bytecode,
// create object with hidden class transitions for each property

// This can be FASTER:
const data = JSON.parse('{"name":"Alice","age":30,"city":"NYC",...}');
// V8: scan bytes, build object with pre-computed shape
// No AST, no bytecode, no compilation
```

Why? JavaScript object literals go through the full JS compilation
pipeline. JSON.parse has a dedicated fast parser that skips all of
that. For large static data (>10KB), `JSON.parse` is often 1.5-2x
faster than a JS object literal.

This is the basis of the **JSON.parse optimization** in build tools:
webpack and others transform large static objects into `JSON.parse()`
calls in production bundles.

### String Internalization

When V8 parses JSON object keys, it **internalizes** (interns) them.
Internalized strings are deduplicated — if two keys have the same
content, they share the same heap object. This makes:

- Property lookup faster (pointer comparison instead of string
  comparison)
- Memory usage lower (shared key strings)
- Hidden class transitions faster (same string → same transition)

```javascript
// These two objects share internalized key strings:
const a = JSON.parse('{"name":"Alice"}');
const b = JSON.parse('{"name":"Bob"}');
// "name" is stored once in memory, both objects reference it
```

---

## JSON.stringify — Serialization Internals

### The Algorithm (per ECMA-262 §25.5.2)

```
JSON.stringify(value, replacer, space):

1. If replacer is a function:
   - Wrap value in {"": value}
   - Call replacer("", value) to get the root value
   - For each property, call replacer(key, value)

2. If replacer is an array:
   - Only include properties whose names are in the array

3. Serialize the value:
   a. If value has a toJSON() method, call it
   b. If value is null → "null"
   c. If boolean → "true" / "false"
   d. If string → quote and escape
   e. If number:
      - NaN or Infinity → "null"  (not the number!)
      - Otherwise → number string
   f. If BigInt → throw TypeError
   g. If object (not function):
      - If array → serialize elements
      - If object → serialize key-value pairs
   h. If function or symbol → return undefined (omitted)

4. If space is provided:
   - Insert newlines and indentation
```

### toJSON — The Serialization Hook

```javascript
const event = {
  name: 'Meeting',
  date: new Date('2024-01-15'),
  internal: 'secret',

  toJSON(key) {
    // key is the property name in the parent object
    // (empty string for root)
    return {
      name: this.name,
      date: this.date.toISOString(),
      // internal is omitted
    };
  }
};

JSON.stringify(event);
// '{"name":"Meeting","date":"2024-01-15T00:00:00.000Z"}'
```

`Date.prototype.toJSON` calls `toISOString()`. This is why dates
serialize as ISO strings, not as Date objects.

### The Replacer Function

```javascript
JSON.stringify(data, (key, value) => {
  // Called for every key-value pair, including the root

  if (key === '') return value;         // Root — always return
  if (typeof value === 'function') return undefined; // Skip functions
  if (key === 'password') return '***';  // Redact
  if (value instanceof Set) return [...value]; // Convert Sets
  return value;
});
```

The replacer is called **after** `toJSON` but **before** the type
check. This means the replacer sees the toJSON'd value, not the
original.

### Circular Reference Detection

```javascript
const a = {};
const b = { ref: a };
a.ref = b;

JSON.stringify(a);
// TypeError: Converting circular structure to JSON
```

V8 maintains a **stack** of objects currently being serialized. Before
serializing an object, it checks if the object is already on the stack.
If so, it throws. This is O(n) per nesting level in the worst case,
but circular structures are rare in practice.

### Performance Characteristics

```
JSON.stringify performance by value type:

  null, boolean, small int:    ~5-10ns per value
  string (short, ASCII):       ~20-50ns (quote + escape scan)
  string (with escapes):       ~50-200ns (must process each char)
  number (double):             ~30-60ns (double-to-string conversion)
  object (5 properties):       ~200-500ns
  array (10 elements):         ~100-300ns
  deeply nested (depth 20):    ~2-5μs
```

The hot path for JSON.stringify in V8 is the **fast string escape
check**: for short ASCII strings with no characters needing escape,
V8 uses SIMD (on supported platforms) to check 16 bytes at once.

---

## The Reviver — Transforming Parsed Values

```javascript
JSON.parse(text, reviver)
```

The reviver function is called bottom-up for every value:

```javascript
const data = JSON.parse('{"date":"2024-01-15T00:00:00.000Z","count":42}',
  (key, value) => {
    // Called for each key-value pair, bottom-up, then root
    if (key === 'date') return new Date(value);
    return value;
  }
);

data.date instanceof Date; // true
```

**Call order** — bottom-up, then root:

```javascript
JSON.parse('{"a":{"b":1},"c":[2,3]}', (key, value) => {
  console.log(key, value);
  return value;
});
// "b" 1
// "a" {b: 1}
// "0" 2
// "1" 3
// "c" [2, 3]
// "" {a: {b: 1}, c: [2, 3]}    ← root (key is empty string)
```

---

## Duplicate Keys

The RFC says: "The names within an object SHOULD be unique." SHOULD,
not MUST. Behavior with duplicates is implementation-defined.

```javascript
JSON.parse('{"a":1,"a":2}')
// In every major engine: { a: 2 }  (last value wins)
// But this is NOT guaranteed by the spec
```

Some parsers reject duplicates (Python's `json.loads` with
`strict=True`). Some take the first value. Some take the last.
**Never rely on duplicate key behavior across systems.**

This is a security issue — an attacker can send JSON with duplicate
keys that parse differently across services:

```json
{"admin":false,"admin":true}
```

A Java parser taking the first value sees `admin: false`.
A JavaScript parser taking the last sees `admin: true`.
If one service validates and another consumes, you have a bypass.

---

## Security Concerns

### Prototype Pollution

```javascript
// DANGEROUS: merging parsed JSON into objects
const payload = JSON.parse('{"__proto__":{"isAdmin":true}}');

const user = {};
Object.assign(user, payload);

// In older engines or naive merge implementations:
const anotherUser = {};
anotherUser.isAdmin; // true — prototype was polluted!
```

`JSON.parse` itself is safe — it creates plain objects without
prototype chains. But **merging** parsed JSON into existing objects
using naive deep-merge can pollute `Object.prototype`.

Defenses:
```javascript
// 1. Use Object.create(null) for merge targets
const safe = Object.create(null);

// 2. Filter dangerous keys
function safeMerge(target, source) {
  for (const key of Object.keys(source)) {
    if (key === '__proto__' || key === 'constructor' || key === 'prototype') {
      continue; // Skip dangerous keys
    }
    target[key] = source[key];
  }
}

// 3. Use Map instead of plain objects for untrusted data
const data = new Map(Object.entries(JSON.parse(untrusted)));
```

### JSON Injection

When embedding JSON in HTML, raw `</script>` or `<!--` in JSON
strings can break out of script tags:

```html
<script>
  const data = {"msg":"</script><script>alert(1)</script>"};
</script>
```

The browser's HTML parser sees `</script>` inside the JSON and
closes the script tag early. The remaining text becomes HTML.

Defense — escape U+2028, U+2029, `<`, and `>`:

```javascript
function safeJSONForHTML(obj) {
  return JSON.stringify(obj)
    .replace(/</g, '\\u003C')
    .replace(/>/g, '\\u003E')
    .replace(/\u2028/g, '\\u2028')
    .replace(/\u2029/g, '\\u2029');
}
```

### ReDoS via JSON Strings

JSON parsers that use regex for string validation can be vulnerable
to catastrophic backtracking:

```javascript
// Naive regex-based JSON string validator:
const stringRegex = /"([^"\\]|\\.)*"/;
// This is actually safe (no nested quantifiers)

// But a poorly written one:
const badRegex = /"(\\.|[^"\\])*?"/;
// With crafted input: "\"\"\"\"\"\"\"\"\"\"..." → exponential backtracking
```

Use spec-compliant parsers (`JSON.parse`), not regex.

---

## Streaming JSON Parsing

Standard `JSON.parse` requires the entire input in memory. For large
documents (100MB+), this is problematic.

### SAX-Style Streaming

```javascript
// Streaming parser emits events, never builds full tree
class StreamingJSONParser {
  // Conceptual API:
  onObjectStart()
  onObjectEnd()
  onArrayStart()
  onArrayEnd()
  onKey(key)
  onValue(value)
  onError(error)
}

// Usage pattern:
parser.onKey = (key) => {
  if (key === 'targetField') {
    parser.onValue = (value) => {
      processResult(value);
      parser.abort(); // Stop parsing early
    };
  }
};

parser.write(chunk1);
parser.write(chunk2);
parser.end();
```

### JSON Lines (NDJSON)

Instead of one giant JSON array, use newline-delimited JSON:

```
{"id":1,"name":"Alice","score":95}
{"id":2,"name":"Bob","score":87}
{"id":3,"name":"Charlie","score":92}
```

Each line is a complete JSON value. Benefits:
- Parse line by line (constant memory)
- Append without rewriting (just add a line)
- Stream over HTTP with chunked transfer encoding
- Grep-friendly (`grep "Alice" data.jsonl`)
- Each line failure is isolated (don't lose the whole file)

```javascript
// Streaming NDJSON parser:
async function* parseNDJSON(stream) {
  const reader = stream.getReader();
  const decoder = new TextDecoder();
  let buffer = '';

  while (true) {
    const { done, value } = await reader.read();
    if (done) break;

    buffer += decoder.decode(value, { stream: true });
    const lines = buffer.split('\n');
    buffer = lines.pop(); // Keep incomplete last line

    for (const line of lines) {
      if (line.trim()) {
        yield JSON.parse(line);
      }
    }
  }

  // Process remaining buffer
  if (buffer.trim()) {
    yield JSON.parse(buffer);
  }
}

// Usage with fetch:
const response = await fetch('/api/stream');
for await (const record of parseNDJSON(response.body)) {
  processRecord(record);
}
```

### JSON Path Streaming

Parse only the values at specific paths without building the full
tree. Libraries like `jsonstream` (Node.js) or `oboe.js` (browser)
implement this:

```javascript
// Only extract prices from a huge product catalog:
JsonStream.parse('products.*.price')
  .on('data', price => totals.push(price));
```

---

## Binary Alternatives

JSON is text-based and verbose. When that's a bottleneck, binary
formats offer better performance.

### Size and Speed Comparison

```
Format          Encode     Decode     Size (typical)
───────────     ────────   ────────   ──────────────
JSON            1x         1x         1x (baseline)
MessagePack     2-5x       2-5x       0.5-0.7x
CBOR            2-4x       2-4x       0.5-0.7x
Protocol Buffers 5-10x     5-10x      0.3-0.5x (schema required)
BSON            1-2x       1-2x       1.0-1.2x (larger than JSON!)
```

### MessagePack

Binary JSON. Same data model (maps, arrays, strings, numbers, booleans,
null) but binary-encoded:

```
JSON:        {"compact":true,"schema":0}
Bytes:       82 A7 63 6F 6D 70 61 63 74 C3 A6 73 63 68 65 6D 61 00
             │  │                       │  │                       │
             │  └─ "compact" (7 chars)  │  └─ "schema" (6 chars)  └─ 0
             └─ fixmap (2 entries)      └─ true

Size: JSON = 27 bytes, MessagePack = 18 bytes (33% smaller)
```

Type tags in MessagePack:

```
0x00-0x7F  → positive fixint (0-127)
0x80-0x8F  → fixmap (0-15 entries)
0x90-0x9F  → fixarray (0-15 elements)
0xA0-0xBF  → fixstr (0-31 bytes)
0xC0       → nil
0xC2       → false
0xC3       → true
0xCA       → float32
0xCB       → float64
0xCC-0xCF  → uint 8/16/32/64
0xD0-0xD3  → int 8/16/32/64
0xD9-0xDB  → str 8/16/32
0xDC-0xDD  → array 16/32
0xDE-0xDF  → map 16/32
0xE0-0xFF  → negative fixint (-32 to -1)
```

### CBOR (Concise Binary Object Representation)

RFC 8949. Similar to MessagePack but IETF-standardized with additional
features:

- Semantic tags (datetime, bignum, UUID, regex)
- Indefinite-length strings/arrays/maps (streaming)
- Half-precision floats (16-bit)
- Canonical encoding for deterministic serialization

Used in: WebAuthn (FIDO2), COSE (CBOR Object Signing and Encryption),
COAP (Constrained Application Protocol for IoT).

### BSON (Binary JSON)

MongoDB's format. Surprisingly, BSON is often **larger** than JSON
because it includes:
- Length prefixes for every element (enables skipping without parsing)
- Explicit type bytes for every value
- Extra types (ObjectId, Date, Binary, Decimal128, Regex)

```
BSON document:
┌──────────┬──────────────────────────────────────────┬────┐
│ int32    │ element list                              │ 0  │
│ (length) │ type(1) name(cstring) value(variable)... │    │
└──────────┴──────────────────────────────────────────┴────┘
```

BSON's advantage isn't size — it's **traversal without full parse**.
MongoDB can seek directly to a field by its offset.

### Protocol Buffers

Unlike JSON, protobufs require a **schema** (`.proto` file):

```protobuf
message Person {
  string name = 1;
  int32 age = 2;
  repeated string emails = 3;
}
```

Wire format uses field numbers instead of names:

```
Field 1 (string): 0A 05 41 6C 69 63 65     → "Alice"
Field 2 (varint): 10 1E                      → 30
Field 3 (string): 1A 11 61 6C 69 63 65 40... → "alice@..."
```

Much smaller than JSON (no key names in the data) and much faster
(no string matching for keys). But requires schema agreement between
producer and consumer.

---

## JSON Schema

JSON Schema (draft 2020-12) is a vocabulary for validating JSON
structure:

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "type": "object",
  "required": ["name", "age"],
  "properties": {
    "name": {
      "type": "string",
      "minLength": 1,
      "maxLength": 100
    },
    "age": {
      "type": "integer",
      "minimum": 0,
      "maximum": 150
    },
    "email": {
      "type": "string",
      "format": "email"
    },
    "tags": {
      "type": "array",
      "items": { "type": "string" },
      "uniqueItems": true
    }
  },
  "additionalProperties": false
}
```

### Schema Composition

```json
{
  "oneOf": [
    { "$ref": "#/$defs/Cat" },
    { "$ref": "#/$defs/Dog" }
  ],
  "$defs": {
    "Cat": {
      "type": "object",
      "properties": {
        "type": { "const": "cat" },
        "whiskerLength": { "type": "number" }
      },
      "required": ["type"]
    },
    "Dog": {
      "type": "object",
      "properties": {
        "type": { "const": "dog" },
        "breed": { "type": "string" }
      },
      "required": ["type"]
    }
  }
}
```

Composition keywords:
- `allOf` — must match ALL schemas (intersection)
- `anyOf` — must match AT LEAST ONE (union)
- `oneOf` — must match EXACTLY ONE (discriminated union)
- `not` — must NOT match

JSON Schema is how OpenAPI/Swagger defines request/response shapes.

---

## JSON Pointer and JSON Patch

### JSON Pointer (RFC 6901)

A string syntax for identifying a specific value within a JSON
document:

```
Document: {"foo":{"bar":[1,2,3]},"baz":"qux"}

Pointer         → Value
────────        ─────────
""              → the whole document
"/foo"          → {"bar":[1,2,3]}
"/foo/bar"      → [1,2,3]
"/foo/bar/0"    → 1
"/foo/bar/2"    → 3
"/baz"          → "qux"

Escaping:
  "~" is escaped as "~0"
  "/" is escaped as "~1"
  "/a~1b"  → the key "a/b"
  "/m~0n"  → the key "m~n"
```

### JSON Patch (RFC 6902)

A format for describing changes to a JSON document:

```json
[
  { "op": "add",     "path": "/name",     "value": "Alice" },
  { "op": "remove",  "path": "/temp" },
  { "op": "replace", "path": "/age",      "value": 31 },
  { "op": "move",    "from": "/old/path", "path": "/new/path" },
  { "op": "copy",    "from": "/source",   "path": "/dest" },
  { "op": "test",    "path": "/version",  "value": 2 }
]
```

`test` is critical — it's a precondition. If the test fails, the
entire patch is rejected. This enables **optimistic concurrency**:

```json
[
  { "op": "test", "path": "/version", "value": 5 },
  { "op": "replace", "path": "/data", "value": "new data" },
  { "op": "replace", "path": "/version", "value": 6 }
]
```

If someone else updated the document (version ≠ 5), the patch fails
atomically.

### JSON Merge Patch (RFC 7396)

Simpler than JSON Patch — just send the diff as a partial document:

```javascript
// Original:
{ "name": "Alice", "age": 30, "city": "NYC" }

// Merge Patch (set to null = delete):
{ "age": 31, "city": null, "email": "a@b.com" }

// Result:
{ "name": "Alice", "age": 31, "email": "a@b.com" }
// age updated, city deleted, email added, name unchanged
```

Limitation: you can't set a value TO null (null means delete). You
can't patch arrays (they're replaced wholesale).

---

## JSON in HTTP

### Content-Type

```http
Content-Type: application/json; charset=utf-8
```

`charset=utf-8` is technically redundant (RFC 8259 mandates UTF-8)
but commonly included for compatibility.

### Common Gotchas

```javascript
// 1. fetch() doesn't reject on HTTP errors
const response = await fetch('/api/data');
// response.ok might be false — check it!
const data = await response.json();

// 2. Empty response body
// 204 No Content has no body — calling .json() throws
if (response.status !== 204) {
  const data = await response.json();
}

// 3. Non-JSON error responses
// Server might return HTML error page for 500s
try {
  const data = await response.json();
} catch (e) {
  // Response wasn't valid JSON
  const text = await response.text();
}

// 4. Content-Type mismatch
// Some APIs return JSON with text/plain Content-Type
// fetch().json() doesn't check Content-Type — it just tries to parse
```

---

## JSON5, JSONC, and Extensions

### JSON5

Extends JSON for human-editable config files:

```json5
{
  // Comments are allowed
  name: 'Alice',           // Unquoted keys
  age: 30,                 // Trailing commas
  'phone-number': null,    // Single-quoted strings
  hex: 0xFF,               // Hex numbers
  infinity: Infinity,      // Infinity and NaN
  multiline: 'hello \
    world',                // Multi-line strings
}
```

Used by: some config files. **Never use for data interchange** —
most parsers don't support it.

### JSONC (JSON with Comments)

JSON but with `//` and `/* */` comments. Used by VS Code's
`settings.json` and `tsconfig.json`:

```jsonc
{
  // This is a comment
  "compilerOptions": {
    "target": "ES2020", /* inline comment */
    "strict": true,     // trailing comma allowed in some impls
  }
}
```

TypeScript's `tsconfig.json` is technically JSONC, not JSON. The
TypeScript compiler uses its own parser that strips comments before
JSON.parse.

---

## Performance Tips

### Minimize Serialized Size

```javascript
// Short keys save bytes at scale
{ "n": "Alice", "a": 30 }          // 24 bytes
{ "name": "Alice", "age": 30 }     // 30 bytes

// But readability matters more in practice.
// Use compression (gzip/brotli) instead of short keys.

// gzip is remarkably effective on JSON because of repeated keys:
// 1MB JSON with repeated structure → ~50-100KB gzipped (90-95% reduction)
```

### Avoid JSON.parse in Hot Loops

```javascript
// BAD: parse the same string repeatedly
for (const item of items) {
  const config = JSON.parse(configString); // Parses every iteration
}

// GOOD: parse once
const config = JSON.parse(configString);
for (const item of items) {
  use(config);
}
```

### structuredClone vs JSON Round-Trip

```javascript
// Old pattern for deep cloning:
const clone = JSON.parse(JSON.stringify(original));
// Problem: loses Date objects, undefined values, functions, Maps, Sets

// Modern alternative:
const clone = structuredClone(original);
// Preserves: Dates, RegExps, Maps, Sets, ArrayBuffers, circular refs
// Does NOT clone: functions, DOM nodes, symbols

// Performance: structuredClone is ~2-5x faster than JSON round-trip
// for complex objects (avoids string serialization entirely)
```

### Benchmarks: JSON Operations

```
Operation                         Time (1KB object)   Time (1MB object)
───────────────────────────       ─────────────────   ─────────────────
JSON.parse                        ~10-20μs            ~5-15ms
JSON.stringify                    ~5-15μs             ~3-10ms
JSON.parse + JSON.stringify       ~15-35μs            ~8-25ms
structuredClone                   ~3-8μs              ~2-8ms
MessagePack decode                ~3-8μs              ~1-5ms
MessagePack encode                ~2-6μs              ~1-4ms
```

---

## Key Takeaways

| Concept | Why It Matters |
|---------|---------------|
| Grammar is LL(1) | One lookahead character decides everything — parsing is trivial and fast |
| Numbers are IEEE 754 doubles | Integers > 2^53 lose precision — use strings for large IDs |
| UTF-8 only (RFC 8259) | Earlier RFCs allowed UTF-16/32 — modern JSON is always UTF-8 |
| Surrogate pairs in strings | Emoji and CJK Extension B require `\uXXXX\uXXXX` pairs |
| JSON.parse beats object literals | For large static data, JSON.parse skips the JS compilation pipeline |
| Duplicate keys are undefined | Different parsers handle them differently — security risk |
| Prototype pollution | Never merge untrusted JSON into objects without filtering `__proto__` |
| JSON in HTML needs escaping | `</script>` in JSON breaks script tags — escape `<` and `>` |
| NDJSON for streaming | One object per line enables line-by-line parsing with constant memory |
| Binary alternatives | MessagePack/CBOR: same model, 2-5x faster, 30-50% smaller |
| JSON Patch for atomic updates | `test` operation enables optimistic concurrency |
| structuredClone > JSON round-trip | Faster deep clone that preserves Dates, Maps, Sets, circular refs |
