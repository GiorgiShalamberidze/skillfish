---
name: javascript-pro
description: Modern JavaScript mastery: ES2024+ features, module systems, event loop internals, Web APIs, memory management, bundler configuration, and browser compatibility.
---

# JavaScript Pro

Advanced JavaScript patterns and techniques for senior developers. Covers the modern language (ES2024+) in depth -- new syntax and built-ins, module systems, event loop internals, prototype mechanics, memory management, browser APIs, and bundler configuration. Use this skill when writing performance-sensitive code, debugging async behavior, architecting module boundaries, or configuring build tooling.

## Table of Contents

- [ES2024+ Features](#es2024-features)
- [Module Systems](#module-systems)
- [Event Loop and Async](#event-loop-and-async)
- [Prototypes and Classes](#prototypes-and-classes)
- [Memory Management](#memory-management)
- [Web APIs](#web-apis)
- [Bundler Configuration](#bundler-configuration)
- [Common Patterns Quick Reference](#common-patterns-quick-reference)

---

## ES2024+ Features

### structuredClone

Deep-copies objects including nested structures, `Date`, `RegExp`, `Map`, `Set`, `ArrayBuffer`, and circular references. Replaces the `JSON.parse(JSON.stringify(obj))` hack.

```javascript
const original = {
  date: new Date(),
  nested: { map: new Map([["key", "value"]]) },
  buffer: new ArrayBuffer(8),
};
original.self = original; // circular reference

const clone = structuredClone(original);
clone.nested.map.set("key", "changed");

console.log(original.nested.map.get("key")); // "value" -- unaffected
console.log(clone.self === clone);            // true -- circular ref preserved
```

**Limitations:** Cannot clone functions, DOM nodes, or `WeakRef`/`WeakMap`/`WeakSet`. Throws a `DataCloneError` for unsupported types.

```javascript
// Anti-pattern: JSON round-trip loses types and breaks on cycles
const broken = JSON.parse(JSON.stringify(original)); // throws on circular ref
```

### Object.groupBy and Map.groupBy

Group iterable elements by a classifier function. `Object.groupBy` returns a null-prototype object; `Map.groupBy` returns a `Map`.

```javascript
const inventory = [
  { name: "asparagus", type: "vegetable", quantity: 5 },
  { name: "banana",    type: "fruit",     quantity: 0 },
  { name: "cherry",    type: "fruit",     quantity: 12 },
  { name: "garlic",    type: "vegetable", quantity: 0 },
];

const byType = Object.groupBy(inventory, (item) => item.type);
// { vegetable: [{...asparagus}, {...garlic}], fruit: [{...banana}, {...cherry}] }

const byStock = Map.groupBy(inventory, (item) =>
  item.quantity > 0 ? "in-stock" : "out-of-stock"
);
// Map { "in-stock" => [...], "out-of-stock" => [...] }
```

```javascript
// Anti-pattern: manual reduce grouping
const grouped = inventory.reduce((acc, item) => {
  (acc[item.type] ??= []).push(item);
  return acc;
}, {}); // works but verbose and error-prone with prototype pollution
```

### Promise.withResolvers

Creates a `Promise` together with its `resolve` and `reject` functions, eliminating the executor callback pattern.

```javascript
function createDeferred() {
  const { promise, resolve, reject } = Promise.withResolvers();
  return { promise, resolve, reject };
}

// Real-world: wrapping a callback-based API
function loadImage(src) {
  const { promise, resolve, reject } = Promise.withResolvers();
  const img = new Image();
  img.onload = () => resolve(img);
  img.onerror = (e) => reject(e);
  img.src = src;
  return promise;
}

const img = await loadImage("/hero.webp");
```

```javascript
// Anti-pattern: leaking resolve/reject through outer scope with let
let resolve, reject;
const promise = new Promise((res, rej) => { resolve = res; reject = rej; });
```

### Temporal API

The `Temporal` API replaces `Date` with immutable, timezone-aware, calendar-aware types. Available behind a flag or via the `@js-temporal/polyfill` package.

```javascript
import { Temporal } from "@js-temporal/polyfill";

// Plain date (no time, no timezone)
const date = Temporal.PlainDate.from("2025-03-15");
const nextWeek = date.add({ days: 7 }); // 2025-03-22

// Zoned date-time -- handles DST correctly
const meeting = Temporal.ZonedDateTime.from({
  timeZone: "America/New_York",
  year: 2025, month: 3, day: 9, hour: 2, minute: 30,
});
console.log(meeting.toInstant().toString()); // resolves DST ambiguity

// Duration arithmetic
const duration = Temporal.Duration.from({ hours: 1, minutes: 45 });
const total = duration.total({ unit: "minutes" }); // 105

// Comparison
const a = Temporal.PlainDate.from("2025-01-01");
const b = Temporal.PlainDate.from("2025-06-15");
console.log(Temporal.PlainDate.compare(a, b)); // -1
```

```javascript
// Anti-pattern: Date arithmetic that ignores DST
const d = new Date("2025-03-09T02:30:00");
d.setDate(d.getDate() + 1); // may silently produce wrong results near DST
```

### Decorators (Stage 3)

TC39 decorators for classes, methods, accessors, and fields.

```javascript
function logged(target, context) {
  if (context.kind === "method") {
    return function (...args) {
      console.log(`Calling ${context.name} with`, args);
      const result = target.call(this, ...args);
      console.log(`${context.name} returned`, result);
      return result;
    };
  }
}

function bound(target, context) {
  if (context.kind === "method") {
    context.addInitializer(function () {
      this[context.name] = this[context.name].bind(this);
    });
  }
}

class Router {
  @logged
  @bound
  handleRequest(path) {
    return `Handled: ${path}`;
  }
}

const router = new Router();
const handler = router.handleRequest; // already bound
handler("/api/users"); // logs call + result
```

### Explicit Resource Management (using / Symbol.dispose)

Deterministic cleanup for resources like file handles, database connections, and locks.

```javascript
class DatabaseConnection {
  #connection;

  constructor(url) {
    this.#connection = connect(url);
  }

  query(sql) {
    return this.#connection.execute(sql);
  }

  [Symbol.dispose]() {
    this.#connection.close();
    console.log("Connection closed");
  }
}

// "using" ensures dispose is called when the block exits
{
  using db = new DatabaseConnection("postgres://localhost/mydb");
  const rows = db.query("SELECT * FROM users");
  // db[Symbol.dispose]() called automatically here
}

// Async variant with Symbol.asyncDispose
class FileWriter {
  #handle;

  static async open(path) {
    const writer = new FileWriter();
    writer.#handle = await fs.open(path, "w");
    return writer;
  }

  async write(data) {
    await this.#handle.write(data);
  }

  async [Symbol.asyncDispose]() {
    await this.#handle.close();
  }
}

{
  await using writer = await FileWriter.open("/tmp/log.txt");
  await writer.write("hello");
  // writer[Symbol.asyncDispose]() called automatically
}
```

```javascript
// Anti-pattern: try/finally boilerplate
const conn = connect(url);
try {
  conn.query("SELECT 1");
} finally {
  conn.close(); // easy to forget, verbose at scale
}
```

### Set Methods

Set operations that were previously impossible without manual iteration.

```javascript
const frontend = new Set(["alice", "bob", "carol"]);
const backend  = new Set(["bob", "carol", "dave"]);

frontend.union(backend);              // Set {"alice", "bob", "carol", "dave"}
frontend.intersection(backend);       // Set {"bob", "carol"}
frontend.difference(backend);         // Set {"alice"}
frontend.symmetricDifference(backend); // Set {"alice", "dave"}
frontend.isSubsetOf(backend);         // false
frontend.isSupersetOf(backend);       // false
frontend.isDisjointFrom(backend);     // false

// Works with any iterable argument
const arr = ["bob", "eve"];
frontend.intersection(arr); // Set {"bob"}
```

### Iterator Helpers

Lazy, chainable methods on iterators -- `map`, `filter`, `take`, `drop`, `flatMap`, `reduce`, `toArray`, `forEach`, `some`, `every`, `find`.

```javascript
function* naturals() {
  let i = 1;
  while (true) yield i++;
}

const result = naturals()
  .filter((n) => n % 2 === 0)  // even numbers
  .map((n) => n ** 2)           // squared
  .take(5)                      // first 5
  .toArray();                   // [4, 16, 36, 64, 100]

// Works with any iterator
const entries = new Map([["a", 1], ["b", 2], ["c", 3]]);
const keys = entries.keys()
  .filter((k) => k !== "b")
  .toArray(); // ["a", "c"]

// Lazy evaluation -- processes only what's needed
const first = naturals()
  .map((n) => { console.log(`processing ${n}`); return n * 10; })
  .find((n) => n > 30);
// logs: processing 1, processing 2, processing 3, processing 4
// first === 40
```

```javascript
// Anti-pattern: eagerly converting to array just to use array methods
const arr = [...hugeIterator]; // loads everything into memory
const filtered = arr.filter(predicate); // then filters
```

---

## Module Systems

### ESM vs CommonJS

| Feature | ESM (`import`/`export`) | CommonJS (`require`/`module.exports`) |
|---|---|---|
| Loading | Static (parse-time) | Dynamic (runtime) |
| Top-level `await` | Supported | Not supported |
| Tree shaking | Yes (static analysis) | No (dynamic shape) |
| `this` at top level | `undefined` | `module.exports` |
| File extension | `.mjs` or `"type": "module"` | `.cjs` or `"type": "commonjs"` |
| Circular deps | Live bindings (updates visible) | Snapshot at require time (stale) |

```javascript
// ESM -- named and default exports
export function add(a, b) { return a + b; }
export default class Calculator { /* ... */ }

// ESM -- namespace import
import * as math from "./math.js";
math.add(1, 2);

// CommonJS
const { add } = require("./math.js");
module.exports = { add };
```

### Dynamic import()

Load modules at runtime for code splitting, conditional loading, and lazy initialization.

```javascript
// Conditional loading
const locale = navigator.language.startsWith("fr") ? "fr" : "en";
const messages = await import(`./i18n/${locale}.js`);

// Lazy loading a heavy dependency
async function renderChart(data) {
  const { Chart } = await import("chart.js");
  return new Chart(canvas, { type: "bar", data });
}

// Dynamic import with error handling
try {
  const mod = await import("./optional-feature.js");
  mod.init();
} catch (e) {
  console.warn("Optional feature unavailable:", e.message);
}
```

```javascript
// Anti-pattern: dynamic require in ESM (not supported)
const mod = require("./something"); // SyntaxError in ESM context
```

### Import Maps

Browser-native module resolution without a bundler.

```html
<script type="importmap">
{
  "imports": {
    "lodash-es": "https://cdn.jsdelivr.net/npm/lodash-es@4/lodash.js",
    "react": "https://esm.sh/react@18",
    "react-dom": "https://esm.sh/react-dom@18",
    "@/utils/": "./src/utils/"
  },
  "scopes": {
    "./vendor/": {
      "lodash-es": "https://cdn.jsdelivr.net/npm/lodash-es@3/lodash.js"
    }
  }
}
</script>

<script type="module">
  import { debounce } from "lodash-es";
  import { formatDate } from "@/utils/date.js";
</script>
```

### package.json exports Field

Control the public API surface and conditional resolution of your package.

```json
{
  "name": "my-library",
  "type": "module",
  "exports": {
    ".": {
      "import": "./dist/esm/index.js",
      "require": "./dist/cjs/index.cjs",
      "types": "./dist/types/index.d.ts"
    },
    "./utils": {
      "import": "./dist/esm/utils.js",
      "require": "./dist/cjs/utils.cjs"
    },
    "./package.json": "./package.json"
  },
  "files": ["dist"]
}
```

Subpath patterns for many files:

```json
{
  "exports": {
    "./components/*": "./dist/components/*/index.js",
    "./hooks/*": "./dist/hooks/*.js"
  }
}
```

### Dual CJS/ESM Package

Build a package that works in both module systems.

```javascript
// src/index.js (ESM source)
export function greet(name) { return `Hello, ${name}`; }

// scripts/build-cjs.js -- generate CJS wrapper
// dist/cjs/index.cjs
// const { greet } = await import("../esm/index.js");
// module.exports = { greet };
```

The CJS wrapper re-exports from the ESM source. This avoids the "dual package hazard" where two copies of state exist.

```json
{
  "exports": {
    ".": {
      "import": "./dist/esm/index.js",
      "require": "./dist/cjs/index.cjs"
    }
  }
}
```

### Tree Shaking

Tree shaking eliminates dead code during bundling. It depends on static ESM `import`/`export` analysis.

```javascript
// GOOD: named exports -- bundler can drop unused ones
export function used() { return 42; }
export function unused() { return 0; } // eliminated if never imported

// BAD: barrel re-exports defeat tree shaking in some bundlers
// index.js
export * from "./moduleA.js";
export * from "./moduleB.js";
// importing one function may pull in everything
```

Mark your package as side-effect free:

```json
{
  "sideEffects": false
}
```

Or declare specific files with side effects:

```json
{
  "sideEffects": ["./src/polyfills.js", "*.css"]
}
```

```javascript
// Anti-pattern: default export of an object -- cannot be tree-shaken
export default { used, unused }; // bundler must keep the entire object
```

---

## Event Loop and Async

### Microtasks vs Macrotasks

The event loop processes all microtasks before moving to the next macrotask. Understanding this ordering is critical for debugging timing issues.

| Queue | Examples | Priority |
|---|---|---|
| Microtasks | `Promise.then`, `queueMicrotask`, `MutationObserver` | Highest -- drains completely before next macrotask |
| Macrotasks | `setTimeout`, `setInterval`, `I/O`, `MessageChannel` | Lower -- one per event loop iteration |
| Animation | `requestAnimationFrame` | Runs before repaint, after microtasks |
| Idle | `requestIdleCallback` | Lowest -- when browser is idle |

```javascript
console.log("1: sync");

setTimeout(() => console.log("5: macrotask"), 0);

Promise.resolve().then(() => console.log("3: microtask 1"));

queueMicrotask(() => console.log("4: microtask 2"));

console.log("2: sync");

// Output: 1, 2, 3, 4, 5
```

### queueMicrotask

Schedule work that must run before the browser renders but after the current synchronous code.

```javascript
// Batching DOM reads/writes
const pendingUpdates = [];

function scheduleUpdate(el, value) {
  pendingUpdates.push({ el, value });
  if (pendingUpdates.length === 1) {
    queueMicrotask(flushUpdates);
  }
}

function flushUpdates() {
  // All reads first (avoid layout thrashing)
  const rects = pendingUpdates.map(({ el }) => el.getBoundingClientRect());
  // Then all writes
  pendingUpdates.forEach(({ el, value }, i) => {
    el.style.transform = `translateX(${rects[i].width + value}px)`;
  });
  pendingUpdates.length = 0;
}
```

```javascript
// Anti-pattern: recursive microtask starvation
function bad() {
  queueMicrotask(bad); // blocks macrotasks and rendering forever
}
```

### Promise Patterns

```javascript
// Promise.allSettled -- wait for all, regardless of rejection
const results = await Promise.allSettled([
  fetch("/api/users"),
  fetch("/api/posts"),
  fetch("/api/comments"),
]);

const successes = results
  .filter((r) => r.status === "fulfilled")
  .map((r) => r.value);
const failures = results
  .filter((r) => r.status === "rejected")
  .map((r) => r.reason);

// Promise.any -- first to fulfill wins (ignores rejections)
const fastest = await Promise.any([
  fetch("https://cdn1.example.com/data.json"),
  fetch("https://cdn2.example.com/data.json"),
]);

// Promise.race -- first to settle wins (fulfill or reject)
const result = await Promise.race([
  fetch("/api/slow-endpoint"),
  new Promise((_, reject) =>
    setTimeout(() => reject(new Error("Timeout")), 5000)
  ),
]);
```

### Async Generators

```javascript
async function* paginate(url) {
  let nextUrl = url;
  while (nextUrl) {
    const response = await fetch(nextUrl);
    const data = await response.json();
    yield data.items;
    nextUrl = data.nextPage ?? null;
  }
}

// Consume with for-await-of
for await (const page of paginate("/api/users?page=1")) {
  for (const user of page) {
    console.log(user.name);
  }
}

// Transform with async generator composition
async function* take(source, n) {
  let count = 0;
  for await (const item of source) {
    yield item;
    if (++count >= n) return;
  }
}
```

### AbortController

Cancel fetch requests, event listeners, and any async operation.

```javascript
// Cancel a fetch after timeout
const controller = new AbortController();
const timeoutId = setTimeout(() => controller.abort(), 5000);

try {
  const response = await fetch("/api/data", { signal: controller.signal });
  clearTimeout(timeoutId);
  return await response.json();
} catch (err) {
  if (err.name === "AbortError") {
    console.log("Request was cancelled");
  } else {
    throw err;
  }
}

// Compose multiple abort signals
const userCancel = new AbortController();
const timeout = AbortSignal.timeout(10_000);
const combined = AbortSignal.any([userCancel.signal, timeout]);

fetch("/api/data", { signal: combined });

// Use with addEventListener
const controller2 = new AbortController();
element.addEventListener("click", handler, { signal: controller2.signal });
// Later: controller2.abort() removes the listener automatically
```

### Structured Concurrency

Run concurrent tasks that are cancelled as a group if any one fails.

```javascript
async function fetchUserProfile(userId, { signal } = {}) {
  const [user, posts, followers] = await Promise.all([
    fetch(`/api/users/${userId}`, { signal }).then((r) => r.json()),
    fetch(`/api/users/${userId}/posts`, { signal }).then((r) => r.json()),
    fetch(`/api/users/${userId}/followers`, { signal }).then((r) => r.json()),
  ]);
  return { user, posts, followers };
}

// Caller controls the lifetime
const controller = new AbortController();
try {
  const profile = await fetchUserProfile("u-123", {
    signal: AbortSignal.timeout(5000),
  });
} catch (err) {
  // all three fetches abort if any one fails or times out
}
```

```javascript
// Anti-pattern: fire-and-forget promises -- unhandled rejections
async function bad() {
  fetch("/api/log"); // no await, no catch -- rejection is silent
}
```

---

## Prototypes and Classes

### Prototype Chain

Every JavaScript object has an internal `[[Prototype]]` link. Property lookup walks this chain until it hits `null`.

```javascript
const animal = {
  type: "animal",
  speak() { return `${this.name} makes a sound`; },
};

const dog = Object.create(animal);
dog.name = "Rex";
dog.bark = function () { return `${this.name} barks`; };

console.log(dog.speak()); // "Rex makes a sound" -- found on prototype
console.log(dog.bark());  // "Rex barks" -- own property

// Inspect the chain
Object.getPrototypeOf(dog) === animal;          // true
Object.getPrototypeOf(animal) === Object.prototype; // true
Object.getPrototypeOf(Object.prototype);         // null (end of chain)
```

```javascript
// Anti-pattern: modifying Object.prototype
Object.prototype.myHelper = function () {}; // pollutes every object
```

### Class Fields and Private Members

```javascript
class Counter {
  // Public field with initializer
  count = 0;

  // Private field -- truly private, not accessible outside
  #limit;

  // Private method
  #checkLimit() {
    if (this.count >= this.#limit) {
      throw new RangeError(`Limit of ${this.#limit} reached`);
    }
  }

  constructor(limit = Infinity) {
    this.#limit = limit;
  }

  increment() {
    this.#checkLimit();
    this.count++;
    return this;
  }

  // Private static
  static #instances = 0;
  static getInstanceCount() { return Counter.#instances; }
}

const counter = new Counter(5);
counter.increment().increment();
// counter.#limit; // SyntaxError -- truly private
```

### Static Blocks

Run initialization logic once when the class is evaluated.

```javascript
class Config {
  static defaults;
  static envOverrides;

  static {
    // Complex initialization that needs try/catch
    try {
      Config.defaults = JSON.parse(readFileSync("defaults.json", "utf-8"));
    } catch {
      Config.defaults = { debug: false, logLevel: "info" };
    }
    Config.envOverrides = Object.freeze({
      debug: process.env.DEBUG === "1",
    });
  }

  get(key) {
    return Config.envOverrides[key] ?? Config.defaults[key];
  }
}
```

### Mixins

JavaScript does not support multiple inheritance, but mixins compose behavior via function application.

```javascript
const Serializable = (Base) =>
  class extends Base {
    toJSON() {
      const obj = {};
      for (const key of Object.keys(this)) {
        obj[key] = this[key];
      }
      return obj;
    }

    static fromJSON(json) {
      return Object.assign(new this(), JSON.parse(json));
    }
  };

const Timestamped = (Base) =>
  class extends Base {
    createdAt = new Date();
    updatedAt = new Date();

    touch() {
      this.updatedAt = new Date();
    }
  };

class Model {
  constructor(id) { this.id = id; }
}

class User extends Timestamped(Serializable(Model)) {
  constructor(id, name) {
    super(id);
    this.name = name;
  }
}

const user = new User(1, "Alice");
user.touch();
console.log(JSON.stringify(user.toJSON()));
```

### Symbol.hasInstance

Customize the behavior of `instanceof`.

```javascript
class ValidString {
  static [Symbol.hasInstance](value) {
    return typeof value === "string" && value.length > 0;
  }
}

console.log("hello" instanceof ValidString); // true
console.log("" instanceof ValidString);      // false
console.log(42 instanceof ValidString);      // false

// Practical use: type-checking utility
class JSONValue {
  static [Symbol.hasInstance](value) {
    try {
      JSON.stringify(value);
      return true;
    } catch {
      return false;
    }
  }
}
```

```javascript
// Anti-pattern: using instanceof across realms (iframes)
// array instanceof Array // may be false for arrays from another iframe
// Prefer: Array.isArray(array)
```

---

## Memory Management

### Garbage Collection Basics

JavaScript uses a mark-and-sweep GC. Objects are collected when no live reference points to them. The engine also performs generational collection (short-lived objects in the nursery, long-lived in the old generation).

### WeakRef

Hold a reference to an object without preventing garbage collection.

```javascript
class Cache {
  #entries = new Map();

  set(key, value) {
    this.#entries.set(key, new WeakRef(value));
  }

  get(key) {
    const ref = this.#entries.get(key);
    if (!ref) return undefined;
    const value = ref.deref();
    if (!value) {
      this.#entries.delete(key); // clean up dead entry
      return undefined;
    }
    return value;
  }
}

let bigObject = { data: new ArrayBuffer(1024 * 1024) };
const cache = new Cache();
cache.set("big", bigObject);

bigObject = null; // eligible for GC now
// cache.get("big") may return undefined after GC runs
```

### FinalizationRegistry

Run cleanup code when an object is garbage-collected.

```javascript
const registry = new FinalizationRegistry((heldValue) => {
  console.log(`Object with id ${heldValue} was collected`);
  // Clean up external resources: close sockets, release GPU memory, etc.
});

function createTracked(id) {
  const obj = { id, data: new ArrayBuffer(1024 * 1024) };
  registry.register(obj, id); // heldValue = id
  return obj;
}

let tracked = createTracked("resource-1");
tracked = null; // eventually logs: "Object with id resource-1 was collected"
```

**Warning:** `FinalizationRegistry` callbacks are non-deterministic. Never rely on them for correctness -- only for cleanup optimization.

### Common Memory Leaks

#### Closures Retaining Scope

```javascript
// LEAK: closure captures the entire scope, including largeData
function createHandler() {
  const largeData = new ArrayBuffer(50 * 1024 * 1024);
  const id = computeId(largeData);

  // This closure only needs `id`, but `largeData` stays alive
  return function handler() {
    console.log(`Handling ${id}`);
  };
}

// FIX: extract what you need, nullify the rest
function createHandlerFixed() {
  let largeData = new ArrayBuffer(50 * 1024 * 1024);
  const id = computeId(largeData);
  largeData = null; // release before returning the closure
  return function handler() {
    console.log(`Handling ${id}`);
  };
}
```

#### Forgotten Event Listeners

```javascript
// LEAK: listener holds a reference to `component`
class Component {
  constructor() {
    this.data = loadExpensiveData();
    window.addEventListener("resize", this.onResize);
  }
  onResize = () => { /* uses this.data */ };
}

// FIX: use AbortController for automatic cleanup
class ComponentFixed {
  #abort = new AbortController();

  constructor() {
    this.data = loadExpensiveData();
    window.addEventListener("resize", this.onResize, {
      signal: this.#abort.signal,
    });
  }
  onResize = () => { /* uses this.data */ };
  destroy() { this.#abort.abort(); }
}
```

#### Forgotten Timers

```javascript
// LEAK: interval keeps `this` alive even after component is removed
class Poller {
  #intervalId;
  start() {
    this.#intervalId = setInterval(() => this.poll(), 5000);
  }
  poll() { /* ... */ }

  // FIX: always clear timers on cleanup
  stop() { clearInterval(this.#intervalId); }
}
```

### Profiling in DevTools

1. **Heap Snapshot** -- Chrome DevTools > Memory > Take snapshot. Compare two snapshots to find objects that grew unexpectedly. Filter by "Objects allocated between Snapshot 1 and Snapshot 2."
2. **Allocation Timeline** -- Record allocations over time. Blue bars that never turn gray are retained objects.
3. **Allocation Sampling** -- Lower overhead profiling for production-like conditions. Shows which functions allocate the most memory.

```javascript
// Programmatic memory check (Node.js)
const used = process.memoryUsage();
console.log({
  rss: `${(used.rss / 1024 / 1024).toFixed(1)} MB`,
  heapUsed: `${(used.heapUsed / 1024 / 1024).toFixed(1)} MB`,
  heapTotal: `${(used.heapTotal / 1024 / 1024).toFixed(1)} MB`,
  external: `${(used.external / 1024 / 1024).toFixed(1)} MB`,
});
```

```javascript
// Anti-pattern: growing arrays without bounds
const log = [];
setInterval(() => {
  log.push({ ts: Date.now(), data: collectMetrics() }); // grows forever
}, 1000);
// FIX: use a ring buffer or cap the array length
```

---

## Web APIs

### Fetch with Proper Error Handling

```javascript
async function fetchJSON(url, options = {}) {
  const response = await fetch(url, {
    headers: { "Content-Type": "application/json", ...options.headers },
    ...options,
  });

  if (!response.ok) {
    const body = await response.text().catch(() => "");
    throw new Error(`HTTP ${response.status}: ${body}`);
  }

  return response.json();
}

// Retry with exponential backoff
async function fetchWithRetry(url, { retries = 3, signal } = {}) {
  for (let attempt = 0; attempt <= retries; attempt++) {
    try {
      return await fetch(url, { signal });
    } catch (err) {
      if (err.name === "AbortError" || attempt === retries) throw err;
      await new Promise((r) => setTimeout(r, 1000 * 2 ** attempt));
    }
  }
}
```

### Streams API

Process data incrementally without loading everything into memory.

```javascript
// ReadableStream: stream a large file to the user
async function streamDownload(url) {
  const response = await fetch(url);
  const reader = response.body.getReader();
  const decoder = new TextDecoder();
  let result = "";

  while (true) {
    const { done, value } = await reader.read();
    if (done) break;
    result += decoder.decode(value, { stream: true });
    updateProgress(result.length);
  }
  return result;
}

// TransformStream: process chunks on the fly
const uppercaseTransform = new TransformStream({
  transform(chunk, controller) {
    controller.enqueue(chunk.toUpperCase());
  },
});

const response = await fetch("/api/stream");
const transformed = response.body
  .pipeThrough(new TextDecoderStream())
  .pipeThrough(uppercaseTransform);

for await (const chunk of transformed) {
  console.log(chunk);
}
```

### Web Workers

Offload CPU-intensive work to a background thread.

```javascript
// worker.js
self.addEventListener("message", (event) => {
  const { data, id } = event.data;
  const result = heavyComputation(data);
  self.postMessage({ id, result });
});

// main.js
const worker = new Worker("worker.js");

function runInWorker(data) {
  const { promise, resolve } = Promise.withResolvers();
  const id = crypto.randomUUID();
  const handler = (event) => {
    if (event.data.id === id) {
      worker.removeEventListener("message", handler);
      resolve(event.data.result);
    }
  };
  worker.addEventListener("message", handler);
  worker.postMessage({ id, data });
  return promise;
}

const result = await runInWorker({ numbers: [1, 2, 3, 4, 5] });
```

### SharedArrayBuffer and Atomics

Share memory between the main thread and workers for high-performance scenarios.

```javascript
// Main thread
const buffer = new SharedArrayBuffer(1024);
const view = new Int32Array(buffer);
worker.postMessage({ buffer });

// Worker thread
self.addEventListener("message", (event) => {
  const view = new Int32Array(event.data.buffer);
  Atomics.add(view, 0, 1);          // atomic increment
  Atomics.notify(view, 0);          // wake waiting threads
});

// Main thread: wait for worker to update
Atomics.wait(view, 0, 0);           // blocks until value changes from 0
console.log(view[0]);               // 1
```

### BroadcastChannel

Communicate between tabs, windows, and iframes of the same origin.

```javascript
// Tab A
const channel = new BroadcastChannel("auth");
channel.postMessage({ type: "logout" });

// Tab B
const channel = new BroadcastChannel("auth");
channel.addEventListener("message", (event) => {
  if (event.data.type === "logout") {
    window.location.href = "/login";
  }
});
```

### Observers

```javascript
// IntersectionObserver -- lazy loading, infinite scroll
const observer = new IntersectionObserver(
  (entries) => {
    entries.forEach((entry) => {
      if (entry.isIntersecting) {
        entry.target.src = entry.target.dataset.src;
        observer.unobserve(entry.target);
      }
    });
  },
  { rootMargin: "200px" }
);
document.querySelectorAll("img[data-src]").forEach((img) => observer.observe(img));

// ResizeObserver -- respond to element size changes
const resize = new ResizeObserver((entries) => {
  for (const entry of entries) {
    const { width, height } = entry.contentRect;
    entry.target.style.fontSize = `${Math.max(12, width / 20)}px`;
  }
});
resize.observe(document.querySelector(".responsive-text"));

// MutationObserver -- react to DOM changes
const mutation = new MutationObserver((mutations) => {
  for (const m of mutations) {
    if (m.type === "childList") {
      console.log("Children changed:", m.addedNodes, m.removedNodes);
    }
  }
});
mutation.observe(document.getElementById("container"), {
  childList: true,
  subtree: true,
});
```

### Web Components

```javascript
class UserCard extends HTMLElement {
  static observedAttributes = ["name", "avatar"];

  #shadow;

  constructor() {
    super();
    this.#shadow = this.attachShadow({ mode: "open" });
  }

  connectedCallback() {
    this.#render();
  }

  attributeChangedCallback(name, oldValue, newValue) {
    if (oldValue !== newValue) this.#render();
  }

  #render() {
    this.#shadow.innerHTML = `
      <style>
        :host { display: flex; align-items: center; gap: 8px; }
        img { width: 40px; height: 40px; border-radius: 50%; }
      </style>
      <img src="${this.getAttribute("avatar") ?? ""}" alt="">
      <span>${this.getAttribute("name") ?? "Unknown"}</span>
    `;
  }
}

customElements.define("user-card", UserCard);
// <user-card name="Alice" avatar="/alice.jpg"></user-card>
```

---

## Bundler Configuration

### Vite

```javascript
// vite.config.js
import { defineConfig } from "vite";
import react from "@vitejs/plugin-react";

export default defineConfig({
  plugins: [react()],
  build: {
    target: "es2022",
    sourcemap: true,
    rollupOptions: {
      output: {
        manualChunks: {
          vendor: ["react", "react-dom"],
          charts: ["chart.js", "d3"],
        },
      },
    },
    chunkSizeWarningLimit: 500, // KB
  },
  resolve: {
    alias: { "@": "/src" },
  },
  server: {
    proxy: {
      "/api": { target: "http://localhost:3001", changeOrigin: true },
    },
  },
});
```

### esbuild

```javascript
// build.mjs
import * as esbuild from "esbuild";

await esbuild.build({
  entryPoints: ["src/index.js"],
  bundle: true,
  outdir: "dist",
  format: "esm",
  splitting: true,       // code splitting (ESM only)
  minify: true,
  sourcemap: true,
  target: ["es2022"],
  treeShaking: true,
  define: {
    "process.env.NODE_ENV": '"production"',
  },
  loader: {
    ".svg": "file",
    ".woff2": "file",
  },
  external: ["fsevents"], // platform-specific, don't bundle
});
```

### Webpack 5

```javascript
// webpack.config.js
import path from "path";

export default {
  mode: "production",
  entry: { main: "./src/index.js" },
  output: {
    path: path.resolve("dist"),
    filename: "[name].[contenthash:8].js",
    chunkFilename: "[name].[contenthash:8].chunk.js",
    clean: true,
  },
  optimization: {
    splitChunks: {
      chunks: "all",
      cacheGroups: {
        vendor: {
          test: /[\\/]node_modules[\\/]/,
          name: "vendor",
          chunks: "all",
          priority: 10,
        },
        common: {
          minChunks: 2,
          priority: 5,
          reuseExistingChunk: true,
        },
      },
    },
    runtimeChunk: "single",
    usedExports: true,   // tree shaking
  },
  devtool: "source-map",
  module: {
    rules: [
      { test: /\.js$/, exclude: /node_modules/, use: "babel-loader" },
      { test: /\.css$/, use: ["style-loader", "css-loader"] },
    ],
  },
};
```

### Rollup

```javascript
// rollup.config.js
import resolve from "@rollup/plugin-node-resolve";
import commonjs from "@rollup/plugin-commonjs";
import terser from "@rollup/plugin-terser";

export default {
  input: "src/index.js",
  output: [
    { file: "dist/index.esm.js", format: "es", sourcemap: true },
    { file: "dist/index.cjs.js", format: "cjs", sourcemap: true },
  ],
  plugins: [
    resolve(),                   // resolve node_modules
    commonjs(),                  // convert CJS to ESM
    terser({ compress: true }),  // minify
  ],
  external: ["react", "react-dom"], // peer deps -- don't bundle
  treeshake: {
    moduleSideEffects: false, // assume modules are side-effect free
    propertyReadSideEffects: false,
  },
};
```

### Code Splitting Strategies

| Strategy | When to Use | Implementation |
|---|---|---|
| Route-based | SPAs with distinct pages | `import("./pages/Dashboard.js")` |
| Component-based | Heavy components (charts, editors) | `import("chart.js")` on demand |
| Vendor splitting | Stable deps change less often | `manualChunks` / `splitChunks.cacheGroups` |
| Shared chunks | Code imported by 2+ entry points | `minChunks: 2` in webpack |
| Prefetch | Likely next navigations | `<link rel="prefetch" href="chunk.js">` |

### Source Maps

| Type | Build Speed | Debugging Quality | Use In |
|---|---|---|---|
| `source-map` | Slow | Full | Production (separate file) |
| `cheap-module-source-map` | Fast | Line-level | Development |
| `hidden-source-map` | Slow | Full (no reference in bundle) | Production + error tracking |
| `eval-source-map` | Fastest | Full | Development (HMR) |
| `nosources-source-map` | Slow | Stack traces only | Production (no source exposure) |

### Tree Shaking Optimization

```javascript
// Mark pure function calls to help the bundler
const result = /*#__PURE__*/ createExpensiveObject();

// Avoid side effects at module top level
// BAD: runs on import, blocks tree shaking
console.log("module loaded");
window.globalThing = setup();

// GOOD: defer side effects
export function init() {
  console.log("module loaded");
  window.globalThing = setup();
}
```

```javascript
// Anti-pattern: dynamic property access defeats static analysis
import * as utils from "./utils.js";
const fn = utils[someVariable]; // bundler cannot determine which export is used
```

---

## Common Patterns Quick Reference

| Pattern | When to Use | Example |
|---|---|---|
| `structuredClone` | Deep copy objects with complex types | `structuredClone(obj)` |
| `Object.groupBy` | Categorize array items by key | `Object.groupBy(items, (i) => i.type)` |
| `Promise.withResolvers` | External resolve/reject control | `const { promise, resolve } = Promise.withResolvers()` |
| `using` / `Symbol.dispose` | Deterministic resource cleanup | `using db = new Connection()` |
| Iterator helpers | Lazy chain on any iterator | `iter.filter(f).map(g).take(n).toArray()` |
| Dynamic `import()` | Code splitting and lazy loading | `const mod = await import("./heavy.js")` |
| `AbortController` | Cancel fetch, listeners, async work | `fetch(url, { signal: controller.signal })` |
| Async generators | Paginate or stream async data | `async function* paginate() { yield* page; }` |
| `queueMicrotask` | Batch DOM updates before repaint | `queueMicrotask(flush)` |
| Private fields (`#`) | True encapsulation in classes | `class C { #secret = 42; }` |
| Mixins | Compose behavior across class hierarchies | `class X extends Mixin(Base) {}` |
| `WeakRef` | Caches that don't prevent GC | `new WeakRef(expensiveObj)` |
| `FinalizationRegistry` | Cleanup when objects are collected | `registry.register(obj, metadata)` |
| Web Workers | CPU-heavy tasks off main thread | `new Worker("worker.js")` |
| `IntersectionObserver` | Lazy load images, infinite scroll | `observer.observe(element)` |
| `BroadcastChannel` | Cross-tab communication | `channel.postMessage({ type: "sync" })` |
| `TransformStream` | Process streaming data in chunks | `readable.pipeThrough(transform)` |
| Manual chunks | Stable vendor caching | `manualChunks: { vendor: ["react"] }` |
| `sideEffects: false` | Enable aggressive tree shaking | `package.json` field |
| Source maps | Debug minified production code | `devtool: "source-map"` |
