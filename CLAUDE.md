# CLAUDE.md: repodocs-go Configuration

Hello Claude! This file provides persistent context for working with the `repodocs-go` codebase. This project is a powerful command-line tool written in Go designed to extract documentation from various sources (websites, Git repos, sitemaps) and convert it into clean, standardized Markdown.

## 1. Project Overview

`repodocs-go` is a documentation extraction and conversion utility. It uses a modular, pluggable architecture to handle different input sources (strategies) and a multi-step pipeline to transform raw HTML into high-quality Markdown documents, complete with caching and stealth features.

## 2. Architecture and Design Patterns

The architecture is a layered, modular design resembling a **Hexagonal Architecture** (Ports and Adapters), centered around an `Orchestrator`.

### Core Patterns:

1.  **Strategy Pattern:** Used to handle different input sources. The `internal/app/Detector` selects the appropriate `domain.Strategy` implementation (`CrawlerStrategy`, `GitStrategy`, `SitemapStrategy`) based on the input URL.
2.  **Pipeline Pattern:** Implemented in `internal/converter/Pipeline`. This defines a fixed sequence of steps (Sanitization, Readability Extraction, Markdown Conversion) to process raw HTML.
3.  **Dependency Injection (DI):** All infrastructure services (`Fetcher`, `Renderer`, `Cache`, `Converter`, `Writer`) are aggregated into a `strategies.Dependencies` struct, which acts as the DI container and is passed to the selected `Strategy`.

### Layered Structure:

*   **Domain (`internal/domain`):** Defines all core interfaces (`Strategy`, `Fetcher`, `Cache`, `Converter`) and data models (`Document`). This is the contract layer.
*   **Application (`internal/app`):** Contains the `Orchestrator` and `Detector`, managing the flow and wiring.
*   **Infrastructure (`internal/strategies`, `internal/fetcher`, `internal/cache`, etc.):** Concrete implementations of the domain interfaces.

## 3. Key Components and Responsibilities

| Component | Package | Role |
| :--- | :--- | :--- |
| **Orchestrator** | `internal/app` | Central coordinator. Loads config, initializes dependencies, selects, and executes the correct `Strategy`. |
| **Dependencies** | `internal/strategies` | The Composition Root. Aggregates and manages the lifecycle of all infrastructure services (e.g., `BadgerCache`, `FetcherClient`, `RodRenderer`). |
| **Fetcher Client** | `internal/fetcher` | Handles network I/O, retries (with exponential backoff), and stealth features using specialized clients (`fhttp`, `tls-client`). |
| **Renderer** | `internal/renderer` | Manages the headless browser (`rod`) for rendering JavaScript-heavy pages. |
| **Converter Pipeline** | `internal/converter` | Executes the multi-step transformation from raw HTML to final Markdown. |
| **BadgerCache** | `internal/cache` | Persistent, on-disk key-value store for caching fetched content. |

## 4. Development Workflow and Commands

The project uses standard Go tooling. Configuration is managed by `viper` and `cobra`.

### Common Bash Commands

| Command | Description |
| :--- | :--- |
| `go build -o repodocs ./cmd/repodocs` | Builds the main executable binary. |
| `go test ./...` | Runs all unit and integration tests. |
| `go run ./cmd/repodocs [url]` | Runs the application directly. |
| `go run ./cmd/repodocs https://example.com -o ./output --concurrency 5` | Example execution command. |
| `golangci-lint run` | Runs the linter based on the `/.golangci.yml` configuration. |

### Testing

*   Tests are located in `*_test.go` files within their respective packages.
*   Focus on mocking external dependencies (like `Fetcher` and `Cache`) when testing core logic in `internal/app` and `internal/strategies`.
*   Integration tests should verify the end-to-end flow of specific strategies (e.g., `CrawlerStrategy` or `GitStrategy`).

## 5. Conventions and Style

*   **Language:** Go (Golang).
*   **Logging:** Use `github.com/rs/zerolog` via the `internal/utils/Logger` wrapper for structured logging. Avoid `fmt.Println` in core logic.
*   **Error Handling:** Return errors explicitly. Use `fmt.Errorf("context: %w", err)` for wrapping errors and preserving stack trace/context.
*   **Decoupling:** Always depend on interfaces defined in `internal/domain` when possible, not concrete implementations from other infrastructure packages.
*   **Configuration:** All configuration must be loaded via `viper` and passed down through the `config.Config` struct.

## 6. Development Gotchas and Warnings

1.  **Chromium Dependency:** The `internal/renderer` package requires a local installation of Chromium/Chrome to function. If tests involving rendering fail, check the environment setup for `rod`.
2.  **Stealth Complexity:** The `internal/fetcher` uses specialized, non-standard Go HTTP clients (`fhttp`, `tls-client`) for stealth. Debugging network issues can be complex; ensure you understand the custom client's behavior before assuming a standard `net/http` issue.
3.  **Composition Root:** The `internal/strategies/Dependencies` struct is the central point of component wiring. If you add a new service or change a constructor signature, this is the first place to update.
4.  **Caching:** The `BadgerCache` is persistent. If you encounter stale data, try clearing the cache path configured in the CLI or config file.
