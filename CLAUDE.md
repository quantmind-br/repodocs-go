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
make test                    # Unit tests with race detection
make test-integration        # Integration tests only
make test-e2e                # End-to-end tests
make test-all                # All tests
make coverage                # Generate HTML coverage report

# Run single test
go test -v -run TestName ./path/to/package/...

# Test coverage
make coverage                # Generate HTML coverage report (opens browser)

# Coverage for internal packages (tests are in separate tests/ directory)
go test -coverprofile=coverage.out -coverpkg=./internal/... ./tests/unit/... ./tests/integration/...
go tool cover -func=coverage.out        # Show coverage by function
go tool cover -html=coverage.out        # Open HTML coverage report

# Code quality
make lint                    # Run golangci-lint
make fmt                     # Format code
make vet                     # Static analysis

# Development
make run ARGS="https://example.com -o ./output"
make install                 # Install to ~/.local/bin
make deps                    # Download and tidy dependencies
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
| `internal/strategies` | Strategy implementations + Dependencies DI container |
| `internal/fetcher` | Stealth HTTP client (tls-client), retry with exponential backoff |
| `internal/renderer` | Headless Chrome via rod, tab pooling |
| `internal/converter` | Pipeline: Encoding → Readability → Sanitization → Markdown |
| `internal/cache` | BadgerDB persistent cache |
| `internal/output` | Markdown + JSON metadata file writing |

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

## Gotchas

1. **Chromium required**: The renderer package needs local Chromium/Chrome. Run `repodocs doctor` to check.
2. **Non-standard HTTP clients**: The fetcher uses `fhttp` and `tls-client` for stealth - these behave differently from `net/http`.
3. **Persistent cache**: BadgerDB cache survives between runs. Clear cache directory or use `--no-cache` if encountering stale data.
4. **Tab pooling**: The renderer manages a browser tab pool - check `internal/renderer/pool.go` when debugging concurrency issues.
