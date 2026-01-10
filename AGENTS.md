# AGENTS.md - Guidelines for repodocs-go

## Build & Test

```bash
make build              # Build binary to ./build/repodocs
make test               # Unit tests (fast, -short)
make test-integration   # Integration tests
make test-e2e           # E2E tests
make lint               # golangci-lint (govet + misspell)
make fmt                # Format code
make deps               # Download and tidy dependencies

# Run single test
go test -v -run TestName ./internal/converter/...
```

## Architecture

**Flow**: URL → Detector → Strategy → Fetcher/Renderer → Converter → Writer

| Package | Purpose |
|---------|---------|
| `internal/app` | Orchestrator, Detector (routes URLs to strategies) |
| `internal/strategies` | crawler, git, sitemap, llms, pkggo |
| `internal/fetcher` | Stealth HTTP client (tls-client) |
| `internal/renderer` | Headless browser (Rod/Chromium) |
| `internal/converter` | HTML → Markdown pipeline |
| `internal/cache` | BadgerDB persistent cache |
| `internal/output` | Markdown writer with frontmatter |
| `internal/domain` | Interfaces, models, errors |

## Code Style

### Imports (3 groups, blank-line separated)
```go
import (
    "context"                                    // 1. Standard library

    "github.com/stretchr/testify/assert"         // 2. External deps

    "github.com/quantmind-br/repodocs-go/internal/domain"  // 3. Internal
)
```

### Naming
- Interfaces: `Fetcher`, `Renderer`, `Cache` (verb-er)
- Structs: `CrawlerStrategy`, `ClientOptions` (PascalCase)
- Constructors: `NewClient()`, `NewOrchestrator()`
- Options: `ClientOptions`, `RetrierOptions`
- Tests: `TestGet_NotFound`

### Error Handling
```go
// Sentinel errors in internal/domain/errors.go
var ErrCacheMiss = errors.New("cache miss")

// Wrap with context
return fmt.Errorf("failed: %w", err)

// Check errors
if errors.Is(err, domain.ErrCacheMiss) { ... }
var fetchErr *domain.FetchError
if errors.As(err, &fetchErr) { ... }
```

### Interfaces (context first, error last)
```go
type Cache interface {
    Get(ctx context.Context, key string) ([]byte, error)
    Set(ctx context.Context, key string, value []byte, ttl time.Duration) error
}

// Options pattern: DefaultXxxOptions() + validation in NewXxx()
type ClientOptions struct { Timeout time.Duration; MaxRetries int }
func DefaultClientOptions() ClientOptions {
    return ClientOptions{Timeout: 30 * time.Second, MaxRetries: 3}
}
```

### Testing
```go
require.NoError(t, err)       // Fails immediately
assert.Equal(t, expected, actual)  // Continues

// Table-driven
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

Test location: `tests/unit/<package>/` mirrors `internal/<package>/`

### Logging (zerolog via utils.Logger)
```go
s.logger.Info().Str("url", url).Msg("Starting extraction")
s.logger.Warn().Err(err).Str("url", url).Msg("Failed")
```

### Concurrency
- Always accept `context.Context` for cancellation
- Use `sync.Map` for concurrent maps, `sync.Mutex` for simple state
- Check context in loops: `select { case <-ctx.Done(): return ctx.Err() default: }`

## DO NOT

- Suppress errors silently (`_ = err`)
- Use `panic` for recoverable errors
- Import from `cmd/` in `internal/`
- Create circular dependencies
- Use type assertions without error checking

## Task Tracking (bd)

```bash
bd ready                    # Find unblocked work
bd create "Title" -t task -p 2    # Create issue
bd update bd-42 --status in_progress   # Update
bd close bd-42 --reason "Done"   # Complete
bd sync                    # Sync with git (run at session end)
```

**Priorities**: 0=Critical, 1=High, 2=Medium, 3=Low, 4=Backlog
**Types**: bug, feature, task, epic, chore

**Workflow**:
1. `bd ready` to find work
2. Claim and implement
3. Create linked issues for discovered work (`--deps discovered-from:<id>`)
4. Complete and sync

## Session Completion

Work is NOT complete until `git push` succeeds:
```bash
git pull --rebase && bd sync && git push
```
