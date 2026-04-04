---
name: rust-engineer
description: Rust development: ownership, lifetimes, traits, async patterns, error handling with thiserror/anyhow, unsafe code review, and performance-critical system design.
---

# Rust Engineer

Expert-level Rust development skill covering ownership, lifetimes, trait design, error handling, async programming, unsafe code, and performance optimization. Use this when writing, reviewing, or architecting Rust code that needs to be safe, idiomatic, and performant.

## Table of Contents

- [Ownership and Borrowing](#ownership-and-borrowing)
- [Lifetimes](#lifetimes)
- [Traits and Generics](#traits-and-generics)
- [Error Handling](#error-handling)
- [Async Rust](#async-rust)
- [Unsafe Rust](#unsafe-rust)
- [Performance Patterns](#performance-patterns)
- [Common Patterns Quick Reference](#common-patterns-quick-reference)

---

## Ownership and Borrowing

### Ownership Rules

Every value in Rust has exactly one owner. When the owner goes out of scope, the value is dropped. Transfers of ownership are called moves.

```rust
fn main() {
    let s1 = String::from("hello");
    let s2 = s1; // s1 is moved into s2; s1 is no longer valid

    // println!("{s1}"); // ERROR: value borrowed after move

    // Clone when you need independent copies
    let s3 = s2.clone();
    println!("{s2} and {s3}"); // both valid
}
```

### Borrowing and References

Borrow a value with `&` (shared) or `&mut` (exclusive). The borrow checker enforces: many shared references OR one mutable reference, never both at the same time.

```rust
fn calculate_length(s: &str) -> usize {
    s.len() // borrows s, does not take ownership
}

fn append_world(s: &mut String) {
    s.push_str(", world");
}

fn main() {
    let mut greeting = String::from("hello");
    let len = calculate_length(&greeting);
    append_world(&mut greeting);
    println!("{greeting} (originally {len} bytes)");
}
```

### Mutable References and Scoping

The borrow checker tracks reference lifetimes lexically in older editions and via NLL (Non-Lexical Lifetimes) since Rust 2018.

```rust
fn main() {
    let mut data = vec![1, 2, 3];

    let first = &data[0]; // shared borrow starts
    println!("first element: {first}"); // shared borrow ends here (NLL)

    data.push(4); // mutable borrow is fine -- shared borrow already ended

    // Common fix: restructure code so shared borrows end before mutation
}
```

### Returning References from Functions

Functions that return references must ensure the referent outlives the reference. The compiler uses lifetime elision rules to infer this in simple cases.

```rust
// Elided lifetime: compiler infers fn first_word<'a>(s: &'a str) -> &'a str
fn first_word(s: &str) -> &str {
    let bytes = s.as_bytes();
    for (i, &byte) in bytes.iter().enumerate() {
        if byte == b' ' {
            return &s[..i];
        }
    }
    s
}
```

### Common Borrow Checker Errors and Fixes

```rust
// ERROR: cannot borrow `items` as mutable because it is also borrowed as immutable
fn bad_example(items: &mut Vec<String>) {
    // let first = &items[0];
    // items.push(String::from("new")); // invalidates `first`
    // println!("{first}");

    // FIX: clone the value you need before mutating
    let first = items[0].clone();
    items.push(String::from("new"));
    println!("{first}");
}

// ERROR: cannot return reference to local variable
// fn dangling() -> &str {
//     let s = String::from("hello");
//     &s // s is dropped at end of function
// }

// FIX: return the owned value
fn not_dangling() -> String {
    String::from("hello")
}
```

**Anti-patterns to avoid:**
- Cloning everything to appease the borrow checker -- restructure code first, clone as a last resort.
- Holding references across `.await` points or long-lived scopes when a clone or index would suffice.
- Using `Rc<RefCell<T>>` as a default -- it is a code smell in most cases; prefer ownership restructuring.
- Ignoring the borrow checker's suggestions -- they almost always point to a real design issue.

---

## Lifetimes

### Explicit Lifetime Annotations

When the compiler cannot infer how long references live, annotate them explicitly. Lifetime annotations describe relationships -- they do not change how long values live.

```rust
// The returned reference lives as long as the shorter of `x` and `y`
fn longest<'a>(x: &'a str, y: &'a str) -> &'a str {
    if x.len() > y.len() { x } else { y }
}

fn main() {
    let result;
    let string1 = String::from("long string");
    {
        let string2 = String::from("xyz");
        result = longest(string1.as_str(), string2.as_str());
        println!("longest: {result}"); // OK: both string1 and string2 are alive
    }
    // println!("{result}"); // ERROR if uncommented: string2 does not live long enough
}
```

### Lifetimes in Structs

A struct that holds references must declare lifetime parameters. The struct cannot outlive the data it borrows.

```rust
#[derive(Debug)]
struct Excerpt<'a> {
    text: &'a str,
    line: usize,
}

impl<'a> Excerpt<'a> {
    fn new(text: &'a str, line: usize) -> Self {
        Self { text, line }
    }

    // Method returning a reference tied to the struct's lifetime
    fn first_word(&self) -> &str {
        self.text.split_whitespace().next().unwrap_or("")
    }
}

fn main() {
    let novel = String::from("Call me Ishmael. Some years ago...");
    let excerpt = Excerpt::new(&novel[..16], 1);
    println!("{:?}, first word: {}", excerpt, excerpt.first_word());
}
```

### Lifetime Bounds on Generics

Combine lifetimes with generic type parameters when the generic must outlive a reference.

```rust
use std::fmt::Display;

fn announce_and_return<'a, T: Display>(announcement: T, value: &'a str) -> &'a str {
    println!("Announcement: {announcement}");
    value
}

// Lifetime bound: T must live at least as long as 'a
struct Wrapper<'a, T: 'a> {
    value: &'a T,
}
```

### The 'static Lifetime

`'static` means the reference can live for the entire duration of the program. String literals are `'static`. Use `'static` bounds sparingly.

```rust
// String literals have 'static lifetime
let s: &'static str = "I live forever";

// Common pattern: trait objects that must be Send + 'static for spawning
fn spawn_task(task: Box<dyn FnOnce() + Send + 'static>) {
    std::thread::spawn(move || task());
}

// Owned data satisfies 'static -- it does not contain non-static references
fn process<T: Send + 'static>(value: T) {
    std::thread::spawn(move || {
        println!("processing in thread");
        drop(value);
    });
}
```

### Higher-Ranked Trait Bounds (HRTBs)

Use `for<'a>` when a closure or function must work for any lifetime, not a specific one.

```rust
// The closure must accept a reference with any lifetime
fn apply_to_ref<F>(f: F, value: &str) -> usize
where
    F: for<'a> Fn(&'a str) -> usize,
{
    f(value)
}

fn main() {
    let counter = apply_to_ref(|s| s.len(), "hello");
    println!("length: {counter}");
}
```

**Anti-patterns to avoid:**
- Sprinkling `'static` everywhere to silence lifetime errors -- it restricts your API unnecessarily.
- Adding lifetime parameters to structs that could own their data instead (`String` vs `&str`).
- Over-annotating lifetimes when elision rules already handle the case.
- Fighting the borrow checker with `unsafe` transmute to extend lifetimes -- this is undefined behavior.

---

## Traits and Generics

### Defining and Implementing Traits

Traits define shared behavior. Implement them for your types to enable polymorphism.

```rust
trait Summary {
    fn summarize(&self) -> String;

    // Default implementation
    fn preview(&self) -> String {
        format!("{}...", &self.summarize()[..20.min(self.summarize().len())])
    }
}

struct Article {
    title: String,
    content: String,
}

impl Summary for Article {
    fn summarize(&self) -> String {
        format!("{}: {}", self.title, &self.content[..100.min(self.content.len())])
    }
}
```

### Trait Objects vs Generics (Static vs Dynamic Dispatch)

Generics use monomorphization (static dispatch, zero cost). Trait objects use vtables (dynamic dispatch, slight overhead).

```rust
// Static dispatch -- monomorphized at compile time, one copy per concrete type
fn notify<T: Summary>(item: &T) {
    println!("Breaking: {}", item.summarize());
}

// Dynamic dispatch -- single function, dispatches via vtable at runtime
fn notify_dynamic(item: &dyn Summary) {
    println!("Breaking: {}", item.summarize());
}

// Use trait objects when you need heterogeneous collections
fn all_summaries(items: &[Box<dyn Summary>]) {
    for item in items {
        println!("{}", item.summarize());
    }
}

// Use generics when you need maximum performance and types are known at compile time
fn longest_summary<T: Summary>(a: &T, b: &T) -> String {
    let sa = a.summarize();
    let sb = b.summarize();
    if sa.len() > sb.len() { sa } else { sb }
}
```

### impl Trait in Argument and Return Position

`impl Trait` is syntactic sugar for generics in arguments and enables returning opaque types.

```rust
// Argument position: equivalent to generic bound
fn print_summary(item: &impl Summary) {
    println!("{}", item.summarize());
}

// Return position: caller does not know the concrete type
fn make_summarizable() -> impl Summary {
    Article {
        title: String::from("Rust 2024"),
        content: String::from("The Rust programming language continues to evolve..."),
    }
}
```

### Associated Types

Use associated types when a trait has a type that is determined by the implementation, not the caller.

```rust
trait Iterator {
    type Item;
    fn next(&mut self) -> Option<Self::Item>;
}

struct Counter {
    count: u32,
    max: u32,
}

impl Iterator for Counter {
    type Item = u32;

    fn next(&mut self) -> Option<Self::Item> {
        if self.count < self.max {
            self.count += 1;
            Some(self.count)
        } else {
            None
        }
    }
}
```

### Supertraits and Trait Composition

Require implementors to also implement other traits.

```rust
use std::fmt;

// Display is a supertrait -- anything implementing PrettyPrint must also implement Display
trait PrettyPrint: fmt::Display {
    fn pretty_print(&self) {
        println!("=== {} ===", self);
    }
}

// Trait composition with + syntax
fn debug_and_display(item: &(impl fmt::Debug + fmt::Display)) {
    println!("Display: {item}");
    println!("Debug: {item:?}");
}
```

### Blanket Implementations

Implement a trait for all types that satisfy a bound. The standard library uses this extensively.

```rust
// Implement ToString for anything that implements Display
// (this is already in std, shown for illustration)
impl<T: fmt::Display> ToString for T {
    fn to_string(&self) -> String {
        format!("{self}")
    }
}

// Custom blanket implementation
trait Loggable {
    fn log(&self);
}

impl<T: fmt::Debug> Loggable for T {
    fn log(&self) {
        println!("[LOG] {:?}", self);
    }
}
```

**Anti-patterns to avoid:**
- Using trait objects (`dyn Trait`) in hot loops where static dispatch would eliminate vtable overhead.
- Creating "god traits" with many methods -- split into focused traits and compose them.
- Implementing traits for types you do not own without a newtype wrapper (orphan rule violation).
- Using associated types when the trait should support multiple implementations for the same type (use generics instead).

---

## Error Handling

### Result and Option Basics

Rust uses `Result<T, E>` for recoverable errors and `Option<T>` for nullable values. The `?` operator propagates errors concisely.

```rust
use std::fs;
use std::io;
use std::num::ParseIntError;

fn read_age_from_file(path: &str) -> Result<u32, Box<dyn std::error::Error>> {
    let contents = fs::read_to_string(path)?; // propagates io::Error
    let age: u32 = contents.trim().parse()?;  // propagates ParseIntError
    Ok(age)
}

// Option for nullable values
fn find_user(users: &[User], id: u64) -> Option<&User> {
    users.iter().find(|u| u.id == id)
}

// Combine Option and Result
fn get_user_age(users: &[User], id: u64) -> Result<u32, String> {
    let user = find_user(users, id).ok_or_else(|| format!("user {id} not found"))?;
    Ok(user.age)
}
```

### thiserror for Library Errors

Use `thiserror` to define structured error types for libraries and public APIs. Each variant maps to a specific failure mode.

```rust
use thiserror::Error;

#[derive(Error, Debug)]
pub enum StorageError {
    #[error("record not found: {id}")]
    NotFound { id: String },

    #[error("duplicate key: {key}")]
    DuplicateKey { key: String },

    #[error("connection failed")]
    Connection(#[source] io::Error),

    #[error("deserialization failed")]
    Deserialization(#[from] serde_json::Error),

    #[error("query timeout after {duration:?}")]
    Timeout { duration: std::time::Duration },
}

// Callers can match on specific variants
fn handle_storage_error(err: &StorageError) {
    match err {
        StorageError::NotFound { id } => eprintln!("missing record: {id}"),
        StorageError::Connection(source) => eprintln!("connection issue: {source}"),
        StorageError::Timeout { duration } => eprintln!("timed out after {duration:?}"),
        _ => eprintln!("storage error: {err}"),
    }
}
```

### anyhow for Application Errors

Use `anyhow` in application code (binaries, CLI tools) where you do not need callers to match on specific variants.

```rust
use anyhow::{Context, Result, bail, ensure};

fn load_config(path: &str) -> Result<Config> {
    let contents = std::fs::read_to_string(path)
        .with_context(|| format!("failed to read config from {path}"))?;

    let config: Config = toml::from_str(&contents)
        .context("failed to parse config as TOML")?;

    ensure!(config.port > 0, "port must be positive, got {}", config.port);

    if config.database_url.is_empty() {
        bail!("database_url is required");
    }

    Ok(config)
}

fn main() -> Result<()> {
    let config = load_config("config.toml")?;
    println!("starting on port {}", config.port);
    Ok(())
}
```

### Custom Error Conversion with From

Implement `From` to enable the `?` operator across error types without `thiserror`.

```rust
#[derive(Debug)]
enum AppError {
    Io(io::Error),
    Parse(ParseIntError),
    Custom(String),
}

impl fmt::Display for AppError {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        match self {
            AppError::Io(e) => write!(f, "I/O error: {e}"),
            AppError::Parse(e) => write!(f, "parse error: {e}"),
            AppError::Custom(msg) => write!(f, "{msg}"),
        }
    }
}

impl std::error::Error for AppError {
    fn source(&self) -> Option<&(dyn std::error::Error + 'static)> {
        match self {
            AppError::Io(e) => Some(e),
            AppError::Parse(e) => Some(e),
            AppError::Custom(_) => None,
        }
    }
}

impl From<io::Error> for AppError {
    fn from(e: io::Error) -> Self { AppError::Io(e) }
}

impl From<ParseIntError> for AppError {
    fn from(e: ParseIntError) -> Self { AppError::Parse(e) }
}
```

**Anti-patterns to avoid:**
- Using `.unwrap()` or `.expect()` in library code -- return `Result` instead.
- Using `anyhow` in library code -- callers cannot match on specific error variants.
- Discarding error context by converting to `String` too early.
- Using `panic!` for expected error conditions like invalid input or missing files.
- Wrapping errors without adding context -- `?` alone is fine, but `.context("what failed")` is better.

---

## Async Rust

### async/await and the Tokio Runtime

Rust's async model is zero-cost: async functions compile to state machines. You need a runtime (usually tokio) to drive them.

```rust
use tokio::time::{sleep, Duration};

#[tokio::main]
async fn main() {
    let result = fetch_data("https://api.example.com/data").await;
    match result {
        Ok(data) => println!("received: {data}"),
        Err(e) => eprintln!("error: {e}"),
    }
}

async fn fetch_data(url: &str) -> Result<String, reqwest::Error> {
    let response = reqwest::get(url).await?;
    let body = response.text().await?;
    Ok(body)
}
```

### Spawning Tasks and JoinHandles

Use `tokio::spawn` for concurrent tasks. Each spawned task runs independently on the runtime's thread pool.

```rust
use tokio::task::JoinHandle;

async fn process_batch(urls: Vec<String>) -> Vec<Result<String, String>> {
    let mut handles: Vec<JoinHandle<Result<String, String>>> = Vec::new();

    for url in urls {
        handles.push(tokio::spawn(async move {
            reqwest::get(&url)
                .await
                .map_err(|e| format!("request failed: {e}"))?
                .text()
                .await
                .map_err(|e| format!("body read failed: {e}"))
        }));
    }

    let mut results = Vec::new();
    for handle in handles {
        match handle.await {
            Ok(result) => results.push(result),
            Err(e) => results.push(Err(format!("task panicked: {e}"))),
        }
    }
    results
}
```

### select! for Racing Futures

`tokio::select!` runs multiple futures concurrently and acts on the first one that completes.

```rust
use tokio::sync::mpsc;
use tokio::time::{sleep, Duration};

async fn run_with_timeout() -> Result<String, &'static str> {
    let (tx, mut rx) = mpsc::channel(1);

    tokio::spawn(async move {
        sleep(Duration::from_secs(2)).await;
        let _ = tx.send("done".to_string()).await;
    });

    tokio::select! {
        Some(msg) = rx.recv() => Ok(msg),
        _ = sleep(Duration::from_secs(5)) => Err("operation timed out"),
    }
}
```

### Streams and Async Iteration

Streams are the async equivalent of iterators. Use `tokio_stream` or `futures::stream`.

```rust
use tokio_stream::{self as stream, StreamExt};

async fn process_stream() {
    let mut stream = stream::iter(vec![1, 2, 3, 4, 5])
        .map(|x| x * 2)
        .filter(|x| *x > 4);

    while let Some(value) = stream.next().await {
        println!("value: {value}");
    }
}

// Buffered concurrency: process up to N items concurrently
use futures::stream::{self, StreamExt as _};

async fn fetch_all(urls: Vec<String>) -> Vec<String> {
    stream::iter(urls)
        .map(|url| async move {
            reqwest::get(&url).await.unwrap().text().await.unwrap_or_default()
        })
        .buffer_unordered(10) // up to 10 concurrent requests
        .collect()
        .await
}
```

### Pinning and Why It Matters

Futures in Rust may contain self-references. `Pin<Box<dyn Future>>` guarantees the future will not be moved in memory, which is required for safe self-referential structures.

```rust
use std::pin::Pin;
use std::future::Future;

// Returning a boxed, pinned future (common in trait objects)
fn make_future(val: i32) -> Pin<Box<dyn Future<Output = i32> + Send>> {
    Box::pin(async move {
        tokio::time::sleep(tokio::time::Duration::from_millis(100)).await;
        val * 2
    })
}

// Async trait methods (Rust 1.75+ supports async fn in traits natively)
trait AsyncProcessor {
    async fn process(&self, input: &str) -> String;
}

// For older Rust or when dyn dispatch is needed, use the async_trait crate
// #[async_trait::async_trait]
// trait AsyncProcessor {
//     async fn process(&self, input: &str) -> String;
// }
```

### Cancellation Safety

When a branch of `select!` is cancelled, the future is dropped. Ensure your futures handle this gracefully.

```rust
use tokio::sync::mpsc;

// UNSAFE: partial read may lose data if cancelled between recv and processing
async fn unsafe_relay(mut rx: mpsc::Receiver<u32>, tx: mpsc::Sender<u32>) {
    loop {
        tokio::select! {
            Some(val) = rx.recv() => {
                // If cancelled here, val is lost
                tx.send(val).await.ok();
            }
        }
    }
}

// SAFE: use tokio::sync::mpsc which is cancellation-safe for recv()
// For complex cases, restructure so no data is "in flight" when select cancels
async fn safe_relay(mut rx: mpsc::Receiver<u32>, tx: mpsc::Sender<u32>) {
    while let Some(val) = rx.recv().await {
        if tx.send(val).await.is_err() {
            break; // receiver dropped
        }
    }
}
```

**Anti-patterns to avoid:**
- Holding a `MutexGuard` (from `std::sync::Mutex`) across `.await` -- use `tokio::sync::Mutex` instead.
- Spawning unbounded tasks without backpressure -- use semaphores or `buffer_unordered`.
- Blocking the async runtime with CPU-heavy or synchronous I/O work -- use `tokio::task::spawn_blocking`.
- Ignoring `JoinHandle` results -- panics in spawned tasks are silently swallowed.
- Using `async` for code that does no I/O and no waiting -- it adds overhead for no benefit.

---

## Unsafe Rust

### When to Use Unsafe

Unsafe Rust is necessary for FFI, raw pointer manipulation, and implementing low-level abstractions. Minimize its surface area and wrap it in safe APIs.

```rust
// The five things unsafe allows:
// 1. Dereference raw pointers
// 2. Call unsafe functions
// 3. Access mutable statics
// 4. Implement unsafe traits
// 5. Access fields of unions

unsafe fn dangerous() {
    // This function's contract must be upheld by the caller
}

fn safe_wrapper(data: &[u8]) -> u32 {
    assert!(data.len() >= 4, "need at least 4 bytes");
    // SAFETY: we checked that data has at least 4 bytes
    unsafe {
        let ptr = data.as_ptr() as *const u32;
        ptr.read_unaligned()
    }
}
```

### Raw Pointers and Dereferencing

Raw pointers (`*const T`, `*mut T`) are like C pointers with no safety guarantees. Creating them is safe; dereferencing requires `unsafe`.

```rust
fn raw_pointer_example() {
    let mut value = 42;

    let r1 = &value as *const i32;         // immutable raw pointer
    let r2 = &mut value as *mut i32;       // mutable raw pointer

    // SAFETY: r1 and r2 both point to a valid, aligned i32 that is still alive.
    // We only read through r1 and write through r2, not simultaneously.
    unsafe {
        println!("r1 = {}", *r1);
        *r2 = 100;
        println!("r2 = {}", *r2);
    }
}

// Working with slices from raw parts
fn slice_from_raw(ptr: *const u8, len: usize) -> &'static [u8] {
    assert!(!ptr.is_null());
    // SAFETY: caller guarantees ptr is valid for `len` bytes and lives for 'static
    unsafe { std::slice::from_raw_parts(ptr, len) }
}
```

### FFI (Foreign Function Interface)

Call C libraries from Rust or expose Rust functions to C.

```rust
// Calling C from Rust
extern "C" {
    fn abs(input: i32) -> i32;
    fn strlen(s: *const std::ffi::c_char) -> usize;
}

fn call_c_abs(x: i32) -> i32 {
    // SAFETY: abs is a pure function with no preconditions
    unsafe { abs(x) }
}

// Exposing Rust to C
#[no_mangle]
pub extern "C" fn rust_add(a: i32, b: i32) -> i32 {
    a + b
}

// Safe wrapper around a C library
pub struct CLibHandle {
    ptr: *mut ffi::c_void,
}

impl CLibHandle {
    pub fn new() -> Result<Self, String> {
        // SAFETY: c_lib_init returns a valid handle or null on failure
        let ptr = unsafe { c_lib_init() };
        if ptr.is_null() {
            Err("initialization failed".into())
        } else {
            Ok(Self { ptr })
        }
    }
}

impl Drop for CLibHandle {
    fn drop(&mut self) {
        // SAFETY: self.ptr was returned by c_lib_init and has not been freed
        unsafe { c_lib_destroy(self.ptr) };
    }
}
```

### Unsafe Traits

An `unsafe trait` signals that the implementor must uphold certain invariants the compiler cannot check.

```rust
// unsafe trait: implementors must guarantee the type has a stable memory layout
unsafe trait StableLayout {
    fn as_bytes(&self) -> &[u8];
}

unsafe impl StableLayout for u32 {
    fn as_bytes(&self) -> &[u8] {
        // SAFETY: u32 is a POD type with a well-defined layout
        unsafe {
            std::slice::from_raw_parts(
                self as *const u32 as *const u8,
                std::mem::size_of::<u32>(),
            )
        }
    }
}
```

### Audit Patterns and Miri

Document every `unsafe` block with a `// SAFETY:` comment explaining why the invariants hold. Use Miri to detect undefined behavior in tests.

```bash
# Install and run Miri to check for undefined behavior
rustup component add miri
cargo miri test

# Miri catches:
# - Out-of-bounds memory access
# - Use-after-free
# - Invalid use of uninitialized data
# - Data races (with -Zmiri-check-data-races)
# - Violations of aliasing rules (Stacked Borrows / Tree Borrows)
```

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_safe_wrapper() {
        // This test can be run under Miri: cargo miri test
        let data = [0x01, 0x02, 0x03, 0x04];
        let result = safe_wrapper(&data);
        assert!(result > 0);
    }
}
```

**Anti-patterns to avoid:**
- Using `unsafe` to bypass the borrow checker instead of restructuring code.
- Missing `// SAFETY:` comments on unsafe blocks -- every block needs a justification.
- Assuming `unsafe` code is correct because "it works in testing" -- UB can be latent.
- Transmuting between types without verifying layout compatibility (`#[repr(C)]` or `#[repr(transparent)]`).
- Not running Miri in CI for crates that contain `unsafe` code.

---

## Performance Patterns

### Zero-Cost Abstractions

Rust's iterators, closures, and generics compile to the same machine code as hand-written loops.

```rust
// This iterator chain compiles to a single optimized loop -- no intermediate allocations
fn sum_of_squares(data: &[i32]) -> i64 {
    data.iter()
        .filter(|&&x| x > 0)
        .map(|&x| (x as i64) * (x as i64))
        .sum()
}

// Equivalent hand-written loop (same assembly output)
fn sum_of_squares_manual(data: &[i32]) -> i64 {
    let mut total: i64 = 0;
    for &x in data {
        if x > 0 {
            total += (x as i64) * (x as i64);
        }
    }
    total
}
```

### Stack vs Heap: Box, Vec, and Allocation Awareness

Know when data lives on the stack vs the heap to minimize allocations.

```rust
// Stack-allocated: fixed-size, fast, no allocator involved
let point = (1.0_f64, 2.0_f64, 3.0_f64); // 24 bytes on the stack

// Heap-allocated: dynamic size, indirection through a pointer
let data: Vec<f64> = vec![1.0, 2.0, 3.0]; // header on stack, buffer on heap

// Box: single heap allocation for a known-size value (useful for large structs or trait objects)
let big_struct: Box<[u8; 1_000_000]> = Box::new([0u8; 1_000_000]);

// Pre-allocate to avoid repeated reallocations
fn collect_results(count: usize) -> Vec<String> {
    let mut results = Vec::with_capacity(count);
    for i in 0..count {
        results.push(format!("item-{i}"));
    }
    results
}
```

### Cow (Clone on Write)

`Cow<'a, T>` holds either a borrowed reference or an owned value. It avoids cloning when mutation is not needed.

```rust
use std::borrow::Cow;

fn normalize_name(name: &str) -> Cow<'_, str> {
    if name.contains(' ') {
        // Only allocate when modification is needed
        Cow::Owned(name.replace(' ', "_"))
    } else {
        // Zero-cost: just returns the borrowed reference
        Cow::Borrowed(name)
    }
}

fn process_names(names: &[&str]) {
    for name in names {
        let normalized = normalize_name(name);
        println!("{normalized}"); // works the same whether borrowed or owned
    }
}
```

### SmallVec for Short Vectors

`SmallVec` stores a small number of elements inline (on the stack) and spills to the heap only when the capacity is exceeded.

```rust
use smallvec::{SmallVec, smallvec};

// Store up to 4 elements on the stack; heap-allocate beyond that
type ShortList = SmallVec<[u32; 4]>;

fn collect_neighbors(adjacency: &[Vec<u32>], node: usize) -> ShortList {
    let mut neighbors: ShortList = smallvec![];
    for &neighbor in &adjacency[node] {
        neighbors.push(neighbor);
    }
    neighbors
}

// Useful in parsers, graph algorithms, and hot paths where most collections are small
```

### Benchmarking with Criterion

Use `criterion` for statistically rigorous benchmarks instead of the unstable `#[bench]` attribute.

```rust
// Cargo.toml:
// [dev-dependencies]
// criterion = { version = "0.5", features = ["html_reports"] }
//
// [[bench]]
// name = "my_benchmark"
// harness = false

// benches/my_benchmark.rs
use criterion::{black_box, criterion_group, criterion_main, Criterion, BenchmarkId};

fn fibonacci(n: u64) -> u64 {
    match n {
        0 => 0,
        1 => 1,
        _ => fibonacci(n - 1) + fibonacci(n - 2),
    }
}

fn bench_fibonacci(c: &mut Criterion) {
    let mut group = c.benchmark_group("fibonacci");
    for size in [10, 15, 20] {
        group.bench_with_input(BenchmarkId::from_parameter(size), &size, |b, &n| {
            b.iter(|| fibonacci(black_box(n)));
        });
    }
    group.finish();
}

criterion_group!(benches, bench_fibonacci);
criterion_main!(benches);
```

```bash
# Run benchmarks
cargo bench

# Compare against a baseline
cargo bench -- --save-baseline main
# ... make changes ...
cargo bench -- --baseline main
```

### Profiling with perf and flamegraph

```bash
# Generate a flamegraph (install: cargo install flamegraph)
cargo flamegraph --bin my_app -- --some-arg

# Use perf on Linux
perf record --call-graph dwarf ./target/release/my_app
perf report

# Compile with debug info in release for useful profiles
# Cargo.toml:
# [profile.release]
# debug = true
```

### SIMD with std::simd (Nightly) and Portable Patterns

```rust
// Stable Rust: use iterator patterns that auto-vectorize
fn dot_product(a: &[f32], b: &[f32]) -> f32 {
    a.iter()
        .zip(b.iter())
        .map(|(x, y)| x * y)
        .sum()
}

// For explicit SIMD on stable Rust, use the `packed_simd2` or `wide` crate
// Or use target-specific intrinsics:
#[cfg(target_arch = "x86_64")]
use std::arch::x86_64::*;

#[cfg(target_arch = "x86_64")]
unsafe fn sum_avx(data: &[f32]) -> f32 {
    // SAFETY: caller must ensure AVX is available (checked via is_x86_feature_detected!)
    let mut acc = _mm256_setzero_ps();
    for chunk in data.chunks_exact(8) {
        let v = _mm256_loadu_ps(chunk.as_ptr());
        acc = _mm256_add_ps(acc, v);
    }
    // horizontal sum of the 8 floats in acc
    let mut result = [0.0f32; 8];
    _mm256_storeu_ps(result.as_mut_ptr(), acc);
    result.iter().sum()
}
```

**Anti-patterns to avoid:**
- Optimizing without profiling -- always measure before changing code.
- Using `Box<dyn Trait>` in hot loops where generics would eliminate vtable overhead.
- Allocating in tight loops -- pre-allocate or use `SmallVec` / stack buffers.
- Ignoring `cargo bench` regressions -- add benchmark comparisons to CI.
- Using `unsafe` SIMD without runtime feature detection (`is_x86_feature_detected!`).
- Returning `Vec<u8>` when `&[u8]` or `Cow<[u8]>` would avoid allocation.

---

## Common Patterns Quick Reference

| Pattern | When to Use | Example |
|---|---|---|
| `Result<T, E>` + `?` | Every fallible operation | `let data = fs::read(path)?;` |
| `thiserror` enum | Library crate with typed errors | `#[derive(Error)] enum MyError { ... }` |
| `anyhow::Result` | Application / binary crate | `fn main() -> anyhow::Result<()>` |
| `Cow<'a, str>` | Conditionally owned strings | `fn normalize(s: &str) -> Cow<str>` |
| `Box<dyn Trait>` | Heterogeneous collections | `Vec<Box<dyn Handler>>` |
| `impl Trait` return | Hide concrete return type | `fn make() -> impl Iterator<Item = u32>` |
| `Arc<T>` + `Clone` | Shared ownership across threads | `let shared = Arc::new(data);` |
| `Arc<Mutex<T>>` | Shared mutable state across threads | `Arc::new(Mutex::new(state))` |
| `tokio::spawn` | Fire-and-forget async tasks | `tokio::spawn(async move { ... });` |
| `tokio::select!` | Race multiple futures | `select! { v = rx.recv() => ..., _ = sleep(d) => ... }` |
| `buffer_unordered(N)` | Bounded concurrent stream processing | `.buffer_unordered(10).collect().await` |
| `SmallVec<[T; N]>` | Small, usually-stack-allocated lists | `SmallVec::<[u32; 4]>::new()` |
| `Vec::with_capacity` | Known-size collection | `Vec::with_capacity(1000)` |
| `spawn_blocking` | CPU-heavy work in async context | `tokio::task::spawn_blocking(move \|\| ...)` |
| `#[repr(C)]` | FFI-safe struct layout | `#[repr(C)] struct Packet { ... }` |
| `Pin<Box<dyn Future>>` | Async trait objects | `fn op() -> Pin<Box<dyn Future<Output=T>>>` |
| `newtype` wrapper | Type safety, orphan rule workaround | `struct Meters(f64);` |
| `derive` macros | Reduce boilerplate | `#[derive(Debug, Clone, Serialize)]` |
| Builder pattern | Complex struct construction | `Config::builder().port(8080).build()` |
| Typestate pattern | Compile-time state machine enforcement | `struct Request<S: State> { state: S }` |

### Choosing Between Smart Pointers

| Type | Ownership | Thread-Safe | When to Use |
|---|---|---|---|
| `Box<T>` | Single owner | No (unless `T: Send`) | Heap allocation, recursive types, trait objects |
| `Rc<T>` | Shared (ref-counted) | No | Single-threaded shared ownership |
| `Arc<T>` | Shared (atomic ref-counted) | Yes | Multi-threaded shared ownership |
| `Cell<T>` | Interior mutability | No | Copy types that need mutation through `&self` |
| `RefCell<T>` | Interior mutability (runtime checks) | No | Non-Copy types, single-threaded interior mutability |
| `Mutex<T>` | Interior mutability (blocking) | Yes | Multi-threaded mutable access |
| `RwLock<T>` | Interior mutability (readers/writer) | Yes | Many readers, occasional writers |

### Cargo Essentials

```bash
# Build and run
cargo build --release
cargo run --release -- --args

# Testing
cargo test                       # all tests
cargo test -- --nocapture        # show stdout
cargo test test_name             # run specific test

# Linting and formatting
cargo fmt                        # format code
cargo clippy -- -D warnings      # lint with warnings as errors

# Dependency management
cargo add serde --features derive # add dependency
cargo update                      # update within semver
cargo audit                       # check for known vulnerabilities
cargo tree                        # dependency graph
```
