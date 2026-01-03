# PLAN: Increase Test Coverage to 80%

**Status**: Draft
**Created**: 2025-01-03
**Current Coverage**: 56.6% (overall), 84.3% (targeted packages)
**Target Coverage**: 80% overall

## Summary

This plan outlines the implementation of tests to increase overall code coverage from 56.6% to 80%. The focus is on three packages with the lowest coverage: `internal/strategies` (46.4%), `internal/renderer` (48.2%), and `internal/strategies` sub-packages.

## Current Coverage by Package

| Package | Coverage | Gap to Target |
|---------|----------|---------------|
| `internal/strategies` | 46.4% | +33.6% |
| `internal/renderer` | 48.2% | +31.8% |
| `cmd/repodocs` | 72.7% | +7.3% |
| `internal/llm` | 83.3% | ✅ OK |
| `internal/converter` | 83.0% | ✅ OK |
| `internal/fetcher` | 89.0% | ✅ OK |
| `internal/utils` | 94.3% | ✅ OK |
| `internal/config` | 94.2% | ✅ OK |
| `internal/app` | 95.5% | ✅ OK |
| `internal/output` | 97.6% | ✅ OK |
| `internal/cache` | 97.4% | ✅ OK |
| `internal/git` | 100.0% | ✅ OK |
| `internal/domain` | 100.0% | ✅ OK |

## Phase 1: DocsRS Strategy Tests (Priority: HIGH)

**Impact**: +5-8% overall coverage
**Files**: `internal/strategies/docsrs*.go`

### 1.1 Create `tests/unit/strategies/docsrs_execute_test.go`

**Test Cases**:
- `TestDocsRSStrategy_Execute_ValidCrate` - Happy path with mock JSON API
- `TestDocsRSStrategy_Execute_FetcherError` - Handle fetcher failures
- `TestDocsRSStrategy_Execute_JSONParseError` - Handle malformed JSON
- `TestDocsRSStrategy_Execute_Limit` - Test limit option
- `TestDocsRSStrategy_Execute_DryRun` - Test dry-run mode
- `TestDocsRSStrategy_Execute_ContextCancellation` - Context handling

**Dependencies**:
- Mock `domain.Fetcher` responses with docs.rs JSON fixtures
- Mock `output.Writer` for verification
- Use `tests/fixtures/docsrs/` for test data

### 1.2 Create `tests/unit/strategies/docsrs_json_test.go`

**Test Cases**:
- `TestDocsRSJSONEndpoint` - Endpoint URL building
- `TestParseRustdocJSON` - JSON parsing with valid/invalid data
- `TestGetItemByID` - Item lookup by ID
- `TestCollectItems` - Recursive item collection
- `TestHasDocumentableChildren` - Children detection
- `TestCheckFormatVersion` - Version validation

**Dependencies**:
- JSON fixtures in `tests/fixtures/docsrs/json/`
- Mock HTTP responses

### 1.3 Create `tests/unit/strategies/docsrs_renderer_test.go`

**Test Cases**:
- `TestRenderItem_Function` - Function rendering
- `TestRenderItem_Struct` - Struct rendering
- `TestRenderItem_Trait` - Trait rendering
- `TestRenderItem_Module` - Module rendering
- `TestRenderType` - Type rendering
- `TestRenderTypeMap` - Generic type rendering
- `TestResolveCrossRefs` - Cross-reference resolution

**Dependencies**:
- Mock JSON items
- Golden files for expected Markdown output

## Phase 2: Renderer Pool Tests (Priority: HIGH)

**Impact**: +2-3% overall coverage
**Files**: `internal/renderer/pool.go`

### 2.1 Create `tests/unit/renderer/pool_test.go`

**Test Cases**:
- `TestNewTabPool_ValidParams` - Constructor validation
- `TestNewTabPool_ZeroMaxTabs` - Edge case: zero max tabs
- `TestNewTabPool_NegativeMaxTabs` - Edge case: negative max tabs
- `TestTabPool_AcquireRelease` - Basic acquire/release cycle
- `TestTabPool_AcquireMultiple` - Multiple concurrent acquires
- `TestTabPool_ContextCancellation` - Context timeout during acquire
- `TestTabPool_Close_WhileAcquired` - Close with tabs in use
- `TestTabPool_Close_Twice` - Double close safety
- `TestTabPool_Acquire_FromClosedPool` - Operations on closed pool
- `TestTabPool_Size` - Current size reporting
- `TestTabPool_MaxSize` - Max size reporting

**Dependencies**:
- Mock `rod.Browser` interface
- Mock `rod.Page` interface
- May need simple hand-rolled mocks for rod interfaces

## Phase 3: Rod Renderer Tests (Priority: MEDIUM)

**Impact**: +3-4% overall coverage
**Files**: `internal/renderer/rod.go`

### 3.1 Create `tests/unit/renderer/rod_test.go`

**Test Cases**:
- `TestNewRenderer_DefaultOptions` - Default configuration
- `TestNewRenderer_CustomTimeout` - Custom timeout option
- `TestNewRenderer_CustomHeadless` - Headless option
- `TestNewRenderer_InvalidBrowserPath` - Invalid path handling
- `TestRenderer_Render_Timeout` - Page timeout
- `TestRenderer_Render_Cookies` - Cookie setting
- `TestRenderer_Render_ScrollToEnd` - Lazy loading scroll
- `TestRenderer_Close_Twice` - Double close safety
- `TestIsAvailable_NoBrowser` - Missing browser detection
- `TestGetBrowserPath_Default` - Default browser detection
- `TestGetTabPool` - Pool access

**Dependencies**:
- Integration tests with actual browser (if available)
- Mock `launcher` for browser detection
- Mock HTTP responses for page content

**Note**: Some tests may need to be integration tests due to tight coupling with rod library.

## Phase 4: GitHub Pages Strategy Tests (Priority: MEDIUM)

**Impact**: +3-4% overall coverage
**Files**: `internal/strategies/github_pages.go`

### 4.1 Create `tests/unit/strategies/github_pages_unit_test.go`

**Test Cases**:
- `TestGitHubPagesStrategy_Execute_HappyPath` - Full execution flow
- `TestGitHubPagesStrategy_Execute_NoURLsDiscovered` - Empty discovery
- `TestGitHubPagesStrategy_Execute_Limit` - URL limit application
- `TestGitHubPagesStrategy_NormalizeBaseURL` - URL normalization
- `TestGitHubPagesStrategy_LooksLikeSPAShell` - SPA shell detection
- `TestGitHubPagesStrategy_IsEmptyOrErrorContent` - Content validation
- `TestGitHubPagesStrategy_FilterURLs` - URL filtering logic
- `TestGitHubPagesStrategy_DiscoverViaHTTP` - HTTP discovery path
- `TestGitHubPagesStrategy_DiscoverViaBrowser` - Browser fallback
- `TestGitHubPagesStrategy_RenderPage` - Page rendering integration

**Dependencies**:
- Mock `domain.Fetcher` for HTTP responses
- Mock `domain.Renderer` for browser rendering
- Mock `converter.Pipeline` for content conversion
- HTML fixtures in `tests/fixtures/github_pages/`

## Phase 5: Main Package Tests (Priority: LOW)

**Impact**: +2-3% overall coverage
**Files**: `cmd/repodocs/main.go`

### 5.1 Extend `tests/unit/cmd/main_test.go`

**Test Cases**:
- `TestRun_InvalidURL` - URL validation errors
- `TestRun_ConfigFileError` - Config loading errors
- `TestRun_StrategyDetection` - Strategy routing
- `TestRun_OutputDirectory` - Output directory creation
- `TestRun_VerboseMode` - Verbose flag handling
- `TestRun_DryRunMode` - Dry-run execution
- `TestCheckInternet` - Internet check logic
- `TestCheckChrome` - Chrome detection

**Dependencies**:
- Mock filesystem operations
- Mock strategy execution
- Use existing test utilities

## Test Infrastructure Additions

### Mock Extensions

Extend `tests/mocks/domain.go` if needed:
- Add more specific mock methods for Renderer interface
- Add specific mock responses for docs.rs API

### Fixtures

Create new fixtures in `tests/fixtures/`:
```
tests/fixtures/
├── docsrs/
│   ├── json/
│   │   ├── std_valid.json
│   │   ├── std_malformed.json
│   │   └── crate_example.json
│   └── html/
│       └── crate_pages/
├── github_pages/
│   ├── spa_shell.html
│   ├── mkdocs_index.html
│   └── regular_page.html
└── renderer/
    └── test_pages/
```

## Implementation Order

1. **Week 1**: Phase 1 (DocsRS Strategy) - Highest impact
2. **Week 2**: Phase 2 (Renderer Pool) + Phase 3 (Rod Renderer)
3. **Week 3**: Phase 4 (GitHub Pages) + Phase 5 (Main Package)

## Success Criteria

- [ ] Overall coverage >= 80%
- [ ] `internal/strategies` coverage >= 75%
- [ ] `internal/renderer` coverage >= 70%
- [ ] `cmd/repodocs` coverage >= 80%
- [ ] All tests pass with `make test-all`
- [ ] Coverage report generated with `make coverage`

## Verification

```bash
# Run all tests
make test-all

# Generate coverage report
go test -coverprofile=coverage.out ./...
go tool cover -html=coverage.out -o coverage.html

# Check specific package coverage
go test -coverprofile=coverage.out ./internal/strategies/...
go tool cover -func=coverage.out | grep strategies

# Verify no regressions
go test -race ./...
```

## Files to Create

1. `tests/unit/strategies/docsrs_execute_test.go`
2. `tests/unit/strategies/docsrs_json_test.go`
3. `tests/unit/strategies/docsrs_renderer_test.go`
4. `tests/unit/renderer/pool_test.go`
5. `tests/unit/renderer/rod_test.go`
6. `tests/unit/strategies/github_pages_unit_test.go`
7. `tests/fixtures/docsrs/json/*.json` (multiple fixture files)
8. `tests/fixtures/github_pages/*.html` (fixture files)

## Files to Modify

1. `tests/mocks/domain.go` - Extend if needed
2. `Makefile` - Add coverage verification targets

## References

- Existing test patterns: `tests/unit/strategies/crawler_strategy_test.go`
- Test utilities: `tests/testutil/`
- Mock patterns: `tests/mocks/`
- Coverage targets: `Makefile` (coverage-* targets)
