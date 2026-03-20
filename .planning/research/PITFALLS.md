# Pitfalls Research

**Domain:** Multi-agent LLM orchestration platform, provider abstraction, Go CLI development tool
**Researched:** 2026-03-20
**Confidence:** HIGH (primary claims verified against multiple sources including academic research, official docs, and post-mortems)

---

## Critical Pitfalls

### Pitfall 1: Treating Provider Abstraction as Purely Syntactic

**What goes wrong:**
You build a unified Provider interface that maps OpenAI, Anthropic, and Google to a common schema. It works in happy-path tests. In production, tool-calling fails intermittently, streaming finish-reasons behave differently across providers, and identical prompts produce divergent outputs. Models below reliability thresholds break entire agent graphs rather than degrading gracefully.

**Why it happens:**
Adapter patterns solve the syntactic translation problem (tool call schema, message format, streaming event structure) but not the semantic problem. OpenAI expects tool calls in a `tools` array; Anthropic uses `tool_use` content blocks; Google uses `function_declarations`. After normalization, tool-call enforcement (`tool_choice`), parallel tool-call execution, and streaming finish-reason semantics remain provider-specific. Behavioral differences across model families compound this — the same instruction produces different behavior from claude-3-5-sonnet vs gpt-4o.

**How to avoid:**
Design three separate layers: (1) syntactic adapter per provider, (2) behavioral contract tests that run against every provider asserting output structure, not just API success, and (3) a routing layer that encodes known behavioral constraints per provider. Never assume a prompt optimized for one model family is portable. Budget for per-provider prompt tuning from the start. Accept that the interface leaks at edge cases — document where it leaks rather than hiding it.

**Warning signs:**
- Agent steps succeed in tests against one provider but fail against another
- Streaming response handling has a large conditional block per provider
- A provider upgrade silently changes tool-call behavior without tests catching it
- Provider interface methods return `interface{}` or untyped maps to handle divergent response shapes

**Phase to address:**
Phase 1 (Provider Layer). Build behavioral contract tests alongside the interface definition. Do not defer until integration testing.

---

### Pitfall 2: Coordination Failures from Underspecified Agent Contracts

**What goes wrong:**
Research shows 41-86.7% of multi-agent systems fail in production. Coordination failures account for 36.94% of all failures: agents duplicate work, talk past each other, or forget their own responsibilities. One misinterpreted handoff message early in a pipeline cascades through subsequent agents, producing confident but wrong final output.

**Why it happens:**
Agent contracts that describe what an agent does but not what constitutes a valid handoff, what the receiving agent expects, or what to do on partial failure are insufficient. In multi-step pipelines, locally reasonable decisions are globally incoherent when agents cannot see the full dependency graph. 79% of failures originate from specification and coordination issues, not technical implementation.

**How to avoid:**
Every agent definition must include: explicit input schema (what it receives), explicit output schema (what it emits), success criteria (how to validate output before handoff), and failure behavior (what the calling agent should do if output is invalid). Treat agent contracts as typed interfaces, not prose descriptions. Validate at every handoff, not just at the end. Build a dependency graph that is loaded and validated at startup — agents must not make implicit assumptions about execution order.

**Warning signs:**
- Agent definitions describe behavior in prose without structured input/output schemas
- Handoff validation only happens at the final output step
- Agent failures surface as unexpected output rather than explicit errors
- Adding a new agent requires reading multiple other agents' code to understand what they emit

**Phase to address:**
Phase 2 (Agent Runtime). Define and validate contracts before implementing agent logic.

---

### Pitfall 3: Context Window Exhaustion Causing Silent Failures

**What goes wrong:**
A 50-step workflow with 20K tokens per call consumes 1 million tokens total. Context windows fill silently — earlier context is dropped without error, agents continue executing with incomplete information, and output looks plausible but is wrong. This does not surface as a "too many tokens" error; it surfaces as confusing product behavior that is hard to reproduce.

**Why it happens:**
Developers assume more context equals better results and stuff code files, history, and accumulated state into every prompt. Models do not use context uniformly — reliability degrades as input length grows. Quality degradation is not linear; it is non-linear and unpredictable. Vector embeddings used naively flatten code structure into undifferentiated chunks, destroying dependency relationships that agents need for multi-hop reasoning.

**How to avoid:**
Treat context as a finite budget, not a free resource. Design every agent to request the minimum context necessary for its task. Use structured tool calls to fetch specific facts rather than pre-loading large documents. Track token usage per agent call and emit a warning when an agent approaches 70% of its context budget. For code intelligence, preserve structural relationships (import graphs, call graphs) as metadata rather than raw file content. Test each agent at various context fill levels, not just at low fill.

**Warning signs:**
- Agents receive full file contents when they need specific functions or sections
- No per-call token counting at the agent layer
- Agents that "should remember" earlier steps produce inconsistent output in long runs
- Context assembly logic is a single function that appends everything available

**Phase to address:**
Phase 3 (Agent Context Assembly). Context budgeting must be designed before agents are built, not retrofitted.

---

### Pitfall 4: File-Based State Without Durability Guarantees

**What goes wrong:**
STATE.md, checkpoint files, and JSONL event logs are written, but when a process crashes mid-write, state is corrupted. Two concurrent agent executions resume from the same checkpoint simultaneously without a lock mechanism. After a crash, the system cannot distinguish a running workflow from an abandoned one. Coarse checkpointing means parallel agents that partially succeeded all re-execute on resume.

**Why it happens:**
Checkpointing (saving state) is mistaken for durable execution (guaranteeing completion). Most agent frameworks save state but leave failure detection, automatic recovery, and duplicate prevention to the application. Atomic file writes (temp file → rename) solve the write-corruption problem on POSIX but not the concurrency problem, and are not atomic on Windows. JSONL appends are not covered by the temp/rename pattern — appends require different protection (file locks or single-writer goroutine).

**How to avoid:**
Use atomic temp-file-then-rename for all full file writes (STATE.md, config, checkpoint files). For JSONL event logs, use a single-writer goroutine with a buffered channel — never write from multiple goroutines. Add a lock file or PID file for in-progress workflow runs so a new process can detect an abandoned run. Checkpoint at activity-level granularity, not superstep-level, so partial completions are not re-executed on resume. On Windows, test atomic writes explicitly — os.Rename is not atomic on Windows.

**Warning signs:**
- Multiple goroutines write to the same JSONL file without synchronization
- The system has no mechanism to detect a crashed-but-not-cleaned-up run
- Checkpoint resume re-runs steps that already wrote their outputs
- No integration test that kills the process mid-write and verifies state integrity on restart

**Phase to address:**
Phase 1 (State Persistence Layer). Durability contracts must be established before any agent writes state.

---

### Pitfall 5: Prompt Drift and Model Version Fragility

**What goes wrong:**
A provider silently upgrades the model behind a version alias (e.g., `gpt-4o` routes to a different snapshot). System prompts that were tuned to produce structured JSON output begin returning malformed JSON at a 40% rate. Agent pipelines built on strict output parsing break without any code change. Rollback is slow because no prompt versioning exists.

**Why it happens:**
Prompts are treated as configuration strings, not versioned artifacts with regression tests. LLM outputs are non-deterministic — the same prompt produces different outputs across model versions, even "minor" updates. In agentic systems, drift at any step spreads downstream. Teams edit prompts in response to feedback without running behavioral tests, treating it as safe because "it's just text."

**How to avoid:**
Version every system prompt alongside the code that consumes it. Build output shape tests for every structured-output prompt (not just "did it respond" but "does the response match the schema"). Use explicit model version pinning (e.g., `claude-3-5-sonnet-20241022`, not `claude-3-5-sonnet-latest`) in production. Test prompt changes against a golden output set before deploying. Log every LLM response with its prompt version and model version for post-hoc debugging.

**Warning signs:**
- System prompts are stored as string literals in Go source files without version identifiers
- Structured output parsing has no fallback or recovery for malformed responses
- Model aliases (latest, turbo) are used in production instead of pinned versions
- There are no regression tests that assert output schema compliance

**Phase to address:**
Phase 2 (Agent Runtime) and Phase 4 (Verification Gates). Prompt versioning in Phase 2; output schema validation in Phase 4.

---

### Pitfall 6: Cost Explosion from Unguarded Agent Loops

**What goes wrong:**
An agent enters a retry loop because its success criteria are not met. Each retry costs tokens. With 12+ agents, parallel execution, and no per-agent cost cap, a single bad run can consume the daily budget in minutes. A routing bug accidentally sends all tasks to the most expensive model. Cost tracking added after the fact misses early runs and provides no real-time budget enforcement.

**Why it happens:**
Cost tracking is treated as an observability concern rather than a runtime constraint. Agent retry logic has no maximum attempt count tied to cost thresholds. Rule-based routing is correct in nominal cases but has no fallback behavior that is validated against cost — a routing rule mismatch silently defaults to the most capable (most expensive) model.

**How to avoid:**
Design cost accounting into the agent execution protocol from day one, not as a post-hoc feature. Every agent call must record input tokens, output tokens, model ID, and cost estimate before the call completes. Implement hard per-run budget caps that halt execution and write a checkpoint before stopping. Retry limits must be enforced at the agent level, not assumed to be handled by provider SDKs. Validate routing rules in tests to ensure no task type falls through to an unintended model tier.

**Warning signs:**
- Token counting is added as a logging concern after agent execution logic is complete
- Retry logic has no maximum attempt count or cost-aware backoff
- Routing rules are tested for correctness but not for exhaustiveness (uncovered task types)
- No budget cap exists in the execution engine; cost is only visible in post-run reports

**Phase to address:**
Phase 1 (Provider Layer) — cost instrumentation must be part of the Provider interface, not added later.

---

### Pitfall 7: Spec-to-Code Drift in Agent-Generated Artifacts

**What goes wrong:**
GopherMind agents generate code that contradicts the spec they just wrote. Planning artifacts (SPEC.md, STATE.md) become stale as code evolves. The system passes its own verification gates because gates check for completeness, not semantic alignment with the original spec. After several sessions, the spec describes a system that no longer exists.

**Why it happens:**
LLM-generated code is non-deterministic — the same spec produces different implementations on different runs. Without bidirectional tracing (spec → code → spec), drift is invisible. Verification gates that check "does the code compile and test?" do not check "does the code match the architectural decisions in the spec?". Planning artifacts are written once and not re-validated as the codebase grows.

**How to avoid:**
Build spec-alignment checks into verification gates: after code generation, verify that key architectural decisions from the spec (package layout, interface names, patterns explicitly chosen) are present in the generated code. Maintain decision logs in STATE.md that record not just what was decided but why, so drift is detectable by reviewing against current code. Use deterministic scaffolding (package structure, interface stubs) generated from the spec before LLM-written implementation — the scaffold anchors the structure.

**Warning signs:**
- Verification gates only run `go build` and `go test`, not semantic checks
- STATE.md has not been updated after the last code generation run
- Planning artifacts use future tense ("will implement") after the implementation phase
- No test asserts that generated code implements the interfaces declared in the spec

**Phase to address:**
Phase 5 (Verification Gates). Define alignment checks as part of gate design.

---

## Technical Debt Patterns

| Shortcut | Immediate Benefit | Long-term Cost | When Acceptable |
|----------|-------------------|----------------|-----------------|
| Model alias (latest) instead of pinned version | No maintenance of version strings | Silent behavior changes break agent pipelines | Never in production |
| Single Provider struct instead of interface | Faster initial implementation | Cannot swap providers; cannot test with mock | Never — interface is the whole point |
| String-literal prompts in source | Simple, no extra files | No versioning, no regression tests, no diff tracking | MVP only if tests exist |
| Skip per-agent cost tracking, add later | Faster agent implementation | Cost explosion risk; retrofitting breaks the execution protocol | Never — instrument from day one |
| In-memory agent state, persist at end | Simpler code | All state lost on crash; no resumability | Never for a resumable system |
| Single JSONL writer without lock | Simpler code path | Log corruption under concurrent execution | Never — single-writer goroutine costs nothing |
| Viper flag binding without explicit getter | Intuitive flag access | Environment variable override silently ignored | Never — always use viper.Get* for configured values |

---

## Integration Gotchas

| Integration | Common Mistake | Correct Approach |
|-------------|----------------|------------------|
| Anthropic Messages API | Using `latest` model alias, missing streaming event types (`message_start`, `content_block_start`, `message_delta`) | Pin exact model versions; handle all SSE event types explicitly |
| OpenAI Chat Completions | Assuming usage tokens appear in every streaming chunk | Usage is only in the final chunk; accumulate across chunks |
| Google Gemini | Treating function_declarations as equivalent to OpenAI tools schema | Gemini's multimodal inputs (video, audio) are not abstracted away — handle Gemini-specific features explicitly |
| Ollama local | Assuming response structure matches OpenAI exactly | Ollama's OpenAI-compatibility mode is partial; streaming and tool-call support vary by model |
| Cobra + Viper config | Reading flag value directly from `cmd.Flag("x").Value.String()` after viper binding | Always use `viper.GetString("x")` after binding; flag defaults override env vars if read directly |
| JSONL event log | Multiple goroutines appending concurrently | Use a dedicated log-writer goroutine receiving from a buffered channel |
| Atomic file writes on Windows | Relying on os.Rename being atomic | os.Rename is not atomic on Windows; test cross-platform explicitly |
| Provider rate limits | Shared rate limiter across all models from same provider | Rate limits differ per model tier; maintain separate limiters per model |

---

## Performance Traps

| Trap | Symptoms | Prevention | When It Breaks |
|------|----------|------------|----------------|
| Naively assembling full file content into every prompt | Escalating costs, context window exhaustion, quality degradation | Fetch minimum required context per agent; use structural metadata not raw content | At 5+ files in a prompt |
| Unbound parallel agent goroutines | Simultaneous rate limit errors from provider; OS file descriptor exhaustion | Use errgroup with explicit goroutine limit; implement semaphore for LLM calls | At 3+ concurrent agents hitting the same provider |
| Accumulating all session history in every prompt | 7-second load times, 2GB memory pressure for long sessions | Summarize and compact history; use rolling window with importance scoring | At 500+ messages or 200K+ tokens of history |
| Superstep-level checkpointing with parallel agents | On resume, completed parallel work is duplicated | Checkpoint at activity level; track per-agent completion status | On any crash during parallel agent execution |
| Synchronous LLM calls on critical path during streaming output | User sees no output for 10+ seconds; perceived as hung | Use goroutines for concurrent LLM calls; stream partial results as they arrive | On any call with >2 second latency |

---

## Security Mistakes

| Mistake | Risk | Prevention |
|---------|------|------------|
| Sending full source tree to provider APIs | Source code exfiltrated to provider; violates privacy promise | Enforce explicit opt-in per-file or per-directory; default to no code transmission; log every API call with what was sent |
| API keys in config.json committed to git | Credential exposure in repository | Store keys in separate credentials file; add to .gitignore at project init; refuse to run if credentials.md is tracked by git |
| LLM-generated shell commands executed without review | Arbitrary code execution; data destruction | Require explicit human approval for any shell command an agent proposes; never auto-execute generated commands |
| Prompt injection via user-supplied project names or file content | Agent behavior manipulation; exfiltration of system prompts | Sanitize user inputs used in prompt construction; separate user content from system instructions using provider-supported roles |
| Logging full LLM responses including source code excerpts | Source code captured in logs; logs may be transmitted to monitoring services | Log response metadata (tokens, model, status) by default; log full response content only to local files with explicit configuration |

---

## UX Pitfalls

| Pitfall | User Impact | Better Approach |
|---------|-------------|-----------------|
| Silent progress during multi-minute agent runs | User kills the process thinking it hung; loses checkpoint | Stream structured progress events to stderr; show which agent is running, what it is doing, estimated tokens used |
| Generic "LLM error" messages without provider context | User cannot diagnose rate limits, quota exhaustion, or auth failures | Surface provider error codes with actionable remediation text (e.g., "Anthropic rate limit — retry in 30s, or switch to backup model") |
| `gophermind resume` with no disambiguation when multiple interrupted runs exist | Wrong run resumed silently | List interrupted runs with timestamp and last-known phase; require explicit selection |
| Cost report only available post-run | User discovers runaway cost after it has happened | Show running cost in progress output; emit a warning at configurable thresholds (e.g., 50% of budget) |
| Verification gate failures as unstructured error text | Developer cannot distinguish fixable failures from system errors | Categorize gate failures: COVERAGE (needs more tests), CONSISTENCY (spec drift), COMPLETENESS (missing implementation), ALIGNMENT (wrong architecture) |
| Cobra/Viper flag defaults silently overriding environment config | Configuration from environment is ignored without warning | Log effective configuration at startup at debug level; make config source visible in `gophermind config` output |

---

## "Looks Done But Isn't" Checklist

- [ ] **Provider interface:** Implementation handles all streaming event types for each provider — verify that Anthropic `message_start`, `content_block_start`, `content_block_delta`, `message_delta`, and `message_stop` are all handled, not just the delta events
- [ ] **Agent contracts:** Each agent has a machine-readable output schema, not just a description — verify by attempting to validate a sample output against the schema programmatically
- [ ] **Resumability:** Checkpoint restore actually recovers full execution context — verify by killing the process mid-run and checking that resume produces the same final output
- [ ] **Cost tracking:** Token counts are recorded for every provider call including failed calls — verify by searching JSONL logs for calls without cost records
- [ ] **Rate limiting:** Separate rate limiters exist per model tier, not just per provider — verify by checking limiter initialization code against provider rate limit documentation
- [ ] **Atomic writes:** All state file writes go through temp-file-rename pattern — verify by grepping for direct `os.WriteFile` calls on .planning/ paths
- [ ] **JSONL concurrency:** Event log has a single writer — verify with `-race` flag in tests that include concurrent agent execution
- [ ] **Prompt versioning:** System prompts have version identifiers that appear in JSONL logs — verify by reading log output and confirming prompt_version field is present
- [ ] **Windows compatibility:** Atomic rename behavior is tested on Windows — verify by checking test suite for OS-specific file write tests
- [ ] **Verification gates:** Gates check semantic alignment with spec, not just compilation and test pass — verify by intentionally introducing an architectural deviation and confirming the gate catches it

---

## Recovery Strategies

| Pitfall | Recovery Cost | Recovery Steps |
|---------|---------------|----------------|
| Provider abstraction semantic mismatch discovered in production | HIGH | Add per-provider behavioral test suite; audit all agent prompts against each provider; add provider-specific prompt variants where behavior diverges |
| Context window corruption (silent truncation) | MEDIUM | Add token counting middleware to all calls; replay affected runs with reduced context; add context budget enforcement before next run |
| Corrupted state file from concurrent write | MEDIUM | Restore from last valid checkpoint (requires checkpoint history); replay events from JSONL log up to corruption point; add write lock going forward |
| Prompt drift from model version change | MEDIUM | Roll back to pinned model version; run output schema tests against both versions; create per-version prompt variants if needed |
| Cost explosion from runaway agent loop | LOW (financially HIGH) | Set provider-level spending alerts immediately; add per-run budget cap; add agent retry limits; audit routing rules for unintended model selection |
| Spec-code drift discovered after multiple sessions | HIGH | Run alignment audit against each planning artifact; rebuild STATE.md from actual codebase state; treat accumulated drift as technical debt requiring a dedicated cleanup phase |
| Agent coordination failure (duplicate work, missed handoff) | MEDIUM | Replay from last checkpoint before the coordination failure; add explicit handoff validation; add contract tests for the affected agent pair |

---

## Pitfall-to-Phase Mapping

| Pitfall | Prevention Phase | Verification |
|---------|------------------|--------------|
| Provider abstraction semantic mismatch | Phase 1: Provider Layer | Behavioral contract tests pass against all configured providers |
| Underspecified agent contracts | Phase 2: Agent Runtime | All agent definitions have machine-readable input/output schemas; handoff validation rejects malformed output |
| Context window exhaustion | Phase 3: Context Assembly | Per-agent token budget enforced; warning emitted at 70% context fill |
| File-based state durability | Phase 1: State Persistence | Process-kill-and-resume test passes; JSONL log is intact after concurrent writes under `-race` |
| Prompt drift / model version fragility | Phase 2: Agent Runtime | All prompts have version IDs; output schema tests pass; model versions are pinned |
| Cost explosion | Phase 1: Provider Layer | Every provider call records cost; per-run budget cap halts execution; routing rules are exhaustiveness-tested |
| Spec-to-code drift | Phase 5: Verification Gates | Alignment gate catches architectural deviations; STATE.md is validated against codebase after each run |
| Cobra/Viper config override | Phase 1: CLI Foundation | Effective config is logged at startup; tests assert env vars take precedence over flag defaults |
| JSONL concurrent write corruption | Phase 1: State Persistence | `-race` tests pass with concurrent agent execution; log shows single-writer goroutine |
| LLM-generated command execution | Phase 2: Agent Runtime | No agent executes shell commands without emitting a HUMAN_APPROVAL_REQUIRED event |

---

## Sources

- [Why Do Multi-Agent LLM Systems Fail? — arXiv 2503.13657 (ICML 2025 Spotlight)](https://arxiv.org/abs/2503.13657): 14-category MAST taxonomy; 41-86.7% production failure rate; coordination failures account for 36.94% of all failures
- [Why Do Multi-Agent LLM Systems Fail? — Augment Code](https://www.augmentcode.com/guides/why-multi-agent-llm-systems-fail-and-how-to-fix-them): coordination and verification gap analysis
- [Still Not Durable: How Agent Frameworks Repeat the Same Mistakes — Diagrid](https://www.diagrid.io/blog/still-not-durable-how-microsoft-agent-framework-and-strands-agents-repeat-the-same-mistake): checkpointing vs durable execution; no failure detection; coarse checkpoint granularity
- [Provider-Agnostic Agents: Why Adapters Alone Aren't Enough — Florian Drechsler](https://fdrechsler.de/blog/provider-agnostic-agents): syntactic vs semantic abstraction gap; prompt portability costs; non-linear performance cliffs
- [The Context Window Problem — Factory.ai](https://factory.ai/news/context-window-problem): context exhaustion failure modes; bulk inclusion pitfall; multi-hop reasoning breakdown
- [Prompt Drift: The Hidden Failure Mode Undermining Agentic Systems — Comet](https://www.comet.com/site/blog/prompt-drift/): prompt drift definition; production failure from un-versioned prompt changes
- [How To Solve LLM Production Challenges — Deepchecks](https://deepchecks.com/llm-production-challenges-prompt-update-incidents/): prompt updates as primary source of production incidents
- [Comparing Streaming Response Structure for Different LLM APIs — Medium/Percolation Labs](https://medium.com/percolation-labs/comparing-the-streaming-response-structure-for-different-llm-apis-2b8645028b41): streaming incompatibilities between Anthropic, OpenAI, Google
- [Why Spec-Driven Development Fails — DEV Community](https://dev.to/casamia918/why-spec-driven-development-fails-and-what-we-can-learn-from-it-2pec): specification rot; AI ignoring its own spec; non-deterministic spec-to-code mapping
- [Sting of the Viper: Cobra + Viper Integration — Carolyn Van Slyck](https://carolynvanslyck.com/blog/2020/08/sting-of-the-viper/): flag default override issue; proper viper getter usage
- [Atomically Writing Files in Go — Michael Stapelberg](https://michael.stapelberg.ch/posts/2017-01-28-golang_atomically_writing/): temp-file-rename pattern; Windows non-atomicity; fsync requirement
- [Building a High-Performance LLM Client in Go — dasroot.net](https://dasroot.net/posts/2026/02/building-high-performance-llm-client-go/): concurrency model; connection pooling; error recovery
- [Common Concurrent Programming Mistakes — Go 101](https://go101.org/article/concurrent-common-mistakes.html): goroutine leak patterns; data races; channel deadlocks
- [Multi-Provider LLM Orchestration in Production — DEV Community](https://dev.to/ash_dubai/multi-provider-llm-orchestration-in-production-a-2026-guide-1g10): routing complexity; abstraction latency overhead; monitoring gaps
- [Detecting and Correcting Hallucinations in LLM-Generated Code via AST Analysis — arXiv 2601.19106](https://arxiv.org/html/2601.19106v1): Knowledge Conflicting Hallucinations; AST-based verification; 100% precision detection

---
*Pitfalls research for: GopherMind — multi-agent LLM orchestration platform in Go*
*Researched: 2026-03-20*
