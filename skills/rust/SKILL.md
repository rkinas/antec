# Antec Rust Coding Standard

> **Version**: 1.0.0
> **Rust Edition**: 2021
> **Runtime**: Tokio 1.x
> **Applies to**: All 15 crates in the Antec workspace
> **Sources**: Rust API Guidelines, Rust Reference, Rust Performance Book, Tokio best practices, production codebases (tokio, serde, axum, polars)

---

## How to Use This Standard

This document defines 179 rules across 14 categories, ordered by priority. Rules use prefixes (`own-`, `err-`, `mem-`, etc.) for easy reference in code reviews and CI.

**Priority levels:**
- **CRITICAL** — violations are build-blockers
- **HIGH** — violations must be fixed before merge
- **MEDIUM** — should be followed, exceptions documented
- **LOW** — best practice, not enforced
- **REFERENCE** — patterns to avoid (anti-pattern catalog)

**When unsure about a Rust API, use Context7 MCP to verify:**
- `rust-lang/reference` — language semantics, lifetime rules, trait system
- `tokio-rs/tokio` — async runtime, channels, I/O
- `tokio-rs/axum` — HTTP framework, extractors, middleware
- `rusqlite/rusqlite` — SQLite bindings
- `bytecodealliance/wasmtime` — WASM runtime

---

## Category 1: Ownership & Borrowing [CRITICAL]

Prefix: `own-`

### own-borrow-over-clone
Prefer borrowing over cloning. Clone only when ownership transfer is genuinely required.

```rust
// GOOD: borrow
fn process_message(content: &str) -> Result<Response> { ... }

// BAD: unnecessary clone
fn process_message(content: String) -> Result<Response> { ... } // caller forced to clone
```

### own-str-param
Accept `&str` in function parameters, not `String`. Return `String` when the function produces a new string.

```rust
// GOOD
fn normalize_channel(name: &str) -> String { ... }

// BAD
fn normalize_channel(name: String) -> String { ... }
```

### own-arc-async
Use `Arc<T>` for shared ownership across async tasks. Never use `Rc<T>` in async code — it is not `Send`.

```rust
// GOOD: shared state across spawned tasks
let state = Arc::new(AppState::new());
let state_clone = Arc::clone(&state);
tokio::spawn(async move { state_clone.handle().await });

// BAD: Rc in async context
let state = Rc::new(AppState::new()); // compile error: Rc is not Send
```

### own-cow-parsing
Use `Cow<'_, str>` in parsing hot paths to avoid allocation when input can be returned as-is.

```rust
use std::borrow::Cow;

fn sanitize(input: &str) -> Cow<'_, str> {
    if input.contains('<') {
        Cow::Owned(input.replace('<', "&lt;"))
    } else {
        Cow::Borrowed(input)
    }
}
```

### own-copy-small
Derive `Copy` for small types (≤ 16 bytes) that have no heap allocation. This includes IDs, flags, enum variants without data.

```rust
#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash)]
pub enum RiskLevel {
    Safe,
    Moderate,
    Dangerous,
}
```

### own-interior-mut-async
Use `tokio::sync::Mutex` (not `std::sync::Mutex`) when the lock must be held across `.await` points.

```rust
// GOOD: tokio mutex for async
let sessions: Arc<tokio::sync::Mutex<HashMap<SessionId, Session>>> = ...;
let mut guard = sessions.lock().await;
guard.insert(id, session);
// guard can be held across .await

// BAD: std mutex held across await
let guard = std_mutex.lock().unwrap(); // DEADLOCK RISK
some_async_fn().await; // other tasks can't acquire lock
```

### own-rwlock-read-heavy
Use `tokio::sync::RwLock` when reads vastly outnumber writes (e.g., tool registry, config, locale data).

```rust
let registry: Arc<RwLock<ToolRegistry>> = ...;
// Many concurrent readers
let tools = registry.read().await;
// Exclusive writer
let mut tools = registry.write().await;
```

### own-drop-explicit
Implement `Drop` only for types managing external resources (file handles, DB connections, temp dirs). Never for logic.

### own-lifetime-elision
Rely on lifetime elision where the compiler can infer. Only write explicit lifetimes when required by the borrow checker.

```rust
// GOOD: elided (compiler infers)
fn first_word(s: &str) -> &str { ... }

// Only explicit when needed
fn longest<'a>(a: &'a str, b: &'a str) -> &'a str { ... }
```

### own-move-closures
Use `move` closures for spawned tasks. Be explicit about what is captured.

```rust
let config = Arc::clone(&config);
tokio::spawn(async move {
    // config is moved into the task
    config.reload().await;
});
```

### own-pin-futures
Use `Pin<Box<dyn Future>>` only when dynamic dispatch on futures is required (trait objects returning futures). Prefer static dispatch otherwise.

### own-phantom-data
Use `PhantomData<T>` when a type parameter is needed for type safety but isn't stored. Common in typestate patterns.

---

## Category 2: Error Handling [CRITICAL]

Prefix: `err-`

### err-thiserror-lib
Library crates use `thiserror` to define typed error enums. Each `antec-*` crate has its own `Error` type.

```rust
// crates/antec-storage/src/error.rs
use thiserror::Error;

#[derive(Debug, Error)]
pub enum StorageError {
    #[error("migration failed at version {version}: {source}")]
    Migration {
        version: u32,
        #[source]
        source: rusqlite::Error,
    },
    #[error("connection pool exhausted")]
    PoolExhausted,
    #[error("query failed: {0}")]
    Query(#[from] rusqlite::Error),
}
```

### err-anyhow-app
Application entry point (`src/main.rs`) uses `anyhow` for ergonomic error propagation with context.

```rust
use anyhow::{Context, Result};

#[tokio::main]
async fn main() -> Result<()> {
    let config = load_config()
        .context("failed to load configuration")?;
    start_server(config)
        .await
        .context("server failed")?;
    Ok(())
}
```

### err-no-unwrap
**Never** use `.unwrap()` or `.expect()` in production code. Tests may use `.unwrap()`.

```rust
// GOOD
let value = map.get(&key).ok_or(Error::NotFound(key))?;

// BAD (production)
let value = map.get(&key).unwrap(); // panics on missing key

// OK (test only)
#[cfg(test)]
fn test_something() {
    let result = do_thing().unwrap();
}
```

### err-from-impl
Implement `From<SourceError>` for crate errors via `#[from]` in thiserror. This enables `?` to auto-convert.

### err-result-alias
Define a crate-level `Result<T>` type alias to reduce boilerplate.

```rust
// crates/antec-core/src/lib.rs
pub type Result<T> = std::result::Result<T, crate::Error>;
```

### err-display-user
Error `Display` messages should be human-readable. Debug representation is for logs.

### err-context-chain
Add context at each abstraction boundary to build a meaningful error chain.

```rust
let db = open_database(&path)
    .map_err(|e| StorageError::Init { path: path.clone(), source: e })?;
```

### err-error-enum-flat
Keep error enums flat (one level). Don't nest error enums inside error enums.

### err-infallible-new
Constructors (`new()`) should return the type directly if they can't fail. Use `try_new()` or `build()` if they can.

### err-panic-never
`panic!` in production code is a bug. Use it only in `unreachable!()` for provably impossible states.

### err-io-context
Always wrap `std::io::Error` with context (file path, operation name).

### err-custom-status
Map domain errors to HTTP status codes in the gateway crate. Don't leak internal error types to API responses.

---

## Category 3: Memory Optimization [CRITICAL]

Prefix: `mem-`

### mem-vec-capacity
Preallocate `Vec::with_capacity(n)` when the size is known or estimable.

```rust
// GOOD
let mut results = Vec::with_capacity(messages.len());
for msg in messages {
    results.push(process(msg)?);
}

// BAD: repeated reallocation
let mut results = Vec::new();
```

### mem-string-capacity
Use `String::with_capacity()` + `push_str()` for string building in loops.

```rust
// GOOD
let mut sql = String::with_capacity(256);
sql.push_str("SELECT * FROM memories WHERE ");
sql.push_str(&condition);

// BAD: repeated allocation
let sql = format!("SELECT * FROM memories WHERE {}", condition);
// (format! is fine for one-shot, bad in hot loops)
```

### mem-smallvec
Use `SmallVec<[T; N]>` for collections that are usually small (< 8 elements) but occasionally larger.

```rust
use smallvec::SmallVec;

// Tool call results — usually 1-3, occasionally more
let mut results: SmallVec<[ToolResult; 4]> = SmallVec::new();
```

### mem-box-large
Box large enum variants to keep the enum's stack size small.

```rust
// GOOD: large variant boxed
enum Message {
    Text(String),
    Image(Box<ImageData>), // ImageData is large
    Ping,
}

// BAD: all variants as large as the biggest
enum Message {
    Text(String),
    Image(ImageData), // bloats all variants
    Ping,
}
```

### mem-iter-chain
Prefer iterator chains over collecting intermediate `Vec`s.

```rust
// GOOD: lazy, zero intermediate allocation
let active_tools: Vec<_> = registry
    .tools()
    .filter(|t| t.is_enabled())
    .map(|t| t.name())
    .collect();

// BAD: allocates intermediate vec
let all: Vec<_> = registry.tools().collect();
let active: Vec<_> = all.iter().filter(|t| t.is_enabled()).collect();
```

### mem-bytes-slice
Use `&[u8]` over `Vec<u8>` in function parameters for read-only byte data.

### mem-arena-batch
Consider arena allocation (`bumpalo`) for batch processing where many small objects have the same lifetime.

### mem-zero-copy
Use `bytes::Bytes` for network buffers to enable zero-copy slicing.

### mem-drop-early
Drop large allocations explicitly with `drop(value)` when they're no longer needed in a long-running scope.

### mem-no-format-hot
Avoid `format!()` in hot paths. Pre-compute strings or use `write!()` to a buffer.

### mem-stack-arrays
Use arrays `[T; N]` instead of `Vec<T>` when size is compile-time constant and small.

### mem-profile
Profile memory with `dhat` or `heaptrack` before optimizing. Don't guess.

---

## Category 4: API Design [HIGH]

Prefix: `api-`

### api-newtype-ids
Wrap primitive IDs in newtypes for type safety.

```rust
#[derive(Debug, Clone, PartialEq, Eq, Hash, Serialize, Deserialize)]
pub struct SessionId(pub Uuid);

#[derive(Debug, Clone, PartialEq, Eq, Hash, Serialize, Deserialize)]
pub struct ToolName(pub String);

// Prevents: fn get_session(id: Uuid) — which Uuid? Session? User? Message?
```

### api-builder
Use builder pattern for structs with 3+ optional fields.

```rust
let config = ServerConfig::builder()
    .bind("127.0.0.1")
    .port(8088)
    .cors_origins(vec!["http://localhost:3000"])
    .build()?;
```

### api-into-params
Accept `impl Into<String>` for string parameters. Return concrete types.

```rust
// GOOD: flexible input
pub fn new(name: impl Into<String>) -> Self {
    Self { name: name.into() }
}

// Caller can pass &str or String without explicit conversion
let tool = Tool::new("calculator");
let tool = Tool::new(dynamic_string);
```

### api-trait-plugins
Define traits for all plugin boundaries. This is the core extensibility pattern of Antec.

```rust
#[async_trait]
pub trait LlmProvider: Send + Sync {
    fn name(&self) -> &str;
    async fn complete(&self, request: CompletionRequest) -> Result<CompletionResponse>;
    async fn stream(&self, request: CompletionRequest) -> Result<CompletionStream>;
    fn supports_tools(&self) -> bool;
}
```

### api-seal-internal
Seal traits that are not meant to be implemented outside the crate.

```rust
mod sealed {
    pub trait Sealed {}
}

pub trait InternalTrait: sealed::Sealed {
    fn internal_method(&self);
}

// Only types in this crate can implement InternalTrait
```

### api-typestate
Use typestate pattern for complex initialization sequences.

```rust
pub struct Connection<S> { inner: RawConn, _state: PhantomData<S> }
pub struct Disconnected;
pub struct Connected;
pub struct Authenticated;

impl Connection<Disconnected> {
    pub fn connect(self) -> Result<Connection<Connected>> { ... }
}
impl Connection<Connected> {
    pub fn authenticate(self, token: &str) -> Result<Connection<Authenticated>> { ... }
}
impl Connection<Authenticated> {
    pub fn query(&self, sql: &str) -> Result<Rows> { ... }
}
```

### api-non-exhaustive
Use `#[non_exhaustive]` on public enums and struct constructors to allow future additions.

```rust
#[derive(Debug)]
#[non_exhaustive]
pub enum ChannelEvent {
    MessageReceived(NormalizedMessage),
    ConnectionLost,
    RateLimited,
}
```

### api-extension-trait
Use extension traits to add methods to foreign types.

```rust
pub trait ResultExt<T> {
    fn log_err(self) -> Option<T>;
}

impl<T, E: std::fmt::Display> ResultExt<T> for Result<T, E> {
    fn log_err(self) -> Option<T> {
        match self {
            Ok(v) => Some(v),
            Err(e) => { tracing::warn!("{e}"); None }
        }
    }
}
```

### api-validate-boundary
Validate all data at system boundaries (API input, config parsing, external responses). Trust internal data.

### api-send-sync
Ensure all public types used in async contexts are `Send + Sync`. Use `#[async_trait]` for async trait methods.

### api-deref-newtype
Implement `Deref` for simple newtypes that should transparently expose the inner type.

### api-default-derive
Derive `Default` for types with sensible defaults. Implement `Default` explicitly when logic is needed.

### api-display-debug
Implement `Display` for user-facing output. Derive `Debug` for all public types.

### api-impl-from
Implement `From` conversions between related types. Use `TryFrom` when conversion can fail.

### api-const-fn
Use `const fn` where possible for compile-time evaluation (constructors, small computations).

---

## Category 5: Async / Await [HIGH]

Prefix: `async-`

### async-tokio-only
Tokio is the only async runtime. Never mix runtimes. Never call `block_on` inside async context.

### async-no-std-mutex-await
Never hold `std::sync::Mutex` across `.await` points. Use `tokio::sync::Mutex`.

```rust
// GOOD
let guard = tokio_mutex.lock().await;
do_async_work().await;
drop(guard);

// BAD: deadlock risk
let guard = std_mutex.lock().unwrap();
do_async_work().await; // other tasks can't acquire
```

### async-spawn-blocking
Offload CPU-heavy or blocking I/O to `spawn_blocking`.

```rust
let hash = tokio::task::spawn_blocking(move || {
    argon2::hash_encoded(password.as_bytes(), &salt, &config)
}).await??;
```

### async-channels
Choose the right channel for the pattern:

| Pattern | Channel | Example |
|---------|---------|---------|
| Fan-in (many producers, one consumer) | `mpsc` | Tool results → agent loop |
| Fan-out (one producer, many consumers) | `broadcast` | Config changes → all components |
| Latest value (single writer, many readers) | `watch` | Server state, health status |
| One-shot (single response) | `oneshot` | Tool approval request → response |

### async-joinset
Use `JoinSet` for parallel task execution with bounded concurrency.

```rust
let mut set = JoinSet::new();
let semaphore = Arc::new(Semaphore::new(4)); // max 4 concurrent

for tool_call in tool_calls {
    let permit = semaphore.clone().acquire_owned().await?;
    set.spawn(async move {
        let result = execute_tool(tool_call).await;
        drop(permit);
        result
    });
}

while let Some(result) = set.join_next().await {
    results.push(result??);
}
```

### async-select-biased
Use `tokio::select!` with `biased;` when priority ordering matters (e.g., shutdown signal takes priority).

```rust
tokio::select! {
    biased;
    _ = shutdown.recv() => break,
    msg = rx.recv() => handle(msg).await,
}
```

### async-cancel-safe
Document cancellation safety of async functions. Use `tokio::pin!` when needed.

### async-timeout
Wrap external calls with `tokio::time::timeout`. Never wait forever on LLM providers, MCP servers, or network requests.

```rust
let response = tokio::time::timeout(
    Duration::from_secs(30),
    provider.complete(request),
).await.map_err(|_| Error::Timeout)??;
```

### async-graceful-shutdown
Use `CancellationToken` or `broadcast` channel for coordinated shutdown across all background tasks.

### async-stream-backpressure
Apply backpressure to streams. Use bounded channels. Never buffer unbounded data from external sources.

### async-no-sleep-loop
Don't poll with `sleep` in a loop. Use `tokio::sync::Notify`, channels, or `tokio::time::interval`.

### async-task-local
Use `tokio::task_local!` for request-scoped context (session ID, trace ID) instead of passing through every function.

### async-instrument
Use `#[tracing::instrument]` on async functions for structured logging with span context.

### async-pin-box-trait
Async trait methods return `Pin<Box<dyn Future>>` via `#[async_trait]`. When performance is critical, use manual impl.

### async-no-recursive
Avoid recursive async functions — they require boxing. Restructure as loops or use explicit stack.

---

## Category 6: Compiler Optimization [HIGH]

Prefix: `opt-`

### opt-release-profile
Use optimized release profile in `Cargo.toml`:

```toml
[profile.release]
opt-level = 3
lto = "fat"
codegen-units = 1
panic = "abort"
strip = "symbols"

[profile.dev]
opt-level = 0
debug = true

[profile.bench]
inherits = "release"
debug = true
strip = "none"
```

### opt-inline-small
Use `#[inline]` on small, frequently-called functions (< 5 lines). Use `#[inline(always)]` sparingly — only for proven hot paths.

### opt-link-time-opt
Enable LTO (`lto = "fat"`) in release builds for cross-crate inlining. Use `lto = "thin"` for faster compile times during CI.

### opt-single-codegen
Use `codegen-units = 1` in release for better optimization. Use default (16) in dev for faster compilation.

### opt-target-cpu
Build with `RUSTFLAGS="-C target-cpu=native"` for deployment on known hardware.

### opt-pgo
Consider Profile-Guided Optimization for the final release binary after benchmarking.

### opt-branch-likely
Use `std::intrinsics::likely`/`unlikely` (nightly) or restructure code to put hot paths first in `match` arms.

### opt-no-panic-release
Use `panic = "abort"` in release to eliminate unwinding code. This reduces binary size.

### opt-strip-binary
Strip symbols in release: `strip = "symbols"`. Debug info is in separate files if needed.

### opt-compile-time
Use `phf` for compile-time hash maps (injection patterns, locale data, MIME types).

### opt-simd-consider
Consider SIMD for bulk text processing (injection detection, memory search). Use `packed_simd2` or auto-vectorization hints.

### opt-bench-before
Always benchmark before optimizing. Use `criterion` for micro-benchmarks. Profile with `perf` / `instruments`.

---

## Category 7: Naming Conventions [MEDIUM]

Prefix: `name-`

### name-types
Types and traits: `PascalCase`. `AgentLoop`, `ToolRegistry`, `LlmProvider`.

### name-functions
Functions and methods: `snake_case`. `process_message`, `find_references`, `build_prompt`.

### name-constants
Constants and statics: `SCREAMING_SNAKE_CASE`. `MAX_CONTEXT_TOKENS`, `DEFAULT_PORT`.

### name-constructors
- `new()` — primary constructor
- `with_config()`, `with_capacity()` — constructor with specific parameter
- `from_*()` — conversion constructor (may allocate)

### name-conversions
- `as_*()` — cheap reference conversion, no allocation (`as_str()`, `as_bytes()`)
- `to_*()` — expensive conversion, may allocate (`to_string()`, `to_vec()`)
- `into_*()` — ownership-consuming conversion (`into_inner()`, `into_bytes()`)

### name-booleans
Boolean methods: `is_*()`, `has_*()`, `can_*()`. Boolean fields: same prefixes.

```rust
fn is_authenticated(&self) -> bool { ... }
fn has_permission(&self, perm: &str) -> bool { ... }
fn can_execute(&self) -> bool { ... }
```

### name-modules
Module names: `snake_case`, singular. `mod agent;` not `mod agents;`. Exception: `mod tests`.

### name-crate-prefix
Don't repeat the crate name in type names. The crate is already a namespace.

```rust
// GOOD: in antec-memory crate
pub struct Manager { ... }      // used as memory::Manager

// BAD:
pub struct MemoryManager { ... } // redundant: memory::MemoryManager
```

### name-acronyms
Treat acronyms as words in PascalCase: `HttpServer`, `SqlitePool`, `WasmSandbox` (not `HTTPServer`).

### name-iterators
Iterator methods: `iter()` for `&T`, `iter_mut()` for `&mut T`, `into_iter()` for `T`.

### name-error-suffix
Error types end with `Error`: `StorageError`, `GatewayError`, `SecurityError`.

### name-feature-flags
Feature flags: `kebab-case`. `discord`, `whatsapp`, `imessage`, `semantic-search`.

---

## Category 8: Type Safety [MEDIUM]

Prefix: `type-`

### type-newtype-wrap
Wrap primitives in newtypes at API boundaries. See `api-newtype-ids`.

### type-enum-state
Use enums for state machines instead of boolean flags or string states.

```rust
// GOOD
enum SessionState {
    Created,
    Active { started_at: Instant },
    Compacting,
    Archived { archived_at: DateTime<Utc> },
}

// BAD
struct Session {
    is_active: bool,
    is_archived: bool,    // what if both true?
    state: String,        // typos, no exhaustiveness check
}
```

### type-phantom
Use `PhantomData` for zero-cost type-level markers. See `api-typestate`.

### type-never
Use `!` (never type) or `std::convert::Infallible` for operations that can't fail.

### type-option-default
Use `Option<T>` with `#[serde(default)]` for optional config fields. Never use sentinel values.

### type-strong-index
Use typed indices (`SessionIdx(usize)`) instead of raw `usize` to prevent index mixups.

### type-exhaustive-match
Match on enums exhaustively. Don't use `_ =>` catch-all on enums you control — compiler should enforce new variant handling.

### type-validate-parse
Parse, don't validate. Convert raw data into typed representations at the boundary.

```rust
// GOOD: parse into validated type
let port = Port::try_from(raw_port)?; // validates 1-65535

// BAD: validate then pass raw
assert!(raw_port > 0 && raw_port < 65536);
let port = raw_port; // still just a u16, can be misused
```

---

## Category 9: Testing [MEDIUM]

Prefix: `test-`

### test-per-function
Every public function gets at least one happy-path and one error-path test.

### test-descriptive-name
Name tests descriptively: `test_memory_search_returns_results_ranked_by_bm25`, not `test_search`.

### test-assert-message
Use descriptive assertion messages.

```rust
assert_eq!(
    result.len(), 3,
    "expected 3 memories for query '{}', got {}",
    query, result.len()
);
```

### test-mock-traits
Mock external dependencies via trait objects and test doubles. Never call real APIs.

```rust
struct MockLlmProvider {
    response: String,
}

#[async_trait]
impl LlmProvider for MockLlmProvider {
    async fn complete(&self, _req: CompletionRequest) -> Result<CompletionResponse> {
        Ok(CompletionResponse { content: self.response.clone(), ..Default::default() })
    }
}
```

### test-tokio-test
Async tests use `#[tokio::test]`. Use `#[tokio::test(start_paused = true)]` for time-dependent tests.

### test-tempfile
Use `tempfile` crate for filesystem tests. Never write to real paths.

### test-no-flaky
No flaky tests. If a test fails intermittently, fix the root cause. Never `#[ignore]` and forget.

### test-offline
All tests pass offline. No real network calls. Mock all external services.

### test-isolated
Each test is self-contained. No shared mutable state. Tests can run in any order.

### test-should-panic
Use `#[should_panic(expected = "...")]` for tests that verify panic behavior. Include the expected message.

### test-proptest
Use `proptest` for property-based testing of parsers, serializers, and algorithms.

### test-snapshot
Use `insta` for snapshot testing of complex output (SQL queries, prompt assembly, API responses).

---

## Category 10: Documentation [MEDIUM]

Prefix: `doc-`

### doc-public-api
All public items have doc comments (`///`). Include: what it does, parameters, return value, errors, examples.

### doc-module-level
Each module file starts with `//!` module-level documentation explaining the module's purpose.

### doc-examples
Include runnable examples in doc comments for complex APIs.

```rust
/// Searches long-term memory using BM25 + TF-IDF ranking.
///
/// # Arguments
/// * `query` - The search query (natural language or keywords)
/// * `limit` - Maximum number of results to return
///
/// # Errors
/// Returns `MemoryError::SearchFailed` if FTS5 query is malformed.
///
/// # Examples
/// ```
/// let results = memory.search("user prefers dark mode", 5).await?;
/// assert!(results.len() <= 5);
/// ```
pub async fn search(&self, query: &str, limit: usize) -> Result<Vec<Memory>> { ... }
```

### doc-panics
Document `# Panics` section if a function can panic (it shouldn't in production, but document if it does).

### doc-safety
Document `# Safety` section for all `unsafe` blocks explaining the invariants maintained.

### doc-internal
Use `//` comments for internal implementation notes. Keep them minimal — prefer self-documenting code.

### doc-todo
`TODO` comments must include a tracking reference: `// TODO(#123): implement rate limit bypass for internal calls`

### doc-changelog
Keep a CHANGELOG.md following Keep a Changelog format.

---

## Category 11: Performance Patterns [MEDIUM]

Prefix: `perf-`

### perf-lazy-static
Use `std::sync::LazyLock` (or `once_cell::sync::Lazy` pre-1.80) for expensive one-time initialization.

```rust
static INJECTION_PATTERNS: LazyLock<Vec<Regex>> = LazyLock::new(|| {
    compile_injection_patterns()
});
```

### perf-hash-map-capacity
Preallocate `HashMap::with_capacity_and_hasher()` when size is known. Use `ahash` or `fxhash` for non-cryptographic needs.

### perf-string-interning
Consider string interning for frequently compared strings (tool names, channel names, locale keys).

### perf-batch-sql
Batch SQLite operations in transactions. Single insert in a loop = O(n) disk syncs.

```rust
// GOOD: batch in transaction
let tx = conn.transaction()?;
for memory in memories {
    tx.execute("INSERT INTO memories ...", params![...])?;
}
tx.commit()?;

// BAD: one transaction per insert
for memory in memories {
    conn.execute("INSERT INTO memories ...", params![...])?; // auto-commit each
}
```

### perf-pool-connections
Use the r2d2 connection pool. Never open ad-hoc SQLite connections. Pool size = 8 (configured).

### perf-streaming
Stream LLM responses to clients. Never buffer the entire response before sending.

### perf-avoid-collect
Avoid `.collect()` when you can process items lazily via iterators.

### perf-cache-computed
Cache frequently computed values (compiled regexes, parsed config, FTS5 queries). Invalidate on change.

---

## Category 12: Project Structure [LOW]

Prefix: `proj-`

### proj-one-type-file
Major types get their own file. `agent_loop.rs`, `tool_registry.rs`, `session.rs`.

### proj-mod-reexport
`mod.rs` (or `lib.rs`) is for re-exports, not logic. Logic lives in dedicated files.

```rust
// crates/antec-core/src/lib.rs
mod agent_loop;
mod context;
mod error;
mod provider;

pub use agent_loop::AgentLoop;
pub use context::ContextWindow;
pub use error::{Error, Result};
pub use provider::LlmProvider;
```

### proj-pub-crate
Internal modules use `pub(crate)`. Only the public API surface is `pub`.

### proj-feature-gates
Optional features behind feature flags. Don't compile unused channel adapters.

```toml
[features]
default = []
discord = ["serenity"]
whatsapp = ["reqwest"]
imessage = ["osascript"]
semantic-search = ["candle-core"]
```

### proj-workspace-deps
Share dependency versions via workspace `[workspace.dependencies]` in root `Cargo.toml`.

### proj-deny-warnings
`#![deny(warnings)]` in CI only. Use `#![warn(warnings)]` in dev to avoid blocking iteration.

---

## Category 13: Clippy & Linting [LOW]

Prefix: `lint-`

### lint-clippy-strict
Run clippy with `-D warnings`. All clippy warnings are errors in CI.

```bash
cargo clippy --workspace --all-targets -- -D warnings
```

### lint-clippy-config
Configure clippy in `clippy.toml` or `Cargo.toml`:

```toml
# Cargo.toml
[lints.clippy]
pedantic = { level = "warn", priority = -1 }
# Allow specific pedantic lints that are too noisy
module_name_repetitions = "allow"
must_use_candidate = "allow"
missing_errors_doc = "allow"
```

### lint-rustfmt
Enforce consistent formatting with `rustfmt.toml`:

```toml
edition = "2021"
max_width = 100
tab_spaces = 4
use_small_heuristics = "Max"
imports_granularity = "Crate"
group_imports = "StdExternalCrate"
```

### lint-deny-unsafe
Use `#![forbid(unsafe_code)]` in all crates except `antec-sandbox` (which needs unsafe for WASM/FFI).

### lint-unused-deps
Periodically audit with `cargo machete` to remove unused dependencies.

---

## Category 14: Anti-Patterns [REFERENCE]

Prefix: `anti-`

These are patterns to **avoid**. Each anti-pattern lists the correct alternative.

### anti-unwrap-abuse
**Don't:** `.unwrap()` everywhere.
**Do:** Use `?` with proper error types. See `err-no-unwrap`.

### anti-clone-everything
**Don't:** `.clone()` to satisfy the borrow checker.
**Do:** Restructure ownership. Use references, `Cow`, or `Arc`. See `own-borrow-over-clone`.

### anti-string-typing
**Don't:** Pass raw `String` for IDs, names, paths.
**Do:** Use newtypes. See `api-newtype-ids`.

### anti-god-struct
**Don't:** One struct with 20+ fields that does everything.
**Do:** Decompose into focused types with single responsibility.

### anti-mutex-await
**Don't:** Hold `std::sync::Mutex` across `.await`.
**Do:** Use `tokio::sync::Mutex`. See `own-interior-mut-async`.

### anti-block-in-async
**Don't:** Call blocking I/O or CPU-heavy code in async context.
**Do:** Use `spawn_blocking`. See `async-spawn-blocking`.

### anti-recursive-async
**Don't:** Recursive async functions (requires boxing, stack overflow risk).
**Do:** Restructure as iterative with explicit stack.

### anti-unbounded-channel
**Don't:** `mpsc::unbounded_channel()` for data from external sources.
**Do:** Use bounded channels with backpressure. See `async-stream-backpressure`.

### anti-match-catchall
**Don't:** `_ =>` on enums you own.
**Do:** Match exhaustively so compiler catches new variants.

### anti-premature-optimize
**Don't:** Optimize without profiling.
**Do:** Benchmark first with `criterion`, then optimize hot paths. See `opt-bench-before`.

### anti-panic-error
**Don't:** `panic!` for recoverable errors.
**Do:** Return `Result`. See `err-panic-never`.

### anti-bare-sql
**Don't:** Build SQL with string concatenation.
**Do:** Use parameterized queries (`?` placeholders). Prevent SQL injection.

### anti-hardcoded-timeout
**Don't:** Hardcode timeout values deep in code.
**Do:** Make timeouts configurable with sensible defaults.

### anti-ignore-error
**Don't:** `let _ = fallible_fn();` silently discarding errors.
**Do:** Handle or explicitly log: `fallible_fn().log_err();`

### anti-over-abstract
**Don't:** Create abstractions for one-time operations.
**Do:** Three similar lines > premature abstraction. Abstract when the pattern repeats 3+ times.

---

## Cargo.toml Profiles

```toml
[workspace]
resolver = "2"
members = ["crates/*"]

[workspace.dependencies]
tokio = { version = "1", features = ["full"] }
axum = "0.7"
serde = { version = "1", features = ["derive"] }
serde_json = "1"
rusqlite = { version = "0.32", features = ["bundled", "vtab", "backup"] }
thiserror = "2"
anyhow = "1"
tracing = "0.1"
tracing-subscriber = { version = "0.3", features = ["env-filter"] }
uuid = { version = "1", features = ["v4", "serde"] }
chrono = { version = "0.4", features = ["serde"] }
clap = { version = "4", features = ["derive"] }
wasmtime = "27"

[profile.release]
opt-level = 3
lto = "fat"
codegen-units = 1
panic = "abort"
strip = "symbols"

[profile.dev]
opt-level = 0
debug = true

[profile.bench]
inherits = "release"
debug = true
strip = "none"
```

---

## Quick Reference Card

| What | Rule |
|------|------|
| Error in library crate | `thiserror` enum per crate |
| Error in main.rs | `anyhow` with `.context()` |
| Shared state in async | `Arc<tokio::sync::Mutex<T>>` |
| Read-heavy shared state | `Arc<tokio::sync::RwLock<T>>` |
| CPU-heavy work in async | `tokio::task::spawn_blocking` |
| Fan-in channel | `tokio::sync::mpsc` |
| Fan-out channel | `tokio::sync::broadcast` |
| Config broadcast | `tokio::sync::watch` |
| One-shot response | `tokio::sync::oneshot` |
| Parallel tasks | `JoinSet` + `Semaphore` |
| ID types | Newtype wrapper: `SessionId(Uuid)` |
| Optional config | `Option<T>` + `#[serde(default)]` |
| String in params | `&str` or `impl Into<String>` |
| SQL operations | Batched in transactions |
| External timeouts | `tokio::time::timeout` |
| Graceful shutdown | `CancellationToken` |
| Compile-time maps | `phf` or `LazyLock` |
