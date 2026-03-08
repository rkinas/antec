# Antec — AI Agent Development Instructions

> This file provides project context and coding standards for any AI coding assistant (OpenAI Codex, Cursor, Windsurf, Copilot Workspace, Cline, Aider, etc.). For Claude Code-specific instructions, see `CLAUDE.md`.

---

## Project Identity

**Antec** is a self-hosted personal AI assistant. Single Rust binary, zero mandatory dependencies, all data local. 15-crate Cargo workspace under `crates/`. Full PRD in `prd/` (30 documents).

**Read `prd/01-ARCHITECTURE.md` before writing any code.**

---

## Mandatory Checks

Run after EVERY change:

```bash
cargo build --workspace --lib
cargo test --workspace
cargo clippy --workspace --all-targets -- -D warnings
cargo fmt --all -- --check
```

All four must pass before any commit. No exceptions.

---

## Tech Stack

| Layer | Choice |
|-------|--------|
| Language | Rust 2021 edition, async via Tokio 1.x |
| HTTP/WS | Axum (Tower-based) — REST + WebSocket + SSE |
| Database | SQLite 3 (rusqlite, bundled) — WAL mode, FTS5, r2d2 pool (8 conns) |
| Config | TOML (human) + JSON (API) — 5-layer precedence |
| WASM Sandbox | Wasmtime — fuel metering + epoch interrupts |
| Serialization | serde + serde_json; MessagePack for internal wire |
| Crypto | ChaCha20-Poly1305 secret vault, HMAC-SHA256 audit chain |
| CLI | clap |
| Frontend | Vanilla HTML/CSS/JS (ES modules), embedded via rust-embed |
| i18n | Compile-time `t!()` macro — EN + PL |

---

## Crate Map

```
antec (workspace root)
 ├── src/main.rs                     # Binary entrypoint, CLI, boot
 └── crates/
      ├── antec-core/                # Agent loop, LLM providers, routing
      ├── antec-gateway/             # Axum HTTP/WS, REST routes, auth
      ├── antec-channels/            # Discord, WhatsApp, iMessage, Console
      ├── antec-tools/               # 50+ tools, risk classification
      ├── antec-memory/              # FTS5, BM25 + TF-IDF + embeddings
      ├── antec-storage/             # SQLite pool, migrations, repos
      ├── antec-security/            # Injection detection, vault, audit
      ├── antec-skills/              # Skill loader, 5 runtimes
      ├── antec-scheduler/           # Cron, NL scheduling, heartbeat
      ├── antec-sandbox/             # WASM + OS sandbox, policy engine
      ├── antec-mcp/                 # MCP client (stdio/SSE/HTTP)
      ├── antec-extensions/          # Integration templates, credentials
      ├── antec-i18n/                # Translation macro, locale files
      ├── antec-console/             # Web Console SPA (embedded)
      └── antec-hands/               # Operational capabilities
```

Dependency DAG: `prd/01-ARCHITECTURE.md` §1.2. **No circular dependencies.**

---

## Rust Coding Standards

Full standard: `.claude/skills/rust/SKILL.md` (179 rules, 14 categories). Key rules:

### Error Handling
- Library crates: `thiserror` for typed error enums per crate
- Application layer: `anyhow` for ergonomic propagation
- **Never** `.unwrap()` or `.expect()` in production code — use `?`
- Chain errors with context: `.map_err(|e| Error::Storage(e))?`

### Ownership & Borrowing
- Borrow (`&T`) over clone. Clone only when ownership transfer is required
- `&str` over `String` in function parameters
- `Arc<T>` for shared ownership in async — never `Rc<T>` in async code
- `tokio::sync::Mutex` (not `std::sync::Mutex`) when holding across `.await`

### Async Patterns
- Tokio-only runtime. Never `block_on` inside async
- Never hold `std::sync::Mutex` across `.await`
- CPU-heavy work: `tokio::task::spawn_blocking`
- Channels: `mpsc` (fan-in), `broadcast` (fan-out), `watch` (config), `oneshot` (request-reply)
- Parallel tasks: `JoinSet` with bounded concurrency

### API Design
- Newtype wrappers for IDs: `SessionId(Uuid)`, not bare `Uuid`
- Builder pattern for complex structs (3+ optional fields)
- Accept `impl Into<String>`, return concrete types
- Traits for plugin boundaries: `LlmProvider`, `Channel`, `MemoryBackend`, `Tool`

### Naming
- Types: `PascalCase`. Functions: `snake_case`. Constants: `SCREAMING_SNAKE_CASE`
- Constructors: `new()`, `with_*()`. Conversions: `as_*()` (cheap), `to_*()` (expensive), `into_*()` (ownership)
- Booleans: `is_*()`, `has_*()`, `can_*()`

### Memory & Performance
- `Vec::with_capacity()` when size is known
- `&[u8]` and `Cow<'_, str>` in parsing hot paths
- `SmallVec` for small collections (< 8 elements)
- `String::with_capacity` + `push_str`, not repeated `format!`

---

## Architecture Constraints

### Dependency Rules
- Crates form a DAG. No circular dependencies
- Leaf crates (`antec-storage`, `antec-i18n`) depend only on external crates
- Cross-crate communication through traits

### Config Pattern
Every new config field needs: struct field with `#[serde(default)]`, `Default` impl entry, serde derives, docs in `prd/14-CONFIGURATION.md`.

### Route Registration
New API routes need: handler in route module, router registration in `server.rs`, request/response types with serde, integration test.

### Tool Registration
New tools need: `Tool` trait impl, JSON Schema, risk classification (`safe`/`moderate`/`dangerous`), registry entry, tests.

---

## Testing Strategy

### Unit Tests
- Location: `crates/<crate>/src/*.rs` — inline `#[cfg(test)] mod tests`
- Run: `cargo test -p antec-<crate>`
- Mock trait objects — never call real APIs
- Every public function: happy-path + error-path test
- Async: `#[tokio::test]`. Filesystem: `tempfile` crate

### Integration Tests
- Location: `tests/` at workspace root
- In-memory SQLite, localhost Axum on random port, mock LLM
- Each test has isolated environment — no shared state

### End-to-End Tests
- Location: `tests/e2e/`
- Black box: build binary → subprocess → HTTP/WS interaction
- 30s timeout per test. `assert_cmd` for CLI, `tokio-tungstenite` for WS

### Test Rules
- No test without assertion
- No flaky tests — fix or remove
- No real network calls — offline-capable
- No shared mutable state
- Descriptive names: `test_memory_search_returns_results_ranked_by_bm25`
- Run `cargo test` before marking any task complete

---

## Web Console Constraints

- Monochrome palette (black/white/gray). Color for status only (green/amber/red)
- Terminal-inspired: JetBrains Mono, dense information, minimal chrome
- Responsive: mobile-first. Breakpoints: <768px, 768-1024px, >1024px
- WCAG 2.1 AA: keyboard nav, screen reader, ARIA, focus management
- CSS variables only — design tokens from `prd/19-UI.md`
- No heavy dependencies. SSE for real-time
- Web UI ships simultaneously with backend features

---

## Workflow Guidelines

### Planning
- Plan before building for any non-trivial task (3+ steps or architectural decisions)
- If the approach isn't working, stop and re-plan — don't push through
- Write specs upfront to reduce ambiguity

### Quality Bar
- Ask: "Would a staff engineer approve this?"
- Run all four mandatory checks before declaring done
- Demonstrate correctness — don't just claim it
- Find root causes. No temporary fixes

### Task Tracking
- Write plans to `tasks/todo.md` with checkable items
- Track progress as you go
- Update `tasks/HANDOVER.md` when finishing phases or sessions

### Principles
- **Simplicity First**: every change as simple as possible
- **Minimize Impact**: touch only what's necessary
- **No Laziness**: senior developer standards at all times
- **Data Sovereignty**: all data stays local
- **Defense in Depth**: 9 security layers

---

## Common Pitfalls

| Pitfall | Fix |
|---------|-----|
| Circular crate dependency | Move shared types to leaf crate or define trait in the dependent |
| `std::sync::Mutex` across `.await` | Use `tokio::sync::Mutex` |
| Missing `#[serde(default)]` on config field | Breaks existing configs |
| Route not registered in router | 404 despite handler existing |
| `.unwrap()` in production | Use `?` with proper error type |
| SQLite pool exhaustion | Ensure connections returned, check pool size |
| Forgot migrations | New tables missing at runtime |
| WASM fuel/epoch not set | Skill code can loop forever |
| Hardcoded UI colors | Use CSS variables |
| Test depends on network | Mock all external services |

---

## Key File References

| What | Where |
|------|-------|
| Architecture | `prd/01-ARCHITECTURE.md` |
| Agent loop | `prd/02-CORE.md` |
| API routes | `prd/03-GATEWAY.md` |
| Database schema | `prd/30-DATABASE.md` |
| Security | `prd/07-SECURITY.md` |
| UI design | `prd/19-UI.md` |
| Configuration | `prd/14-CONFIGURATION.md` |
| Deployment | `prd/29-DEPLOYMENT.md` |
| Rust standard | `.claude/skills/rust/SKILL.md` |
| Tasks | `tasks/todo.md` |
| Handover | `tasks/HANDOVER.md` |
