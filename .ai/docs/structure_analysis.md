# Code Structure Analysis

## Architectural Overview
The codebase is a modular Go application designed for high-performance web crawling and documentation extraction. It employs a **Multi-Strategy Architecture** that allows it to handle various content sources (standard websites, Git repositories, Go package documentation, wikis, and LLM-specific files) through a unified interface. The system follows a **Pipeline-based processing flow**: detection, acquisition (fetching/rendering), conversion, enhancement, and persistence.

The architecture emphasizes **Stealth and Robustness**, utilizing advanced TLS fingerprinting to bypass anti-bot measures and headless browser rendering for JavaScript-heavy applications. Dependency Injection is used extensively via a `Dependencies` container struct, ensuring that core services are shared and easily testable.

## Core Components
*   **Orchestrator (`internal/app`)**: The central brain that coordinates the extraction lifecycle. It handles configuration validation, strategy detection, and execution flow.
*   **Strategy Detector (`internal/app/detector.go`)**: A pattern-matching engine that analyzes input URLs to determine the most effective extraction strategy.
*   **Extraction Strategies (`internal/strategies`)**: Specialized modules (Crawler, Git, Sitemap, LLMS, PkgGo, Wiki) that implement the logic for discovering and acquiring raw content from different sources.
*   **Conversion Pipeline (`internal/converter`)**: A sequential processor that transforms raw HTML into clean, LLM-ready Markdown using a series of filters (encoding, extraction, sanitization, and conversion).
*   **Metadata Enhancer (`internal/llm`)**: An optional AI-driven component that leverages Large Language Models to generate summaries, tags, and categories for processed documents.
*   **Stealth Fetcher (`internal/fetcher`)**: A specialized HTTP client that mimics modern browser TLS fingerprints to avoid detection by security services.
*   **Headless Renderer (`internal/renderer`)**: A browser-based rendering engine (using `rod`) that executes JavaScript to capture content from Single Page Applications (SPAs).

## Service Definitions
*   **Cache Service**: A persistence layer powered by **Badger DB** that stores fetched content to reduce redundant network requests and improve performance.
*   **LLM Service**: An abstraction layer for interacting with multiple AI providers (OpenAI, Anthropic, Google Gemini), including built-in rate limiting and circuit breaking.
*   **Output Service**: Manages the structured writing of Markdown files to disk, including frontmatter generation and metadata indexing.
*   **Renderer Pool Service**: Manages a pool of browser tabs to optimize resource usage during concurrent rendering tasks.

## Interface Contracts
*   **`Strategy`**: Defines the lifecycle of an extraction method (`Name`, `CanHandle`, `Execute`).
*   **`Fetcher`**: Abstracts HTTP operations, supporting custom headers, cookies, and stealth transport.
*   **`Renderer`**: Interface for browser-based rendering of dynamic content.
*   **`Converter`**: Contract for transforming HTML/raw content into structured `Document` models.
*   **`Cache`**: Standardizes key-value storage operations (Get, Set, Has, Delete).
*   **`LLMProvider`**: Abstraction for text completion services used in metadata enhancement.

## Design Patterns Identified
*   **Strategy Pattern**: Used to select different acquisition algorithms based on the source URL type.
*   **Factory Pattern**: Employed for creating strategies and LLM providers based on configuration and detection results.
*   **Pipe and Filter**: The conversion logic is organized as a series of independent filters (Sanitizer, Extractor, MarkdownConverter).
*   **Decorator Pattern**: Used to wrap LLM providers with cross-cutting concerns like rate limiting, retries, and circuit breaking.
*   **Dependency Injection**: Dependencies are injected into strategies through a centralized `Dependencies` struct.
*   **Object Pool**: The renderer maintains a `TabPool` to reuse browser instances and tabs for efficiency.

## Component Relationships
1.  **Detection Phase**: The `Orchestrator` uses the `Detector` to map a URL to a specific `Strategy`.
2.  **Dependency Assembly**: The `Orchestrator` initializes a `Dependencies` container with shared services (Fetcher, Cache, Converter, etc.).
3.  **Execution Phase**: The chosen `Strategy` uses the `Fetcher` or `Renderer` to acquire content.
4.  **Transformation Phase**: Raw content is passed through the `Converter` pipeline to produce a `Document`.
5.  **Enhancement Phase**: If enabled, the `MetadataEnhancer` interacts with the `LLMProvider` to enrich the `Document` with AI-generated metadata.
6.  **Persistence Phase**: The `Writer` saves the final document to the filesystem and updates the `MetadataCollector` for the final index.

## Key Methods & Functions
*   **`Orchestrator.Run`**: The primary entry point for the extraction process.
*   **`DetectStrategy`**: The logic that selects the acquisition method based on URL patterns.
*   **`Pipeline.Convert`**: The core transformation logic from HTML to Markdown.
*   **`MetadataEnhancer.Enhance`**: The method that manages prompt engineering and JSON extraction from LLM responses.
*   **`Client.GetWithHeaders`**: Orchestrates cached fetching with stealth capabilities and retries.
*   **`Renderer.Render`**: Handles the complexities of browser navigation, waiting for selectors, and network stability.
*   **`Writer.Write`**: Handles path generation, frontmatter injection, and file I/O.