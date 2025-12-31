# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Build & Development Commands

```bash
# Build
make build                    # Build binary to ./build/repodocs
make build-all               # Build for all platforms

# Test
make test                    # Run unit tests (fast, -short flag)
make test-integration        # Run integration tests
make test-e2e                # Run E2E tests
make test-all                # Run all test suites
go test -v -run TestName ./internal/converter/...  # Run single test

# Lint & Format
make lint                    # Run golangci-lint (v2)
make fmt                     # Format code
make vet                     # Run go vet

# Run
make run ARGS="https://example.com/docs"  # Run in dev mode
./build/repodocs https://example.com/docs  # Run built binary
```

## Architecture Overview

**repodocs-go** is a CLI tool that extracts documentation from various sources (websites, Git repos, sitemaps, pkg.go.dev, llms.txt) and converts to Markdown.

### Core Flow

```
URL → Detector → Strategy → Fetcher/Renderer → Converter Pipeline → Writer
```

### Key Components

| Package | Purpose |
|---------|---------|
| `internal/app` | **Orchestrator** coordinates extraction; **Detector** routes URLs to strategies |
| `internal/strategies` | Strategy implementations: `crawler`, `git`, `sitemap`, `llms`, `pkggo` |
| `internal/fetcher` | Stealth HTTP client (`tls-client`) with retry and caching |
| `internal/renderer` | Headless browser (Rod/Chromium) for JS rendering |
| `internal/converter` | Pipeline: Sanitize → Readability → Markdown conversion |
| `internal/cache` | BadgerDB persistent cache |
| `internal/output` | Writer for Markdown files with frontmatter |
| `internal/domain` | Interfaces (`Strategy`, `Cache`, `Renderer`) and models (`Document`, `Page`) |

### Strategy Pattern

Strategies implement `internal/strategies.Strategy` interface:
- `Name() string`
- `CanHandle(url string) bool`
- `Execute(ctx context.Context, url string, opts Options) error`

Strategy detection order in `internal/app/detector.go`:
1. LLMS (`/llms.txt`)
2. PkgGo (`pkg.go.dev`)
3. Sitemap (`sitemap.xml`)
4. Git (`github.com`, `gitlab.com`, `.git`)
5. Crawler (fallback for HTTP URLs)

### Dependency Injection

`strategies.Dependencies` struct is the composition root:
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

Sequential transformation in `internal/converter/pipeline.go`:
1. UTF-8 encoding normalization
2. Content extraction (CSS selector or Readability algorithm)
3. HTML sanitization (remove scripts, nav, ads)
4. Markdown conversion
5. Metadata extraction (headers, links, stats)

## Test Structure

```
tests/
├── unit/           # Fast unit tests (make test)
├── integration/    # Network-dependent tests (make test-integration)
├── e2e/            # Full CLI tests (make test-e2e)
├── mocks/          # Generated mocks (go.uber.org/mock)
├── testutil/       # Shared test helpers
└── fixtures/       # Test HTML/data files
```

## Task Tracking

Use `bd` for task tracking in this project.
