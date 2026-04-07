# Coding Conventions

**Analysis Date:** 2026-04-07

## Naming Patterns

**Files:**
- Package-level files: lowercase with underscores (e.g., `markdown.go`, `git_strategy.go`)
- Test files: `{name}_test.go` or `{name}_{suffix}_test.go` (e.g., `orchestrator_test.go`, `git_integration_test.go`)
- Strategy implementations: `{type}_strategy.go` and `{type}_strategy_test.go` (e.g., `crawler_strategy.go`, `sitemap.go`)
- Nested packages: lowercase (e.g., `strategies/git/strategy.go`)

**Functions:**
- Exported (public): PascalCase (e.g., `NewOrchestrator`, `Execute`, `CanHandle`)
- Unexported (private): camelCase (e.g., `checkInternet`, `cleanMarkdown`, `parseLogLevel`)
- Constructor pattern: `New{TypeName}` (e.g., `NewLogger`, `NewDependencies`, `NewOrchestrator`)
- Predicates: `Can{Action}` or `Is{Property}` (e.g., `CanHandle`, `IsRetryable`)

**Variables:**
- Package-level constants: UPPER_SNAKE_CASE (e.g., `MinFormatVersion`)
- Configuration structs: PascalCase (e.g., `Config`, `LoggerOptions`, `DependencyOptions`)
- Options params: `{Name}Options` suffix (e.g., `OrchestratorOptions`, `WriterOptions`, `LoggerOptions`)
- Sentinel errors: `Err{Name}` prefix (e.g., `ErrNotFound`, `ErrCacheMiss`, `ErrInvalidURL`)
- Interface types: `{Name}` with -er suffix for behavior (e.g., `Fetcher`, `Renderer`, `Strategy`)

**Types:**
- Structs: PascalCase (e.g., `Document`, `FetchError`, `Dependencies`)
- Interfaces: PascalCase, typically -er suffix (e.g., `Strategy`, `Fetcher`, `Cache`, `Logger`)
- Custom error types: `{Name}Error` (e.g., `FetchError`, `ValidationError`, `StrategyError`)
- Config/options: `{Name}Config` or `{Name}Options` (e.g., `CacheConfig`, `LoggerOptions`)

## Code Style

**Formatting:**
- Tool: `gofmt` (Go standard formatter)
- Run: `make fmt` (applies gofmt with -s flag for simplification)

**Linting:**
- Tool: `golangci-lint` v2
- Config: `.golangci.yml`
- Key rules enabled:
  - `govet`: Type-related linting (atomic, bools, composites, copylocks, nilfunc, printf, stdmethods, structtag)
  - `misspell`: Spelling error detection
- Exclusions: Known issues noted in config (e.g., ineffectual assignment in `internal/converter/encoding.go`)
- Run: `make lint`

**Line Length:**
- No explicit limit enforced; follows Go conventions (80-100 char preferred for readability)

## Import Organization

**Order:**
1. Standard library imports (e.g., `fmt`, `io`, `context`)
2. External imports (third-party packages)
3. Internal imports (relative to project, e.g., `github.com/quantmind-br/repodocs/...`)

**Path Aliases:**
Not used in current codebase. Imports use full module paths without custom aliases.

**Example from `cmd/repodocs/main.go`:**
```go
import (
	"context"
	"fmt"
	"net/http"
	"os"
	// ... more stdlib

	"github.com/spf13/cobra"
	"github.com/spf13/viper"
	// ... more third-party

	"github.com/quantmind-br/repodocs/internal/app"
	"github.com/quantmind-br/repodocs/internal/config"
	// ... more internal
)
```

## Error Handling

**Patterns:**

1. **Wrapping errors with context:**
   - Use `fmt.Errorf("message: %w", err)` to wrap errors with context
   - Always include `%w` verb for error chain preservation
   - Add descriptive message (e.g., "failed to load config", "failed to fetch URL")
   - Example from `cmd/repodocs/main.go`:
     ```go
     cfg, err := config.Load()
     if err != nil {
         return fmt.Errorf("failed to load config: %w", err)
     }
     ```

2. **Sentinel errors (package-level):**
   - Define in `internal/domain/errors.go` as package-level `var` with `errors.New()`
   - Use `errors.Is()` to check sentinel errors
   - Examples: `ErrNotFound`, `ErrCacheMiss`, `ErrInvalidURL`, `ErrNoStrategy`
   - From `internal/domain/errors.go`:
     ```go
     var (
         ErrNotFound = errors.New("not found")
         ErrCacheMiss = errors.New("cache miss")
     )
     ```

3. **Custom error types:**
   - Use structs for errors with additional context
   - Implement `Error()` string method
   - Implement `Unwrap()` error method for error chain support
   - Examples: `FetchError`, `ValidationError`, `StrategyError`, `LLMError`
   - From `internal/domain/errors.go`:
     ```go
     type FetchError struct {
         URL        string
         StatusCode int
         Err        error
     }
     
     func (e *FetchError) Error() string {
         // Custom formatting
     }
     
     func (e *FetchError) Unwrap() error {
         return e.Err
     }
     
     // Constructor
     func NewFetchError(url string, statusCode int, err error) *FetchError {
         return &FetchError{URL: url, StatusCode: statusCode, Err: err}
     }
     ```

4. **Error checking:**
   - Use `if err != nil` pattern (idiomatic Go)
   - Check immediately after operation
   - Return wrapped error or sentinel error
   - Use `errors.As()` for type assertion on custom error types
   - Use `errors.Is()` for sentinel error checking

5. **Strategy pattern for retryable errors:**
   - Use `IsRetryable(err error) bool` helper from `internal/domain/errors.go`
   - Checks for specific status codes (429, 503, 502, 504, 520-530)
   - Handles custom `RetryableError` type and timeout/rate-limit errors

## Logging

**Framework:** `zerolog` (https://github.com/rs/zerolog)

**Logger wrapper:** `internal/utils/logger.go`
- `Logger` struct wraps `zerolog.Logger`
- Methods: `Info()`, `Warn()`, `Error()`, `Debug()` (chainable API)
- Example: `logger.Info().Str("key", val).Msg("message")`

**Patterns:**

1. **Creating loggers:**
   ```go
   logger := utils.NewLogger(utils.LoggerOptions{
       Level:   "info",
       Format:  "pretty",  // or "json"
       Verbose: verbose,
   })
   ```

2. **Logging calls (fluent/chainable):**
   ```go
   logger.Info().Msg("starting process")
   logger.Warn().Err(err).Msg("warning occurred")
   logger.Debug().Str("url", url).Int("retries", count).Msg("retrying")
   logger.Error().Stack().Err(err).Msg("critical error")
   ```

3. **Log levels:**
   - `Debug`: Detailed diagnostic info (enabled with `-v` flag or verbose mode)
   - `Info`: General informational messages
   - `Warn`: Warning conditions that should be noted
   - `Error`: Error conditions
   - JSON format available for structured logging

4. **When to log:**
   - Log at operation boundaries (start/end of major operations)
   - Log errors with context (URL, status code, etc.)
   - Log retry attempts and backoff timing
   - Log configuration choices (providers, timeouts, etc.)
   - Use debug level for detailed state information

## Comments

**When to Comment:**
- Complex algorithms or non-obvious logic
- Public API exports (functions, types, interfaces)
- Important design decisions or gotchas
- Section headers for logical groupings
- Deprecation notices with alternatives
- Examples of typical usage

**JSDoc/TSDoc:**
Not used (Go project). Uses standard Go documentation conventions.

**Go Documentation Pattern:**
- Comment immediately precedes the declaration
- Starts with the name of the thing being documented
- Example from `internal/domain/errors.go`:
  ```go
  // Document represents a processed documentation page
  type Document struct { ... }
  
  // ErrNotFound indicates a resource was not found
  var ErrNotFound = errors.New("not found")
  
  // NewFetchError creates a new FetchError
  func NewFetchError(url string, statusCode int, err error) *FetchError { ... }
  ```

**Helper comments in test files:**
```go
// LoadFixture loads a test fixture file and returns its contents.
// The path is relative to the tests/fixtures directory.
// Usage:
//
//	data := helpers.LoadFixture(t, "git/sample-repo.tar.gz")
func LoadFixture(t *testing.T, path string) []byte {
```

## Function Design

**Size:** Functions typically under 50 lines; long functions broken into helpers

**Parameters:**
- Use options structs for 3+ parameters (e.g., `LoggerOptions`, `ClientOptions`)
- Context as first parameter for functions doing async work
- Testing param `t *testing.T` as first param in test functions

**Return Values:**
- Error as last return value (Go convention)
- Use named returns sparingly (only for clarity or cleanup)
- Return concrete types when possible (not interfaces, except for injection)

**Example patterns:**
```go
// Single return with error
func (c *Client) Do(ctx context.Context) error { ... }

// Multiple returns with error last
func NewOrchestrator(opts Options) (*Orchestrator, error) { ... }

// Constructor with options struct
func NewLogger(opts LoggerOptions) *Logger { ... }

// Interface-based strategy pattern
func (o *Orchestrator) detectStrategy(url string) (Strategy, error) { ... }
```

## Module Design

**Exports (Public):**
- Functions and types intended for external use
- Must be documented with Go comments
- Examples: `NewOrchestrator`, `Config`, `Strategy` interface

**Unexported (Private):**
- Functions and types for internal use within package
- No documentation requirement (but helpful for complex logic)
- Examples: `checkInternet()`, `cleanMarkdown()`, helper functions

**Barrel Files (Re-exports):**
Not used in current structure. Each package exports from its own files directly.

**Package Organization:**
- `internal/`: All non-public packages (interfaces, implementations, utilities)
- `cmd/`: Entry points (CLI commands)
- `pkg/`: Potentially reusable packages (currently only `version`)
- `tests/`: Test infrastructure (helpers, fixtures, test suites)

**Dependency Injection Pattern:**
Used throughout for testability. Example from `internal/strategies/strategy.go`:
```go
type Dependencies struct {
    Fetcher     domain.Fetcher
    Renderer    domain.Renderer
    Cache       domain.Cache
    Converter   *converter.Pipeline
    Writer      *output.Writer
    Logger      *utils.Logger
    LLMProvider domain.LLMProvider
}
```

Strategies receive `Dependencies` allowing tests to inject mocks via the `domain` interfaces.

## Variable Scoping

**Global variables (package-level):**
- Pre-compiled regex patterns: `var linkRegex = regexp.MustCompile(...)`
- Sentinel errors: `var ErrNotFound = errors.New(...)`
- Test-injectable functions in main: `var osStat = os.Stat`, `var execLookPath = exec.LookPath`
- Cobra commands: `var rootCmd = &cobra.Command{...}`

**Context usage:**
- Context passed as first parameter to async functions
- Never store context in structs (except for cleanup/shutdown)
- Use `context.WithCancel()` for graceful shutdown patterns
- Example from `cmd/repodocs/main.go`:
  ```go
  ctx, cancel := context.WithCancel(context.Background())
  defer cancel()
  ```

## Interface Design

**Strategy pattern:** Core architectural pattern
- Location: `internal/strategies/strategy.go`
- Interface definition:
  ```go
  type Strategy interface {
      Name() string
      CanHandle(url string) bool
      Execute(ctx context.Context, url string, opts Options) error
  }
  ```
- Multiple concrete implementations (Crawler, Sitemap, Git, etc.)

**Domain interfaces:** Located in `internal/domain/interfaces.go`
- `Fetcher`: HTTP fetching with caching
- `Renderer`: JavaScript rendering (browser automation)
- `Cache`: Key-value cache abstraction
- `LLMProvider`: External LLM service abstraction

**Options pattern:** For configuration without signature bloat
```go
type OrchestratorOptions struct {
    domain.CommonOptions
    Config           *config.Config
    Split            bool
    IncludeAssets    bool
    ContentSelector  string
    ExcludeSelector  string
    ExcludePatterns  []string
    FilterURL        string
}
```

---

*Convention analysis: 2026-04-07*
