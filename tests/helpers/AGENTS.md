<!-- Parent: ../AGENTS.md -->
<!-- Generated: 2026-03-15 | Updated: 2026-03-15 -->

# tests/helpers

Legacy test helpers package. Prefer `tests/testutil/` for new code.

## Purpose

Original test utility functions. Kept for backward compatibility with existing tests. New code should use `tests/testutil/` which provides more comprehensive helpers with better auto-cleanup.

## Key Files

| File | Description |
|------|-------------|
| `fixtures.go` | Fixture loading (LoadFixture, LoadFixtureString), temp file/dir (TempDir, TempFile) |
| `http.go` | HTTP test server helpers (NewMockServer, MockResponse, MockJSONResponse, MockRedirect, MockNotFound, MockError) |

## Subdirectories

None - flat package structure.

## For AI Agents

### Migration Note

New tests should use `tests/testutil/` instead:
- `testutil.TempDir(t)` instead of `helpers.TempDir(t)`
- `testutil.NewTestServer(t)` instead of `helpers.NewMockServer(t)`
- `testutil.NewDocument(t)` for document factories
- `testutil.NewBadgerCache(t)` for cache testing

### Legacy Usage

```go
// Fixture loading
data := helpers.LoadFixture(t, "git/sample-repo.tar.gz")
content := helpers.LoadFixtureString(t, "pkggo/sample_page.html")

// HTTP server
server := helpers.NewMockServer(t)
server.Handler = helpers.MockResponse(200, "response")
```

### Dependencies

- **External:** `github.com/stretchr/testify`

<!-- MANUAL: Any manually added notes below this line are preserved on regeneration -->