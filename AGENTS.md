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


<!-- BEGIN BEADS INTEGRATION -->
## Issue Tracking with bd (beads)

**IMPORTANT**: This project uses **bd (beads)** for ALL issue tracking. Do NOT use markdown TODOs, task lists, or other tracking methods.

### Why bd?

- Dependency-aware: Track blockers and relationships between issues
- Git-friendly: Auto-syncs to JSONL for version control
- Agent-optimized: JSON output, ready work detection, discovered-from links
- Prevents duplicate tracking systems and confusion

### Quick Start

**Check for ready work:**

```bash
bd ready --json
```

**Create new issues:**

```bash
bd create "Issue title" --description="Detailed context" -t bug|feature|task -p 0-4 --json
bd create "Issue title" --description="What this issue is about" -p 1 --deps discovered-from:bd-123 --json
```

**Claim and update:**

```bash
bd update bd-42 --status in_progress --json
bd update bd-42 --priority 1 --json
```

**Complete work:**

```bash
bd close bd-42 --reason "Completed" --json
```

### Issue Types

- `bug` - Something broken
- `feature` - New functionality
- `task` - Work item (tests, docs, refactoring)
- `epic` - Large feature with subtasks
- `chore` - Maintenance (dependencies, tooling)

### Priorities

- `0` - Critical (security, data loss, broken builds)
- `1` - High (major features, important bugs)
- `2` - Medium (default, nice-to-have)
- `3` - Low (polish, optimization)
- `4` - Backlog (future ideas)

### Workflow for AI Agents

1. **Check ready work**: `bd ready` shows unblocked issues
2. **Claim your task**: `bd update <id> --status in_progress`
3. **Work on it**: Implement, test, document
4. **Discover new work?** Create linked issue:
   - `bd create "Found bug" --description="Details about what was found" -p 1 --deps discovered-from:<parent-id>`
5. **Complete**: `bd close <id> --reason "Done"`

### Auto-Sync

bd automatically syncs with git:

- Exports to `.beads/issues.jsonl` after changes (5s debounce)
- Imports from JSONL when newer (e.g., after `git pull`)
- No manual export/import needed!

### Important Rules

- ✅ Use bd for ALL task tracking
- ✅ Always use `--json` flag for programmatic use
- ✅ Link discovered work with `discovered-from` dependencies
- ✅ Check `bd ready` before asking "what should I work on?"
- ❌ Do NOT create markdown TODO lists
- ❌ Do NOT use external issue trackers
- ❌ Do NOT duplicate tracking systems

For more details, see README.md and docs/QUICKSTART.md.

<!-- END BEADS INTEGRATION -->
