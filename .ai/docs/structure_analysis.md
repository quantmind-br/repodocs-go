# Code Structure Analysis

## Architectural Overview
The codebase follows a modular, interface-driven architecture in Go, designed for high extensibility and testability. It is structured around an **Orchestrator** pattern that coordinates specialized **Strategies** (e.g., web crawling, Git repository extraction) using a shared set of **Dependencies**. The system employs a "Hexagonal-lite" approach where the core business logic in `internal/domain` defines interfaces that are implemented by various providers in `internal/fetcher`, `internal/cache`, `internal/llm`, etc.

## Core Components
- **Orchestrator (`internal/app/orchestrator.go`)**: The central engine that detects the appropriate strategy for a given URL and manages the end-to-end execution flow.
- **Strategy System (`internal/strategies/`)**: A collection of specialized extraction logic. Key strategies include `CrawlerStrategy` for website crawling, `GitStrategy` for repository cloning, and others like `SitemapStrategy` and `WikiStrategy`.
- **Dependency Container (`internal/strategies/strategy.go`)**: A unified structure that bundles services like the Fetcher, Renderer, Cache, and Converter, ensuring consistent resource management across different strategies.
- **Conversion Pipeline (`internal/converter/pipeline.go`)**: A multi-stage processor that transforms raw HTML into structured Markdown, handling encoding, content extraction, sanitization, and metadata generation.
- **Output Engine (`internal/output/`)**: Manages the persistence of documents and metadata to the filesystem, supporting both flat and hierarchical directory structures.

## Service Definitions
- **Fetcher (`internal/fetcher/`)**: An HTTP client service with "stealth" capabilities (custom User-Agents, retry logic) to retrieve raw content.
- **Renderer (`internal/renderer/`)**: A headless browser service (using Rod) that executes JavaScript on pages where content is dynamically loaded.
- **Cache (`internal/cache/`)**: A persistent storage service (using BadgerDB) that caches HTTP responses to improve performance and reduce network load.
- **LLM Service (`internal/llm/`)**: Provides AI-powered metadata enhancement, supporting multiple providers (OpenAI, Anthropic, Google) with built-in rate limiting and circuit breaking.

## Interface Contracts
The application relies on several critical interfaces defined in `internal/domain/interfaces.go`:
- **`Strategy`**: Defines `Name()`, `CanHandle(url)`, and `Execute(ctx, url, opts)`.
- **`Fetcher`**: Abstracts HTTP operations with `Get()` and `GetWithHeaders()`.
- **`Renderer`**: Abstracts browser-based rendering via `Render(ctx, url, opts)`.
- **`Converter`**: Defines the transformation from HTML string to `domain.Document`.
- **`LLMProvider`**: Abstracts interactions with AI models via `Complete(ctx, req)`.

## Design Patterns Identified
- **Strategy Pattern**: Dynamically selects the extraction method based on the input URL (e.g., switching from `GitStrategy` for GitHub links to `CrawlerStrategy` for standard sites).
- **Dependency Injection**: Services are injected into the `Dependencies` struct, which is then passed to strategies, facilitating easier mocking in tests.
- **Middleware/Wrappers**: Used in the LLM service to add rate-limiting and circuit-breaking functionality around standard providers.
- **Pipeline Pattern**: Used in the converter to process HTML through a sequence of discrete steps (UTF-8 conversion -> Extraction -> Sanitization -> MD Conversion).
- **Factory Pattern**: The `Orchestrator` uses a factory function to instantiate strategies based on detected types.

## Component Relationships
1. The **CLI/User** interacts with the **Orchestrator**.
2. The **Orchestrator** uses a detector to identify the URL type and selects a **Strategy**.
3. The **Strategy** utilizes the **Dependency** container to access the **Fetcher** or **Renderer**.
4. Raw content is passed to the **Converter Pipeline**, which produces a `domain.Document`.
5. If enabled, the **LLM MetadataEnhancer** processes the document to add AI-generated tags and summaries.
6. The final document is handed to the **Writer** for disk persistence.

## Key Methods & Functions
- `Orchestrator.Run()`: The primary entry point for the documentation extraction process.
- `Pipeline.Convert()`: The core transformation function that turns raw web content into Markdown documents.
- `MetadataEnhancer.Enhance()`: Orchestrates the LLM prompt-response flow to enrich document metadata.
- `DetectStrategy()`: Logic that analyzes a URL to decide which strategy to deploy.
- `NewDependencies()`: Bootstraps the entire service layer, including caching and browser rendering.