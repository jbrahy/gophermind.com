# Requirements: GopherMind

**Defined:** 2026-03-20
**Core Value:** Every development decision is explicit, every step is verifiable, and every session is resumable

## v1 Requirements

Requirements for initial release. Each maps to roadmap phases.

### Provider Layer

- [ ] **PROV-01**: System supports Anthropic Claude API with streaming responses
- [ ] **PROV-02**: System supports OpenAI API (Chat Completions) with streaming responses
- [ ] **PROV-03**: System supports Google Gemini API with streaming responses
- [ ] **PROV-04**: System supports Ollama local models with streaming responses
- [ ] **PROV-05**: All providers implement a unified Provider interface (Complete, Stream, HealthCheck, GetCost)
- [ ] **PROV-06**: Provider requests are normalized across different API formats
- [ ] **PROV-07**: Provider failures retry up to 3 times with exponential backoff (1s, 2s, 4s)
- [ ] **PROV-08**: Failed provider falls back to equivalent-tier provider from different vendor
- [ ] **PROV-09**: Provider health checks detect unavailability before routing requests
- [ ] **PROV-10**: All provider calls log token usage, cost, duration, and success/failure to JSONL

### Cost Tracking

- [ ] **COST-01**: Every LLM call records input tokens, output tokens, and calculated cost
- [ ] **COST-02**: Costs accumulate per provider, per agent, and per tier in costs.jsonl
- [ ] **COST-03**: User can view cost breakdown via `gsd costs --breakdown`
- [ ] **COST-04**: Per-run budget cap prevents runaway costs (configurable in config.json)

### Routing

- [ ] **ROUT-01**: Task-based routing assigns providers/models by task category (architecture, code gen, summarization, etc.)
- [ ] **ROUT-02**: Routing rules are configured in config.json, not hardcoded
- [ ] **ROUT-03**: Routing respects model tier assignments (fast/balanced/powerful)
- [ ] **ROUT-04**: Tier escalation occurs when all same-tier providers fail on critical tasks

### State Persistence

- [ ] **STAT-01**: All project state persists in .planning/ directory as structured files
- [ ] **STAT-02**: STATE.md tracks current phase, task, progress, decisions, risks, and metrics
- [ ] **STAT-03**: File writes use atomic temp-file-rename pattern to prevent corruption
- [ ] **STAT-04**: JSONL log appends use single-writer goroutine for concurrency safety
- [ ] **STAT-05**: In-progress runs are detected via PID/lock files
- [ ] **STAT-06**: System is fully resumable after interruption — reads state from disk on restart
- [ ] **STAT-07**: Checkpoint markers in STATE.md enable resume from last completed step

### Agent System

- [ ] **AGNT-01**: Each agent has a machine-readable contract: inputs, outputs, success criteria, failure modes
- [ ] **AGNT-02**: Agent execution follows 7-step protocol: load → context → select provider → execute → validate → write → handoff
- [ ] **AGNT-03**: Agents are pure functions (context in, output out) — the executor owns disk writes
- [ ] **AGNT-04**: Agent output is validated against success criteria before acceptance
- [ ] **AGNT-05**: Handoff validation rejects malformed agent output
- [ ] **AGNT-06**: System includes 12+ specialized agents: Researcher, Architect, Requirements Synthesizer, Roadmapper, Go Conventions Guardian, Test Strategist, Security Reviewer, Operations Planner, State Keeper, Code Generator, Code Reviewer, Debugger
- [ ] **AGNT-07**: Agent prompts are versioned; version IDs are embedded in JSONL logs
- [ ] **AGNT-08**: Per-agent context budget enforcement prevents context window exhaustion

### Orchestration

- [ ] **ORCH-01**: Workflow orchestrator coordinates agent execution sequences
- [ ] **ORCH-02**: Task dependency graph determines execution order (topological sort)
- [ ] **ORCH-03**: Independent agents execute in parallel via errgroup
- [ ] **ORCH-04**: Dependent agents block until predecessors complete
- [ ] **ORCH-05**: Phase transitions are enforced — cannot advance without exit criteria met
- [ ] **ORCH-06**: Orchestrator handles resumption after interruptions using STATE.md

### CLI Interface

- [ ] **CLI-01**: `gsd init` runs wave-based questioning and produces planning artifacts
- [ ] **CLI-02**: `gsd status` displays current phase, task progress, and metrics
- [ ] **CLI-03**: `gsd resume` continues interrupted workflow from last checkpoint
- [ ] **CLI-04**: `gsd view <artifact>` displays planning artifacts with formatting
- [ ] **CLI-05**: `gsd verify` runs verification checks against planning artifacts
- [ ] **CLI-06**: `gsd logs [--provider]` shows structured event and provider logs
- [ ] **CLI-07**: `gsd costs [--breakdown]` displays cost breakdown by provider/agent/tier
- [ ] **CLI-08**: `gsd config [--edit]` views or modifies configuration
- [ ] **CLI-09**: CLI streams LLM output to terminal in real-time (not buffered)
- [ ] **CLI-10**: CLI displays agent progress indicators during execution

### Initialization

- [ ] **INIT-01**: Wave-based questioning protocol (8 waves) gathers complete project context
- [ ] **INIT-02**: Produces PROJECT.md with mission, requirements, constraints, decisions
- [ ] **INIT-03**: Produces config.json with all workflow settings
- [ ] **INIT-04**: Research phase spawns 4 parallel researcher agents
- [ ] **INIT-05**: Requirements synthesis generates numbered, testable requirements
- [ ] **INIT-06**: Roadmapper creates phased delivery plan mapped to requirements
- [ ] **INIT-07**: All init artifacts are committed to git
- [ ] **INIT-08**: Partial answers are saved on interruption for resume

### Go Intelligence

- [ ] **GINT-01**: Detects Go modules (go.mod parsing) and workspaces (go.work)
- [ ] **GINT-02**: Builds package dependency graph for target repository
- [ ] **GINT-03**: Identifies test files, benchmarks, and examples
- [ ] **GINT-04**: Checks Go conventions: package layout, naming, error wrapping, context usage
- [ ] **GINT-05**: Enforces structured logging patterns (slog usage)
- [ ] **GINT-06**: Generates repository intelligence summary for agent context
- [ ] **GINT-07**: Distinguishes internal vs public API packages

### Verification

- [ ] **VERF-01**: Coverage checks verify every requirement appears in at least one phase
- [ ] **VERF-02**: Consistency checks verify architecture matches v1 scope
- [ ] **VERF-03**: Completeness checks verify no placeholder text or TODOs in artifacts
- [ ] **VERF-04**: Alignment checks verify out-of-scope items are documented
- [ ] **VERF-05**: Semantic alignment checks compare implementation against spec artifacts
- [ ] **VERF-06**: Verification results are logged to events.jsonl

### Configuration

- [ ] **CONF-01**: config.json stores all system settings with sensible defaults
- [ ] **CONF-02**: Configuration supports mode (interactive/yolo), granularity, parallelization
- [ ] **CONF-03**: LLM settings: multi-provider, routing mode, fallback, retry, timeout
- [ ] **CONF-04**: Go settings: conventions enforcement, tool paths, coverage targets
- [ ] **CONF-05**: Safety settings: file write permissions, command execution restrictions
- [ ] **CONF-06**: Logging settings: level, structured format, file paths

### Logging

- [ ] **LOG-01**: Structured event log (events.jsonl) records agent lifecycle events
- [ ] **LOG-02**: Provider call log (provider-calls.jsonl) records every LLM API call
- [ ] **LOG-03**: Cost log (costs.jsonl) tracks cumulative spending
- [ ] **LOG-04**: Application logging uses slog with JSON output
- [ ] **LOG-05**: Log levels: debug, info, warn, error — configurable at runtime

### Git Integration

- [ ] **GIT-01**: Agent file writes are auto-committed with descriptive messages
- [ ] **GIT-02**: Planning artifacts in .planning/ are git-tracked
- [ ] **GIT-03**: Phase-based branching strategy supported (configurable)

### Security

- [ ] **SEC-01**: API keys stored in environment variables, never in committed files
- [ ] **SEC-02**: Source code privacy: code only sent to providers when explicitly configured
- [ ] **SEC-03**: File system writes restricted to project directory
- [ ] **SEC-04**: Command execution requires explicit approval (configurable)
- [ ] **SEC-05**: All security-relevant actions logged for audit

## v2 Requirements

### Extended Intelligence

- **GINT-10**: Support for non-Go repositories (Python, TypeScript, Rust) with Go-optimized defaults
- **GINT-11**: Advanced Go analysis: dead code detection, interface compliance checking
- **GINT-12**: Performance profiling integration (pprof analysis)

### Web Interface

- **WEB-01**: Local web UI for artifact viewing and project navigation
- **WEB-02**: Real-time execution dashboard with agent progress
- **WEB-03**: Diff viewer for code changes

### Collaboration

- **COLLAB-01**: Multi-user session support with conflict resolution
- **COLLAB-02**: Shared planning artifact review and approval

### Advanced Routing

- **ROUT-10**: Dynamic routing based on task complexity scoring
- **ROUT-11**: A/B testing of model performance per task type
- **ROUT-12**: Automatic model selection based on cost/quality optimization

## Out of Scope

| Feature | Reason |
|---------|--------|
| Chat/conversational interface | GopherMind wins on structured outputs, not chat; chat trains vague responses |
| Plugin/extension API | Premature extensibility; internal architecture first |
| LLM fine-tuning/model hosting | Operating ML infra is a separate business |
| Cloud sync/SaaS mode | Git push handles sync; cloud requires auth, billing, encryption |
| IDE integration (LSP, extension) | CLI works in any terminal including IDE terminals |
| Real-time collaboration | Distributed systems problem disguised as a feature |
| Auto-approve all actions | Silent automation destroys trust; gates exist for a reason |

## Traceability

| Requirement | Phase | Status |
|-------------|-------|--------|
| PROV-01 | Phase 1 | Pending |
| PROV-02 | Phase 1 | Pending |
| PROV-05 | Phase 1 | Pending |
| PROV-06 | Phase 1 | Pending |
| PROV-07 | Phase 1 | Pending |
| PROV-10 | Phase 1 | Pending |
| COST-01 | Phase 1 | Pending |
| COST-02 | Phase 1 | Pending |
| STAT-01 | Phase 1 | Pending |
| STAT-02 | Phase 1 | Pending |
| STAT-03 | Phase 1 | Pending |
| STAT-04 | Phase 1 | Pending |
| STAT-06 | Phase 1 | Pending |
| CONF-01 | Phase 1 | Pending |
| CONF-02 | Phase 1 | Pending |
| CONF-03 | Phase 1 | Pending |
| LOG-04 | Phase 1 | Pending |
| LOG-05 | Phase 1 | Pending |
| AGNT-01 | Phase 2 | Pending |
| AGNT-02 | Phase 2 | Pending |
| AGNT-03 | Phase 2 | Pending |
| AGNT-04 | Phase 2 | Pending |
| AGNT-05 | Phase 2 | Pending |
| AGNT-07 | Phase 2 | Pending |
| AGNT-08 | Phase 2 | Pending |
| LOG-01 | Phase 2 | Pending |
| LOG-02 | Phase 2 | Pending |
| LOG-03 | Phase 2 | Pending |
| INIT-01 | Phase 3 | Pending |
| INIT-02 | Phase 3 | Pending |
| INIT-03 | Phase 3 | Pending |
| INIT-08 | Phase 3 | Pending |
| CLI-01 | Phase 3 | Pending |
| ORCH-01 | Phase 4 | Pending |
| ORCH-02 | Phase 4 | Pending |
| ORCH-05 | Phase 4 | Pending |
| ORCH-06 | Phase 4 | Pending |
| CLI-02 | Phase 4 | Pending |
| CLI-03 | Phase 4 | Pending |
| CLI-04 | Phase 4 | Pending |
| CLI-06 | Phase 4 | Pending |
| CLI-08 | Phase 4 | Pending |
| STAT-05 | Phase 4 | Pending |
| STAT-07 | Phase 4 | Pending |
| VERF-01 | Phase 5 | Pending |
| VERF-02 | Phase 5 | Pending |
| VERF-03 | Phase 5 | Pending |
| VERF-04 | Phase 5 | Pending |
| VERF-05 | Phase 5 | Pending |
| VERF-06 | Phase 5 | Pending |
| CLI-05 | Phase 5 | Pending |
| GINT-01 | Phase 6 | Pending |
| GINT-02 | Phase 6 | Pending |
| GINT-03 | Phase 6 | Pending |
| GINT-04 | Phase 6 | Pending |
| GINT-05 | Phase 6 | Pending |
| GINT-06 | Phase 6 | Pending |
| GINT-07 | Phase 6 | Pending |
| PROV-03 | Phase 7 | Pending |
| PROV-04 | Phase 7 | Pending |
| PROV-08 | Phase 7 | Pending |
| PROV-09 | Phase 7 | Pending |
| ORCH-03 | Phase 7 | Pending |
| ORCH-04 | Phase 7 | Pending |
| ROUT-01 | Phase 7 | Pending |
| ROUT-02 | Phase 7 | Pending |
| ROUT-03 | Phase 7 | Pending |
| ROUT-04 | Phase 7 | Pending |
| AGNT-06 | Phase 8 | Pending |
| INIT-04 | Phase 8 | Pending |
| INIT-05 | Phase 8 | Pending |
| INIT-06 | Phase 8 | Pending |
| INIT-07 | Phase 8 | Pending |
| COST-03 | Phase 8 | Pending |
| COST-04 | Phase 8 | Pending |
| CLI-07 | Phase 8 | Pending |
| CLI-09 | Phase 8 | Pending |
| CLI-10 | Phase 8 | Pending |
| GIT-01 | Phase 8 | Pending |
| GIT-02 | Phase 8 | Pending |
| GIT-03 | Phase 8 | Pending |
| SEC-01 | Phase 9 | Pending |
| SEC-02 | Phase 9 | Pending |
| SEC-03 | Phase 9 | Pending |
| SEC-04 | Phase 9 | Pending |
| SEC-05 | Phase 9 | Pending |
| CONF-04 | Phase 9 | Pending |
| CONF-05 | Phase 9 | Pending |
| CONF-06 | Phase 9 | Pending |

**Coverage:**
- v1 requirements: 82 total
- Mapped to phases: 82
- Unmapped: 0 ✓

---
*Requirements defined: 2026-03-20*
*Last updated: 2026-03-20 — traceability confirmed against ROADMAP.md*
