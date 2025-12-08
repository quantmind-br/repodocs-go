# AGENTS.md: Universal AI Agent Configuration

## Project Overview
`repodocs-go` is a Go CLI tool for extracting documentation from various sources (web, Git, sitemaps) and converting the content into clean, standardized Markdown. It uses a modular, interface-driven architecture for high extensibility.

## Build & Test Commands
Use standard Go tooling. The main binary is built from `cmd/repodocs`.

| Action | Command |
| :--- | :--- |
| **Build** | `go build -o repodocs ./cmd/repodocs` |
| **Test** | `go test ./...` |
| **Lint** | `golangci-lint run` |
| **Run Example** | `go run ./cmd/repodocs https://example.com -o ./output` |

## Architecture
The system is built on a layered, Hexagonal Architecture pattern.

*   **Strategy Pattern:** Used for source-specific extraction logic (e.g., `CrawlerStrategy`, `GitStrategy`). The `Orchestrator` selects the strategy.
*   **Pipeline Pattern:** The `internal/converter` package uses a fixed pipeline (Sanitize -> Readability -> Markdown) for content transformation.
*   **Domain Layer:** `internal/domain` defines all core interfaces (`Strategy`, `Fetcher`, `Cache`) to decouple application logic from infrastructure.
*   **Dependencies:** Infrastructure services are aggregated and injected via the `strategies.Dependencies` struct (the Composition Root).

## Key Conventions
1.  **Language:** Go (Golang).
2.  **Configuration:** Managed by `cobra` and `viper`. All settings are passed via the `config.Config` struct.
3.  **Logging:** Use the `internal/utils/Logger` wrapper for structured logging (`zerolog`).
4.  **Caching:** Persistent caching is handled by `internal/cache/BadgerCache`.
5.  **Web Rendering:** JavaScript-heavy pages are rendered using `internal/renderer` (requires Chromium/Chrome via `rod`).

## Git Workflow
*   **Branching:** Feature branches off `main`.
*   **Commits:** Use conventional commits (e.g., `feat: add new strategy`, `fix: resolve cache issue`).
*   **Pull Requests:** Require at least one approval and passing CI checks (`.github/workflows/ci.yml`) before merging into `main`.
