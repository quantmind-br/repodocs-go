# Code Structure Analysis

## Architectural Overview
The codebase is structured as a modular Go application designed for documentation extraction and processing. It follows a decoupled, interface-driven architecture where specialized strategies handle different source types (Web, Git, Wiki, etc.) while sharing a common infrastructure for fetching, rendering, converting, and storing content. 

The system operates as a pipeline:
1. **Detection**: Identifying the appropriate strategy for a given URL.
2. **Execution**: Orchestrating the extraction process (crawling, cloning, or API interaction).
3. **Processing**: Converting raw content (HTML/Git) into a standardized Markdown format with metadata.
4. **Enhancement**: (Optional) Using LLMs to enrich document metadata.
5. **Output**: Persisting the processed documentation to disk.

## Core Components
*   **Orchestrator (`internal/app`)**: The central entry point that manages the lifecycle of documentation extraction. it coordinates dependency injection, strategy selection, and execution flow.
*   **Strategy Factory**: A mechanism within the app layer that maps URLs to specific extraction implementations (e.g., `CrawlerStrategy`, `GitStrategy`, `WikiStrategy`).
*   **Converter Pipeline (`internal/converter`)**: A multi-stage processor that handles encoding conversion, content extraction (selecting main content and removing noise), HTML sanitization, and Markdown generation.
*   **Dependencies Container (`internal/strategies`)**: A shared structure providing strategies with access to the Fetcher, Renderer, Cache, Converter, Writer, and LLM services.
*   **Metadata Enhancer (`internal/llm`)**: An optional component that interacts with LLM providers (OpenAI, Anthropic, Google) to generate summaries, tags, and categories for documents.

## Service Definitions
*   **Fetcher (`internal/fetcher`)**: A high-level HTTP client providing "stealth" capabilities (User-Agent rotation, etc.) and integrated caching to minimize network load.
*   **Renderer (`internal/renderer`)**: A headless browser service (powered by `go-rod`) used for processing Single Page Applications (SPAs) or JavaScript-heavy sites that cannot be handled by static fetching.
*   **Cache (`internal/cache`)**: A persistence service using BadgerDB to store fetched content, reducing redundant requests during iterative runs.
*   **Writer/Collector (`internal/output`)**: Services responsible for organizing the final file structure, handling document overwrites, and collecting metadata into a unified JSON index.

## Interface Contracts
The application relies heavily on interfaces defined in `internal/domain` to maintain flexibility:
*   **`Strategy`**: Defines `Name()`, `CanHandle(url)`, and `Execute(ctx, url, opts)`. This allows adding new source types without modifying the core logic.
*   **`Fetcher`**: Abstracts HTTP operations, enabling both standard and cached/stealth implementations.
*   **`Renderer`**: Abstracts the headless browser interactions for JavaScript rendering.
*   **`LLMProvider`**: A contract for completion services, supporting multiple AI backends (OpenAI, Vertex AI, Anthropic).
*   **`Converter`**: Defines the transformation from raw HTML/content into the system's `Document` model.

## Design Patterns Identified
*   **Strategy Pattern**: Used to handle different documentation sources (Git, Web, Wiki, Pkg.go.dev) through a common interface.
*   **Dependency Injection**: Dependencies like the fetcher, logger, and cache are injected into strategies and the orchestrator, facilitating testing and configuration.
*   **Pipeline Pattern**: The HTML-to-Markdown conversion is implemented as a series of discrete steps (Sanitize -> Extract -> Convert -> Stats).
*   **Factory Pattern**: Used for creating LLM providers and selecting the appropriate extraction strategy based on the input URL.
*   **Circuit Breaker & Rate Limiter**: Implemented in the LLM service to protect against external API failures and quota limits.
*   **Worker Pool**: Utilized in various strategies to parallelize fetching and processing tasks.

## Component Relationships
*   **Orchestrator → Strategies**: The Orchestrator selects and executes a Strategy.
*   **Strategies → Dependencies**: Strategies use the `Dependencies` container to access cross-cutting concerns (Cache, Fetcher, Converter).
*   **Converter → Sanitizer/Markdown**: The Converter delegates low-level HTML/Markdown transformations to specialized sub-modules.
*   **Fetcher ↔ Cache**: The Fetcher interacts with the Cache to store and retrieve responses.
*   **LLMProvider ↔ MetadataEnhancer**: The Enhancer uses the Provider to fulfill AI-related tasks.

## Key Methods & Functions
*   **`orchestrator.Run()`**: The main execution loop that triggers detection and strategy execution.
*   **`DetectStrategy(url)`**: A logic gate that inspects URLs to determine the best handling approach.
*   **`converter.Pipeline.Convert()`**: The core transformation logic that turns raw input into a structured `domain.Document`.
*   **`renderer.Renderer.Render()`**: Orchestrates the headless browser tab pool to fetch and render dynamic content.
*   **`llm.MetadataEnhancer.Enhance()`**: Manages the prompt engineering and retry logic for AI-assisted metadata extraction.
*   **`strategies.CrawlerStrategy.Execute()`**: Uses `colly` for asynchronous web crawling and link discovery.