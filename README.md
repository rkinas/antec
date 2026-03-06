# Antec Features

This is a short comparison-friendly summary of what the current system already implements.

| Area | Implemented feature | Short description |
| --- | --- | --- |
| Core runtime | Self-hosted assistant server | Single Rust server process with WebSocket, REST, SSE, and embedded web console. |
| Deployment | Local-first persistence | SQLite with WAL, embedded migrations, pooled access, and local workspace storage. |
| Access | OTP pairing | Startup prints a 6-digit OTP; WebSocket clients can pair and receive persisted auth tokens. |
| Chat | Streaming assistant runtime | Supports synchronous REST chat, WebSocket chat, and SSE streaming of response/tool events. |
| Models | Multi-provider support | Anthropic, OpenAI, Ollama, and Google providers are implemented. |
| Routing | Automatic model selection | Heuristic routing picks cheaper/simple vs complex models and exposes routing-savings stats. |
| Reliability | Provider failover | Ordered fallback providers with retry rules, circuit breakers, and provider-switch events. |
| Tools | Native tool platform | 43 built-in native tools plus 4 messaging tools, with dynamic skill and MCP tool registration. |
| Files | Workspace file operations | Read, write, patch, search, list, stat, move, delete, diff, version, and revert files inside a jailed workspace. |
| Shell | Sandboxed command execution | Foreground and background shell execution with process tracking, blocklists, and sandbox limits. |
| Web | Safe web retrieval | Web fetch/search/extract tools with SSRF protections, DNS pinning, and body-size limits. |
| Memory | Persistent long-term memory | FTS-backed storage with categories, tags, importance, pin/archive, snapshots, and hybrid recall. |
| Memory | Auto memory extraction | User messages can be mined for facts, preferences, and decisions, then approved or rejected later. |
| Scheduler | Cron and reminders | Recurring cron jobs, natural-language reminders, one-shot jobs, and heartbeat scheduling. |
| Agents | Multi-agent architecture | Named agents with persona, tool/skill/model isolation, mention routing, and bundled agent definitions. |
| Parallelism | Parallel execution API | Parallel execution tracking and node/sub-agent APIs exist; default runtime currently simulates worker outputs. |
| Channels | Multi-channel messaging | Console, iMessage, Discord, and WhatsApp channel abstractions with send/status/test/config surfaces. |
| Skills | Skill packaging system | 66 bundled skills plus local/hub skills, manifests, scaffolding, enable/disable, import, build, and test flows. |
| MCP | External tool federation | Persisted MCP server configs, discovery, reconnect, and dynamic MCP tools bridged into the tool registry. |
| Extensions | Integration templates | 25+ bundled extension templates with enable/disable/install flows and credential support. |
| Hands | Higher-level packaged capabilities | 7 bundled hands with activation, pause/resume, categories, and associated agent metadata. |
| Security | Secret vault and redaction | Encrypted secrets, runtime secret scanning, output redaction, and provider credential lookup from vault or env. |
| Security | Prompt and tool defenses | Injection detection, chain detection, loop detection, per-tool rate limits, and circuit breakers. |
| Security | Execution isolation | OS sandboxing, command blocklists, channel allowlists, and dangerous-tool approval flows in WebSocket mode. |
| Workspace | Built-in development tools | REPL, file versioning, session export/merge, and code-evaluation support. |
| Observability | Admin and diagnostics APIs | Health, readiness, diagnostics, metrics, tool metrics, retrieval metrics, audit export, and stats endpoints. |
| Governance | Data management | Session deletion, export endpoints, storage-path introspection, and retention/purge jobs. |
| Console | Embedded SPA | Console assets are compiled into the binary and served with SPA fallback routing. |

## Product Comparison Checklist

- AI runtime: multi-provider, routing, failover, tool-use, streaming
- Memory: persistent, searchable, auto-extracted, decayed, snapshot-capable
- Automation: cron, reminders, heartbeat, proactive outbound delivery
- Extensibility: skills, MCP, extensions, hands
- Channels: console plus messaging adapters
- Security: vault, redaction, injection detection, approvals, sandboxing
- Ops: metrics, diagnostics, audit, config APIs, storage/export tools

## Important Current-State Caveats

- REST and SSE are exposed without the same auth gating used by WebSocket.
- Discord and WhatsApp surfaces exist, but their end-to-end runtime wiring should be treated as partial until re-verified.
- Embedding support exists, but it does not appear enabled in the default startup path.
- Real sub-agent execution exists in library code, but the default server path uses simulated node/parallel outputs.
- Scheduler action types such as `ask_agent` and `run_skill` are stored distinctly, but current execution forwards prompt content through the proactive sender.
