# Data Flow Analysis

## Data Models

The system centers around several key data structures that represent information at different stages of the lifecycle:

*   **`Response`**: Represents raw fetched data from a source (HTML, Markdown, or binary). Includes status code, body, headers, and content type.
*   **`Page`**: An internal representation of a fetched page before conversion.
*   **`Document`**: The primary data structure containing processed information.
    *   **Core**: URL, Title, Content (Markdown), HTMLContent.
    *   **Metrics**: WordCount, CharCount, ContentHash.
    *   **Structure**: Headers (h1-h6 mapping), Links.
    *   **AI Metadata**: Summary, Tags, Category.
*   **`Metadata`**: A JSON-serializable version of `Document` used for the final output index.
*   **`MetadataIndex`**: A global structure consolidating all processed documents, including aggregate stats like `TotalWordCount` and `GeneratedAt`.
*   **`CacheEntry`**: Data structure for persistent storage of fetched content in the Badger database.

## Input Sources

Data enters the system through multiple entry points depending on the detected strategy:

*   **Web URLs**: Crawled via `CrawlerStrategy` or targeted via `SitemapStrategy`.
*   **Git Repositories**: Cloned or downloaded as archives via `GitStrategy`.
*   **Standard Documentation Files**: Discovered in repos (e.g., `.md`, `.txt`, `.rst`).
*   **Sitemaps & Wiki Pages**: Parsed for URL discovery.
*   **LLMS.txt**: Specialized discovery format for LLM-friendly documentation.
*   **User Configuration**: YAML/Environment variables defining crawl depth, selectors, and LLM settings.

## Data Transformations

Data undergoes a multi-stage transformation pipeline:

1.  **Encoding Normalization**: Raw bytes are converted to UTF-8.
2.  **HTML Extraction**: Uses a combination of CSS selectors and Readability-like logic to extract the "main" content and title while discarding noise.
3.  **Sanitization**:
    *   Removes non-content tags (`<script>`, `<style>`, `<form>`, etc.).
    *   Strips UI elements by ID/Class (`sidebar`, `nav`, `footer`).
    *   Removes hidden elements and empty containers.
4.  **URL Normalization**: Converts relative links and image sources to absolute URLs based on the source origin.
5.  **Markdown Conversion**: HTML is transformed into clean Markdown using `html-to-markdown`.
6.  **AI Enhancement**: (Optional) The Markdown content is sent to an LLM provider (Anthropic, OpenAI, or Google) to generate a summary, tags, and category.
7.  **Statistics Generation**: Markdown is stripped of formatting to calculate accurate word and character counts.

## Storage Mechanisms

*   **BadgerDB**: A persistent key-value store used to cache HTTP responses to avoid redundant network calls and improve performance on subsequent runs.
*   **Local File System**:
    *   **Markdown Files**: Persisted as `.md` files in a structured or flat directory.
    *   **JSON Metadata**: Global `metadata.json` stores the consolidated index of all processed documents.
*   **Memory**: `MetadataCollector` maintains an in-memory registry of all processed documents during the execution lifecycle.

## Data Validation

*   **Configuration Validation**: Checks for valid concurrency limits, timeouts, and directory paths.
*   **URL Filter Validation**: Ensures only URLs within the allowed domain or path prefix are processed.
*   **Content-Type Validation**: Filters out non-document content (images, PDFs, binary files) unless explicitly allowed.
*   **Sanitization Rules**: Strictly removes executable scripts and irrelevant UI components to ensure data purity for LLM consumption.

## Output Formats

The final data leaves the system in two primary formats:

*   **Markdown (.md)**: Files containing the processed documentation content, prefixed with **YAML Frontmatter** (containing Title, Source URL, Summary, etc.).
*   **JSON (.json)**: A consolidated `metadata.json` file providing a manifest of all documents, their relative file paths, and extracted metadata.