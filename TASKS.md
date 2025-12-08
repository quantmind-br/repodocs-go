# RepoDocs-Go - Development Tasks

## Status Legend
- [ ] Not started
- [x] Completed
- [🔄] In progress

---

## Phase 1: Foundation & Project Setup

### 1.1 Module Initialization
- [x] Initialize Go module (`go mod init github.com/quantmind-br/repodocs-go`)
- [x] Create directory structure as specified in PLAN.md

### 1.2 Build System
- [x] Create Makefile with build, test, lint, coverage targets
- [x] Create `.golangci.yml` linter configuration
- [x] Create `.github/workflows/ci.yml` for GitHub Actions CI/CD

### 1.3 Domain Layer (`internal/domain`)
- [x] Define `interfaces.go` (Strategy, Fetcher, Cache interfaces)
- [x] Define `models.go` (Page, Document, Metadata structs)
- [x] Define `errors.go` (custom typed errors)

### 1.4 Configuration (`internal/config`)
- [x] Create `config.go` (configuration struct)
- [x] Create `defaults.go` (default values and constants)
- [x] Create `loader.go` (Viper-based config loading from file/env/flags)

### 1.5 Utilities (`internal/utils`)
- [x] Create `logger.go` (Zerolog wrapper)
- [x] Create `fs.go` (filename sanitization)
- [x] Create `url.go` (URL normalization)
- [x] Create `workerpool.go` (worker pool with context)

### 1.6 Version (`pkg/version`)
- [x] Create `version.go` (version info for CLI)

---

## Phase 2: HTTP Stealth Module (`internal/fetcher`)

### 2.1 Client Implementation
- [x] Create `client.go` (tls-client wrapper with stealth features)
- [x] Implement `NewClient()` with configurable options
- [x] Implement `Get()` and `GetWithHeaders()` methods
- [x] Implement `GetCookies()` for session sharing with Rod

### 2.2 Stealth Features
- [x] Create `stealth.go` (User-Agent rotation, header randomization)
- [x] Configure TLS fingerprinting (Chrome_131 profile)
- [x] Implement connection pooling

### 2.3 Retry Logic
- [x] Create `retry.go` (exponential backoff)
- [x] Handle 429, 503, 520-530 status codes
- [x] Handle timeout and connection reset errors

### 2.4 Transport
- [x] Create `transport.go` (http.RoundTripper for Colly integration)

---

## Phase 3: Renderer Module (`internal/renderer`)

### 3.1 Rod Integration
- [x] Create `rod.go` (headless browser renderer)
- [x] Implement `NewRenderer()` with configurable options
- [x] Implement `Render()` with timeout, wait conditions, scroll support
- [x] Implement `Close()` for cleanup

### 3.2 Stealth Mode
- [x] Create `stealth.go` (go-rod/stealth plugin integration)
- [x] Remove webdriver flag detection
- [x] Emulate real browser plugins and WebGL

### 3.3 Tab Pool
- [x] Create `pool.go` (tab pool for concurrency)
- [x] Implement `NewTabPool()` with configurable max tabs
- [x] Implement `Acquire()` and `Release()` methods
- [x] Implement tab cleanup and reuse

### 3.4 SPA Detection
- [x] Create `detector.go` (detect if page needs JS rendering)
- [x] Detect React, Vue, Next.js, Nuxt patterns
- [x] Analyze content vs script ratio

---

## Phase 4: Cache Module (`internal/cache`)

### 4.1 Interface
- [x] Create `interface.go` (Cache interface definition)
- [x] Define CacheEntry struct

### 4.2 BadgerDB Implementation
- [x] Create `badger.go` (BadgerDB cache implementation)
- [x] Implement Get, Set, Has, Delete, Close methods
- [x] Implement TTL-based expiration

### 4.3 Key Generation
- [x] Create `keys.go` (SHA256-based key generation from URLs)

---

## Phase 5: Converter Pipeline (`internal/converter`)

### 5.1 Pipeline Orchestrator
- [x] Create `pipeline.go` (main conversion pipeline)
- [x] Implement stage-by-stage processing

### 5.2 Encoding Detection
- [x] Create `encoding.go` (charset detection and UTF-8 conversion)

### 5.3 Content Extraction
- [x] Create `readability.go` (go-readability integration)
- [x] Support CSS selector-based extraction
- [x] Implement heuristic-based extraction

### 5.4 Sanitization
- [x] Create `sanitizer.go` (remove script, style, iframe, noscript)
- [x] Normalize relative URLs to absolute

### 5.5 Markdown Conversion
- [x] Create `markdown.go` (html-to-markdown integration)
- [x] Configure fenced code blocks, ATX headings
- [x] Preserve syntax highlighting
- [x] Handle tables and details elements
- [x] Add YAML frontmatter generation

---

## Phase 6: Strategies (`internal/strategies`)

### 6.1 Base Interface
- [x] Create `strategy.go` (Strategy interface and Options struct)

### 6.2 LLMS Strategy
- [x] Create `llms.go` (llms.txt parser)
- [x] Parse markdown links
- [x] Download and convert referenced pages

### 6.3 Pkg.go.dev Strategy
- [x] Create `pkggo.go` (pkg.go.dev extractor)
- [x] Extract documentation content with goquery
- [x] Implement `--split` option for section-based output

### 6.4 Sitemap Strategy
- [x] Create `sitemap.go` (XML sitemap parser)
- [x] Support sitemap index files
- [x] Support gzipped sitemaps
- [x] Order by lastmod date

### 6.5 Git Strategy
- [x] Create `git.go` (git repository cloner)
- [x] Implement shallow clone (depth=1)
- [x] Walk and filter documentation files
- [x] Support `--include-assets` flag

### 6.6 Crawler Strategy
- [x] Create `crawler.go` (Colly-based web crawler)
- [x] Integrate stealth transport
- [x] Implement URL deduplication
- [x] Respect max-depth and limit flags
- [x] Auto-detect SPA and fallback to renderer

---

## Phase 7: Output Module (`internal/output`)

### 7.1 Writer
- [x] Create `writer.go` (output orchestrator)
- [x] Support flat and nested directory structures

### 7.2 Markdown Output
- [x] Markdown output integrated in `writer.go`

### 7.3 JSON Metadata
- [x] JSON metadata generation in `writer.go`
- [x] Include URL, title, word count, headers, links

### 7.4 Filesystem Operations
- [x] Filesystem operations in `internal/utils/fs.go`

---

## Phase 8: Application Layer (`internal/app`)

### 8.1 URL Detector
- [x] Create `detector.go` (auto-detect strategy from URL)
- [x] Detect llms.txt, sitemap, git, pkg.go.dev, crawler patterns

### 8.2 Orchestrator
- [x] Create `orchestrator.go` (main application orchestrator)
- [x] Coordinate fetcher, renderer, cache, converter, output
- [x] Implement graceful shutdown

---

## Phase 9: CLI (`cmd/repodocs`)

### 9.1 Main Command
- [x] Create `main.go` (entrypoint with Cobra)
- [x] Define all global flags (output, concurrency, limit, max-depth, etc.)
- [x] Define cache flags (no-cache, cache-ttl, refresh-cache)
- [x] Define rendering flags (render-js, timeout)
- [x] Define output flags (json-meta, dry-run)
- [x] Define specific flags (split, include-assets, user-agent, content-selector)

### 9.2 Doctor Command
- [x] Create `doctor.go` (system dependency checker)
- [x] Check internet connection
- [x] Check Chrome/Chromium availability
- [x] Validate config file

### 9.3 Graceful Shutdown
- [x] Implement SIGINT/SIGTERM handling
- [x] Propagate cancellation via context

---

## Phase 10: Testing

### 10.1 Test Data
- [x] Create `tests/testdata/fixtures/` with HTML/XML fixtures
- [x] Create `tests/testdata/golden/` with expected outputs

### 10.2 Unit Tests
- [x] Create `tests/unit/converter_test.go`
- [x] Create `tests/unit/sanitizer_test.go`
- [x] Create `tests/unit/cache_test.go`
- [x] Create `tests/unit/retry_test.go`
- [x] Create `tests/unit/config_test.go`

### 10.3 Integration Tests
- [x] Create `tests/integration/fetcher_test.go`
- [x] Create `tests/integration/renderer_test.go`
- [x] Create `tests/integration/strategies_test.go`

### 10.4 E2E Tests
- [x] Create `tests/e2e/crawl_test.go`
- [x] Create `tests/e2e/sitemap_test.go`

---

## Phase 11: Documentation

### 11.1 README
- [x] Create comprehensive README.md
- [x] Include installation instructions
- [x] Include usage examples
- [x] Include configuration reference

---

## Validation Checklist

- [ ] TLS fingerprint matches real Chrome browser
- [ ] Cloudflare managed challenge bypass works
- [ ] Basic rate limiting bypass works
- [ ] SPA rendering works (React/Vue/Angular)
- [ ] Cache resume functionality works
- [ ] Test coverage > 80%

---

## Development Progress Summary

### Completed (Phase 1-9)
- Project setup and build system
- Domain layer with interfaces and models
- Configuration management with Viper
- Utilities (logger, filesystem, URL, worker pool)
- HTTP stealth fetcher with TLS fingerprinting
- Cache system with BadgerDB
- Headless renderer with Rod and stealth mode
- HTML to Markdown converter pipeline
- All strategies (LLMS, Sitemap, Git, PkgGoDev, Crawler)
- Output writer with frontmatter and JSON metadata
- CLI with all flags and doctor command
- Application orchestrator and URL detector
- Doctor command with actual dependency checks

### Remaining
- All phases completed!
