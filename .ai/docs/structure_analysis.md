# Code Structure Analysis

## Architectural Overview
The project is a Go-based documentation extraction and conversion tool designed with a modular, strategy-oriented architecture. It follows a decoupled service-based approach where core logic is encapsulated in the `internal` package, strictly separating domain definitions, infrastructure services, and high-level orchestration. The system is designed for extensibility, allowing different extraction methods (crawling, git cloning, sitemap parsing) to be plugged in through a unified strategy interface.

## Core Components

### 1. Orchestrator (`internal/app/orchestrator.go`)
The central coordinator of the application. It is responsible for:
- Detecting the appropriate extraction strategy based on the target URL.
- Managing the lifecycle of the extraction process (initialization, execution, and cleanup).
- Managing shared dependencies and injecting them into selected strategies.
- Handling high-level configuration and logging setup.

### 2. Strategy Engine (`internal/strategies/`)
A collection of specialized modules that implement the logic for retrieving documentation from various sources:
- **Crawler Strategy**: A generic web crawler using `colly` for deep-site navigation and content discovery.
- **Git Strategy**: Clones repositories directly to extract markdown files and documentation folders.
- **Sitemap Strategy**: Parses `sitemap.xml` files for efficient discovery of relevant pages.
- **Wiki Strategy**: Specifically targets GitHub-style wikis.
- **PkgGo Strategy**: Tailored for extracting Go package documentation from `pkg.go.dev`.
- **LLMS Strategy**: Implements the `llms.txt` standard for AI-consumable documentation discovery.

### 3. Conversion Pipeline (`internal/converter/pipeline.go`)
A sequential processing engine that transforms raw HTML into clean, structured Markdown. It orchestrates:
- Encoding detection and conversion to UTF-8.
- Main content extraction (removing boilerplate like navs/footers).
- HTML sanitization and element filtering.
- Conversion to Markdown with custom styling rules.
- Metadata and statistics extraction (word counts, link discovery, hashing).

### 4. Output & Metadata Management (`internal/output/`)
Handles the persistence of processed documentation.
- **Writer**: Manages file system operations, directory structuring, and frontmatter injection.
- **Metadata Collector**: Aggregates metadata across multiple processed documents to generate a consolidated `metadata.json` index file.

## Service Definitions

| Service | Responsibility |
| :--- | :--- |
| **Fetcher** | Handles HTTP communication using a "stealth" client with rotating headers and TLS fingerprinting to avoid anti-bot measures. |
| **Renderer** | Provides headless browser capabilities (via `rod`) for rendering JavaScript-heavy Single Page Applications (SPAs). |
| **Cache** | Persistent storage (via `BadgerDB`) for fetched content to enable offline processing and improve performance. |
| **LLM Provider** | Interface for interacting with AI models (OpenAI, Anthropic, Google) to enhance documentation with summaries and tags. |
| **Worker Pool** | Manages concurrency limits and task distribution for parallel fetching and processing. |

## Interface Contracts

- **`domain.Strategy`**: Defines `CanHandle(url)` for detection and `Execute(ctx, url, opts)` for execution.
- **`domain.Fetcher`**: Abstracts HTTP operations, providing methods like `Get`, `GetWithHeaders`, and access to the underlying `Transport`.
- **`domain.Renderer`**: Defines the contract for JavaScript rendering via a `Render` method.
- **`domain.LLMProvider`**: Abstracts various AI backends into a single `Complete` method.
- **`domain.Cache`**: Standardized key-value storage interface for binary content.

## Design Patterns Identified

- **Strategy Pattern**: Decouples the "how" of data extraction from the "what" (the URL), allowing dynamic selection of the best tool for the job.
- **Pipeline Pattern**: Used in the converter to process HTML through a series of discrete, sequential transformations.
- **Factory Pattern**: Utilized in `orchestrator.go` and `llm/provider.go` to instantiate strategies and AI providers based on configuration or URL patterns.
- **Decorator/Wrapper Pattern**: Used for LLM providers to transparently add rate-limiting, retries, and circuit-breaking capabilities without modifying the base provider logic.
- **Dependency Injection**: The `Dependencies` struct acts as a service container, which the `Orchestrator` populates and injects into strategies.

## Component Relationships

- **Orchestrator → Strategies**: The Orchestrator uses a `StrategyFactory` to create the appropriate `Strategy` and then calls its `Execute` method.
- **Strategies → Dependencies**: Every strategy receives a `Dependencies` object, giving it access to the `Fetcher`, `Converter`, `Renderer`, and `Writer`.
- **Fetcher → Cache**: The `Fetcher` is typically wrapped with caching logic, checking the `Cache` service before performing network operations.
- **Converter → Sanitizer/Extractor/Markdown**: The `Pipeline` coordinates these internal specialized classes to perform the actual transformation logic.
- **Writer → MetadataCollector**: The `Writer` notifies the `Collector` every time a file is successfully saved to build the final index.

## Key Methods & Functions

- `DetectStrategy(url string)`: The logic engine that maps URL patterns to specific extraction strategies.
- `Pipeline.Convert(ctx, html, url)`: The core transformation function that turns raw web content into structured domain documents.
- `MetadataEnhancer.Enhance(ctx, doc)`: Orchestrates LLM calls to generate summaries and categories for a document.
- `StealthHeaders(userAgent)`: Generates sophisticated browser-like headers to minimize bot detection.
- `NewDependencies(opts)`: A complex constructor that wires together the entire infrastructure stack based on application configuration.