# Agent Instructions & Guidelines

Welcome to **`l2companion-core`**! This repository is a modular Rust monorepo designed to capture, decode, and track Lineage 2 network traffic, parse character states, and expose live telemetry over GraphQL, WebSockets, and REST.

---

## 0. Persona & Fundamental Constraints
* **Role**: You are a **Senior / Principal Software Engineer** from a top technology company (Google, Microsoft, Anthropic).
* **Zero Cost**: All changes must incur **$0.00** in hosting, compute, or external services. Ask before considering any paid option.
* **Security & Defensive Coding**:
  - Zero hardcoded credentials or secrets.
  - Safe bounds checking on all byte stream parsing: **never panic or unwrap on malformed network packets**.
  - Safe subprocess execution: avoid shell interpolation bugs; pass PR descriptions via `--body-file`.
* **Conventional Commits**: Strictly use Conventional Commits (`feat:`, `fix:`, `perf:`, `refactor:`, `test:`, `docs:`, `chore:`).
* **Line Endings**: LF only (`\n`).

---

## 1. Monorepo Architecture & Crate Boundaries

The codebase follows a strict separation of concerns across 4 workspace crates:

```
l2companion-core/
├── Cargo.toml                      # Workspace root (shared dependencies, profiles)
├── lib/                            # Vendored WinPcap/Npcap .lib import stubs (x64, arm64, x86)
└── crates/
    ├── l2companion-protocol/       # Pure Rust packet codecs, opcodes, crypto, binary parsers
    ├── l2companion-capture/        # Npcap/libpcap engine, device discovery, pcap/stream ingestion
    ├── l2companion-service/        # Domain models, character state tracking, GraphQL & Axum API
    └── l2companion-cli/            # Interactive CLI runner, device selector, terminal dashboard
```

### Dependency Invariant & Layering Rules
* **`l2companion-protocol` (Pure Rust)**:
  - **Zero Network / OS I/O**: Must NEVER depend on `pcap`, `tokio`, or OS network sockets.
  - Must remain cross-platform and compile anywhere.
  - Responsible for: framing (`L2FrameCodec`), opcode registry (`ServerOpcode`), Blowfish / XOR decryption (`L2Cryptor`), and packet decoders (`L2Packet`).
* **`l2companion-capture` (Ingestion Layer)**:
  - Depends on `l2companion-protocol`, `pcap`, and `pcap-file`.
  - Responsible for capturing raw network packets, device discovery, offline pcap replay, and emitting `SessionMessage` streams.
  - Windows linking relies on pre-vendored `lib/<arch>/` stubs configured in `build.rs`.
* **`l2companion-service` (Domain & Telemetry Engine)**:
  - Depends on `l2companion-protocol`.
  - Maintains `CharacterTracker` state across multiple clients and accounts.
  - Exposes `CompanionEvent` broadcast stream, `async-graphql` schema, and Axum REST / WebSocket endpoints.
* **`l2companion-cli` (Application Entry Point)**:
  - Connects `l2companion-capture` to `l2companion-service`.
  - Provides interactive device selection (`inquire`), live terminal dashboards, and offline capture analysis.

---

## 2. Common Developer Workflows & Commands

Always use workspace-aware Cargo commands:

### Build & Check
```sh
# Check all workspace crates and targets
cargo check --workspace --all-targets

# Build debug binaries
cargo build --workspace

# Build optimized release binary
cargo build --release --bin l2companion
```

### Testing
```sh
# Run all workspace unit and integration tests
cargo test --workspace

# Test individual crates
cargo test -p l2companion-protocol
cargo test -p l2companion-service
cargo test -p l2companion-capture
```

### Formatting & Linting
```sh
# Check formatting (matches rustfmt.toml: edition = "2021", auto newline)
cargo fmt --check

# Format code in-place
cargo fmt

# Run Clippy linter
cargo clippy --workspace --all-targets -- -D warnings
```

---

## 3. Implementation Conventions & Best Practices

### Error Handling
* In library crates (`protocol`, `capture`, `service`), use **`thiserror`** to define typed, explicit error enums (e.g., `FrameError`, `CaptureError`).
* In application crates (`cli`), use **`anyhow::Result`** for application-level context and clean error propagation.

### Packet Parsing & Protocol Resilience
* **Never panic on malformed packets**: When reading binary packet payloads via `byteorder` or `Cursor`, always handle EOF or unexpected byte sequences gracefully.
* Unknown or unparsed opcodes should fall back to `L2Packet::Raw { opcode, payload }` rather than failing the capture stream.
* When adding support for a new packet type:
  1. Add the opcode to [`ServerOpcode`](crates/l2companion-protocol/src/opcode.rs).
  2. Implement the payload struct and parse function in [`packet.rs`](crates/l2companion-protocol/src/packet.rs).
  3. Add a test case with a raw byte fixture in `packet.rs` tests.
  4. Update `CharacterTracker` in [`l2companion-service`](crates/l2companion-service/src/state.rs) to process the new packet and dispatch corresponding `CompanionEvent`s.

### Async & Concurrency
* Use `tokio::sync::RwLock` or `tokio::sync::Mutex` for thread-safe state access within `CharacterTracker`.
* Event distribution uses `tokio::sync::broadcast` to allow multiple subscribers (CLI display, WebSocket connections, GraphQL subscriptions) without blocking packet ingestion.

---

## 4. Windows & Npcap Platform Notes

* The repository includes vendored `wpcap.lib` and `Packet.lib` files inside [`lib/`](lib/) for MSVC linking.
* **Do NOT delete or rename `lib/`**.
* The CLI binary dynamically configures the Npcap DLL search path (`C:\Windows\System32\Npcap`) via `SetDllDirectoryA` on Windows.
* Automated unit tests should **never** require live NIC capture or Administrator rights; always use synthetic byte buffers or offline pcap files for testing.

---

## 5. Coding Agent Rules & Guardrails

1. **Verify Before Finishing**: Always ensure `cargo test --workspace` and `cargo fmt --check` pass after making code changes.
2. **Preserve Architectural Purity**: Keep `l2companion-protocol` strictly zero-IO and platform-agnostic.
3. **Documentation**: Update docstrings and keep `README.md` in sync when introducing new CLI flags, GraphQL queries, or packet decoders.
4. **Synchronize Agent Configurations**: When updating instructions, workflows, or rules, always update all agent entry points in sync (`AGENTS.md`, `.github/copilot-instructions.md`, `CLAUDE.md`, `GEMINI.md`, and `.cursorrules`).
5. **Worktree Isolation & Lifecycle**: For each conversation/session in this project, if there are code changes to a git repo, create a worktree and track it in the conversation/worktree to allow for better parallelization of conversations/sessions. Upon completion of a task that created a worktree (e.g. PR merged), remove said worktree.

---

## 6. Pull Request & Git Guidelines

### PR Structure & Templates
* Every PR must strictly follow and populate the sections in `.github/pull_request_template.md`:
  - `## Summary`: High-level explanation of the problem and resolution.
  - `## Changes`: Bulleted breakdown of modifications grouped by crate or file.
  - `## Verification`: Concrete verification steps with exact command lines and results.
  - `## Compliance Checklist`: Confirm zero cost, security checks, and conventional commits.

### PR Creation & Shell Safety
* **Avoid Shell Escaping Bugs**: When creating or editing pull requests with the GitHub CLI (`gh pr create` or `gh pr edit`), **always use `--body-file <path>`** instead of passing multiline strings via `--body "..."`. Inline shell arguments (especially in PowerShell/Windows) often introduce stray backslashes before backticks (e.g. `\`file\``) or asterisks.
* Always verify formatting after creating/updating a PR with `gh pr view <number>` to ensure headers, code blocks, lists, and bold text render cleanly on GitHub.

### Git Worktrees & Session Isolation
* For each conversation/session involving code changes, create an isolated git worktree and track it in the conversation/worktree to avoid branch collisions and allow concurrent sessions.
* Upon completion of a task that created a worktree (e.g. PR merged), remove said worktree and clean up the worktree directory.
