# Project Research Summary

**Project:** GopherMind
**Domain:** Go CLI multi-agent LLM orchestration platform (spec-driven development tool)
**Researched:** 2026-03-20
**Confidence:** HIGH

## Executive Summary

GopherMind is a spec-driven, multi-agent LLM orchestration CLI targeting Go developers. The competitive landscape — Aider, Cursor, Kiro, OpenHands — reveals a clear gap: no existing tool combines CLI-first operation, structured spec lifecycle management, specialized agent contracts with explicit success criteria, and Go-specific code intelligence in a single product. The recommended implementation approach is a layered Go CLI built around a narrow Provider interface, a DAG-based agent scheduler, and file-based state persistence in `.planning/`. Every layer is explicit, auditable, and testable — this is not a "chat with your codebase" tool but a deterministic, verification-gated pipeline that produces durable, reviewable artifacts.

The recommended stack is tight: Go 1.23+, Cobra + Viper for CLI structure, official provider SDKs (anthropic-sdk-go v1.27.1, openai-go v3.29.0, google.golang.org/genai, ollama/api), errgroup + semaphore for parallel agent coordination, and renameio for atomic file writes. No external database. No heavyweight framework. A single cross-platform binary with file-based state. The architecture follows a strict component build order — config, state, and provider layers must be solid before any agent logic is written.

The dominant risk class is not technical but behavioral: multi-agent systems fail in production at 41–86.7% rates, primarily from underspecified agent contracts and coordination failures, not from infrastructure problems. The second major risk is cost explosion from unguarded retry loops across 12+ agents. Both risks are addressed by building cost instrumentation and agent contract validation into the core execution protocol from day one — not as observability afterthoughts. Prompt drift, context window exhaustion, and spec-to-code drift round out the critical pitfall set and are best addressed by pinning model versions, budgeting context per agent call, and building semantic alignment checks into verification gates.

## Key Findings

### Recommended Stack

The stack is narrow by design and all core libraries are verified as of March 2026. Go 1.23 is the effective minimum (driven by Viper v1.21 and backoff v5). The official SDKs for all four providers are confirmed stable: Anthropic SDK reached v1+ in early 2026, openai-go v3 is the current official replacement for the community fork, google.golang.org/genai is GA and replaces the deprecated generative-ai-go (deprecated Nov 30, 2025 — do not use). For concurrency, golang.org/x/sync/errgroup with a weighted semaphore is the correct tool for agent fanout. No DI framework, no gRPC, no database — manual constructor injection and file I/O are sufficient and keep the binary cross-compilable.

**Core technologies:**
- Go 1.23+: Single cross-platform binary, explicit error handling, strong stdlib concurrency — matches target audience
- github.com/spf13/cobra v1.10.2: CLI command structure — de facto standard (Kubernetes, GitHub CLI), shell completions, help generation
- github.com/spf13/viper v1.21.0: Config layering (flags > env > config.json > defaults) — natural Cobra companion, Go 1.23+ required
- log/slog (stdlib): Structured JSONL logging — zero external dependency, concurrent-safe, idiomatic for Go 1.21+
- golang.org/x/sync (errgroup + semaphore): Parallel agent execution with context cancellation and bounded concurrency
- github.com/anthropics/anthropic-sdk-go v1.27.1: Official Anthropic SDK, v1+ stable with streaming and tool use
- github.com/openai/openai-go/v3 v3.29.0: Official OpenAI SDK replacing community forks
- google.golang.org/genai v1.51+: Official Google Gen AI SDK (GA), replaces deprecated generative-ai-go
- github.com/ollama/ollama/api v0.18.2: Official Ollama client for local model support
- github.com/cenkalti/backoff/v5: Exponential backoff with jitter for provider retries
- github.com/google/renameio: Atomic file writes (temp → rename) for all .planning/ artifacts

**What not to use:** github.com/google/generative-ai-go (deprecated), github.com/liushuangls/go-anthropic (unofficial), logrus (maintenance mode), SQLite (breaks cross-compilation), heavyweight LLM frameworks (eino adds 35+ transitive deps), Uber Fx or Wire (DI containers not warranted for a CLI).

### Expected Features

GopherMind's feature landscape has 12 P1 must-haves, all needed to validate the core thesis. The spec-driven lifecycle, specialized agent contracts, and verification gates are the primary differentiators — no competitor implements all three. The Go intelligence layer and parallel agent execution are P2 (add after v1 validates the sequential workflow). Web UI, cloud sync, and plugin APIs are explicitly deferred to v2+ to avoid scope explosion before product-market fit.

**Must have — table stakes (users expect these at launch):**
- Multi-provider LLM support (Anthropic + OpenAI minimum) — foundational; nothing works without this
- Wave-based project initialization (8 waves) — spec-driven entry point; validates the core thesis
- File-based state persistence in `.planning/` — required for resumability and agent handoffs
- Specialized agent execution protocol with 6 core agents (Architect, Planner, Coder, Reviewer, Tester, Verifier)
- Rule-based task routing — routes tasks to cost-appropriate models; industry data shows 30–70% cost reduction
- Verification gates at phase transitions — blocks phase advance until quality criteria are met
- Cost tracking per session — users need spend visibility before committing to the tool
- Resumable sessions via checkpoint markers — long runs will be interrupted; must work before users trust it
- Structured JSONL event logging — audit trail is part of the value proposition
- Atomic file writes — correctness requirement, not optional
- `init`, `status`, `resume`, `verify`, `logs`, `costs` CLI commands — minimum usable command set
- Git-aware operations (commit after each agent write) — all serious tools do this

**Should have — competitive differentiators (add in v1.x):**
- Go intelligence layer (module graph, convention checks, AST analysis) — no competitor has this; validates Go-specific moat
- Parallel agent execution with DAG scheduler — throughput improvement; defer until sequential execution is solid
- Provider health monitoring and circuit breakers — add when reliability becomes a real user complaint
- Remaining 6+ specialized agents — add as workflows expand beyond core lifecycle
- Ollama / local model support — serves privacy-first users; add after primary providers are stable
- Google Gemini support — third provider after Anthropic + OpenAI are solid

**Defer to v2+:**
- Web UI / dashboard — significant scope; validate CLI value first
- Cloud sync — Git is sufficient for v1; auth + storage is a product pivot
- Plugin / extension API — stabilize internal contracts before making them public
- Non-Go repository support — Go-first is the competitive moat; expand after Go market proven
- Real-time collaboration — single-user tool is correct v1 scope

**Anti-features to refuse:** Full chat/conversational interface (undermines spec-driven value prop), auto-approve all agent actions (destroys trust on failure), IDE extensions (orthogonal to CLI), LLM fine-tuning/model hosting (different business).

### Architecture Approach

GopherMind follows a strict layered architecture with a defined component build order. The system has seven internal packages with explicit dependencies: `config` (no internal deps) → `state` + `provider` + `gointel` → `agent` → `orchestrator` → `cmd/`. This order is not arbitrary — it reflects real runtime dependencies and dictates phase sequencing. All disk I/O for planning artifacts is centralized in `internal/state`; agents never write directly to `.planning/`. The provider abstraction is a narrow Go interface with per-provider adapters, not a unified framework.

**Major components:**
1. CLI Layer (cmd/): Cobra commands — thin wiring only, no business logic; delegates to orchestrator
2. Orchestration Layer (internal/orchestrator/): Session lifecycle, DAG construction, parallel task scheduling with errgroup
3. Agent Runtime (internal/agent/): Agent interface + 7-step execution protocol (load def → assemble context → select provider → execute → validate → write artifact → signal handoff)
4. Provider Layer (internal/provider/): Unified Provider interface with per-provider adapters, rule-based router, fallback chain with exponential backoff
5. Go Intelligence Layer (internal/gointel/): Module detection, package graph via `go list`, convention checking — completely standalone from LLM concerns
6. State Layer (internal/state/): Atomic file writes, JSONL event log (single-writer goroutine), cost accumulator
7. Config (internal/config/): Config struct loaded once at startup, passed via dependency injection — no global singleton

**Key patterns:**
- Provider interface with adapter per backend (swappable, testable via mock)
- Rule-based router with fallback chain (auditable, predictable)
- Agent execution protocol as 7 explicit steps (inspectable, resumable, testable per-step)
- DAG scheduler with errgroup for parallel execution (context-canceling, error-propagating)
- Atomic file writes for all planning artifacts (temp → rename; zero corruption risk)

### Critical Pitfalls

Research surfaced 7 critical pitfalls specific to this problem domain. Multi-agent systems fail in production at 41–86.7% rates per ICML 2025 research — this is not an edge concern. The pitfalls are mapped to phases so prevention is built in, not bolted on.

1. **Provider abstraction is syntactic only, not semantic** — Build three layers: syntactic adapter, behavioral contract tests per provider, and a routing layer encoding known behavioral constraints. Never assume prompts are portable across model families. Pin exact model versions (e.g., `claude-3-5-sonnet-20241022`, not `latest`) in production.

2. **Underspecified agent contracts causing coordination failures** — Every agent definition must have machine-readable input schema, output schema, success criteria, and failure behavior. Validate at every handoff. Treat agent contracts as typed interfaces, not prose. Research: 36.94% of multi-agent failures are coordination failures; 79% of all failures originate from specification issues.

3. **Context window exhaustion causing silent failures** — Treat context as a finite budget. Track tokens per agent call; emit warning at 70% of context budget. Use structural metadata (import graphs, call graphs) not raw file content for code intelligence. Test each agent at high context fill levels.

4. **File-based state without durability guarantees** — Atomic writes (temp → rename) for all full file writes. Single-writer goroutine with buffered channel for JSONL appends. PID/lock file to detect abandoned runs. Checkpoint at activity level, not superstep level. os.Rename is NOT atomic on Windows — test explicitly.

5. **Cost explosion from unguarded agent loops** — Cost instrumentation must be part of the Provider interface from day one, not added later. Per-run budget cap that halts execution and writes checkpoint before stopping. Retry limits enforced at agent level. Validate routing rules for exhaustiveness (no task type falls through to unintended model tier).

6. **Prompt drift and model version fragility** — Version every system prompt alongside the code. Build output schema tests for every structured-output prompt. Log every LLM response with prompt_version and model_version. Test prompt changes against a golden output set.

7. **Spec-to-code drift in agent-generated artifacts** — Build spec-alignment checks into verification gates. Maintain decision logs in STATE.md. Use deterministic scaffolding generated from spec before LLM-written implementation to anchor structure.

## Implications for Roadmap

Architecture research explicitly defines a component build order that maps directly to phases. The pitfalls research adds must-address items to early phases (cost instrumentation, durability guarantees, behavioral contract tests). Feature research validates the ordering — multi-provider support and state persistence are foundational P1 items that everything else depends on.

### Phase 1: Foundation — Config, State, and Provider Layer

**Rationale:** Config, state, and provider have no internal dependencies and are prerequisites for everything else. Cost instrumentation and durability guarantees (two of the top pitfalls) must be addressed here or the cost of fixing them later is HIGH.
**Delivers:** Working Provider interface with Anthropic adapter + mock, atomic state writes, JSONL event log with single-writer goroutine, cost recording per provider call, config loading with Viper, exponential backoff and fallback chain.
**Addresses features:** Multi-provider LLM support (Anthropic v1), file-based state persistence, atomic file writes, JSONL event logging, cost tracking infrastructure, error recovery and retry.
**Avoids pitfalls:** Provider abstraction semantic mismatch (behavioral contract tests built alongside), file-based state durability (atomic writes + single-writer JSONL from day one), cost explosion (Provider interface includes cost recording), Cobra/Viper config override (viper.Get* usage enforced).
**Research flag:** Needs research-phase for provider behavioral contract test patterns — streaming event type differences between Anthropic, OpenAI, and Gemini are non-trivial.

### Phase 2: Agent Runtime — Contracts, Execution Protocol, Core Agents

**Rationale:** Agent execution protocol builds on provider layer. Contracts must be designed before implementation — research shows 79% of multi-agent failures originate from specification issues, not technical failures. Six core agents are the minimum viable set to demonstrate the full lifecycle.
**Delivers:** Agent interface, 7-step execution protocol, AgentDefinition struct with machine-readable input/output schemas and success criteria, 6 core agents (Architect, Planner, Coder, Reviewer, Tester, Verifier), prompt versioning infrastructure, output schema validation.
**Addresses features:** Specialized agent contracts (6 core agents), resumable sessions via checkpoint markers, structured output artifacts, git-aware operations.
**Avoids pitfalls:** Underspecified agent contracts (machine-readable schemas enforced at contract definition), prompt drift (prompt versioning and schema validation built at agent layer), LLM-generated command execution (HUMAN_APPROVAL_REQUIRED gate enforced in agent protocol).
**Research flag:** Standard patterns for this phase — agent execution protocols are well-documented in Google ADK and academic literature.

### Phase 3: Wave-Based Init and Context Assembly

**Rationale:** Project initialization (8-wave questioning) depends on agent runtime being functional. Context assembly strategy must be designed before agents are in production use — context window exhaustion is a silent failure mode that is expensive to retrofit.
**Delivers:** `gophermind init` command with 8-wave questioning protocol, context assembly with token budgeting per agent, context fill warnings at 70% threshold, spec artifact structure (SPEC.md, STATE.md, planning directory conventions).
**Addresses features:** Wave-based project initialization, multi-file codebase understanding, structured output artifacts.
**Avoids pitfalls:** Context window exhaustion (token budgeting built into context assembly, not retrofitted), spec-to-code drift (spec artifacts established at project start anchor all downstream agents).
**Research flag:** Standard patterns — wave-based elicitation is a known domain and the 8-wave structure is already defined in PROJECT.md.

### Phase 4: Orchestration and CLI Command Set

**Rationale:** Orchestrator builds on agent runtime and state. DAG scheduler for sequential execution first — parallel comes in Phase 7. Full CLI command set wired to orchestrator makes the tool usable end-to-end for the first time.
**Delivers:** Session Manager (load/save/resume), sequential task scheduler, `gophermind resume` / `gophermind status` / `gophermind verify` / `gophermind logs` / `gophermind costs` commands, streaming output to terminal, progress visibility during agent runs.
**Addresses features:** Resumable sessions, status and progress visibility, streaming output to terminal, `resume`, `status`, `verify`, `logs`, `costs` commands.
**Avoids pitfalls:** Silent progress (structured progress events streamed to stderr), resume disambiguation (list interrupted runs with timestamp, require explicit selection).
**Research flag:** Standard patterns — session lifecycle and DAG scheduling are well-documented.

### Phase 5: Verification Gates

**Rationale:** Gates depend on agent runtime and orchestrator. Spec-to-code drift and underspecified success criteria are two of the top 7 pitfalls — gates are the prevention mechanism. Building gates after agents are running means first real project runs have quality enforcement.
**Delivers:** Phase transition gates with coverage, consistency, and alignment checks; categorized gate failure output (COVERAGE, CONSISTENCY, COMPLETENESS, ALIGNMENT); deterministic scaffolding from spec before LLM implementation; `gophermind verify` command with structured failure reporting.
**Addresses features:** Verification gates between phases.
**Avoids pitfalls:** Spec-to-code drift (semantic alignment gate checks architectural decisions against generated code), verification gates checking only compilation and tests (gates are semantically opinionated, not just `go build && go test`).
**Research flag:** Needs research-phase — AST-based spec-alignment verification against generated code is not well-documented as a standard pattern. The arXiv paper on AST-based hallucination detection (2601.19106) is a starting point but implementation details need validation.

### Phase 6: Go Intelligence Layer

**Rationale:** Go intelligence is a P2 differentiator. It integrates into the context assembly designed in Phase 3 without requiring architectural changes. Deferring this keeps Phase 1-5 focused on the core protocol. Once added, it makes every earlier agent better.
**Delivers:** Go module detection and parsing, package graph via `go list`, convention checking (error wrapping, context propagation, structured logging patterns), integration into agent context assembly for Go-aware prompts.
**Addresses features:** Go intelligence layer (module graph, convention checks, AST analysis) — the primary competitive moat vs. all existing tools.
**Avoids pitfalls:** Context window exhaustion via selective context loading (only relevant packages, not full file tree).
**Research flag:** Needs research-phase — go/packages API, go/ast usage patterns, and `go list` output parsing have nuances worth verifying before implementation.

### Phase 7: OpenAI + Extended Providers and Parallel Execution

**Rationale:** OpenAI is a required v1 provider but can be deferred until sequential execution is proven. Adding it alongside parallel execution and DAG parallelism keeps the two related concurrency concerns in one phase. Provider behavioral contract tests from Phase 1 expand to cover OpenAI.
**Delivers:** OpenAI provider adapter, rule-based router (routing table in config.json), DAG-parallel agent scheduler with semaphore-bounded goroutines, provider health monitoring with circuit breakers.
**Addresses features:** Multi-provider LLM support (both required providers now complete), rule-based task routing, parallel agent execution, provider health monitoring.
**Avoids pitfalls:** Unbound parallel agent goroutines (weighted semaphore caps concurrent LLM calls), rate limit explosion (separate rate limiters per model tier), provider routing exhaustiveness (routing rules tested for completeness).
**Research flag:** Standard patterns for OpenAI adapter — well-documented. Semaphore-bounded errgroup patterns are well-established in golang.org/x/sync.

### Phase 8: Extended Agents, Gemini, Ollama, and Hardening

**Rationale:** Once core workflow is validated, expand the agent set, add remaining providers, and harden against the operational pitfalls that surface during real use.
**Delivers:** 6+ additional specialized agents, Google Gemini provider adapter, Ollama local model support, cost budget cap enforcement, per-run budget warnings in progress output, Windows atomic write testing, `gophermind config` command with effective configuration display.
**Addresses features:** Extended agent set (12+ total), Ollama support, Google Gemini support, provider health monitoring edge cases, full CLI command set.
**Avoids pitfalls:** Cobra/Viper config override (config source visible in `gophermind config` output), Windows atomic write failure (explicit cross-platform test), cost explosion in extended agent workflows (budget cap enforcement at execution engine level).
**Research flag:** Gemini behavioral differences need research-phase — function_declarations vs. OpenAI tools schema, multimodal handling, and streaming event structure are documented as non-equivalent in pitfalls research.

### Phase Ordering Rationale

- **Config → State → Provider must be Phase 1:** Every other component depends on these. Cost instrumentation and durability guarantees retrofitted later have HIGH recovery cost per pitfalls research.
- **Agent contracts before agent logic (Phase 2):** 79% of multi-agent failures originate from specification issues. Contracts must be machine-readable before any agent implementation.
- **Context assembly strategy in Phase 3, not Phase 6:** Context budgeting is a design constraint on agent implementation. Retrofitting it is expensive; doing it before agents run production workloads is cheap.
- **Orchestrator and CLI in Phase 4:** Depends on agent runtime. Enables end-to-end testing for the first time.
- **Verification gates before Go intelligence (Phase 5 before Phase 6):** Gates use Go intelligence when available but must work without it. Build the gate structure first; Go intelligence plugs in as a context source.
- **Parallel execution deferred to Phase 7:** Sequential execution must be reliable first. Parallel adds concurrency complexity that is easier to validate against a working sequential baseline.
- **Extended providers and agents in Phase 8:** Anthropic-only v1 validates the architecture. Gemini and Ollama behavioral differences are better addressed after the provider contract test suite is mature.

### Research Flags

Phases likely needing deeper research during planning:
- **Phase 1:** Provider behavioral contract tests for streaming event type differences — Anthropic, OpenAI, Gemini all have distinct streaming SSE schemas; test patterns need validation
- **Phase 5:** AST-based spec-alignment verification — no established standard pattern; arXiv 2601.19106 is a starting point but implementation approach needs research
- **Phase 6:** go/packages API and go/ast patterns for convention checking — interface nuances worth verifying before committing to implementation approach
- **Phase 8:** Gemini provider behavioral differences — function_declarations schema, streaming semantics, and multimodal handling are non-equivalent to OpenAI tools

Phases with standard patterns (skip research-phase):
- **Phase 2:** Agent execution protocols are well-documented in Google ADK and academic literature
- **Phase 3:** Wave-based elicitation structure is pre-defined; context budgeting patterns are straightforward
- **Phase 4:** Session lifecycle and DAG scheduling in Go are well-documented; errgroup scheduler is a known pattern
- **Phase 7:** OpenAI adapter and semaphore-bounded goroutine patterns are well-established

## Confidence Assessment

| Area | Confidence | Notes |
|------|------------|-------|
| Stack | HIGH | All libraries verified on pkg.go.dev as of March 2026 with exact versions; Go minimum version driven by concrete compatibility constraints (viper v1.21 + backoff v5 require Go 1.23) |
| Features | HIGH | Competitive landscape thoroughly documented against Aider, Cursor, Kiro, OpenHands with official sources; MVP definition validated against dependency graph |
| Architecture | HIGH | Component build order corroborated by multiple production Go patterns; code examples are concrete and compilable; anti-patterns backed by specific failure modes |
| Pitfalls | HIGH | Primary claims backed by ICML 2025 academic research (MAST taxonomy), production post-mortems, and official documentation; failure rates are empirically cited |

**Overall confidence:** HIGH

### Gaps to Address

- **Prompt schema design for agent contracts:** The research recommends machine-readable schemas for agent input/output but does not specify the schema format (JSON Schema, Go structs, custom DSL). Decide during Phase 2 planning whether schemas are defined as Go structs, JSON Schema files, or embedded YAML — this decision affects the agent definition loading pattern.
- **Windows cross-platform testing strategy:** Atomic writes via os.Rename are not atomic on Windows. Research identifies this as a known issue but does not recommend an alternative write strategy for Windows. Decide during Phase 1 whether to support Windows at v1 or defer with explicit documentation.
- **Context compaction strategy for long sessions:** Pitfalls research flags 500+ message history as a performance trap requiring summarization and rolling windows, but does not specify the compaction algorithm. This needs a concrete design decision before Phase 3 context assembly work.
- **Checkpoint granularity definition:** Research recommends activity-level checkpointing (not superstep-level) but does not define what constitutes an "activity" within GopherMind's agent execution protocol. Define this precisely during Phase 2 agent contract design.

## Sources

### Primary (HIGH confidence)
- pkg.go.dev/github.com/anthropics/anthropic-sdk-go — v1.27.1 verified Mar 18, 2026
- pkg.go.dev/github.com/openai/openai-go — v3.29.0 verified Mar 17, 2026
- pkg.go.dev/google.golang.org/genai — v1.51.0 GA status confirmed Mar 18, 2026
- pkg.go.dev/github.com/spf13/cobra — v1.10.2 confirmed
- pkg.go.dev/github.com/spf13/viper — v1.21.0, Go 1.23+ required
- pkg.go.dev/golang.org/x/sync — official Go extended library
- arXiv 2503.13657 (ICML 2025 Spotlight) — MAST taxonomy; 41-86.7% multi-agent failure rate; 36.94% coordination failures
- kiro.dev/docs/specs/ — official Kiro spec-driven development documentation
- golang-standards/project-layout — standard Go project structure
- Michael Stapelberg — atomically writing files in Go (temp-file-rename pattern)
- Carolyn Van Slyck — Cobra + Viper flag binding pitfalls
- arXiv 2601.19106 — AST-based hallucination detection in LLM-generated code

### Secondary (MEDIUM confidence)
- dasroot.net/posts/2026/02 — AI workflow patterns in Go CLI tools
- dasroot.net/posts/2026/03 — building multi-model LLM gateway in Go
- diagrid.io — checkpointing vs. durable execution analysis
- fdrechsler.de — provider-agnostic agents: syntactic vs. semantic abstraction
- factory.ai — context window problem analysis
- comet.com — prompt drift in agentic systems
- portkey.ai — retries, fallbacks, and circuit breakers in LLM apps
- lushbinary.com — AI coding agents 2026 comparison
- mindstudio.ai — model routing cost reduction data (30-70% claim)

### Tertiary (LOW confidence)
- Community comparisons of Aider, Kiro, OpenHands feature sets — used for competitive analysis only; individual feature claims validated against official sources where possible

---
*Research completed: 2026-03-20*
*Ready for roadmap: yes*
