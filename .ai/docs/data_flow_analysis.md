# Data Flow Analysis

## Data Models

The system architecture revolves around several key data structures that represent the lifecycle of a documentation page:

- **Page**: Represents raw content fetched from a source. Contains `URL`, `Content` ([]byte), `ContentType`, `StatusCode`, and technical metadata like `FetchedAt` and `RenderedJS`.
- **Document**: The primary model for processed data. It stores the final `Content` (Markdown), `HTMLContent` (original), `Title`, `Description`, and calculated metrics (`WordCount`, `CharCount`, `ContentHash`). It also contains LLM-enhanced fields like `Summary`, `Tags`, and `Category`.
- **CacheEntry**: A persistent wrapper for `Page` objects, stored in the database with `ExpiresAt` for TTL management.
- **Sitemap/LLMSLink**: Intermediate structures used for URL discovery during the extraction process.
- **Frontmatter**: A YAML-serializable structure used to prepend metadata to Markdown output files.
- **SimpleMetadata**: A flattened, JSON-optimized representation of a `Document` used for consolidated indexing and LLM evaluation.

## Input Sources

Data enters the system through multiple entry points depending on the detected strategy:

- **Web URLs**: Direct HTTP/HTTPS requests to documentation websites.
- **Git Repositories**: SSH or HTTPS Git URLs used for cloning or fetching repository archives.
- **Sitemaps**: XML sitemaps used to discover bulk sets of URLs.
- **llms.txt**: Specialized discovery files used to locate LLM-friendly documentation.
- **Package Registries**: Specific integrations for `pkg.go.dev` and `docs.rs`.
- **Local Cache**: Previously fetched content retrieved from the BadgerDB storage to avoid redundant network calls.

## Data Transformations

The transformation pipeline is a multi-stage process that converts raw input into structured documentation:

1.  **Encoding Normalization**: Raw bytes are converted to UTF-8 using detected charset information.
2.  **DOM Parsing**: HTML content is parsed into a searchable DOM tree using `goquery`.
3.  **Content Extraction**:
    - **Selector-based**: Uses specific CSS selectors to isolate documentation content.
    - **Readability-based**: Uses an automated algorithm (similar to Mozilla's Readability) to extract the main "article" content if no selector is provided.
4.  **Sanitization**: Removes non-content elements (navigation, scripts, styles, comments) and cleans attributes.
5.  **Markdown Conversion**: Converts the sanitized HTML fragment into clean Markdown.
6.  **Metadata Enhancement (Optional AI Step)**:
    - The Markdown content is sent to an LLM (OpenAI, Anthropic, or Google).
    - The LLM generates a `Summary`, identifies `Tags`, and assigns a `Category`.
7.  **Statistics Calculation**: Generates word counts, character counts, and a SHA-256 content hash for change detection.

## Storage Mechanisms

The system employs two distinct storage patterns:

- **Intermediate/Persistent Cache**: Uses **BadgerDB**, a high-performance key-value store. It maps URL hashes to `CacheEntry` objects, allowing the system to resume interrupted runs or refresh documentation incrementally without re-fetching everything.
- **Final Output**: Data is persisted to the **Local File System**:
    - **Markdown Files**: Individual documentation pages saved as `.md` files.
    - **Directory Structure**: Preserves the original site hierarchy or flattens it based on configuration.
    - **JSON Metadata**: A consolidated `metadata.json` file containing the `SimpleMetadataIndex` for all processed documents.

## Data Validation

Validation occurs at several boundary points:

- **Configuration Validation**: Ensures workers, timeouts, and storage paths are valid before execution.
- **URL Validation**: Parses and verifies schemes and hostnames during the strategy detection phase.
- **Content Validation**: Checks for successful HTML parsing and ensures that content extraction yielded non-empty results.
- **LLM Rate Limiting**: Implements token and request-based rate limiting with circuit breakers to ensure data flows to/from AI providers within safety limits.
- **Path Sanitization**: Ensures that generated file paths from URLs are safe for the target operating system.

## Output Formats

Data leaves the system in standardized formats designed for both human consumption and machine (LLM) ingestion:

- **Markdown**: Standardized `.md` files with consistent heading styles and fenced code blocks.
- **YAML Frontmatter**: Embedded at the top of Markdown files, providing per-file metadata (URL, Source, Word Count, AI Summary).
- **Consolidated JSON**: A structured `metadata.json` containing an array of all processed documents, their locations, and AI-generated tags, facilitating easy indexing by search engines or RAG (Retrieval-Augmented Generation) pipelines.