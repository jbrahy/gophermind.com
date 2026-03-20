# Stack Research

**Domain:** Go CLI multi-agent LLM orchestration platform
**Researched:** 2026-03-20
**Confidence:** HIGH (all core libraries verified via pkg.go.dev as of March 2026)

## Recommended Stack

### Core Technologies

| Technology | Version | Purpose | Why Recommended |
|------------|---------|---------|-----------------|
| Go | 1.23+ | Implementation language | Matches target audience, cross-platform single binary, strong stdlib concurrency primitives, explicit error handling aligns with GopherMind's auditability requirements |
| github.com/spf13/cobra | v1.10.2 | CLI command structure and flag parsing | De facto standard for Go CLIs (Kubernetes, Hugo, GitHub CLI). Provides subcommands, persistent flags, shell completions, help generation. Zero magic. |
| github.com/spf13/viper | v1.21.0 | Configuration management (config.json, env vars, flags) | Natural Cobra companion. Multi-source config with precedence chain: flags > env > config.json > defaults. Requires Go 1.23. |
| log/slog (stdlib) | Go 1.21+ | Structured JSON/JSONL logging | Zero external dependency. JSONHandler produces JSONL events. Concurrent-safe by design. Idiomatic since 1.21 — use for all internal logging. |
| golang.org/x/sync | v0.x (latest) | errgroup + weighted semaphore for parallel agent execution | Official Go extended sync library. errgroup gives context-canceling goroutine groups. Weighted semaphore bounds concurrent provider calls. The correct tool for GopherMind's dependency-graph agent coordination. |

### LLM Provider SDKs

| Library | Version | Provider | Why This One |
|---------|---------|----------|-------------|
| github.com/anthropics/anthropic-sdk-go | v1.27.1 | Anthropic / Claude | Official Anthropic SDK. v1+ stable. Streaming, tool use, batching, error types, token counting. Published Mar 18, 2026. |
| github.com/openai/openai-go/v3 | v3.29.0 | OpenAI | Official OpenAI SDK. v3 series is current. Chat completions, streaming, tool calls. Go 1.22+. Published Mar 17, 2026. |
| google.golang.org/genai | v1.51+ | Google Gemini | Official Google Gen AI SDK (GA since May 2025). Replaces deprecated generative-ai-go. Supports both Gemini Developer API and Vertex AI. Apache-2.0. Published Mar 18, 2026. |
| github.com/ollama/ollama/api | v0.18.2 | Ollama (local) | Official Ollama client — embedded in the Ollama project itself. Typed client for all REST endpoints. Pre-v1 but actively maintained alongside Ollama releases. |

### Supporting Libraries

| Library | Version | Purpose | When to Use |
|---------|---------|---------|-------------|
| github.com/cenkalti/backoff/v5 | v5.x | Exponential backoff for provider failover | Provider call retries with jitter. Use for all LLM API calls — prevents thundering-herd on rate limit errors. v5 requires Go 1.23 (matches our baseline). |
| github.com/google/renameio | v1.x | Atomic file writes (temp → rename) | All .planning/ artifact writes. Write to temp in same directory, then os.Rename — either full write or nothing. Prevents corrupt state on interrupt. |

### Development Tools

| Tool | Purpose | Notes |
|------|---------|-------|
| goreleaser | Cross-platform binary release (macOS, Linux, Windows) | Single config builds all targets. Use with GitHub Actions for distribution. |
| golangci-lint | Linting and static analysis | Run errcheck, staticcheck, govet, revive at minimum. Catches unhandled errors in provider callbacks. |
| go test -race | Race condition detection | Required for all concurrent agent execution tests. Run in CI. |
| go tool cover | Coverage reporting | Verification gate: GopherMind enforces its own coverage thresholds. Dogfood it. |

## Installation

```bash
# Initialize module
go mod init gophermind

# CLI framework
go get github.com/spf13/cobra@v1.10.2
go get github.com/spf13/viper@v1.21.0

# LLM provider SDKs
go get github.com/anthropics/anthropic-sdk-go@latest
go get github.com/openai/openai-go/v3@latest
go get google.golang.org/genai@latest
go get github.com/ollama/ollama/api@latest

# Concurrency and reliability
go get golang.org/x/sync@latest
go get github.com/cenkalti/backoff/v5@latest

# Atomic file writes
go get github.com/google/renameio@latest
```

## Alternatives Considered

| Recommended | Alternative | When to Use Alternative |
|-------------|-------------|-------------------------|
| github.com/spf13/cobra | urfave/cli v2 | Only if you prefer functional options over struct-based commands. Cobra's annotation-based shell completion and documentation generation are better for a tool like GopherMind. |
| log/slog (stdlib) | github.com/rs/zerolog | Only if benchmarks show slog is a bottleneck (unlikely for a CLI tool). zerolog is ~1.3x faster but adds a dependency. slog is the idiomatic 2025 choice for new Go projects. |
| log/slog (stdlib) | github.com/uber-go/zap | Same rationale as zerolog — zap adds complexity and external dependency with marginal gain for CLI I/O rates. |
| github.com/openai/openai-go/v3 | github.com/sashabaranov/go-openai | sashabaranov is community-maintained and widely used, but openai/openai-go is the official SDK now. Use official for forward compatibility with OpenAI API changes. |
| google.golang.org/genai | github.com/google/generative-ai-go | generative-ai-go is deprecated. Support ends November 30, 2025. Do not use. |
| golang.org/x/sync (errgroup + semaphore) | Custom goroutine pool | Custom pools are fine for simple cases but errgroup handles error propagation and context cancellation correctly — critical for agent fanout patterns. |
| github.com/cenkalti/backoff/v5 | github.com/avast/retry-go | Both are solid. backoff v5 aligns with the Google-originated exponential backoff spec and is more commonly used in production Go services. Either works. |
| Pure DI (manual constructors) | google/wire or uber-go/fx | For a CLI tool of this size, manual constructor injection is simpler to read and debug. Wire/Fx add code generation or reflection overhead that isn't warranted for a single-binary CLI. |

## What NOT to Use

| Avoid | Why | Use Instead |
|-------|-----|-------------|
| github.com/google/generative-ai-go | Deprecated. Google ended support November 30, 2025. Will not receive security patches. | google.golang.org/genai |
| github.com/liushuangls/go-anthropic | Community fork — not official Anthropic. Will lag official API changes. | github.com/anthropics/anthropic-sdk-go |
| github.com/sirupsen/logrus | Maintenance mode. Author recommends migrating to slog. Mutex-per-write design predates modern Go idioms. | log/slog |
| Database (SQLite, bbolt, etc.) | Project explicitly requires file-based .planning/ persistence — no external DB. SQLite adds CGo dependency breaking cross-compilation. | Plain file I/O with atomic writes via renameio |
| github.com/cloudwego/eino | Heavyweight LLM framework with its own abstractions. Adds 35+ dependencies and imposes opinionated graph execution model that conflicts with GopherMind's explicit agent contract design. | Thin Provider interface wrapping official SDKs directly |
| gRPC / protobuf | No inter-service communication needed. Single-binary CLI calling external HTTP APIs. gRPC would add complexity with zero benefit. | net/http (stdlib) for all provider calls, handled by provider SDKs |
| Uber Fx / Google Wire | DI containers add reflection overhead and generated code complexity that is not justified for a CLI tool. | Manual constructor injection |

## Stack Patterns by Variant

**For the Provider interface (unified LLM abstraction):**
- Define a single `Provider` interface: `Complete(ctx, messages, opts) (Response, error)` and `Stream(ctx, messages, opts) (<-chan Token, error)`
- Each provider SDK wraps into a thin struct implementing this interface
- Rule-based router selects provider by task tier, not dynamically
- Because: explicit routing is auditable; GopherMind's stated design goal is predictable, debuggable behavior

**For parallel agent execution:**
- Use `errgroup.WithContext` for the agent fan-out
- Use `golang.org/x/sync/semaphore` weighted semaphore to cap concurrent provider calls (e.g., max 5 simultaneous LLM calls)
- Dependency graph: topological sort of agent dependencies, execute each wave with errgroup
- Because: errgroup propagates first error and cancels remaining agents — correct behavior for a verification-gated pipeline

**For file-based state (STATE.md, JSONL events, artifact files):**
- All writes use atomic pattern: `renameio.WriteFile(path, data, perm)`
- Reads use `os.ReadFile` — no locking needed since writes are atomic
- JSONL event log: open with `os.OpenFile(O_APPEND|O_WRONLY|O_CREATE)`, protected by a single `sync.Mutex` in the event logger
- Because: atomic writes prevent half-written artifacts on SIGINT; append+mutex is the standard JSONL write pattern

**For CLI configuration layering:**
- Viper binds: flags (cobra PersistentPreRun) > env vars (GOPHERMIND_*) > config.json > defaults
- Config file location: `~/.config/gophermind/config.json` or `.gophermind/config.json` in project root
- Because: this is the standard Cobra+Viper precedence pattern used by kubectl, helm, and most production Go CLIs

## Version Compatibility

| Package | Compatible With | Notes |
|---------|-----------------|-------|
| github.com/spf13/viper v1.21.0 | Go 1.23+ | Viper v1.21 requires Go 1.23 minimum — sets our Go floor |
| github.com/cenkalti/backoff/v5 | Go 1.23+ | v5 also requires Go 1.23 — consistent with viper constraint |
| github.com/anthropics/anthropic-sdk-go v1.27.1 | Go 1.18+ | No conflict with Go 1.23 baseline |
| github.com/openai/openai-go/v3 v3.29.0 | Go 1.22+ | No conflict with Go 1.23 baseline |
| google.golang.org/genai v1.51+ | Go 1.21+ | No conflict with Go 1.23 baseline |
| github.com/ollama/ollama/api v0.18.2 | Go 1.22+ | Pre-v1; pin to minor version in go.sum, watch for breaking changes |

**Effective Go minimum: 1.23** (driven by viper v1.21 and backoff v5)

## Sources

- `pkg.go.dev/github.com/anthropics/anthropic-sdk-go` — v1.27.1 verified Mar 18, 2026 (HIGH confidence)
- `pkg.go.dev/github.com/openai/openai-go` + releases page — v3.29.0 verified Mar 17, 2026 (HIGH confidence)
- `pkg.go.dev/google.golang.org/genai` — v1.51.0 verified Mar 18, 2026, GA status confirmed (HIGH confidence)
- `pkg.go.dev/github.com/ollama/ollama/api` — v0.18.2 verified Mar 18, 2026 (HIGH confidence)
- `pkg.go.dev/github.com/spf13/cobra` — v1.10.2 Dec 2025 (HIGH confidence)
- `pkg.go.dev/github.com/spf13/viper` — v1.21.0 Sep 2025, Go 1.23+ required (HIGH confidence)
- `pkg.go.dev/golang.org/x/sync/errgroup` + `semaphore` — official Go extended library (HIGH confidence)
- `github.com/cenkalti/backoff` — v5 requires Go 1.23, confirmed via pkg.go.dev (HIGH confidence)
- `github.com/google/renameio` — pkg.go.dev + Michael Stapelberg's original post on atomic Go file writes (MEDIUM confidence — widely used pattern, library lightly maintained but pattern is stable)
- WebSearch: "Google generative-ai-go deprecated" — confirmed deprecated, support ends Nov 30, 2025 (HIGH confidence, multiple sources)
- WebSearch: "Go slog vs zerolog production 2025" — slog recommended for new projects, zerolog for extreme perf (HIGH confidence, multiple sources agree)

---
*Stack research for: GopherMind — Go multi-agent LLM orchestration platform*
*Researched: 2026-03-20*
