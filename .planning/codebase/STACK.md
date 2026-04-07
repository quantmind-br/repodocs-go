# Technology Stack

**Analysis Date:** 2026-04-07

## Languages

**Primary:**
- Go 1.24.1 - The entire codebase and CLI tool is written in Go

**Secondary:**
- YAML - Configuration files and manifest definitions

## Runtime

**Environment:**
- Go 1.24.1
- Linux/macOS/Windows compatible (cross-platform binary builds)

**Package Manager:**
- Go modules
- `go.mod` and `go.sum` for dependency management
- Lock file: present and maintained

## Frameworks

**Core Framework:**
- Cobra 1.10.2 - CLI framework for command parsing and subcommands
  - Location: `cmd/repodocs/main.go`
  - Implements main CLI structure with global flags and subcommands

**Configuration Management:**
- Viper 1.21.0 - Configuration loading and management
  - Location: `internal/config/`
  - Binds CLI flags to configuration values
  - Supports YAML and environment variable overrides

**CLI/TUI:**
- Charmbracelet Bubbletea 1.3.6 - Terminal UI framework
- Charmbracelet Huh 0.8.0 - Form and prompt components
- Charmbracelet Lipgloss 1.1.0 - Terminal styling
  - Location: `internal/tui/`
  - Used for interactive config editor (`config edit` command)

**HTTP Client & Transport:**
- tls-client (bogdanfinn/tls-client v1.11.2) - Stealth HTTP client with TLS fingerprinting
- fhttp (bogdanfinn/fhttp v0.6.2) - HTTP client matching Chrome headers
  - Location: `internal/fetcher/`
  - Provides anti-bot detection capabilities

**Browser Automation:**
- Rod v0.116.2 - Headless browser control via DevTools Protocol
- Rod Stealth v0.4.9 - Anti-automation detection bypass
  - Location: `internal/renderer/`
  - Renders JavaScript for dynamic sites
  - Controls Chrome/Chromium process

**Web Parsing:**
- goquery v1.11.0 - jQuery-like DOM querying
- go-readability v0.0.0-20251205110129-5db1dc9836f0 - Content extraction algorithm
- html-to-markdown/v2 v2.5.0 - HTML to Markdown conversion
  - Location: `internal/converter/`
  - Complete HTML to Markdown pipeline

**Version Control:**
- go-git/v5 v5.16.4 - Pure Go Git implementation
  - Location: `internal/git/`
  - Clones and processes Git repositories without external `git` command

**Caching:**
- BadgerDB v4.8.0 - Embedded key-value database
  - Location: `internal/cache/`
  - Persistent cache with TTL support
  - Automatic garbage collection

**Logging:**
- zerolog v1.34.0 - Fast structured logging
  - Location: `internal/utils/`
  - JSON and pretty-print output modes

**Utilities:**
- cenkalti/backoff/v4 v4.3.0 - Exponential backoff retry strategy
  - Location: `internal/fetcher/`
- schollz/progressbar/v3 v3.18.0 - Progress bar display
- golang.org/x/text v0.31.0 - Unicode text handling
- golang.org/x/net v0.47.0 - Network utilities
- gopkg.in/yaml.v3 v3.0.1 - YAML parsing and marshaling

**Testing:**
- testify v1.11.1 - Assertion and mocking library
- go.uber.org/mock v0.5.0 - Mock generation (mockgen)
  - Location: `tests/mocks/`
  - Generated mocks for interfaces

**Build & Release:**
- goreleaser/v2 - Multi-platform binary builds
  - Used in `make release-dry` target

## Key Dependencies

**Critical:**
- tls-client (bogdanfinn) - Enables stealth mode for avoiding bot detection
  - Why: Essential for scraping protected sites without detection
  - Used in: `internal/fetcher/` for HTTP requests with fingerprint spoofing
  
- Rod (go-rod) - Headless browser automation
  - Why: Renders JavaScript and handles dynamic content
  - Used in: `internal/renderer/` for complex SPAs
  
- go-git - Git repository interaction
  - Why: Avoids external git dependency, self-contained Git support
  - Used in: `internal/git/` for cloning and processing repositories
  
- html-to-markdown - HTML to Markdown conversion
  - Why: Core document conversion pipeline
  - Used in: `internal/converter/` for content transformation
  
- BadgerDB - Persistent caching
  - Why: Reduces redundant fetches, improves performance
  - Used in: `internal/cache/` for request deduplication

**Infrastructure:**
- Cobra - CLI framework
  - Used in: `cmd/repodocs/main.go`
  - Enables subcommands and flag management

- Viper - Configuration management
  - Used in: `internal/config/`
  - Merges config files, environment variables, CLI flags

- Charmbracelet suite - Terminal UI
  - Used in: `internal/tui/`
  - Interactive configuration editor

- zerolog - Structured logging
  - Used in: `internal/utils/`
  - All logging throughout the codebase

## Configuration

**Environment:**
- Configuration file: `~/.repodocs/config.yaml` (default location)
- Can override all config values with `REPODOCS_` prefixed environment variables
  - Example: `REPODOCS_LLM_API_KEY=...`
- Loaded via Viper in `internal/config/loader.go`

**Build:**
- `Makefile` with targets: `build`, `test`, `lint`, `coverage`
- CGO_ENABLED=0 for static binary builds
- Version injection via ldflags:
  - `github.com/quantmind-br/repodocs/pkg/version.Version`
  - `github.com/quantmind-br/repodocs/pkg/version.BuildTime`
  - `github.com/quantmind-br/repodocs/pkg/version.Commit`

**Key Config Files:**
- `configs/config.yaml.template` - Template with all available options
- `go.mod` - Module definition and dependencies
- `Makefile` - Build and test automation

## Platform Requirements

**Development:**
- Go 1.24.1 or later
- golangci-lint v2 for linting
- goreleaser v2 for releases (optional)
- Chrome/Chromium for JavaScript rendering (optional but recommended)

**Production:**
- Linux, macOS, or Windows runtime (cross-platform compiled binary)
- Chrome/Chromium in PATH (required only if `--render-js` flag is used)
- Network connectivity for fetching remote documentation
- ~100MB disk space for cache directory (`~/.repodocs/cache`)

---

*Stack analysis: 2026-04-07*
