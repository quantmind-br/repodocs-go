# CLAUDE.md

## Quick Commands

```bash
make build              # Build binary to ./build/repodocs
make test               # Unit tests (fast, -short)
make test-integration   # Integration tests
make test-e2e           # E2E tests
make lint               # golangci-lint (v2)
make fmt                # Format code
make vet                # Run go vet
```

## Architecture

**repodocs-go** extracts documentation from websites, Git repos, sitemaps, pkg.go.dev, llms.txt and converts to Markdown.

**Flow**: URL → Detector → Strategy → Fetcher/Renderer → Converter → Writer

### Strategy Pattern
Strategies implement `internal/strategies.Strategy`:
- `Name() string`
- `CanHandle(url string) bool`
- `Execute(ctx context.Context, url string, opts Options) error`

Detection order: LLMS → PkgGo → Sitemap → Git → Crawler

### Dependency Injection
`strategies.Dependencies` is composition root:
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

### Converter Pipeline
1. UTF-8 normalization
2. Content extraction (CSS selector or Readability)
3. HTML sanitization (remove scripts, nav, ads)
4. Markdown conversion
5. Metadata extraction

## Test Structure

```
tests/
├── unit/           # Fast unit tests (make test)
├── integration/    # Network-dependent tests
├── e2e/            # Full CLI tests
├── mocks/          # Generated mocks (go.uber.org/mock)
├── testutil/       # Shared helpers
└── fixtures/       # Test HTML/data
```

## Task Tracking

Use `bd` for task tracking (see AGENTS.md).
