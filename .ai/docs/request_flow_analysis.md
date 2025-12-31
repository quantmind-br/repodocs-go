# Request Flow Analysis

## API Endpoints
The application is a CLI-based tool. The "API" is exposed through the command-line interface:

- `repodocs [url]`: Primary entry point to extract documentation from a source.
- **Global Flags**:
  - `--output, -o`: Specifies the output directory (default: `./docs`).
  - `--concurrency, -j`: Number of concurrent workers (default: `5`).
  - `--limit, -l`: Maximum number of pages to process.
  - `--max-depth, -d`: Maximum crawl depth (default: `4`).
  - `--render-js`: Forces JavaScript rendering for the request.
  - `--no-cache`: Disables the caching layer.
  - `--filter`: Base URL filter for crawling.
  - `--exclude`: Regex patterns to exclude specific paths.
  - `--json-meta`: Generates JSON metadata files alongside Markdown.

## Request Processing Pipeline
1.  **CLI Initialization**: `cobra` parses command arguments and `viper` loads configuration from flags and config files.
2.  **Orchestration**: The `Orchestrator` receives the URL and options, coordinating the lifecycle.
3.  **Strategy Detection**: `DetectStrategy` analyzes the URL pattern to route it to the appropriate handler.
4.  **Dependency Setup**: The orchestrator initializes a shared dependency container:
    - **Fetcher**: A stealth HTTP client (`tls-client`) with retry and cache support.
    - **Renderer**: A browser-based renderer (`Rod`) for JavaScript-heavy sites.
    - **Cache**: A persistent store (`BadgerDB`) for HTTP responses.
    - **Converter**: A processing pipeline for HTML-to-Markdown transformation.
    - **Writer**: An output handler for filesystem operations.
5.  **Strategy Execution**: The selected strategy (e.g., `CrawlerStrategy`, `GitStrategy`, `SitemapStrategy`) executes its specific extraction logic.
6.  **Data Processing**:
    - **Fetching/Rendering**: Content is retrieved via the `Fetcher` (Stealth HTTP) or `Renderer` (Headless Browser).
    - **Conversion Pipeline**: HTML passes through multiple stages: `ConvertToUTF8` -> `ExtractContent` (CSS selectors/Readability) -> `Sanitize` -> `MarkdownConverter`.
7.  **Output Generation**: The `Writer` saves the final Markdown and metadata to the disk.

## Routing Logic
Routing is handled by the `DetectStrategy` function in `internal/app/detector.go`. It performs pattern matching on the input URL:
- **`llms.txt` Strategy**: For URLs ending in `/llms.txt`.
- **`pkggo` Strategy**: For URLs containing `pkg.go.dev`.
- **`sitemap` Strategy**: For URLs ending in `sitemap.xml` or containing `sitemap` keywords with XML extensions.
- **`git` Strategy**: For SSH patterns (`git@`), `.git` suffixes, or base repository URLs from GitHub, GitLab, and Bitbucket.
- **`crawler` Strategy**: The default fallback for any standard `http://` or `https://` URLs that don't match specific patterns.

## Response Generation
Responses are generated as physical files on the local filesystem:
1.  **Markdown Files**: The core output, converted from the cleaned HTML content.
2.  **Frontmatter**: YAML-style metadata (URL, Title, Description, Date) is prepended to the Markdown content.
3.  **JSON Metadata**: If the `--json-meta` flag is active, a structured `.json` file is produced containing document statistics (word count, char count) and link maps.
4.  **File Mapping**: `internal/utils/url.go` translates URLs into sanitized filesystem paths, maintaining site structure unless `--nofolders` is specified.

## Error Handling
Error handling is implemented using custom error types and a retry mechanism:
- **Custom Domain Errors**: `internal/domain/errors.go` defines specific errors like `ErrBlocked`, `ErrRateLimited`, and `FetchError`.
- **Retry Logic**: The `Retrier` in the fetcher package handles transient network issues and specific HTTP status codes (429, 502, 503, 504) with exponential backoff.
- **Stealth & Fallbacks**: The `GitStrategy` attempts fast archive downloads before falling back to a full `git clone`. The `Fetcher` uses browser-like TLS fingerprints to avoid detection errors.
- **Context Management**: A global context with cancellation is used to handle `SIGINT`/`SIGTERM` signals, ensuring browsers and database connections are closed gracefully.
- **Validation**: Early validation in the `Orchestrator` prevents processing of unsupported URL formats.