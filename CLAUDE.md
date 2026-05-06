# CLAUDE.md

## Quick Commands

```bash
make build              # Build binary to ./build/repodocs
make test               # Unit tests (fast, -short)
make test-integration   # Integration tests
make test-e2e           # E2E tests
make lint               # golangci-lint (v2)
make fmt                # Format code
make vet                # Run go vet
```

## Architecture

**repodocs** extracts documentation from websites, Git repos, sitemaps, pkg.go.dev, llms.txt and converts to Markdown.

**Flow**: URL → Detector → Strategy → Fetcher/Renderer → Converter → Writer

### Strategy Pattern
Strategies implement `internal/strategies.Strategy`:
- `Name() string`
- `CanHandle(url string) bool`
- `Execute(ctx context.Context, url string, opts Options) error`

Detection order: LLMS → PkgGo → Sitemap → Git → Crawler

### Dependency Injection
`strategies.Dependencies` is composition root:
```go
type Dependencies struct {
    Fetcher   *fetcher.Client
    Renderer  domain.Renderer
    Cache     domain.Cache
    Converter *converter.Pipeline
    Writer    *output.Writer
    Logger    *utils.Logger
}
```

### Converter Pipeline
1. UTF-8 normalization
2. Content extraction (CSS selector or Readability)
3. HTML sanitization (remove scripts, nav, ads)
4. Markdown conversion
5. Metadata extraction

## Test Structure

```
tests/
├── unit/           # Fast unit tests (make test)
├── integration/    # Network-dependent tests
├── e2e/            # Full CLI tests
├── mocks/          # Generated mocks (go.uber.org/mock)
├── testutil/       # Shared helpers
└── fixtures/       # Test HTML/data
```

## Task Tracking

Use `bd` for task tracking (see AGENTS.md).

<!-- GSD:project-start source:PROJECT.md -->
## Project

**LM Studio Provider for repodocs**

Adding first-class LM Studio support to repodocs as a dedicated LLM provider. LM Studio runs local models and exposes an OpenAI-compatible API. This gives users a local, free alternative for metadata enhancement without needing cloud API keys.

**Core Value:** Users can run repodocs metadata enhancement with local LLM models via LM Studio, with zero-config defaults (no API key required, auto-detected localhost URL).

### Constraints

- **API compatibility**: Must use OpenAI-compatible chat completions format (LM Studio's API)
- **Existing patterns**: Must follow the established provider implementation pattern (see `openai.go`, `ollama.go`)
- **Config structure**: Must integrate with existing `LLMConfig` struct and Viper config loading
- **No breaking changes**: Adding a new provider must not affect existing provider behavior
<!-- GSD:project-end -->

<!-- GSD:stack-start source:codebase/STACK.md -->
## Technology Stack

## Languages
- Go 1.24.1 - The entire codebase and CLI tool is written in Go
- YAML - Configuration files and manifest definitions
## Runtime
- Go 1.24.1
- Linux/macOS/Windows compatible (cross-platform binary builds)
- Go modules
- `go.mod` and `go.sum` for dependency management
- Lock file: present and maintained
## Frameworks
- Cobra 1.10.2 - CLI framework for command parsing and subcommands
- Viper 1.21.0 - Configuration loading and management
- Charmbracelet Bubbletea 1.3.6 - Terminal UI framework
- Charmbracelet Huh 0.8.0 - Form and prompt components
- Charmbracelet Lipgloss 1.1.0 - Terminal styling
- tls-client (bogdanfinn/tls-client v1.11.2) - Stealth HTTP client with TLS fingerprinting
- fhttp (bogdanfinn/fhttp v0.6.2) - HTTP client matching Chrome headers
- Rod v0.116.2 - Headless browser control via DevTools Protocol
- Rod Stealth v0.4.9 - Anti-automation detection bypass
- goquery v1.11.0 - jQuery-like DOM querying
- go-readability v0.0.0-20251205110129-5db1dc9836f0 - Content extraction algorithm
- html-to-markdown/v2 v2.5.0 - HTML to Markdown conversion
- go-git/v5 v5.16.4 - Pure Go Git implementation
- BadgerDB v4.8.0 - Embedded key-value database
- zerolog v1.34.0 - Fast structured logging
- cenkalti/backoff/v4 v4.3.0 - Exponential backoff retry strategy
- schollz/progressbar/v3 v3.18.0 - Progress bar display
- golang.org/x/text v0.31.0 - Unicode text handling
- golang.org/x/net v0.47.0 - Network utilities
- gopkg.in/yaml.v3 v3.0.1 - YAML parsing and marshaling
- testify v1.11.1 - Assertion and mocking library
- go.uber.org/mock v0.5.0 - Mock generation (mockgen)
- goreleaser/v2 - Multi-platform binary builds
## Key Dependencies
- tls-client (bogdanfinn) - Enables stealth mode for avoiding bot detection
- Rod (go-rod) - Headless browser automation
- go-git - Git repository interaction
- html-to-markdown - HTML to Markdown conversion
- BadgerDB - Persistent caching
- Cobra - CLI framework
- Viper - Configuration management
- Charmbracelet suite - Terminal UI
- zerolog - Structured logging
## Configuration
- Configuration file: `~/.repodocs/config.yaml` (default location)
- Can override all config values with `REPODOCS_` prefixed environment variables
- Loaded via Viper in `internal/config/loader.go`
- `Makefile` with targets: `build`, `test`, `lint`, `coverage`
- CGO_ENABLED=0 for static binary builds
- Version injection via ldflags:
- `configs/config.yaml.template` - Template with all available options
- `go.mod` - Module definition and dependencies
- `Makefile` - Build and test automation
## Platform Requirements
- Go 1.24.1 or later
- golangci-lint v2 for linting
- goreleaser v2 for releases (optional)
- Chrome/Chromium for JavaScript rendering (optional but recommended)
- Linux, macOS, or Windows runtime (cross-platform compiled binary)
- Chrome/Chromium in PATH (required only if `--render-js` flag is used)
- Network connectivity for fetching remote documentation
- ~100MB disk space for cache directory (`~/.repodocs/cache`)
<!-- GSD:stack-end -->

<!-- GSD:conventions-start source:CONVENTIONS.md -->
## Conventions

## Naming Patterns
- Package-level files: lowercase with underscores (e.g., `markdown.go`, `git_strategy.go`)
- Test files: `{name}_test.go` or `{name}_{suffix}_test.go` (e.g., `orchestrator_test.go`, `git_integration_test.go`)
- Strategy implementations: `{type}_strategy.go` and `{type}_strategy_test.go` (e.g., `crawler_strategy.go`, `sitemap.go`)
- Nested packages: lowercase (e.g., `strategies/git/strategy.go`)
- Exported (public): PascalCase (e.g., `NewOrchestrator`, `Execute`, `CanHandle`)
- Unexported (private): camelCase (e.g., `checkInternet`, `cleanMarkdown`, `parseLogLevel`)
- Constructor pattern: `New{TypeName}` (e.g., `NewLogger`, `NewDependencies`, `NewOrchestrator`)
- Predicates: `Can{Action}` or `Is{Property}` (e.g., `CanHandle`, `IsRetryable`)
- Package-level constants: UPPER_SNAKE_CASE (e.g., `MinFormatVersion`)
- Configuration structs: PascalCase (e.g., `Config`, `LoggerOptions`, `DependencyOptions`)
- Options params: `{Name}Options` suffix (e.g., `OrchestratorOptions`, `WriterOptions`, `LoggerOptions`)
- Sentinel errors: `Err{Name}` prefix (e.g., `ErrNotFound`, `ErrCacheMiss`, `ErrInvalidURL`)
- Interface types: `{Name}` with -er suffix for behavior (e.g., `Fetcher`, `Renderer`, `Strategy`)
- Structs: PascalCase (e.g., `Document`, `FetchError`, `Dependencies`)
- Interfaces: PascalCase, typically -er suffix (e.g., `Strategy`, `Fetcher`, `Cache`, `Logger`)
- Custom error types: `{Name}Error` (e.g., `FetchError`, `ValidationError`, `StrategyError`)
- Config/options: `{Name}Config` or `{Name}Options` (e.g., `CacheConfig`, `LoggerOptions`)
## Code Style
- Tool: `gofmt` (Go standard formatter)
- Run: `make fmt` (applies gofmt with -s flag for simplification)
- Tool: `golangci-lint` v2
- Config: `.golangci.yml`
- Key rules enabled:
- Exclusions: Known issues noted in config (e.g., ineffectual assignment in `internal/converter/encoding.go`)
- Run: `make lint`
- No explicit limit enforced; follows Go conventions (80-100 char preferred for readability)
## Import Organization
## Error Handling
## Logging
- `Logger` struct wraps `zerolog.Logger`
- Methods: `Info()`, `Warn()`, `Error()`, `Debug()` (chainable API)
- Example: `logger.Info().Str("key", val).Msg("message")`
## Comments
- Complex algorithms or non-obvious logic
- Public API exports (functions, types, interfaces)
- Important design decisions or gotchas
- Section headers for logical groupings
- Deprecation notices with alternatives
- Examples of typical usage
- Comment immediately precedes the declaration
- Starts with the name of the thing being documented
- Example from `internal/domain/errors.go`:
## Function Design
- Use options structs for 3+ parameters (e.g., `LoggerOptions`, `ClientOptions`)
- Context as first parameter for functions doing async work
- Testing param `t *testing.T` as first param in test functions
- Error as last return value (Go convention)
- Use named returns sparingly (only for clarity or cleanup)
- Return concrete types when possible (not interfaces, except for injection)
## Module Design
- Functions and types intended for external use
- Must be documented with Go comments
- Examples: `NewOrchestrator`, `Config`, `Strategy` interface
- Functions and types for internal use within package
- No documentation requirement (but helpful for complex logic)
- Examples: `checkInternet()`, `cleanMarkdown()`, helper functions
- `internal/`: All non-public packages (interfaces, implementations, utilities)
- `cmd/`: Entry points (CLI commands)
- `pkg/`: Potentially reusable packages (currently only `version`)
- `tests/`: Test infrastructure (helpers, fixtures, test suites)
## Variable Scoping
- Pre-compiled regex patterns: `var linkRegex = regexp.MustCompile(...)`
- Sentinel errors: `var ErrNotFound = errors.New(...)`
- Test-injectable functions in main: `var osStat = os.Stat`, `var execLookPath = exec.LookPath`
- Cobra commands: `var rootCmd = &cobra.Command{...}`
- Context passed as first parameter to async functions
- Never store context in structs (except for cleanup/shutdown)
- Use `context.WithCancel()` for graceful shutdown patterns
- Example from `cmd/repodocs/main.go`:
## Interface Design
- Location: `internal/strategies/strategy.go`
- Interface definition:
- Multiple concrete implementations (Crawler, Sitemap, Git, etc.)
- `Fetcher`: HTTP fetching with caching
- `Renderer`: JavaScript rendering (browser automation)
- `Cache`: Key-value cache abstraction
- `LLMProvider`: External LLM service abstraction
<!-- GSD:conventions-end -->

<!-- GSD:architecture-start source:ARCHITECTURE.md -->
## Architecture

## Pattern Overview
- Multiple extraction strategies (8 types) selected via URL pattern detection
- Pluggable components (Fetcher, Renderer, Converter, Writer, Cache, LLM)
- Linear processing pipeline: Detect → Fetch → Render → Convert → Write
- Concurrent processing with sync state management for incremental runs
- Built-in support for caching, stealth mode, and JavaScript rendering
## Layers
- Location: `cmd/repodocs/main.go`
- Contains: Cobra command definitions, config binding, flag parsing
- Depends on: `internal/app`, `internal/config`, `internal/tui`
- Used by: End-user command-line interaction
- Location: `internal/app/orchestrator.go`
- Contains: `Orchestrator` struct that coordinates extraction flow
- Depends on: Strategy system, Dependencies container
- Used by: CLI entry points, manifest runner
- Key responsibility: Strategy detection, initialization, execution, and state management
- Location: `internal/strategies/`
- Contains: LLMS, PkgGo, DocsRS, Sitemap, Wiki, GitHubPages, Git, Crawler
- Depends on: Domain interfaces, Fetcher, Converter, Writer
- Used by: Orchestrator
- Pattern: Each implements `strategies.Strategy` interface with `Name()`, `CanHandle()`, `Execute()`
- Location: `internal/domain/interfaces.go`
- Defines: Fetcher, Renderer, Cache, Converter, Writer, LLMProvider
- Purpose: Abstraction layer allowing swappable implementations
- No dependencies on concrete implementations
- Location: `internal/fetcher/`
- Contains: TLS-based stealth client using `bogdanfinn/tls-client` (Chrome_131 profile)
- Features: Retry logic, user-agent rotation, cookie handling
- Dependencies: `bogdanfinn/tls-client`, `bogdanfinn/fhttp`
- Location: `internal/renderer/`
- Contains: Rod-based browser automation with stealth scripts
- Features: Lazy loading, network idle wait, element visibility wait
- Dependencies: `go-rod/rod`, `go-rod/stealth`
- Optional: Enabled when JavaScript rendering needed or forced
- Location: `internal/cache/`
- Contains: Badger-based key-value cache with TTL support
- Features: Per-URL caching with content hash tracking
- Dependencies: `dgraph-io/badger/v4`
- Location: `internal/converter/`
- Contains: Multi-stage pipeline (UTF-8 → Extract → Sanitize → Convert → Metadata)
- Features: CSS selector-based extraction, Readability fallback, code language preservation
- Dependencies: `PuerkitoBio/goquery`, `go-shiori/go-readability`, `JohannesKaufmann/html-to-markdown`
- Location: `internal/output/`
- Contains: File writer with metadata collection
- Features: Flat or hierarchical output, frontmatter injection, metadata JSON generation
- Used by: All strategies
- Location: `internal/state/`
- Contains: `Manager` tracking processed URLs, content hashes, and sync state
- Features: Incremental sync (skip unchanged), full sync, pruning (delete removed files)
- State file: `.repodocs-state.json` in output directory
- Location: `internal/config/`
- Contains: Viper-based config with YAML file support
- Features: Flag binding, environment override, defaults
- Config file: `~/.repodocs/config.yaml`
- Location: `internal/llm/`
- Contains: Provider wrappers (OpenAI, Anthropic, Google, Ollama)
- Features: Metadata enhancement (summary, tags, category), rate limiting, circuit breaker
- Optional: Enabled when `enhance_metadata: true`
## Data Flow
- `Orchestrator` passes control to strategy
- Strategy manages concurrency (workers count from config)
- Each strategy batches URLs and processes using worker pool pattern
- Progress bars track completion (progressbar/v3)
- Context cancellation enables graceful shutdown (SIGINT/SIGTERM)
- **First Run:** Process all discovered URLs, create state file
- **Incremental Sync** (`--sync`): Skip URLs with unchanged content hash
- **Full Sync** (`--full-sync`): Reprocess all, ignore state
- **Prune** (`--prune`): Delete files for URLs no longer in source
## Key Abstractions
- Purpose: Pluggable extraction algorithms
- Examples: `internal/strategies/crawler.go`, `internal/strategies/llms.go`, `internal/strategies/git/strategy.go`
- Pattern: Each strategy handles different source types with custom discovery/processing logic
- Purpose: Shared state and factories for strategies
- Location: `internal/strategies/strategy.go` (type `Dependencies`)
- Manages: Fetcher, Renderer, Converter, Writer, Cache, Logger, LLMProvider
- Lazy initialization: Renderer created on first use
- Purpose: Composable HTML-to-Markdown transformation
- Location: `internal/converter/pipeline.go`
- Stages: Encoding → Extraction → Sanitization → Markdown → Metadata
- Extensible: Each stage can be customized independently
- Purpose: Functional configuration without tight coupling
- Examples: `fetcher.ClientOptions`, `converter.PipelineOptions`, `state.ManagerOptions`
- Reduces constructor parameter count, improves testability
## Entry Points
- Location: `cmd/repodocs/main.go`
- Triggers: User runs `repodocs [url]` or `repodocs --manifest [path]`
- Responsibilities: Flag parsing, config loading, orchestrator creation, error handling
```go
```
```go
```
- Reads manifest file (YAML/JSON) with source list
- Iterates sources, runs orchestrator for each
- Continues on error if configured
- Location: `cmd/repodocs/main.go` (`doctorCmd`)
- Validates: Internet, Chrome/Chromium, write permissions, config, cache dir
- `config edit`: TUI-based interactive editor (`internal/tui/`)
- `config show`: Display YAML config
- `config init`: Create default config at `~/.repodocs/config.yaml`
- `config path`: Print config file location
## Error Handling
## Cross-Cutting Concerns
- Framework: `rs/zerolog`
- Level: Debug, Info, Warn, Error
- Format: Structured (JSON or pretty)
- Verbose flag enables debug level
- URL validation at entry (`orchestrator.ValidateURL()`)
- Strategy type validation via `IsValidStrategy()`
- Config validation in `config.Load()`
- Manifest schema validation in `manifest.Loader`
- HTTP: User-Agent rotation, TLS fingerprinting (stealth mode)
- Git: SSH keys via `go-git/v5`
- LLM: API key from config (env var or file)
- No hardcoded credentials
- HTTP: No built-in limit (relies on target server)
- LLM: Token bucket with configurable RPM, burst, backoff
- Circuit breaker with configurable failure threshold
- Location: `internal/llm/circuit_breaker.go`, `internal/llm/ratelimit.go`
- Worker pool pattern (configurable count)
- Context-based cancellation
- Thread-safe state via `sync.Map`, `sync.Mutex`
- Progress bar updates with mutex protection
<!-- GSD:architecture-end -->

<!-- GSD:skills-start source:skills/ -->
## Project Skills

No project skills found. Add skills to any of: `.claude/skills/`, `.agents/skills/`, `.cursor/skills/`, or `.github/skills/` with a `SKILL.md` index file.
<!-- GSD:skills-end -->

<!-- GSD:workflow-start source:GSD defaults -->
## GSD Workflow Enforcement

Before using Edit, Write, or other file-changing tools, start work through a GSD command so planning artifacts and execution context stay in sync.

Use these entry points:
- `/gsd-quick` for small fixes, doc updates, and ad-hoc tasks
- `/gsd-debug` for investigation and bug fixing
- `/gsd-execute-phase` for planned phase work

Do not make direct repo edits outside a GSD workflow unless the user explicitly asks to bypass it.
<!-- GSD:workflow-end -->

<!-- GSD:profile-start -->
## Developer Profile

> Profile not yet configured. Run `/gsd-profile-user` to generate your developer profile.
> This section is managed by `generate-claude-profile` -- do not edit manually.
<!-- GSD:profile-end -->

<!-- gitnexus:start -->
# GitNexus — Code Intelligence

This project is indexed by GitNexus as **repodocs-go** (9636 symbols, 24639 relationships, 292 execution flows). Use the GitNexus MCP tools to understand code, assess impact, and navigate safely.

> If any GitNexus tool warns the index is stale, run `npx gitnexus analyze` in terminal first.

## Always Do

- **MUST run impact analysis before editing any symbol.** Before modifying a function, class, or method, run `gitnexus_impact({target: "symbolName", direction: "upstream"})` and report the blast radius (direct callers, affected processes, risk level) to the user.
- **MUST run `gitnexus_detect_changes()` before committing** to verify your changes only affect expected symbols and execution flows.
- **MUST warn the user** if impact analysis returns HIGH or CRITICAL risk before proceeding with edits.
- When exploring unfamiliar code, use `gitnexus_query({query: "concept"})` to find execution flows instead of grepping. It returns process-grouped results ranked by relevance.
- When you need full context on a specific symbol — callers, callees, which execution flows it participates in — use `gitnexus_context({name: "symbolName"})`.

## Never Do

- NEVER edit a function, class, or method without first running `gitnexus_impact` on it.
- NEVER ignore HIGH or CRITICAL risk warnings from impact analysis.
- NEVER rename symbols with find-and-replace — use `gitnexus_rename` which understands the call graph.
- NEVER commit changes without running `gitnexus_detect_changes()` to check affected scope.

## Resources

| Resource | Use for |
|----------|---------|
| `gitnexus://repo/repodocs-go/context` | Codebase overview, check index freshness |
| `gitnexus://repo/repodocs-go/clusters` | All functional areas |
| `gitnexus://repo/repodocs-go/processes` | All execution flows |
| `gitnexus://repo/repodocs-go/process/{name}` | Step-by-step execution trace |

## CLI

| Task | Read this skill file |
|------|---------------------|
| Understand architecture / "How does X work?" | `.claude/skills/gitnexus/gitnexus-exploring/SKILL.md` |
| Blast radius / "What breaks if I change X?" | `.claude/skills/gitnexus/gitnexus-impact-analysis/SKILL.md` |
| Trace bugs / "Why is X failing?" | `.claude/skills/gitnexus/gitnexus-debugging/SKILL.md` |
| Rename / extract / split / refactor | `.claude/skills/gitnexus/gitnexus-refactoring/SKILL.md` |
| Tools, resources, schema reference | `.claude/skills/gitnexus/gitnexus-guide/SKILL.md` |
| Index, status, clean, wiki CLI commands | `.claude/skills/gitnexus/gitnexus-cli/SKILL.md` |

<!-- gitnexus:end -->
