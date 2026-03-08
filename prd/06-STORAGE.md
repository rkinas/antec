# 06 -- Storage Layer (SQLite)

> **Module Goal:** Provide the persistence foundation for the entire system — 21 SQLite tables, connection pooling, migrations, full-text search, and repository traits — enabling zero-dependency data storage with ACID guarantees.

### Why This Module Exists

Every subsystem in Antec needs to persist data: conversations, memories, tool approvals, model usage, audit logs, schedules, and more. Using an external database would violate Antec's "zero mandatory dependencies" principle and complicate deployment.

The Storage module solves this with SQLite — a single file database embedded in the binary. It provides 21 tables covering all system needs, r2d2 connection pooling for concurrent access, ordered migrations for schema evolution, FTS5 virtual tables for full-text search, and clean repository traits that decouple storage logic from business logic. WAL mode ensures readers never block writers.

### Business Benefits

| Benefit | Description |
|---------|-------------|
| **Zero dependencies** | SQLite embedded in binary — no external database to install or maintain |
| **ACID compliance** | Transactions, WAL mode, and integrity checks ensure data safety |
| **Full-text search** | FTS5 virtual table enables fast keyword search across memories |
| **Clean abstractions** | Repository traits allow testing with in-memory databases |
| **Automatic migrations** | Schema evolves safely across versions without manual intervention |
| **Single-file backup** | Entire database is one file — trivial to backup, restore, or move |

This document specifies the complete storage layer: database configuration, migration system, schema definitions, repository pattern, workspace manager, and data lifecycle operations.

---

## 1. Database Configuration

### 1.1 Engine

SQLite 3 with WAL (Write-Ahead Logging) mode. Bundled via the `rusqlite` crate -- no external database server required.

### 1.2 Connection Pool

| Parameter | Value |
|-----------|-------|
| Library | `r2d2` with `r2d2_sqlite::SqliteConnectionManager` |
| Max connections | 8 |
| Min idle | 1 |
| Connection customizer | `WalForeignKeys` (applies PRAGMAs on acquire) |

The pool type alias:

```rust
pub type PooledConnection = r2d2::PooledConnection<SqliteConnectionManager>;
```

### 1.3 PRAGMAs

Applied automatically on every connection acquired from the pool via the `WalForeignKeys` customizer:

```sql
PRAGMA journal_mode = WAL;
PRAGMA foreign_keys = ON;
```

### 1.4 Database Paths

| Mode | Path |
|------|------|
| Production | `~/.antec/antec.db` (parent directory created automatically if missing) |
| Tests | `:memory:` (in-memory SQLite, no disk I/O) |
| Custom | Any path passed to `Database::open(path)` |

### 1.5 Database Handle

```rust
#[derive(Clone)]
pub struct Database {
    pool: Pool<SqliteConnectionManager>,
}
```

**Public API:**

| Method | Signature | Description |
|--------|-----------|-------------|
| `open` | `fn open(path: &Path) -> Result<Self>` | Open or create a file-backed database, run migrations |
| `open_memory` | `fn open_memory() -> Result<Self>` | Create an in-memory database, run migrations |
| `conn` | `fn conn(&self) -> Result<PooledConnection>` | Obtain a connection from the pool |

---

## 2. Migration System

### 2.1 Strategy

- **22 versioned migrations** tracked via the `user_version` PRAGMA.
- Applied sequentially on startup in a single connection.
- Each migration is an embedded SQL string (`include_str!`).
- Idempotent: re-running migrations on an already-migrated database is safe.

### 2.2 Version Tracking

```sql
PRAGMA user_version;  -- read current version (0 = fresh database)
PRAGMA user_version = N;  -- set after applying migration N
```

### 2.3 Migration Index

| Version | File | Purpose |
|---------|------|---------|
| 1 | `001_initial.sql` | Core tables: sessions, messages, audit_log, tool_policies, approval_requests, model_usage. Indexes on session_id, timestamp |
| 2 | `002_security.sql` | Add `hmac_sig` column to audit_log. Create `secrets` table |
| 3 | `003_memory.sql` | Create `memories` table, `memory_fts` FTS5 virtual table, sync triggers (INSERT/DELETE/UPDATE), indexes on key/source/category/expires_at |
| 4 | `004_scheduler.sql` | Create `cron_jobs` table with indexes on next_run and enabled |
| 5 | `005_skills.sql` | Create initial `skills` table (v1 schema with version/manifest columns) |
| 6 | `006_cron_action.sql` | Add `action_type` column to cron_jobs (default: `'send_message'`) |
| 7 | `007_one_shot.sql` | Add `one_shot` column to cron_jobs (default: `0`) |
| 8 | `008_memory_archive.sql` | Add `archived` column to memories (default: `0`) |
| 9 | `009_auth_tokens.sql` | Create `auth_tokens` table |
| 10 | `010_tool_enabled.sql` | Add `enabled` column to tool_policies (default: `1`) |
| 11 | `011_skills_v2.sql` | Recreate skills table with new schema: drop version/manifest/updated_at, add content/skill_type/has_code/source_url. Migrate data from v1 |
| 12 | `012_mcp_servers.sql` | Create `mcp_servers` table with unique name index |
| 13 | `013_memory_pinned.sql` | Add `pinned` column to memories (default: `0`), partial index on pinned=1 |
| 14 | `014_memory_embeddings.sql` | Create `memory_embeddings` table with ON DELETE CASCADE |
| 15 | `015_session_archive.sql` | Add `archived_at` column to sessions (default: NULL), index on archived_at |
| 16 | `016_memory_snapshots.sql` | Create `memory_snapshots` table |
| 17 | `017_routing_mode.sql` | Add `routing_mode` column to model_usage (default: NULL) |
| 18 | `018_workspace_repl.sql` | Create `workspace_file_versions` and `repl_history` tables with indexes |
| 19 | `019_agents.sql` | Create `agents` table with unique constraint on is_default=1 |
| 20 | `020_memory_links.sql` | Create `memory_links` table with UNIQUE(source_id, target_id, relation), indexes on source/target/relation |
| 21 | `021_model_instances.sql` | Create `model_instances` table. Add `model_instance_id` column to agents (with safety-net fallback) |
| 22 | `022_memory_ops_log.sql` | Create `memory_ops_log` table for dedicated memory mutation audit trail |

### 2.4 Safety Net (Migration 21)

Migration 21 includes a post-migration safety check that:

1. Verifies the `model_instances` table exists; creates it if missing.
2. Verifies the `model_instance_id` column exists on `agents`; adds it if missing.

This handles partially-applied migrations from prior builds.

---

## 3. Complete Schema

All timestamps are `INTEGER` Unix epoch seconds unless otherwise noted. All primary keys are `TEXT` (UUIDs) except where noted as `INTEGER PRIMARY KEY AUTOINCREMENT`.

### 3.1 sessions

```sql
CREATE TABLE sessions (
    id         TEXT    PRIMARY KEY,
    channel    TEXT    NOT NULL,          -- 'console', 'discord', 'whatsapp', 'imessage'
    channel_id TEXT,                      -- channel-specific conversation identifier
    created_at INTEGER NOT NULL,          -- Unix timestamp
    updated_at INTEGER NOT NULL,          -- Unix timestamp, updated on each message
    metadata   TEXT,                      -- optional JSON metadata
    archived_at INTEGER DEFAULT NULL      -- NULL = active, timestamp = archived
);

CREATE INDEX idx_sessions_archived_at ON sessions(archived_at);
```

**Rust model:**

```rust
pub struct SessionRow {
    pub id: String,
    pub channel: String,
    pub channel_id: Option<String>,
    pub created_at: i64,
    pub updated_at: i64,
    pub metadata: Option<String>,
    pub archived_at: Option<i64>,
}
```

### 3.2 messages

```sql
CREATE TABLE messages (
    id          TEXT    PRIMARY KEY,
    session_id  TEXT    NOT NULL REFERENCES sessions(id),
    role        TEXT    NOT NULL,          -- 'user', 'assistant', 'system', 'tool'
    content     TEXT    NOT NULL,
    tool_calls  TEXT,                      -- JSON array of tool call objects
    token_count INTEGER,                   -- estimated token count for context management
    created_at  INTEGER NOT NULL
);

CREATE INDEX idx_messages_session_id ON messages(session_id);
```

**Rust model:**

```rust
pub struct MessageRow {
    pub id: String,
    pub session_id: String,
    pub role: String,
    pub content: String,
    pub tool_calls: Option<String>,
    pub token_count: Option<i32>,
    pub created_at: i64,
}
```

### 3.3 audit_log

```sql
CREATE TABLE audit_log (
    id         INTEGER PRIMARY KEY AUTOINCREMENT,
    timestamp  INTEGER NOT NULL,
    actor      TEXT    NOT NULL,          -- 'agent', 'user', 'system', 'scheduler'
    action     TEXT    NOT NULL,          -- e.g. 'tool_executed', 'injection_detected'
    target     TEXT,                      -- target resource (tool name, session id, etc.)
    details    TEXT,                      -- JSON details
    session_id TEXT,                      -- associated session (nullable)
    risk_level TEXT    DEFAULT 'safe',    -- 'safe', 'moderate', 'dangerous', 'unknown'
    hmac_sig   TEXT                       -- chained HMAC-SHA256 signature (added in migration 2)
);

CREATE INDEX idx_audit_log_session_id ON audit_log(session_id);
CREATE INDEX idx_audit_log_timestamp  ON audit_log(timestamp);
```

**Rust model:**

```rust
pub struct AuditLogRow {
    pub id: i64,
    pub timestamp: i64,
    pub actor: String,
    pub action: String,
    pub target: Option<String>,
    pub details: Option<String>,
    pub session_id: Option<String>,
    pub risk_level: String,
    pub hmac_sig: Option<String>,
}
```

### 3.4 secrets

```sql
CREATE TABLE secrets (
    id         TEXT PRIMARY KEY,
    name       TEXT NOT NULL UNIQUE,      -- human-readable secret name
    ciphertext BLOB NOT NULL,             -- ChaCha20-Poly1305 encrypted data
    nonce      BLOB NOT NULL,             -- 12-byte random nonce
    created_at INTEGER NOT NULL
);
```

**Rust model:**

```rust
pub struct SecretRow {
    pub id: String,
    pub name: String,
    pub ciphertext: Vec<u8>,
    pub nonce: Vec<u8>,
    pub created_at: i64,
}
```

### 3.5 memories

```sql
CREATE TABLE memories (
    id           TEXT    PRIMARY KEY,
    key          TEXT    NOT NULL,         -- short identifier / title
    content      TEXT    NOT NULL,         -- full memory content
    category     TEXT,                     -- 'fact', 'preference', 'decision', 'task', 'contact', 'skill', 'relationship', 'event', 'conversation', 'other'
    tags         TEXT,                     -- JSON array of tag strings
    source       TEXT    NOT NULL DEFAULT 'user',  -- 'user', 'auto', 'pending'
    importance   REAL    NOT NULL DEFAULT 0.5,     -- 0.0 to 1.0
    access_count INTEGER NOT NULL DEFAULT 0,
    created_at   INTEGER NOT NULL,
    updated_at   INTEGER NOT NULL,
    expires_at   INTEGER,                 -- optional TTL (NULL = permanent)
    archived     INTEGER NOT NULL DEFAULT 0,       -- 0 = active, 1 = archived by decay
    pinned       INTEGER NOT NULL DEFAULT 0        -- 1 = always recalled regardless of threshold
);

CREATE INDEX idx_memories_key        ON memories(key);
CREATE INDEX idx_memories_source     ON memories(source);
CREATE INDEX idx_memories_category   ON memories(category);
CREATE INDEX idx_memories_expires_at ON memories(expires_at);
CREATE INDEX idx_memories_pinned     ON memories(pinned) WHERE pinned = 1;
```

**Rust model:**

```rust
pub struct MemoryRow {
    pub id: String,
    pub key: String,
    pub content: String,
    pub category: Option<String>,
    pub tags: Option<String>,
    pub source: String,
    pub importance: f64,
    pub access_count: i32,
    pub created_at: i64,
    pub updated_at: i64,
    pub expires_at: Option<i64>,
    pub archived: bool,
    pub pinned: bool,
}
```

### 3.6 memory_fts (FTS5 Virtual Table)

```sql
CREATE VIRTUAL TABLE memory_fts USING fts5(
    key, content, tags,
    content='memories',
    content_rowid='rowid',
    tokenize='porter unicode61'
);
```

**Sync triggers** keep the FTS index synchronized with the `memories` table:

```sql
-- After INSERT
CREATE TRIGGER memory_ai AFTER INSERT ON memories BEGIN
    INSERT INTO memory_fts(rowid, key, content, tags)
    VALUES (new.rowid, new.key, new.content, new.tags);
END;

-- After DELETE
CREATE TRIGGER memory_ad AFTER DELETE ON memories BEGIN
    INSERT INTO memory_fts(memory_fts, rowid, key, content, tags)
    VALUES ('delete', old.rowid, old.key, old.content, old.tags);
END;

-- After UPDATE
CREATE TRIGGER memory_au AFTER UPDATE ON memories BEGIN
    INSERT INTO memory_fts(memory_fts, rowid, key, content, tags)
    VALUES ('delete', old.rowid, old.key, old.content, old.tags);
    INSERT INTO memory_fts(rowid, key, content, tags)
    VALUES (new.rowid, new.key, new.content, new.tags);
END;
```

### 3.7 memory_embeddings

```sql
CREATE TABLE memory_embeddings (
    memory_id  TEXT PRIMARY KEY REFERENCES memories(id) ON DELETE CASCADE,
    embedding  BLOB NOT NULL,             -- little-endian f32 byte array
    model      TEXT NOT NULL,             -- e.g. 'all-MiniLM-L6-v2', 'text-embedding-3-small'
    dimensions INTEGER NOT NULL,          -- vector dimensionality (e.g. 384, 1536)
    created_at INTEGER NOT NULL DEFAULT (unixepoch())
);
```

**Rust model:**

```rust
pub struct MemoryEmbeddingRow {
    pub memory_id: String,
    pub embedding: Vec<u8>,   // serialized as little-endian f32 bytes
    pub model: String,
    pub dimensions: i32,
    pub created_at: i64,
}
```

### 3.8 memory_links

```sql
CREATE TABLE memory_links (
    id          INTEGER PRIMARY KEY AUTOINCREMENT,
    source_id   TEXT NOT NULL,            -- outbound memory ID
    target_id   TEXT NOT NULL,            -- inbound memory ID
    relation    TEXT NOT NULL,            -- 'related_to', 'contradicts', 'supersedes', 'part_of'
    metadata    TEXT,                     -- optional JSON metadata
    created_at  INTEGER NOT NULL,
    UNIQUE(source_id, target_id, relation)
);

CREATE INDEX idx_memory_links_source   ON memory_links(source_id);
CREATE INDEX idx_memory_links_target   ON memory_links(target_id);
CREATE INDEX idx_memory_links_relation ON memory_links(relation);
```

**Rust model:**

```rust
pub struct MemoryLinkRow {
    pub id: i64,
    pub source_id: String,
    pub target_id: String,
    pub relation: String,   // related_to | contradicts | supersedes | part_of
    pub metadata: Option<String>,
    pub created_at: i64,
}
```

### 3.9 memory_snapshots

```sql
CREATE TABLE memory_snapshots (
    id            TEXT PRIMARY KEY,
    reason        TEXT,
    created_at    INTEGER NOT NULL,
    memory_count  INTEGER NOT NULL DEFAULT 0,
    size_bytes    INTEGER NOT NULL DEFAULT 0,
    snapshot_data TEXT NOT NULL             -- JSON-serialized array of MemoryRow entries
);
```

**Rust model:**

```rust
pub struct MemorySnapshotRow {
    pub id: String,
    pub reason: Option<String>,
    pub created_at: i64,
    pub memory_count: i64,
    pub size_bytes: i64,
    pub snapshot_data: String,
}
```

### 3.10 tool_policies

```sql
CREATE TABLE tool_policies (
    tool_name  TEXT    PRIMARY KEY,
    risk_class TEXT    NOT NULL,          -- 'safe', 'moderate', 'dangerous'
    policy     TEXT,                      -- 'always_allow', 'always_deny', etc.
    network_scope TEXT,                   -- network access scope
    fs_scope   TEXT,                      -- filesystem access scope
    rate_limit INTEGER,                   -- requests per minute (NULL = default)
    timeout_ms INTEGER,                   -- execution timeout in ms
    updated_at INTEGER NOT NULL,
    enabled    INTEGER NOT NULL DEFAULT 1  -- 0 = disabled, 1 = enabled
);
```

**Rust model:**

```rust
pub struct ToolPolicyRow {
    pub tool_name: String,
    pub risk_class: String,
    pub policy: Option<String>,
    pub network_scope: Option<String>,
    pub fs_scope: Option<String>,
    pub rate_limit: Option<i32>,
    pub timeout_ms: Option<i32>,
    pub updated_at: i64,
    pub enabled: bool,
}
```

### 3.11 approval_requests

```sql
CREATE TABLE approval_requests (
    id          TEXT PRIMARY KEY,
    session_id  TEXT NOT NULL,
    tool_name   TEXT NOT NULL,
    tool_params TEXT NOT NULL,            -- JSON-serialized tool arguments
    risk_level  TEXT NOT NULL,            -- 'safe', 'moderate', 'dangerous'
    status      TEXT DEFAULT 'pending',   -- 'pending', 'approved', 'denied', 'expired'
    scope       TEXT DEFAULT 'once',      -- 'once', 'session', 'always'
    created_at  INTEGER NOT NULL,
    resolved_at INTEGER,                  -- timestamp when resolved
    resolved_by TEXT                      -- who resolved ('user', 'timeout', etc.)
);
```

**Rust model:**

```rust
pub struct ApprovalRequestRow {
    pub id: String,
    pub session_id: String,
    pub tool_name: String,
    pub tool_params: String,
    pub risk_level: String,
    pub status: String,
    pub scope: String,
    pub created_at: i64,
    pub resolved_at: Option<i64>,
    pub resolved_by: Option<String>,
}
```

### 3.12 cron_jobs

```sql
CREATE TABLE cron_jobs (
    id          TEXT PRIMARY KEY,
    name        TEXT NOT NULL,
    cron_expr   TEXT NOT NULL,            -- standard cron expression (5 fields)
    prompt      TEXT NOT NULL DEFAULT '', -- message to send when triggered
    enabled     INTEGER NOT NULL DEFAULT 1,
    next_run    INTEGER,                  -- precomputed next fire timestamp
    last_run    INTEGER,                  -- last execution timestamp
    run_count   INTEGER NOT NULL DEFAULT 0,
    action_type TEXT NOT NULL DEFAULT 'send_message',  -- 'send_message', 'run_tool', etc.
    created_at  INTEGER NOT NULL,
    updated_at  INTEGER NOT NULL,
    one_shot    INTEGER NOT NULL DEFAULT 0  -- 1 = auto-disable after single execution
);

CREATE INDEX idx_cron_jobs_next_run ON cron_jobs(next_run) WHERE enabled = 1;
CREATE INDEX idx_cron_jobs_enabled  ON cron_jobs(enabled);
```

**Rust model:**

```rust
pub struct CronJobRow {
    pub id: String,
    pub name: String,
    pub cron_expr: String,
    pub prompt: String,
    pub enabled: bool,
    pub next_run: Option<i64>,
    pub last_run: Option<i64>,
    pub run_count: i32,
    pub action_type: String,
    pub created_at: i64,
    pub updated_at: i64,
    pub one_shot: bool,
}
```

### 3.13 skills

```sql
-- After migration 011 (skills v2):
CREATE TABLE skills (
    id           TEXT PRIMARY KEY,
    name         TEXT NOT NULL UNIQUE,
    description  TEXT,
    content      TEXT NOT NULL,           -- full SKILL.md text (frontmatter + body)
    enabled      INTEGER DEFAULT 0,
    skill_type   TEXT NOT NULL,           -- 'builtin', 'hub', 'local'
    has_code     INTEGER DEFAULT 0,       -- 1 if scripts/agents directories exist
    source_url   TEXT,                    -- import origin URL (NULL for local/builtin)
    installed_at INTEGER NOT NULL
);

CREATE INDEX idx_skills_name       ON skills(name);
CREATE INDEX idx_skills_enabled    ON skills(enabled);
CREATE INDEX idx_skills_skill_type ON skills(skill_type);
```

**Rust model:**

```rust
pub struct SkillRow {
    pub id: String,
    pub name: String,
    pub description: String,
    pub content: String,
    pub enabled: bool,
    pub skill_type: String,
    pub has_code: bool,
    pub source_url: Option<String>,
    pub installed_at: i64,
}
```

### 3.14 model_usage

```sql
CREATE TABLE model_usage (
    id            INTEGER PRIMARY KEY AUTOINCREMENT,
    timestamp     INTEGER NOT NULL,
    provider      TEXT    NOT NULL,       -- 'anthropic', 'openai', 'google', 'ollama', 'openrouter'
    model         TEXT    NOT NULL,       -- full model ID string
    session_id    TEXT,                   -- associated session (nullable)
    input_tokens  INTEGER NOT NULL,
    output_tokens INTEGER NOT NULL,
    cost_usd      REAL,                  -- estimated cost in USD
    latency_ms    INTEGER,               -- response latency
    routing_mode  TEXT DEFAULT NULL       -- 'simple', 'complex', NULL (unrouted)
);

CREATE INDEX idx_model_usage_timestamp ON model_usage(timestamp);
```

**Rust model:**

```rust
pub struct ModelUsageRow {
    pub id: i64,
    pub timestamp: i64,
    pub provider: String,
    pub model: String,
    pub session_id: Option<String>,
    pub input_tokens: i32,
    pub output_tokens: i32,
    pub cost_usd: Option<f64>,
    pub latency_ms: Option<i32>,
    pub routing_mode: Option<String>,
}
```

### 3.15 auth_tokens

```sql
CREATE TABLE auth_tokens (
    id          TEXT PRIMARY KEY,
    token_hash  TEXT NOT NULL,
    last_used_at INTEGER,
    revoked     INTEGER NOT NULL DEFAULT 0,
    created_at  INTEGER NOT NULL,
    expires_at  INTEGER NOT NULL,
    paired_at   INTEGER NOT NULL
);
```

**Rust model:**

```rust
pub struct AuthTokenRow {
    pub id: String,
    pub token_hash: String,
    pub last_used_at: Option<i64>,
    pub revoked: bool,
    pub created_at: i64,
    pub expires_at: i64,
    pub paired_at: i64,
}
```

### 3.16 mcp_servers

```sql
CREATE TABLE mcp_servers (
    id       TEXT PRIMARY KEY,
    name     TEXT NOT NULL UNIQUE,
    config   TEXT NOT NULL,               -- JSON: {transport, command, args, url, env}
    enabled  INTEGER NOT NULL DEFAULT 1,
    added_at INTEGER NOT NULL
);

```

**Rust model:**

```rust
pub struct McpServerRow {
    pub id: String,
    pub name: String,
    pub config: String,   // JSON-serialized server configuration
    pub enabled: bool,
    pub added_at: i64,
}
```

### 3.17 workspace_file_versions

```sql
CREATE TABLE workspace_file_versions (
    id         TEXT PRIMARY KEY,
    file_path  TEXT    NOT NULL,          -- relative path within workspace
    content    TEXT    NOT NULL,          -- full file content at this version
    version    INTEGER NOT NULL DEFAULT 1,
    created_at INTEGER NOT NULL,
    size_bytes INTEGER NOT NULL DEFAULT 0
);

CREATE INDEX idx_wfv_path         ON workspace_file_versions(file_path);
CREATE INDEX idx_wfv_path_version ON workspace_file_versions(file_path, version DESC);
```

**Rust model:**

```rust
pub struct FileVersionRow {
    pub id: String,
    pub file_path: String,
    pub content: String,
    pub version: i64,
    pub created_at: i64,
    pub size_bytes: i64,
}
```

### 3.18 agents

```sql
CREATE TABLE agents (
    id          TEXT PRIMARY KEY,
    name        TEXT NOT NULL UNIQUE,
    description TEXT NOT NULL DEFAULT '',
    persona     TEXT NOT NULL DEFAULT '',  -- system persona text
    tools       TEXT NOT NULL DEFAULT '[]', -- JSON array of tool names
    skills      TEXT NOT NULL DEFAULT '[]', -- JSON array of skill names
    model       TEXT,                      -- model override (NULL = use default)
    provider    TEXT,                      -- provider override (NULL = use default)
    model_instance_id TEXT REFERENCES model_instances(id),  -- added in migration 21
    enabled     INTEGER NOT NULL DEFAULT 1,
    is_default  INTEGER NOT NULL DEFAULT 0,
    created_at  INTEGER NOT NULL,
    updated_at  INTEGER NOT NULL
);

-- Only one agent can be the default:
CREATE UNIQUE INDEX idx_agents_default ON agents(is_default) WHERE is_default = 1;
```

**Rust model:**

```rust
pub struct AgentDefinitionRow {
    pub id: String,
    pub name: String,
    pub description: String,
    pub persona: String,
    pub tools: String,
    pub skills: String,
    pub model: Option<String>,
    pub provider: Option<String>,
    pub model_instance_id: Option<String>,
    pub enabled: bool,
    pub is_default: bool,
    pub created_at: i64,
    pub updated_at: i64,
}
```

### 3.19 model_instances

```sql
CREATE TABLE model_instances (
    id         TEXT PRIMARY KEY,
    name       TEXT NOT NULL UNIQUE,      -- human-readable instance name
    provider   TEXT NOT NULL,             -- 'anthropic', 'openai', etc.
    model      TEXT NOT NULL,             -- full model ID
    is_default INTEGER NOT NULL DEFAULT 0,
    created_at INTEGER NOT NULL,
    updated_at INTEGER NOT NULL
);
```

**Rust model:**

```rust
pub struct ModelInstanceRow {
    pub id: String,
    pub name: String,
    pub provider: String,
    pub model: String,
    pub is_default: bool,
    pub created_at: i64,
    pub updated_at: i64,
}
```

### 3.20 repl_history

```sql
CREATE TABLE repl_history (
    id         INTEGER PRIMARY KEY AUTOINCREMENT,
    session_id TEXT NOT NULL,
    language   TEXT NOT NULL,             -- 'javascript', 'python'
    code       TEXT NOT NULL,             -- executed code
    output     TEXT,                      -- execution output
    success    INTEGER NOT NULL DEFAULT 1,
    created_at INTEGER NOT NULL
);

CREATE INDEX idx_repl_session ON repl_history(session_id, created_at DESC);
```

**Rust model:**

```rust
pub struct ReplHistoryRow {
    pub id: i64,
    pub session_id: String,
    pub language: String,
    pub code: String,
    pub output: Option<String>,
    pub success: bool,
    pub created_at: i64,
}
```

### 3.21 memory_ops_log

Dedicated mutation audit trail for memory operations. Tracks every create, update, delete, pin, and archive action on memories, providing a complete history of how the memory store evolved.

```sql
CREATE TABLE memory_ops_log (
    id             INTEGER PRIMARY KEY AUTOINCREMENT,
    timestamp      INTEGER NOT NULL,            -- Unix timestamp of the operation
    memory_id      TEXT NOT NULL,               -- references memories(id), not FK to allow logging deletes
    operation      TEXT NOT NULL,               -- 'create', 'update', 'delete', 'pin', 'archive'
    actor          TEXT NOT NULL,               -- 'agent', 'user', 'system', 'skill:<name>'
    source_session TEXT,                        -- which session triggered the op (nullable)
    details        TEXT                         -- JSON diff or reason (nullable)
);
```

**Rust model:**

```rust
pub struct MemoryOpsLogRow {
    pub id: i64,
    pub timestamp: i64,
    pub memory_id: String,
    pub operation: String,
    pub actor: String,
    pub source_session: Option<String>,
    pub details: Option<String>,
}
```

**Design notes:**

- `memory_id` is intentionally **not** a foreign key — operations on deleted memories must still be queryable.
- `operation` values are: `create`, `update`, `delete`, `pin`, `archive`.
- `actor` identifies who/what initiated the operation (e.g., `agent`, `user`, `system`, `skill:weather`).
- `details` stores a JSON diff for updates, or a reason string for deletes/archives.

---

## 4. Entity Relationship Diagram

```mermaid
erDiagram
    sessions ||--o{ messages : "has many"
    sessions ||--o{ approval_requests : "has many"
    sessions ||--o{ model_usage : "references"
    sessions ||--o{ repl_history : "has many"

    messages {
        TEXT id PK
        TEXT session_id FK
        TEXT role
        TEXT content
        TEXT tool_calls
        INTEGER token_count
        INTEGER created_at
    }

    sessions {
        TEXT id PK
        TEXT channel
        TEXT channel_id
        INTEGER created_at
        INTEGER updated_at
        TEXT metadata
        INTEGER archived_at
    }

    audit_log {
        INTEGER id PK
        INTEGER timestamp
        TEXT actor
        TEXT action
        TEXT target
        TEXT details
        TEXT session_id
        TEXT risk_level
        TEXT hmac_sig
    }

    secrets {
        TEXT id PK
        TEXT name UK
        BLOB ciphertext
        BLOB nonce
        INTEGER created_at
    }

    memories ||--o| memory_embeddings : "has one"
    memories ||--o{ memory_links : "source"
    memories ||--o{ memory_links : "target"
    memories ||--o{ memory_ops_log : "audited by"

    memories {
        TEXT id PK
        TEXT key
        TEXT content
        TEXT category
        TEXT tags
        TEXT source
        REAL importance
        INTEGER access_count
        INTEGER created_at
        INTEGER updated_at
        INTEGER expires_at
        INTEGER archived
        INTEGER pinned
    }

    memory_embeddings {
        TEXT memory_id PK_FK
        BLOB embedding
        TEXT model
        INTEGER dimensions
        INTEGER created_at
    }

    memory_links {
        INTEGER id PK
        TEXT source_id
        TEXT target_id
        TEXT relation
        TEXT metadata
        INTEGER created_at
    }

    memory_snapshots {
        TEXT id PK
        TEXT reason
        INTEGER created_at
        INTEGER memory_count
        INTEGER size_bytes
        TEXT snapshot_data
    }

    memory_ops_log {
        INTEGER id PK
        INTEGER timestamp
        TEXT memory_id
        TEXT operation
        TEXT actor
        TEXT source_session
        TEXT details
    }

    tool_policies {
        TEXT tool_name PK
        TEXT risk_class
        TEXT policy
        TEXT network_scope
        TEXT fs_scope
        INTEGER rate_limit
        INTEGER timeout_ms
        INTEGER updated_at
        INTEGER enabled
    }

    approval_requests {
        TEXT id PK
        TEXT session_id
        TEXT tool_name
        TEXT tool_params
        TEXT risk_level
        TEXT status
        TEXT scope
        INTEGER created_at
        INTEGER resolved_at
        TEXT resolved_by
    }

    cron_jobs {
        TEXT id PK
        TEXT name
        TEXT cron_expr
        TEXT prompt
        INTEGER enabled
        INTEGER next_run
        INTEGER last_run
        INTEGER run_count
        TEXT action_type
        INTEGER created_at
        INTEGER updated_at
        INTEGER one_shot
    }

    skills {
        TEXT id PK
        TEXT name UK
        TEXT description
        TEXT content
        INTEGER enabled
        TEXT skill_type
        INTEGER has_code
        TEXT source_url
        INTEGER installed_at
    }

    model_usage {
        INTEGER id PK
        INTEGER timestamp
        TEXT provider
        TEXT model
        TEXT session_id
        INTEGER input_tokens
        INTEGER output_tokens
        REAL cost_usd
        INTEGER latency_ms
        TEXT routing_mode
    }

    auth_tokens {
        TEXT id PK
        TEXT token_hash
        INTEGER last_used_at
        INTEGER revoked
        INTEGER created_at
        INTEGER expires_at
        INTEGER paired_at
    }

    mcp_servers {
        TEXT id PK
        TEXT name UK
        TEXT config
        INTEGER enabled
        INTEGER added_at
    }

    workspace_file_versions {
        TEXT id PK
        TEXT file_path
        TEXT content
        INTEGER version
        INTEGER created_at
        INTEGER size_bytes
    }

    agents ||--o| model_instances : "references"

    agents {
        TEXT id PK
        TEXT name UK
        TEXT description
        TEXT persona
        TEXT tools
        TEXT skills
        TEXT model
        TEXT provider
        TEXT model_instance_id FK
        INTEGER enabled
        INTEGER is_default
        INTEGER created_at
        INTEGER updated_at
    }

    model_instances {
        TEXT id PK
        TEXT name UK
        TEXT provider
        TEXT model
        INTEGER is_default
        INTEGER created_at
        INTEGER updated_at
    }

    repl_history {
        INTEGER id PK
        TEXT session_id
        TEXT language
        TEXT code
        TEXT output
        INTEGER success
        INTEGER created_at
    }
```

---

## 5. Repository Pattern

All data access goes through Rust traits defined in `crates/antec-storage/src/repository.rs`. The single concrete implementation is `SqliteRepository`, which holds an `Arc<Database>`.

```rust
pub struct SqliteRepository {
    db: Arc<Database>,
}

impl SqliteRepository {
    pub fn new(db: Arc<Database>) -> Self {
        Self { db }
    }
}
```

### 5.1 SessionRepo

```rust
pub trait SessionRepo {
    fn create_session(&self, session: &SessionRow) -> Result<()>;
    fn get_session(&self, id: &str) -> Result<Option<SessionRow>>;
    fn list_sessions(&self) -> Result<Vec<SessionRow>>;
    fn delete_session(&self, id: &str) -> Result<()>;
    fn update_session_timestamp(&self, id: &str, updated_at: i64) -> Result<()>;
    fn archive_session(&self, id: &str, archived_at: i64) -> Result<bool>;
    fn unarchive_session(&self, id: &str) -> Result<bool>;
    fn list_sessions_filtered(
        &self,
        channel: Option<&str>,
        archived: Option<bool>,
        from_ts: Option<i64>,
        to_ts: Option<i64>,
        limit: i64,
    ) -> Result<Vec<SessionRow>>;
    fn merge_sessions(&self, target_id: &str, source_id: &str) -> Result<u64>;
    fn get_session_messages_ordered(&self, session_id: &str) -> Result<Vec<MessageRow>>;
}
```

**Implementation notes:**

- `list_sessions` orders by `updated_at DESC`.
- `list_sessions_filtered` builds a dynamic WHERE clause with parameterized values.
- `merge_sessions` moves all messages from source to target (`UPDATE messages SET session_id = ?`), then deletes the source session. Returns the count of messages moved.
- `get_session_messages_ordered` orders by `created_at ASC` (chronological).

### 5.2 MessageRepo

```rust
pub trait MessageRepo {
    fn insert_message(&self, msg: &MessageRow) -> Result<()>;
    fn get_messages_for_session(&self, session_id: &str) -> Result<Vec<MessageRow>>;
    fn get_message(&self, id: &str) -> Result<Option<MessageRow>>;
    fn delete_messages_for_session(&self, session_id: &str) -> Result<()>;
}
```

**Implementation notes:**

- `get_messages_for_session` orders by `created_at ASC`.

### 5.3 AuditRepo

```rust
pub trait AuditRepo {
    fn log_audit(&self, entry: &AuditLogRow) -> Result<()>;
    fn get_last_audit_sig(&self) -> Result<String>;
    fn log_audit_chained(
        &self,
        entry: &mut AuditLogRow,
        sign_fn: &dyn Fn(&str) -> String,
    ) -> Result<()>;
    fn query_audit_log(
        &self,
        session_id: Option<&str>,
        limit: i64,
    ) -> Result<Vec<AuditLogRow>>;
    fn query_audit_log_filtered(
        &self,
        session_id: Option<&str>,
        actor: Option<&str>,
        action: Option<&str>,
        risk_level: Option<&str>,
        from_ts: Option<i64>,
        to_ts: Option<i64>,
        limit: i64,
    ) -> Result<Vec<AuditLogRow>>;
    fn export_audit_csv(&self, limit: i64) -> Result<String>;
    fn delete_audit_before(&self, before_ts: i64) -> Result<u64>;
    fn query_audit_chain(&self, limit: i64) -> Result<Vec<AuditLogRow>>;
}
```

**Implementation notes:**

- `get_last_audit_sig` queries `SELECT hmac_sig FROM audit_log ORDER BY id DESC LIMIT 1`. Returns empty string if no entries or if the last entry has no signature.
- `log_audit_chained` atomically: (1) fetches the previous signature, (2) calls `sign_fn(previous_sig)` to compute the new HMAC, (3) sets `entry.hmac_sig`, (4) inserts.
- `query_audit_log` orders by `timestamp DESC`.
- `query_audit_log_filtered` builds a dynamic WHERE clause from all optional filter parameters.
- `query_audit_chain` orders by `id ASC` (insertion order) for chain verification.
- `export_audit_csv` returns a CSV string with headers: `id,timestamp,actor,action,target,risk_level,session_id,details`.
- `delete_audit_before` deletes entries with `timestamp < before_ts`, returns count.

### 5.4 MemoryRepo

```rust
pub trait MemoryRepo {
    fn insert_memory(&self, mem: &MemoryRow) -> Result<()>;
    fn get_memory(&self, id: &str) -> Result<Option<MemoryRow>>;
    fn list_memories(&self, limit: i64) -> Result<Vec<MemoryRow>>;
    fn delete_memory(&self, id: &str) -> Result<bool>;
    fn search_memory_fts(&self, query: &str, limit: i64) -> Result<Vec<MemoryRow>>;
    fn update_memory(&self, id: &str, content: &str, tags: Option<&str>) -> Result<()>;
    fn update_memory_full(
        &self,
        id: &str,
        key: &str,
        content: &str,
        category: Option<&str>,
        tags: Option<&str>,
        importance: f64,
    ) -> Result<()>;
    fn list_memories_filtered(
        &self,
        category: Option<&str>,
        sort_by: &str,        // 'importance', 'created_at', 'updated_at', 'access_count'
        sort_order: &str,     // 'asc', 'desc'
        limit: i64,
    ) -> Result<Vec<MemoryRow>>;
    fn memory_stats(&self) -> Result<(i64, f64)>;  // (count, avg_importance)
    fn increment_access_count(&self, id: &str) -> Result<()>;
    fn get_memory_by_key(&self, key: &str) -> Result<Option<MemoryRow>>;
    fn upsert_memory(&self, mem: &MemoryRow) -> Result<()>;
    fn list_all_memories(&self) -> Result<Vec<MemoryRow>>;
    fn archive_memory(&self, id: &str) -> Result<()>;
    fn delete_archived_before(&self, before_ts: i64) -> Result<u64>;
    fn unarchive_memory(&self, id: &str, new_importance: f64) -> Result<bool>;
    fn list_archived_memories(&self, limit: i64) -> Result<Vec<MemoryRow>>;
    fn set_memory_pinned(&self, id: &str, pinned: bool) -> Result<()>;
    fn list_pinned_memories(&self) -> Result<Vec<MemoryRow>>;
    fn memory_stats_full(&self) -> Result<MemoryStatsFullRow>;
    fn update_memory_source(&self, id: &str, source: &str) -> Result<()>;
}
```

**Implementation notes:**

- `search_memory_fts` queries the `memory_fts` FTS5 virtual table and joins back to `memories` for full row data. Filters out archived memories (`WHERE m.archived = 0`).
- `upsert_memory` uses `INSERT OR REPLACE` semantics.
- `list_all_memories` returns all non-archived memories (used by the decay sweep).
- `archive_memory` sets `archived = 1` and `updated_at` to current timestamp.
- `unarchive_memory` sets `archived = 0`, resets `importance` to `new_importance`, and resets `access_count` to 0.
- `memory_stats_full` returns:

```rust
pub struct MemoryStatsFullRow {
    pub total: i64,
    pub avg_importance: f64,
    pub archived_count: i64,
    pub pinned_count: i64,
    pub pending_count: i64,        // source = 'pending'
    pub categories: HashMap<String, i64>,
    pub storage_bytes: i64,        // page_count * page_size estimate
}
```

### 5.5 MemoryEmbeddingRepo

```rust
pub trait MemoryEmbeddingRepo {
    fn upsert_embedding(&self, row: &MemoryEmbeddingRow) -> Result<()>;
    fn get_embedding(&self, memory_id: &str) -> Result<Option<MemoryEmbeddingRow>>;
    fn get_embeddings_batch(&self, memory_ids: &[String]) -> Result<Vec<MemoryEmbeddingRow>>;
    fn list_all_embeddings(&self) -> Result<Vec<MemoryEmbeddingRow>>;
    fn delete_embedding(&self, memory_id: &str) -> Result<bool>;
    fn count_embeddings(&self) -> Result<i64>;
}
```

### 5.6 MemoryLinkRepo

```rust
pub trait MemoryLinkRepo {
    fn create_memory_link(&self, link: &MemoryLinkRow) -> Result<()>;
    fn get_links_from(&self, source_id: &str) -> Result<Vec<MemoryLinkRow>>;
    fn get_links_to(&self, target_id: &str) -> Result<Vec<MemoryLinkRow>>;
    fn get_links_by_relation(
        &self, source_id: &str, relation: &str,
    ) -> Result<Vec<MemoryLinkRow>>;
    fn delete_memory_link(&self, id: i64) -> Result<bool>;
    fn delete_links_for_memory(&self, memory_id: &str) -> Result<u64>;
    fn traverse_links(&self, start_id: &str, max_depth: u32) -> Result<Vec<String>>;
}
```

**Implementation notes:**

- `create_memory_link` uses `INSERT OR IGNORE` to handle the UNIQUE constraint on `(source_id, target_id, relation)`.
- `delete_links_for_memory` deletes links where the memory appears as either source or target.
- `traverse_links` performs a BFS traversal:
  1. Initialize a `HashSet<String>` with `start_id` as visited, and a frontier queue.
  2. For each depth level (up to `max_depth`):
     - For each node in the current frontier, query outbound links (`source_id = ?`) and inbound links (`target_id = ?`).
     - Add unvisited neighbors to the next frontier and mark as visited.
  3. Return all visited IDs (excluding the start node).

### 5.7 ToolPolicyRepo

```rust
pub trait ToolPolicyRepo {
    fn get_policy(&self, tool_name: &str) -> Result<Option<ToolPolicyRow>>;
    fn set_policy(&self, tool_name: &str, policy: &str) -> Result<()>;
    fn is_tool_enabled(&self, tool_name: &str) -> Result<bool>;
    fn set_tool_enabled(&self, tool_name: &str, enabled: bool) -> Result<()>;
    fn list_disabled_tools(&self) -> Result<Vec<String>>;
    fn get_rate_limits(&self) -> Result<HashMap<String, u32>>;
    fn set_rate_limit(&self, tool_name: &str, rate_limit: Option<i32>) -> Result<()>;
}
```

**Implementation notes:**

- `is_tool_enabled` returns `true` if no policy row exists (tools are enabled by default).
- `set_tool_enabled` creates a policy row with default values if one does not exist.
- `get_rate_limits` returns only tools with non-NULL `rate_limit` values.

### 5.8 ModelUsageRepo

```rust
pub trait ModelUsageRepo {
    fn record_usage(&self, usage: &ModelUsageRow) -> Result<()>;
    fn usage_by_provider(&self, since_ts: i64) -> Result<Vec<UsageByProvider>>;
    fn usage_by_model(&self, since_ts: i64) -> Result<Vec<UsageByModel>>;
    fn usage_by_day(&self, since_ts: i64) -> Result<Vec<UsageByDay>>;
    fn total_cost(&self, since_ts: i64) -> Result<TotalCost>;
    fn session_count(&self, since_ts: i64) -> Result<i64>;
    fn messages_by_channel(&self, since_ts: i64) -> Result<Vec<MessagesByChannel>>;
    fn tool_usage_top(&self, since_ts: i64, limit: i64) -> Result<Vec<ToolUsageCount>>;
    fn routing_savings(&self, since_ts: i64) -> Result<RoutingSavingsRow>;
}
```

**Aggregate result types:**

```rust
pub struct UsageByProvider {
    pub provider: String, pub total_input: i64, pub total_output: i64,
    pub cost_usd: f64, pub call_count: i64,
}
pub struct UsageByModel {
    pub model: String, pub total_input: i64, pub total_output: i64,
    pub cost_usd: f64, pub call_count: i64,
}
pub struct UsageByDay {
    pub date: String, pub total_input: i64, pub total_output: i64,
    pub cost_usd: f64, pub call_count: i64,
}
pub struct TotalCost {
    pub total_cost_usd: f64, pub total_input: i64, pub total_output: i64, pub total_calls: i64,
}
pub struct MessagesByChannel { pub channel: String, pub message_count: i64 }
pub struct ToolUsageCount { pub tool_name: String, pub call_count: i64 }
pub struct RoutingSavingsRow {
    pub simple_count: i64, pub complex_count: i64,
    pub simple_cost_usd: f64, pub complex_cost_usd: f64,
}
```

**Implementation notes:**

- `messages_by_channel` joins messages to sessions on `session_id` and groups by `sessions.channel`.
- `tool_usage_top` queries the audit_log for `action = 'tool_call'`, groups by `target` (tool name), orders by count DESC.
- `routing_savings` groups model_usage by `routing_mode` into 'simple' vs 'complex' buckets.

### 5.9 ApprovalRepo

```rust
pub trait ApprovalRepo {
    fn create_request(&self, req: &ApprovalRequestRow) -> Result<()>;
    fn get_pending_requests(&self, session_id: &str) -> Result<Vec<ApprovalRequestRow>>;
    fn resolve_request(&self, id: &str, status: &str, resolved_by: &str) -> Result<()>;
    fn list_approval_requests(
        &self, session_id: Option<&str>, limit: i64,
    ) -> Result<Vec<ApprovalRequestRow>>;
}
```

### 5.10 SecretRepo

```rust
pub trait SecretRepo {
    fn store_secret(&self, secret: &SecretRow) -> Result<()>;
    fn get_secret(&self, name: &str) -> Result<Option<SecretRow>>;
    fn list_secrets(&self) -> Result<Vec<String>>;   // returns names only
    fn delete_secret(&self, name: &str) -> Result<bool>;
}
```

### 5.11 CronJobRepo

```rust
pub trait CronJobRepo {
    fn create_cron_job(&self, job: &CronJobRow) -> Result<()>;
    fn get_cron_job(&self, id: &str) -> Result<Option<CronJobRow>>;
    fn list_cron_jobs(&self) -> Result<Vec<CronJobRow>>;
    fn update_cron_job(&self, job: &CronJobRow) -> Result<()>;
    fn delete_cron_job(&self, id: &str) -> Result<bool>;
    fn get_due_jobs(&self, now: i64) -> Result<Vec<CronJobRow>>;
    fn mark_job_run(&self, id: &str, last_run: i64, next_run: Option<i64>) -> Result<()>;
}
```

**Implementation notes:**

- `get_due_jobs` selects jobs where `enabled = 1 AND next_run IS NOT NULL AND next_run <= ?`.
- `mark_job_run` updates `last_run`, `next_run`, increments `run_count`, and sets `updated_at`. For `one_shot = 1` jobs, also sets `enabled = 0`.

### 5.12 SkillRepo

```rust
pub trait SkillRepo {
    fn install_skill(&self, skill: &SkillRow) -> Result<()>;
    fn get_skill(&self, id: &str) -> Result<Option<SkillRow>>;
    fn list_skills(&self) -> Result<Vec<SkillRow>>;
    fn set_skill_enabled(&self, id: &str, enabled: bool) -> Result<()>;
    fn delete_skill(&self, id: &str) -> Result<bool>;
    fn update_skill_content(&self, id: &str, content: &str) -> Result<bool>;
}
```

### 5.13 AuthTokenRepo

```rust
pub trait AuthTokenRepo {
    fn store_token(&self, id: &str, token_hash: &str, expires_at: i64, paired_at: i64) -> Result<()>;
    fn verify_token(&self, token_hash: &str) -> Result<Option<AuthTokenRow>>;
    fn revoke_token(&self, id: &str) -> Result<bool>;
    fn touch_last_used(&self, id: &str) -> Result<()>;
    fn cleanup_expired(&self, now: i64) -> Result<u64>;
}
```

### 5.14 McpServerRepo

```rust
pub trait McpServerRepo {
    fn create_mcp_server(&self, server: &McpServerRow) -> Result<()>;
    fn get_mcp_server(&self, name: &str) -> Result<Option<McpServerRow>>;
    fn list_mcp_servers(&self) -> Result<Vec<McpServerRow>>;
    fn update_mcp_server(&self, name: &str, config: &str, enabled: bool) -> Result<bool>;
    fn set_mcp_server_enabled(&self, name: &str, enabled: bool) -> Result<bool>;
    fn delete_mcp_server(&self, name: &str) -> Result<bool>;
}
```

### 5.15 MemorySnapshotRepo

```rust
pub trait MemorySnapshotRepo {
    fn create_snapshot(&self, snapshot: &MemorySnapshotRow) -> Result<()>;
    fn get_snapshot(&self, id: &str) -> Result<Option<MemorySnapshotRow>>;
    fn list_snapshots(&self, limit: i64) -> Result<Vec<MemorySnapshotRow>>;
    fn delete_snapshot(&self, id: &str) -> Result<bool>;
}
```

### 5.16 WorkspaceFileRepo

```rust
pub trait WorkspaceFileRepo {
    fn save_file_version(&self, row: &FileVersionRow) -> Result<()>;
    fn get_latest_file_version(&self, file_path: &str) -> Result<Option<FileVersionRow>>;
    fn get_file_version(&self, file_path: &str, version: i64) -> Result<Option<FileVersionRow>>;
    fn list_file_versions(&self, file_path: &str, limit: i64) -> Result<Vec<FileVersionRow>>;
    fn revert_file_to_version(&self, file_path: &str, version: i64) -> Result<FileVersionRow>;
}
```

**Implementation notes:**

- `revert_file_to_version` reads the content at the specified version, then creates a new version entry with version = max(current) + 1, returning the newly created row.

### 5.17 ReplHistoryRepo

```rust
pub trait ReplHistoryRepo {
    fn insert_repl_entry(&self, entry: &ReplHistoryRow) -> Result<()>;
    fn get_repl_history(&self, session_id: &str, limit: i64) -> Result<Vec<ReplHistoryRow>>;
    fn get_all_repl_history(&self, limit: i64) -> Result<Vec<ReplHistoryRow>>;
}
```

### 5.18 AgentRepo

```rust
pub trait AgentRepo {
    fn create_agent(&self, agent: &AgentDefinitionRow) -> Result<()>;
    fn get_agent(&self, id: &str) -> Result<Option<AgentDefinitionRow>>;
    fn get_agent_by_name(&self, name: &str) -> Result<Option<AgentDefinitionRow>>;
    fn get_default_agent(&self) -> Result<Option<AgentDefinitionRow>>;
    fn list_agents(&self) -> Result<Vec<AgentDefinitionRow>>;
    fn update_agent(&self, agent: &AgentDefinitionRow) -> Result<bool>;
    fn delete_agent(&self, id: &str) -> Result<bool>;
    fn set_agent_enabled(&self, id: &str, enabled: bool) -> Result<bool>;
}
```

### 5.19 ModelInstanceRepo

```rust
pub trait ModelInstanceRepo {
    fn create_model_instance(&self, instance: &ModelInstanceRow) -> Result<()>;
    fn get_model_instance(&self, id: &str) -> Result<Option<ModelInstanceRow>>;
    fn get_model_instance_by_name(&self, name: &str) -> Result<Option<ModelInstanceRow>>;
    fn get_default_model_instance(&self) -> Result<Option<ModelInstanceRow>>;
    fn list_model_instances(&self) -> Result<Vec<ModelInstanceRow>>;
    fn update_model_instance(&self, instance: &ModelInstanceRow) -> Result<bool>;
    fn delete_model_instance(&self, id: &str) -> Result<bool>;
    fn set_model_instance_default(&self, id: &str) -> Result<bool>;
}
```

### 5.20 DiagnosticsRepo

```rust
pub trait DiagnosticsRepo {
    fn db_integrity_check(&self) -> Result<String>;
}
```

Runs `PRAGMA integrity_check` and returns `"ok"` or a diagnostic error string.

### 5.21 DataDeletionRepo

```rust
pub trait DataDeletionRepo {
    fn delete_session_data(&self, session_id: &str) -> Result<()>;
    fn delete_all_user_data(&self) -> Result<()>;
    fn list_data_paths(&self) -> Vec<String>;
    fn purge_old_messages(&self, before_ts: i64) -> Result<u64>;
    fn purge_old_memories(&self, before_ts: i64) -> Result<u64>;
    fn export_all_data(&self) -> Result<ExportBundle>;
    fn storage_info(&self) -> Result<StorageInfo>;
}
```

**Implementation notes:**

- `delete_session_data` deletes messages and the session row in a transaction.
- `delete_all_user_data` truncates all user-facing tables (sessions, messages, memories, cron_jobs, approval_requests, secrets).
- `purge_old_messages` deletes messages with `created_at < before_ts`, returns count.
- `purge_old_memories` deletes non-archived memories with `updated_at < before_ts`, returns count.
- `export_all_data` returns an `ExportBundle` (GDPR data portability):

```rust
pub struct ExportBundle {
    pub sessions: Vec<SessionRow>,
    pub messages: Vec<MessageRow>,
    pub memories: Vec<MemoryRow>,
    pub audit_log: Vec<AuditLogRow>,
    pub cron_jobs: Vec<CronJobRow>,
    pub exported_at: i64,
}
```

- `storage_info` returns row counts:

```rust
pub struct StorageInfo {
    pub sessions: i64,
    pub messages: i64,
    pub memories: i64,
    pub audit_entries: i64,
    pub cron_jobs: i64,
    pub skills: i64,
}
```

**Additional data model:**

```rust
pub struct StoragePaths {
    pub data_dir: String,
    pub db_path: String,
    pub workspace_dir: String,
    pub skills_dir: String,
    pub config_path: String,
}
```

### 5.22 MemoryOpsLogRepo

```rust
pub trait MemoryOpsLogRepo {
    fn log_operation(&self, entry: &MemoryOpsLogRow) -> Result<()>;
    fn get_ops_for_memory(&self, memory_id: &str, limit: usize) -> Result<Vec<MemoryOpsLogRow>>;
    fn get_recent_ops(&self, limit: usize) -> Result<Vec<MemoryOpsLogRow>>;
}
```

**Implementation notes:**

- `log_operation` inserts a new row into `memory_ops_log`. Called by `MemoryRepo` methods internally (create, update, delete, pin, archive) to maintain a complete audit trail.
- `get_ops_for_memory` returns operations for a specific memory, ordered by `timestamp DESC`, limited to `limit` rows.
- `get_recent_ops` returns the most recent operations across all memories, ordered by `timestamp DESC`, limited to `limit` rows.

---

## 6. Error Types

```rust
#[derive(Debug, Error)]
pub enum StorageError {
    #[error("database error: {0}")]
    Database(String),

    #[error("connection pool error: {0}")]
    Pool(String),

    #[error("migration error: {0}")]
    Migration(String),

    #[error("path violation: {0}")]
    PathViolation(String),

    #[error("query error: {0}")]
    Query(String),

    #[error("not found: {0}")]
    NotFound(String),

    #[error("I/O error: {0}")]
    Io(#[from] std::io::Error),
}

pub type Result<T> = std::result::Result<T, StorageError>;
```

Automatic conversions from:

- `rusqlite::Error` to `StorageError::Database`
- `r2d2::Error` to `StorageError::Pool`
- `std::io::Error` to `StorageError::Io`

---

## 7. Workspace Manager

The `Workspace` struct manages a sandboxed directory for file operations, preventing path-traversal attacks.

### 7.1 API

```rust
pub struct Workspace {
    root: PathBuf,  // canonicalized absolute path
}

impl Workspace {
    pub fn new(root: PathBuf) -> Result<Self>;
    pub fn root(&self) -> &Path;
    pub fn resolve(&self, relative: &str) -> Result<PathBuf>;
    pub fn validate_path(&self, path: &Path) -> Result<()>;
}
```

### 7.2 Security Properties

| Property | Mechanism |
|----------|-----------|
| Jail enforcement | All resolved paths must start with the canonicalized workspace root |
| Path traversal prevention | `../` sequences are resolved via canonicalization, then checked against root prefix |
| Empty path rejection | `resolve("")` returns `StorageError::PathViolation` |
| Directory auto-creation | `new()` creates the root directory if it does not exist |
| Canonical root | Root path is canonicalized at construction time (resolves symlinks) |

### 7.3 Resolution Algorithm

1. Reject empty relative paths immediately.
2. Join the relative path onto the workspace root.
3. If the target path exists, canonicalize it directly.
4. If the target does not exist, walk up to find the nearest existing ancestor, canonicalize that ancestor, then append the remaining path components.
5. Validate that the resolved path starts with the workspace root.
6. Return the resolved absolute path.

### 7.4 File Versioning

File operations are tracked via the `workspace_file_versions` table:

- **Write**: Each write creates a new version entry with `version = max(current) + 1`.
- **Read**: Always reads the latest version.
- **History**: List all versions of a file, ordered newest first.
- **Diff**: Compare two versions by reading their content.
- **Revert**: Creates a new version containing the content of a previous version.

### 7.5 File Tree

The workspace provides a recursive directory listing endpoint that returns a tree structure for the web console file browser. The tree is built by walking the workspace directory, respecting the jail boundary.

---

## 8. Crate Exports

```rust
// crates/antec-storage/src/lib.rs
pub mod error;
pub mod db;
pub mod models;
pub mod repository;
pub mod workspace;

pub use db::Database;
pub use error::{Result, StorageError};
pub use repository::{AgentRepo, MemoryLinkRepo, MemoryOpsLogRepo, ReplHistoryRepo, WorkspaceFileRepo};
pub use workspace::Workspace;
```

---

## 9. Testing Strategy

| Test Category | Approach |
|---------------|----------|
| Migration idempotency | Open in-memory DB, verify `user_version = 22`, re-run migrations, verify no error |
| Table existence | After migration, query `sqlite_master` for all expected table names |
| WAL mode | Verify `PRAGMA journal_mode` returns `"wal"` (or `"memory"` for in-memory) |
| Foreign keys | Verify `PRAGMA foreign_keys` returns `1` |
| Repository CRUD | Use `Database::open_memory()` for isolated tests with real SQL execution |
| Workspace jail | Test that `resolve("../../../etc/passwd")` returns `PathViolation` |
| Path canonicalization | Verify symlink resolution on macOS (`/tmp` to `/private/tmp`) |
