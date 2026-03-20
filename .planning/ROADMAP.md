# Roadmap: GopherMind

## Overview

GopherMind is built in strict component dependency order. Config, state, and provider layers form the foundation everything else depends on. Agent contracts and execution protocol come next — research shows 79% of multi-agent failures originate from specification issues, so contracts must be machine-readable before any implementation. From there, initialization, orchestration, and CLI complete the working system. Verification gates harden quality enforcement. Go intelligence adds the primary competitive differentiator. Extended providers and parallel execution expand throughput. The full agent set plus git, cost, and streaming CLI complete the feature surface. Security and operational hardening close the loop before v1 ships.

## Phases

**Phase Numbering:**
- Integer phases (1, 2, 3): Planned milestone work
- Decimal phases (2.1, 2.2): Urgent insertions (marked with INSERTED)

Decimal phases appear between their surrounding integers in numeric order.

- [ ] **Phase 1: Foundation** - Config, state, provider layer with Anthropic, cost instrumentation, atomic writes, JSONL logging
- [ ] **Phase 2: Agent Runtime** - Agent contracts, 7-step execution protocol, core agent set, prompt versioning
- [ ] **Phase 3: Initialization** - Wave-based 8-wave questioning, PROJECT.md/config.json production, resume on interruption
- [ ] **Phase 4: Orchestration and Core CLI** - Session manager, sequential DAG scheduler, status/resume/logs/config commands
- [ ] **Phase 5: Verification Gates** - Coverage, consistency, completeness, and alignment checks; `gsd verify` command
- [ ] **Phase 6: Go Intelligence** - Module detection, package graph, convention checking, repo intelligence for agent context
- [ ] **Phase 7: Extended Providers and Parallel Execution** - OpenAI, routing, fallback chains, parallel agent DAG with errgroup
- [ ] **Phase 8: Full Agent Set, Git, Costs, Streaming CLI** - 12+ agents, parallel init, git commits, cost caps, streaming output
- [ ] **Phase 9: Security and Operational Hardening** - API key handling, source privacy, write restrictions, audit logging, Go/logging config

## Phase Details

### Phase 1: Foundation
**Goal**: Developers can make real LLM calls through a unified provider interface, with every call costed, logged, and durably persisted — the bedrock layer all other components depend on
**Depends on**: Nothing (first phase)
**Requirements**: PROV-01, PROV-02, PROV-05, PROV-06, PROV-07, PROV-10, COST-01, COST-02, STAT-01, STAT-02, STAT-03, STAT-04, STAT-06, CONF-01, CONF-02, CONF-03, LOG-04, LOG-05
**Success Criteria** (what must be TRUE):
  1. A developer can call the Anthropic and OpenAI APIs through a single Provider interface and receive a streaming response
  2. Every provider call records input tokens, output tokens, calculated cost, and duration in costs.jsonl and provider-calls.jsonl without any manual instrumentation
  3. Files written to .planning/ survive a crash mid-write with no corruption (atomic temp-file-rename pattern in use)
  4. JSONL log appends from concurrent goroutines never interleave partial records (single-writer goroutine enforced)
  5. The system reads config.json on startup and applies the correct provider routing mode, granularity, and log level without code changes
**Plans**: TBD

Plans:
- [ ] 01-01: Project scaffolding — go.mod, directory structure, Cobra/Viper wiring, build and test pipeline
- [ ] 01-02: Config layer — config.json struct, Viper loading, defaults, environment variable override
- [ ] 01-03: State layer — atomic file writes, JSONL single-writer goroutine, .planning/ directory conventions, STATE.md bootstrap
- [ ] 01-04: Provider interface and Anthropic adapter — Provider interface, streaming, request normalization, behavioral contract tests
- [ ] 01-05: OpenAI adapter — OpenAI Chat Completions adapter implementing unified Provider interface with streaming
- [ ] 01-06: Cost instrumentation — per-call token accounting, cost accumulation in costs.jsonl, provider-calls.jsonl
- [ ] 01-07: Retry and backoff — exponential backoff (1s/2s/4s), 3-retry limit, provider-level error classification
- [ ] 01-08: Structured logging — slog JSON output, configurable log level at runtime, log file routing

### Phase 2: Agent Runtime
**Goal**: Agents with machine-readable contracts execute the 7-step protocol reliably, validate their outputs, and hand off cleanly — the execution primitive every workflow depends on
**Depends on**: Phase 1
**Requirements**: AGNT-01, AGNT-02, AGNT-03, AGNT-04, AGNT-05, AGNT-07, AGNT-08, LOG-01, LOG-02, LOG-03
**Success Criteria** (what must be TRUE):
  1. Every agent definition ships with a machine-readable contract: typed input schema, typed output schema, success criteria, and documented failure modes
  2. An agent execution run produces a JSONL event log entry at each of the 7 steps (load, context, select-provider, execute, validate, write, handoff) — missed steps are detectable in the log
  3. An agent that produces output failing its success criteria is rejected before any file write occurs
  4. Malformed handoff output (wrong schema, missing fields) causes the handoff to fail with a structured error, not a silent write
  5. Every LLM response in events.jsonl carries a prompt_version and model_version field for traceability
**Plans**: TBD

Plans:
- [ ] 02-01: Agent interface and AgentDefinition struct — input/output schemas, success criteria, failure modes, contract loader
- [ ] 02-02: 7-step execution protocol — load def → assemble context → select provider → execute → validate → write → handoff
- [ ] 02-03: Output validation and handoff rejection — success criteria evaluation, schema validation, structured failure reporting
- [ ] 02-04: Prompt versioning infrastructure — version IDs embedded in agent definitions, version logged in every JSONL record
- [ ] 02-05: Per-agent context budget enforcement — token budget per agent call, warning at 70% threshold, hard stop at limit
- [ ] 02-06: Agent event logging — events.jsonl lifecycle events, provider-calls.jsonl per-call records, costs.jsonl accumulation
- [ ] 02-07: Core agent definitions — Architect, Planner, Coder, Reviewer, Tester, Verifier (contracts + prompts, not full implementations)

### Phase 3: Initialization
**Goal**: Running `gsd init` produces complete, committed planning artifacts from an 8-wave interactive interview, and a partial run can be resumed exactly where it stopped
**Depends on**: Phase 2
**Requirements**: INIT-01, INIT-02, INIT-03, INIT-08, CLI-01
**Success Criteria** (what must be TRUE):
  1. `gsd init` walks the user through 8 questioning waves, collecting answers interactively, and produces a valid PROJECT.md and config.json in .planning/
  2. Interrupting `gsd init` mid-wave saves all answers collected so far; re-running `gsd init` detects the partial state and resumes from the next unanswered wave
  3. The generated config.json contains all required fields with sensible defaults and passes schema validation
  4. PROJECT.md contains mission, requirements, constraints, and decisions sections populated from wave answers
**Plans**: TBD

Plans:
- [ ] 03-01: CLI scaffolding — `gsd init` Cobra command, top-level help, command dispatch wiring
- [ ] 03-02: Wave-based questioning engine — 8-wave protocol, question sequencing, answer collection, progress display
- [ ] 03-03: Partial-state persistence — save answers on each wave completion, detect and load incomplete init state on restart
- [ ] 03-04: Artifact generation — PROJECT.md writer, config.json writer from collected answers, validation of produced artifacts

### Phase 4: Orchestration and Core CLI
**Goal**: A completed `gsd init` run can be resumed, monitored, and navigated through the full CLI command set — the system is usable end-to-end for the first time
**Depends on**: Phase 3
**Requirements**: ORCH-01, ORCH-02, ORCH-05, ORCH-06, CLI-02, CLI-03, CLI-04, CLI-06, CLI-08, STAT-05, STAT-07
**Success Criteria** (what must be TRUE):
  1. `gsd status` displays the current phase name, task progress count, and key metrics read from STATE.md without starting any LLM calls
  2. `gsd resume` detects the last checkpoint in STATE.md and continues execution from that step, not from the beginning of the phase
  3. `gsd view <artifact>` renders any .planning/ artifact with readable formatting to the terminal
  4. `gsd logs` shows the structured event log and `gsd logs --provider` shows provider-call records, both paginated
  5. A workflow cannot advance to the next phase if the current phase exit criteria have not been met — the orchestrator rejects the transition with a clear message
**Plans**: TBD

Plans:
- [ ] 04-01: Session manager — load/save/resume lifecycle, STATE.md read/write, session lock via PID file
- [ ] 04-02: Sequential task scheduler — topological sort of agent dependency graph, sequential execution, phase exit criteria enforcement
- [ ] 04-03: Checkpoint system — checkpoint markers written to STATE.md at activity completion, resume reads last checkpoint
- [ ] 04-04: `gsd status` command — phase, progress, metrics display from STATE.md
- [ ] 04-05: `gsd resume` command — detect interrupted run, present checkpoint, continue from last step
- [ ] 04-06: `gsd view` command — artifact rendering for .planning/ files with formatting
- [ ] 04-07: `gsd logs` command — event log and provider log display with --provider flag
- [ ] 04-08: `gsd config` command — view effective configuration from config.json

### Phase 5: Verification Gates
**Goal**: Every phase transition is guarded by automated checks — no phase advances until coverage, consistency, completeness, and alignment criteria are all satisfied
**Depends on**: Phase 4
**Requirements**: VERF-01, VERF-02, VERF-03, VERF-04, VERF-05, VERF-06, CLI-05
**Success Criteria** (what must be TRUE):
  1. `gsd verify` runs all four check categories (COVERAGE, CONSISTENCY, COMPLETENESS, ALIGNMENT) and outputs a structured pass/fail report with per-check details
  2. A phase with any requirement not appearing in any plan fails the COVERAGE check and blocks transition
  3. An artifact containing placeholder text or TODO markers fails the COMPLETENESS check with the specific file and line number reported
  4. Out-of-scope items present in phase artifacts fail the ALIGNMENT check with a reference to the scope boundary
  5. All verification results are appended to events.jsonl with check type, result, and any failure details
**Plans**: TBD

Plans:
- [ ] 05-01: Verification framework — check registry, result types, structured failure output, gate enforcement integration
- [ ] 05-02: Coverage check — every v1 requirement appears in at least one phase plan
- [ ] 05-03: Consistency check — architecture artifacts match v1 scope definitions
- [ ] 05-04: Completeness check — no placeholder text or TODOs in committed artifacts
- [ ] 05-05: Alignment check — out-of-scope items documented, not present in phase deliverables
- [ ] 05-06: Semantic alignment check — implementation artifacts compared against spec artifacts for decision drift
- [ ] 05-07: `gsd verify` command — runs all checks, structured output, exits non-zero on failure, logs to events.jsonl

### Phase 6: Go Intelligence
**Goal**: Every agent that generates or reviews Go code receives a repository intelligence summary — module structure, package graph, and convention violations — injected into its context automatically
**Depends on**: Phase 5
**Requirements**: GINT-01, GINT-02, GINT-03, GINT-04, GINT-05, GINT-06, GINT-07
**Success Criteria** (what must be TRUE):
  1. On a Go repository with go.mod, the intelligence layer correctly identifies all packages, their import relationships, and distinguishes internal from public API packages
  2. Convention checking detects and reports violations of: error wrapping patterns, context propagation, structured logging (slog) usage, and package layout standards
  3. The generated repository intelligence summary is injected into agent context assembly for Go-aware agents, and the token budget enforcement from Phase 2 applies to this summary
  4. Test files, benchmarks, and examples are identified separately from production code in the package graph
**Plans**: TBD

Plans:
- [ ] 06-01: Module detection — go.mod parsing, go.work workspace detection, module graph construction
- [ ] 06-02: Package graph — `go list` integration, import graph, internal vs public API package classification
- [ ] 06-03: Test file identification — test files, benchmarks, examples separated from production code
- [ ] 06-04: Convention checker — error wrapping, context propagation, slog usage, package layout validation
- [ ] 06-05: Structured logging enforcement — slog pattern detection, non-slog logging library flagging
- [ ] 06-06: Repository intelligence summary generator — compact summary struct for agent context injection
- [ ] 06-07: Context assembly integration — inject Go intelligence into agent context assembly within token budget

### Phase 7: Extended Providers and Parallel Execution
**Goal**: Routing sends tasks to cost-appropriate models across multiple providers, independent agents run in parallel, and providers that go down trigger automatic fallback without user intervention
**Depends on**: Phase 6
**Requirements**: PROV-03, PROV-04, PROV-08, PROV-09, ORCH-03, ORCH-04, ROUT-01, ROUT-02, ROUT-03, ROUT-04
**Success Criteria** (what must be TRUE):
  1. Tasks are routed to the correct provider and model tier based on the routing table in config.json — an architecture task goes to the powerful tier and a summarization task goes to the fast tier
  2. When all providers in a tier fail a critical task, the orchestrator automatically escalates to the next tier and logs the escalation event
  3. Independent agents in a phase execute concurrently via errgroup with a semaphore cap on simultaneous LLM calls, completing faster than sequential execution
  4. A provider that fails health checks is removed from routing until it recovers, with no manual intervention required
  5. Google Gemini and Ollama local models are available as routing targets alongside Anthropic and OpenAI
**Plans**: TBD

Plans:
- [ ] 07-01: Rule-based router — task category to provider/model mapping, routing table loaded from config.json, tier assignments
- [ ] 07-02: Tier escalation — all-same-tier failure detection, critical task escalation to next tier, escalation event logging
- [ ] 07-03: Provider health monitoring — health check before routing, unavailability detection, recovery detection
- [ ] 07-04: Fallback chain — failed provider falls back to equivalent-tier provider from different vendor
- [ ] 07-05: Parallel agent scheduler — errgroup-based parallel execution, weighted semaphore cap, dependency blocking
- [ ] 07-06: Google Gemini provider adapter — genai SDK integration, streaming, request normalization, behavioral contract tests
- [ ] 07-07: Ollama local model adapter — ollama/api integration, streaming, local model discovery

### Phase 8: Full Agent Set, Git Integration, Cost Controls, Streaming CLI
**Goal**: GopherMind runs the complete 12+ agent lifecycle with git-committed artifacts, enforced cost budgets, real-time streaming output, and parallel research during initialization
**Depends on**: Phase 7
**Requirements**: AGNT-06, INIT-04, INIT-05, INIT-06, INIT-07, COST-03, COST-04, CLI-07, CLI-09, CLI-10, GIT-01, GIT-02, GIT-03
**Success Criteria** (what must be TRUE):
  1. `gsd costs --breakdown` displays costs grouped by provider, agent, and tier with session totals readable at a glance
  2. When a run hits the configured per-run budget cap, execution halts, a checkpoint is written, and the user sees a clear message with current spend before the process exits
  3. Every agent file write to .planning/ is followed by an auto-commit with a descriptive message — git log shows one commit per artifact produced
  4. LLM output streams to the terminal in real-time during agent execution — no buffering until response is complete
  5. Agent progress indicators update during execution so the user always knows which agent is running and what step it is on
  6. During `gsd init`, four researcher agents run in parallel and their results are synthesized by the Requirements Synthesizer agent
**Plans**: TBD

Plans:
- [ ] 08-01: Extended agent definitions — Researcher, Requirements Synthesizer, Roadmapper, Go Conventions Guardian, Test Strategist, Security Reviewer, Operations Planner, State Keeper, Debugger (contracts + prompts)
- [ ] 08-02: Parallel init — 4 parallel researcher agents via errgroup during `gsd init` research phase
- [ ] 08-03: Requirements synthesis agent — synthesizes parallel researcher outputs into numbered, testable requirements
- [ ] 08-04: Roadmapper agent — produces phased delivery plan from requirements, integrated with init workflow
- [ ] 08-05: Git integration — auto-commit after agent writes, .planning/ tracking, phase-based branching (configurable)
- [ ] 08-06: Cost controls — per-run budget cap enforcement, budget warning at configurable threshold, `gsd costs --breakdown` command
- [ ] 08-07: Streaming CLI output — real-time LLM response streaming to terminal, no buffering
- [ ] 08-08: Agent progress indicators — per-agent step display during execution, current agent and step always visible

### Phase 9: Security and Operational Hardening
**Goal**: GopherMind handles API keys safely, restricts writes to the project directory, requires approval for command execution, and logs all security-relevant actions — ready for real-world daily use
**Depends on**: Phase 8
**Requirements**: SEC-01, SEC-02, SEC-03, SEC-04, SEC-05, CONF-04, CONF-05, CONF-06
**Success Criteria** (what must be TRUE):
  1. API keys are read from environment variables only — the config.json and .planning/ files contain no secrets, and `git log --all -p` reveals no key material
  2. Source code is never sent to an external LLM provider unless the user has explicitly set `code_privacy: false` in config.json — the default behavior keeps code local
  3. All file writes from agents are restricted to the project directory — any attempt to write outside is blocked and logged as a security event
  4. Command execution by agents requires explicit user approval in interactive mode — the approval prompt shows the exact command before execution
  5. All security-relevant actions (key read, file write, command execution attempt, approval granted/denied) appear in events.jsonl with a `security` tag
**Plans**: TBD

Plans:
- [ ] 09-01: API key security — env-var-only key loading, key redaction from logs, no-key-in-file enforcement
- [ ] 09-02: Source code privacy — code_privacy config flag, provider call interception to strip code when privacy=true
- [ ] 09-03: File write restrictions — project directory sandbox, out-of-bounds write blocking, security event logging
- [ ] 09-04: Command execution approval — interactive approval gate, command display before execution, configurable auto-approve for yolo mode
- [ ] 09-05: Security audit logging — security-tagged events in events.jsonl, audit log completeness verification
- [ ] 09-06: Go conventions config — conventions enforcement toggles, tool path config, coverage target config (CONF-04)
- [ ] 09-07: Safety and logging config — file write permissions config, command execution restrictions config, log level/format/path config (CONF-05, CONF-06)

## Progress

**Execution Order:**
Phases execute in numeric order: 1 → 2 → 3 → 4 → 5 → 6 → 7 → 8 → 9

| Phase | Plans Complete | Status | Completed |
|-------|----------------|--------|-----------|
| 1. Foundation | 0/8 | Not started | - |
| 2. Agent Runtime | 0/7 | Not started | - |
| 3. Initialization | 0/4 | Not started | - |
| 4. Orchestration and Core CLI | 0/8 | Not started | - |
| 5. Verification Gates | 0/7 | Not started | - |
| 6. Go Intelligence | 0/7 | Not started | - |
| 7. Extended Providers and Parallel Execution | 0/7 | Not started | - |
| 8. Full Agent Set, Git, Costs, Streaming CLI | 0/8 | Not started | - |
| 9. Security and Operational Hardening | 0/7 | Not started | - |
