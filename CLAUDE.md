# Antec — Claude Code Instructions

## Project Identity

Antec is a self-hosted personal AI assistant. Single Rust binary, zero mandatory dependencies, all data local. 15-crate Cargo workspace under `crates/`. Full PRD in `prd/` (30 documents).

**Do not start coding without reading `prd/01-ARCHITECTURE.md` first.**

---

## Mandatory Build Verification

Run after EVERY feature, fix, or refactoring:

```bash
cargo build --workspace --lib
cargo test --workspace
cargo clippy --workspace --all-targets -- -D warnings
cargo fmt --all -- --check
```

All four must pass. If clippy or fmt fails, fix before committing. No exceptions.

---

## Tech Stack (Quick Reference)

| Layer | Choice | Notes |
|-------|--------|-------|
| Language | Rust 2021 edition | Async via Tokio 1.x |
| HTTP/WS | Axum (Tower-based) | REST + WebSocket + SSE |
| Database | SQLite 3 (rusqlite, bundled) | WAL mode, FTS5, 8-conn pool via r2d2 |
| Config | TOML (human) + JSON (API) | 5-layer: defaults → TOML → env → CLI → runtime API |
| WASM | Wasmtime | Fuel metering + epoch interrupts |
| Serialization | serde + serde_json | MessagePack for internal wire |
| Crypto | ChaCha20-Poly1305 | Secret vault, HMAC-SHA256 audit chain |
| CLI | clap | Subcommands, env fallback |
| Frontend | Vanilla HTML/CSS/JS (ES modules) | Embedded via `rust-embed`, no framework |
| i18n | Compile-time `t!()` macro | EN + PL locales |
| Testing | cargo test + wiremock + assert_cmd | 3 test levels |

---

## Crate Map

| Crate | Purpose | Key Deps |
|-------|---------|----------|
| `antec-core` | Agent loop, LLM providers, context management, model routing | antec-storage, antec-security, antec-i18n, antec-sandbox |
| `antec-gateway` | Axum HTTP/WS server, 126+ REST routes, OTP auth | antec-core, antec-channels, antec-console |
| `antec-channels` | Channel adapters (Discord, WhatsApp, iMessage, Console) | antec-core |
| `antec-tools` | 50+ built-in tools, JSON Schema validation, risk classification | antec-sandbox |
| `antec-memory` | Long-term memory, FTS5, BM25 + TF-IDF + embeddings, temporal decay | antec-storage |
| `antec-sandbox` | WASM + OS sandbox, capabilities, policy engine | — |
| `antec-skills` | Skill loader, manifest parsing, hub client, 5 runtimes | antec-sandbox, antec-tools |
| `antec-mcp` | MCP client (stdio, SSE, HTTP transports) | — |
| `antec-scheduler` | Cron, natural language scheduling, heartbeat | antec-storage, antec-core |
| `antec-security` | Injection detection, secret vault, rate limiting, audit chain | antec-storage |
| `antec-storage` | SQLite pool, 21 migrations, repository pattern | — (leaf crate) |
| `antec-i18n` | Translation macro, EN/PL locale files | — (leaf crate) |
| `antec-console` | Web Console SPA, embedded static assets | — |
| `antec-hands` | Operational capability registry, Hand trait | — |
| `antec-extensions` | Integration templates, credential vault, health monitoring | antec-hands, antec-storage |

Full dependency DAG: `prd/01-ARCHITECTURE.md` §1.2

---

## Development Tooling

### LSP — rust-analyzer

Claude Code uses rust-analyzer for real-time code intelligence. Install the plugin:

```bash
# Ensure rust-analyzer is in PATH
rustup component add rust-analyzer

# Install the Claude Code LSP plugin
/plugin install rust-analyzer-lsp@claude-plugins-official
```

**What this gives you:**
- **Auto-diagnostics** after every edit — type errors, missing imports, lifetime issues caught immediately
- **Code navigation** — go-to-definition, find-references, hover-for-types, call hierarchy
- **Use LSP before grep** — prefer `goToDefinition`, `findReferences`, `hover` over text search when navigating Rust code

**LSP operations available:**
- `goToDefinition` — find where a symbol is defined
- `findReferences` — find all usages of a symbol
- `hover` — get type info and docs
- `documentSymbol` — list all symbols in a file
- `workspaceSymbol` — search symbols across workspace
- `goToImplementation` — find trait implementations
- `prepareCallHierarchy` / `incomingCalls` / `outgoingCalls` — trace call chains

**Use `mcp__ide__getDiagnostics` after edits** to catch compiler errors before running `cargo build`.

### Context7 MCP — Live Documentation

Context7 provides up-to-date library documentation during development. Configure in `.claude/settings.json`:

```json
{
  "mcpServers": {
    "context7": {
      "command": "npx",
      "args": ["-y", "@upstash/context7-mcp@latest"]
    }
  }
}
```

**Use Context7 when:**
- Looking up current Axum, Tokio, rusqlite, wasmtime, serde API signatures
- Checking latest Rust reference for lifetime rules, trait bounds, async patterns
- Verifying crate compatibility before adding dependencies
- Key libraries to query: `rust-lang/reference`, `tokio-rs/tokio`, `tokio-rs/axum`, `rusqlite/rusqlite`, `bytecodealliance/wasmtime`

**Do NOT guess API signatures from memory — verify via Context7 or docs.rs.**

---

## Rust Coding Standards

**Full standard: `skills/rust/SKILL.md`** (179 rules, 14 categories). Key rules enforced on this project:

### Error Handling
- Libraries (`antec-*` crates): `thiserror` for typed errors. Every crate defines its own `Error` enum
- Application layer (`src/main.rs`): `anyhow` for ergonomic propagation
- **Never** `.unwrap()` or `.expect()` in non-test code. Use `?` operator
- Chain errors with context: `.map_err(|e| Error::Storage(e))?` or `.context("loading config")?`

### Ownership & Borrowing
- Borrow (`&T`) over clone. Clone only when ownership transfer is required
- Prefer `&str` over `String` in function parameters
- Use `Arc<T>` for shared ownership across async tasks, never `Rc<T>` in async code
- Interior mutability: `tokio::sync::Mutex` (not `std::sync::Mutex`) when holding across `.await`

### Async Patterns
- Tokio is the only runtime. Never `block_on` inside async context
- Never hold `std::sync::Mutex` across `.await` points — use `tokio::sync::Mutex`
- Offload CPU-heavy work to `tokio::task::spawn_blocking`
- Channels: `mpsc` for fan-in, `broadcast` for fan-out, `watch` for config, `oneshot` for request-reply
- Structured concurrency: `JoinSet` for parallel tasks with bounded concurrency

### API Design
- Newtype wrappers for IDs: `SessionId(Uuid)`, `ToolName(String)`, not bare primitives
- Builder pattern for complex structs (3+ optional fields)
- Accept `impl Into<String>` for string params, return concrete types
- Traits for plugin boundaries: `LlmProvider`, `Channel`, `MemoryBackend`, `Tool`
- Seal internal traits with `mod sealed { pub trait Sealed {} }`

### Naming Conventions
- Types: `PascalCase`. Functions/methods: `snake_case`. Constants: `SCREAMING_SNAKE_CASE`
- Constructors: `new()` or `with_*()`. Conversions: `as_*()` (cheap), `to_*()` (expensive), `into_*()` (ownership)
- Boolean methods: `is_*()`, `has_*()`, `can_*()`
- Crate-level re-exports via `pub use` in `lib.rs` for public API

### Memory & Performance
- Preallocate: `Vec::with_capacity()` when size is known
- Prefer `&[u8]` and `Cow<'_, str>` in parsing hot paths
- Use `SmallVec` for small, fixed-size collections (< 8 elements)
- String building: `String::with_capacity` + `push_str`, not repeated `format!`

### Project Structure
- One type per file for major types (agent loop, providers, etc.)
- `mod.rs` only for re-exports, not logic
- Public API surface in `lib.rs` — internal modules stay `pub(crate)`
- Feature flags for optional channels: `--features "discord whatsapp imessage"`

---

## Architecture Constraints

### Dependency Rules
- Crates form a DAG. **No circular dependencies.** If crate A depends on B, B must not depend on A
- Leaf crates (`antec-storage`, `antec-i18n`) depend only on external crates
- Cross-crate communication through traits defined in the dependent crate
- `antec-core` is the orchestrator — it depends on tools, memory, MCP, hands

### Config Pattern
Every new config field requires:
1. Struct field with `#[serde(default)]`
2. Entry in `Default` impl
3. `Serialize` + `Deserialize` derives
4. Documentation in `prd/14-CONFIGURATION.md`

### Route Registration
New API routes require:
1. Handler function in `routes.rs` (or appropriate route module)
2. Registration in the Axum router in `server.rs`
3. JSON request/response types with serde derives
4. Integration test proving the endpoint works end-to-end

### Tool Registration
New tools require:
1. Struct implementing the `Tool` trait
2. JSON Schema for input/output
3. Risk classification: `safe`, `moderate`, or `dangerous`
4. Registration in the tool registry during boot
5. Unit test for the tool logic + integration test for end-to-end execution

---

## Workflow Orchestration

### 1. Plan Before Build
- Enter plan mode for ANY non-trivial task (3+ steps or architectural decisions)
- If something goes sideways, STOP and re-plan — don't keep pushing
- Write detailed specs upfront to reduce ambiguity
- Ask yourself: "Would a staff engineer approve this approach?"

### 2. Subagent Strategy
- Use subagents liberally to keep main context window clean
- Offload research, exploration, and parallel analysis to subagents
- One task per subagent for focused execution
- For complex problems, parallelize with multiple subagents

### 3. Verification Before Done
- Never mark a task complete without proving it works
- Run all four mandatory checks (build, test, clippy, fmt)
- Diff behavior between main and your changes when relevant
- Demonstrate correctness — don't just claim it

### 4. Autonomous Bug Fixing
- When given a bug report: just fix it. Don't ask for hand-holding
- Point at logs, errors, failing tests — then resolve them
- Zero context switching required from the user
- Go fix failing CI tests without being told how

### 5. Demand Elegance (Balanced)
- For non-trivial changes: pause and ask "is there a more elegant way?"
- If a fix feels hacky: "Knowing everything I know now, implement the elegant solution"
- Skip this for simple, obvious fixes — don't over-engineer

### 6. Self-Improvement Loop
- After ANY correction from the user: update `tasks/lessons.md` with the pattern
- Write rules for yourself that prevent the same mistake
- Review lessons at session start

---

## Task Management

1. **Plan First**: Write plan to `tasks/todo.md` with checkable items
2. **Verify Plan**: Check in before starting implementation
3. **Track Progress**: Mark items complete as you go
4. **Explain Changes**: High-level summary at each step
5. **Document Results**: Add review section to `tasks/todo.md`
6. **Capture Lessons**: Update `tasks/lessons.md` after corrections

---

## Testing Strategy

Every feature ships with tests. Three levels.

### Unit Tests
- Location: `crates/<crate>/src/*.rs` — inline `#[cfg(test)] mod tests`
- Run: `cargo test -p antec-<crate>`
- Mock trait objects — never call real APIs in unit tests
- Every public function: at least one happy-path + one error-path test
- Async tests: `#[tokio::test]`
- Filesystem tests: `tempfile` crate — never write to real paths

### Integration Tests
- Location: `tests/` at workspace root
- Run: `cargo test --test <name>`
- Infrastructure: in-memory SQLite, localhost Axum on random port, mock LLM
- Each test spins up isolated env — no shared state
- Use `tests/common/mod.rs` for shared `TestEnv` setup

### End-to-End Tests
- Location: `tests/e2e/`
- Run: `cargo test --test e2e`
- Black box: build binary → start as subprocess → interact via HTTP/WS
- Timeout: 30s per test. Slow E2E test = bug
- `assert_cmd` + `predicates` for CLI, `tokio-tungstenite` for WS, `reqwest` for HTTP

### Frontend Tests
- Location: `crates/antec-console/frontend/tests/`
- Unit: pure JS function tests
- DOM: component rendering, keyboard nav, ARIA
- Accessibility: axe-core audit — zero AA violations

### Test Rules
- No test without assertion
- No flaky tests — fix or remove, never `#[ignore]`-and-forget
- No real network calls — all external services mocked, tests pass offline
- No shared mutable state
- Tests are documentation: `test_memory_search_returns_results_ranked_by_bm25`
- **Run tests before marking any task complete**

---

## Web Console Constraints

- **Monochrome**: Black/white/gray only. Color reserved for status (green/amber/red)
- **Terminal-inspired**: JetBrains Mono, minimal chrome, dense information
- **Responsive**: mobile-first. Breakpoints: <768px, 768-1024px, >1024px
- **WCAG 2.1 AA**: keyboard nav, screen reader, ARIA labels, focus management
- **CSS variables only**: use design tokens from `prd/19-UI.md` §2
- **No heavy deps**: SSE for real-time. Minimal JS
- **Co-develop**: web UI ships simultaneously with backend features it monitors/configures

Full UI spec: `prd/19-UI.md`

---

## Common Pitfalls

| Pitfall | Fix |
|---------|-----|
| Circular crate dependency | Move shared types to leaf crate or define trait in the dependent |
| `std::sync::Mutex` across `.await` | Use `tokio::sync::Mutex` |
| Missing `#[serde(default)]` on config field | Build breaks on existing configs without the new field |
| Route handler not registered in router | Endpoint returns 404 despite handler existing |
| `.unwrap()` in production code | Use `?` with proper error type. `.unwrap()` only in tests |
| SQLite pool exhaustion | Ensure connections are returned. Check r2d2 pool size (8) |
| Forgot to run migrations | New tables/columns missing at runtime. Check `antec-storage` migration list |
| WASM fuel/epoch not set | Untrusted skill code can loop forever. Always configure limits |
| Hardcoded UI colors | Use CSS variables from design tokens |
| Test depends on network | Mock all external services. Tests must pass offline |
| Large context window | Don't dump full files into context. Use targeted reads and LSP navigation |

---

## Session Continuity

### HANDOVER Protocol
When finishing a phase, wave, or when context is nearly exhausted:

1. Update `tasks/HANDOVER.md` with:
   - What was completed (with file paths)
   - What is in progress (current state)
   - What is planned next (ordered)
   - Any blockers or decisions needed
   - Relevant test results
2. This file is critical for work continuation across context resets

### Core Principles
- **Simplicity First**: make every change as simple as possible
- **No Laziness**: find root causes. No temporary fixes. Senior developer standards
- **Minimize Impact**: changes touch only what's necessary
- **Data Sovereignty**: all data stays local. No telemetry, no cloud sync
- **Defense in Depth**: 9 security layers. No single bypass compromises the system

---

## Key File References

| What | Where |
|------|-------|
| Architecture blueprint | `prd/01-ARCHITECTURE.md` |
| Agent loop & LLM providers | `prd/02-CORE.md` |
| HTTP server & API routes | `prd/03-GATEWAY.md` |
| Database schema (all tables) | `prd/30-DATABASE.md` |
| Security layers | `prd/07-SECURITY.md` |
| UI design system | `prd/19-UI.md` |
| Configuration schema | `prd/14-CONFIGURATION.md` |
| Deployment & build | `prd/29-DEPLOYMENT.md` |
| Rust coding standard | `skills/rust/SKILL.md` |
| Current tasks | `tasks/todo.md` |
| Lessons learned | `tasks/lessons.md` |
| Session handover | `tasks/HANDOVER.md` |
