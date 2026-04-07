# Codebase Structure

**Analysis Date:** 2026-04-07

## Directory Layout

```
repodocs/
├── cmd/                    # CLI entry point
│   └── repodocs/          # Main executable package
├── internal/              # Core application logic (not exported)
│   ├── app/               # Orchestration and strategy detection
│   ├── cache/             # Caching implementation (Badger)
│   ├── config/            # Configuration loading and defaults
│   ├── converter/         # HTML to Markdown conversion pipeline
│   ├── domain/            # Interfaces and domain models
│   ├── fetcher/           # HTTP client with stealth features
│   ├── git/               # Git utilities for strategies
│   ├── llm/               # LLM provider implementations
│   ├── manifest/          # Manifest file parsing (YAML/JSON)
│   ├── output/            # Output writing and metadata collection
│   ├── renderer/          # JavaScript rendering (Rod-based)
│   ├── state/             # Sync state management
│   ├── strategies/        # Strategy implementations (8 types)
│   │   └── git/          # Git-specific strategy implementation
│   ├── tui/               # Terminal UI for config editing
│   └── utils/             # Utilities (logging, path helpers, progress)
├── pkg/                   # Public packages
│   └── version/           # Version constants
├── tests/                 # Test suite
│   ├── unit/              # Fast unit tests (no I/O)
│   ├── integration/       # Network-dependent tests
│   ├── e2e/               # Full CLI integration tests
│   ├── benchmark/         # Performance benchmarks
│   ├── fixtures/          # Test HTML/data snapshots
│   ├── testdata/          # Golden files and config samples
│   ├── helpers/           # Test utilities
│   └── mocks/             # Generated mocks (uber/mock)
├── configs/               # Example configuration files
├── examples/              # Example manifests
├── scripts/               # Build and utility scripts
├── build/                 # Compiled binaries (gitignored)
├── go.mod                 # Module definition (Go 1.24.1)
└── go.sum                 # Dependency checksums
```

## Directory Purposes

**cmd/repodocs:**
- Purpose: CLI application entry point
- Contains: Command definitions (root, doctor, version, config, config subcommands)
- Key files: `main.go` (600+ lines), `main_test.go`
- Responsibilities: Flag parsing, subcommand routing, graceful shutdown, error formatting

**internal/app:**
- Purpose: Core orchestration logic
- Contains: `Orchestrator` (coordinates strategies), strategy detection, factory
- Key files: `orchestrator.go` (300+ lines), `detector.go` (strategy selection)
- Responsibilities: Strategy selection, strategy execution, manifest processing

**internal/strategies:**
- Purpose: URL-specific extraction algorithms
- Contains: 8 strategy implementations + discovery utilities
- Key files: 
  - `strategy.go` (Dependencies, Options, Strategy interface)
  - `crawler.go` (web crawling with colly)
  - `sitemap.go` (XML parsing)
  - `llms.go` (llms.txt parsing)
  - `pkggo.go` (pkg.go.dev scraping)
  - `docsrs.go` (docs.rs Rust documentation)
  - `wiki.go` (wiki format parsing)
  - `github_pages.go` (GitHub Pages static sites)
  - `git/strategy.go` (Git repository cloning/archiving)
- Responsibilities: URL validation, content discovery, content processing

**internal/strategies/git:**
- Purpose: Git-specific extraction logic
- Contains: Clone, archive, parser, processor implementations
- Key files: `strategy.go`, `fetcher.go`, `parser.go`, `processor.go`
- Uses: `go-git/v5` for repository operations

**internal/converter:**
- Purpose: HTML to Markdown transformation pipeline
- Contains: Multi-stage conversion with sanitization
- Key files:
  - `pipeline.go` (orchestrates all stages)
  - `encoding.go` (UTF-8 normalization)
  - `markdown.go` (markdown generation)
  - `readability.go` (content extraction fallback)
  - `sanitizer.go` (HTML cleaning)
  - `code_blocks.go` (code language preservation)
- Dependencies: `goquery`, `html-to-markdown`, `go-readability`

**internal/fetcher:**
- Purpose: HTTP client with stealth capabilities
- Contains: TLS-based client, retry logic, cookie handling
- Key files: 
  - `client.go` (main client)
  - `stealth.go` (user-agent rotation)
  - `transport.go` (HTTP transport wrapper)
  - `retry.go` (exponential backoff)
- Uses: `bogdanfinn/tls-client` (Chrome_131 profile)

**internal/renderer:**
- Purpose: JavaScript rendering for SPAs
- Contains: Rod-based browser automation
- Key files: 
  - `rod.go` (main renderer)
  - `stealth.go` (stealth scripts)
  - `pool.go` (browser pool management)
  - `detector.go` (Chrome detection)
- Uses: `go-rod/rod`, `go-rod/stealth`

**internal/cache:**
- Purpose: Content caching with TTL
- Contains: Badger-based key-value store
- Key files: 
  - `badger.go` (implementation)
  - `interface.go` (Cache interface)
  - `keys.go` (cache key generation)

**internal/config:**
- Purpose: Configuration management
- Contains: Viper integration, YAML defaults, flag binding
- Key files: 
  - `config.go` (Config struct)
  - `defaults.go` (Default() function)
  - `loader.go` (Load() function)
- Config file: `~/.repodocs/config.yaml`
- Sections: output, concurrency, cache, rendering, stealth, logging, llm, git

**internal/domain:**
- Purpose: Interfaces and core models (no implementations)
- Contains: Strategy, Fetcher, Renderer, Cache, Converter, Writer, LLMProvider interfaces
- Models: Document, Page, Sitemap, LLMSLink, Metadata, Frontmatter
- Options: CommonOptions, StrategyOptions, RenderOptions
- Dependencies: None on concrete implementations

**internal/output:**
- Purpose: File writing and metadata collection
- Contains: Writer, MetadataCollector
- Key files: 
  - `writer.go` (file writing with path generation)
  - `collector.go` (JSON metadata aggregation)

**internal/state:**
- Purpose: Incremental sync state management
- Contains: State tracker for URLs and hashes
- Key files: 
  - `manager.go` (Load, Save, ShouldProcess)
  - `models.go` (SyncState, PageState)
- State file: `.repodocs-state.json` in output directory

**internal/llm:**
- Purpose: LLM provider integration
- Contains: OpenAI, Anthropic, Google, Ollama providers
- Key files: 
  - `provider.go` (Provider interface)
  - `provider_wrapper.go` (wrapper with rate limit/circuit breaker)
  - `openai.go`, `anthropic.go`, `google.go`, `ollama.go` (implementations)
  - `metadata.go` (MetadataEnhancer)
  - `ratelimit.go` (token bucket)
  - `circuit_breaker.go` (failure handling)

**internal/manifest:**
- Purpose: Manifest file parsing
- Contains: YAML/JSON manifest loader
- Key files: `loader.go`
- Supports: Batch processing of multiple sources

**internal/tui:**
- Purpose: Terminal UI for configuration
- Contains: Interactive config editor
- Uses: `charmbracelet/huh` (form builder), `bubbletea` (TUI framework)

**internal/utils:**
- Purpose: Shared utilities
- Contains: 
  - Logger (rs/zerolog wrapper)
  - Path generation (URL → file path)
  - Domain checking (same-domain validation)
  - Progress bars (schollz/progressbar)
  - File utilities
- Key files: Logger, path generation functions

**pkg/version:**
- Purpose: Version information
- Contains: Version constants and formatting
- Used by: `version` command, binary metadata

**tests/unit:**
- Purpose: Fast unit tests (no I/O, mocked dependencies)
- Contains: Tests for each internal package
- Follows: `tests/unit/{package}/{module}_test.go` pattern
- Uses: `testify/assert`, `testify/mock`, `uber/mock`

**tests/integration:**
- Purpose: Network-dependent tests
- Contains: Tests that use real HTTP, file I/O, or external services
- Mirrors: Internal package structure
- Examples: Fetcher with real HTTP, Cache with real Badger, Git operations

**tests/e2e:**
- Purpose: Full CLI integration tests
- Contains: Tests that run binary end-to-end
- Examples: `crawl_test.go`, `sitemap_test.go`, `config_test.go`
- Validates: CLI flags, config loading, strategy selection

**tests/benchmark:**
- Purpose: Performance benchmarks
- Contains: `git_clone_benchmark_test.go`
- Run with: `make test-bench`

**tests/fixtures/**
- Purpose: Test data snapshots
- Contains: Golden HTML files for strategies
  - `fixtures/git/` — Git repo samples
  - `fixtures/pkggo/` — pkg.go.dev HTML
  - `fixtures/docsrs/` — docs.rs HTML
  - `fixtures/llms/` — llms.txt samples

**tests/testdata/**
- Purpose: Configuration and data for tests
- Contains: 
  - `testdata/config/` — Sample config files
  - `testdata/golden/` — Expected output for regression
  - `testdata/fixtures/` — Additional test files

**tests/helpers/**
- Purpose: Shared test utilities
- Key files: `fixtures.go`, `http.go`
- Provides: HTTP test server, mock builders

**tests/mocks/**
- Purpose: Generated mock implementations
- Generated by: `go generate` using `uber/mock`
- Folder structure: Mirrors internal package hierarchy

**configs/**
- Purpose: Example configuration files
- Contains: Default config templates

**examples/**
- Purpose: Example manifest files
- Contains: Sample manifests for batch processing

**scripts/**
- Purpose: Build and development utilities
- Contains: Build scripts, test runners

**build/**
- Purpose: Compiled binaries (gitignored)
- Created by: `make build`
- Output: `./build/repodocs` (Go binary)

## Key File Locations

**Entry Points:**
- `cmd/repodocs/main.go`: CLI application root
  - `main()`: Entry point
  - `rootCmd`: Root Cobra command with all flags
  - `run()`: Handler for URL processing
  - `runManifest()`: Handler for manifest processing

**Core Logic:**
- `internal/app/orchestrator.go`: Main orchestration logic
  - `NewOrchestrator()`: Create orchestrator
  - `Run()`: Process single URL
  - `RunManifest()`: Process manifest
- `internal/app/detector.go`: Strategy detection
  - `DetectStrategy()`: URL → strategy type
  - `CreateStrategy()`: Factory function

**Strategy System:**
- `internal/strategies/strategy.go`: Base interfaces and dependencies
  - `Strategy` interface
  - `Dependencies` container
- `internal/strategies/crawler.go`: Web crawler
- `internal/strategies/git/strategy.go`: Git repository handler
- `internal/strategies/sitemap.go`: XML sitemap processor
- `internal/strategies/pkggo.go`: pkg.go.dev handler
- `internal/strategies/docsrs.go`: docs.rs handler
- `internal/strategies/llms.go`: llms.txt handler
- `internal/strategies/wiki.go`: Wiki parser
- `internal/strategies/github_pages.go`: GitHub Pages detector

**Configuration:**
- `internal/config/config.go`: Config struct definitions
- `internal/config/defaults.go`: Default config values
- `internal/config/loader.go`: Load from file/environment
- Config file: `~/.repodocs/config.yaml`

**Conversion Pipeline:**
- `internal/converter/pipeline.go`: Main orchestrator
  - `Convert()`: HTML → Document

**Testing:**
- `tests/unit/...`: Unit tests (fast, mocked)
- `tests/integration/...`: Integration tests (network I/O)
- `tests/e2e/...`: End-to-end tests (CLI)
- `tests/helpers/fixtures.go`: Test data builders
- `tests/mocks/...`: Generated mocks

## Naming Conventions

**Files:**
- `*_test.go`: Unit tests for package
- `*_integration_test.go`: Integration tests
- `interface.go` or `{name}_interface.go`: Interface definitions
- `types.go`: Domain type definitions
- `errors.go`: Error types

**Directories:**
- `internal/{package}`: Unexported packages
- `pkg/{package}`: Public packages
- `tests/{type}/{package}`: Mirror internal structure
- `examples/manifests`: Sample manifests

**Functions:**
- `New{Type}()`: Constructors
- `{Receiver}_{Action}()`: Methods
- `Is{Condition}()`: Boolean checks
- `Get{Property}()`, `Set{Property}()`: Accessors
- Private functions: lowercase first letter

**Interfaces:**
- `{Name}` (noun, often ending in "er": Reader, Writer, Fetcher)
- Empty interface methods discouraged
- Cohesive single-responsibility interfaces

**Types:**
- `{Name}Config` or `{Name}Options`: Configuration structs
- Exported types: PascalCase
- Unexported types: camelCase

## Where to Add New Code

**New Extraction Strategy:**
1. Create `internal/strategies/{name}.go` implementing `Strategy` interface
2. Add constructor `NewStrategy(deps)` 
3. Register in `app/detector.go` → `CreateStrategy()` function
4. Add URL detection logic to `DetectStrategy()` in `app/detector.go`
5. Add tests: `tests/unit/strategies/{name}_test.go` + `tests/integration/strategies/{name}_integration_test.go`
6. Update `GetAllStrategies()` function if needed

**New LLM Provider:**
1. Create `internal/llm/{provider}.go` implementing `LLMProvider` interface
2. Add constructor following pattern of `openai.go`, `anthropic.go`
3. Add provider name constant
4. Register in `config/defaults.go` as option
5. Add tests: `tests/unit/llm/{provider}_test.go`
6. Update `internal/llm/provider.go` factory if needed

**New Config Section:**
1. Add struct to `internal/config/config.go` (e.g., `NewFeatureConfig`)
2. Add to `Config` struct with `mapstructure` and `yaml` tags
3. Add defaults to `internal/config/defaults.go` → `Default()`
4. Bind flags in `cmd/repodocs/main.go` if CLI flag needed
5. Update TUI in `internal/tui/` if interactive editing needed
6. Add tests: `tests/unit/config/config_test.go`

**New Converter Stage:**
1. Add method to `internal/converter/pipeline.go`
2. Call from `Convert()` pipeline
3. Create dedicated file if large: `internal/converter/{stage}.go`
4. Add interface if pluggable (e.g., `Sanitizer`, `ExtractContent`)
5. Add tests: `tests/unit/converter/{stage}_test.go`

**New Output Format:**
1. Extend `internal/output/writer.go` or create writer variant
2. Add format option to `OutputConfig` in `internal/config/config.go`
3. Update `Write()` method with format handling
4. Add tests for format output

**Utilities:**
- General helpers: `internal/utils/` (string, path, domain checks)
- Shared test helpers: `tests/helpers/`
- Shared fixtures: `tests/fixtures/`
- Type-specific utilities: Keep in relevant package

## Special Directories

**build/:**
- Purpose: Compiled binaries
- Generated: By `make build`
- Committed: No (.gitignored)
- Contents: `./repodocs` executable

**.planning/:**
- Purpose: Planning and analysis documents
- Generated: By GSD analysis tools
- Committed: Yes (tracking tool evolves)
- Contents: ARCHITECTURE.md, STRUCTURE.md, CONCERNS.md, etc.

**internal/domain/:**
- Purpose: Interfaces-only package (no implementations)
- Special: No dependencies on other internal packages (dependency inversion)
- Rationale: Enables all other packages to import without cycles

**tests/mocks/:**
- Purpose: Generated mock implementations
- Generated: By `go generate` using `uber/mock`
- Committed: Yes (predictable, part of test infrastructure)
- Regenerate: `go generate ./...`

**configs/**
- Purpose: Example configurations
- Used: As templates for `~/.repodocs/config.yaml`

**examples/manifests/**
- Purpose: Sample manifest files
- Used: For documentation and testing

## Key Dependencies Structure

```
cmd/repodocs
  ↓
internal/app (Orchestrator)
  ↓
internal/strategies/{*}
  ↓
internal/domain (interfaces)
  ↓
internal/fetcher
internal/renderer
internal/converter
internal/output
internal/cache
internal/llm
internal/state
internal/config
internal/utils

No package imports internal/domain:
- Domain defines interfaces
- All implementations depend on domain
- Zero up-dependencies in domain (dependency inversion)
```

---

*Structure analysis: 2026-04-07*
