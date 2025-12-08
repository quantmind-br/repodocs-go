The codebase for `repodocs-go` is structured as a command-line application designed to extract documentation from various sources (websites, Git repos, sitemaps, etc.) and convert it into a standardized format, primarily Markdown. The architecture follows a clear separation of concerns, primarily using the **Strategy Pattern** to handle different input sources and a **Pipeline Pattern** for content processing.

# Code Structure Analysis

## Architectural Overview

The architecture is a layered, modular design centered around an **Orchestrator** that manages the entire extraction process.

1.  **Entry Point (`cmd/repodocs/main.go`):** Handles CLI parsing using `cobra` and configuration loading using `viper`. It initializes the core `Orchestrator` and executes the main `Run` function.
2.  **Application Layer (`internal/app`):** Contains the `Orchestrator` and `Detector`. This layer is responsible for coordinating the flow: detecting the source type, initializing dependencies, selecting the appropriate strategy, and executing it.
3.  **Domain Layer (`internal/domain`):** Defines the core business entities (`Document`, `Response`) and the key architectural contracts (interfaces) that decouple the application from infrastructure details: `Strategy`, `Fetcher`, `Renderer`, `Cache`, `Converter`, and `Writer`.
4.  **Infrastructure/Service Layers (`internal/strategies`, `internal/fetcher`, `internal/renderer`, `internal/cache`, `internal/converter`, `internal/output`):** These packages provide concrete implementations for the domain interfaces.
    *   `internal/strategies`: Implements the `Strategy` interface for different source types (Crawler, Git, Sitemap, etc.).
    *   `internal/fetcher`: Implements the `Fetcher` interface, handling HTTP requests, retries, and stealth features.
    *   `internal/renderer`: Implements the `Renderer` interface for JavaScript rendering using `rod` (a high-level Chrome DevTools Protocol library).
    *   `internal/converter`: Implements the `Converter` interface, using a multi-step `Pipeline` for HTML cleaning, content extraction, and Markdown conversion.
    *   `internal/cache`: Implements the `Cache` interface using `Badger` for persistent storage.
    *   `internal/output`: Implements the `Writer` interface for saving the final documents.

This structure resembles a **Hexagonal Architecture** (Ports and Adapters), where the `internal/domain` defines the "ports" (interfaces), and the other infrastructure packages provide the "adapters" (implementations).

## Core Components

| Component | Package | Responsibility |
| :--- | :--- | :--- |
| **Orchestrator** | `internal/app` | The central coordinator. It loads configuration, initializes all dependencies (`Dependencies` struct), detects the appropriate `Strategy`, and executes the extraction process. |
| **Detector** | `internal/app` | Determines the correct `StrategyType` (e.g., `StrategyGit`, `StrategySitemap`, `StrategyCrawler`) based on the input URL pattern. |
| **Dependencies** | `internal/strategies` | A struct that aggregates all necessary infrastructure services (`Fetcher`, `Renderer`, `Cache`, `Converter`, `Writer`, `Logger`). It acts as a Dependency Injection container for all `Strategy` implementations. |
| **Strategy Implementations** | `internal/strategies` | Concrete implementations of the `domain.Strategy` interface (e.g., `CrawlerStrategy`, `GitStrategy`, `SitemapStrategy`). Each handles the specific logic for its source type. |
| **Converter Pipeline** | `internal/converter` | A multi-step process for transforming raw HTML into a structured `domain.Document`. Steps include encoding conversion, content extraction, HTML sanitization, and Markdown conversion. |
| **Fetcher Client** | `internal/fetcher` | Handles all network I/O, including HTTP requests, retries, and applying stealth/caching logic. |
| **Renderer** | `internal/renderer` | Manages the headless browser (likely Chrome/Chromium via `rod`) for rendering JavaScript-heavy pages. |

## Service Definitions

| Service | Package | Purpose |
| :--- | :--- | :--- |
| **`config.Loader`** | `internal/config` | Loads application configuration from file and CLI flags, merging them using `viper`. |
| **`utils.Logger`** | `internal/utils` | Provides structured and verbose logging capabilities throughout the application. |
| **`cache.BadgerCache`** | `internal/cache` | Provides a persistent, key-value store implementation for the `domain.Cache` interface, used to avoid re-fetching content. |
| **`output.Writer`** | `internal/output` | Handles the final step of writing the extracted `domain.Document` (Markdown and optional JSON metadata) to the file system. |
| **`converter.Sanitizer`** | `internal/converter` | Cleans up HTML by removing unwanted elements (scripts, styles, navigation) before conversion. |
| **`converter.MarkdownConverter`** | `internal/converter` | The core logic for transforming cleaned HTML into Markdown format. |

## Interface Contracts

The `internal/domain/interfaces.go` file defines the critical contracts that establish the boundaries between the application logic and external services.

| Interface | Package | Key Methods | Role in System |
| :--- | :--- | :--- | :--- |
| **`Strategy`** | `internal/domain` | `Name()`, `CanHandle(url string)`, `Execute(ctx, url, opts)` | Defines how to extract documentation from a specific source type. |
| **`Fetcher`** | `internal/domain` | `Get(ctx, url)`, `GetWithHeaders()`, `Close()` | Abstract layer for network requests, handling caching and stealth. |
| **`Renderer`** | `internal/domain` | `Render(ctx, url, opts)`, `Close()` | Abstract layer for rendering web pages with JavaScript. |
| **`Cache`** | `internal/domain` | `Get(key)`, `Set(key, value, ttl)`, `Close()` | Defines the contract for persistent data storage. |
| **`Converter`** | `internal/domain` | `Convert(ctx, html, sourceURL)` | Defines the contract for transforming raw HTML into a structured `Document`. |
| **`Writer`** | `internal/domain` | `Write(ctx, doc)` | Defines the contract for saving the final output. |

## Design Patterns Identified

1.  **Strategy Pattern:**
    *   **Context:** `Orchestrator`
    *   **Interface:** `domain.Strategy`
    *   **Concrete Strategies:** `CrawlerStrategy`, `GitStrategy`, `SitemapStrategy`, `LLMSStrategy`, `PkgGoStrategy`.
    *   *Purpose:* Allows the application to select an extraction algorithm dynamically based on the input URL, decoupling the core logic from source-specific details.

2.  **Pipeline Pattern (Chain of Responsibility variant):**
    *   **Implementation:** `converter.Pipeline`
    *   *Purpose:* Defines a fixed sequence of processing steps (Encoding -> Extraction -> Sanitization -> Markdown Conversion) to transform HTML into a final `Document`.

3.  **Dependency Injection (via Factory):**
    *   **Implementation:** `strategies.NewDependencies`
    *   *Purpose:* The `Orchestrator` uses `NewDependencies` to construct and aggregate all required services (`Fetcher`, `Renderer`, etc.) based on configuration, which are then passed to the selected `Strategy`. This promotes loose coupling.

4.  **Singleton/Resource Pool:**
    *   **Implementation:** `internal/renderer/pool.go` (implied by `renderer.NewRenderer` which manages browser tabs/instances).
    *   *Purpose:* Manages a pool of browser instances (`rod` tabs) to efficiently handle concurrent JavaScript rendering requests.

## Component Relationships

| Component A | Relationship | Component B | Description |
| :--- | :--- | :--- | :--- |
| `main.go` | **Initializes** | `app.Orchestrator` | The CLI entry point creates the main application controller. |
| `app.Orchestrator` | **Uses** | `app.Detector` | Determines which strategy to use for the input URL. |
| `app.Orchestrator` | **Manages** | `strategies.Dependencies` | Creates and closes the shared infrastructure services. |
| `app.Orchestrator` | **Executes** | `strategies.Strategy` | Calls the `Execute` method on the selected strategy. |
| `strategies.Strategy` | **Depends on** | `strategies.Dependencies` | Uses `Fetcher`, `Renderer`, `Converter`, and `Writer` to perform its task. |
| `fetcher.Client` | **Uses** | `cache.BadgerCache` | Implements the `domain.Cache` interface to store/retrieve HTTP responses. |
| `strategies.CrawlerStrategy` | **Uses** | `fetcher.Client` & `renderer.Renderer` | Fetches/renders content, then passes it to the `converter.Pipeline`. |
| `converter.Pipeline` | **Produces** | `domain.Document` | The final, structured output object. |

## Key Methods & Functions

| Method/Function | Package | Role |
| :--- | :--- | :--- |
| `run(cmd, args)` | `cmd/repodocs` | The main execution function. Loads config, initializes logger, creates `Orchestrator`, and calls `Orchestrator.Run()`. |
| `NewOrchestrator(opts)` | `internal/app` | Factory function that sets up the entire application context, including creating the `strategies.Dependencies` based on configuration. |
| `Run(ctx, url, opts)` | `internal/app/Orchestrator` | The core application loop. Detects strategy, creates strategy instance, and executes it. |
| `DetectStrategy(url)` | `internal/app` | Contains the pattern-matching logic to classify the input URL (e.g., is it a sitemap, a git repo, or a standard web page?). |
| `NewDependencies(opts)` | `internal/strategies` | The central factory for all infrastructure components (`Fetcher`, `Renderer`, `Cache`, `Converter`, `Writer`). |
| `Execute(ctx, url, opts)` | `internal/strategies/Strategy` | The entry point for source-specific extraction logic (e.g., a crawler loop, a git clone operation, or a sitemap parser). |
| `Convert(ctx, html, sourceURL)` | `internal/converter/Pipeline` | Executes the multi-step content processing pipeline (sanitize, extract, convert to Markdown). |

## Available Documentation

The user prompt indicates that some documents are available in `./.ai/docs/`. Since I cannot list the contents of that directory, I will evaluate the documentation based on the files listed in the repository structure:

| Document Path | Content Type | Evaluation |
| :--- | :--- | :--- |
| `README.md` | Project Overview | Essential for high-level understanding and usage instructions. (Assumed to exist and be of good quality). |
| `PLAN.md` | Planning/Roadmap | Provides insight into future direction and design intent. |
| `TASKS.md` | Task Management | Provides insight into current development focus and outstanding work. |
| `/.golangci.yml` | Configuration | Documents linting and code quality standards. |
| `/.github/workflows/ci.yml` | CI/CD | Documents the automated testing and build process. |

**Documentation Quality Assessment:**

The presence of `README.md`, `PLAN.md`, and `TASKS.md` suggests a project that values documentation and planning. The code structure itself, with clear separation into `internal/domain` and well-defined interfaces, acts as a form of self-documentation for the architecture. The core logic is well-organized into distinct packages corresponding to their responsibilities (e.g., `fetcher`, `renderer`, `cache`). The lack of a dedicated `docs/` directory in the main repository structure (outside of the `.ai/docs/` path) suggests that most documentation is either in the root files or in the code comments.