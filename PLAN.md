# Test Coverage Improvement Plan

## Executive Summary

**Current Coverage**: 68.6%
**Target Coverage**: 85%+

This plan addresses critical gaps in test coverage, prioritizing business-critical strategies and core functionality.

---

## Phase 1: Critical Strategies (P0)

### 1.1 Git Strategy Tests
**File**: `internal/strategies/git.go`
**Current Coverage**: ~5% (only NewGitStrategy partially tested)
**Target**: 80%+

#### Functions to Test

| Function | Priority | Test Approach |
|----------|----------|---------------|
| `Execute` | Critical | Integration test with mock HTTP server |
| `parseGitURL` | High | Unit tests with various URL formats |
| `detectDefaultBranch` | High | Mock HTTP responses for GitHub/GitLab APIs |
| `buildArchiveURL` | High | Unit tests for GitHub, GitLab, Bitbucket |
| `tryArchiveDownload` | Medium | Integration test with mock server |
| `downloadAndExtract` | Medium | Test with sample tar.gz files |
| `extractTarGz` | Medium | Unit test with test fixtures |
| `cloneRepository` | Low | Skip (requires git binary, covered by e2e) |
| `findDocumentationFiles` | Medium | Unit test with temp directories |
| `processFiles` | Medium | Unit test with mock converter |
| `processFile` | Medium | Unit test with sample markdown files |
| `extractTitleFromPath` | High | Unit tests with path variations |

#### Test Files to Create
```
tests/unit/strategies/git_test.go
tests/unit/strategies/git_url_test.go
tests/integration/strategies/git_integration_test.go
tests/fixtures/git/sample.tar.gz
tests/fixtures/git/docs/
```

#### Implementation Steps

1. **Create URL parsing tests** (git_url_test.go)
   ```go
   func TestParseGitURL(t *testing.T) {
       tests := []struct {
           name     string
           input    string
           wantHost string
           wantOwner string
           wantRepo string
           wantPath string
           wantErr  bool
       }{
           {"github https", "https://github.com/owner/repo", "github.com", "owner", "repo", "", false},
           {"github with path", "https://github.com/owner/repo/tree/main/docs", "github.com", "owner", "repo", "docs", false},
           {"gitlab", "https://gitlab.com/owner/repo", "gitlab.com", "owner", "repo", "", false},
           {"bitbucket", "https://bitbucket.org/owner/repo", "bitbucket.org", "owner", "repo", "", false},
           {"invalid", "not-a-url", "", "", "", "", true},
       }
       // ... test implementation
   }
   ```

2. **Create archive URL builder tests**
   ```go
   func TestBuildArchiveURL(t *testing.T) {
       tests := []struct {
           host   string
           owner  string
           repo   string
           branch string
           want   string
       }{
           {"github.com", "owner", "repo", "main", "https://github.com/owner/repo/archive/refs/heads/main.tar.gz"},
           {"gitlab.com", "owner", "repo", "main", "https://gitlab.com/owner/repo/-/archive/main/repo-main.tar.gz"},
       }
   }
   ```

3. **Create integration test with mock server**
   ```go
   func TestGitStrategy_Execute_ArchiveDownload(t *testing.T) {
       // Setup mock HTTP server serving test tar.gz
       // Test full Execute flow
   }
   ```

4. **Create test fixtures**
   - Sample tar.gz with markdown files
   - Directory structure mimicking real repos

---

### 1.2 PkgGo Strategy Tests
**File**: `internal/strategies/pkggo.go`
**Current Coverage**: ~30% (Name, CanHandle tested)
**Target**: 80%+

#### Functions to Test

| Function | Priority | Test Approach |
|----------|----------|---------------|
| `Execute` | Critical | Integration test with mock HTTP |
| `extractSections` | High | Unit test with sample HTML |

#### Test Files to Create
```
tests/unit/strategies/pkggo_test.go
tests/integration/strategies/pkggo_integration_test.go
tests/fixtures/pkggo/sample_page.html
```

#### Implementation Steps

1. **Create HTML fixture** with real pkg.go.dev structure
2. **Test extractSections** with various HTML inputs
3. **Integration test** with mock HTTP server returning fixture HTML

---

## Phase 2: High Priority (P1)

### 2.1 LLMS Strategy Tests
**File**: `internal/strategies/llms.go`
**Current Coverage**: 31.4%
**Target**: 80%+

#### Functions to Test

| Function | Priority | Test Approach |
|----------|----------|---------------|
| `Execute` | High | Integration test with mock HTTP |
| `parseLLMSLinks` | High | Unit test with sample llms.txt content |

#### Test Cases for parseLLMSLinks
```go
func TestParseLLMSLinks(t *testing.T) {
    tests := []struct {
        name    string
        content string
        want    []string
    }{
        {"simple links", "# Title\nhttps://example.com/doc1\nhttps://example.com/doc2", []string{"https://example.com/doc1", "https://example.com/doc2"}},
        {"with comments", "# Comment\nhttps://example.com/doc", []string{"https://example.com/doc"}},
        {"empty lines", "\n\nhttps://example.com/doc\n\n", []string{"https://example.com/doc"}},
        {"invalid urls", "not-a-url\nhttps://valid.com", []string{"https://valid.com"}},
    }
}
```

---

### 2.2 Sitemap Strategy - Missing Functions
**File**: `internal/strategies/sitemap.go`
**Functions**: `processSitemapIndex`, `decompressGzip`
**Current Coverage**: ~60%
**Target**: 85%+

#### Test Cases

1. **processSitemapIndex**
   ```go
   func TestProcessSitemapIndex(t *testing.T) {
       // Test with nested sitemap index
       // Test with mixed sitemaps and sitemap indexes
       // Test with invalid XML
   }
   ```

2. **decompressGzip**
   ```go
   func TestDecompressGzip(t *testing.T) {
       // Test with valid gzip content
       // Test with invalid gzip
       // Test with empty content
   }
   ```

---

## Phase 3: Medium Priority (P2)

### 3.1 Fetcher Client Cache Integration
**File**: `internal/fetcher/client.go`
**Functions**: `getFromCache`, `saveToCache`, `GetCookies`, `SetCacheEnabled`

#### Implementation

```go
// tests/unit/fetcher/client_cache_test.go

func TestClient_CacheIntegration(t *testing.T) {
    // Create mock cache
    cache := mocks.NewMockCache()
    client := NewClient(WithCache(cache))

    // Test getFromCache
    t.Run("cache hit", func(t *testing.T) {
        cache.Set("url", cachedResponse)
        resp, err := client.Get(ctx, "url")
        assert.NoError(t, err)
        assert.Equal(t, cachedResponse, resp)
    })

    // Test cache miss -> fetch -> saveToCache
    t.Run("cache miss saves to cache", func(t *testing.T) {
        // ...
    })
}

func TestClient_GetCookies(t *testing.T) {
    // Test cookie extraction from response
}

func TestClient_SetCacheEnabled(t *testing.T) {
    // Test enabling/disabling cache
}
```

---

### 3.2 Converter Missing Functions
**File**: `internal/converter/*.go`

#### Functions to Test

| File | Function | Test Approach |
|------|----------|---------------|
| `encoding.go` | `IsUTF8` | Unit test with various byte sequences |
| `encoding.go` | `GetEncoder` | Unit test with charset names |
| `markdown.go` | `DefaultMarkdownOptions` | Simple unit test |
| `readability.go` | `extractBody` | Unit test with HTML samples |
| `sanitizer.go` | `normalizeSrcset` | Unit test with srcset attributes |
| `pipeline.go` | `ConvertHTMLWithSelector` | Unit test with CSS selectors |

#### Implementation

```go
// tests/unit/converter/encoding_test.go

func TestIsUTF8(t *testing.T) {
    tests := []struct {
        name  string
        input []byte
        want  bool
    }{
        {"valid utf8", []byte("Hello, 世界"), true},
        {"ascii", []byte("Hello"), true},
        {"invalid", []byte{0xff, 0xfe}, false},
    }
}

func TestGetEncoder(t *testing.T) {
    tests := []struct {
        charset string
        wantNil bool
    }{
        {"utf-8", false},
        {"iso-8859-1", false},
        {"unknown-charset", true},
    }
}
```

---

### 3.3 Renderer Pool Functions
**File**: `internal/renderer/pool.go`
**Functions**: `Size`, `MaxSize`, `Error`

Simple getter functions - add basic unit tests:

```go
func TestTabPool_Size(t *testing.T) {
    pool := NewTabPool(5)
    assert.Equal(t, 0, pool.Size())

    tab := pool.Acquire()
    assert.Equal(t, 1, pool.Size())
}

func TestTabPool_MaxSize(t *testing.T) {
    pool := NewTabPool(5)
    assert.Equal(t, 5, pool.MaxSize())
}
```

---

## Phase 4: Low Priority (P3)

### 4.1 Cache Keys Functions
**File**: `internal/cache/keys.go`
**Functions**: `GenerateKeyWithPrefix`, `PageKey`, `SitemapKey`, `MetadataKey`

```go
func TestGenerateKeyWithPrefix(t *testing.T) {
    key := GenerateKeyWithPrefix("prefix", "https://example.com/page")
    assert.True(t, strings.HasPrefix(key, "prefix:"))
}

func TestPageKey(t *testing.T) {
    key := PageKey("https://example.com/page")
    assert.Contains(t, key, "page:")
}

func TestSitemapKey(t *testing.T) {
    key := SitemapKey("https://example.com/sitemap.xml")
    assert.Contains(t, key, "sitemap:")
}

func TestMetadataKey(t *testing.T) {
    key := MetadataKey("https://example.com/page")
    assert.Contains(t, key, "meta:")
}
```

---

### 4.2 Logger Utility Functions
**File**: `internal/utils/logger.go`
**Functions**: `NewDefaultLogger`, `NewVerboseLogger`, `With*` methods, `SetGlobalLevel`

These are convenience wrappers - consider if testing is worth the effort:

```go
func TestNewDefaultLogger(t *testing.T) {
    logger := NewDefaultLogger()
    assert.NotNil(t, logger)
}

func TestNewVerboseLogger(t *testing.T) {
    logger := NewVerboseLogger()
    assert.NotNil(t, logger)
}

func TestLogger_WithComponent(t *testing.T) {
    logger := NewLogger(LogOptions{Level: "info"})
    componentLogger := logger.WithComponent("test")
    assert.NotNil(t, componentLogger)
}
```

---

### 4.3 App Detector Functions
**File**: `internal/app/detector.go`
**Functions**: `GetAllStrategies`, `FindMatchingStrategy`

```go
func TestGetAllStrategies(t *testing.T) {
    deps := strategies.NewDependencies(...)
    allStrategies := GetAllStrategies(deps)
    assert.Len(t, allStrategies, 5) // crawler, git, sitemap, llms, pkggo
}

func TestFindMatchingStrategy(t *testing.T) {
    deps := strategies.NewDependencies(...)

    t.Run("finds git strategy", func(t *testing.T) {
        strategy := FindMatchingStrategy("https://github.com/owner/repo", deps)
        assert.Equal(t, "git", strategy.Name())
    })
}
```

---

## Test Infrastructure Improvements

### Mock Interfaces

Create comprehensive mocks for all domain interfaces:

```
tests/mocks/
├── cache_mock.go      # MockCache implementing domain.Cache
├── fetcher_mock.go    # MockFetcher implementing domain.Fetcher
├── renderer_mock.go   # MockRenderer implementing domain.Renderer
├── converter_mock.go  # MockConverter implementing domain.Converter
└── writer_mock.go     # MockWriter implementing domain.Writer
```

### Test Fixtures

```
tests/fixtures/
├── git/
│   ├── sample-repo.tar.gz
│   └── docs/
│       ├── README.md
│       └── guide.md
├── pkggo/
│   └── sample_page.html
├── sitemap/
│   ├── simple.xml
│   ├── index.xml
│   └── compressed.xml.gz
├── llms/
│   └── sample.txt
└── html/
    ├── spa_react.html
    ├── static_page.html
    └── malformed.html
```

### Test Helpers

```go
// tests/helpers/http.go
func NewMockServer(t *testing.T) *httptest.Server
func MockResponse(status int, body string) http.HandlerFunc

// tests/helpers/fixtures.go
func LoadFixture(t *testing.T, path string) []byte
func TempDir(t *testing.T) string
```

---

## Implementation Timeline

### Week 1: Foundation
- [ ] Create mock interfaces
- [ ] Create test fixtures
- [ ] Create test helpers
- [ ] Git URL parsing tests

### Week 2: Git Strategy
- [ ] Git archive URL builder tests
- [ ] Git extractTarGz tests
- [ ] Git findDocumentationFiles tests
- [ ] Git integration tests

### Week 3: Other Strategies
- [ ] PkgGo strategy tests
- [ ] LLMS strategy tests
- [ ] Sitemap processSitemapIndex tests

### Week 4: Fetcher & Converter
- [ ] Fetcher cache integration tests
- [ ] Converter encoding tests
- [ ] Converter sanitizer tests

### Week 5: Cleanup
- [ ] Renderer pool tests
- [ ] Cache keys tests
- [ ] Logger tests
- [ ] App detector tests
- [ ] Coverage verification

---

## Success Metrics

| Metric | Current | Target |
|--------|---------|--------|
| Overall Coverage | 68.6% | 85%+ |
| strategies/* | ~45% | 80%+ |
| fetcher/* | ~75% | 90%+ |
| converter/* | ~70% | 85%+ |
| Functions at 0% | 47 | < 10 |

---

## Commands

```bash
# Run tests with coverage
make coverage

# Coverage for specific package
go test -coverprofile=coverage.out -coverpkg=./internal/strategies/... ./tests/...
go tool cover -func=coverage.out

# Run specific test
go test -v -run TestGitStrategy ./tests/unit/strategies/...

# Generate HTML report
go tool cover -html=coverage.out -o coverage.html
```

---

## Notes

1. **Git strategy clone tests** are intentionally deprioritized as they require git binary and are better covered by e2e tests

2. **Logger With* methods** are low priority as they're simple wrappers with no business logic

3. **Some 0% functions** may be intentionally untested if they're simple getters or deprecated code candidates

4. **Integration tests** should use `t.Parallel()` where possible for faster execution

5. Consider using **testify/mock** or **gomock** for generating mocks automatically
