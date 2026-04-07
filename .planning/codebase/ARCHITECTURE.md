# Architecture

**Analysis Date:** 2026-04-07

## Pattern Overview

**Overall:** Strategy Pattern + Dependency Injection with layered pipeline architecture

**Key Characteristics:**
- Multiple extraction strategies (8 types) selected via URL pattern detection
- Pluggable components (Fetcher, Renderer, Converter, Writer, Cache, LLM)
- Linear processing pipeline: Detect → Fetch → Render → Convert → Write
- Concurrent processing with sync state management for incremental runs
- Built-in support for caching, stealth mode, and JavaScript rendering

## Layers

**Presentation (CLI):**
- Location: `cmd/repodocs/main.go`
- Contains: Cobra command definitions, config binding, flag parsing
- Depends on: `internal/app`, `internal/config`, `internal/tui`
- Used by: End-user command-line interaction

**Orchestration:**
- Location: `internal/app/orchestrator.go`
- Contains: `Orchestrator` struct that coordinates extraction flow
- Depends on: Strategy system, Dependencies container
- Used by: CLI entry points, manifest runner
- Key responsibility: Strategy detection, initialization, execution, and state management

**Strategy Implementations (8 types):**
- Location: `internal/strategies/`
- Contains: LLMS, PkgGo, DocsRS, Sitemap, Wiki, GitHubPages, Git, Crawler
- Depends on: Domain interfaces, Fetcher, Converter, Writer
- Used by: Orchestrator
- Pattern: Each implements `strategies.Strategy` interface with `Name()`, `CanHandle()`, `Execute()`

**Domain Interfaces:**
- Location: `internal/domain/interfaces.go`
- Defines: Fetcher, Renderer, Cache, Converter, Writer, LLMProvider
- Purpose: Abstraction layer allowing swappable implementations
- No dependencies on concrete implementations

**Fetcher (HTTP Client):**
- Location: `internal/fetcher/`
- Contains: TLS-based stealth client using `bogdanfinn/tls-client` (Chrome_131 profile)
- Features: Retry logic, user-agent rotation, cookie handling
- Dependencies: `bogdanfinn/tls-client`, `bogdanfinn/fhttp`

**Renderer (JavaScript Execution):**
- Location: `internal/renderer/`
- Contains: Rod-based browser automation with stealth scripts
- Features: Lazy loading, network idle wait, element visibility wait
- Dependencies: `go-rod/rod`, `go-rod/stealth`
- Optional: Enabled when JavaScript rendering needed or forced

**Cache (Content Storage):**
- Location: `internal/cache/`
- Contains: Badger-based key-value cache with TTL support
- Features: Per-URL caching with content hash tracking
- Dependencies: `dgraph-io/badger/v4`

**Converter (HTML to Markdown):**
- Location: `internal/converter/`
- Contains: Multi-stage pipeline (UTF-8 → Extract → Sanitize → Convert → Metadata)
- Features: CSS selector-based extraction, Readability fallback, code language preservation
- Dependencies: `PuerkitoBio/goquery`, `go-shiori/go-readability`, `JohannesKaufmann/html-to-markdown`

**Output Writer:**
- Location: `internal/output/`
- Contains: File writer with metadata collection
- Features: Flat or hierarchical output, frontmatter injection, metadata JSON generation
- Used by: All strategies

**State Management:**
- Location: `internal/state/`
- Contains: `Manager` tracking processed URLs, content hashes, and sync state
- Features: Incremental sync (skip unchanged), full sync, pruning (delete removed files)
- State file: `.repodocs-state.json` in output directory

**Configuration:**
- Location: `internal/config/`
- Contains: Viper-based config with YAML file support
- Features: Flag binding, environment override, defaults
- Config file: `~/.repodocs/config.yaml`

**LLM Integration:**
- Location: `internal/llm/`
- Contains: Provider wrappers (OpenAI, Anthropic, Google, Ollama)
- Features: Metadata enhancement (summary, tags, category), rate limiting, circuit breaker
- Optional: Enabled when `enhance_metadata: true`

## Data Flow

**Standard URL Processing:**

1. **Detection** (`app/detector.go`)
   - Parse URL to determine strategy type
   - Order: LLMS → PkgGo → DocsRS → Sitemap → Wiki → GitHubPages → Git → Crawler

2. **Strategy-Specific Discovery** (varies)
   - Some strategies auto-discover resources (e.g., Crawler discovers sitemaps)
   - Strategies return list of URLs to process

3. **Fetch** (`internal/fetcher/`)
   - Check cache first if enabled
   - Use stealth client with TLS fingerprinting
   - Return raw response (cached or fresh)

4. **Render** (optional, `internal/renderer/`)
   - If `RenderJS` flag or strategy requires JS execution
   - Use Rod to render page, wait for stability
   - Return rendered HTML

5. **Convert** (`internal/converter/pipeline.go`)
   - Step 1: UTF-8 normalization
   - Step 2: Parse HTML
   - Step 3: Extract main content (CSS selector or Readability)
   - Step 4: Sanitize (remove scripts, nav, ads)
   - Step 5: Convert to Markdown
   - Step 6: Extract metadata (headers, links, word count)

6. **Enhance (optional)** (`internal/llm/`)
   - If `enhance_metadata` enabled
   - Send content to LLM provider
   - Extract summary, tags, category

7. **Write** (`internal/output/`)
   - Check if file exists (skip unless `--force`)
   - Generate output path (hierarchical or flat)
   - Write Markdown with YAML frontmatter
   - Collect metadata if `json_metadata` enabled

8. **State Update** (`internal/state/`)
   - Store URL, content hash, file path
   - Mark URL as processed (for pruning)
   - Save `.repodocs-state.json`

**Concurrent Processing:**

- `Orchestrator` passes control to strategy
- Strategy manages concurrency (workers count from config)
- Each strategy batches URLs and processes using worker pool pattern
- Progress bars track completion (progressbar/v3)
- Context cancellation enables graceful shutdown (SIGINT/SIGTERM)

**State Management:**

- **First Run:** Process all discovered URLs, create state file
- **Incremental Sync** (`--sync`): Skip URLs with unchanged content hash
- **Full Sync** (`--full-sync`): Reprocess all, ignore state
- **Prune** (`--prune`): Delete files for URLs no longer in source

## Key Abstractions

**Strategy Interface:**
- Purpose: Pluggable extraction algorithms
- Examples: `internal/strategies/crawler.go`, `internal/strategies/llms.go`, `internal/strategies/git/strategy.go`
- Pattern: Each strategy handles different source types with custom discovery/processing logic

**Dependencies Container:**
- Purpose: Shared state and factories for strategies
- Location: `internal/strategies/strategy.go` (type `Dependencies`)
- Manages: Fetcher, Renderer, Converter, Writer, Cache, Logger, LLMProvider
- Lazy initialization: Renderer created on first use

**Converter Pipeline:**
- Purpose: Composable HTML-to-Markdown transformation
- Location: `internal/converter/pipeline.go`
- Stages: Encoding → Extraction → Sanitization → Markdown → Metadata
- Extensible: Each stage can be customized independently

**Options Pattern:**
- Purpose: Functional configuration without tight coupling
- Examples: `fetcher.ClientOptions`, `converter.PipelineOptions`, `state.ManagerOptions`
- Reduces constructor parameter count, improves testability

## Entry Points

**CLI:**
- Location: `cmd/repodocs/main.go`
- Triggers: User runs `repodocs [url]` or `repodocs --manifest [path]`
- Responsibilities: Flag parsing, config loading, orchestrator creation, error handling

**Single URL Processing:**
```go
url := args[0]  // From CLI argument
orchestrator.Run(ctx, url, opts)
```

**Manifest Processing:**
```go
orchestrator.RunManifest(ctx, manifestConfig, opts)
```
- Reads manifest file (YAML/JSON) with source list
- Iterates sources, runs orchestrator for each
- Continues on error if configured

**Doctor Command:**
- Location: `cmd/repodocs/main.go` (`doctorCmd`)
- Validates: Internet, Chrome/Chromium, write permissions, config, cache dir

**Config Commands:**
- `config edit`: TUI-based interactive editor (`internal/tui/`)
- `config show`: Display YAML config
- `config init`: Create default config at `~/.repodocs/config.yaml`
- `config path`: Print config file location

## Error Handling

**Strategy:** Multiple error types with recovery paths

**Patterns:**

1. **Cached Fallback:**
   - CSS selector extraction fails → Fall back to Readability
   - Example: `internal/converter/pipeline.go` line 89-100

2. **Retry with Backoff:**
   - HTTP request fails → Retry up to 3 times with exponential backoff
   - Location: `internal/fetcher/retry.go`
   - Initial: 1 second, cap: 1 minute

3. **Strategy Fallback (Crawler only):**
   - Sitemap discovery fails → Continue with crawling
   - Location: `internal/app/orchestrator.go` line 154-165

4. **Graceful Degradation:**
   - JavaScript rendering unavailable → Render without JS
   - LLM enhancement disabled → Skip metadata enhancement
   - Progress bar disabled → Continue silently

5. **Context Cancellation:**
   - SIGINT/SIGTERM → Cancel context, finish in-flight requests
   - Ensures clean state file saves
   - Location: `cmd/repodocs/main.go` line 166-178

6. **State Recovery:**
   - Version mismatch → Discard old state, rebuild
   - Corrupted state file → ErrStateCorrupted, start fresh
   - Location: `internal/state/manager.go` line 65-73

## Cross-Cutting Concerns

**Logging:** 
- Framework: `rs/zerolog`
- Level: Debug, Info, Warn, Error
- Format: Structured (JSON or pretty)
- Verbose flag enables debug level

**Validation:**
- URL validation at entry (`orchestrator.ValidateURL()`)
- Strategy type validation via `IsValidStrategy()`
- Config validation in `config.Load()`
- Manifest schema validation in `manifest.Loader`

**Authentication:**
- HTTP: User-Agent rotation, TLS fingerprinting (stealth mode)
- Git: SSH keys via `go-git/v5`
- LLM: API key from config (env var or file)
- No hardcoded credentials

**Rate Limiting:**
- HTTP: No built-in limit (relies on target server)
- LLM: Token bucket with configurable RPM, burst, backoff
- Circuit breaker with configurable failure threshold
- Location: `internal/llm/circuit_breaker.go`, `internal/llm/ratelimit.go`

**Concurrency Control:**
- Worker pool pattern (configurable count)
- Context-based cancellation
- Thread-safe state via `sync.Map`, `sync.Mutex`
- Progress bar updates with mutex protection

---

*Architecture analysis: 2026-04-07*
