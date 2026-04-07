# Testing Patterns

**Analysis Date:** 2026-04-07

## Test Framework

**Runner:**
- `testing` package (Go standard library)
- All tests use `*testing.T` parameter convention

**Assertion Library:**
- `github.com/stretchr/testify/assert` - Soft assertions, tests continue after failure
- `github.com/stretchr/testify/require` - Hard assertions, test exits on failure
- Pattern: Use `require` for setup, `assert` for actual test conditions

**Run Commands:**
```bash
make test                    # Run unit tests only (-short flag for fast tests)
make test-integration        # Run integration tests (may hit network)
make test-all               # Run all tests (unit + integration + e2e)
make coverage               # Generate coverage report (HTML in ./coverage/)
```

**Direct invocation:**
```bash
go test -v -race ./...              # Verbose with race detector
go test -short ./...                # Short tests only (marked with t.Short())
go test -run TestName ./...         # Run specific test
go test -timeout 30s ./...          # Custom timeout
```

## Test File Organization

**Location:**
- **Unit tests:** Co-located with source files or in `tests/unit/` subdirectory
  - Examples: `internal/app/orchestrator_test.go`, `tests/unit/strategies/crawler_strategy_test.go`
- **Integration tests:** `tests/integration/{category}/` directory
  - Examples: `tests/integration/strategies/git_integration_test.go`, `tests/integration/fetcher/`
- **E2E tests:** `tests/e2e/` directory
  - Examples: `tests/e2e/crawl_test.go`, `tests/e2e/config_test.go`
- **Benchmarks:** `tests/benchmark/` directory
  - Example: `tests/benchmark/git_clone_benchmark_test.go`
- **Test helpers:** `tests/helpers/` and `tests/testutil/`
  - `helpers/fixtures.go`: Fixture loading utilities
  - `helpers/http.go`: HTTP test server utilities
  - `testutil/`: Common test utilities

**Naming:**
- Test file: `{name}_test.go` (co-located) or `{name}_{suffix}_test.go` (for multiple test files)
  - Examples: `orchestrator_test.go`, `git_integration_test.go`, `git_clone_benchmark_test.go`
- Test functions: `Test{FunctionName}` or `Test{TypeName}_{Method}` or `Test{TypeName}_{Scenario}`
  - Examples: `TestOrchestrator_Run`, `TestCrawlerStrategy_CanHandle`, `TestCheckInternet`
- Sub-tests (table-driven): Named with underscores or descriptive strings
  - Example: `"successful execution"`, `"no_connection_context_timeout"`

**Package structure in tests:**
```
tests/
├── unit/                    # Fast unit tests
│   ├── strategies/
│   │   ├── crawler_strategy_test.go
│   │   ├── sitemap_strategy_test.go
│   │   └── ...
│   ├── utils/
│   │   └── logger_test.go
│   └── ...
├── integration/             # Network-dependent, slower
│   ├── strategies/
│   │   ├── git_integration_test.go
│   │   └── ...
│   ├── fetcher/
│   │   └── fetcher_integration_test.go
│   └── ...
├── e2e/                     # Full pipeline tests
│   ├── crawl_test.go
│   ├── config_test.go
│   └── ...
├── benchmark/              # Performance benchmarks
│   └── git_clone_benchmark_test.go
├── helpers/                # Test utilities
│   ├── fixtures.go
│   ├── http.go
│   └── ...
└── fixtures/               # Test data files
    ├── git/
    │   └── sample-repo.tar.gz
    ├── pkggo/
    │   └── sample_page.html
    └── ...
```

## Test Structure

**Suite Organization - Table-Driven Tests (PRIMARY PATTERN):**

All tests in the codebase use table-driven testing. Structure:

```go
func TestFunctionName(t *testing.T) {
    tests := []struct {
        name          string
        input         string
        expected      string
        setupFunc     func()
        expectedError bool
        errorContains string
    }{
        {
            name:     "successful case",
            input:    "valid",
            expected: "result",
            setupFunc: func() {
                // Setup if needed
            },
            expectedError: false,
        },
        {
            name:          "error case",
            input:         "invalid",
            expected:      "",
            expectedError: true,
            errorContains: "specific error text",
        },
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            // Arrange
            if tt.setupFunc != nil {
                tt.setupFunc()
            }

            // Act
            result, err := FunctionUnderTest(tt.input)

            // Assert
            if tt.expectedError {
                require.Error(t, err)
                if tt.errorContains != "" {
                    assert.Contains(t, err.Error(), tt.errorContains)
                }
            } else {
                require.NoError(t, err)
                assert.Equal(t, tt.expected, result)
            }
        })
    }
}
```

**Example from `cmd/repodocs/main_test.go`:**
```go
func TestCheckCacheDir(t *testing.T) {
    tests := []struct {
        name           string
        setup          func() string
        expectedResult bool
    }{
        {
            name: "cache directory exists",
            setup: func() string {
                tmpDir := testutil.TempDir(t)
                cacheDir := filepath.Join(tmpDir, "cache")
                err := os.Mkdir(cacheDir, 0755)
                require.NoError(t, err)
                return cacheDir
            },
            expectedResult: true,
        },
        // ... more cases
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            cachePath := tt.setup()
            result := checkCacheDir(cachePath)
            assert.Equal(t, tt.expectedResult, result)
        })
    }
}
```

**Patterns:**

1. **Setup pattern:**
   - Use anonymous setup functions in test struct field (most common)
   - Use `t.Cleanup()` for resource cleanup
   - Use `defer` for manual cleanup

2. **Teardown pattern:**
   - Use `t.Cleanup()` for automatic cleanup (registered in test struct's setup functions)
   - Example from `tests/helpers/fixtures.go`:
     ```go
     func TempDir(t *testing.T) string {
         tmpDir, err := os.MkdirTemp("", "repodocs-test-*")
         require.NoError(t, err)
         t.Cleanup(func() {
             os.RemoveAll(tmpDir)
         })
         return tmpDir
     }
     ```

3. **Assertion pattern:**
   - `require.{Assertion}` for critical path setup assertions
   - `assert.{Assertion}` for actual test condition checks
   - Common: `require.NoError()`, `assert.Equal()`, `assert.Contains()`, `assert.True()`

## Mocking

**Framework:** `go.uber.org/mock` (gomock)
- Generated mocks are version controlled in `tests/mocks/` or alongside tests
- Mocks generated from interfaces in `internal/domain/interfaces.go`

**Patterns:**

1. **Using mocks in tests:**
   ```go
   func TestOrchestrator_Run(t *testing.T) {
       tests := []struct {
           name         string
           strategyType StrategyType
           executeError bool
           expectError  bool
       }{
           {
               name:         "successful execution",
               strategyType: StrategyCrawler,
               executeError: false,
               expectError:  false,
           },
           {
               name:         "strategy execution error",
               strategyType: StrategyCrawler,
               executeError: true,
               expectError:  true,
           },
       }

       for _, tt := range tests {
           t.Run(tt.name, func(t *testing.T) {
               mockFactory := func(st StrategyType, deps *strategies.Dependencies) strategies.Strategy {
                   if tt.executeError {
                       return &mockErrorStrategy{name: string(st)}
                   }
                   return &mockStrategy{name: string(st)}
               }

               orch, err := NewOrchestrator(OrchestratorOptions{
                   Config:          cfg,
                   StrategyFactory: mockFactory,
               })
               require.NoError(t, err)

               ctx := context.Background()
               err = orch.Run(ctx, tt.url, OrchestratorOptions{})

               if tt.expectError {
                   assert.Error(t, err)
               } else {
                   assert.NoError(t, err)
               }
           })
       }
   }
   ```

2. **Mock strategy implementations (for testing strategy selection):**
   From `internal/app/orchestrator_test.go`:
   ```go
   type mockStrategy struct {
       name string
   }

   func (m *mockStrategy) Name() string              { return m.name }
   func (m *mockStrategy) CanHandle(url string) bool { return true }
   func (m *mockStrategy) Execute(ctx context.Context, url string, opts strategies.Options) error {
       return nil
   }

   type mockErrorStrategy struct {
       name string
   }

   func (m *mockErrorStrategy) Name() string              { return m.name }
   func (m *mockErrorStrategy) CanHandle(url string) bool { return true }
   func (m *mockErrorStrategy) Execute(ctx context.Context, url string, opts strategies.Options) error {
       return fmt.Errorf("mock error")
   }
   ```

**What to Mock:**
- External services (HTTP servers, LLM providers, git remote)
- Slow operations (network I/O, browser rendering)
- Non-deterministic behavior (random delays, system checks)
- Strategy implementations when testing orchestrator logic

**What NOT to Mock:**
- Configuration structures
- Core business logic (domain models, converters)
- Data structures and helpers
- Logger (use real logger with error level to suppress output)

## Fixtures and Factories

**Test Data - Helper Functions:**
Location: `tests/helpers/fixtures.go`

```go
// LoadFixture loads a test fixture file
func LoadFixture(t *testing.T, path string) []byte {
    t.Helper()
    root := findProjectRoot()
    fixturePath := filepath.Join(root, "tests", "fixtures", path)
    data, err := os.ReadFile(fixturePath)
    require.NoError(t, err, "Failed to load fixture: %s", fixturePath)
    return data
}

// LoadFixtureString loads a fixture as string
func LoadFixtureString(t *testing.T, path string) string {
    t.Helper()
    return string(LoadFixture(t, path))
}

// TempDir creates temporary directory with auto-cleanup
func TempDir(t *testing.T) string {
    t.Helper()
    tmpDir, err := os.MkdirTemp("", "repodocs-test-*")
    require.NoError(t, err)
    t.Cleanup(func() {
        os.RemoveAll(tmpDir)
    })
    return tmpDir
}

// TempFile creates temporary file with content
func TempFile(t *testing.T, content, pattern string) *os.File {
    t.Helper()
    tmpDir := TempDir(t)
    tmpFile, err := os.CreateTemp(tmpDir, pattern)
    require.NoError(t, err)
    _, err = tmpFile.WriteString(content)
    require.NoError(t, err)
    tmpFile.Close()
    return tmpFile
}
```

**Location:**
- `tests/fixtures/` - Test data files (HTML, JSON, YAML, tar.gz archives)
- `tests/helpers/` - Helper functions for fixture loading

**Common fixture paths:**
- `tests/fixtures/git/sample-repo.tar.gz` - Sample git repository archive
- `tests/fixtures/pkggo/sample_page.html` - Sample pkg.go.dev page
- `tests/fixtures/llms/sample.txt` - Sample llms.txt file
- `tests/fixtures/docsrs/json/` - Sample docs.rs JSON responses

## Coverage

**Requirements:**
- No explicit percentage requirement enforced in CI
- Pragmatic coverage: unit tests for logic, integration tests for flow
- Acceptance of some untested paths (e.g., system-specific checks, error paths that are hard to reproduce)

**View Coverage:**
```bash
make coverage           # Generates ./coverage/coverage.html
go tool cover -html=coverage.out
```

**Coverage profile:**
- Generated to `./coverage/coverage.out`
- HTML report: `./coverage/coverage.html`
- Includes all test packages (unit + integration + e2e)

## Test Types

**Unit Tests:**
- Scope: Single function or small unit of code
- Location: `tests/unit/` or co-located with source
- Speed: Fast (< 1ms typical)
- Approach:
  - Test individual behaviors
  - Mock external dependencies
  - Use table-driven tests
  - Example: `tests/unit/strategies/crawler_strategy_test.go` tests URL handling, HTML detection

**Integration Tests:**
- Scope: Multiple components together (but not full system)
- Location: `tests/integration/{category}/`
- Speed: Slower (may hit network, file system, browsers)
- Approach:
  - Use real fetcher, converter, writer (not mocks)
  - May use HTTP test servers for controlled responses
  - May use temporary directories for file operations
  - Example: `tests/integration/strategies/git_integration_test.go` tests actual git strategy execution

**E2E Tests:**
- Scope: Full pipeline from CLI to file output
- Location: `tests/e2e/`
- Speed: Slowest (full orchestration, multiple strategies)
- Approach:
  - Test from entry point (`cmd/repodocs/main.go`)
  - May use real network (controlled via flags)
  - Test full configuration loading and execution
  - Example: `tests/e2e/crawl_test.go` tests complete crawl operation

**Benchmark Tests:**
- Scope: Performance characteristics
- Location: `tests/benchmark/`
- Pattern: Use Go's `testing.B` parameter
- Example from `tests/benchmark/git_clone_benchmark_test.go`:
  ```go
  func BenchmarkGitClone(b *testing.B) {
      for i := 0; i < b.N; i++ {
          tmpDir, _ := os.MkdirTemp("", "bench-clone-*")
          
          b.StartTimer()
          _, err := git.PlainClone(tmpDir, false, &git.CloneOptions{
              URL: repo.url,
          })
          b.StopTimer()
          
          os.RemoveAll(tmpDir)
      }
  }
  ```

## Common Patterns

**Async Testing - Using Context and Goroutines:**

```go
func TestOrchestrator_Run_ContextCancellation(t *testing.T) {
    cfg := &config.Config{
        Cache: config.CacheConfig{Enabled: false},
        Concurrency: config.ConcurrencyConfig{
            Timeout: 10 * time.Second,
            Workers: 1,
        },
        Output: config.OutputConfig{
            Directory: t.TempDir(),
        },
        Logging: config.LoggingConfig{
            Level:  "error",
            Format: "pretty",
        },
    }

    mockFactory := func(st StrategyType, deps *strategies.Dependencies) strategies.Strategy {
        return &mockCancelStrategy{name: string(st)}
    }

    orch, err := NewOrchestrator(OrchestratorOptions{
        Config:          cfg,
        StrategyFactory: mockFactory,
    })
    require.NoError(t, err)

    ctx, cancel := context.WithCancel(context.Background())
    cancel() // Cancel immediately

    err = orch.Run(ctx, "https://example.com/docs", OrchestratorOptions{})
    assert.Error(t, err)
}
```

**Error Testing - Checking Error Messages:**

```go
func TestRun(t *testing.T) {
    tests := []struct {
        name          string
        args          []string
        expectedError bool
        errorContains string
    }{
        {
            name:          "invalid URL",
            args:          []string{"not-a-valid-url"},
            expectedError: true,
            errorContains: "invalid URL",
        },
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            // Setup
            tt.setupConfig()

            // Act
            err := run(rootCmd, tt.args)

            // Assert
            if tt.expectedError {
                require.Error(t, err)
                assert.Contains(t, err.Error(), tt.errorContains)
            } else {
                require.NoError(t, err)
            }
        })
    }
}
```

**Dependency Setup - Creating Test Dependencies:**

From `tests/unit/strategies/sitemap_strategy_test.go`:
```go
func setupSitemapTestDependencies(t *testing.T, tmpDir string) *strategies.Dependencies {
    logger := utils.NewLogger(utils.LoggerOptions{Level: "error"})
    writer := output.NewWriter(output.WriterOptions{
        BaseDir: tmpDir,
        Force:   true,
    })

    fetcherClient, err := fetcher.NewClient(fetcher.ClientOptions{
        Timeout:     10 * time.Second,
        MaxRetries:  1,
        EnableCache: false,
    })
    require.NoError(t, err)

    converterPipeline := converter.NewPipeline(converter.PipelineOptions{})

    return &strategies.Dependencies{
        Logger:    logger,
        Writer:    writer,
        Fetcher:   fetcherClient,
        Converter: converterPipeline,
    }
}

func TestNewSitemapStrategy(t *testing.T) {
    tmpDir := t.TempDir()
    deps := setupSitemapTestDependencies(t, tmpDir)
    
    strategy := strategies.NewSitemapStrategy(deps)
    assert.NotNil(t, strategy)
}
```

**HTTP Server Testing - Mock Servers:**

From `tests/helpers/http.go`:
```go
// Use httptest.NewServer for controlled HTTP responses
server := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
    w.WriteHeader(http.StatusOK)
    w.Write([]byte("<html>...</html>"))
}))
defer server.Close()

// Pass server.URL to code under test
```

**Platform-Specific Skipping:**

From `cmd/repodocs/main_test.go`:
```go
func TestCheckWritePermissions_Denied(t *testing.T) {
    // Skip permission-based tests on Windows
    if runtime.GOOS == "windows" {
        t.Skip("Unix file permissions not supported on Windows")
    }
    // ... test Unix-specific behavior
}
```

**Concurrent Testing:**

From `cmd/repodocs/main_test.go`:
```go
func TestCheckWritePermissions_Concurrent(t *testing.T) {
    // Run multiple checks concurrently
    var wg sync.WaitGroup
    results := make([]bool, 10)
    var mu sync.Mutex

    for i := 0; i < 10; i++ {
        wg.Add(1)
        go func(idx int) {
            defer wg.Done()
            tmpFile := fmt.Sprintf(".repodocs_test_write_%d", idx)
            f, err := os.Create(tmpFile)
            if err != nil {
                mu.Lock()
                results[idx] = false
                mu.Unlock()
                return
            }
            f.Close()
            os.Remove(tmpFile)
            mu.Lock()
            results[idx] = true
            mu.Unlock()
        }(i)
    }

    wg.Wait()

    for _, result := range results {
        assert.True(t, result)
    }
}
```

**Test Injection (for main.go functions):**

From `cmd/repodocs/main.go`:
```go
var (
    osStat       = os.Stat          // Injectable for testing
    execLookPath = exec.LookPath     // Injectable for testing
)

// Used in checkChrome():
if _, err := osStat(path); err == nil {
    return path
}
```

From `cmd/repodocs/main_test.go`:
```go
func TestCheckChrome_AllPaths(t *testing.T) {
    originalStat := osStat
    originalLookPath := execLookPath

    defer func() {
        osStat = originalStat
        execLookPath = originalLookPath
    }()

    t.Run("chrome found via os.Stat", func(t *testing.T) {
        osStat = func(name string) (os.FileInfo, error) {
            if name == "/usr/bin/google-chrome" {
                return nil, nil  // Simulate found
            }
            return nil, &os.PathError{...}
        }

        result := checkChrome()
        assert.Equal(t, "/usr/bin/google-chrome", result)
    })
}
```

---

*Testing analysis: 2026-04-07*
