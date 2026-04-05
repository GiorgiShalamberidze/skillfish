---
name: debugger
description: Systematic debugging: root cause analysis techniques, debugger configurations (VS Code, Chrome DevTools), logging strategies, memory leak detection, and race conditions.
---

# Debugger

Systematic debugging skill covering root cause analysis methodology, IDE and browser debugger configurations, structured logging, memory leak detection, race condition diagnosis, and production debugging techniques. Applies the scientific method to bugs: observe, hypothesize, test, fix, verify.

---

## Table of Contents

1. [Debugging Methodology](#1-debugging-methodology)
2. [VS Code Debugger](#2-vs-code-debugger)
3. [Chrome DevTools](#3-chrome-devtools)
4. [Logging Strategies](#4-logging-strategies)
5. [Memory Leak Detection](#5-memory-leak-detection)
6. [Race Conditions](#6-race-conditions)
7. [Production Debugging](#7-production-debugging)
8. [Common Patterns Quick Reference](#8-common-patterns-quick-reference)

---

## When to Use

- A bug has no obvious cause and needs systematic investigation
- You need to configure debugger launch settings for a project
- Application performance degrades over time (memory leak suspected)
- Intermittent failures suggest race conditions or timing issues
- Production incidents require distributed tracing or log correlation
- You want structured logging that actually helps during incidents

---

## 1. Debugging Methodology

### The Scientific Method for Bugs

Every debugging session should follow a disciplined process. Guessing wastes time.

```
1. OBSERVE     — Collect symptoms. What exactly fails? When? For whom?
2. HYPOTHESIZE — Form a specific, falsifiable theory about the cause.
3. PREDICT     — If hypothesis is correct, what should a test reveal?
4. TEST        — Run the smallest experiment that confirms or refutes.
5. CONCLUDE    — Root cause confirmed? Fix it. Wrong? Return to step 2.
6. VERIFY      — Confirm the fix resolves the original symptom AND
                 does not introduce regressions.
```

### Binary Search / Bisection

When you know "it worked before," bisect to find the breaking change.

```bash
# Git bisect — finds the exact commit that introduced the bug
git bisect start
git bisect bad                  # current HEAD is broken
git bisect good v2.3.0          # this tag was known-good

# Git runs binary search; at each step, test and mark:
git bisect good   # or
git bisect bad

# Automate with a test script (exit 0 = good, exit 1 = bad):
git bisect run npm test

# When done:
git bisect reset
```

```python
# Binary search in code — narrow down the offending line/module
# Comment out half the suspect code, test, repeat.
# Works for: config files, CSS rules, middleware chains, plugin lists.

# Example: which middleware breaks the request?
middlewares = [auth, cors, rate_limit, body_parser, logger, validator]
# Test with first half only:
middlewares = [auth, cors, rate_limit]
# Still broken? Problem is in first half. Split again.
# Not broken? Problem is in second half.
```

### Rubber Duck Debugging

Explain the problem out loud, line by line. The act of articulating forces you to confront assumptions.

```markdown
## Rubber Duck Template

**What should happen:**
[Describe expected behavior precisely]

**What actually happens:**
[Describe actual behavior with exact error messages]

**What I have verified so far:**
- [ ] Input data is correct (logged it)
- [ ] Function is actually being called (added breakpoint / log)
- [ ] Return value is what I expect (inspected it)
- [ ] No exceptions are swallowed silently (checked catch blocks)
- [ ] Environment variables are set correctly (printed them)

**My current hypothesis:**
[One specific theory]

**How I will test it:**
[One specific experiment]
```

### Minimal Reproduction

Isolate the bug from everything else. A minimal repro is the fastest path to a fix.

```bash
# Steps to create a minimal reproduction:
# 1. Start from a clean project (npx create-react-app repro / cargo new repro)
# 2. Add ONLY the code needed to trigger the bug
# 3. Remove all unrelated dependencies
# 4. Document exact steps to reproduce

# Example: Node.js minimal repro
mkdir bug-repro && cd bug-repro
npm init -y
npm install express@4.18.2  # pin exact version
cat > index.js << 'EOF'
const express = require('express');
const app = express();
// Minimal code that triggers the bug:
app.use(express.json({ limit: '1kb' }));
app.post('/data', (req, res) => {
  res.json({ received: req.body });
});
app.listen(3000);
EOF
# Test: curl -X POST -H "Content-Type: application/json" -d '{"big":"data..."}' localhost:3000/data
```

### Anti-Patterns

```
AVOID: Shotgun debugging
  Changing random things until it works. You won't understand WHY it
  works, and you may introduce new bugs.

AVOID: Debugging by superstition
  "It works on my machine" is not a conclusion. Identify the
  environmental difference.

AVOID: Fixing the symptom instead of the cause
  Adding a try/catch that swallows an error is not a fix.
  Retrying a flaky request without understanding why it fails
  just hides the problem.

AVOID: Skipping the verification step
  A fix that isn't tested against the original reproduction
  is not confirmed. Write a regression test.
```

---

## 2. VS Code Debugger

### Node.js — launch.json

```jsonc
// .vscode/launch.json
{
  "version": "0.2.0",
  "configurations": [
    {
      // Basic Node.js debugging
      "name": "Debug Node App",
      "type": "node",
      "request": "launch",
      "program": "${workspaceFolder}/src/index.ts",
      "preLaunchTask": "tsc: build",
      "outFiles": ["${workspaceFolder}/dist/**/*.js"],
      "sourceMaps": true,
      "console": "integratedTerminal",
      "env": {
        "NODE_ENV": "development",
        "DEBUG": "app:*"
      }
    },
    {
      // Debug current test file with Jest
      "name": "Debug Jest Test",
      "type": "node",
      "request": "launch",
      "runtimeExecutable": "${workspaceFolder}/node_modules/.bin/jest",
      "args": [
        "${relativeFile}",
        "--no-coverage",
        "--runInBand",
        "--watchAll=false"
      ],
      "console": "integratedTerminal",
      "sourceMaps": true
    },
    {
      // Attach to running Node process (e.g., started with --inspect)
      "name": "Attach to Node",
      "type": "node",
      "request": "attach",
      "port": 9229,
      "restart": true,
      "sourceMaps": true
    },
    {
      // Debug Next.js server-side
      "name": "Debug Next.js",
      "type": "node",
      "request": "launch",
      "runtimeExecutable": "${workspaceFolder}/node_modules/.bin/next",
      "args": ["dev"],
      "env": { "NODE_OPTIONS": "--inspect" },
      "sourceMaps": true
    }
  ]
}
```

### Python — launch.json

```jsonc
{
  "version": "0.2.0",
  "configurations": [
    {
      // Standard Python debugging
      "name": "Debug Python",
      "type": "debugpy",
      "request": "launch",
      "program": "${file}",
      "console": "integratedTerminal",
      "justMyCode": false,  // step into library code when needed
      "env": {
        "PYTHONDONTWRITEBYTECODE": "1"
      }
    },
    {
      // Debug FastAPI with uvicorn
      "name": "Debug FastAPI",
      "type": "debugpy",
      "request": "launch",
      "module": "uvicorn",
      "args": ["app.main:app", "--reload", "--port", "8000"],
      "jinja": true,
      "justMyCode": false
    },
    {
      // Debug pytest
      "name": "Debug Pytest",
      "type": "debugpy",
      "request": "launch",
      "module": "pytest",
      "args": [
        "${file}",
        "-xvs",
        "--no-header"
      ],
      "justMyCode": false
    },
    {
      // Debug Django
      "name": "Debug Django",
      "type": "debugpy",
      "request": "launch",
      "program": "${workspaceFolder}/manage.py",
      "args": ["runserver", "--noreload"],
      "django": true
    }
  ]
}
```

### Go — launch.json

```jsonc
{
  "version": "0.2.0",
  "configurations": [
    {
      "name": "Debug Go",
      "type": "go",
      "request": "launch",
      "mode": "auto",
      "program": "${fileDirname}",
      "args": [],
      "env": {
        "CGO_ENABLED": "1"
      },
      "showLog": true
    },
    {
      // Debug Go tests
      "name": "Debug Go Test",
      "type": "go",
      "request": "launch",
      "mode": "test",
      "program": "${fileDirname}",
      "args": ["-v", "-run", "TestFunctionName"],
      "showLog": true
    },
    {
      // Attach to running Go process via Delve
      "name": "Attach to Delve",
      "type": "go",
      "request": "attach",
      "mode": "remote",
      "remotePath": "/app",
      "port": 2345,
      "host": "127.0.0.1"
    }
  ]
}
```

### Rust — launch.json

```jsonc
{
  "version": "0.2.0",
  "configurations": [
    {
      // Debug Rust binary (CodeLLDB)
      "name": "Debug Rust",
      "type": "lldb",
      "request": "launch",
      "cargo": {
        "args": ["build", "--bin=myapp", "--package=myapp"],
        "filter": {
          "name": "myapp",
          "kind": "bin"
        }
      },
      "args": [],
      "cwd": "${workspaceFolder}",
      "env": {
        "RUST_BACKTRACE": "1",
        "RUST_LOG": "debug"
      }
    },
    {
      // Debug Rust tests
      "name": "Debug Rust Test",
      "type": "lldb",
      "request": "launch",
      "cargo": {
        "args": ["test", "--no-run", "--package=myapp"],
        "filter": {
          "name": "myapp",
          "kind": "lib"
        }
      },
      "args": ["--test-threads=1"],
      "cwd": "${workspaceFolder}",
      "env": { "RUST_BACKTRACE": "full" }
    }
  ]
}
```

### Conditional Breakpoints and Logpoints

```
Conditional Breakpoint:
  Right-click gutter > "Add Conditional Breakpoint"
  Expression: userId === "abc123" && items.length > 100
  Hit Count: break only after N hits (e.g., 50)

Logpoint (non-breaking log):
  Right-click gutter > "Add Logpoint"
  Message: User {userId} requested {items.length} items at {Date.now()}
  Logpoints do NOT pause execution — they print to Debug Console.
  Use when you need printf-debugging without modifying source.

Data Breakpoint (VS Code 1.70+):
  In Variables pane, right-click a variable > "Break on Value Change"
  Pauses whenever that memory address is written to.
  Supported for C/C++, Rust (via CodeLLDB), and .NET.
```

### Remote Debugging (Docker / SSH)

```jsonc
{
  // Debug Node.js inside Docker
  "name": "Docker: Attach Node",
  "type": "node",
  "request": "attach",
  "port": 9229,
  "address": "localhost",
  "localRoot": "${workspaceFolder}",
  "remoteRoot": "/app",
  "sourceMaps": true,
  "restart": true
}
```

```dockerfile
# Dockerfile — expose debug port
FROM node:20-alpine
WORKDIR /app
COPY . .
RUN npm ci
# Start with --inspect for debugger attachment
CMD ["node", "--inspect=0.0.0.0:9229", "dist/server.js"]
```

```yaml
# docker-compose.yml
services:
  app:
    build: .
    ports:
      - "3000:3000"
      - "9229:9229"   # debugger port
```

### Anti-Patterns

```
AVOID: Leaving "stopOnEntry": true in shared launch.json
  This pauses on the first line of every debug session and confuses
  teammates who don't expect it.

AVOID: Forgetting sourceMaps for TypeScript
  You'll step through compiled JS instead of your source TS.
  Always set "sourceMaps": true and "outFiles" correctly.

AVOID: Debugging optimized builds
  Compiler optimizations inline functions and reorder code. Debug
  with development/debug builds only.

AVOID: Hardcoding absolute paths in launch.json
  Use ${workspaceFolder}, ${file}, ${relativeFile} variables.
  Absolute paths break when teammates clone to different locations.
```

---

## 3. Chrome DevTools

### Sources Panel

```
Key Features:
  Cmd+P / Ctrl+P       — Open file by name (works with source maps)
  Cmd+Shift+P          — Run command (like VS Code command palette)
  Cmd+D                — Add selection to watch
  Breakpoints pane     — See all breakpoints, toggle, add conditions
  Call Stack pane      — Navigate frames, restart frame
  Scope pane           — Inspect local/closure/global variables

Source Maps:
  DevTools automatically loads .map files referenced in bundled JS.
  If source maps are missing, check that your bundler outputs them
  and the //# sourceMappingURL= comment is present at the end of
  the bundle.

Event Listener Breakpoints:
  Sources > Event Listener Breakpoints > expand category
  Useful for: click handlers, XHR/fetch, DOM mutation, timer events

XHR/Fetch Breakpoints:
  Sources > XHR/Fetch Breakpoints > add URL pattern
  Pauses when any fetch/XHR matches the pattern (e.g., "/api/users")
```

### Network Panel

```bash
# Key techniques:
# 1. Filter by type: XHR, JS, CSS, Img, Media, Font, Doc, WS
# 2. Filter by text: "api/v1" or "status-code:500" or "method:POST"
# 3. Throttle: Fast 3G, Slow 3G, Offline (test loading states)
# 4. Preserve log: keeps requests across page navigations
# 5. Disable cache: forces fresh loads during development

# Copy as cURL:
#   Right-click any request > Copy > Copy as cURL
#   Paste into terminal to replay the exact request with all headers

# Block requests:
#   Right-click request > Block request URL (or domain)
#   Useful for testing fallback behavior when a CDN/API is unavailable

# HAR export:
#   Right-click > Save all as HAR with content
#   Share HAR files for reproducing network-related bugs
```

### Performance Profiler

```
Recording a profile:
  1. Open Performance tab
  2. Click Record (or Cmd+E)
  3. Perform the slow action in the app
  4. Click Stop
  5. Analyze the flame chart

What to look for:
  Long Tasks    — Yellow/red bars over 50ms block the main thread
  Layout Shift  — Purple bars indicate forced reflows
  Paint         — Green bars show repaint regions
  Scripting     — Identify which functions consume the most time
  Idle          — Gaps where the main thread is available

Bottom-Up vs. Call Tree:
  Bottom-Up     — Shows which functions have the highest self-time
                   (time spent in the function itself, not callees)
  Call Tree     — Shows the full call hierarchy from root to leaf
  Event Log     — Chronological list of all events

Tip: Use "Performance Insights" panel (newer Chrome) for automatic
     annotations of layout shifts, long tasks, and render-blocking
     resources.
```

### Memory Snapshots

```
Taking a heap snapshot:
  1. Open Memory tab
  2. Select "Heap snapshot" > Take snapshot
  3. Perform actions that may leak
  4. Take another snapshot
  5. Compare snapshots using "Comparison" view

Three-snapshot technique for leak detection:
  Snapshot 1: Baseline (after page load)
  Action:     Trigger the suspected leaky operation
  Snapshot 2: After first trigger
  Action:     Trigger the same operation again
  Snapshot 3: After second trigger
  Compare 2→3: Objects that grow between 2 and 3 are likely leaking

Allocation instrumentation on timeline:
  Memory tab > "Allocation instrumentation on timeline"
  Shows blue bars for allocations, gray for GC'd objects
  Blue bars that never turn gray = leaked objects

Detached DOM nodes:
  In snapshot, filter by "Detached"
  Shows DOM nodes removed from the tree but still referenced in JS
  Common cause: event listeners or closures holding references
```

### Framework DevTools

```
React DevTools:
  Components tab — inspect component tree, props, state, hooks
  Profiler tab   — record renders, see which components re-rendered and why
  Highlight updates: Settings > "Highlight updates when components render"

  Useful techniques:
  - Click component > "Log this component" to get it in console as $r
  - Use "Rendered by" to trace why a component rendered
  - Filter by name to find components in deep trees

Vue DevTools:
  Components    — inspect component hierarchy, data, computed, vuex
  Timeline      — track events, mutations, route changes
  Pinia/Vuex    — inspect store state, replay mutations
  Routes        — view current route, params, matched components

Angular DevTools:
  Component Explorer — view component tree, inputs, outputs
  Profiler           — detect change detection cycles
  Dependency injection graph — visualize DI hierarchy
```

### Anti-Patterns

```
AVOID: Leaving console.log debugging in production
  Use proper logging infrastructure instead of console.log.
  At minimum, strip console.* calls in production builds.

AVOID: Ignoring the Performance tab and guessing
  Perceived slowness is unreliable. Profile first, then optimize
  the actual bottleneck.

AVOID: Taking heap snapshots of the DevTools page itself
  Make sure you're profiling your application tab, not the
  DevTools window.

AVOID: Disabling source maps in production entirely
  Upload source maps to your error-tracking service (Sentry, etc.)
  but exclude them from public serving. You get readable stack
  traces without exposing source code.
```

---

## 4. Logging Strategies

### Structured Logging

Structured logs (JSON) are searchable, parseable, and work with log aggregation tools. Unstructured text logs are for humans reading a terminal; structured logs are for machines and dashboards.

```typescript
// Node.js — pino (fastest JSON logger)
import pino from 'pino';

const logger = pino({
  level: process.env.LOG_LEVEL || 'info',
  transport: process.env.NODE_ENV === 'development'
    ? { target: 'pino-pretty', options: { colorize: true } }
    : undefined,  // JSON in production
  base: {
    service: 'user-api',
    version: process.env.APP_VERSION,
    env: process.env.NODE_ENV,
  },
  redact: ['req.headers.authorization', 'password', 'ssn', 'creditCard'],
});

// Usage with context
logger.info({ userId: 'abc123', action: 'login', ip: req.ip }, 'User logged in');
logger.error({ err, orderId: '789', step: 'payment' }, 'Payment processing failed');

// Child loggers carry context automatically
const reqLogger = logger.child({ requestId: req.id, userId: req.user?.id });
reqLogger.info({ path: req.path }, 'Request started');
reqLogger.info({ statusCode: res.statusCode, durationMs: elapsed }, 'Request completed');
```

```python
# Python — structlog
import structlog

structlog.configure(
    processors=[
        structlog.contextvars.merge_contextvars,
        structlog.processors.add_log_level,
        structlog.processors.TimeStamper(fmt="iso"),
        structlog.processors.StackInfoRenderer(),
        structlog.processors.format_exc_info,
        structlog.processors.JSONRenderer(),  # JSON in production
    ],
    wrapper_class=structlog.make_filtering_bound_logger(logging.INFO),
)

log = structlog.get_logger()

# Bind context that carries through the request
log = log.bind(request_id=request_id, user_id=user.id)
log.info("order_created", order_id=order.id, total=order.total)
log.error("payment_failed", order_id=order.id, error=str(exc))
```

```javascript
// Node.js — winston (widely used, more features than pino)
import winston from 'winston';

const logger = winston.createLogger({
  level: 'info',
  format: winston.format.combine(
    winston.format.timestamp(),
    winston.format.errors({ stack: true }),
    winston.format.json()
  ),
  defaultMeta: { service: 'order-api' },
  transports: [
    new winston.transports.Console({
      format: process.env.NODE_ENV === 'development'
        ? winston.format.combine(winston.format.colorize(), winston.format.simple())
        : winston.format.json(),
    }),
  ],
});
```

### Log Levels

```
LEVEL       WHEN TO USE                                    EXAMPLE
─────────── ────────────────────────────────────────────── ───────────────────────────────
fatal/crit  Process cannot continue, about to crash        "Database connection pool exhausted"
error       Operation failed, needs attention               "Payment API returned 503"
warn        Unexpected but handled, may indicate a problem  "Cache miss rate above 50%"
info        Normal business events worth recording          "Order #789 shipped"
debug       Developer context for troubleshooting           "SQL query returned 42 rows in 12ms"
trace       Very granular, function-entry-level detail      "Entering validateToken()"

RULES:
- Production: info and above by default
- Increase to debug for specific modules during investigation
- Never log at trace in production (performance impact)
- error means "something is broken" — not "user sent bad input" (that's warn)
- Logs at error/fatal should be actionable — include what failed and what to do
```

### Correlation IDs

```typescript
// Express middleware — generate or propagate correlation ID
import { randomUUID } from 'crypto';
import { AsyncLocalStorage } from 'async_hooks';

const asyncLocalStorage = new AsyncLocalStorage<{ requestId: string }>();

app.use((req, res, next) => {
  const requestId = req.headers['x-request-id'] as string || randomUUID();
  res.setHeader('x-request-id', requestId);

  asyncLocalStorage.run({ requestId }, () => {
    next();
  });
});

// In any downstream function, retrieve the correlation ID:
function getRequestId(): string {
  return asyncLocalStorage.getStore()?.requestId || 'no-request-id';
}

// Logger automatically includes correlation ID
function createLogger() {
  return {
    info(msg: string, data?: object) {
      console.log(JSON.stringify({
        level: 'info',
        requestId: getRequestId(),
        msg,
        ...data,
        timestamp: new Date().toISOString(),
      }));
    }
  };
}

// Cross-service propagation:
// When calling another service, forward the correlation ID:
// headers: { 'x-request-id': getRequestId() }
```

### Anti-Patterns

```
AVOID: Logging sensitive data
  Never log passwords, tokens, credit card numbers, SSNs, or PII.
  Use pino's redact option or structlog processors to strip fields.

AVOID: String interpolation in log messages
  BAD:  logger.debug(`Found ${results.length} users matching ${query}`);
  GOOD: logger.debug({ count: results.length, query }, 'User search completed');
  Structured fields are searchable; interpolated strings are not.

AVOID: Logging inside tight loops
  Logging 10,000 times per second will kill throughput.
  Log summaries (batch counts) or sample at a rate.

AVOID: Using console.log as your logging strategy
  console.log has no levels, no structure, no redaction, no rotation.
  Use a real logging library even for small projects.

AVOID: Inconsistent log message formats across services
  Agree on a schema: { timestamp, level, service, requestId, msg, ... }
  Inconsistent formats make log aggregation and alerting painful.
```

---

## 5. Memory Leak Detection

### JavaScript / Node.js

```bash
# Start Node with inspector enabled
node --inspect dist/server.js

# Or attach inspector to a running process
kill -USR1 <pid>   # sends SIGUSR1, enables inspector on 9229

# Open chrome://inspect in Chrome to connect
```

```javascript
// Heap snapshot from code (useful in tests)
const v8 = require('v8');
const fs = require('fs');

function takeHeapSnapshot(label) {
  const snapshotStream = v8.writeHeapSnapshot();
  console.log(`Heap snapshot written to ${snapshotStream}`);
}

// Memory usage tracking
function logMemoryUsage(label) {
  const usage = process.memoryUsage();
  console.log({
    label,
    rss: `${(usage.rss / 1024 / 1024).toFixed(1)} MB`,
    heapTotal: `${(usage.heapTotal / 1024 / 1024).toFixed(1)} MB`,
    heapUsed: `${(usage.heapUsed / 1024 / 1024).toFixed(1)} MB`,
    external: `${(usage.external / 1024 / 1024).toFixed(1)} MB`,
  });
}
```

```javascript
// Common Node.js memory leak patterns

// LEAK: Unbounded cache
const cache = {};  // grows forever
function getUser(id) {
  if (!cache[id]) cache[id] = db.fetchUser(id);
  return cache[id];
}

// FIX: Use LRU cache with max size
import { LRUCache } from 'lru-cache';
const cache = new LRUCache({ max: 1000, ttl: 1000 * 60 * 5 });

// LEAK: Event listener not removed
class JsonStream {
  constructor(socket) {
    socket.on('data', this.handleData.bind(this));  // never removed
  }
}

// FIX: Remove listener on cleanup
class JsonStream {
  constructor(socket) {
    this.handler = this.handleData.bind(this);
    socket.on('data', this.handler);
  }
  destroy() {
    this.socket.off('data', this.handler);
  }
}

// LEAK: Closures capturing large scope
function processLargeData(hugeArray) {
  const summary = computeSummary(hugeArray);
  return function getSummary() {
    // This closure captures `hugeArray` even though it only needs `summary`
    return summary;
  };
}

// FIX: Null out the reference
function processLargeData(hugeArray) {
  const summary = computeSummary(hugeArray);
  hugeArray = null;  // allow GC to collect it
  return function getSummary() {
    return summary;
  };
}
```

### Python

```python
# tracemalloc — built-in memory tracing
import tracemalloc

tracemalloc.start()

# ... run the suspected leaky code ...

snapshot = tracemalloc.take_snapshot()
top_stats = snapshot.statistics('lineno')

print("Top 10 memory allocations:")
for stat in top_stats[:10]:
    print(stat)

# Compare two snapshots to find growth
snapshot1 = tracemalloc.take_snapshot()
# ... more operations ...
snapshot2 = tracemalloc.take_snapshot()

top_stats = snapshot2.compare_to(snapshot1, 'lineno')
print("\nTop 10 memory increases:")
for stat in top_stats[:10]:
    print(stat)
```

```python
# objgraph — visualize object references
import objgraph

# Show types with the most instances
objgraph.show_most_common_types(limit=10)

# Find objects that are increasing between calls
objgraph.show_growth(limit=10)

# Show reference chain to a leaked object
leaked = objgraph.by_type('MyClass')
if leaked:
    objgraph.show_backrefs(leaked[0], max_depth=5, filename='refs.png')
```

```python
# Common Python leak patterns

# LEAK: Default mutable argument accumulates data
def add_item(item, items=[]):  # items persists across calls
    items.append(item)
    return items

# FIX: Use None as default
def add_item(item, items=None):
    if items is None:
        items = []
    items.append(item)
    return items

# LEAK: Circular references with __del__
class Node:
    def __init__(self):
        self.parent = None
        self.children = []
    def __del__(self):
        print("deleted")  # prevents GC of circular refs in older Python

# FIX: Use weakref for back-references
import weakref
class Node:
    def __init__(self):
        self._parent = None
        self.children = []
    @property
    def parent(self):
        return self._parent() if self._parent else None
    @parent.setter
    def parent(self, value):
        self._parent = weakref.ref(value) if value else None
```

### Go

```go
// pprof — built-in profiling
import (
    "net/http"
    _ "net/http/pprof"  // register /debug/pprof/ handlers
)

func main() {
    // Expose pprof on a separate port for security
    go func() {
        http.ListenAndServe("localhost:6060", nil)
    }()
    // ... main application ...
}
```

```bash
# Capture and analyze heap profile
go tool pprof http://localhost:6060/debug/pprof/heap

# Inside pprof interactive shell:
(pprof) top 10           # top memory consumers
(pprof) list MyFunction  # annotated source for a function
(pprof) web              # open flamegraph in browser

# Compare two heap profiles (find growth)
go tool pprof -base heap1.pb.gz heap2.pb.gz

# Allocation profiling (track all allocations, not just live objects)
go tool pprof http://localhost:6060/debug/pprof/allocs

# Goroutine leak detection
go tool pprof http://localhost:6060/debug/pprof/goroutine
# Look for growing goroutine counts — each leaked goroutine holds memory
```

```go
// Common Go leak patterns

// LEAK: Goroutine leak — channel never consumed
func leaky() {
    ch := make(chan int)
    go func() {
        result := expensiveComputation()
        ch <- result  // blocks forever if nobody reads ch
    }()
    // function returns without reading from ch
}

// FIX: Use context for cancellation
func fixed(ctx context.Context) (int, error) {
    ch := make(chan int, 1)  // buffered so goroutine won't block
    go func() {
        ch <- expensiveComputation()
    }()
    select {
    case result := <-ch:
        return result, nil
    case <-ctx.Done():
        return 0, ctx.Err()
    }
}
```

### Anti-Patterns

```
AVOID: Taking heap snapshots under heavy load
  Snapshots pause the runtime. In production, take them during
  low-traffic windows or use sampling profilers instead.

AVOID: Ignoring "Detached DOM tree" warnings in DevTools
  Detached DOM nodes are a classic browser memory leak.
  Always investigate them — usually caused by lingering references
  in event handlers or framework state.

AVOID: Using global variables as caches without bounds
  Every unbounded in-memory cache is a memory leak waiting to happen.
  Always set max size and TTL.

AVOID: Assuming the garbage collector will save you
  GC cannot collect objects that are still reachable. Leaked
  references (closures, event listeners, global maps) defeat GC.
```

---

## 6. Race Conditions

### Identifying Race Conditions

```
Symptoms of a race condition:
  - Test passes 99 times, fails on the 100th
  - Bug appears under load but not in development
  - Data corruption that "shouldn't be possible"
  - Operations complete in wrong order intermittently
  - "Lost update" — two writes, only one persists

Mental model:
  If two operations access shared state, and at least one is a write,
  and there is no synchronization, you have a data race.

Categories:
  1. Check-then-act   — "if file exists, read file" (file deleted between check and read)
  2. Read-modify-write — counter++ (two threads read same value, both write value+1)
  3. Publication       — object partially constructed when another thread reads it
  4. Order violation   — operation B assumes A completed, but A hasn't yet
```

### Thread Sanitizer (C/C++/Rust)

```bash
# Compile with ThreadSanitizer (TSan)
# C/C++
clang -fsanitize=thread -g -O1 myprogram.c -o myprogram
./myprogram
# TSan output shows: data race between threads, with stack traces for both accesses

# Rust (nightly only)
RUSTFLAGS="-Z sanitizer=thread" cargo +nightly test -Z build-std --target x86_64-unknown-linux-gnu
```

### Go Race Detector

```bash
# Build and run with race detector
go run -race ./cmd/server
go test -race ./...
go build -race -o myapp ./cmd/myapp && ./myapp

# Race detector output example:
# WARNING: DATA RACE
# Write at 0x00c0000a0000 by goroutine 7:
#   main.incrementCounter()
#       /app/main.go:25 +0x4a
# Previous read at 0x00c0000a0000 by goroutine 6:
#   main.readCounter()
#       /app/main.go:30 +0x3e
```

```go
// RACE: Unsynchronized map access
var cache = map[string]string{}

func Set(key, value string) { cache[key] = value }  // concurrent write
func Get(key string) string { return cache[key] }    // concurrent read

// FIX: Use sync.RWMutex or sync.Map
var (
    cache = map[string]string{}
    mu    sync.RWMutex
)

func Set(key, value string) {
    mu.Lock()
    defer mu.Unlock()
    cache[key] = value
}

func Get(key string) string {
    mu.RLock()
    defer mu.RUnlock()
    return cache[key]
}

// OR use sync.Map for simple key-value cases
var cache sync.Map
cache.Store(key, value)
val, ok := cache.Load(key)
```

### Async / JavaScript Race Conditions

```typescript
// RACE: Stale closure in React
function SearchResults() {
  const [query, setQuery] = useState('');
  const [results, setResults] = useState([]);

  useEffect(() => {
    // If user types "cat" then "car", the "cat" response may arrive
    // after "car" and overwrite the correct results
    fetch(`/api/search?q=${query}`)
      .then(r => r.json())
      .then(data => setResults(data));  // stale result may win
  }, [query]);
}

// FIX: AbortController cancels superseded requests
function SearchResults() {
  const [query, setQuery] = useState('');
  const [results, setResults] = useState([]);

  useEffect(() => {
    const controller = new AbortController();

    fetch(`/api/search?q=${query}`, { signal: controller.signal })
      .then(r => r.json())
      .then(data => setResults(data))
      .catch(err => {
        if (err.name !== 'AbortError') throw err;
      });

    return () => controller.abort();  // cancel on re-render
  }, [query]);
}
```

```typescript
// RACE: Check-then-act in async code
async function createUserIfNotExists(email: string) {
  const existing = await db.users.findOne({ email });
  if (!existing) {
    await db.users.insertOne({ email });  // two requests can pass the check
  }
}

// FIX: Use database-level unique constraint + upsert
async function createUserIfNotExists(email: string) {
  await db.users.updateOne(
    { email },
    { $setOnInsert: { email, createdAt: new Date() } },
    { upsert: true }
  );
}

// FIX (alternative): Use advisory locks for complex operations
async function withLock(key: string, fn: () => Promise<void>) {
  const lock = await redis.set(`lock:${key}`, '1', 'NX', 'EX', 30);
  if (!lock) throw new Error('Could not acquire lock');
  try {
    await fn();
  } finally {
    await redis.del(`lock:${key}`);
  }
}
```

### Lock Ordering (Deadlock Prevention)

```
Deadlock occurs when:
  Thread A holds Lock 1, waits for Lock 2
  Thread B holds Lock 2, waits for Lock 1

Prevention — consistent lock ordering:
  Always acquire locks in a globally defined order.
  If you need locks on Account A and Account B,
  always lock the one with the lower ID first.
```

```python
# DEADLOCK: Inconsistent lock ordering
def transfer(from_account, to_account, amount):
    with from_account.lock:          # Thread 1: locks A, Thread 2: locks B
        with to_account.lock:        # Thread 1: waits B, Thread 2: waits A → deadlock
            from_account.balance -= amount
            to_account.balance += amount

# FIX: Always lock in consistent order (by account ID)
def transfer(from_account, to_account, amount):
    first, second = sorted([from_account, to_account], key=lambda a: a.id)
    with first.lock:
        with second.lock:
            from_account.balance -= amount
            to_account.balance += amount
```

### Anti-Patterns

```
AVOID: Using sleep() to "fix" race conditions
  sleep(100) just makes the race less likely, not impossible.
  It will still fail in CI, under load, or on slow hardware.

AVOID: Ignoring race detector warnings
  Go's -race flag has near-zero false positives. Every warning
  is a real data race. Fix them all.

AVOID: Double-checked locking without memory barriers
  Classic pattern in Java/C++ that is subtly broken without
  volatile/atomic semantics. Use language-provided patterns
  (sync.Once in Go, std::call_once in C++).

AVOID: Testing concurrency with a single thread
  Race conditions only manifest under concurrent execution.
  Test with multiple goroutines/threads/workers.
```

---

## 7. Production Debugging

### Distributed Tracing

```
Distributed tracing tracks a request across multiple services.
Each service adds a "span" to the trace, forming a tree of operations.

Key concepts:
  Trace ID    — unique ID for the entire request journey
  Span ID     — unique ID for one operation within the trace
  Parent Span — links spans into a tree
  Baggage     — key-value pairs propagated across services

Popular tools: Jaeger, Zipkin, Tempo, AWS X-Ray, Datadog APT
Standard: OpenTelemetry (OTEL) — vendor-neutral instrumentation
```

```typescript
// OpenTelemetry setup for Node.js
import { NodeSDK } from '@opentelemetry/sdk-node';
import { OTLPTraceExporter } from '@opentelemetry/exporter-trace-otlp-http';
import { getNodeAutoInstrumentations } from '@opentelemetry/auto-instrumentations-node';

const sdk = new NodeSDK({
  traceExporter: new OTLPTraceExporter({
    url: 'http://otel-collector:4318/v1/traces',
  }),
  instrumentations: [
    getNodeAutoInstrumentations({
      // Auto-instrument HTTP, Express, pg, redis, etc.
      '@opentelemetry/instrumentation-fs': { enabled: false },
    }),
  ],
  serviceName: 'order-api',
});

sdk.start();

// Manual span for custom operations
import { trace } from '@opentelemetry/api';

const tracer = trace.getTracer('order-api');

async function processOrder(orderId: string) {
  return tracer.startActiveSpan('processOrder', async (span) => {
    span.setAttribute('order.id', orderId);
    try {
      await validateOrder(orderId);
      await chargePayment(orderId);
      await notifyShipping(orderId);
      span.setStatus({ code: SpanStatusCode.OK });
    } catch (err) {
      span.setStatus({ code: SpanStatusCode.ERROR, message: err.message });
      span.recordException(err);
      throw err;
    } finally {
      span.end();
    }
  });
}
```

### Log Aggregation

```yaml
# Typical production logging stack:
# Application → Fluentd/Fluent Bit/Vector → Elasticsearch/Loki → Grafana/Kibana

# Fluent Bit config — ship JSON logs to Loki
[INPUT]
    Name              tail
    Path              /var/log/app/*.log
    Parser            json
    Tag               app.*
    Refresh_Interval  5

[OUTPUT]
    Name   loki
    Match  app.*
    Host   loki.monitoring.svc
    Port   3100
    Labels job=app, env=production

# Useful log queries (Loki LogQL):
# All errors in the last hour:
{job="app"} |= "error" | json | level="error"

# Specific request by correlation ID:
{job="app"} | json | requestId="abc-123-def"

# Error rate over time:
sum(rate({job="app"} | json | level="error" [5m])) by (service)
```

### Core Dumps

```bash
# Enable core dumps (Linux)
ulimit -c unlimited
echo '/tmp/core.%e.%p' | sudo tee /proc/sys/kernel/core_pattern

# Trigger core dump from a running process
kill -ABRT <pid>   # sends SIGABRT
gcore <pid>        # generates core dump without killing process

# Analyze with GDB
gdb /path/to/binary /tmp/core.myapp.12345
(gdb) bt           # full backtrace
(gdb) bt full      # backtrace with local variables
(gdb) thread apply all bt  # backtraces for all threads
(gdb) info threads          # list all threads
(gdb) frame 3               # switch to frame 3
(gdb) print myVariable      # inspect variable value

# For Node.js core dumps
npm install -g llnode
llnode -c /tmp/core.node.12345
(llnode) v8 bt     # V8-aware backtrace
(llnode) v8 findjsobjects  # find JS objects on the heap
```

### strace / dtrace

```bash
# strace (Linux) — trace system calls
strace -p <pid>                    # attach to running process
strace -p <pid> -e trace=network   # only network syscalls
strace -p <pid> -e trace=file      # only file operations
strace -p <pid> -c                 # summary of syscall counts and times
strace -f -p <pid>                 # follow child processes/threads

# Common patterns:
strace -e openat myapp 2>&1 | grep "No such file"  # find missing files
strace -e connect myapp 2>&1                         # see outbound connections
strace -T -e read,write -p <pid>                     # show time per syscall

# dtrace (macOS/BSD/Solaris) — dynamic tracing
# Trace all syscalls by process name
sudo dtrace -n 'syscall:::entry /execname == "node"/ { @[probefunc] = count(); }'

# Trace file opens
sudo dtrace -n 'syscall::open*:entry { printf("%s %s", execname, copyinstr(arg0)); }'
```

### eBPF

```bash
# eBPF (Linux) — programmable kernel tracing with minimal overhead
# Requires: bpftrace or bcc-tools

# Trace TCP connections by process
sudo bpftrace -e 'kprobe:tcp_connect { printf("connect: %s pid=%d\n", comm, pid); }'

# Trace file opens
sudo bpftrace -e 'tracepoint:syscalls:sys_enter_openat { printf("%s %s\n", comm, str(args.filename)); }'

# Measure read latency distribution
sudo bpftrace -e '
kprobe:vfs_read { @start[tid] = nsecs; }
kretprobe:vfs_read /@start[tid]/ {
  @us = hist((nsecs - @start[tid]) / 1000);
  delete(@start[tid]);
}'

# bcc-tools — prebuilt eBPF tools
sudo tcplife           # trace TCP connections with duration
sudo biolatency        # disk I/O latency histogram
sudo funccount 'go:*'  # count Go runtime function calls
sudo offcputime -p <pid> 5  # trace off-CPU time (blocked threads)
```

### Anti-Patterns

```
AVOID: Debugging production by adding code and redeploying
  Use observability tools (tracing, metrics, logs) that are
  already in place. Add them proactively, not reactively.

AVOID: SSH-ing into production containers to debug
  Containers are ephemeral. Use log aggregation, distributed
  tracing, and remote debugging ports (secured behind VPN).

AVOID: Leaving debug endpoints exposed
  /debug/pprof, /debug/vars, and similar endpoints must be
  protected by authentication or bound to localhost only.

AVOID: Ignoring signal handling
  SIGTERM should trigger graceful shutdown. SIGUSR1 can enable
  debug logging. Plan your signal handling before production.

AVOID: Running strace/dtrace without understanding overhead
  strace adds significant latency (~2-5x). Use it briefly on
  a single process, not across the fleet.
```

---

## 8. Common Patterns Quick Reference

| Bug Type | Symptoms | Primary Tool | Fix Strategy |
|---|---|---|---|
| **Null/undefined access** | TypeError in stack trace | Debugger breakpoint at crash site | Add null checks, use optional chaining, validate inputs |
| **Off-by-one** | Wrong number of iterations, missing first/last element | Unit test with edge cases (empty, 1, 2, N) | Verify loop bounds, use inclusive/exclusive consistently |
| **Memory leak** | RSS grows over hours/days, OOM kills | Heap snapshots (3-snapshot technique), pprof | Find and remove unbounded caches, detached DOM, leaked listeners |
| **Race condition** | Intermittent failures, data corruption under load | Go `-race`, TSan, stress testing | Add locks, use atomic operations, apply consistent ordering |
| **Deadlock** | Process hangs, 100% CPU on lock spin, zero throughput | Thread dump (`kill -3`), goroutine dump, `jstack` | Enforce lock ordering, use timeouts, reduce lock scope |
| **N+1 query** | Page loads slowly, hundreds of DB queries per request | Query logging, APM slow query panel | Eager loading (JOIN/include), batching, DataLoader |
| **Infinite loop** | 100% CPU, UI frozen, request timeout | CPU profiler, `top`/`htop`, break in debugger | Add termination condition, set iteration limit |
| **Resource exhaustion** | "Too many open files", connection pool empty | `lsof`, `ss`, connection pool metrics | Close resources in `finally`, increase limits, add pooling |
| **Encoding issue** | Garbled text, mojibake, wrong character display | Hex dump of bytes, check HTTP Content-Type header | Use UTF-8 everywhere, set charset in DB/HTTP/files |
| **Timezone bug** | Events show at wrong time, off by N hours | Log timestamps in UTC, compare server/client clocks | Store UTC, convert at display layer, use libraries (date-fns-tz) |
| **SSL/TLS failure** | ECONNREFUSED, certificate errors, handshake failure | `openssl s_client`, `curl -v`, check cert expiry | Renew certs, fix CA chain, match hostname, check clock skew |
| **DNS resolution** | Intermittent connection failures, slow first request | `dig`, `nslookup`, `resolvectl status` | Check /etc/resolv.conf, flush cache, reduce TTL during migration |
| **Permission denied** | EACCES, 403, operation not permitted | `ls -la`, check user/group, `strace -e openat` | Fix file ownership/permissions, check SELinux/AppArmor |
| **Serialization bug** | Data looks correct in DB but wrong in API response | Log raw bytes before/after serialization | Check field naming (camelCase vs snake_case), date format, BigInt handling |
| **Event loop blocking** | High P99 latency, unresponsive health checks | `--prof`, clinic.js bubbles, blocked-at package | Move CPU work to worker threads, chunk large operations |
| **Connection leak** | Pool exhausted, "too many connections" error | Pool metrics, `pg_stat_activity`, `SHOW PROCESSLIST` | Always release in `finally`, set pool `max` and `idleTimeout` |

### Debugging Checklist

```
Before you start:
  [ ] Can I reproduce the bug consistently?
  [ ] Do I have the exact error message and stack trace?
  [ ] Do I know when it started (commit, deploy, config change)?
  [ ] Is this the simplest reproduction case?

During investigation:
  [ ] Am I changing only one variable at a time?
  [ ] Am I writing down my hypotheses and test results?
  [ ] Have I checked the most recent changes first?
  [ ] Have I read the actual error message (not just the first line)?

After fixing:
  [ ] Does the original reproduction case now pass?
  [ ] Have I written a regression test?
  [ ] Did I check for similar bugs elsewhere in the codebase?
  [ ] Have I documented the root cause for the team?
```

### When to Escalate

```
Escalate to a teammate or senior engineer when:
  - You have spent more than 2 hours without a new hypothesis
  - The bug requires domain knowledge you don't have
  - The fix requires changes to infrastructure you don't own
  - You suspect a compiler/runtime/OS bug (very rare, but real)
  - The bug involves security implications (data exposure, auth bypass)

Before escalating, prepare:
  1. Clear description of the symptom
  2. Minimal reproduction steps
  3. List of hypotheses you tested and ruled out
  4. Relevant logs, traces, and profiler output
```
