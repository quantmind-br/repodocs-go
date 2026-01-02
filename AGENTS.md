# AGENTS.md - AI Agent Guidelines for repodocs-go

## Issue Tracking

This project uses **bd (beads)** for issue tracking. Run `bd prime` for workflow context.

```bash
bd ready                              # Find unblocked work
bd create "Title" --type task         # Create issue
bd close <id>                         # Complete work
bd sync                               # Sync with git (run at session end)
```

## Quick Reference

```bash
# Build
make build                            # Build binary to ./build/repodocs

# Tests
make test                             # Unit tests (fast, -short flag)
make test-integration                 # Integration tests
make test-e2e                         # E2E tests
make test-all                         # All test suites

# Run single test
go test -v -run TestFunctionName ./path/to/package/...
go test -v -run "TestGet_Found" ./tests/unit/cache/...

# Lint & Format
make lint                             # golangci-lint (govet + misspell)
make fmt                              # Format code
make vet                              # Go vet only
make deps                             # Download and tidy dependencies
```

## Architecture

- **Hexagonal Architecture** with Strategy pattern
- Entry: `cmd/repodocs/main.go` -> `internal/app/Orchestrator` -> `internal/strategies/*`
- Interfaces in `internal/domain/interfaces.go`
- Dependency injection via `strategies.Dependencies` struct

## Code Style

### Import Organization (3 groups, blank-line separated)
```go
import (
    "context"                                              // 1. Standard library

    "github.com/stretchr/testify/assert"                   // 2. External deps

    "github.com/quantmind-br/repodocs-go/internal/domain"  // 3. Internal
)
```

### Naming Conventions

| Type | Convention | Example |
|------|------------|---------|
| Interfaces | Verb-er suffix | `Fetcher`, `Renderer`, `Cache` |
| Structs | Noun, PascalCase | `CrawlerStrategy`, `ClientOptions` |
| Constructors | `New` + type | `NewClient()`, `NewOrchestrator()` |
| Options structs | Type + `Options` | `ClientOptions`, `RetrierOptions` |
| Test functions | `Test` + Func + `_` + Scenario | `TestGet_NotFound` |

### Error Handling

```go
// Sentinel errors in internal/domain/errors.go
var ErrCacheMiss = errors.New("cache miss")

// Wrap errors with context
return fmt.Errorf("failed to create dependencies: %w", err)

// Check errors
if errors.Is(err, domain.ErrCacheMiss) { ... }
var fetchErr *domain.FetchError
if errors.As(err, &fetchErr) { ... }
```

### Interface Design & Configuration
```go
// Interfaces: context.Context first, error last
type Cache interface {
    Get(ctx context.Context, key string) ([]byte, error)
    Set(ctx context.Context, key string, value []byte, ttl time.Duration) error
}

// Options pattern: DefaultXxxOptions() + validation in NewXxx()
type ClientOptions struct { Timeout time.Duration; MaxRetries int }
func DefaultClientOptions() ClientOptions { return ClientOptions{Timeout: 30 * time.Second, MaxRetries: 3} }
```

### Testing Patterns

```go
// Use testify: require (fails immediately), assert (continues)
require.NoError(t, err)
assert.Equal(t, expected, actual)

// Table-driven tests
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

// Use t.TempDir() for temp directories (auto-cleanup)
```

**Test location:** `tests/unit/<package>/` mirrors `internal/<package>/`

### Logging (zerolog via utils.Logger)
```go
s.logger.Info().Str("url", url).Msg("Starting extraction")
s.logger.Warn().Err(err).Str("url", url).Msg("Failed to convert")
```

### Concurrency
- Always accept `context.Context` for cancellation
- Use `sync.Map` for concurrent map access, `sync.Mutex` for simple state
- Check context in loops: `select { case <-ctx.Done(): return ctx.Err() default: }`

## File Organization

| Directory | Purpose |
|-----------|---------|
| `internal/domain/` | Interfaces, models, errors (no deps) |
| `internal/app/` | Orchestrator, strategy detection |
| `internal/strategies/` | Extraction strategies (crawler, git, sitemap, llms, pkggo) |
| `internal/fetcher/` | HTTP client with stealth (tls-client) |
| `internal/renderer/` | Headless browser (rod) |
| `internal/converter/` | HTML -> Markdown pipeline |
| `internal/cache/` | BadgerDB implementation |
| `internal/output/` | File writer |

## Linter Configuration

From `.golangci.yml`:
- `govet` (atomic, bools, composites, copylocks, nilfunc, printf, stdmethods, structtag)
- `misspell`

## DO NOT

- Suppress errors silently (`_ = err`)
- Use `panic` for recoverable errors
- Import from `cmd/` in `internal/`
- Create circular dependencies between packages
- Use type assertions without error checking

## Session Completion

**Work is NOT complete until `git push` succeeds.**

1. Run quality gates (if code changed): tests, lint, build
2. Update issues: `bd close <id>` for completed work
3. Push to remote:
   ```bash
   git pull --rebase && bd sync && git push
   ```

**Critical:** Never stop before pushing - that leaves work stranded locally.
