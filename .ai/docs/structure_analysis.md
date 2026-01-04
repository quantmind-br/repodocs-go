# Code Structure Analysis

## Architectural Overview
The codebase is organized as a modular documentation extraction and processing engine written in Go. It follows a strategy-based architecture where the application identifies the type of documentation source (URL or repository) and selects an appropriate strategy for extraction. The system is built around a central orchestrator that manages a set of decoupled services (fetching, rendering, conversion, caching, and writing) through well-defined interfaces.

## Core Components
- **Orchestrator (`internal/app`)**: The central coordinator that manages the end-to-end flow of documentation extraction. It detects the source type, initializes dependencies, and executes the selected strategy.
- **Strategies (`internal/strategies`)**: Specialized modules for different documentation sources:
    - `GitStrategy`: For cloning and processing Git repositories.
    - `SitemapStrategy`: For crawling websites based on XML sitemaps.
    - `LLMSStrategy`: For processing `llms.txt` discovery files.
    - `CrawlerStrategy`: A generic web crawler for discovery and extraction.
    - `PkgGoStrategy` & `DocsRSStrategy`: Specialized handlers for Go and Rust documentation.
- **Converter Pipeline (`internal/converter`)**: A multi-stage processing engine that transforms raw HTML into clean Markdown. It handles encoding, content extraction (via CSS selectors or readability algorithms), sanitization, and metadata generation.
- **Fetcher (`internal/fetcher`)**: A high-level HTTP client with support for retries, custom headers, and stealth techniques to avoid bot detection.
- **Renderer (`internal/renderer`)**: A headless browser integration (using Rod) for rendering JavaScript-heavy documentation that cannot be fetched via static HTTP.
- **Output Engine (`internal/output`)**: Manages the persistence of processed documents, including path generation, frontmatter injection, and consolidated metadata indexing.

## Service Definitions
- **Strategy Detector**: Analyzes URLs using regex and pattern matching to map them to the most efficient extraction strategy.
- **Metadata Enhancer (`internal/llm`)**: Uses LLM providers (OpenAI, Anthropic, Google) to enrich documents with AI-generated summaries, tags, and categories.
- **Cache Service**: A persistence layer (using BadgerDB) that caches raw HTTP responses to minimize network overhead and respect rate limits.
- **Sanitizer**: A component within the converter that strips boilerplate, navigation, and scripts from HTML while preserving structural elements.

## Interface Contracts
- **`domain.Strategy`**: Defines the contract for extraction logic (`Execute`, `CanHandle`, `Name`).
- **`domain.Fetcher`**: Abstraction for content retrieval, allowing for both simple HTTP and cached fetching.
- **`domain.Renderer`**: Interface for browser-based rendering of dynamic content.
- **`domain.Cache`**: Contract for key-value storage of fetched content and metadata.
- **`domain.LLMProvider`**: Abstraction for different AI backends, providing a unified `Complete` method for text generation.
- **`domain.Converter`**: Contract for transforming HTML strings into structured `Document` models.

## Design Patterns Identified
- **Strategy Pattern**: Used to handle different documentation sources polymorphically.
- **Dependency Injection**: Shared services (Cache, Fetcher, Renderer) are injected into strategies through a `Dependencies` struct.
- **Pipeline Pattern**: The HTML-to-Markdown conversion is implemented as a series of discrete, sequential transformations.
- **Factory Pattern**: Used for creating specific strategy instances and LLM provider implementations based on configuration.
- **Decorator/Wrapper Pattern**: Used for LLM providers to add features like rate limiting, retries, and circuit breaking without modifying core logic.
- **Pool Pattern**: The renderer manages a pool of browser tabs/instances to handle concurrent rendering requests efficiently.

## Component Relationships
1. **Orchestrator** calls the **Detector** to identify the **Strategy**.
2. **Orchestrator** initializes **Dependencies** (Fetcher, Renderer, Cache, Converter, Writer).
3. The **Strategy** uses the **Fetcher** (which may use the **Cache**) or **Renderer** to retrieve content.
4. The **Strategy** passes raw content to the **Converter Pipeline**.
5. The **Converter** uses **Sanitizer** and **MarkdownConverter** to produce a `Document`.
6. If enabled, the **MetadataEnhancer** sends the document content to an **LLMProvider**.
7. The **Writer** persists the final `Document` and updates the **MetadataCollector**.

## Key Methods & Functions
- `orchestrator.Run(ctx, url, opts)`: The main entry point for the extraction process.
- `pipeline.Convert(ctx, html, url)`: The core transformation function for HTML processing.
- `detector.DetectStrategy(url)`: Maps an input string to a specific `StrategyType`.
- `fetcher.Get(ctx, url)`: Retrieves content with automatic retry and stealth headers.
- `renderer.Render(ctx, url, opts)`: Performs headless browser rendering for a specific URL.
- `writer.Write(ctx, doc)`: Persists the processed document to the file system.