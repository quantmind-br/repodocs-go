# Data Flow Analysis

## Data Models

The system centers around several key Go structures that represent information at different stages of the pipeline:

*   **`domain.Page`**: The raw stage of data. It holds the raw bytes from a fetch, status codes, and basic metadata (URL, Content-Type, fetch timestamp).
*   **`domain.Document`**: The primary data structure for processed documentation. It includes the Markdown content, original HTML, word/character counts, links, headers, and enhanced metadata fields (Summary, Tags, Category).
*   **`domain.CacheEntry`**: Represents a persisted version of a fetched page, including expiration logic.
*   **`domain.SimpleMetadataIndex` / `domain.SimpleMetadata`**: Structures optimized for LLM consumption and final indexing, stripping away technical metadata like content hashes.
*   **`domain.Frontmatter`**: Data structure specifically for YAML frontmatter generation in output Markdown files.

## Input Sources

Data enters the system via several "Strategies" that dictate how content is discovered and retrieved:

*   **Web Crawling**: Standard HTTP/HTTPS requests to websites via the `CrawlerStrategy` using `colly`.
*   **Sitemaps**: Parsing `sitemap.xml` or sitemap index files to identify target URLs.
*   **LLMS.txt**: Specialized parsing for `llms.txt` files (a standard for LLM-friendly documentation).
*   **Git Repositories**: Data enters by cloning or archiving Git repositories, processing files from the local filesystem.
*   **CLI Inputs**: User-provided configurations, URLs, and local file paths through the orchestrator.

## Data Transformations

The transformation pipeline is highly structured within the `converter.Pipeline`:

1.  **Encoding Conversion**: Raw bytes are converted to UTF-8.
2.  **Extraction**: The `ExtractContent` component uses CSS selectors (provided by user or defaults) to isolate relevant documentation from boilerplates.
3.  **Sanitization**: A `Sanitizer` removes navigation, comments, and scripts, while rewriting URLs to be relative or absolute based on configuration.
4.  **HTML-to-Markdown**: The `mdConverter` transforms the sanitized HTML into GitHub Flavored Markdown (GFM).
5.  **Metadata Enrichment**:
    *   **Heuristic**: Extraction of titles, descriptions, and headers via `goquery`.
    *   **LLM-based**: Markdown content is sent to LLM providers (OpenAI, Anthropic, or Google) to generate summaries, tags, and categories.
6.  **Stats Calculation**: Generation of word counts, character counts, and SHA256 content hashes.

## Storage Mechanisms

*   **BadgerDB**: A key-value store used for persistent caching of fetched pages. This prevents redundant network requests and respects TTLs.
*   **Local Filesystem**: The final destination for processed data.
    *   **Markdown Files**: Structured hierarchically based on the source URL or Git path.
    *   **JSON Index**: A consolidated `metadata.json` (or similar) containing a registry of all processed documents.
*   **Memory**: Temporary storage in worker pools and `sync.Map` during crawling for deduplication.

## Data Validation

Validation occurs at multiple boundaries:

*   **Input Validation**: `Orchestrator` validates URLs and configuration parameters.
*   **Content Type Validation**: `fetcher` and `strategies` check `Content-Type` headers to ensure only HTML or Markdown is processed.
*   **LLM Output Validation**: The `MetadataEnhancer` performs strict validation of LLM responses, ensuring they are valid JSON and contain the required `summary`, `tags`, and `category` fields before applying them to the `Document`.
*   **JSON Schema**: Validation of the produced metadata index during the flush process.

## Output Formats

Data leaves the system in three primary formats:

1.  **Markdown Files**: Each documentation page is saved as a `.md` file with YAML frontmatter containing metadata.
2.  **Structured JSON**: A central `index.json` or simplified metadata file providing a programmatic map of all documents for LLM applications.
3.  **Console Output**: Logs and progress bars (via `zerolog` and `progressbar`) provide real-time status to the user.
4.  **Asset Files**: If configured, static assets (images, etc.) are preserved in their original or relative directory structure.