# Data Flow Analysis

## Data Models

The system architecture is built around several key data structures that represent information at different stages of its lifecycle:

*   **Page**: Represents a raw, unprocessed HTTP response. It contains the URL, binary content (body), content type, status code, and metadata such as fetch timestamp and rendering flags.
*   **Document**: The central domain model. It holds the processed documentation, including:
    *   **Content**: Final Markdown representation.
    *   **HTMLContent**: Original sanitized HTML.
    *   **Metadata**: Title, Description, Word/Character counts, and Content Hash (SHA256).
    *   **Structure**: Link arrays, Header mappings (h1-h6), and Source Strategy used.
*   **CacheEntry**: A persistence-optimized structure used for storing fetched content in the local cache, including expiration logic.
*   **Sitemap / LLMSLink**: Intermediate models used for discovery and URL extraction from standard documentation indices.
*   **Metadata / Frontmatter**: Specialized structures used for formatting output as JSON metadata or YAML headers in Markdown files.

## Input Sources

Data enters the system through several distinct acquisition channels:

*   **HTTP/HTTPS URLs**: Standard web pages fetched using a stealth-capable HTTP client (`tls-client`) to bypass bot detection.
*   **Headless Browser (Rod)**: Pages requiring JavaScript execution are rendered via a browser pool before processing.
*   **Git Repositories**: Cloned via SSH or HTTPS. Data is extracted directly from the repository's file system (Markdown, text, and code files).
*   **Sitemaps (XML)**: Used as a discovery source to batch-import documentation URLs.
*   **llms.txt**: A specialized discovery format for LLM-friendly documentation.
*   **Pkg.go.dev**: Specialized input for Go package documentation.

## Data Transformations

The transformation pipeline converts raw input into structured, clean Markdown through a series of sequential steps:

1.  **Encoding Normalization**: Raw bytes are detected and converted to UTF-8.
2.  **Content Extraction**: The system identifies the "main" content of a page, stripping away boilerplate using readability algorithms or CSS selectors.
3.  **Sanitization**:
    *   **Tag Removal**: Scripts, styles, forms, and navigation elements are stripped.
    *   **Attribute Cleaning**: Unsafe or unnecessary HTML attributes are removed.
    *   **URL Normalization**: Relative links and image sources are resolved to absolute URLs.
4.  **Markdown Conversion**: Sanitized HTML is transformed into GitHub Flavored Markdown (GFM).
5.  **Metadata Enrichment**: Headers are indexed, links are extracted, and descriptive metadata (title/description) is harvested from meta tags.
6.  **Statistical Analysis**: Word counts and character counts are calculated, and a unique hash is generated for version tracking.

## Storage Mechanisms

*   **BadgerDB (v4)**: A high-performance, embedded key-value store used to cache raw responses. It uses URL-based keys (hashed) and supports TTL-based expiration to prevent redundant network requests.
*   **Local Filesystem**: The primary persistence layer for the final output. Documents are stored in a structured directory hierarchy that mirrors the source URL path or a flattened structure.
*   **In-Memory Cache**: Used during active crawls to track visited URLs and prevent circular references.

## Data Validation

Validation occurs at multiple boundaries to ensure data integrity:

*   **Strategy Detection**: URLs are validated against regex patterns to determine the appropriate processing strategy (Git vs. Crawler vs. Sitemap).
*   **Content-Type Filtering**: The system validates headers to ensure only processable formats (HTML, text, Markdown) are ingested.
*   **Duplicate Detection**: Before writing to disk or fetching, the system checks the output directory and cache to avoid redundant processing.
*   **Schema Validation**: Configuration files (YAML) are validated upon loading against default schemas.
*   **Integrity Checks**: SHA256 hashes are used to detect content changes across different runs.

## Output Formats

The system produces three primary output artifacts:

*   **Markdown (.md)**: Cleaned documentation content, enriched with YAML frontmatter containing source metadata (URL, Title, Date).
*   **JSON Metadata**: Optional sidecar files containing the full `Metadata` model for integration with search engines or LLM ingestors.
*   **Console/Logs**: Real-time progress reporting via structured JSON logs or "pretty" human-readable output, including performance statistics.