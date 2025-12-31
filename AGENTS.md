# AGENTS.md - AI Agent Guidelines for repodocs-go

## Issue Tracking

This project uses **bd (beads)** for issue tracking.
Run `bd prime` for workflow context, or install hooks (`bd hooks install`) for auto-injection.

**Quick reference:**
- `bd ready` - Find unblocked work
- `bd create "Title" --type task --priority 2` - Create issue
- `bd close <id>` - Complete work
- `bd sync` - Sync with git (run at session end)

For full workflow details: `bd prime`

## Quick Reference

```bash
# Build
make build                    # Build binary to ./build/repodocs

# Tests
make test                     # Unit tests with race detection
make test-integration         # Integration tests
make test-e2e                 # End-to-end tests
make test-all                 # All test suites

# Run single test
go test -v -run TestFunctionName ./path/to/package/...
go test -v -run "TestGet_Found" ./tests/unit/cache/...

# Lint & Format
make lint                     # golangci-lint (govet + misspell)
make fmt                      # Format code
make vet                      # Go vet only
make deps                     # Download and tidy dependencies
```

## Project Architecture

- **Hexagonal Architecture** with Strategy pattern
- Entry: `cmd/repodocs/main.go` -> `internal/app/Orchestrator` -> `internal/strategies/*`
- Interfaces in `internal/domain/interfaces.go`
- Dependency injection via `strategies.Dependencies` struct

## Code Style Guidelines

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
| Interfaces | Verb-er | `Fetcher`, `Renderer`, `Cache` |
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

### Interface Design (context first, error last)
```go
type Cache interface {
    Get(ctx context.Context, key string) ([]byte, error)
    Set(ctx context.Context, key string, value []byte, ttl time.Duration) error
    Close() error
}
```

### Struct Configuration Pattern
```go
type ClientOptions struct {
    Timeout    time.Duration
    MaxRetries int
}

func DefaultClientOptions() ClientOptions {
    return ClientOptions{Timeout: 30 * time.Second, MaxRetries: 3}
}

func NewClient(opts ClientOptions) (*Client, error) {
    if opts.Timeout <= 0 { opts.Timeout = 30 * time.Second }
    // ...
}
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
    {name: "empty", input: "", wantErr: true},
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
| `tests/unit/` | Unit tests |
| `tests/integration/` | Integration tests |
| `tests/e2e/` | End-to-end tests |

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

## Landing the Plane (Session Completion)

**When ending a work session**, you MUST complete ALL steps below. Work is NOT complete until `git push` succeeds.

**MANDATORY WORKFLOW:**

1. **File issues for remaining work** - Create issues for anything that needs follow-up
2. **Run quality gates** (if code changed) - Tests, linters, builds
3. **Update issue status** - Close finished work, update in-progress items
4. **PUSH TO REMOTE** - This is MANDATORY:
   ```bash
   git pull --rebase
   bd sync
   git push
   git status  # MUST show "up to date with origin"
   ```
5. **Clean up** - Clear stashes, prune remote branches
6. **Verify** - All changes committed AND pushed
7. **Hand off** - Provide context for next session

**CRITICAL RULES:**
- Work is NOT complete until `git push` succeeds
- NEVER stop before pushing - that leaves work stranded locally
- NEVER say "ready to push when you are" - YOU must push
- If push fails, resolve and retry until it succeeds
