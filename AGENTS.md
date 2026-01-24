# AGENTS.md - repodocs-go

**Generated:** 2026-01-24 | **Commit:** 63c3d91 | **Branch:** main

## Overview

CLI tool extracting documentation from websites, Git repos, sitemaps, pkg.go.dev, docs.rs, llms.txt into clean Markdown. Interface-driven architecture with Strategy pattern.

## Structure

```
repodocs-go/
├── cmd/repodocs/      # Single entry point (Cobra CLI, all commands in main.go)
├── internal/
│   ├── app/           # Orchestrator + Detector (strategy routing)
│   ├── strategies/    # 8 extraction strategies (crawler, git, sitemap, llms, wiki, pkggo, docsrs, github_pages)
│   ├── converter/     # HTML→Markdown pipeline (readability, sanitizer, encoding)
│   ├── fetcher/       # Stealth HTTP client (tls-client, bot avoidance)
│   ├── renderer/      # Headless browser pool (Rod/Chromium)
│   ├── cache/         # BadgerDB persistence
│   ├── llm/           # Multi-provider AI (OpenAI, Anthropic, Google) + circuit breakers
│   ├── output/        # Markdown writer with YAML frontmatter
│   ├── domain/        # Interfaces, models, sentinel errors
│   ├── tui/           # Interactive config (Bubble Tea/Huh)
│   └── config/        # YAML config handling
├── tests/             # External test suite (unit/, integration/, e2e/)
└── pkg/version/       # Public version info
```

## Flow

```
URL → Detector → Strategy → Fetcher/Renderer → Converter → Writer
                    ↓
              (Optional) MetadataEnhancer (LLM)
```

Detection order: LLMS → PkgGo → DocsRS → Sitemap → Git → GitHub Pages → Wiki → Crawler

## Build & Test

```bash
make build              # Binary → ./build/repodocs (CGO_ENABLED=0)
make test               # Unit tests (-short, -race)
make test-integration   # Network-dependent tests
make test-e2e           # Full CLI workflow tests
make lint               # golangci-lint v2 (govet + misspell)
make fmt                # gofmt + goimports
make coverage           # HTML report → ./coverage/

# Single test
go test -v -run TestName ./internal/converter/...
```

## Where to Look

| Task | Location | Notes |
|------|----------|-------|
| Add new source type | `internal/strategies/` + `internal/app/detector.go` | Implement Strategy interface, add to detection order |
| Modify HTML→MD conversion | `internal/converter/` | Pipeline: encoding → readability → sanitizer → markdown |
| Add CLI flag | `cmd/repodocs/main.go` | All Cobra commands in single file |
| Add LLM provider | `internal/llm/` | Implement LLMProvider interface |
| Change caching behavior | `internal/cache/` | BadgerDB wrapper |
| JS rendering issues | `internal/renderer/` | Rod pool management |
| Config TUI changes | `internal/tui/` | Bubble Tea + Huh forms |

## Code Style

### Imports (3 groups, blank-line separated)
```go
import (
    "context"                                    // 1. stdlib

    "github.com/stretchr/testify/assert"         // 2. external

    "github.com/quantmind-br/repodocs-go/internal/domain"  // 3. internal
)
```

### Naming
- Interfaces: `Fetcher`, `Renderer`, `Cache` (verb-er)
- Structs: `CrawlerStrategy`, `ClientOptions` (PascalCase)
- Constructors: `NewClient()`, `NewOrchestrator()`
- Tests: `TestGet_NotFound` (Method_Scenario)

### Error Handling
```go
// Sentinel errors in internal/domain/errors.go
var ErrCacheMiss = errors.New("cache miss")

// Always wrap with context
return fmt.Errorf("failed to fetch %s: %w", url, err)

// Check errors
if errors.Is(err, domain.ErrCacheMiss) { ... }
var fetchErr *domain.FetchError
if errors.As(err, &fetchErr) { ... }
```

### Interfaces
```go
type Cache interface {
    Get(ctx context.Context, key string) ([]byte, error)  // context first
    Set(ctx context.Context, key string, value []byte, ttl time.Duration) error  // error last
}
```

### Options Pattern
```go
type ClientOptions struct { Timeout time.Duration; MaxRetries int }
func DefaultClientOptions() ClientOptions {
    return ClientOptions{Timeout: 30 * time.Second, MaxRetries: 3}
}
```

## Testing

```go
require.NoError(t, err)       // Fails immediately
assert.Equal(t, expected, actual)  // Continues

// Table-driven (standard)
tests := []struct {
    name    string
    input   string
    wantErr bool
}{
    {name: "valid", input: "test", wantErr: false},
}
for _, tt := range tests {
    t.Run(tt.name, func(t *testing.T) { ... })
}

// Use t.TempDir() for temp files (auto-cleanup)
```

Test locations:
- `tests/unit/<pkg>/` mirrors `internal/<pkg>/` (external package tests)
- `internal/<pkg>/*_test.go` for internal logic tests
- `tests/mocks/` for gomock-generated mocks
- `tests/testutil/` for shared helpers (NewBadgerCache, NewTestServer)
- `tests/fixtures/` for test HTML/XML data

## Logging

```go
s.logger.Info().Str("url", url).Msg("Starting extraction")
s.logger.Warn().Err(err).Str("url", url).Msg("Failed")
```

## Concurrency

- Always accept `context.Context` for cancellation
- Check context in loops: `select { case <-ctx.Done(): return ctx.Err() default: }`
- Use `sync.Map` for concurrent maps, `sync.Mutex` for simple state

## DO NOT

- Suppress errors silently (`_ = err`)
- Use `panic` for recoverable errors
- Import from `cmd/` in `internal/`
- Create circular dependencies
- Type assertions without error checking (`value, ok := x.(Type)`)
- Add `nolint` directives (only exception: `internal/converter/encoding.go`)
- Refactor while fixing bugs (minimal, targeted fixes only)

## Deprecated (avoid in new code)

| Item | Location | Replacement |
|------|----------|-------------|
| `MaxRetries` | `internal/config/` | Use `RateLimit.MaxRetries` |
| `Metadata` struct | `internal/domain/models.go` | Use `SimpleMetadata` |
| `MetadataIndex` | `internal/domain/models.go` | Use `SimpleMetadataIndex` |
| `DocumentMetadata` | `internal/domain/models.go` | Use `SimpleDocumentMetadata` |

## Dependency Injection

```go
// Composition root: internal/strategies/strategy.go
type Dependencies struct {
    Fetcher          *fetcher.Client
    Renderer         domain.Renderer
    Cache            domain.Cache
    Converter        *converter.Pipeline
    Writer           *output.Writer
    Logger           *utils.Logger
    LLMProvider      domain.LLMProvider
    MetadataEnhancer *llm.MetadataEnhancer
}

// Orchestrator creates Dependencies, passes to strategies
```

## Key Interfaces

| Interface | Location | Purpose |
|-----------|----------|---------|
| `Strategy` | `internal/domain` | Extraction strategy (Name, CanHandle, Execute) |
| `Fetcher` | `internal/domain` | HTTP client with caching |
| `Renderer` | `internal/domain` | Headless browser rendering |
| `Cache` | `internal/domain` | Persistent cache operations |
| `Converter` | `internal/domain` | HTML→Markdown conversion |
| `LLMProvider` | `internal/domain` | AI completion interface |

## Complexity Hotspots

| File | Lines | Notes |
|------|-------|-------|
| `strategies/docsrs_renderer.go` | 628 | Rustdoc JSON→Markdown, complex signatures |
| `strategies/github_pages.go` | 600 | Multi-phase discovery, deduplication |
| `strategies/docsrs_types.go` | 591 | Rustdoc JSON schema structures |
| `utils/url.go` | 423 | URL normalization (critical for caching) |
| `tui/forms.go` | 409 | Interactive config state machine |
| `app/orchestrator.go` | 370 | Main coordination logic |

## Session Completion

Work is NOT complete until `git push` succeeds:
```bash
git pull --rebase && git push
```
