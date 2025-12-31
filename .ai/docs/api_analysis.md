# API Analysis

## Project Type
RepoDocs is a Go-based Command Line Interface (CLI) application and library designed to extract documentation from various sources (websites, git repositories, sitemaps, etc.) and convert them into Markdown.

## Endpoints Overview
No HTTP endpoints - this is a CLI application and Go library.

## Authentication
Not applicable for the application itself. The tool performs unauthenticated requests to public resources or uses local system git credentials for repository access.

## Detailed Endpoints
Not applicable.

## Programmatic API (Go Library)
The application can be used as a library by importing the `internal/app` and `internal/config` packages.

### `app.Orchestrator`
The main entry point for programmatic usage.

- **`NewOrchestrator(opts OrchestratorOptions) (*Orchestrator, error)`**
    - Initializes the orchestrator with configuration and dependency injection.
- **`Run(ctx context.Context, url string, opts OrchestratorOptions) error`**
    - Executes the documentation extraction for a given URL.
- **`Close() error`**
    - Releases resources like cache connections and browser instances.
- **`ValidateURL(url string) error`**
    - Checks if a URL format is supported by any available strategy.

### `config.Config`
The configuration structure used to customize the behavior of the orchestrator.

```go
type Config struct {
    Output      OutputConfig      // Output directory and format
    Concurrency ConcurrencyConfig // Worker limits and timeouts
    Cache       CacheConfig       // BadgerDB cache settings
    Rendering   RenderingConfig   // Chrome/Rod rendering settings
    Stealth     StealthConfig     // Bot detection avoidance settings
    Exclude     []string          // Global regex patterns to exclude
    Logging     LoggingConfig     // Log levels and formats
}
```

## CLI Commands
The primary interface for users is the `repodocs` binary.

### `repodocs [url]`
Extracts documentation from the provided URL.

**Global Flags:**
| Flag | Short | Description | Default |
| :--- | :--- | :--- | :--- |
| `--config` | | Path to config file | `~/.repodocs/config.yaml` |
| `--output` | `-o` | Output directory | `./docs` |
| `--concurrency` | `-j` | Number of concurrent workers | `5` |
| `--limit` | `-l` | Max pages to process (0=unlimited) | `0` |
| `--max-depth` | `-d` | Max crawl depth | `4` |
| `--exclude` | | Regex patterns to exclude | |
| `--filter` | | Base URL filter | |
| `--nofolders` | | Flat output structure | `false` |
| `--force` | | Overwrite existing files | `false` |
| `--verbose` | `-v` | Verbose output | `false` |
| `--no-cache` | | Disable caching | `false` |
| `--cache-ttl` | | Cache TTL | `24h` |
| `--render-js` | | Force JavaScript rendering | `false` |
| `--timeout` | | Request timeout | `30s` |
| `--json-meta` | | Generate JSON metadata files | `false` |
| `--dry-run` | | Simulate without writing files | `false` |
| `--split` | | Split output (e.g., pkg.go.dev) | `false` |
| `--include-assets` | | Include referenced images (git) | `false` |
| `--user-agent` | | Custom User-Agent string | |
| `--content-selector` | | CSS selector for main content | |

### `repodocs doctor`
Checks system dependencies (Internet connection, Chrome/Chromium installation, write permissions).

### `repodocs version`
Prints version information.

## Common Patterns
- **Strategy Pattern**: The tool automatically detects the source type (Git, Sitemap, pkg.go.dev, llms.txt, or Generic Web) and applies the appropriate extraction strategy.
- **Stealth Mode**: Uses custom User-Agents and browser-level rendering to bypass bot protection.
- **Caching**: Uses BadgerDB to cache raw HTML content and rendered results, respecting the configured TTL.
- **Graceful Shutdown**: Listens for SIGINT/SIGTERM to cancel contexts and safely close resource pools.