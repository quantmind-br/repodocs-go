# AGENTS.md - tests/

External test suite using `package_test` pattern for black-box testing of internal packages.

## Structure

```
tests/
├── unit/              # External package tests (mirrors internal/)
│   ├── fetcher/       # fetcher_test package tests internal/fetcher
│   ├── converter/     # converter_test package tests internal/converter
│   ├── strategies/    # Strategy-specific tests
│   └── ...            # Other packages follow same pattern
├── integration/       # Network-dependent, cross-component tests
├── e2e/               # Full CLI workflow tests
├── benchmark/         # Performance benchmarks (*_benchmark_test.go)
├── mocks/             # Generated mocks (go.uber.org/mock)
├── testutil/          # Shared test helpers
├── fixtures/          # Static test files (HTML, archives)
└── testdata/          # Test configs, golden files, fixtures
```

## Running Tests

```bash
go test ./tests/unit/...                    # Unit tests only
go test ./tests/integration/... -tags=integration  # Integration tests
go test ./tests/e2e/...                     # End-to-end tests
go test ./tests/benchmark/... -bench=.     # Benchmarks
```

## Test Utilities (testutil/)

| Helper | Purpose |
|--------|---------|
| `NewBadgerCache(t)` | In-memory cache with auto-cleanup |
| `NewTestServer(t)` | HTTP test server with route helpers |
| `NewTestLogger(t)` | Test logger capturing output |
| `NewDocument(t)` | Factory for domain.Document |
| `AssertDocumentContent(t, doc, url, title, content)` | Validate document fields |
| `AssertFileExists(t, path)` | File existence check |
| `AssertFileContains(t, path, content)` | File content assertion |
| `TempDir(t)` / `TempOutputDir(t)` | Auto-cleanup temp directories |

## Adding Fixtures

1. Place static files in `fixtures/` by type: `git/`, `pkggo/`, `llms/`, `docsrs/`
2. Place test configs in `testdata/config/`
3. Place golden files in `testdata/golden/`
4. Reference via `testdata/fixtures/` path in tests

## Regenerating Mocks

```bash
mockgen -source=internal/domain/interfaces.go -destination=tests/mocks/domain.go -package=mocks
```

## External Package Test Pattern

Tests use `package_test` naming (e.g., `fetcher_test`) to test only exported APIs:
```go
package fetcher_test  // NOT package fetcher

import "github.com/quantmind-br/repodocs-go/internal/fetcher"
```
