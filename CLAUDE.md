# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

`repodocs-go` is a Go CLI tool that extracts documentation from diverse sources (websites, Git repos, sitemaps, pkg.go.dev) and converts it into standardized Markdown. It handles JavaScript-heavy SPAs via headless Chrome, implements stealth HTTP features for anti-bot measures, and uses persistent caching with BadgerDB.

## Build and Development Commands

```bash
# Build
make build                    # Build binary to ./build/repodocs
make build-all               # Cross-compile for all platforms

# Test
make test                    # Unit tests with race detection (-short flag)
make test-integration        # Integration tests only
make test-e2e                # End-to-end tests
make test-all                # All tests (unit, integration, e2e)
make coverage                # Generate HTML coverage report

# Run single test
go test -v -run TestName ./tests/unit/...
go test -v -run TestName ./tests/integration/...
go test -v -run TestName ./tests/e2e/...

# Test coverage for internal packages
# Note: Tests are in separate tests/ directory, not alongside code
go test -coverprofile=coverage.out -coverpkg=./internal/... ./tests/unit/... ./tests/integration/...
go tool cover -func=coverage.out        # Show coverage by function
go tool cover -html=coverage.out        # Open HTML coverage report

# Regenerate mocks (after changing domain interfaces)
go generate ./...
# Or manually:
mockgen -source=internal/domain/interfaces.go -destination=tests/mocks/domain.go -package=mocks

# Code quality
make lint                    # Run golangci-lint with custom config (.golangci.yml)
make fmt                     # Format code
make vet                     # Static analysis

# Development
make run ARGS="https://example.com -o ./output"
make dev                     # Watch mode (requires air)
make deps                    # Download and tidy dependencies
make deps-update             # Update all dependencies

# Installation
make install                 # Install to ~/.local/bin (user installation)
make uninstall              # Remove from ~/.local/bin
make install-global         # Install to /usr/local/bin (requires sudo)
make uninstall-global       # Remove from /usr/local/bin (requires sudo)
make check-install          # Check installation status and version

# Check environment dependencies
./build/repodocs doctor      # Verify Chromium/Chrome availability
```

## Architecture

The codebase follows **Hexagonal Architecture** with the **Strategy Pattern** at its core:

```
cmd/repodocs/main.go          CLI entry point (Cobra/Viper)
        ↓
internal/app/orchestrator.go  Central coordinator
        ↓
internal/app/detector.go      Selects strategy based on URL pattern
        ↓
internal/strategies/*         Strategy implementations (Crawler, Git, Sitemap, PkgGo)
        ↓
internal/strategies/strategy.go  Dependencies struct (Composition Root)
```

### Key Packages

| Package | Responsibility |
|---------|---------------|
| `internal/domain` | Interfaces (Strategy, Fetcher, Cache, Converter, Writer) and models (Document) |
| `internal/strategies` | Strategy implementations (Crawler, Git, Sitemap, PkgGo, LLMs*) + Dependencies DI container |
| `internal/fetcher` | Stealth HTTP client (tls-client/fhttp), retry with exponential backoff, custom User-Agent |
| `internal/renderer` | Headless Chrome via rod + stealth, browser tab pooling |
| `internal/converter` | Pipeline: Encoding → Readability → Sanitization → Markdown (html-to-markdown) |
| `internal/cache` | BadgerDB persistent cache with TTL support |
| `internal/output` | Markdown + JSON metadata file writing |
| `internal/app` | Orchestrator (main coordinator) + Detector (URL pattern → Strategy selection) |
| `internal/config` | Viper-based configuration loading and management |
| `internal/utils` | Zerolog logger, WorkerPool for concurrency |

*LLMs strategy is a placeholder for future integration

### Strategy Selection

The Detector (`internal/app/detector.go`) selects the appropriate strategy based on URL patterns:

| Strategy | URL Pattern | Purpose |
|----------|-------------|---------|
| **Git** | `git@`, `.git`, `github.com`, `gitlab.com`, `bitbucket.org` | Clone Git repos and extract markdown/docs |
| **PkgGo** | `pkg.go.dev/` | Extract Go package documentation from pkg.go.dev |
| **Sitemap** | URLs ending in `.xml` or containing `sitemap` | Parse sitemap.xml and crawl listed URLs |
| **Crawler** | All other HTTP/HTTPS URLs | Crawl website starting from URL |
| **LLMs** | Not implemented | Placeholder for future LLM-based extraction |

### Design Rules

1. **Depend on interfaces**: Infrastructure packages must import `internal/domain` interfaces, not concrete types from other infrastructure packages
2. **Composition Root**: All service instantiation happens in `strategies.NewDependencies()` - update this when adding new services
3. **Strategy detection**: Update `internal/app/detector.go` when adding new source types
4. **Pipeline sequence**: Converter steps must run in order: Encoding → Readability → Sanitization → Markdown

## Conventions

**Error handling**: Wrap errors with context using `fmt.Errorf("context: %w", err)`. Define domain errors in `internal/domain/errors.go`.

**Logging**: Use zerolog via `internal/utils/Logger`. Levels: Debug (internal flow), Info (milestones), Error (failures).

**Context**: All I/O operations must accept `context.Context` as first parameter and respect cancellation.

**Configuration**: All settings flow through `config.Config` struct loaded by Viper.

## Test Organization

Tests are in a **separate `tests/` directory**, not alongside production code:

```
tests/
├── unit/           # Unit tests for internal packages
├── integration/    # Integration tests with external dependencies
├── e2e/           # End-to-end tests
├── benchmark/     # Performance benchmarks
├── mocks/         # Generated mocks (mockgen)
├── testutil/      # Test helpers and utilities
├── testdata/      # Test fixtures and golden files
└── fixtures/      # HTML/XML fixtures for strategies
```

**Key test utilities** (`tests/testutil/`):
- `NewBadgerCache(t)` - In-memory BadgerDB for tests
- `NewTestServer(t)` - httptest server with HTML fixtures
- `NewTestLogger(t)` - Zerolog test logger
- `NewDocument(t)` - Document factory
- `AssertDocumentContent(t, ...)` - Custom assertions

## Gotchas

1. **Chromium required**: The renderer package needs local Chromium/Chrome. Run `repodocs doctor` to check.
2. **Non-standard HTTP clients**: The fetcher uses `fhttp` and `tls-client` for stealth - these behave differently from `net/http`.
3. **Persistent cache**: BadgerDB cache survives between runs. Clear cache directory or use `--no-cache` if encountering stale data.
4. **Tab pooling**: The renderer manages a browser tab pool - check `internal/renderer/pool.go` when debugging concurrency issues.
5. **Test location**: Tests are in `tests/` directory, not `*_test.go` alongside code. Use `-coverpkg=./internal/...` to measure coverage correctly.
6. **Strategy detection**: URL patterns determine strategy selection in `internal/app/detector.go` - update this when adding new strategies.
7. **Pipeline order matters**: The converter pipeline steps must run in sequence (Encoding → Readability → Sanitization → Markdown) - don't reorder.
