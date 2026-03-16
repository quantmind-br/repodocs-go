<!-- Parent: ../AGENTS.md -->
<!-- Generated: 2026-03-15 | Updated: 2026-03-15 -->

# tests/testutil

Shared test utilities and factory functions used across all test packages. Provides helpers for caching, HTTP servers, document creation, and assertions.

## Purpose

Comprehensive test utility package offering reproducible test fixtures, auto-cleanup resources, and assertion helpers. Preferred over `tests/helpers/` for new code.

## Key Files

| File | Description |
|------|-------------|
| `assertions.go` | Document and file assertion helpers (AssertDocumentContent, AssertFileExists, AssertFileContains) |
| `cache.go` | In-memory BadgerDB cache with auto-cleanup, test cache entry factories |
| `converter.go` | NewHTMLConverter - creates test converter pipeline |
| `documents.go` | Document factories (NewDocument, NewHTMLDocument, NewMarkdownDocument, NewEmptyDocument) |
| `fetcher.go` | SimpleFetcher - deterministic mock fetcher for tests |
| `http.go` | TestServer wrapper around httptest.Server with route handlers |
| `logger.go` | Test loggers (NewTestLogger, NewNoOpLogger, NewVerboseLogger) |
| `strategies.go` | Test dependency factories (NewTestDependencies, NewMinimalDependencies) |
| `temp.go` | Temp directory helpers (TempDir, TempOutputDir, TempSubDir, CreateTempFile) |

## Subdirectories

None - flat package structure.

## For AI Agents

### Common Patterns

```go
// Cache for testing
cache := testutil.NewBadgerCache(t)

// HTTP test server
server := testutil.NewTestServer(t)
server.HandleHTML(t, "/page", "<html><body>Content</body></html>")

// Document factory
doc := testutil.NewDocument(t)

// Assertions
testutil.AssertDocumentContent(t, doc, "https://example.com", "Title", "Content")
testutil.AssertFileExists(t, "output/docs/guide.md")

// Temp directories
tmpDir := testutil.TempDir(t)
baseDir, docsDir := testutil.TempOutputDir(t)
```

### Dependencies

- **Internal:** `internal/domain`, `internal/cache`, `internal/converter`, `internal/fetcher`, `internal/output`, `internal/strategies`, `internal/utils`
- **External:** `github.com/stretchr/testify`, `github.com/rs/zerolog`, `github.com/dgraph-io/badger/v4`

<!-- MANUAL: Any manually added notes below this line are preserved on regeneration -->