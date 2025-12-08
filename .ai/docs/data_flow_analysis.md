# Data Flow Analysis

## Data Models Overview

The core data models are defined within the `/internal/domain` package, supplemented by configuration and caching structures. These models represent the data as it moves through the system, from raw input to final output.

| Model (Inferred) | Location | Purpose | Key Attributes (Inferred) |
| :--- | :--- | :--- | :--- |
| **Configuration** | `/internal/config/config.go` | Holds application settings, including cache paths, output directories, and strategy parameters. | `CachePath`, `OutputFormat`, `ConcurrencyLimit`, `SourceURL`. |
| **Document/Content** | `/internal/domain/models.go` | The primary entity representing the data being processed. | `SourceURL`, `RawContent` (HTML/XML), `Metadata` (Title, Date), `TransformedContent` (Markdown). |
| **Strategy Interface** | `/internal/domain/interfaces.go` | Defines the contract for data acquisition methods (e.g., `Fetch`, `Crawl`). | Methods for acquiring and returning raw content models. |
| **Cache Key** | `/internal/cache/keys.go` | A structure used to uniquely identify cached content, ensuring cache integrity. | `URL`, `StrategyType`, `ConfigurationHash`. |

## Data Transformation Map

Data transformation is centralized in the `/internal/converter` package, which implements a sequential pipeline (`pipeline.go`) to convert raw, often messy, web content into clean, structured Markdown documentation.

| Stage | Component | Input Data Format | Output Data Format | Key Transformation Logic |
| :--- | :--- | :--- | :--- | :--- |
| **1. Acquisition** | `/internal/fetcher`, `/internal/renderer` | HTTP Response (Bytes) | Raw Content Model (HTML/Text) | Network I/O, handling retries and stealth settings. |
| **2. Sanitization** | `/internal/converter/sanitizer.go` | Raw Content Model (HTML) | Sanitized HTML | Removal of non-content elements (scripts, styles, ads) to prepare for extraction. |
| **3. Readability** | `/internal/converter/readability.go` | Sanitized HTML | Core Content HTML Fragment | Algorithmically extracts the main article body, discarding navigation and boilerplate. |
| **4. Markdown Conversion** | `/internal/converter/markdown.go` | Core Content HTML Fragment | Markdown String | Converts HTML tags and structure into Markdown syntax. |
| **5. Encoding** | `/internal/converter/encoding.go` | Markdown String | Encoded Markdown String | Ensures correct character encoding (e.g., UTF-8) for final output. |

## Storage Interactions

The system employs a two-pronged approach to storage: a local, persistent cache for intermediate data and a file writer for final output.

### Caching Mechanism
*   **Technology**: BadgerDB, as indicated by `/internal/cache/badger.go`. This is an embedded, fast key-value store.
*   **Purpose**: To cache the raw fetched content (HTML/XML) based on the source URL and configuration, minimizing external network requests and improving performance.
*   **Data Flow**: The `Orchestrator` checks the cache before initiating a fetch. If a cache miss occurs, the fetched data is written to BadgerDB before proceeding to the conversion pipeline.

### Persistence Mechanism
*   **Component**: `/internal/output/writer.go`.
*   **Purpose**: To write the final, transformed Markdown document to the local filesystem.
*   **Data Flow**: The final Markdown string from the transformation pipeline is passed to the `Writer`, which handles file creation and atomic writing to the configured output path.

## Validation Mechanisms

Validation is distributed across the configuration and content processing layers to ensure data integrity and system stability.

1.  **Configuration Validation**: Handled by `/internal/config/loader.go`, ensuring that required parameters are present and correctly formatted (e.g., valid file paths, numerical limits).
2.  **URL and Input Validation**: Occurs within the strategies and the `/internal/utils/url.go` utility, verifying that input URLs are well-formed and adhere to expected protocols.
3.  **Content Integrity**: The transformation pipeline, particularly the `sanitizer.go` and `readability.go` components, implicitly validates content by checking for structural integrity (e.g., valid HTML parsing) and the presence of meaningful content. Errors related to unparsable or empty content are likely mapped to domain errors defined in `/internal/domain/errors.go`.

## State Management Analysis

The application operates primarily as a stateless processor, with state management confined to configuration and the lifecycle of a single processing request.

*   **Global State**: Managed by the immutable `Configuration` object, loaded once at startup.
*   **Request State**: The `Document` model acts as the request-scoped state, holding the data as it is transformed sequentially. The `/internal/app/orchestrator.go` manages the flow of this state through the various components (Fetcher, Cache, Converter).
*   **Concurrency State**: The `/internal/utils/workerpool.go` manages the state of concurrent tasks, particularly when processing multiple URLs (e.g., from a sitemap or crawler). This involves managing job queues, worker status, and collecting results.

## Serialization Processes

Serialization is necessary for external communication, caching, and final output formatting.

*   **Deserialization (Input)**: Raw bytes from HTTP responses (HTML, XML, JSON) are deserialized by the `Fetcher` and strategy components (e.g., `sitemap.go`) into internal Go structures or DOM representations for processing.
*   **Serialization (Caching)**: Data stored in BadgerDB is serialized (likely using a standard Go encoding like `gob` or JSON) by the `/internal/cache` layer before being written to disk.
*   **Serialization (Output)**: The final and most critical serialization is the conversion of the internal HTML/DOM representation into a **Markdown** string, handled by `/internal/converter/markdown.go`. This is the final data format persisted by the `Writer`.

## Data Lifecycle Diagrams

```mermaid
graph TD
    subgraph Initialization
        A[Config Loader] --> B(Configuration Model);
    end

    subgraph Data Acquisition
        B --> C{Orchestrator};
        C --> D[Strategy Selection];
        D --> E{Cache Check (BadgerDB)};
        E -- Cache Miss --> F[Fetcher/Renderer];
        F --> G[Raw Content Model];
        G --> H[Cache Write];
        H --> I[Raw Content Model];
        E -- Cache Hit --> I;
    end

    subgraph Transformation Pipeline
        I --> J[Sanitizer];
        J --> K[Readability Extractor];
        K --> L[Markdown Converter];
        L --> M[Final Markdown Document];
    end

    subgraph Persistence
        M --> N[Output Writer];
        N --> Z(Final Documentation File);
    end

    style A fill:#f9f,stroke:#333
    style Z fill:#ccf,stroke:#333
```