# Code Structure Analysis

## Architectural Overview
The codebase follows a **Strategy-driven Orchestration** architecture. A central orchestrator coordinates the execution flow, but the specific logic for content discovery and extraction is delegated to specialized strategies. The system is designed to handle various documentation sources (websites, Git repositories, sitemaps, Go packages) by decoupling URL detection from the extraction process.

The architecture is highly modular, utilizing a **Service Container** pattern via a `Dependencies` structure that provides shared services like fetching, rendering, caching, and conversion to all strategy implementations. This ensures consistent behavior (e.g., stealth mode, retry logic) across different extraction methods.

## Core Components
*   **Orchestrator (`internal/app`):** The main entry point that manages the lifecycle of the extraction process. it detects the source type, initializes the appropriate strategy, and executes the pipeline.
*   **Strategy Factory (`internal/app/detector.go`):** Identifies the correct extraction strategy based on URL patterns and keywords.
*   **Extraction Strategies (`internal/strategies`):** Specialized modules (Crawler, Git, Sitemap, LLMS, PkgGo) that implement the discovery and processing logic for different documentation formats.
*   **Conversion Pipeline (`internal/converter`):** A multi-stage processor that transforms raw HTML into clean, structured Markdown, handling encoding, sanitization, and metadata extraction.
*   **Stealth Fetcher (`internal/fetcher`):** A sophisticated HTTP client that uses TLS fingerprinting and browser-like headers to avoid bot detection.
*   **JavaScript Renderer (`internal/renderer`):** A headless browser integration (using Rod) that executes JavaScript on pages where content is dynamically loaded.

## Service Definitions
*   **`Fetcher`**: Responsible for raw content retrieval. It implements retry logic, status code handling, and integrates directly with the cache.
*   **`Renderer`**: Provides DOM rendering capabilities. It manages a pool of browser tabs to allow concurrent rendering while minimizing resource overhead.
*   **`Cache`**: A persistent storage layer (backed by BadgerDB) that stores fetched responses to prevent redundant network requests and improve performance.
*   **`Converter`**: Handles the transformation of HTML to Markdown. It includes specialized filters for readability and sanitization.
*   **`Writer`**: Manages the persistence of processed documents to the local filesystem, organizing them into a structured directory or a flat layout.

## Interface Contracts
*   **`Strategy`**: Defines `CanHandle(url)` to check compatibility and `Execute(ctx, url, opts)` to perform the work.
*   **`Fetcher`**: Defines standard HTTP methods (`Get`, `GetWithHeaders`) with a specialized `Response` model.
*   **`Renderer`**: Provides a `Render` method that takes `RenderOptions` (wait selectors, scrolling, timeouts).
*   **`Cache`**: A standard Key-Value contract (`Get`, `Set`, `Has`, `Delete`).
*   **`Converter`**: Transforms HTML strings into the `domain.Document` model.
*   **`Writer`**: Handles the physical writing of `domain.Document` to disk.

## Design Patterns Identified
*   **Strategy Pattern:** Used for discovery and extraction logic, allowing the system to extend support for new sources easily.
*   **Dependency Injection:** Shared services are bundled into a `Dependencies` struct and injected into strategies during initialization.
*   **Pipes and Filters:** The `Converter` pipeline processes HTML through a series of filters (Sanitizer -> Readability -> Markdown -> Statistics).
*   **Object Pool:** The `TabPool` in the renderer manages browser tabs to optimize resource usage during concurrent processing.
*   **Retry Pattern:** A generic `Retrier` component is used in the fetcher to handle transient network errors and rate limiting.
*   **Factory Pattern:** The `CreateStrategy` function acts as a factory for strategy instances based on detected types.

## Component Relationships
1.  **Orchestrator** calls **Detector** to determine the **StrategyType**.
2.  **Orchestrator** uses **StrategyFactory** to instantiate a **Strategy** with a shared **Dependencies** container.
3.  **Strategy** uses the **Fetcher** or **Renderer** to retrieve content.
4.  **Fetcher** checks the **Cache** before performing network requests.
5.  **Strategy** passes retrieved HTML to the **Converter Pipeline**.
6.  **Converter** returns a structured **Document** model.
7.  **Strategy** passes the **Document** to the **Writer** for filesystem persistence.

## Key Methods & Functions
*   `app.DetectStrategy(url)`: Pattern matches URLs to determine the best extraction approach.
*   `converter.Pipeline.Convert(...)`: Orchestrates the full HTML-to-Markdown transformation.
*   `fetcher.Client.GetWithHeaders(...)`: Performs stealthy HTTP requests with integrated caching and retries.
*   `renderer.Renderer.Render(...)`: Manages headless browser navigation, JS execution, and DOM extraction.
*   `strategies.CrawlerStrategy.Execute(...)`: Implements asynchronous web crawling with depth control and domain filtering.
*   `output.Writer.Write(...)`: Handles directory creation, frontmatter injection, and metadata serialization.