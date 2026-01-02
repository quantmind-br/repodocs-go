# Data Flow Analysis

## Data Models
The system uses several Go structures to manage documentation data throughout its lifecycle:
- **`domain.Page`**: Captures raw fetched content, including the URL, raw bytes, content type, and HTTP status code.
- **`domain.Document`**: The primary internal representation of processed content. It includes the Markdown content, original HTML, and rich metadata (Title, Description, ContentHash, Word/Char counts, Links, Headers).
- **`domain.SimpleMetadata`**: A specialized, lightweight structure optimized for JSON output and LLM evaluation, excluding technical fields like hashes.
- **`domain.CacheEntry`**: Encapsulates cached pages with expiration timestamps.
- **`domain.Sitemap` & `domain.LLMSLink`**: Structures used for discovery and navigation during the initial phase of extraction.

## Input Sources
Data enters the system through various external sources, determined by specialized strategies:
- **HTTP/HTTPS**: Standard web scraping of documentation sites.
- **Sitemaps**: Parsing `sitemap.xml` files for systematic URL discovery.
- **GitHub/Git**: Cloning repositories or fetching specific paths from Git providers.
- **llms.txt**: Parsing standardized discovery files for LLM-friendly documentation.
- **Wiki**: Extracting content from Wiki-style structures.
- **Local Files**: Reading existing documents during development or testing.

## Data Transformations
Data undergoes a multi-stage pipeline as it moves from raw input to final output:
1. **Encoding Normalization**: Raw bytes are converted to UTF-8 using a dedicated encoding converter.
2. **Content Extraction**: Target content is isolated from raw HTML based on CSS selectors or automatic "readability" detection.
3. **HTML Sanitization**: Removal of navigation elements, scripts, comments, and unwanted tags.
4. **Markdown Conversion**: Sanity-checked HTML is transformed into clean Markdown with configurable styles (fenced code blocks, atx headings).
5. **Metadata Enhancement**:
    - **Technical**: Automated calculation of word counts, character counts, and SHA256 content hashes.
    - **AI-Driven**: Optional processing via LLMs (OpenAI, Anthropic, Google) to generate summaries, tags, and categories.

## Storage Mechanisms
Information persists in two primary ways:
- **BadgerDB Cache**: A local KV store (using BadgerDB) stores raw fetched pages (`CacheEntry`) to prevent redundant network requests and respect rate limits.
- **Local Filesystem**: The final processed output is written to a structured directory on disk.
- **In-Memory Collector**: During execution, the `MetadataCollector` aggregates metadata for all processed documents.

## Data Validation
Validation occurs at multiple boundaries:
- **Config Validation**: Ensures workers, timeouts, and paths are valid before execution.
- **URL Validation**: Checks URL schemes and formats before fetching.
- **Content-Type Validation**: Verifies that fetched content can be processed (e.g., text/html).
- **Integrity Checks**: Content hashes verify if data has changed since the last fetch.
- **Schema Validation**: Sitemaps and LLM responses are validated against expected structural schemas.

## Output Formats
The system exports processed documentation in several standardized formats:
- **Markdown (.md)**: Individual files containing the converted content.
- **YAML Frontmatter**: Injected at the top of Markdown files, containing metadata (Title, URL, Summary, etc.).
- **JSON Metadata Index**: A consolidated `repodocs.json` file containing the `SimpleMetadataIndex` for all extracted documents, facilitating integration with RAG pipelines or other tools.
- **Asset Files**: Optional preservation of images and other static resources found during extraction.