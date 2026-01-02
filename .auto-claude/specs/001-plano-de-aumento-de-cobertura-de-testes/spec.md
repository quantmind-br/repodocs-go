# Specification: Increase Test Coverage from 20.2% to 80%+

## Overview

This task aims to comprehensively increase test coverage across the repodocs-go codebase from the current 20.2% to a minimum of 80% per package. The project is a CLI tool that extracts documentation from various sources (websites, Git repos, sitemaps, pkg.go.dev, llms.txt) and converts to Markdown. Given the complexity of certain packages (particularly those with external dependencies like Chrome/Chromium), adjusted thresholds will be applied (40-85% range). The implementation follows a phased approach prioritized by business criticality, using a hybrid testing strategy of unit tests with mocks, selected integration tests, and contract tests.

## Workflow Type

**Type**: feature

**Rationale**: This is a new testing implementation feature that adds comprehensive test coverage across multiple packages without modifying existing business logic. It involves creating new test files, generating mocks, and adding fixtures following established testing patterns.

## Task Scope

### Services Involved
- **main** (primary) - Go CLI application for documentation extraction and conversion

### This Task Will:
- [ ] Create comprehensive unit tests for 12 packages with 0% or low coverage
- [ ] Generate mocks using go.uber.org/mock for external dependencies
- [ ] Create HTML, Git, sitemap, and llms.txt fixtures for test scenarios
- [ ] Add integration tests for critical paths (converter, fetcher, cache, LLM, renderer)
- [ ] Expand existing tests for strategies (31% → 85%)
- [ ] Achieve 80%+ coverage for business-critical packages (strategies, converter, app, llm, config)
- [ ] Achieve 75-80% for supporting packages (output, cache, fetcher, git, cmd, domain)
- [ ] Achieve 40-50% for high-complexity renderer package (Chrome dependency)
- [ ] Update CI to report coverage per package

### Out of Scope:
- Modifying existing business logic or production code
- Performance benchmarking tests (unless critical for correctness)
- End-to-end documentation generation tests (covered by existing e2e tests)
- Refactoring code to make it testable (assumed to be already testable)

## Service Context

### main (Go CLI Application)

**Tech Stack:**
- Language: Go
- Framework: Standard library + custom architecture
- Key dependencies: Rod (headless browser), BadgerDB (cache), tls-client (HTTP), go-git (Git operations), Viper (config)
- Key directories: `internal/` (business logic), `cmd/repodocs/` (CLI entry point), `tests/` (test suites)

**Entry Point:** `cmd/repodocs/main.go`

**Architecture:**
```
URL → Detector → Strategy → Fetcher/Renderer → Converter Pipeline → Writer
```

**Core Packages:**
- `internal/app`: Orchestrator coordinates extraction; Detector routes URLs to strategies
- `internal/strategies`: Strategy implementations (crawler, git, sitemap, llms, pkggo)
- `internal/converter`: Pipeline (sanitize → readability → markdown)
- `internal/fetcher`: Stealth HTTP client with retry and caching
- `internal/renderer`: Headless browser (Rod/Chromium) for JS rendering
- `internal/llm`: Provider factory, circuit breaker, retry, rate limiting
- `internal/cache`: BadgerDB persistent cache
- `internal/output`: Writer for Markdown files with frontmatter
- `internal/domain`: Interfaces and models
- `internal/config`: Configuration loading and validation
- `internal/git`: Git client wrapper
- `internal/utils`: Utilities (94.3% coverage - complete)

**How to Run:**
```bash
# Run unit tests (fast)
make test

# Run integration tests
make test-integration

# Run e2e tests
make test-e2e

# Run all tests
make test-all

# Build
make build

# Run specific test
go test -v -run TestName ./internal/converter/...
```

**Port:** CLI tool (no HTTP server)

**CI:** GitHub Actions (ci.yml, release.yml)

## Files to Modify

| File | Service | What to Change |
|------|---------|---------------|
| `tests/unit/converter/pipeline_test.go` | main | Create new test file for pipeline orchestration |
| `tests/unit/converter/sanitizer_test.go` | main | Create new test file for HTML sanitization |
| `tests/unit/converter/readability_test.go` | main | Create new test file for content extraction |
| `tests/unit/converter/markdown_test.go` | main | Create new test file for Markdown conversion |
| `tests/unit/converter/encoding_test.go` | main | Create new test file for encoding normalization |
| `tests/unit/strategies/git_strategy_test.go` | main | Create new test file for GitStrategy |
| `tests/unit/strategies/llms_strategy_test.go` | main | Create new test file for LLMS strategy |
| `tests/unit/strategies/strategy_base_test.go` | main | Create new test file for base strategy methods |
| `tests/unit/app/detector_test.go` | main | Expand existing test coverage |
| `tests/unit/app/orchestrator_test.go` | main | Expand existing test coverage |
| `tests/unit/llm/provider_test.go` | main | Create new test file for LLM providers |
| `tests/unit/llm/circuit_breaker_test.go` | main | Create new test file for circuit breaker |
| `tests/unit/llm/retry_test.go` | main | Create new test file for retry logic |
| `tests/unit/llm/ratelimit_test.go` | main | Create new test file for rate limiting |
| `tests/unit/config/config_test.go` | main | Create new test file for config validation |
| `tests/unit/output/writer_test.go` | main | Create new test file for writer operations |
| `tests/unit/output/collector_test.go` | main | Create new test file for metadata collector |
| `tests/unit/cache/badger_test.go` | main | Create new test file for BadgerCache |
| `tests/unit/cache/keys_test.go` | main | Create new test file for key generation |
| `tests/unit/fetcher/client_test.go` | main | Create new test file for HTTP client |
| `tests/unit/fetcher/retry_test.go` | main | Create new test file for fetcher retry |
| `tests/unit/fetcher/stealth_test.go` | main | Create new test file for stealth headers |
| `tests/unit/git/client_test.go` | main | Create new test file for Git client wrapper |
| `tests/unit/domain/models_test.go` | main | Create new test file for model methods |
| `tests/unit/domain/errors_test.go` | main | Create new test file for error types |
| `tests/unit/renderer/detector_test.go` | main | Create new test file for framework detection |
| `tests/unit/renderer/pool_test.go` | main | Create new test file for tab pool |
| `cmd/repodocs/main_test.go` | main | Create new test file for CLI operations |
| `tests/mocks/git.go` | main | Generate mocks for GitClient interface |
| `tests/mocks/domain.go` | main | Generate mocks for domain interfaces |
| `Makefile` | main | Update to include new test targets |
| `.github/workflows/ci.yml` | main | Add coverage reporting per package |

## Files to Reference

These files show patterns to follow:

| File | Pattern to Copy |
|------|----------------|
| `tests/unit/app/orchestrator_test.go` | Existing test structure for app package, mock injection pattern |
| `tests/unit/strategies/crawler_strategy_test.go` | Existing strategy test pattern, table-driven tests |
| `tests/testutil/` | Test utilities for temp dirs, HTTP servers, cache, assertions |
| `tests/fixtures/` | Existing fixture file organization |
| `internal/strategies/base.go` | Strategy base implementation with metadata methods |
| `internal/converter/pipeline.go` | Converter pipeline with sequential transformations |
| `internal/llm/circuit_breaker.go` | State machine pattern for circuit breaker |
| `internal/fetcher/retry.go` | Retry logic with exponential backoff |
| `internal/domain/interfaces.go` | Interface definitions for mocking |

## Patterns to Follow

### Table-Driven Tests for Strategy Execution

From `tests/unit/strategies/crawler_strategy_test.go`:

```go
func TestCrawlerStrategy_Execute(t *testing.T) {
	tests := []struct {
		name    string
		url     string
		opts    strategies.Options
		mock    func(*MockFetcher)
		want    string
		wantErr bool
	}{
		{
			name: "successful crawl",
			url:  "https://example.com/docs",
			// ... test cases
		},
	}
	for _, tt := range tests {
		t.Run(tt.name, func(t *testing.T) {
			// Setup mocks
			// Execute
			// Assert
		})
	}
}
```

**Key Points:**
- Use table-driven tests for multiple scenarios
- Separate setup, execution, and assertion phases
- Run subtests with `t.Run()` for better test output
- Use descriptive test names

### Mock Injection via Dependencies Struct

From `internal/app/detector.go` and `tests/unit/app/orchestrator_test.go`:

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

**Key Points:**
- Use dependency injection for all external dependencies
- Create `Dependencies` struct as composition root
- Inject mocks via constructor or field setters
- Test each package in isolation with mocked dependencies

### Test Utilities for Setup/Teardown

From `tests/testutil/`:

```go
// Use testutil.TempDir() for temporary directories
tempDir := testutil.TempDir(t)
defer testutil CleanupTempDir(t, tempDir)

// Use testutil.SetupTestCache() for BadgerDB cache
cache := testutil.SetupTestCache(t)
defer cache.Close()

// Use testutil.NewHTTPTestServer() for HTTP mocking
server := testutil.NewHTTPTestServer(t, handler)
defer server.Close()
```

**Key Points:**
- Reuse existing test utilities for common operations
- Always cleanup resources in defer statements
- Use `t *testing.T` parameter for automatic cleanup on test failure
- Test utilities handle setup and teardown automatically

### HTML Fixtures for Converter Tests

From `tests/fixtures/`:

```go
// Load fixture from tests/fixtures/html/sample.html
htmlContent := testutil.LoadFixture(t, "tests/fixtures/html/sample.html")

// Test converter pipeline with real HTML
result, err := pipeline.Convert(htmlContent, url)
assert.NoError(t, err)
assert.Contains(t, result, "# Expected Heading")
```

**Key Points:**
- Create HTML fixtures for different scenarios (SPA, tables, code blocks)
- Store fixtures in `tests/testdata/fixtures/html/`
- Load fixtures with utility function
- Test edge cases (malformed HTML, empty content, encoding issues)

### State Machine Testing for Circuit Breaker

From `internal/llm/circuit_breaker.go`:

```go
func TestCircuitBreaker_StateTransitions(t *testing.T) {
	cb := llm.NewCircuitBreaker(llm.CircuitBreakerConfig{
		Threshold: 3,
		Timeout:   time.Minute,
	})

	// Test: Closed → Open on threshold failures
	for i := 0; i < 3; i++ {
		cb.RecordFailure()
	}
	assert.Equal(t, llm.StateOpen, cb.State())

	// Test: Open → HalfOpen after timeout
	time.Sleep(time.Minute + 100*time.Millisecond)
	assert.True(t, cb.Allow()) // Transition to HalfOpen

	// Test: HalfOpen → Closed on success
	cb.RecordSuccess()
	assert.Equal(t, llm.StateClosed, cb.State())
}
```

**Key Points:**
- Test all state transitions explicitly
- Use timing control for timeout-based transitions
- Verify threshold counting logic
- Test both success and failure paths

### Integration Tests with Real Dependencies

From existing integration tests pattern:

```go
func TestConverter_Integration(t *testing.T) {
	if testing.Short() {
		t.Skip("Skipping integration test in short mode")
	}

	// Use real BadgerDB cache
	cache := testutil.SetupRealCache(t)
	defer cache.Close()

	// Test pipeline with real cache
	pipeline := converter.NewPipeline(converter.Config{
		Cache: cache,
	})

	result, err := pipeline.Convert(html, url)
	assert.NoError(t, err)
	assert.NotNil(t, result)
}
```

**Key Points:**
- Skip integration tests with `-short` flag
- Use real dependencies (BadgerDB, HTTP server)
- Cleanup resources properly
- Focus on integration points between components

## Requirements

### Functional Requirements

1. **Phase 1: Business-Critical Components (3-4 weeks)**
   - Description: Achieve 80-85% test coverage for strategies, converter, and app packages
   - Acceptance: Unit tests created for all public functions, integration tests for critical paths, coverage report shows ≥80% per package

2. **Phase 2: LLM and Configuration (2-3 weeks)**
   - Description: Achieve 80% test coverage for LLM provider factory, circuit breaker, retry, rate limiting, and config packages
   - Acceptance: Unit tests with HTTP mocking, state machine tests for circuit breaker, timing-controlled tests for rate limiter, coverage ≥80%

3. **Phase 3: Output and Cache (2 weeks)**
   - Description: Achieve 75-80% test coverage for output writer and cache packages
   - Acceptance: Unit tests with in-memory mocks, integration tests with real BadgerDB, coverage ≥75%

4. **Phase 4: Fetcher and Git (2 weeks)**
   - Description: Achieve 70-80% test coverage for HTTP client with retry/stealth and Git client wrapper
   - Acceptance: Unit tests with HTTP mocking, integration tests with real HTTP server, go-git mocks, coverage ≥70%

5. **Phase 5: CLI and Domain (1-2 weeks)**
   - Description: Achieve 80-85% test coverage for CLI operations and domain models/errors
   - Acceptance: CLI tests with mocked dependencies, model method tests, error constructor tests, coverage ≥80%

6. **Phase 6: Renderer (2 weeks)**
   - Description: Achieve 40-50% test coverage for browser rendering package
   - Acceptance: Unit tests for framework detection and pool management, limited integration tests with real browser, coverage ≥40%

7. **Mock Generation**
   - Description: Generate mocks for all external dependencies using go.uber.org/mock
   - Acceptance: `mockgen` commands executed, mocks generated in `tests/mocks/`, all test files compile

8. **Fixture Creation**
   - Description: Create HTML, Git, sitemap, and llms.txt fixtures for test scenarios
   - Acceptance: Fixtures created in `tests/testdata/fixtures/`, used in unit and integration tests

9. **CI Integration**
   - Description: Update CI workflow to report coverage per package
   - Acceptance: Coverage report visible in GitHub Actions, fails if below threshold

### Edge Cases

1. **Malformed HTML** - Test with incomplete tags, invalid encoding, charset mismatches
2. **Network Failures** - Test retry logic with connection timeouts, DNS failures, HTTP errors
3. **Empty Responses** - Test with zero-length content, missing fields, null values
4. **Concurrent Access** - Test cache and pool operations with multiple goroutines
5. **Race Conditions** - Test circuit breaker state transitions under concurrent load
6. **Large Files** - Test with 10MB+ HTML, large repositories, massive sitemaps
7. **Special Characters** - Test Unicode, emojis, RTL languages, escape sequences
8. **Browser Crashes** - Test renderer recovery from Chrome/Chromium crashes
9. **Git Clone Failures** - Test with invalid URLs, authentication failures, network errors
10. **Rate Limiting** - Test with burst traffic, token bucket exhaustion

## Implementation Notes

### DO
- Follow existing test patterns in `tests/unit/` and `tests/integration/`
- Reuse test utilities from `tests/testutil/` for setup/teardown
- Use table-driven tests for multiple scenarios
- Create descriptive test names that explain what is being tested
- Use `t.Run()` for subtests to get better failure output
- Mock all external dependencies (HTTP, Git, Cache, Renderer)
- Test both success and error paths
- Test edge cases (nil inputs, empty strings, invalid types)
- Use `assert` package from `github.com/stretchr/testify` for assertions
- Run tests with `-race` flag to detect race conditions
- Run tests with `-short` flag to skip integration/slow tests in unit mode
- Cleanup resources in defer statements (temp dirs, cache, HTTP servers)
- Use fixtures for complex HTML/XML/Git data
- Test public API surface (don't test internal functions unless exported)
- Follow Go testing conventions (test file: `package_test.go`, test function: `TestFunction`)
- Generate mocks with `mockgen` from interface definitions
- Update Makefile to include new test targets
- Update CI to report coverage per package

### DON'T
- Don't create new test utilities when existing ones work
- Don't write tests that depend on external services (use mocks)
- Don't test unexported functions unless they contain complex logic
- Don't ignore race detector warnings
- Don't skip cleanup of resources (temp dirs, cache, servers)
- Don't hardcode file paths (use relative paths from project root)
- Don't sleep in tests (use timing control or channels)
- Don't test third-party library code (mock it)
- Don't create 100% coverage for renderer (40-50% is acceptable due to Chrome dependency)
- Don't test error messages (test error types and conditions)
- Don't use production config in tests (use test config)
- Don't write tests that are flaky (timing-dependent, race-prone)
- Don't modify production code to make it testable (it should already be testable)
- Don't skip testing of error paths (they are critical for reliability)

## Development Environment

### Start Services

This is a CLI tool (no background services required). Run tests directly:

```bash
# Run all unit tests (fast, with -short flag)
make test

# Run integration tests (network-dependent)
make test-integration

# Run e2e tests (full CLI workflows)
make test-e2e

# Run all tests
make test-all

# Run specific package tests
go test -v -race ./internal/converter/...
go test -v ./internal/strategies/...
```

### Test Coverage Reports

```bash
# Generate coverage report
go test -coverprofile=coverage.out ./...
go tool cover -html=coverage.out -o coverage.html

# Check coverage per package
go test -coverprofile=coverage.out ./internal/converter/...
go tool cover -func=coverage.out | grep internal/converter
```

### Required Environment Variables

No environment variables required for running tests. Tests use:
- Temporary directories for cache and output
- In-memory or test BadgerDB for cache
- Mock HTTP servers for fetcher tests
- Mock Git operations for Git tests
- Test fixtures for HTML, sitemap, llms.txt files

### Project Structure

```
repodocs-go/
├── cmd/repodocs/          # CLI entry point
├── internal/
│   ├── app/              # Orchestrator, detector (0% → 85%)
│   ├── cache/            # BadgerDB cache (0% → 75%)
│   ├── config/           # Configuration (0% → 85%)
│   ├── converter/        # Pipeline (0% → 85%)
│   ├── domain/           # Interfaces, models (0% → 85%)
│   ├── fetcher/          # HTTP client (0% → 70%)
│   ├── git/              # Git wrapper (0% → 80%)
│   ├── llm/              # LLM providers (6% → 80%)
│   ├── output/           # Writer (0% → 80%)
│   ├── renderer/         # Browser (0% → 40-50%)
│   ├── strategies/       # Strategies (31% → 85%)
│   └── utils/            # Utilities (94.3% ✅)
├── tests/
│   ├── unit/             # Unit tests (fast, -short)
│   ├── integration/      # Integration tests (real deps)
│   ├── e2e/              # E2E tests (full CLI)
│   ├── mocks/            # Generated mocks
│   ├── testutil/         # Test utilities
│   └── fixtures/         # Test fixtures
└── Makefile
```

## Success Criteria

The task is complete when:

1. [ ] Phase 1 complete: strategies (85%), converter (85%), app (85%)
2. [ ] Phase 2 complete: llm (80%), config (85%)
3. [ ] Phase 3 complete: output (80%), cache (75%)
4. [ ] Phase 4 complete: fetcher (70%), git (80%)
5. [ ] Phase 5 complete: cmd (80%), domain (85%)
6. [ ] Phase 6 complete: renderer (40-50%)
7. [ ] All tests pass with `make test-all`
8. [ ] No race detector warnings (`-race` flag)
9. [ ] Coverage report generated per package
10. [ ] CI updated with coverage reporting
11. [ ] All mocks generated and tests compile
12. [ ] Fixtures created and used in tests
13. [ ] Existing tests still pass (no regressions)
14. [ ] New functionality verified via test execution

## QA Acceptance Criteria

**CRITICAL**: These criteria must be verified by the QA Agent before sign-off.

### Unit Tests

| Test | File | What to Verify |
|------|------|----------------|
| Converter Pipeline | `tests/unit/converter/pipeline_test.go` | Convert, ConvertHTML, ConvertHTMLWithSelector, removeExcluded |
| Converter Sanitizer | `tests/unit/converter/sanitizer_test.go` | Sanitize, normalizeURLs, resolveURL, normalizeSrcset, removeEmptyElements |
| Converter Readability | `tests/unit/converter/readability_test.go` | Extract, extractWithSelector, extractWithReadability, ExtractDescription, ExtractHeaders, ExtractLinks |
| Converter Markdown | `tests/unit/converter/markdown_test.go` | Convert, cleanMarkdown, GenerateFrontmatter, AddFrontmatter, StripMarkdown, CountWords, CountChars |
| Converter Encoding | `tests/unit/converter/encoding_test.go` | DetectEncoding, ConvertToUTF8, IsUTF8, GetEncoder |
| Git Strategy | `tests/unit/strategies/git_strategy_test.go` | parseGitURL, tryArchiveDownload, downloadAndExtract, extractTarGz, findDocumentationFiles, processFiles, processFile, detectDefaultBranch |
| Crawler Strategy | `tests/unit/strategies/crawler_strategy_test.go` | Execute, isHTMLContentType, crawling logic |
| LLMS Strategy | `tests/unit/strategies/llms_strategy_test.go` | Execute, parseLLMSLinks, filterLLMSLinks |
| PkgGo Strategy | `tests/unit/strategies/pkggo_strategy_test.go` | Execute, extractSections |
| Strategy Base | `tests/unit/strategies/strategy_base_test.go` | DefaultOptions, FlushMetadata, SetStrategy, SetSourceURL, WriteDocument |
| App Detector | `tests/unit/app/detector_test.go` | DetectStrategy, CreateStrategy, GetAllStrategies, FindMatchingStrategy |
| App Orchestrator | `tests/unit/app/orchestrator_test.go` | NewOrchestrator, Run, Close, GetStrategyName, ValidateURL |
| LLM Provider | `tests/unit/llm/provider_test.go` | NewAnthropicProvider, NewGoogleProvider, NewOpenAIProvider, Complete, Close, handleHTTPError |
| LLM Circuit Breaker | `tests/unit/llm/circuit_breaker_test.go` | NewCircuitBreaker, Allow, RecordSuccess, RecordFailure, State, transitionTo |
| LLM Retry | `tests/unit/llm/retry_test.go` | NewRetrier, Execute, calculateBackoff, IsRetryableError, ShouldRetryStatusCode |
| LLM Rate Limiter | `tests/unit/llm/ratelimit_test.go` | NewTokenBucket, Wait, TryAcquire, Available, refill |
| LLM Metadata | `tests/unit/llm/metadata_test.go` | Enhance, EnhanceAll, applyMetadata |
| Config Validation | `tests/unit/config/config_test.go` | Validate, all validation methods |
| Config Loader | `tests/unit/config/loader_test.go` | Load, LoadWithViper, setDefaults, EnsureConfigDir, EnsureCacheDir |
| Output Writer | `tests/unit/output/writer_test.go` | Write, WriteMultiple, FlushMetadata, Exists, EnsureBaseDir, Clean, Stats |
| Output Collector | `tests/unit/output/collector_test.go` | Add, Flush, buildIndex, GetIndex, configuration methods |
| Cache Badger | `tests/unit/cache/badger_test.go` | NewBadgerCache, Get, Set, Has, Delete, Close, Clear, Size, Stats |
| Cache Keys | `tests/unit/cache/keys_test.go` | GenerateKey, GenerateKeyWithPrefix, normalizeForKey, PageKey, SitemapKey, MetadataKey |
| Fetcher Client | `tests/unit/fetcher/client_test.go` | NewClient, Get, GetWithHeaders, doRequest, GetCookies, Close, cache operations |
| Fetcher Retry | `tests/unit/fetcher/retry_test.go` | NewRetrier, Retry, RetryWithValue, ShouldRetryStatus, ParseRetryAfter |
| Fetcher Stealth | `tests/unit/fetcher/stealth_test.go` | RandomUserAgent, RandomAcceptLanguage, StealthHeaders, RandomDelay |
| Git Client | `tests/unit/git/client_test.go` | NewClient, PlainCloneContext |
| Domain Models | `tests/unit/domain/models_test.go` | ToMetadata, ToFrontmatter, ToDocumentMetadata, ToSimpleMetadata, ToSimpleDocumentMetadata |
| Domain Errors | `tests/unit/domain/errors_test.go` | All error constructors and methods |
| Renderer Detector | `tests/unit/renderer/detector_test.go` | NeedsJSRendering, DetectFramework, HasDynamicContent, hasSPAPattern |
| Renderer Pool | `tests/unit/renderer/pool_test.go` | NewTabPool, Acquire, Release, Close, Size, MaxSize |
| CLI Main | `cmd/repodocs/main_test.go` | run, initConfig, checkInternet, checkChrome, checkWritePermissions, checkCacheDir |

### Integration Tests

| Test | Services | What to Verify |
|------|----------|----------------|
| Converter Integration | converter + cache | Pipeline with real BadgerDB cache, HTML → Markdown conversion |
| Fetcher Integration | fetcher + HTTP | Real HTTP requests with retry, stealth headers, response handling |
| Cache Integration | cache + BadgerDB | Real BadgerDB persistence, Get/Set/Has/Delete/Clear operations |
| LLM Integration | llm + HTTP | LLM provider HTTP mocking, error handling, rate limiting |
| Renderer Integration | renderer + Chrome | Real browser rendering (limited scope), framework detection |
| Strategies Integration | strategies + all deps | End-to-end strategy execution with real dependencies |

### Browser Verification (if frontend)

N/A - This is a CLI tool, no browser verification required.

### Database Verification (if applicable)

| Check | Query/Command | Expected |
|-------|---------------|----------|
| Cache persistence | BadgerDB operations after restart | Data survives process restart |
| Concurrent access | Multiple goroutines reading/writing | No race conditions, data consistency |
| Cache clearing | cache.Clear() | All entries removed, size = 0 |

### Coverage Threshold Verification

| Package | Target | Command | Expected |
|---------|--------|---------|----------|
| internal/strategies | 85% | `go test -coverprofile=coverage.out ./internal/strategies/... && go tool cover -func=coverage.out | grep total` | ≥85% |
| internal/converter | 85% | `go test -coverprofile=coverage.out ./internal/converter/... && go tool cover -func=coverage.out | grep total` | ≥85% |
| internal/app | 85% | `go test -coverprofile=coverage.out ./internal/app/... && go tool cover -func=coverage.out | grep total` | ≥85% |
| internal/llm | 80% | `go test -coverprofile=coverage.out ./internal/llm/... && go tool cover -func=coverage.out | grep total` | ≥80% |
| internal/config | 85% | `go test -coverprofile=coverage.out ./internal/config/... && go tool cover -func=coverage.out | grep total` | ≥85% |
| internal/output | 80% | `go test -coverprofile=coverage.out ./internal/output/... && go tool cover -func=coverage.out | grep total` | ≥80% |
| internal/cache | 75% | `go test -coverprofile=coverage.out ./internal/cache/... && go tool cover -func=coverage.out | grep total` | ≥75% |
| internal/fetcher | 70% | `go test -coverprofile=coverage.out ./internal/fetcher/... && go tool cover -func=coverage.out | grep total` | ≥70% |
| internal/git | 80% | `go test -coverprofile=coverage.out ./internal/git/... && go tool cover -func=coverage.out | grep total` | ≥80% |
| cmd/repodocs | 80% | `go test -coverprofile=coverage.out ./cmd/repodocs/... && go tool cover -func=coverage.out | grep total` | ≥80% |
| internal/domain | 85% | `go test -coverprofile=coverage.out ./internal/domain/... && go tool cover -func=coverage.out | grep total` | ≥85% |
| internal/renderer | 40% | `go test -coverprofile=coverage.out ./internal/renderer/... && go tool cover -func=coverage.out | grep total` | ≥40% |

### QA Sign-off Requirements

- [ ] All unit tests pass with `make test`
- [ ] All integration tests pass with `make test-integration`
- [ ] All E2E tests pass with `make test-e2e`
- [ ] No race detector warnings (`go test -race ./...`)
- [ ] Coverage meets or exceeds thresholds for all 12 packages
- [ ] No regressions in existing functionality (existing tests pass)
- [ ] Code follows established testing patterns (table-driven, mock injection, cleanup)
- [ ] All fixtures are valid and used in tests
- [ ] All mocks compile and implement correct interfaces
- [ ] CI workflow includes coverage reporting
- [ ] No security vulnerabilities introduced (no hardcoded credentials, API keys in tests)
- [ ] Test execution time is reasonable (unit tests < 30s total)
- [ ] Tests are deterministic (no flaky tests, no timing-dependent failures)
- [ ] Documentation updated (if applicable)
