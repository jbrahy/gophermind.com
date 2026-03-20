# Feature Research

**Domain:** AI-powered spec-driven Go development platform / LLM orchestration CLI
**Researched:** 2026-03-20
**Confidence:** HIGH (competitive landscape well-documented; GopherMind differentiators validated against Kiro, OpenHands, Aider, Claude Code)

---

## Feature Landscape

### Table Stakes (Users Expect These)

Features users assume exist. Missing these = product feels incomplete.

| Feature | Why Expected | Complexity | Notes |
|---------|--------------|------------|-------|
| Multi-file codebase understanding | Every serious AI coding tool (Aider, Cursor, OpenHands) builds a repo map; users expect the agent to understand the whole project, not just one file | MEDIUM | Go-specific: parse `go.mod`, import graph, package layout |
| Streaming output to terminal | All CLI tools stream LLM output; waiting silently for results feels broken | LOW | Use `io.Writer` streaming from provider; avoid buffering entire responses |
| Git-aware operations | Aider auto-commits with descriptive messages; users expect AI changes to be committed and attributable | MEDIUM | Wrap `git` via `os/exec`; stage + commit after each agent write |
| Multi-provider LLM support | Aider, Cursor, Kiro all support multiple models; users expect to use their preferred provider or switch for cost | HIGH | Anthropic, OpenAI required for v1; Google/Ollama stretch goals |
| Cost tracking per session | API costs are real; every serious tool either shows usage or is expected to | MEDIUM | Token accounting per provider + per agent call; session totals |
| Project initialization flow | Every tool has an `init` or onboarding command; users expect guided setup, not manual config editing | MEDIUM | 8-wave questioning protocol already planned; this is the expected entry point |
| Resumable sessions | Claude Code Tasks, LangGraph checkpointing, and OpenHands all support resumability; losing context mid-session is unacceptable | HIGH | File-based state in `.planning/` gives this "for free" if maintained rigorously |
| Structured output artifacts | Kiro produces `requirements.md`, `design.md`, `tasks.md`; OpenHands produces `PLAN.md`; users expect readable, persistent artifacts — not ephemeral chat | MEDIUM | Planning directory conventions already defined in PROJECT.md |
| Error recovery and retry | Providers fail; users expect transparent retry with backoff, not silent hangs | MEDIUM | Exponential backoff + provider fallback chains already in requirements |
| Status and progress visibility | `status` command expected by any CLI tool managing long-running workflows | LOW | Read from STATE.md; display current phase, tasks, open risks |

---

### Differentiators (Competitive Advantage)

Features that set the product apart. Not required by the market, but high value for GopherMind's target user.

| Feature | Value Proposition | Complexity | Notes |
|---------|-------------------|------------|-------|
| Spec-first lifecycle management | Kiro introduced spec-driven development but is IDE-only and AWS-centric; GopherMind owns this as a pure CLI with no IDE dependency — the only spec-driven CLI tool targeting Go devs | HIGH | Three-phase flow: spec generation → technical plan → phased implementation; each phase produces durable artifacts |
| Specialized agent contracts with explicit success criteria | All current tools use a single general-purpose agent or chat; GopherMind's 12+ specialized agents with defined contracts and handoff rules prevent vague outputs — a direct response to "AI that produces garbage you can't verify" | HIGH | Each agent has: input preconditions, output postconditions, success criteria, failure modes; orchestrator validates handoffs |
| Rule-based task routing (right model for right job) | Most tools use one model for everything; routing architecture vs. summarization vs. code-gen jobs to appropriately-priced models achieves 30-70% cost reduction (industry data) without sacrificing quality | HIGH | Routing table in config.json; architecture tasks → Claude Opus; summarization → Haiku/Flash; code gen → Sonnet |
| Go intelligence layer | No current tool deeply understands Go idioms: `go.mod` parsing, package graph analysis, convention enforcement (error wrapping, context propagation, structured logging patterns) | HIGH | `go/ast`, `go/packages`, `golang.org/x/tools/go/analysis` as foundations; internal knowledge of Go project layouts |
| Verification gates between phases | No competitor enforces quality gates before advancing phases; GopherMind blocks phase transitions on coverage checks, consistency checks, alignment checks — prevents "half-built" codebases | HIGH | Gate config per phase; coverage threshold, interface completeness, spec alignment checks |
| Auditable JSONL event log | Every provider call, every token spent, every agent decision logged in structured JSONL; users can query history, debug failures, replay events — no competitor offers this level of auditability | MEDIUM | Events, costs, and provider calls as separate JSONL streams; parseable by `jq` or any log tooling |
| Parallel agent execution with dependency graph | OpenHands and Claude Code show parallel agents are valuable but ad-hoc; GopherMind makes the dependency graph explicit and coordinates parallel execution deterministically | HIGH | DAG-based task graph; independent agents run in goroutines; dependent agents block on channel signals |
| Checkpoint + atomic file writes | Session crashes are common during long agentic runs; atomic writes (temp → rename) + explicit checkpoint markers mean zero partial-write corruption | MEDIUM | `os.Rename` atomicity on POSIX; checkpoint markers in STATE.md |
| Wave-based elicitation protocol | Generic tools ask vague "what do you want to build?" questions; 8-wave progressive questioning produces complete, unambiguous specs before a single line of code is written | MEDIUM | Wave progression: domain → constraints → architecture → data → integrations → quality → deployment → review |
| Provider health monitoring | Tools fail silently when providers are degraded; GopherMind monitors provider availability and reroutes proactively — not reactively | MEDIUM | Lightweight health check goroutine; circuit breaker pattern per provider |

---

### Anti-Features (Commonly Requested, Often Problematic)

Features that seem good but create problems. Build these and regret it.

| Feature | Why Requested | Why Problematic | Alternative |
|---------|---------------|-----------------|-------------|
| Web UI / dashboard | "Easier to visualize project state and interact with" | Doubles scope, breaks CLI-first philosophy, requires frontend stack, authentication, sessions — a multi-month detour from v1 | Rich terminal output (`lipgloss`, `bubbletea`) achieves 80% of the visual value at 5% of the cost; defer web UI to post-v1 |
| Real-time collaboration / multi-user | "Teams want to share AI sessions" | Concurrent writes to `.planning/` need locking, conflict resolution, presence awareness — a distributed systems problem disguised as a feature | Single-user v1; git-based collaboration (`.planning/` is committed) gives async teamwork without complexity |
| Chat / conversational interface | "I want to just chat with the AI about my code" | Chat is Cursor/Copilot's game; GopherMind wins on structured, verifiable outputs — not chat. Adding chat trains users to expect vague responses and undermines the spec-driven value prop | `ask` mode for one-off questions is fine; full conversational loop is an anti-pattern for this product |
| Plugin / extension API | "Let the community build integrations" | Premature extensibility is the root of all evil in tooling; public APIs become permanent contracts before the internals are stable | Internal architecture designed for extension; expose plugin API only after v1 proves out the contracts |
| LLM fine-tuning / model hosting | "Host our own tuned model for Go" | Operating ML infrastructure is a separate business; distraction from the dev tool | Stay provider-agnostic; benefit from frontier model improvements for free |
| Cloud sync / SaaS mode | "Sync planning state across machines" | Requires auth, storage, encryption-at-rest, billing — a product pivot, not a feature | Git push `.planning/`; it's already a directory. Users already know how to sync git repos. |
| Auto-approve all agent actions | "I want full automation with no confirmations" | Silent automation destroys trust when wrong; in code generation context this produces unreviewed commits and cascading mistakes | Default to confirmation gates at phase transitions; allow `--non-interactive` flag for CI/scripted use only |
| IDE integration (LSP, extension) | "Make it work inside VS Code / Cursor" | Deep IDE integration requires per-IDE SDKs, extension lifecycle management, different UX paradigm — all orthogonal to CLI | CLI works inside any terminal including those inside IDEs; users invoke it from the integrated terminal |

---

## Feature Dependencies

```
[Project Initialization / Wave-Based Elicitation]
    └──produces──> [Spec Artifacts (.planning/)]
                       └──requires──> [Agent Execution Protocol]
                                          └──requires──> [Multi-Provider LLM Support]
                                          └──requires──> [Rule-Based Task Routing]
                                          └──requires──> [Specialized Agent Contracts]

[Parallel Agent Execution]
    └──requires──> [Dependency Graph Coordination]
    └──requires──> [File-Based State Persistence]

[Verification Gates]
    └──requires──> [Go Intelligence Layer]
    └──requires──> [Agent Execution Protocol]
    └──requires──> [File-Based State Persistence]

[Resumability / Checkpoint]
    └──requires──> [File-Based State Persistence]
    └──requires──> [Atomic File Writes]

[Cost Tracking]
    └──requires──> [Multi-Provider LLM Support]
    └──enhances──> [Rule-Based Task Routing]  (cost data informs routing decisions)

[Provider Health Monitoring]
    └──enhances──> [Multi-Provider LLM Support]
    └──requires──> [Fallback Chains]

[Auditable JSONL Event Log]
    └──requires──> [Agent Execution Protocol]
    └──requires──> [Cost Tracking]

[Go Intelligence Layer]
    └──enhances──> [Specialized Agent Contracts]  (Go-aware agents produce better outputs)
    └──enhances──> [Verification Gates]  (Go-specific checks: coverage, conventions)
```

### Dependency Notes

- **Spec Artifacts require Wave-Based Elicitation:** The 8-wave questioning protocol produces the planning artifacts that all downstream agents consume. This is the root of the dependency tree — nothing meaningful runs before `gophermind init` completes.
- **Agent Execution Protocol requires Multi-Provider LLM Support:** Agents cannot execute without a functioning provider layer. Provider layer must be built and tested before any agent logic.
- **Verification Gates require Go Intelligence:** Gates that check Go conventions (error handling, context propagation, package layout) need the Go intelligence layer to be operational first. Generic coverage checks can run earlier.
- **Cost Tracking enhances Task Routing:** Once cost data is collected per agent call, the routing rules can be validated and tuned. Routing can launch with static rules; cost data makes it smarter over time.
- **Parallel Execution requires Dependency Graph:** Goroutines without an explicit graph lead to race conditions. Build the graph coordinator before enabling parallel agent runs.

---

## MVP Definition

### Launch With (v1)

Minimum viable product — what's needed to validate the spec-driven, agent-orchestrated approach.

- [ ] Multi-provider LLM support (Anthropic + OpenAI) — without providers, nothing works
- [ ] Wave-based project initialization (8 waves) — this is the spec-driven entry point; validates the core thesis
- [ ] File-based state persistence in `.planning/` — required for all resumability and agent handoffs
- [ ] Specialized agent execution protocol (load → assemble context → execute → validate → write → handoff) — the core differentiator
- [ ] At minimum 6 agents (Architect, Planner, Coder, Reviewer, Tester, Verifier) — covers the full lifecycle minimally
- [ ] Rule-based task routing (static table) — directs different jobs to appropriate models; controls costs day one
- [ ] Verification gates at phase transitions — without gates, the "verifiable" part of the value prop is missing
- [ ] Cost tracking per session — users need visibility into spend before they'll commit to the tool
- [ ] Resumable sessions via checkpoint markers — long runs will be interrupted; this must work before users trust it
- [ ] Structured JSONL event logging — audit trail is part of the core value proposition
- [ ] `init`, `status`, `resume`, `verify`, `logs`, `costs` CLI commands — minimum command set for a usable workflow
- [ ] Atomic file writes — correctness requirement; not optional

### Add After Validation (v1.x)

Features to add once core workflow is proven.

- [ ] Go intelligence layer (module graph, convention checks) — validates that Go-specific value is real; adds after first users report need for it
- [ ] Parallel agent execution with dependency graph — adds throughput; defer until sequential execution is solid and users request speed improvement
- [ ] Provider health monitoring and circuit breakers — add when provider reliability becomes a real user complaint
- [ ] Remaining 6+ specialized agents — add as workflows expand beyond core lifecycle
- [ ] Ollama / local model support — add after primary provider integrations are solid; serves privacy-first users
- [ ] Google Gemini support — add after Anthropic + OpenAI are stable; third provider has diminishing returns until first two work well

### Future Consideration (v2+)

Features to defer until product-market fit is established.

- [ ] Web UI — significant investment; validate CLI value first
- [ ] Cloud sync — requires auth and storage infrastructure; Git is sufficient for v1 users
- [ ] Plugin / extension API — stabilize internal contracts before making them public
- [ ] Non-Go repository support — Go-first focus is the competitive moat; expand only after Go market is proven
- [ ] Real-time collaboration — single-user tool is the correct v1 scope

---

## Feature Prioritization Matrix

| Feature | User Value | Implementation Cost | Priority |
|---------|------------|---------------------|----------|
| Multi-provider LLM support | HIGH | HIGH | P1 |
| Wave-based initialization | HIGH | MEDIUM | P1 |
| File-based state persistence | HIGH | LOW | P1 |
| Agent execution protocol | HIGH | HIGH | P1 |
| Verification gates | HIGH | HIGH | P1 |
| Cost tracking | HIGH | MEDIUM | P1 |
| Resumable sessions | HIGH | MEDIUM | P1 |
| JSONL event logging | HIGH | LOW | P1 |
| CLI command set | HIGH | LOW | P1 |
| Atomic file writes | HIGH | LOW | P1 |
| Rule-based task routing | HIGH | MEDIUM | P1 |
| Specialized agent contracts (6 core) | HIGH | HIGH | P1 |
| Go intelligence layer | HIGH | HIGH | P2 |
| Parallel agent execution | MEDIUM | HIGH | P2 |
| Provider health monitoring | MEDIUM | MEDIUM | P2 |
| Extended agent set (12+) | MEDIUM | HIGH | P2 |
| Ollama / local model support | MEDIUM | MEDIUM | P2 |
| Google Gemini support | LOW | MEDIUM | P2 |
| Web UI | HIGH | HIGH | P3 |
| Cloud sync | MEDIUM | HIGH | P3 |
| Plugin API | LOW | HIGH | P3 |

**Priority key:**
- P1: Must have for launch
- P2: Should have, add when possible
- P3: Nice to have, future consideration

---

## Competitor Feature Analysis

| Feature | Aider | Cursor | OpenHands | Kiro | GopherMind Approach |
|---------|-------|--------|-----------|------|---------------------|
| Spec / requirements generation | No (architect mode is conversational, not structured) | No | Planning agent (March 2026 addition) | Yes — requirements.md, design.md, tasks.md | Yes — wave-based elicitation produces structured spec artifacts |
| Multi-provider LLM | Yes (100+ via LiteLLM) | Yes (GPT-5, Claude Opus, Gemini) | Yes (any OpenAI-compatible) | Limited (Claude + Auto mix) | Yes (Anthropic + OpenAI v1; Gemini + Ollama v1.x) |
| Task routing by job type | No — single model | Partially (user selects per chat) | No | No | Yes — explicit routing table by agent role |
| Cost tracking | Basic (token counts) | No (subscription hides cost) | No | No | Yes — per agent, per provider, per session |
| Persistent planning artifacts | No — ephemeral chat | No | PLAN.md only | Yes — specs directory | Yes — full .planning/ directory with STATE.md |
| Resumable sessions | Partial (git history) | No | Partial | Partial | Yes — explicit checkpoint + STATE.md |
| Specialized agents with contracts | No — single agent | No | No | No | Yes — 12+ agents with defined contracts and handoff rules |
| Verification gates | No | No | No | No | Yes — phase gates with coverage, consistency, alignment checks |
| Go-specific intelligence | No | No | No | No | Yes — module graph, convention checks, Go idiom enforcement |
| Parallel agent execution | No | Background agents (beta) | Basic | No | Yes (v1.x) — DAG-based goroutine coordination |
| CLI-first | Yes | No (IDE) | No (web UI) | Partially (IDE + CLI) | Yes — terminal-native |
| Local / offline capable | Partial (needs API) | No | No | No | Yes (Ollama support in v1.x) |
| Auditable event log | No | No | No | No | Yes — structured JSONL events, costs, provider calls |

---

## Sources

- [Aider features overview](https://aider.chat/) — official site; HIGH confidence
- [Aider GitHub repository](https://github.com/Aider-AI/aider) — source of truth for capabilities; HIGH confidence
- [Kiro spec-driven development docs](https://kiro.dev/docs/specs/) — official Kiro docs; HIGH confidence
- [Kiro hooks documentation](https://kiro.dev/docs/hooks/) — official; HIGH confidence
- [Introducing Kiro blog post](https://kiro.dev/blog/introducing-kiro/) — canonical feature announcement; HIGH confidence
- [OpenHands product update March 2026](https://openhands.dev/blog/openhands-product-update---march-2026) — official; HIGH confidence
- [OpenHands vs SWE-Agent comparison](https://localaimaster.com/blog/openhands-vs-swe-agent) — community analysis; MEDIUM confidence
- [OpenHands Software Agent SDK paper](https://arxiv.org/html/2511.03690v1) — academic/technical; HIGH confidence
- [Spec-driven development with AI — GitHub Blog](https://github.blog/ai-and-ml/generative-ai/spec-driven-development-with-ai-get-started-with-a-new-open-source-toolkit/) — GitHub official; HIGH confidence
- [Thoughtworks on spec-driven development 2025](https://www.thoughtworks.com/en-us/insights/blog/agile-engineering-practices/spec-driven-development-unpacking-2025-new-engineering-practices) — industry analysis; MEDIUM confidence
- [AI coding agents 2026 comparison — Lushbinary](https://lushbinary.com/blog/ai-coding-agents-comparison-cursor-windsurf-claude-copilot-kiro-2026/) — multi-tool feature comparison; MEDIUM confidence
- [Multi-provider LLM cost reduction data — MindStudio](https://www.mindstudio.ai/blog/what-is-ai-model-router-optimize-cost-llm-providers) — routing ROI claims; MEDIUM confidence
- [Parallel agent patterns — Google ADK docs](https://google.github.io/adk-docs/agents/workflow-agents/parallel-agents/) — official Google ADK; HIGH confidence
- [Claude Code Tasks persistence — DEV Community](https://dev.to/simone_callegari_1f56a902/claude-code-new-tasks-persisting-between-sessions-and-swarms-of-agents-against-context-rot-5dan) — community report on Claude Code internals; MEDIUM confidence
- [Quality gates in agentic AI coding — CodeScene](https://codescene.com/blog/agentic-ai-coding-best-practice-patterns-for-speed-with-quality) — industry practice analysis; MEDIUM confidence
- [Kiro vs Claude Code comparison — Morph](https://www.morphllm.com/comparisons/kiro-vs-claude-code) — feature comparison; MEDIUM confidence

---

*Feature research for: GopherMind — spec-driven Go development platform*
*Researched: 2026-03-20*
