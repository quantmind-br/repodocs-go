# Data Flow Analysis

## Data Models
The system relies on several core data structures to manage the documentation lifecycle:

*   **Page**: Represents raw content fetched from a source (HTTP or Git). Contains fields for URL, raw byte content, content type, and metadata like status code and fetch time.
*   **Document**: The primary internal model for processed documentation. It includes original URL, title, description, content (Markdown), word/char counts, extracted links, and headers. It also holds AI-enhanced fields like summary, tags, and category.
*   **CacheEntry**: Used for persistence in the Badger database, storing raw page content with expiration data.
*   **Sitemap & SitemapURL**: Intermediate structures for parsing XML sitemaps to discover discovery links.
*   **SimpleMetadata & SimpleMetadataIndex**: Simplified versions of the Document model used for final JSON output, optimized for consumption by LLMs and other tools.
*   **Frontmatter**: A structure representing YAML frontmatter added to the top of generated Markdown files.

## Input Sources
Data enters the system through multiple specialized strategies based on the source type:

*   **Web URLs**: Direct HTTP/HTTPS crawling of websites.
*   **Git Repositories**: Cloning or fetching documentation directly from Git providers (GitHub, GitLab, Bitbucket).
*   **Specialized Endpoints**: 
    *   `llms.txt`: Discovery via specialized LLM-friendly index files.
    *   `Sitemaps`: Discovery via `sitemap.xml` files.
    *   `GitHub Wiki`: Specialized handling for wiki repositories.
    *   `pkg.go.dev`: Extraction of Go documentation.

## Data Transformations
Information undergoes a rigorous multi-stage pipeline as it moves through the system:

1.  **Normalization**: Raw bytes are converted to UTF-8 encoding.
2.  **Rendering**: If enabled or required (detected by DOM markers), raw HTML is processed through a headless browser (Rod) to execute JavaScript and capture dynamic content.
3.  **Extraction**: The system identifies the "main" content of a page, stripping away boilerplate like navigation, sidebars, and footers.
4.  **Sanitization**: HTML is cleaned by removing scripts, comments, and non-essential elements.
5.  **Conversion**: HTML is transformed into structured Markdown using the `html-to-markdown` library.
6.  **AI Enrichment**: If configured, the Markdown content is sent to an LLM (OpenAI, Anthropic, or Google) to generate a concise summary, categorize the document, and suggest relevant tags.
7.  **Decoration**: YAML frontmatter is generated and prepended to the Markdown content.

## Storage Mechanisms
The system utilizes two primary storage layers:

*   **Persistent Cache**: A Badger DB (key-value store) is used to cache raw fetched pages and responses, reducing redundant network requests and respecting source rate limits.
*   **Local File System**: The final destination for processed data.
    *   **Markdown Files**: Individual `.md` files organized in a directory structure (optionally flat).
    *   **JSON Metadata**: A consolidated `metadata.json` file acting as an index for all processed documents.

## Data Validation
Validation occurs at several boundary points:

*   **Configuration**: Input options are validated for sanity (concurrency limits, timeouts, directory paths).
*   **Source Validation**: URLs are checked for validity and filtered based on domain or path constraints.
*   **Content Validation**: The system verifies content types (HTML vs Markdown) before processing.
*   **LLM Response Validation**: AI-generated metadata is validated against a strict JSON schema (summary/tags/category) with retry logic for malformed responses.

## Output Formats
The system produces three main types of output:

*   **Markdown Files**: Clean, readable Markdown containing the documentation content and YAML metadata.
*   **Consolidated JSON Index**: A structured `metadata.json` containing a summary of all processed files, including their paths, URLs, and AI-generated metadata.
*   **Structured Logs**: Execution progress and metrics provided via Zerolog (JSON or pretty-printed console output).