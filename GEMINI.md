# GEMINI.md - Context for AI Assistants

## Project Overview

**repodocs-go** is a modular, high-performance CLI tool and Go library designed to extract documentation from various online sources (websites, Git repositories, sitemaps, specialized hubs like `pkg.go.dev`) and convert it into structured, LLM-friendly Markdown.

**Key Features:**
*   **Multi-Strategy Extraction:** Automatically detects source types (Web, Git, Sitemap, LLMS.txt, PkgGo).
*   **Stealth Fetching:** Uses `tls-client` to bypass bot detection.
*   **Headless Rendering:** Optional JS execution via Rod/Chromium.
*   **Conversion Pipeline:** HTML sanitization, Readability algorithms, and Markdown conversion.
*   **AI Metadata:** Integrates with LLMs to enrich documents (summaries, tags).
*   **Caching:** BadgerDB based persistent caching.

## Tech Stack

*   **Language:** Go 1.24.1
*   **CLI Framework:** `cobra`, `viper`
*   **Browser Automation:** `rod`
*   **HTTP Client:** `tls-client`
*   **Database:** `badger` (KV store)
*   **HTML/Markdown:** `goquery`, `html-to-markdown`

## Building and Running

The project uses a `Makefile` for standard operations.

| Command | Description |
| :--- | :--- |
| `make build` | Build the binary to `./build/repodocs` |
| `make run ARGS="..."` | Run the application in development mode |
| `make test` | Run unit tests (fast, `-short`) |
| `make test-integration` | Run integration tests |
| `make test-e2e` | Run end-to-end tests |
| `make test-all` | Run all test suites |
| `make lint` | Run `golangci-lint` |
| `make fmt` | Format code using `gofmt` |

## Development Conventions

### Architecture
The project follows a **Hexagonal Architecture** with a strong emphasis on the **Strategy Pattern**.
*   **Orchestrator (`internal/app`):** Central engine that detects input types and manages the extraction lifecycle.
*   **Strategies (`internal/strategies`):** Implementations for content discovery (Crawler, Git, Sitemap, etc.).
*   **Dependencies:** Injected via a `Dependencies` struct containing Fetcher, Renderer, Cache, etc.

### Coding Style
*   **Imports:** Organized in 3 groups (Standard Lib, External Deps, Internal Deps), separated by blank lines.
*   **Naming:**
    *   Interfaces: Verb-er suffix (`Fetcher`, `Renderer`).
    *   Structs: Noun, PascalCase (`CrawlerStrategy`).
    *   Constructors: `New` + Type (`NewClient`).
*   **Error Handling:** Use `fmt.Errorf` with `%w` for wrapping. Define sentinel errors in `internal/domain`.

### Testing
*   **Framework:** Standard `testing` package with `testify` (require/assert) and `go.uber.org/mock`.
*   **Location:** `tests/unit/<package>`, `tests/integration`, `tests/e2e`.
*   **Pattern:** Table-driven tests are preferred.

## Project Structure

*   `cmd/repodocs`: Main entry point.
*   `internal/app`: Core logic (Orchestrator, Detector).
*   `internal/strategies`: Extraction strategies (Git, Crawler, etc.).
*   `internal/fetcher`: HTTP client implementation.
*   `internal/renderer`: Headless browser management.
*   `internal/converter`: HTML-to-Markdown pipeline.
*   `internal/cache`: Caching implementation.
*   `internal/domain`: Core interfaces and models.
*   `tests`: Test suites (unit, integration, e2e, mocks).

## Active Implementation Plans

*   **Git Path Filtering:** Currently implementing support for filtering Git repository processing to specific subdirectories (e.g., handling `/tree/` URLs). See `PLAN.md` for details.
