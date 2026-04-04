---
name: golang-pro
description: Go development patterns: goroutines, channels, context propagation, interfaces, error handling, testing, modules, and high-performance service design.
---

# Golang Pro

Expert-level Go development skill covering concurrency, interface design, error handling, generics, testing, module management, performance tuning, and production-ready service patterns. Use this when writing, reviewing, or architecting Go code that needs to be idiomatic, performant, and maintainable.

## Table of Contents

- [Concurrency Patterns](#concurrency-patterns)
- [Interface Design](#interface-design)
- [Error Handling](#error-handling)
- [Generics](#generics)
- [Testing and Benchmarking](#testing-and-benchmarking)
- [Module System and Project Layout](#module-system-and-project-layout)
- [Performance](#performance)
- [Production Patterns](#production-patterns)

---

## Concurrency Patterns

### Goroutines and Channels

Launch goroutines for concurrent work. Use channels for communication between them.

```go
// Fan-out: distribute work across multiple goroutines
func fanOut(jobs <-chan int, workers int) <-chan int {
    results := make(chan int, workers)
    var wg sync.WaitGroup
    for i := 0; i < workers; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            for job := range jobs {
                results <- process(job)
            }
        }()
    }
    go func() {
        wg.Wait()
        close(results)
    }()
    return results
}
```

### Select and Context Cancellation

Always use `context.Context` to propagate cancellation and deadlines through call chains.

```go
func fetchWithTimeout(ctx context.Context, url string) ([]byte, error) {
    ctx, cancel := context.WithTimeout(ctx, 5*time.Second)
    defer cancel()

    req, err := http.NewRequestWithContext(ctx, http.MethodGet, url, nil)
    if err != nil {
        return nil, fmt.Errorf("creating request: %w", err)
    }
    resp, err := http.DefaultClient.Do(req)
    if err != nil {
        return nil, fmt.Errorf("executing request: %w", err)
    }
    defer resp.Body.Close()
    return io.ReadAll(resp.Body)
}

// Select with cancellation
func worker(ctx context.Context, jobs <-chan Job, results chan<- Result) {
    for {
        select {
        case <-ctx.Done():
            return
        case job, ok := <-jobs:
            if !ok {
                return
            }
            results <- job.Execute()
        }
    }
}
```

### Errgroup for Coordinated Goroutines

`errgroup` collects errors from goroutines and cancels siblings on first failure.

```go
func fetchAll(ctx context.Context, urls []string) ([]Response, error) {
    g, ctx := errgroup.WithContext(ctx)
    responses := make([]Response, len(urls))

    for i, url := range urls {
        i, url := i, url // capture loop vars (unnecessary in Go 1.22+)
        g.Go(func() error {
            resp, err := fetchWithTimeout(ctx, url)
            if err != nil {
                return fmt.Errorf("fetching %s: %w", url, err)
            }
            responses[i] = resp
            return nil
        })
    }
    if err := g.Wait(); err != nil {
        return nil, err
    }
    return responses, nil
}
```

### Pipeline Pattern

Chain processing stages via channels for streaming data transformation.

```go
func pipeline(ctx context.Context, input <-chan int) <-chan int {
    stage1 := transform(ctx, input, func(v int) int { return v * 2 })
    stage2 := transform(ctx, stage1, func(v int) int { return v + 1 })
    return stage2
}

func transform(ctx context.Context, in <-chan int, fn func(int) int) <-chan int {
    out := make(chan int)
    go func() {
        defer close(out)
        for v := range in {
            select {
            case <-ctx.Done():
                return
            case out <- fn(v):
            }
        }
    }()
    return out
}
```

### Worker Pool

Bounded concurrency with a fixed number of workers.

```go
func workerPool(ctx context.Context, tasks []Task, maxWorkers int) []Result {
    jobs := make(chan Task, len(tasks))
    results := make(chan Result, len(tasks))

    // Start workers
    var wg sync.WaitGroup
    for i := 0; i < maxWorkers; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            for task := range jobs {
                select {
                case <-ctx.Done():
                    return
                case results <- task.Run():
                }
            }
        }()
    }

    // Send jobs
    for _, t := range tasks {
        jobs <- t
    }
    close(jobs)

    // Collect
    go func() {
        wg.Wait()
        close(results)
    }()

    var out []Result
    for r := range results {
        out = append(out, r)
    }
    return out
}
```

**Anti-patterns to avoid:**
- Launching unbounded goroutines without backpressure -- always use a worker pool or semaphore.
- Forgetting to close channels, causing goroutine leaks.
- Using `time.Sleep` instead of `context.WithTimeout` for deadline control.
- Sharing memory without synchronization; prefer channels or `sync.Mutex`.

---

## Interface Design

### Implicit Interfaces and Small Interfaces

Go interfaces are satisfied implicitly. Keep them small -- one or two methods.

```go
// Good: small, focused interface
type Reader interface {
    Read(p []byte) (n int, err error)
}

type Writer interface {
    Write(p []byte) (n int, err error)
}

// Compose larger interfaces from small ones
type ReadWriter interface {
    Reader
    Writer
}
```

### Accept Interfaces, Return Structs

Define interfaces at the call site, not the implementation site. Return concrete types.

```go
// Good: function accepts an interface, returns a concrete type
type Store interface {
    Get(ctx context.Context, key string) ([]byte, error)
}

func NewCachedStore(backend Store, ttl time.Duration) *CachedStore {
    return &CachedStore{backend: backend, ttl: ttl}
}

// Anti-pattern: exporting interfaces from the package that defines them
// when there is only one implementation. Define interfaces where consumed.
```

### Type Assertions and Type Switches

Use type switches for polymorphic behavior when interface methods are insufficient.

```go
func describe(i interface{}) string {
    switch v := i.(type) {
    case string:
        return fmt.Sprintf("string of length %d", len(v))
    case int:
        return fmt.Sprintf("integer %d", v)
    case error:
        return fmt.Sprintf("error: %s", v.Error())
    default:
        return fmt.Sprintf("unknown type: %T", v)
    }
}

// Optional interface pattern: check for capability at runtime
type Flusher interface {
    Flush() error
}

func writeAndFlush(w io.Writer, data []byte) error {
    if _, err := w.Write(data); err != nil {
        return err
    }
    if f, ok := w.(Flusher); ok {
        return f.Flush()
    }
    return nil
}
```

### Common Interface Patterns

```go
// sort.Interface
type ByAge []Person

func (a ByAge) Len() int           { return len(a) }
func (a ByAge) Less(i, j int) bool { return a[i].Age < a[j].Age }
func (a ByAge) Swap(i, j int)      { a[i], a[j] = a[j], a[i] }

// Stringer
func (p Person) String() string {
    return fmt.Sprintf("%s (age %d)", p.Name, p.Age)
}

// io.Reader implementation
type CountingReader struct {
    r     io.Reader
    count int64
}

func (cr *CountingReader) Read(p []byte) (int, error) {
    n, err := cr.r.Read(p)
    cr.count += int64(n)
    return n, err
}
```

**Anti-patterns to avoid:**
- Interfaces with more than 5 methods -- split them.
- Defining interfaces in the implementing package when only one implementation exists.
- Using `interface{}` / `any` when a concrete type or a narrower interface works.

---

## Error Handling

### Error Wrapping with %w

Wrap errors to add context while preserving the original for inspection.

```go
func readConfig(path string) (*Config, error) {
    data, err := os.ReadFile(path)
    if err != nil {
        return nil, fmt.Errorf("reading config %s: %w", path, err)
    }
    var cfg Config
    if err := json.Unmarshal(data, &cfg); err != nil {
        return nil, fmt.Errorf("parsing config %s: %w", path, err)
    }
    return &cfg, nil
}
```

### Custom Error Types

Define domain-specific error types for programmatic error handling.

```go
type NotFoundError struct {
    Resource string
    ID       string
}

func (e *NotFoundError) Error() string {
    return fmt.Sprintf("%s %q not found", e.Resource, e.ID)
}

// Sentinel errors for fixed conditions
var (
    ErrUnauthorized = errors.New("unauthorized")
    ErrRateLimited  = errors.New("rate limited")
)

// Use errors.Is for sentinels, errors.As for typed errors
func handleError(err error) {
    if errors.Is(err, ErrUnauthorized) {
        // handle auth failure
        return
    }
    var nfe *NotFoundError
    if errors.As(err, &nfe) {
        log.Printf("resource not found: %s %s", nfe.Resource, nfe.ID)
        return
    }
    log.Printf("unexpected error: %v", err)
}
```

### Error Handling in Goroutines

Never let goroutine panics crash the whole program. Use recover or errgroup.

```go
func safeGo(fn func() error) <-chan error {
    ch := make(chan error, 1)
    go func() {
        defer func() {
            if r := recover(); r != nil {
                ch <- fmt.Errorf("panic recovered: %v\n%s", r, debug.Stack())
            }
        }()
        ch <- fn()
    }()
    return ch
}
```

### Panic/Recover Patterns

Reserve `panic` for truly unrecoverable states. Use `recover` at API boundaries.

```go
// HTTP middleware that recovers panics
func recoveryMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        defer func() {
            if rec := recover(); rec != nil {
                log.Printf("panic: %v\n%s", rec, debug.Stack())
                http.Error(w, "internal server error", http.StatusInternalServerError)
            }
        }()
        next.ServeHTTP(w, r)
    })
}
```

**Anti-patterns to avoid:**
- Using `fmt.Errorf` with `%v` instead of `%w` when the caller needs to inspect the cause.
- Logging an error and returning it (leads to duplicate log entries).
- Using `panic` for expected error conditions like invalid input or missing records.
- Ignoring errors with `_` -- handle or explicitly document why it is safe to ignore.

---

## Generics

### Type Parameters and Constraints

Use generics to eliminate repetitive type-specific functions.

```go
// Generic map function
func Map[T any, U any](slice []T, fn func(T) U) []U {
    result := make([]U, len(slice))
    for i, v := range slice {
        result[i] = fn(v)
    }
    return result
}

// Generic filter
func Filter[T any](slice []T, predicate func(T) bool) []T {
    var result []T
    for _, v := range slice {
        if predicate(v) {
            result = append(result, v)
        }
    }
    return result
}

// Constrained type: comparable + ordered
func Max[T cmp.Ordered](a, b T) T {
    if a > b {
        return a
    }
    return b
}
```

### Generic Data Structures

```go
// Thread-safe generic cache
type Cache[K comparable, V any] struct {
    mu    sync.RWMutex
    items map[K]V
}

func NewCache[K comparable, V any]() *Cache[K, V] {
    return &Cache[K, V]{items: make(map[K]V)}
}

func (c *Cache[K, V]) Get(key K) (V, bool) {
    c.mu.RLock()
    defer c.mu.RUnlock()
    v, ok := c.items[key]
    return v, ok
}

func (c *Cache[K, V]) Set(key K, value V) {
    c.mu.Lock()
    defer c.mu.Unlock()
    c.items[key] = value
}
```

### When to Use Generics vs Interfaces

- **Use generics** when the logic is identical across types and you need type safety (collections, data structures, utility functions).
- **Use interfaces** when behavior varies by type and you need polymorphism (strategy pattern, dependency injection).
- **Avoid generics** for one or two concrete types -- just write the concrete code.

**Anti-patterns to avoid:**
- Over-genericizing code that only operates on one type.
- Using `any` constraint when a narrower constraint (`comparable`, `cmp.Ordered`) works.
- Complex nested type constraints that hurt readability -- simplify or use interfaces instead.

---

## Testing and Benchmarking

### Table-Driven Tests

The idiomatic Go testing pattern. Each case is a row in a table.

```go
func TestAdd(t *testing.T) {
    tests := []struct {
        name     string
        a, b     int
        expected int
    }{
        {"positive numbers", 2, 3, 5},
        {"negative numbers", -1, -2, -3},
        {"zero", 0, 0, 0},
        {"mixed signs", -1, 5, 4},
    }
    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            got := Add(tt.a, tt.b)
            if got != tt.expected {
                t.Errorf("Add(%d, %d) = %d, want %d", tt.a, tt.b, got, tt.expected)
            }
        })
    }
}
```

### HTTP Testing with httptest

```go
func TestGetUser(t *testing.T) {
    // Create a test server
    handler := NewRouter()
    srv := httptest.NewServer(handler)
    defer srv.Close()

    resp, err := http.Get(srv.URL + "/api/v1/users/123")
    if err != nil {
        t.Fatalf("request failed: %v", err)
    }
    defer resp.Body.Close()

    if resp.StatusCode != http.StatusOK {
        t.Errorf("status = %d, want %d", resp.StatusCode, http.StatusOK)
    }

    var user User
    if err := json.NewDecoder(resp.Body).Decode(&user); err != nil {
        t.Fatalf("decode failed: %v", err)
    }
    if user.ID != "123" {
        t.Errorf("user.ID = %q, want %q", user.ID, "123")
    }
}
```

### Testify for Assertions

```go
import (
    "github.com/stretchr/testify/assert"
    "github.com/stretchr/testify/require"
)

func TestService(t *testing.T) {
    svc := NewService(mockStore{})

    result, err := svc.Process(context.Background(), "input")
    require.NoError(t, err)             // fail immediately if error
    assert.Equal(t, "expected", result)  // soft assertion, test continues
    assert.Len(t, result.Items, 3)
    assert.Contains(t, result.Tags, "go")
}
```

### Benchmarking

```go
func BenchmarkProcess(b *testing.B) {
    svc := NewService()
    input := generateTestInput()

    b.ResetTimer()
    for i := 0; i < b.N; i++ {
        svc.Process(input)
    }
}

// Run: go test -bench=BenchmarkProcess -benchmem ./...
// Output:
// BenchmarkProcess-8  50000  23456 ns/op  1024 B/op  12 allocs/op
```

### Fuzz Testing

```go
func FuzzParseConfig(f *testing.F) {
    // Seed corpus
    f.Add([]byte(`{"port": 8080}`))
    f.Add([]byte(`{}`))
    f.Add([]byte(`invalid`))

    f.Fuzz(func(t *testing.T, data []byte) {
        cfg, err := ParseConfig(data)
        if err != nil {
            return // invalid input is fine
        }
        // If parsing succeeded, re-marshal and parse again
        out, err := json.Marshal(cfg)
        if err != nil {
            t.Fatalf("marshal failed: %v", err)
        }
        cfg2, err := ParseConfig(out)
        if err != nil {
            t.Fatalf("round-trip failed: %v", err)
        }
        if !reflect.DeepEqual(cfg, cfg2) {
            t.Error("round-trip mismatch")
        }
    })
}

// Run: go test -fuzz=FuzzParseConfig ./...
```

### Race Detection and Coverage

```bash
# Run tests with race detector
go test -race ./...

# Generate coverage report
go test -coverprofile=coverage.out ./...
go tool cover -html=coverage.out -o coverage.html

# Coverage by function
go tool cover -func=coverage.out
```

**Anti-patterns to avoid:**
- Testing implementation details instead of behavior.
- Not using `t.Parallel()` for independent tests.
- Hardcoding ports in tests -- use `httptest.NewServer` or port 0.
- Skipping race detection in CI -- always run with `-race`.

---

## Module System and Project Layout

### go.mod and Dependency Management

```bash
# Initialize a module
go mod init github.com/yourorg/yourproject

# Add a dependency
go get github.com/lib/pq@v1.10.9

# Tidy: remove unused, add missing
go mod tidy

# Vendor dependencies
go mod vendor

# Upgrade all dependencies
go get -u ./...

# Check for known vulnerabilities
govulncheck ./...
```

### Workspace Mode for Multi-Module Repos

```bash
# Initialize workspace
go work init ./service-a ./service-b ./shared

# go.work file
# go 1.22
# use (
#     ./service-a
#     ./service-b
#     ./shared
# )
```

### Standard Project Layout

```
myproject/
  cmd/
    server/           # main.go for the server binary
    cli/              # main.go for the CLI binary
  internal/           # Private packages, not importable externally
    auth/
    store/
    handler/
  pkg/                # Public library code (optional, use sparingly)
  api/                # OpenAPI specs, protobuf definitions
  configs/            # Configuration file templates
  scripts/            # Build and CI scripts
  go.mod
  go.sum
  Makefile
```

### Internal Packages

The `internal/` directory enforces import boundaries at the compiler level.

```
github.com/yourorg/myproject/internal/store
// Can only be imported by code rooted at github.com/yourorg/myproject
```

**Anti-patterns to avoid:**
- Putting everything in `package main` -- extract packages early.
- Using `pkg/` as a dumping ground -- only use it for truly reusable library code.
- Committing `go.sum` changes without running `go mod tidy`.
- Pinning to branch names instead of tagged versions.

---

## Performance

### Profiling with pprof

```go
import _ "net/http/pprof"

func main() {
    // Expose pprof endpoint (do NOT expose in production without auth)
    go func() {
        log.Println(http.ListenAndServe("localhost:6060", nil))
    }()
    // ... rest of application
}
```

```bash
# CPU profile
go tool pprof http://localhost:6060/debug/pprof/profile?seconds=30

# Heap profile
go tool pprof http://localhost:6060/debug/pprof/heap

# Goroutine dump
curl http://localhost:6060/debug/pprof/goroutine?debug=2
```

### Memory Allocation Optimization

```go
// Pre-allocate slices when length is known
func processItems(items []Item) []Result {
    results := make([]Result, 0, len(items)) // set capacity
    for _, item := range items {
        results = append(results, transform(item))
    }
    return results
}

// Use strings.Builder for string concatenation
func buildQuery(fields []string) string {
    var b strings.Builder
    b.Grow(256) // pre-allocate estimated capacity
    b.WriteString("SELECT ")
    for i, f := range fields {
        if i > 0 {
            b.WriteString(", ")
        }
        b.WriteString(f)
    }
    b.WriteString(" FROM records")
    return b.String()
}
```

### sync.Pool for Reducing GC Pressure

```go
var bufPool = sync.Pool{
    New: func() interface{} {
        return new(bytes.Buffer)
    },
}

func processRequest(data []byte) string {
    buf := bufPool.Get().(*bytes.Buffer)
    defer func() {
        buf.Reset()
        bufPool.Put(buf)
    }()

    buf.Write(data)
    // ... process buffer
    return buf.String()
}
```

### Escape Analysis

```bash
# See what escapes to the heap
go build -gcflags="-m" ./...

# Verbose escape analysis
go build -gcflags="-m -m" ./...
```

```go
// Avoid unnecessary heap allocations
// Bad: pointer escapes to heap
func newValue() *int {
    v := 42
    return &v // v escapes to heap
}

// Better: let caller decide allocation
func computeValue() int {
    return 42 // stays on stack
}
```

**Anti-patterns to avoid:**
- Premature optimization without profiling first -- always measure.
- Using `sync.Pool` for small objects where GC handles them efficiently.
- Ignoring `benchmem` output -- allocation count often matters more than speed.
- String concatenation with `+` in loops instead of `strings.Builder`.

---

## Production Patterns

### Graceful Shutdown

```go
func main() {
    srv := &http.Server{Addr: ":8080", Handler: newRouter()}

    // Start server in goroutine
    go func() {
        if err := srv.ListenAndServe(); err != nil && err != http.ErrServerClosed {
            log.Fatalf("server error: %v", err)
        }
    }()

    // Wait for interrupt signal
    quit := make(chan os.Signal, 1)
    signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM)
    <-quit

    log.Println("shutting down server...")
    ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
    defer cancel()

    if err := srv.Shutdown(ctx); err != nil {
        log.Fatalf("forced shutdown: %v", err)
    }
    log.Println("server stopped")
}
```

### Health Checks

```go
func healthHandler(db *sql.DB) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        ctx, cancel := context.WithTimeout(r.Context(), 2*time.Second)
        defer cancel()

        status := map[string]string{"status": "ok"}
        if err := db.PingContext(ctx); err != nil {
            status["status"] = "degraded"
            status["db"] = err.Error()
            w.WriteHeader(http.StatusServiceUnavailable)
        }
        w.Header().Set("Content-Type", "application/json")
        json.NewEncoder(w).Encode(status)
    }
}
```

### Structured Logging with slog

```go
import "log/slog"

func main() {
    logger := slog.New(slog.NewJSONHandler(os.Stdout, &slog.HandlerOptions{
        Level: slog.LevelInfo,
    }))
    slog.SetDefault(logger)

    slog.Info("server starting",
        "port", 8080,
        "env", "production",
    )
}

// Add context to logger and pass through request chain
func requestLogger(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        logger := slog.With(
            "request_id", r.Header.Get("X-Request-ID"),
            "method", r.Method,
            "path", r.URL.Path,
        )
        ctx := context.WithValue(r.Context(), loggerKey, logger)
        next.ServeHTTP(w, r.WithContext(ctx))
    })
}
```

### Configuration Management

```go
type Config struct {
    Port        int           `env:"PORT" default:"8080"`
    DatabaseURL string        `env:"DATABASE_URL,required"`
    LogLevel    string        `env:"LOG_LEVEL" default:"info"`
    Timeout     time.Duration `env:"TIMEOUT" default:"30s"`
}

// Load from environment with validation
func LoadConfig() (*Config, error) {
    cfg := &Config{}
    if err := envconfig.Process("APP", cfg); err != nil {
        return nil, fmt.Errorf("loading config: %w", err)
    }
    if cfg.Port < 1 || cfg.Port > 65535 {
        return nil, fmt.Errorf("invalid port: %d", cfg.Port)
    }
    return cfg, nil
}
```

### Middleware Chains

```go
type Middleware func(http.Handler) http.Handler

func Chain(middlewares ...Middleware) Middleware {
    return func(final http.Handler) http.Handler {
        for i := len(middlewares) - 1; i >= 0; i-- {
            final = middlewares[i](final)
        }
        return final
    }
}

// Usage
func newRouter() http.Handler {
    mux := http.NewServeMux()
    mux.HandleFunc("GET /api/v1/users", listUsers)
    mux.HandleFunc("GET /api/v1/users/{id}", getUser)

    stack := Chain(
        recoveryMiddleware,
        requestLogger,
        corsMiddleware,
        rateLimitMiddleware,
    )
    return stack(mux)
}
```

### Dependency Injection

Prefer constructor injection with explicit dependencies. No magic.

```go
type Server struct {
    store  Store
    cache  Cache
    logger *slog.Logger
}

func NewServer(store Store, cache Cache, logger *slog.Logger) *Server {
    return &Server{
        store:  store,
        cache:  cache,
        logger: logger,
    }
}

// Wire it up in main
func main() {
    db := connectDB()
    store := postgres.NewStore(db)
    cache := redis.NewCache(redisClient)
    logger := slog.New(slog.NewJSONHandler(os.Stdout, nil))

    srv := NewServer(store, cache, logger)
    // ...
}
```

**Anti-patterns to avoid:**
- No graceful shutdown -- connections drop, data corrupts.
- Using `log.Fatal` deep in library code -- it calls `os.Exit` and skips deferred cleanup.
- Global mutable state for configuration -- use struct fields and constructor injection.
- Missing health checks -- orchestrators (Kubernetes) need them to manage restarts.
- Unstructured log output in production -- always use structured logging for observability.
