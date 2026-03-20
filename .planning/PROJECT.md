# GopherMind

## What This Is

A spec-driven Go development platform that orchestrates multiple LLM providers to handle the full software project lifecycle — planning, code generation, review, debugging, testing, and verification. GopherMind converts ideas into durable execution systems with persistent planning artifacts, phased delivery, and strong verification gates. It is built in Go, for Go developers, with deep understanding of Go idioms, tooling, and ecosystem conventions.

## Core Value

Every development decision is explicit, every step is verifiable, and every session is resumable — no hidden assumptions, no lost context, no wasted work.

## Requirements

### Validated

(None yet — ship to validate)

### Active

- [ ] 6-layer architecture: CLI, Orchestration, Agent Runtime, LLM Provider, Go Intelligence, State/Persistence
- [ ] CLI interface with commands: init, status, resume, view, verify, logs, costs, config
- [ ] Multi-provider LLM support (Anthropic, OpenAI, Google, Ollama) with unified Provider interface
- [ ] Rule-based task routing — different models for different jobs (architecture vs summarization vs code gen)
- [ ] Fallback chains with exponential backoff and provider failover
- [ ] Cost tracking and token accounting per provider, agent, and tier
- [ ] 12+ specialized agents with explicit contracts, success criteria, and handoff rules
- [ ] Agent execution protocol: load definition → assemble context → select provider → execute → validate → write → handoff
- [ ] Parallel agent execution with dependency graph coordination
- [ ] File-based state persistence in .planning/ directory for full resumability
- [ ] STATE.md maintenance with current phase, tasks, decisions, risks, metrics
- [ ] Structured event logging (JSONL) for events, provider calls, and costs
- [ ] Wave-based interactive questioning protocol (8 waves) for project initialization
- [ ] Go repository intelligence: module detection, package graphs, convention checking
- [ ] Go conventions enforcement: package layout, naming, error handling, context usage, structured logging
- [ ] Verification gates: coverage checks, consistency checks, completeness checks, alignment checks
- [ ] Configuration via config.json with sensible defaults
- [ ] Checkpoint and resume support after interruptions
- [ ] Atomic file writes (temp file → rename) for artifact safety
- [ ] Provider health checks and availability monitoring

### Out of Scope

- Web UI — CLI-first; web interface deferred to post-v1
- Cloud sync — local-only storage for v1
- Non-Go repository support — Go-optimized only in v1
- Real-time collaboration — single-user tool for v1
- Plugin/extension API — internal architecture only for v1

## Context

GopherMind is inspired by the spec-first, agent-driven development workflow. The current landscape of AI coding tools (Copilot, Cursor, Aider) focuses on code completion and chat. GopherMind takes a fundamentally different approach: it manages the entire project lifecycle through specialized agents with explicit contracts, not a single general-purpose assistant.

The system is designed for a developer who wants deterministic, auditable, resumable development — someone who has been burned by vague AI outputs, lost context across sessions, and tools that don't understand project structure.

Key technical context:
- Go 1.23+ with modules
- Standard library first, third-party only when justified
- Structured logging via slog or zerolog
- SQLite or file-based persistence for local state
- Provider APIs: Anthropic Messages API, OpenAI Chat Completions, Google Gemini, Ollama local

## Constraints

- **Language**: Go 1.23+ — the system is built in Go and optimized for Go projects
- **Platform**: Cross-platform (macOS, Linux, Windows) — Go makes this natural
- **Privacy**: Source code stays local by default; only sends code to providers when explicitly configured
- **Dependencies**: Minimal third-party; stdlib-first bias
- **Persistence**: File-based (.planning/) — no external databases required
- **LLM Providers**: Must work with at least Anthropic + OpenAI in v1; Google and Ollama are stretch goals
- **Timeline**: Ship v1 tonight — full system, no compromises

## Key Decisions

| Decision | Rationale | Outcome |
|----------|-----------|---------|
| Go as implementation language | Target audience is Go devs, Go is cross-platform, strong stdlib | — Pending |
| File-based persistence over database | Simpler, portable, git-trackable, no external deps | — Pending |
| CLI-first over web UI | Fastest path to usable tool, matches developer workflow | — Pending |
| Multi-provider from day one | Avoid provider lock-in, enable cost optimization | — Pending |
| Rule-based routing over dynamic | Predictable, auditable, simple to debug | — Pending |
| Agent contracts with success criteria | Prevents vague outputs, enables automated verification | — Pending |
| Structured JSONL logging | Parseable, appendable, tooling-friendly | — Pending |

---
*Last updated: 2026-03-20 after initialization*
