# Architecture Research

**Domain:** Go CLI multi-agent LLM orchestration platform
**Researched:** 2026-03-20
**Confidence:** HIGH (corroborated by multiple production Go frameworks and community patterns)

## Standard Architecture

### System Overview

```
┌─────────────────────────────────────────────────────────────────────┐
│                          CLI Layer (cmd/)                           │
│  ┌──────────┐ ┌──────────┐ ┌────────┐ ┌────────┐ ┌─────────────┐  │
│  │  init    │ │  resume  │ │ status │ │ verify │ │  logs/costs │  │
│  └────┬─────┘ └────┬─────┘ └───┬────┘ └───┬────┘ └──────┬──────┘  │
└───────┴────────────┴───────────┴───────────┴─────────────┴─────────┘
                              │
┌─────────────────────────────▼───────────────────────────────────────┐
│                     Orchestration Layer (internal/orchestrator/)    │
│  ┌──────────────────────┐     ┌──────────────────────────────────┐  │
│  │   Session Manager    │     │    Dependency Graph Scheduler    │  │
│  │  (load/save/resume)  │     │  (DAG, parallel execution)       │  │
│  └──────────────────────┘     └──────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────┘
                              │
┌─────────────────────────────▼───────────────────────────────────────┐
│                    Agent Runtime Layer (internal/agent/)            │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐  │
│  │Architect │ │ Planner  │ │ Coder    │ │ Reviewer │ │  Tester  │  │
│  └──────────┘ └──────────┘ └──────────┘ └──────────┘ └──────────┘  │
│         [12+ specialized agents, each a definition + executor]      │
└─────────────────────────────────────────────────────────────────────┘
                              │
┌──────────────┬──────────────▼──────────────┬────────────────────────┐
│  Go Intel    │    LLM Provider Layer        │   State Layer          │
│  Layer       │   (internal/provider/)       │  (internal/state/)     │
│ (internal/   │  ┌────────┐ ┌────────────┐  │  ┌──────────────────┐  │
│  gointel/)   │  │Anthropic│ │  OpenAI    │  │  │  .planning/      │  │
│              │  └────────┘ └────────────┘  │  │  STATE.md        │  │
│ - mod detect │  ┌────────┐ ┌────────────┐  │  │  artifacts/      │  │
│ - pkg graph  │  │ Gemini │ │   Ollama   │  │  │  events.jsonl    │  │
│ - convention │  └────────┘ └────────────┘  │  └──────────────────┘  │
│   check      │  [unified Provider interface]│                        │
└──────────────┴─────────────────────────────┴────────────────────────┘
```

### Component Responsibilities

| Component | Responsibility | Typical Implementation |
|-----------|----------------|------------------------|
| CLI (cmd/) | Parse commands, validate flags, wire dependencies, call orchestrator | Cobra commands, one file per subcommand |
| Session Manager | Load/save session state, detect prior runs, enable resume | Reads/writes `.planning/STATE.md` and checkpoint files |
| Dependency Graph Scheduler | Build DAG of agent tasks, execute independent tasks in parallel | `sync.WaitGroup` + `errgroup`, topological sort |
| Agent Runtime | Execute agent protocol: load def → assemble context → select provider → execute → validate → write | Struct implementing `Agent` interface with `Run(ctx) error` |
| Agent Definition | Declarative spec: name, prompt, tools, success criteria, handoff rules | Embedded struct or YAML/JSON definition loaded at startup |
| Provider Interface | Unified `Complete(ctx, req) (resp, error)` over all LLM backends | Go interface + per-provider `struct` implementing it |
| Router | Select provider based on task type and routing rules | Config-driven rule table, returns `Provider` |
| Fallback Chain | Retry with backoff, failover to next provider on error | Wraps `Provider`, exponential backoff, error classification |
| Go Intelligence | Detect Go module, build package graph, check conventions | `go/packages` or shell-out to `go list`, AST analysis |
| State Persistence | Atomic writes of all planning artifacts, JSONL event log | `os.Rename` pattern (write temp → rename), append-only log |
| Cost Tracker | Accumulate token counts and costs per provider/agent/session | In-memory counter flushed to JSONL on each completion |
| Config | Load `config.json`, apply defaults, expose typed config struct | `encoding/json` unmarshal into config struct |

## Recommended Project Structure

```
gophermind/
├── cmd/
│   └── gophermind/
│       ├── main.go               # Entry point — wire deps, run root command
│       ├── root.go               # Root cobra command, global flags
│       ├── init.go               # `gophermind init` — questioning protocol
│       ├── resume.go             # `gophermind resume` — restore session
│       ├── status.go             # `gophermind status` — show current state
│       ├── verify.go             # `gophermind verify` — run verification gates
│       ├── logs.go               # `gophermind logs` — view event log
│       └── costs.go              # `gophermind costs` — show cost breakdown
├── internal/
│   ├── orchestrator/
│   │   ├── orchestrator.go       # Top-level coordinator, session lifecycle
│   │   ├── session.go            # Session load/save/resume logic
│   │   ├── graph.go              # DAG construction from agent dependencies
│   │   └── scheduler.go          # Parallel task execution with errgroup
│   ├── agent/
│   │   ├── agent.go              # Agent interface and base executor
│   │   ├── definition.go         # AgentDefinition struct (load from config)
│   │   ├── context.go            # Context assembly (gather inputs for prompt)
│   │   ├── validator.go          # Post-execution success criteria checking
│   │   ├── handoff.go            # Output routing to downstream agents
│   │   └── agents/               # One file per specialized agent
│   │       ├── architect.go
│   │       ├── planner.go
│   │       ├── coder.go
│   │       ├── reviewer.go
│   │       └── ...               # 12+ agents total
│   ├── provider/
│   │   ├── provider.go           # Provider interface and Request/Response types
│   │   ├── router.go             # Rule-based provider selection
│   │   ├── fallback.go           # Retry + backoff + failover chain
│   │   ├── health.go             # Provider availability monitoring
│   │   ├── anthropic/
│   │   │   └── anthropic.go      # Anthropic Messages API client
│   │   ├── openai/
│   │   │   └── openai.go         # OpenAI Chat Completions client
│   │   ├── gemini/
│   │   │   └── gemini.go         # Google Gemini client
│   │   └── ollama/
│   │       └── ollama.go         # Ollama local client
│   ├── gointel/
│   │   ├── module.go             # Go module detection and parsing
│   │   ├── packages.go           # Package graph via `go list`
│   │   └── conventions.go        # Convention checking (naming, errors, ctx)
│   ├── state/
│   │   ├── state.go              # STATE.md read/write
│   │   ├── atomic.go             # Atomic file write helper (write-temp-rename)
│   │   ├── events.go             # JSONL event log append
│   │   └── costs.go              # Cost accumulator and flush
│   └── config/
│       ├── config.go             # Config struct and defaults
│       └── load.go               # Load config.json, apply env overrides
├── config.json                   # Default configuration
├── go.mod
└── go.sum
```

### Structure Rationale

- **cmd/gophermind/:** All cobra command wiring lives here. Commands are thin — they parse flags, build dependencies, and call into `internal/`. No business logic.
- **internal/orchestrator/:** The only package that knows about the full session lifecycle. Owns the DAG and controls which agents run when.
- **internal/agent/:** Agents are composable units. The base `agent.go` defines the interface and common execution protocol. Each specialized agent in `agents/` provides its own definition (model, system prompt, tools, success criteria).
- **internal/provider/:** Pure adapter layer. `provider.go` defines the interface; each subdirectory is one backend. `router.go` and `fallback.go` are standalone — they compose providers, not extend them.
- **internal/gointel/:** Completely standalone from LLM concerns. Takes a directory path, returns Go-specific intelligence. Agents call it to assemble context.
- **internal/state/:** Owns all disk I/O for planning artifacts. Nothing else writes `.planning/` directly — all writes funnel through here, ensuring atomic guarantees.
- **internal/config/:** Loaded once at startup, passed down via dependency injection. No global config singleton.

## Architectural Patterns

### Pattern 1: Provider Interface with Adapter per Backend

**What:** Define a narrow `Provider` interface (`Complete`, `Stream`, `Health`) implemented by each LLM backend as a standalone struct. The rest of the system only sees the interface.

**When to use:** Always — this is the foundation of multi-provider support.

**Trade-offs:** Small additional indirection; massive benefit of swappable backends and testability via mock providers.

**Example:**
```go
// internal/provider/provider.go
type Request struct {
    SystemPrompt string
    Messages     []Message
    MaxTokens    int
    Temperature  float64
}

type Response struct {
    Content      string
    InputTokens  int
    OutputTokens int
    Model        string
}

type Provider interface {
    Complete(ctx context.Context, req Request) (Response, error)
    Name() string
    Health(ctx context.Context) error
}
```

### Pattern 2: Rule-Based Router with Fallback Chain

**What:** A `Router` selects a `Provider` by matching task type against a rule table from config. A `FallbackChain` wraps any `Provider` to add retry-with-backoff and secondary provider failover.

**When to use:** When you need predictable routing without dynamic/ML-based selection — rules are auditable and debuggable.

**Trade-offs:** Rigid rules require explicit maintenance when providers change; significantly simpler than dynamic routing.

**Example:**
```go
// internal/provider/router.go
type RoutingRule struct {
    TaskType string   // "architecture", "code_gen", "summarization"
    Primary  string   // provider name from config
    Fallback []string // ordered fallback chain
}

func (r *Router) Select(taskType string) Provider {
    rule := r.rules[taskType] // lookup by task type
    return NewFallbackChain(r.providers[rule.Primary], rule.Fallback...)
}
```

### Pattern 3: Agent Execution Protocol as Explicit Steps

**What:** Each agent follows the same five-step protocol: (1) load definition, (2) assemble context, (3) select provider via router, (4) execute + stream, (5) validate output against success criteria, (6) write artifacts, (7) signal handoff. Each step is a separate function with a defined input/output.

**When to use:** All agent execution — makes the flow inspectable, testable per-step, and resumable after interruption.

**Trade-offs:** More verbose than a single "run" function; necessary for auditability and debugging.

**Example:**
```go
// internal/agent/agent.go
type Agent interface {
    Definition() AgentDefinition
    Run(ctx context.Context, input AgentInput) (AgentOutput, error)
}

// base executor runs the protocol
func Execute(ctx context.Context, a Agent, input AgentInput, router *Router, state *State) error {
    def  := a.Definition()
    assembled := assembleContext(ctx, def, input, state)
    provider  := router.Select(def.TaskType)
    resp, err := provider.Complete(ctx, buildRequest(def, assembled))
    if err != nil { return err }
    if err := validate(def.SuccessCriteria, resp); err != nil { return err }
    return state.WriteArtifact(def.OutputPath, resp.Content)
}
```

### Pattern 4: DAG Scheduler with errgroup for Parallel Execution

**What:** Build a directed acyclic graph (DAG) from agent dependency declarations. Use `golang.org/x/sync/errgroup` to run independent agents concurrently; block each agent until its declared dependencies have completed.

**When to use:** Multi-agent phases where some agents are independent (e.g., multiple file generators that can run in parallel after planning).

**Trade-offs:** Complexity of dependency tracking; significant throughput improvement on multi-agent phases.

**Example:**
```go
// internal/orchestrator/scheduler.go
func (s *Scheduler) Run(ctx context.Context, tasks []Task) error {
    g, ctx := errgroup.WithContext(ctx)
    done := make(map[string]chan struct{})
    for _, t := range tasks {
        done[t.ID] = make(chan struct{})
    }
    for _, task := range tasks {
        t := task // capture
        g.Go(func() error {
            for _, dep := range t.DependsOn {
                <-done[dep] // wait for dependency
            }
            err := s.execute(ctx, t)
            close(done[t.ID]) // signal completion
            return err
        })
    }
    return g.Wait()
}
```

### Pattern 5: Atomic File Writes for Artifact Safety

**What:** All planning artifact writes go through a helper that writes to a temp file alongside the target, then calls `os.Rename` (atomic on POSIX). Partial writes never corrupt existing files.

**When to use:** Every write to `.planning/` — this is non-negotiable for a resumable system.

**Trade-offs:** Slight overhead; zero risk of corrupted state files.

**Example:**
```go
// internal/state/atomic.go
func WriteAtomic(path string, data []byte) error {
    dir  := filepath.Dir(path)
    tmp, err := os.CreateTemp(dir, ".tmp-*")
    if err != nil { return err }
    if _, err = tmp.Write(data); err != nil {
        os.Remove(tmp.Name())
        return err
    }
    tmp.Close()
    return os.Rename(tmp.Name(), path) // atomic on POSIX
}
```

## Data Flow

### Command Execution Flow

```
User: gophermind resume
        │
        ▼
cmd/resume.go          Parse flags, load config, create Orchestrator
        │
        ▼
orchestrator.Resume()  Read STATE.md → determine incomplete tasks
        │
        ▼
scheduler.Run()        Build DAG → launch goroutines per task wave
        │         ┌─────────────────────────────────────────────┐
        ▼         │ For each agent task (possibly concurrent):  │
agent.Execute()   │   1. assembleContext() ← state + gointel   │
        │         │   2. router.Select(taskType) → Provider     │
        │         │   3. provider.Complete(ctx, req)            │
        │         │   4. validate(successCriteria, response)    │
        │         │   5. state.WriteArtifact(path, content)     │
        │         │   6. state.AppendEvent(event)               │
        │         └─────────────────────────────────────────────┘
        │
        ▼
orchestrator       All tasks done → update STATE.md phase/status
        │
        ▼
cmd/resume.go      Print summary to stdout
```

### Provider Call Flow

```
agent.Execute()
        │
        ▼
router.Select(taskType)        Look up routing rule → primary provider name
        │
        ▼
FallbackChain.Complete()       Attempt primary provider
        │                           ├── Success → return Response
        │                           └── Error → classify error
        │                               ├── Transient (429, 503) → exponential backoff retry
        │                               └── Persistent → next provider in fallback list
        ▼
state.AppendEvent(ProviderCallEvent)    Log: provider, tokens, cost, latency, outcome
        │
        ▼
costs.Record(provider, tokens)         Accumulate cost counters for session
```

### State Write Flow

```
Agent produces artifact content
        │
        ▼
state.WriteArtifact(relativePath, content)
        │
        ▼
atomic.WriteAtomic(.planning/<path>)   Write temp → rename (atomic)
        │
        ▼
state.AppendEvent(ArtifactWrittenEvent)    Append to events.jsonl
        │
        ▼
state.UpdateSTATE()                    Rewrite STATE.md with current phase/tasks/risks
```

### Key Data Flows

1. **Init flow:** Wave-based questioning (8 waves) → answers assembled into project spec → spec written to `.planning/` → STATE.md initialized → orchestrator begins phase 1.
2. **Resume flow:** STATE.md parsed → incomplete tasks identified → DAG rebuilt from remaining tasks → scheduler resumes from first incomplete wave.
3. **Context assembly flow:** Agent definition declares which state files and Go intel it needs → `assembleContext()` reads those files and runs gointel queries → assembled context feeds into prompt construction.
4. **Cost tracking flow:** Every `provider.Complete()` call records tokens in the response → `costs.Record()` accumulates per-provider/per-agent → `gophermind costs` command reads the JSONL event log and aggregates.

## Component Build Order

The dependency graph between components determines the order in which they should be built:

```
1. internal/config        ← no internal deps
2. internal/state         ← depends on: config
3. internal/provider      ← depends on: config
4. internal/gointel       ← depends on: config
5. internal/agent         ← depends on: config, state, provider, gointel
6. internal/orchestrator  ← depends on: agent, state, config
7. cmd/gophermind         ← depends on: orchestrator, config
```

**Phase implications:**
- Phase 1 (Foundation): `config`, `state`, provider interface + one real provider (Anthropic)
- Phase 2 (Agent Runtime): `agent` base executor, 2-3 core agents (architect, planner, coder)
- Phase 3 (Orchestration): `orchestrator`, session, DAG scheduler
- Phase 4 (CLI): All `cmd/` commands wired to orchestrator
- Phase 5 (Go Intelligence): `gointel` integrated into agent context assembly
- Phase 6 (Full Provider Coverage): OpenAI, Gemini, Ollama, router, fallback chain
- Phase 7 (Remaining Agents): All 12+ specialized agents
- Phase 8 (Verification): Gates, cost tracking, health checks

## Scaling Considerations

This is a single-user local CLI tool. Scaling in the traditional sense does not apply. The relevant scaling concerns are:

| Scale | Architecture Adjustments |
|-------|--------------------------|
| Small project (< 20 files) | Single-phase execution, sequential agents acceptable |
| Medium project (20-200 files) | DAG parallelism needed; context window management becomes important |
| Large project (200+ files) | Chunked analysis passes; package-level rather than file-level context; streaming required |

### Scaling Priorities

1. **First bottleneck:** LLM context window limits — large codebases cannot be passed whole to an agent. Fix: `gointel` package graph enables selective context loading (only relevant packages).
2. **Second bottleneck:** Sequential agent execution latency — a 12-agent pipeline with 30s/agent is 6 minutes. Fix: DAG scheduler with parallel execution reduces wall time to the critical path length.

## Anti-Patterns

### Anti-Pattern 1: Global Provider or Config Singleton

**What people do:** Declare `var globalProvider provider.Provider` at package level and access it directly from agents.

**Why it's wrong:** Untestable (cannot inject mock), creates hidden coupling between packages, causes race conditions under parallel execution.

**Do this instead:** Pass `Provider` and `Config` through function arguments or a dependency struct. The orchestrator owns the wired graph; agents receive what they need.

### Anti-Pattern 2: Agents Calling State Directly

**What people do:** Have individual agent implementations write to `.planning/` themselves via `os.WriteFile`.

**Why it's wrong:** Bypasses atomic write guarantee, creates multiple code paths for writes, makes it impossible to enforce event logging consistently.

**Do this instead:** All disk writes flow through `internal/state`. Agents return their output; the executor (in `agent/agent.go`) calls `state.WriteArtifact`. Agents are pure functions of context → output.

### Anti-Pattern 3: Monolithic Agent with Branching Prompts

**What people do:** Build one large "do everything" agent that uses if/else prompt construction to handle architecture, coding, and review tasks.

**Why it's wrong:** Prompt quality degrades when a single agent tries to do too many things. Success criteria become impossible to validate. Hard to route to the right model.

**Do this instead:** 12+ specialized agents with narrow contracts. Each agent has one responsibility, one success criterion, and one model selection. Orchestrator composes them.

### Anti-Pattern 4: Chatty State Files (One File per Token)

**What people do:** Write a separate small file for every agent output, creating hundreds of tiny files in `.planning/`.

**Why it's wrong:** Hard to scan, hard to diff in git, creates `os.Open` overhead in context assembly.

**Do this instead:** Organize artifacts by phase and concern. Phase-level summary files plus per-agent outputs within phase directories. `STATE.md` is the single source of truth for current status.

### Anti-Pattern 5: Blocking on Provider Calls Without Context Cancellation

**What people do:** Call `provider.Complete(...)` without threading the caller's `context.Context` through, or without checking `ctx.Done()` during retries.

**Why it's wrong:** A cancelled CLI invocation (Ctrl-C) will not abort in-flight LLM calls. Retry loops become unkillable. Costs accumulate for abandoned sessions.

**Do this instead:** All provider methods accept `context.Context` as their first argument. Retry loops check `ctx.Err()` before each attempt. The CLI's root command sets a cancellable context wired to `os.Signal`.

## Integration Points

### External Services

| Service | Integration Pattern | Notes |
|---------|---------------------|-------|
| Anthropic API | HTTP client, `POST /v1/messages`, streaming SSE | API key from config; respect rate limits via backoff |
| OpenAI API | HTTP client, `POST /v1/chat/completions`, streaming | Same structure as Anthropic but different schema |
| Google Gemini | Official `google.golang.org/genai` SDK or HTTP | SDK preferred to handle auth and streaming |
| Ollama | HTTP client, `POST /api/chat`, local only | No API key; health check via `/api/tags` |

### Internal Boundaries

| Boundary | Communication | Notes |
|----------|---------------|-------|
| cmd → orchestrator | Direct function call (dependency injection) | Commands hold an `*Orchestrator` reference |
| orchestrator → agent | Interface call via scheduler | `scheduler.Run()` calls `agent.Execute()` per task |
| agent → provider | Interface call via router | `router.Select(taskType)` returns a `Provider` |
| agent → state | Function call via state package | `state.WriteArtifact()`, `state.AppendEvent()` |
| agent → gointel | Function call, returns structured data | `gointel.PackageGraph(dir)` returns typed result |
| provider → external | HTTP over TLS | Each provider client owns its own `http.Client` with timeout |

## Sources

- [CloudWego Eino — Go LLM framework architecture](https://github.com/cloudwego/eino)
- [GoAI — Clean multi-provider LLM client for Go](https://dev.to/dariubs/goai-a-clean-multi-provider-llm-client-for-go-27o5)
- [golang-standards/project-layout — Standard Go Project Layout](https://github.com/golang-standards/project-layout)
- [Go Modules official layout docs](https://go.dev/doc/modules/layout)
- [AI Workflow Patterns in Go: CLI Tools to Agents (Feb 2026)](https://dasroot.net/posts/2026/02/ai-workflow-patterns-go-cli-tools-agents/)
- [Go Microservices for AI/ML Orchestration Patterns](https://www.glukhov.org/app-architecture/integration-patterns/go-microservices-for-ai-ml-orchestration-patterns/)
- [Deep Dive into WaitGroup — Concurrent Task Orchestration in Go](https://dev.to/jones_charles_ad50858dbc0/deep-dive-into-waitgroup-mastering-concurrent-task-orchestration-in-go-ao)
- [Dependency Graphs and Orchestration in AI Agent Frameworks](https://www.gocodeo.com/post/dependency-graphs-orchestration-and-control-flows-in-ai-agent-frameworks)
- [Retries, fallbacks, and circuit breakers in LLM apps](https://portkey.ai/blog/retries-fallbacks-and-circuit-breakers-in-llm-apps/)
- [Atomically writing files in Go — Michael Stapelberg](https://michael.stapelberg.ch/posts/2017-01-28-golang_atomically_writing/)
- [Building a Production-Grade LLM Client in Go](https://www.ksred.com/building-a-production-ready-go-package-for-llm-integration/)
- [Spf13/cobra — CLI framework](https://github.com/spf13/cobra)
- [Concurrency in Go: Foundations, Patterns, and Agentic Architectures](https://aminmsv01.medium.com/concurrency-in-go-foundations-patterns-and-agentic-architectures-ee69ac212df9)
- [Building a Multi-Model LLM Gateway in Go (Mar 2026)](https://dasroot.net/posts/2026/03/building-multi-model-llm-gateway-go/)

---
*Architecture research for: Go CLI multi-agent LLM orchestration platform (GopherMind)*
*Researched: 2026-03-20*
